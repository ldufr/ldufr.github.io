---
layout: default
title: Stumbling on the solution
---

For about half a decade, two developers and I are maintaining an open source project, [GWToolboxpp](http://www.gwtoolbox.com/), which offer several quality of life improvements for the game [Guild Wars](http://guildwars.com/).
We achieved a considerable amount of stability over the years considering that we rely on undocumented
structures, functions, flows and variables that we had to reverse and that change every now and then.
We have also named those different constructs and we find them by searching for binary patterns in the
game executable. Finally, we created processes to document those structures, so let's start by looking
at how we do some of that.

First of all, we don't use any threads. We insert ourself in the game thread and we make sure to
not block it. This get rid of a class of bugs, the race conditions. This is an extremely important
guarantee that allow us to write robust code.

Secondly, we distinguish two kinds of structure. The first kind are the "contexts" and the second
kind are the "game entities". The contexts are how the game is organized, those structures have an
unique instance and are mostly the path used to access the game entities. The game entities are all
the structures that are "typical" objects and they often live in arrays. Moreover, we document all
the offsets of the structures and the unknown members are named "h" followed by their offset.
Finally, we document the size of the structures and `static_assert` it. e.g:
```cpp
struct Skillbar { // total: 0xBC/188
    /* +h0000 */ uint32_t agent_id; // id of the agent whose skillbar this is
    /* +h0004 */ SkillbarSkill skills[8];
    /* +h00A4 */ uint32_t disabled;
    /* +h00A8 */ uint32_t h00A8[2];
    /* +h00B0 */ uint32_t casting;
    /* +h00B4 */ uint32_t h00B4[2];

    bool IsValid() const { return agent_id > 0; }

    SkillbarSkill *GetSkillById(Constants::SkillID skill_id);
};
static_assert(sizeof(Skillbar) == 188, "struct Skillbar has incorrect size");
```
It is fairly easy to figure out the size of most game entities, because the game stores them
in arrays. Indeed, as soon as the structure is stored in an array, the access will leak the
size of the structure, because it's needed to compute the offset from the base of the array.
For instance, the assembly to access a `Skillbar` from the array is:
```
; eax is the index in the array
; esi is the base of the array
69C0 BC000000              | imul eax,eax,BC
03C6                       | add eax,esi
3B18                       | cmp ebx,dword ptr ds:[eax]
```
We can easily infer that the size of the structure is 0xBC in this case. Sometime, especially
with older version of MSVC, other tricks are used. For instance, combination of lea, shl and add.
Nonetheless, we know the size of most entities that we documented.

Thirdly, Guild Wars developers named "Agent" the structure that hold information about game units.
As expected, the Agent structure is one of the largest and can be found [here](https://github.com/GregLando113/GWCA/blob/0e72d64d5af0cd111b6161589e1f45e4c99fda98/Include/GWCA/GameEntities/Agent.h#L13-L250).
This structure is fairly special, because it's one of the rare structure having virtual functions and
there is no array of agents, but instead an array of agent pointers. Consequently, we don't know the size
of the structure.

Recently, someone asked me on GWToolbox Discord what was the size of Agent. So, I explained to him
that we didn't know, because we only have an array of pointers, but regardless, there is no
"size of Agent". Indeed, the structure is a parent class and has child classes with different sizes.
Furthermore, the agent structure has virtual functions, so we can easily find the number of child
classes, simply by counting the number of different virtual tables. I also told him that it would
be easy to find the size of the different child classes by simply looking where the instances of
those classes are allocated. To make it clearer, I showed how to do it and it was the following:

First, I wrote a Python script using [process.py](https://github.com/reduf/process) to dump the
different virtual tables. The script is:
```python
from process import *
 
proc = GetProcesses('Gw.exe')[0]
scanner = ProcessScanner(proc)
address = scanner.find(b'\xFF\x50\x10\x47\x83\xC6\x04\x3B\xFB\x75\xE1', +0xD)
agent_array, = proc.read(address, 'I')

addr, capacity, size = proc.read(agent_array, 'III')
agents = proc.read(addr, 'I' * size)

vtables = set()
for agent in agents:
    if agent != 0:
        vtable, = proc.read(agent, 'I')
        vtables.add(vtable)
 
print('len(vtables):', len(vtables))
for i, vtable in enumerate(vtables):
    print(i, '=>', hex(vtable))
```

It gave us three different virtual tables and it was not surprising, because the game has three
distinct types of agent, namely the livings, the items and the gadgets. So the virtual tables are at
the RVAs 0x5D1EEC, 0x5D2784 and 0x5D2868 for the version 37015. We can point out that those virtual
tables are indeed living in ".rdata", the read only section, because they are shared between every
instances of the same class. But now that we have that, we can easily find the references of those
virtual tables in the code section and we also know that it will give us all the constructors for
it's associated class. If we take the first virtual table reference which is the first constructor
and we look at the references of this constructor, we obtain the following:
```
01554AF6 | 68 FB050000                | push 5FB                   |
01554AFB | 68 94FA7B01                | push gw.17BFA94            | 17BFA94:"p:\\code\\gw\\agentview\\avapi.cpp"
01554B00 | 33D2                       | xor edx,edx                |
01554B02 | B9 E4000000                | mov ecx,E4                 |
01554B07 | E8 4483D1FF                | call gw.126CE50            |
01554B0C | FF75 0C                    | push dword ptr ss:[ebp+C]  |
01554B0F | 8BC8                       | mov ecx,eax                |
01554B11 | FF75 08                    | push dword ptr ss:[ebp+8]  |
01554B14 | E8 B7C80100                | call gw.15713D0            | <== The constructor call is here
```
Notice that the "this" pointer is passed in ecx, which is set with the line `mov ecx,eax`.
This eax is the result of the previous function call which is a bit particular. The arguments are 0xE4,
0, "p:\\code\\gw\\agentview\\avapi.cpp" (i.e. __FILE__) and 0x5FB (i.e. __LINE__). If you have reversed
Guild Wars long enough, it's obvious that this is their allocator which is similar to [_malloc_dbg](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/malloc-dbg?view=vs-2019).
In fact the prototype is:
```cpp
#define ALLOCF_ZERO (1)
__fastcall void * MemAlloc(size_t size, int flags, const char *file, unsigned line);
```

We repeat the same operations for all the different virtual tables and we find the class hierarchy
and class sizes.
```cpp
struct Agent
{
};
 
struct AgentGadget : public Agent // total: 0xE4/228
{
};
 
struct AgentLiving : public Agent // total: 0x1C0/448
{
};
 
struct AgentItem : public Agent // total: 0xD4/212
{
};
```

Up to this point, there was nothing hard, but it was an nice explanation and visualization of
how virtual functions generate RTTI, namely the virtual table and how those virtual tables leaks
the addresses of the constructors.

This explanation made me think of a bug we had in GWToolbox. It's a bug that wasn't frequent, but every
now and then a GWToolbox user was crashing. I did get one crash dump, but the code looked correct.
The bug was [here](https://github.com/HasKha/GWToolboxpp/blob/dd2e5ca12352b178d69539a3c5c948593ed86441/GWToolbox/GWToolbox/Widgets/Minimap/AgentRenderer.cpp#L371-L389)
more specifically the last line of:
```cpp
for (size_t i = 0; i < agents.size(); ++i) {
    GW::Agent* agent = agents[i];
    if (agent == nullptr) continue;
    if (agent->GetIsDead()) continue; // <== crash on this line
```
The function `GetIsDead` is define as:
```cpp
inline bool GetIsDead() const { return (effects & 0x0010) != 0; }
```
And, the effect is declared as:
```cpp
 /* +h0138 */ uint32_t effects; // Bitmap for effects to display when targeted. DOES include hexes
 ```

At this point, it was obvious. We were accessing the offset +0x138 without making sure it was a
`AgentLiving` which was causing access violation on rare occasions. Bug fixed! In fact, I added the
class hierarchy and solved a lot of potential bugs as well as removing some comments that was pointing
in that direction. i.e:
```
/* +h00C8 */ uint32_t item_id; // Only valid if agent is type 0x400 (item)
/* +h00D0 */ uint32_t extra_type; // same as GadgetId. Could be used as such.
```

So, the conclusion is that our code didn't have a bug, but our abstraction did. There is always
difficulties in those kind of projects, because you don't have access to the abstractions. You
have to make a lot of guesses and heuristic tests. Nonetheless, this case could have been easily
avoided when we noticed the structure had a virtual table.

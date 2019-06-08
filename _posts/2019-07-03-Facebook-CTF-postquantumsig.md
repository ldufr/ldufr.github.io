---
layout: default
title: Facebook CTF postquantumsig
---

# Description

Our hot new cryptocurrency uses powerful techniques to make the asymmetric key
signatures just as hard to crack on a quantum computer as a binary one. Maybe
we forgot something though? Send some money to a weird address if you get any.

nc challenges.fbctf.com 8088

--------------------------------------------------------------

The challenge begin with the above description and two files. The first one
is a python script named `verifier.py` and the second is a csv database of
transactions named `signature.csv`. The python script verify that the
transactions in the database are genuine. Moreover, when using the netcat
command, the server is asking to enter a signature row.

Clearly, we need to craft a transaction that send money to a custom address
and that transaction needs to be validated by `verifier.py`. A transaction
is a row in the database and the format of such a row is:
```
sender identity, transaction msg, H1, ... H512, R1:O1, ...)
```

H1 to H512 are sha256 digests, the R(n) are numbers (0 or 1) and the O(n)
are sha256 digests. Moreover, R(n) and O(n) are optional.

The core of verification algorithm is in the function `verify_signed_message`
which does:
```python
def verify_signed_message(msg_w_signature):
    identity, h_msg, signature, others = parse_signed_message(msg_w_signature)
    sign_msg = msg_to_hashes(h_msg, signature)
    initial = make_top_hash_from_leaves(sign_msg)
    top = make_top_hash_from_others(initial, others)
    return (identity, top)
```
And `parse_signed_message` is:
```python
def parse_signed_message(msg_w_signature):
    # separate a formatted message and signature into components
    top_identity = msg_w_signature[0]
    msg = msg_w_signature[1]
    h_msg = s256(msg)
    signature = msg_w_signature[2:514]
    others = group_by_n(msg_w_signature[514:])
    return (top_identity, h_msg, signature, others)
```

There is a major mistake in `parse_signed_message` on the line
`signature = msg_w_signature[2:514]`. Indeed, the slice (i.e. 2:514) is
clamped to the boundaries of the tuple `msg_w_signature`. So if there is
only 2 elements in `msg_w_signature`, `msg_w_signature[2:514]` will returns
an empty tuple.

Hence, if we have a transaction with the format
`sender identity, transaction, H1, H2`, the function `msg_to_hashes` will
only create one group of signatures and hence only iterate once.
```python
def msg_to_hashes(msg, signature):
    # turn a message with signature into an ordered list of key pairs
    bit_stream = bit_stream_from_msg(msg)
    sign_stream = group_by_n(signature, 2)
    return_stream = []
    for bit, sign in zip(bit_stream, sign_stream):
        if bit:
            return_stream.append(sign[0])
            return_stream.append(s256(sign[1]))
        else:
            return_stream.append(s256(sign[0]))
            return_stream.append(sign[1])
    return return_stream
```

So, we will have the following operations to compute our identity.
```python
# msg_to_hashes
return_stream = []
# b1 = first bit of h_msg
if b1 == 1:
    return_stream.append(H1)
    return_stream.append(s256(H2))
else:
    return_stream.append(s256(H1))
    return_stream.append(H2)

# make_top_hash_from_leaves which implement the Merkle signature scheme
identity = s256(return_stream[0] + return_stream[1])
```

Now, assuming that b1 is 1, we need to find H1 & H2 such that
`sha256(H1 + sha256(H2)) = identity`. For that we can use
`make_top_hash_from_others`.
```python
def make_top_hash_from_others(initial, others):
    # traverse asymmetic segments of public key signature to get public key
    top_hash = initial
    for rank, other in others:
        if int(rank):
            top_hash = s256(other + top_hash)
        else:
            top_hash = s256(top_hash + other)
    return top_hash
```

Indeed, if the last hash in others is of rank 1, we get:
```python
top_hash = s256(other + top_hash)
# or top_hash = s256(top_hash + other)
top_hash = s256(other + top_hash)
```
That is:
```
HH = H(H2) # "H2 = top_hash + other" or H2 = "other + top_hash"
identity = H(H1 + HH) # H1 = other
```

Similarly if b1 is 0 and the rank of last "other" is 0.

Because of that, the difficulty boils down to building a transaction message
with first bit of the hash equals to an existing transaction of the same sender.
It's easy to do, by changing the ammount of zuccoins sent.

In my case the final transaction was:
```
Dest: 34a4e8ed19e3e08cf4719af6550afde1f97137dfe8af1565d8c9a8a5cbcb99df
Msg:  34a4e8ed19e3e08cf4719af6550afde1f97137dfe8af1565d8c9a8a5cbcb99df sent 102.10247476840064 zuccoins to 59bb2642bbf7f009e9bf3c6c45ac30cbc1a897235e0e3a407d49966aa4867a93

H1: 1a96f6d19e9258935a97d41f9507ba9b9241113d641362fa55642c4a1dad277a
H2: e7cfafb7f08de90f7cb44994024b6baa870b8a780837c0bb9e0835e6c4a856a03163753c32343907f466d4af6823b8db62e5594cc90f97762c86104ceb6a9217
```

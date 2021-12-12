---
layout: default
title: Secrets in configuration files is wrong!
---

## The Goal

The goal of this article is to describe an approach that can be used to specify
secrets or credentials without ever writing them in configuration files.
Moreover, the proposed approach is meant to be non-restrictive, compatible with
most popular secrets management tools and still offer easy implementation for
teams that do not have the budget or the incentive to setup more complicated
solutions. Finally, this approach re-use a very powerful abstraction that has
been proved to work extremely well, the "object name".

It is important to specified that this article do not try to define how to
securely store passwords, how to securely exchange them, how to securely manage
access rights, but rather propose a scheme that would improve interoperability
of software.

## The problem

Most software having some kind of authorization usually need secrets specified
somewhere. Consider the following scenario. You write a service that output some
logs and you want to re-use the well proven technology of [Filebeat](https://www.elastic.co/beats/filebeat)
to write the logs from a file to an [Elasticsearch Database](https://www.elastic.co/).
If you use authentication, you may need to specify a password somewhere, but
will almost certainly have to specify some kind of secret. This is usually done
in the configuration file.

There is several problems here.
* It's very inconvenient to version this file in your version control.
* It's very inconvenient to deploy, because the secrets need to be specified as
  part of the deployment.
* It's inconvenient to use on a dev machine, because the developers have to
  give the plain text password, meaning he has to decrypt it manually.

From the point of view of Filebeat developers, it make perfect sense. The
configuration files is what hold the variable controlled by the users. I can
appreciate the reason, but I still think that storing secrets in a configuration
file is a broken approach and I will try to explore an alternative way that
would solve this problem, but that would also be easy and powerful.

My approach is to list some observations I have down during my work with this
problem and try to use those observations to construct a scheme with the
learning.

## Observations

1. The developers and the servers have drastically different usage of secrets.
   A solution that fits both, probably do so very poorly.
2. A configuration file should never contain secrets, encrypted or not. It
   doesn't fit with configuration versioning, it often add extra steps to be
   used and it's incompatible with solutions that are often taken when a product
   get more mature. (i.e. Key Management Service)
3. Connecting to your developer account (e.g. Windows account within corporate
   network) is the contract that define what secrets you have access to. No
   further manual action should be necessary.
4. Secrets are not limited to a tuple of username password. They can contain
   certificates, private keys, token and more. Assume they are unpredictable.

## Existing solutions

### Key Management Service
Several solutions to better manage secrets exists. [Vault by HashiCorp](https://www.vaultproject.io/),
[Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/) and
[AWS Key Management Service](https://aws.amazon.com/kms/) to name a few. Those
solutions are rather complex solutions that also offer far more powerful
features. Secrets rotation, access right management, single use secrets, and
more. Where large team will want to leverage those features it can be an
important burden for smaller team or for young project. Moreover, experience
shows that currently very few 3rd party project support those different
providers well.

### Encrypted credentials in configuration
Other simpler approach exists. One particularly interesting is [Mozilla SOPS](https://github.com/mozilla/sops)
which also have a lot of similarities with [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html).
Those two solutions are quite interesting, but also fairly inconvenient to use.

First of all, the files need to be pre-process before being use. Where this can
be acceptable when using a software that only accept unencrypted secrets in the
configuration file, it's particularly annoying when you own software could read
and decrypt the file automatically.

Secondly, it's not compatible with other approaches making it harder to update
or potentially use two different solutions. Indeed, my first observation is that
the use cases for developers and for servers is drastically different.

### Certificates and Private Keys
Let's start by looking at this from a server perspective.

What is the point to have accounts if I have N servers using the exact same
account? Then it mostly boils down to the secret and a certificate is a
convenient way to store a very long secret. Moreover, in this case, it's common
to specify the path to the private key in the configuration file. Including
the private key in the configuration file (e.g. with PEM encoding) would violate
my second observation.

## Alternative solution

### A very powerful abstraction
If we are going to fit with my second observation, we won't be able to specify
the secrets in the configuration file. Yet, we still need to specify the secret
somehow. So, let's define the **Secret Name** with the following properties:
* The secret name uniquely identify the secret.
* The secret name is a sequence of Unicode characters.
* The secret name can contain alphanumeric characters (0-9, a-z, A-Z), forward
  slash `/`, hyphen `-`, underscore `_` and period.

Example of valid secret name:
* secret-keys
* uat/database/db-writer.sec

This can generally be understood as path, slightly more restrictive and
compatible with URL, object store, file path, object name (e.g. named pipe)
and more.

### The providers
This article do not want to define how to retrieve the secrets, because it would
simply not achieve what it's trying to do. If that was the goal, we could simply
refer to existing solutions. Yet, to explain the rational of the secret name
format we can look at multiple potential secret providers.

Note that one goal of the format is to be able to seemingly switch between
providers where one could be use by developers and an other by servers.

1. First of all, let's consider the provider which is the file system starting
   at the directory `$secrets`. The secret `uat/db-writer` would be the content
   of the file `$secrets/uat/db-writer`. For now, do not consider the format of
   the file, we will comeback to that in a later section.

2. Using the filesystem may not be ideal, but is actually far more powerful than
   one might think. For instance, you could leverage [Windows Active Directory](https://en.wikipedia.org/wiki/Active_Directory)
   technology in combination with [Shared resource](https://en.wikipedia.org/wiki/Shared_resource).

   Let's say `$secret` is `\\company.com\ProgramX\secrets`, the path to the
   secret with name `uat/db-writer` would be `\\company.com\ProgramX\secrets\uat\db-writer`.
   In such case, [Active Directories Security Groups](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups)
   could be used.

   This can be even more powerful on Windows if a service run as a service
   account that has read writes on `\\company.com\ProgramX\secrets\prod\`, but
   where developers do not.

   Finally, it's important to notice that connecting to your Windows account
   would "reveal" all secrets you have access to and also allow for secrets
   rotation.

3. An other usage of the file system can be interesting. Let's assume that a
   team is using [Git](https://git-scm.com/) and wants to version it's secrets
   as part of the repo which is cloned at the directory `$repo`.

   First of all, an initial assumption is that secrets shouldn't and aren't
   saved as plain text in Git. The tools [git-crypt](https://github.com/AGWA/git-crypt)
   and [git-secret](https://git-secret.io/) can be of utility in that regard.

   Then, you can re-use the filesystem provider on the decrypted directory.

4. Using encrypted Zip archive can be quite interesting if you already have
   the code necessary to work with Zip archives, including encryption. Indeed,
   the provider can open the encrypted archive and simply use it as a
   filesystem.

   Note that although it would work with Git, it's not ideal, because it's not
   Git friendly as in it's not text based and although you know when at least
   one secret change, you don't know which secret changed. This is the result
   of using a single binary file.

   Similar advantage and problem would exists if you were using a KeePass
   database and the library to programmatically read it.

5. Single credential file could also be used. For instance, with JSON:
   ```json
   {
     "uat/db-writer": {
       "username": "db-writer",
       "password": "Passw0rd!"
     },
     "uat/db-reader": {
       "username": "db-reader",
       "password": "pASSW0RD!"
     }
   }
   ```

   I'm not sure why this would be better than file, but not that if you want to
   version it in Git with git-crypt or git-secret, you will loose the ability
   of knowing which secret was modified. You will simply be able to know that
   at least one was modified.

6. All previous example are generally easy to implement, but the nice thing about
   the proposed format is that it's compatible with popular Key Management Service.

   Vault and AWS KMS works similarly. I won't dive too much in, because they are
   so much more than a simple secrets provider. Nonetheless, what is interesting
   for this article  is that you can do REST requests to read the secrets.
   See the [API here](https://www.vaultproject.io/api). Moreover, the server
   response is usually JSON formatted which we will comeback to in the next
   section.


### Format for the secrets
Until now, we didn't specify what should be the format of the request. We
described how from a single secret name format, we could use different providers
to "read" the secret. Similarly as for the providers, this doesn't need to be
answered by this document, because the implementation could be different.
Nonetheless, I will recommend to use the JSON format.

First of all it's a structured format that will allow you to support all
potential secret type. Secondly, if you ever choose to move to a Key Management
Service and use REST request, you will almost certainly use the JSON format.

## Support of multiple providers at the same time

While the propose approach doesn't prevent nor force implementation to support
several providers, we can look at the advantages and disadvantages of doing so.

First of all, we can try to think of example where it would be useful. I have
one very simple use cases in mind. Consider wanting to use a encrypted zip
archive for secret provider as described earlier. I use the encrypted zip as
an example, because it's easier, but the main use case would be with a Key
Management Service. This encrypted archive will need a password to be open.
Where should this password live? Including it in the configuration files would
go against the observations made earlier. This seem to be a case where a
generalization to support multiple providers make sense. It's not strictly
necessary as an application could assume the file system provider, but it might
not be too hard to support multiple providers in this case. One of the
difficulty in this case is how do you specify which providers need to be used?

## Conclusion

During this article we looked at several very concrete implementations of
providers. The goal of this article isn't to ask projects to implement those
different providers and especially not to ask to implement all of them.

The main goal is to ask projects to **never** ask users to specify secrets in
the configuration files. Instead, ask for the secret name and document how your
provider work. In many case, a filesystem providers will be more than enough,
because it's easy for a tool to use their own provider to download files before
installing or starting a 3rd party tool. In fact, tools that request secrets in
the configuration file forces users to consider the all configuration file as a
secret or to manually parser and re-write the configuration file.

# pam-onelogin
## What is it ?
pam-onelogin is a pretty complete pam/nss stack for using [OneLogin](https://www.onelogin.com)
as authentication source (with MFA) and user/group lookups.

It consists of the following parts,
* a standalone binary which fetches and caches users and groups
* a standalone nss-library which handles user/group lookups
* a standalone pam-library which handles authentication from the pam-stack.

The idea is simple and the goal is to have OneLogin as authentication source, as
well as users/group information.

You simply specify a required onelogin-role that the users must have to gain
access to the system (primarily via SSH). Only users that has that required role
in OneLogin will be able to access the system, the role will also be 'translated'
into standard Linux group (ie. users that has the role *example-role* in oneLogin,
will belong to a Linux group called *example-role*).

Users do not need to exist on the system since they will be created dynamically
when running the onelogin-mkcache-binary. Users and group information will then
be fetched from OneLogin and cached locally. This information is then used when
looking up users and groups through the onelogin-nss-library.

When an authentication attempt is being made, the users is looked up from cache,
and if found, the user is the asked to authenticate with password and OTP.

![Alt Text](./demo.gif)

**:warning: NOTE :warning:**
* This should **only** be used as proof-of-concept - **you have been warned**.

## Why ?
Well the reason is quite simple,
* If you have an organization that uses [OneLogin](https://www.onelogin.com) for
user and group administration and you want to utilize this on servers, you can
simply use this pam/nss-stack. This gives you the benefit of moving the administration
for user, groups **and** access from your servers to [OneLogin](https://www.onelogin.com).

## :computer: How do I use it ?
### Prerequisites

* A working OneLogin organization
* OneLogin API-keys
* A standard Linux distro with gcc, curl-devel and pam-devel libraries available.
  * Tested configuration on CentOS 7, 8 and 9.

> **Note**<br><br>
> You need **two** OneLogin keys.
> * One API-key with the "read users" permission
> * One API key with the "authentication only" permission
>
> For some reason there is no API-permission that grants you both "read users"
> and "authentication only", except the permissions "Manage users" and "Manage all",
> which both gives you permission to **delete** users, which you probably don't want.
>
> You can read more about the API permissions [here](https://developers.onelogin.com/api-docs/1/getting-started/working-with-api-credentials).
> If you however decide to use **one** API-key (that has "Manage users/ Manage all"
permission), you can just configure pam-onelogin (onelogin_client_read_id/secret,
onelogin_client_auth_id/secret) to use the same API-key (although I don't recommend it).

### Installation
While this has only been tested on CentOS 7 / 8 / 9, it should work on any modern
distribution. It only has two "external" requirements,
* pam-devel
* libcurl-devel

#### CentOS 7 / 8 / 9
This is literally a copy and paste from a clean install of CentOS 9.
1. [Install dependencies and compile pam-onelogin](#Install-dependencies-and-compile-pam-onelogin)
2. [Edit config and create cache](#Edit-config-and-create-cache)
3. [Add onelogin to nsswitch and verify user / group lookup](#Add-onelogin-to-nsswitch-and-verify-user--group-lookup)
4. [Add pam_onelogin.so to SSH](#Add-pam_oneloginso-to-SSH)
5. [Configure SSH and SELinux](#Configure-SSH-and-SELinux)

#### Install dependencies and compile pam-onelogin
```bash

# Get dependencies and clone repo
$ > sudo [ dnf | yum ] install git gcc libcurl-devel make pam-devel
$ > git clone https://github.com/patchon/pam-onelogin
$ > cd pam-onelogin

# Compile and copy libraries and binary into correct path's
$ > sudo make install
```

#### Edit config and create cache
```bash
# Edit configuration-file
$ > sudo vim /etc/pam-onelogin/pam_onelogin.yaml

# You *must* specifically set the following parameters,
# * onelogin_client_read_id
# * onelogin_client_read_secret
# * onelogin_client_auth_only_id
# * onelogin_client_auth_only_secret
# * onelogin_region
# * onelogin_subdomain
# * onelogin_user_domain_append
# * onelogin_user_roles

# Once these parameters are set, run the onelogin-mkcache-binary
$ > sudo onelogin-mkcache

# If no errors occured you should be good to go, you can verify that the cache
# has been generated by looking at the 'cache-files'.
#
# In the example below, jane.doe and uncle.tom both has a onelogin-role called
# 'sysadmins'. Jane Doe also has a onelogin-role called 'developers'. Both
# these roles has been added to the 'onelogin_user_roles' parameter in the
# pam_onelogin.yaml

$ > cat /etc/pam-onelogin/.cache_passwd
 jane.doe::12345678:12345678:jane.doe@example-domain.org:/home/jane.doe:/bin/bash
 uncle.tom::87654321:87654321:uncle.tom@example-domain.org:/home/uncle.tom:/bin/bash

$ > cat /etc/pam-onelogin/.cache_group
 jane.doe:x:12345678:
 uncle.tom:x:87654321:
 sysadmins:x:12345:jane.doe,uncle.tom
 developers:x:54321:jane.doe
```
> **Note**<br><br>
> For a detailed description for each option, have a look at [pam_onelogin.yaml](https://github.com/patchon/pam-onelogin/blob/master/pam_onelogin.yaml)

#### Add onelogin to nsswitch and verify user / group lookup
```bash
# Add onelogin to nsswitch
$ > sudo vim /etc/nsswitch.conf
                                  ⤹ Add 'onelogin' to passwd and group
 passwd:     sss files systemd onelogin
 group:      sss files systemd onelogin

# At this point you should be able to verify user lookup
$ > id jane.doe
 uid=12345678(jane.doe) gid=12345678(jane.doe) groups=12345678(jane.doe),12345(sysadmis),54321(developers)

# And same for group lookups
$ > groups jane.doe
 jane.doe : jane.doe sysadmins developers
```

#### Add pam_onelogin.so to SSH
```bash
# Add pam_onelogin.so to the pam-config for SSH
$ > sudo vim /etc/pam.d/sshd
                                           # Add the following row
 auth       sufficient   pam_onelogin.so  ⤶
 auth       substack     password-auth
 auth       include      postlogin
 account    required     pam_sepermit.so
```

> **Note**<br><br>
> Adding this row first in your pam stack, in the authentication section, means
that, if the onelogin-pam-module succeeds, **no other** pam-modules will be
tried. If that is **not** what you want, you have to configure your pam accordingly.

#### Configure SSH and SELinux
```bash
# Enable ChallengeResponseAuthentication for SSH
$ > sudo sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/' /etc/ssh/sshd_config.d/50-redhat.conf

# Allow SSH daemon to connect to your onelogin server (denied default by SELinux)
$ > sudo setsebool -P authlogin_yubikey 1

# Restart SSHd
$ > sudo systemctl restart sshd

# Enable autocreation of home directory,
$ > sudo [ dnf | yum ] install oddjob-mkhomedir
$ > sudo systemctl --now enable oddjobd
$ > sudo vim /etc/pam.d/password-auth

session     optional                                     pam_keyinit.so revoke
session     required                                     pam_limits.so
-session    optional                                     pam_systemd.so
                                                                                  # Add the following row
session     optional                                     pam_oddjob_mkhomedir.so ⤶
session     [success=1 default=ignore]                   pam_succeed_if.so service in crond quiet use_uid
session     required                                     pam_unix.so
session     optional                                     pam_sss.so
```

> **Note**<br><br>
> For CentOS 7 / 8 the SSH configuration file is '/etc/ssh/sshd_config'

At this point, your users should be able to use their OneLogin usernames for
authenticating to your server.

## :warning: Caveats
This is a proof of concept and the author does not take any responsibility for
what may happen if you decide to use it.

A few other warnings,

* I'm no developer, not in C, nor in any other language, hence the code certainly
contains quite a few bugs and might not be the most beautiful thing you've looked
at. Please don't judge.

* The configuration file for pam-onelogin is readable by everyone that has access
to the system.

* ~~*When authentication is being made against OneLogin, both password **and** OTP
is required. One could argue (even though it's not implemented
at the moment) that you never want to enter your OneLogin password anywhere else
than to OneLogin.com website. This would mean that you only would have to enter
the OTP when accessing a server and no password.*~~ Option to disable password
verification has been added.

With that being said, it seems to work as expected in my testing environment.

## :hammer: Todo's
- [ ] Major cleanup (in all regards).
- [ ] More dynamic allocations instead of all those static buffers that are used
everywhere.
- [ ] Make **onelogin-mkcache** a daemon so it can sync users / groups at
configured interval, also create systemd service for it.
- [x] Make an option for not verifying user password and only ask for OTP.
- [ ] Make the '/etc/pam-onelogin/pam_onelogin.yaml' only readable by root and
redesign the **pam_module** so it somehow can get the API credentials, without
being root.
- [ ] Make the 'auto_add_group_to_user'-option configurable so it can accept
different additional groups depending on which role you may have. Today it only
accepts one group, and all users that have **any** of the Onelogin-roles will
get this additional group.
- [ ] Make integration against Active Directory group membership, if your OneLogin is backed by it
- [ ] Package everything into an RPM.

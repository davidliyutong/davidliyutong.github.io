---
layout: post
title: Logging in to Ubuntu with LDAP Accounts via nslcd
date: 2023-10-29 00:01:00
description: Set up libpam-ldapd (nslcd) so that users from an LDAP directory can log in to an Ubuntu machine.
tags: ldap nslcd ubuntu IT
categories: IT
---

If your users already live in an LDAP directory, you can let them log in to an Ubuntu machine with their directory accounts instead of creating local ones. This post uses nss-pam-ldapd: `nslcd` is the daemon that talks to the LDAP server, `libnss-ldapd` makes LDAP users and groups visible to the system through NSS, and `libpam-ldapd` checks their passwords through PAM. Home directories are created automatically on first login.

## Install the packages

```bash
sudo apt-get install libpam-ldapd
```

This also pulls in `nslcd` and `libnss-ldapd`. During installation, debconf asks a few questions, among them the LDAP server URI, the search base, and which name services should use LDAP. You can just press Enter through all of them. The next steps replace `/etc/nslcd.conf` and edit `/etc/nsswitch.conf` by hand, and leaving the name-services list empty means the installer doesn't touch `nsswitch.conf` at all.

## Configure nslcd

`/etc/nslcd.conf` tells nslcd where the directory is, which account to bind with, and how LDAP entries map to Unix users and groups. Here is the configuration I use. Replace the server address, base DN, and bind credentials with your own.

```conf
# /etc/nslcd.conf
# nslcd configuration file. See nslcd.conf(5)
# for details.

# The user and group nslcd should run as.
uid nslcd
gid nslcd

# The location at which the LDAP server(s) should be reachable.
uri ldap://<LDAP_SERVER_IP>:30389

# The search base that will be used for all queries.
base dc=example,dc=com

# The LDAP protocol version to use.
#ldap_version 3

# The DN to bind with for normal lookups.
binddn cn=ldap-reader,ou=users,dc=example,dc=com
bindpw <BIND_PASSWORD>

# The DN used for password modifications by root.
#rootpwmoddn cn=admin,dc=example,dc=com

# SSL options
#ssl off
#tls_reqcert never
tls_cacertfile /etc/ssl/certs/ca-certificates.crt

# The search scope.
#scope sub
filter passwd (objectClass=person)
map    passwd uid           uid
map    passwd homeDirectory "/mnt/homes/$uid"
filter shadow (objectClass=person)
map    shadow uid           uid
filter group  (&(|(objectclass=groupOfUniqueNames)(objectclass=posixGroup))(|(cn=admins)(cn=developers)))
```

Here is what the non-default lines do:

- `binddn` / `bindpw`: a read-only service account that nslcd uses for lookups. The file holds that account's password, so make sure it isn't world-readable.
- `filter passwd` / `filter shadow`: treat every `person` entry as a Unix account, instead of nslcd's default of `posixAccount` entries only.
- `map passwd uid` / `map shadow uid`: take the login name from the `uid` attribute. Use the same attribute in both maps. On Active Directory that attribute is usually `sAMAccountName`.
- `map passwd homeDirectory`: ignore any `homeDirectory` stored in the directory and put every user's home under `/mnt/homes/<username>`.
- `filter group`: expose only the `admins` and `developers` groups (either `groupOfUniqueNames` or `posixGroup` entries) to the system, so the rest of the directory's groups don't show up.

These filters and mappings depend on your directory's schema, so adjust them to match it. Users and groups still need numeric `uidNumber` and `gidNumber` attributes to become usable Unix accounts. If your groups list their members in `uniqueMember`, as `groupOfUniqueNames` entries do, you may also need `map group member uniqueMember`.

A note on transport security: this example uses plain `ldap://` without `ssl start_tls`, so the `tls_cacertfile` line has no effect. The bind password travels unencrypted, and so do users' login passwords, because pam_ldap authenticates a user by binding to the server as that user. Use `ldaps://` or `ssl start_tls` unless the network path is already encrypted, for example by a VPN.

## Enable LDAP in NSS

Add `ldap` as the last source for `passwd`, `group`, and `shadow` in `/etc/nsswitch.conf`. Leave the rest of the file as it is.

```conf
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files ldap
gshadow:        files

hosts:          files mdns4_minimal [NOTFOUND=return] dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis
```

## Create home directories on first login

You don't need to edit the PAM authentication stack yourself. Installing `libpam-ldapd` already enabled its "LDAP Authentication" profile through `pam-auth-update`, which adds `pam_ldap.so` lines to `/etc/pam.d/common-auth`, `common-account`, `common-password`, and `common-session`. What's still missing is a home directory for a user's first login. The cleanest way to add one is the `mkhomedir` profile that ships with Ubuntu's PAM packages:

```bash
sudo pam-auth-update --enable mkhomedir
```

This adds `session optional pam_mkhomedir.so` to `/etc/pam.d/common-session`.

If you want different options, add the line by hand instead. For example, `required` makes the login fail when the home directory can't be created, and you can also set `skel` and `umask` explicitly. Put the line after the `# end of pam-auth-update config` marker. If you add a module inside the managed block, `pam-auth-update` treats the file as locally modified and stops updating it unless you run it with `--force`. With the manual line in place, my `/etc/pam.d/common-session` looks like this:

```conf
#
# /etc/pam.d/common-session - session-related modules common to all services
#
# This file is included from other service-specific PAM config files,
# and should contain a list of modules that define tasks to be performed
# at the start and end of sessions of *any* kind (both interactive and
# non-interactive).
#
# As of pam 1.0.1-6, this file is managed by pam-auth-update by default.
# To take advantage of this, it is recommended that you configure any
# local modules either before or after the default block, and use
# pam-auth-update to manage selection of other modules.  See
# pam-auth-update(8) for details.

# here are the per-package modules (the "Primary" block)
session	[default=1]			pam_permit.so
# here's the fallback if no module succeeds
session	requisite			pam_deny.so
# prime the stack with a positive return value if there isn't one already;
# this avoids us returning an error just because nothing sets a success code
# since the modules above will each just jump around
session	required			pam_permit.so
# The pam_umask module will set the umask according to the system default in
# /etc/login.defs and user settings, solving the problem of different
# umask settings with different shells, display managers, remote sessions etc.
# See "man pam_umask".
session optional			pam_umask.so
# and here are more per-package modules (the "Additional" block)
session	required	pam_unix.so
session	[success=ok default=ignore]	pam_ldap.so minimum_uid=1000
session	optional	pam_systemd.so
# end of pam-auth-update config
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## Restart the services

```bash
sudo systemctl enable nslcd
sudo systemctl restart nslcd
sudo systemctl restart ssh
```

On Ubuntu the OpenSSH unit is called `ssh.service`. `sshd.service` is only an alias, and it doesn't exist on releases where SSH runs through socket activation, so `systemctl restart sshd` can fail there.

## Verify

Check that the system can see an LDAP user and their groups:

```bash
getent passwd <username>
id <username>
```

`getent` should print the user with a home directory under `/mnt/homes`, and `id` should list `admins` or `developers` if the user belongs to either group. After that, log in over SSH as that user. The home directory is created on the first login.

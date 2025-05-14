Linux-CryptHome: Mount an encrypted user's home at login
======================================================================

* SPDX-FileCopyrightText: 2016-2025 SATOH Fumiyasu @ OSSTech Corp., Japan
* SPDX-License-Identifier: GPL-3.0-or-later
* URL: <https://GitHub.com/fumiyas/linux-crypthome>
* Author's home: <https://fumiyas.github.io/>

What's this?
----------------------------------------------------------------------

* At user login:
    * Save an entered valid login password to a keyring.
    * Open a LUKS encrypted volume for the user with the login password.
    * Create `home-<username>.mount` systemd unit if not exists.
    * Enable `home-<username>.mount` systemd unit.
    * Create `crypthome-<username>.service` systemd unit if not exists.
    * Enable `crypthome-<username>.service` systemd unit.
* At user logout:
    * Unmount the user's home directory.
    * Close the LUKS volume.

Requirements
----------------------------------------------------------------------

* Linux environment with LUKS support and:
    * systemd
    * cryptsetup
    * lvm2
* LUKS volumes for each user that are encrypted with user's login password

Usage
----------------------------------------------------------------------

### Install files

```console
$ sudo install -m 0755 crypthome-pam /usr/local/sbin/
```

### Configure PAM

#### Debian / Ubuntu

Add `pam_exec.so` line after `# end of pam-auth-update config` line in
`/etc/pam.d/common-auth` as the following:

```
...snipped...
# and here are more per-package modules (the "Additional" block)
# end of pam-auth-update config
auth optional pam_exec.so expose_authtok /usr/local/sbin/crypthome-pam
```

#### RHEL / CentOS

Add `pam_exec.so` line into `/etc/pam.d/postlogin` as the following:

```
...snipped...
auth optional pam_exec.so expose_authtok /usr/local/sbin/crypthome-pam
...snipped...
```

### Create an encrypted user's home for Linux-CryptHome

  1. Create an LVM volume with named `crypthome.<username>`.
  2. Initializes a LUKS volume in the created LVM volume and sets
     the initial password (passphrase). **NOTE**: The password is
     necessarily the same as the user's login password.
  3. Open the LUKS volume with the password.
  4. Create a filesystem on the opened LUKS volume.
  5. Close the LUKS volume.

```console
# lvcreate -n crypthome.alice -L 10g VolGroup
  Logical volume "crypthome.alice" created.
# cryptsetup luksFormat /dev/VolGroup/crypthome.alice

WARNING!
========
This will overwrite data on /dev/VolGroup/crypthome.alice irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase: ********
Verify passphrase: ********
# cryptsetup luksDump /dev/VolGroup/crypthome.alice
LUKS header information for /dev/VolGroup/crypthome.alice

Version:        1
Cipher name:    aes
Cipher mode:    xts-plain64
Hash spec:      sha256
...snipped...
# cryptsetup open /dev/VolGroup/crypthome.alice decrypthome.alice
Enter passphrase for /dev/VolGroup/crypthome.alice: ********
# mkfs -t xfs /dev/mapper/decrypthome.alice
...snipped...
# mkdir -p -m 0755 ~alice
# mount /dev/mapper/decrypthome.alice ~alice
# cp -a /etc/skel/. ~alice/
# chown -R alice: ~alice
# chmod 0750 ~alice
# umount ~alice
# cryptsetup close decrypthome.alice
```

Limitations
----------------------------------------------------------------------

* Does NOT work on `su - alice`

TODO
----------------------------------------------------------------------

* Support changing password
* More logging
* Suspend and Resume the LUKS volume when screen lock and unlock
* Create a LUKS encrypted volume and a home directory if not exist at login
* How to sesize a LUKS encrypted volume?

References
----------------------------------------------------------------------

* Dm-crypt/Mounting at login - ArchWiki
    * https://wiki.archlinux.org/index.php/Dm-crypt/Mounting_at_login
* Dm-crypt/ログイン時にマウント - ArchWiki
    * https://wiki.archlinuxjp.org/index.php/Dm-crypt/%E3%83%AD%E3%82%B0%E3%82%A4%E3%83%B3%E6%99%82%E3%81%AB%E3%83%9E%E3%82%A6%E3%83%B3%E3%83%88
* Linuxで"luks"を使ってホームフォルダを暗号化する方法 - みくにまるのブログ
    * https://mikunimaru.hatenablog.jp/entry/2021/06/05/083655#google_vignette

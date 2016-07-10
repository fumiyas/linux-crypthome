Linux-CryptHome: Mount an encrypted user's home at login
======================================================================

  * Copyright (c) 2016 SATOH Fumiyasu @ OSS Technology Corp., Japan
  * License: GPLv3
  * URL: <https://GitHub.com/fumiyas/linux-crypthome>
  * Author's home: <https://fumiyas.github.io/>

What's this?
----------------------------------------------------------------------

  * At user login:
    * Save an entered valid login password to a keyring.
    * Open a LUKS encrypted volume for the user with the password
      in the keyring.
    * Revoke the password in the keyring.
    * Mount the opened LUKS volume to the user's home directory.
  * At user logout:
    * Unmount the user's home directory.
    * Close the LUKS volume.

Requirements
----------------------------------------------------------------------

  * Linux environment with LUKS support and:
    * systemd
    * keyutils
    * cryptsetup
    * lvm2
  * LUKS volumes for each user that are encrypted with user's login password

Usage
----------------------------------------------------------------------

### Install files

```console
$ sudo install -m 0755 crypthome-{pam,mount,umount} /usr/local/sbin/
$ sudo install -m 0644 crypthome@.service /etc/systemd/system/
$ sudo systemctl daemon-reload
```

### Configure PAM

#### Debian

Add `pam_exec.so` line after `pam_cap.so` line in `/etc/pam.d/common-auth`
as the following:

```
...
auth	optional			pam_cap.so
auth	optional			pam_exec.so expose_authtok /usr/local/sbin/crypthome-pam
...
```

#### RHEL, CentOS

Replace all `auth` type lines in `/etc/pam.d/system-auth` and/or
`/etc/pam.d/password-auth` as the following:

```
...
auth	required			pam_env.so
auth	[success=1 default=ignore]	pam_unix.so nullok try_first_pass
auth	requisite			pam_deny.so
auth	required			pam_permit.so
auth	optional			pam_exec.so expose_authtok /usr/local/sbin/crypthome-pam
...
```

If you are using `pam_sss.so` or another module for authentication:

```
...
auth	required			pam_env.so
auth	[success=2 default=ignore]	pam_unix.so nullok try_first_pass
auth	[success=1 default=ignore]	pam_sss.so use_first_pass
auth	requisite			pam_deny.so
auth	required			pam_permit.so
auth	optional			pam_exec.so expose_authtok /usr/local/sbin/crypthome-pam
...
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
Payload offset: 4096
MK bits:        256
MK XigXst:      XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX
MK sXlt:        XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX
                XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX
MK itXrXtions:  XXXXXX
UUIX:           XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

Key Slot 0: ENABLED
        ItXrXtions:             XXXXXXX
        SXlt:                   XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX
                                XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX
        Key material offset:    8
        AF stripes:             4000
Key Slot 1: DISABLED
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
# cryptsetup open /dev/VolGroup/crypthome.alice decrypthome.alice
Enter passphrase for /dev/VolGroup/crypthome.alice: ********
# mkfs -t ext4 /dev/mapper/decrypthome.alice
...
# cryptsetup close decrypthome.alice
```

TODO
----------------------------------------------------------------------

  * Support changing password
  * More logging
  * Suspend and Resume the LUKS volume when screen lock and unlock
  * Create a LUKS encrypted volume and a home directory if not exist at login

References
----------------------------------------------------------------------

  * Dm-crypt/Mounting at login - ArchWiki
    * https://wiki.archlinux.org/index.php/Dm-crypt/Mounting_at_login


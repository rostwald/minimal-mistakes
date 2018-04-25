# Migrating to GPG keys stored on a yubikey and using pass for password storage

## Preface

Over the years I've scattered passwords, GPG- and SSH-keys over varios systems. Passwords were usually stored inside the browser built-in managers, some of them were reused on multiple accounts on varios sites because the password managers on different systems were always out of sync and I still had to remember a lot of the passwords. SSH- and GPG-keys were shared by scp'ing keys and/or keyrings between systems and because many years ago I used Thunderbird with enigmail, which kindly offered to generate new key pairs for every mail account, at some point I had to manage 7 or 8 GPG key pairs. I had SSH keys for essentially every system and ssh account, so the "authorized_keys" files on most systems grew into a multiple-dozend line beast at some time and were always completely out of sync.
In short: It was a huge, unmaintainable mess.


Sometime in February, when reading the "morning news" I stumbled over this headline: [Pass: The Standard Unix Password Manager](https://news.ycombinator.com/item?id=14819136)  
I read around half of the introduction and was already hooked. As it was readily available from the FreeBSD pkg repository I installed it, played around for a few minutes and after discovering there was a [Firefox plugin](https://github.com/jvenant/passff#readme) I completely fell in love.  
My first attempt was to use my various GPG keys for different subfolders within the password store: GPG-keys from my work email for work related passwords; the private keys for passwords/accounts created with the matching mail address etc.. At the end I had a pretty neat layout and everything was nicely separated - but I had to type in the various passwords for all of these GPG keys over and over; the whole day; again and again... After a few days I had enough and went back to the drawing board.

I remembered people were talking about yubikeys in the HN comments, so I did some more research and ordered 2 Yubikey NEO (in hindsight I should have gone with a YK4 and a nano as I still haven't really used the NFC capabilities). When I received the keys, I've spent 2 nights testing, fiddling around and quite some cursing, but I learned _a lot_ from it. This Blog entry is basically a braindump from my experiences and how I set up what I am using now.


## Bits and pieces

Searching for "yubikey", "gpg", "password-store" and various other related keywords brings up tons of results of varying quality and relevance, but I got a pretty good impression on what was possible and how to accomplish it. Combined with an attitude of "doing it right" this time from the early beginning, I came up with these basic requirements:

1. share all my login credentials between the systems I regularly use - in a secure manner
2. use multiple factors to protect this data
3. access/use this data without re-typing a password over and over again during a session (let alone multiple passwords)
4. replace the multitude of GPG- and SSH-keys if possible


Reading through manpages, documentations and tons of blog posts, mailing list and forum entries on the various topics included in this pursuit, it became more and more clear what could be used for each specific mechanism in the whole chain.

1. 
   - pass uses git; sharing of the password storage therefore is a no-brainer
   - pass uses GPG to encrypt the entries; so sharing (and storing) the data is secure
2.
   - yubikeys can be used as a smartcard which holds GPG-keys
   - needs a password to unlock each individual key
3.
   - using a single set of (sub)keys on the yubikey, I only need to remember the PIN for the yubikey
   - gpg-agent, pinentry and pcscd handle the keys, pin caching and smartcard access
4.
   - signing a new GPG key with all the old ones creates a chain of trust; enabling me to transition to a single key/ID
   - gpg-agent handles SSH keys and enables you to use GPG-keys for SSH authentication



## The big picture

Putting all the parts together, the whole setup still looks rather simple with well-defined tasks for each tool:

- yubikey for holding a set of GPG-keys
- gpg-agent/pinentry/pcscd to actually access/use these keys
- pass for storing and sharing login/password data
- passff to make logins/passwords available in firefox
- gpg-agents ssh-support to use the GPG-key with SSH


The (very) simplified workflow:

- Create yet another (final) GPG-"master" key, that is _only_ used for generating sub-keys and sign that key with all of my old keys that are still valid
- Generate the subkeys for encryption, signing and authentication, put them on a yubikey and upload the publik key
- encrypt the password-store for this key-ID and put the git repo on a server I can reach from every machine (keybase is great for this)
- register the keys on the yubikey with gpg-agent on all my machines and enable the smartcard service (pcscd)
- enable gpg-agents ssh-support and update/replace the authorized_hosts files on all my systems one last time

## Set up all the things

### System Prerequisites

For the smartcard part there are some additional packages required on FreeBSD, namely `pcsc(-lite), opensc, libccid`. If not already installed, `gnupg2` is also needed as well some suitable GUI-frontend for pinentry like e.g. `pinentry-qt5`.

### Generating the new gpg keys

As we are using the master key only to generate/sign new subkeys, It's a good idea to do these steps within a dedicated Jail or VM and destroy it afterwards to not leave any remains of this key in any keyring or cache. The master key backup should be kept on offline storage like an USB drive stored somewhere safe. As these USB flash disks tend to die if you need them, it is also a good idea to make a backup on dead trees, e.g. with `paperkey`.  

#### Master key

Because I can't remember every single gpg option from the top of my head, I just use the interactive process for generating the keys:

```shell
% gpg2 --full-generate-key
gpg (GnuPG) 2.2.4; Copyright (C) 2017 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: example
Email address: ex@mp.le
Comment:
You selected this USER-ID:
    "example <ex@mp.le>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: /home/sko/.gnupg/trustdb.gpg: trustdb created
gpg: key EA0F1EEB5C0470BB marked as ultimately trusted
gpg: directory '/home/sko/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/home/sko/.gnupg/openpgp-revocs.d/FEEE228F404A0C446DE5E918EA0F1EEB5C0470BB.rev'
public and secret key created and signed.

Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
pub   rsa4096 2018-04-25 [SC]
      FEEE228F404A0C446DE5E918EA0F1EEB5C0470BB
uid                      example <ex@mp.le>
```
The options chosen are "4 RSA (sign only)", a keysize of 4096 bits, no expiry date (I will manage the expiry dates on my subkeys) and finally some personal info + password to protect this key.  

### Subkeys

Yubikeys based on earlier hardware versions than 4 (like the NEO I am using), can only store 2048 bit keys. Maybe upon first expiry of the subkeys I'll upgrade to a newer key and use 4096 keys then...  

#### Signing Key

```shell
% gpg2 --expert --edit-key FEEE228F404A0C446DE5E918EA0F1EEB5C0470BB
gpg (GnuPG) 2.2.4; Copyright (C) 2017 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC
     trust: ultimate      validity: ultimate
[ultimate] (1). example <ex@mp.le>

gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Thu Apr 25 14:59:18 2019 CEST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/E36CA9675F6E096A
     created: 2018-04-25  expires: 2019-04-25  usage: S   
[ultimate] (1). example <ex@mp.le>

gpg>
```

I chose "(4) RSA (sign only)", 2048 bits key size (due to restrictions of older yubikey) and expiration in 1 year

#### Encryption Key

```shell
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 6
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Thu Apr 25 15:03:26 2019 CEST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/E36CA9675F6E096A
     created: 2018-04-25  expires: 2019-04-25  usage: S   
ssb  rsa2048/552B3CC161134ABD
     created: 2018-04-25  expires: 2019-04-25  usage: E   
[ultimate] (1). example <ex@mp.le>

gpg>
```

#### Authentication Key

```shell
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Sign Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? s

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished
Your selection? e

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? a

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Authenticate 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Thu Apr 25 15:08:33 2019 CEST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/E36CA9675F6E096A
     created: 2018-04-25  expires: 2019-04-25  usage: S   
ssb  rsa2048/552B3CC161134ABD
     created: 2018-04-25  expires: 2019-04-25  usage: E   
ssb  rsa2048/A19F885AD690E214
     created: 2018-04-25  expires: 2019-04-25  usage: A   
[ultimate] (1). example <ex@mp.le>

gpg> save
```

GnuPG doesn't have a "RSA (authenticate only)" template for key creation, so I created it by using the template "(8) RSA (set your own capabilities" and toggling the capabilities of the key to end up with a key that only has the authentication capability enabled.  
There are now 3 subkeys present, each of them only intended for a single usage: Sign, Encrypt or Authenticate. If everything looks fine, save these changes.


### Export and back up all keys

After moving the subkeys to a smartcard, they can't be extracted afterwards. I wanted to keep some backups e.g. if the yubikey should break, so I put the private keys and revocation files on a USB key and created an additional paper backup:

```shell
gpg2 --export-secret-key EA0F1EEB5C0470BB | paperkey | lpr
```

## Preparing the yubikey

All yubikey NEOs shipped after 2015 and with Firmware >3.1 already have all modes enabled; older Versions might need an update of the USB mode via the `ykpersonalize` tool.

### Configuring the Smartcard

Smartcards are configured using the gpg toolset:

```shell

% gpg2 --card-edit

Reader ...........: 1050:0116:X:0
Application ID ...: D2760001240102000006012345670000
Version ..........: 2.0
Manufacturer .....: Yubico
Serial number ....: 01234567
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none] 

gpg/card>
```

### Set passwords/PINs

The Yubikeys have 2 PINs: the "everyday PIN" and an Admin PIN. The latter one is required for modifying card settings, to reset retry counters if the wrong PIN has been entered more than 3 times or to set a new PIN. The defaults are usually "12345678" for the Admin PIN and "123456" for the PIN. Although they are called PIN, all ASCII characters are allowed.

```shell
gpg/card> admin
Admin commands are allowed

gpg/card> passwd
gpg: OpenPGP card no D2760001240102000006012345670000 detected


1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q
```

### Set some Informations / Options

I've set up the "Name of cardholder" and "Signature PIN" on my yubikey. Enforcing the Signature PIN requires me to always enter the PIN if I want to sign e.g. an email or git commit. This is less convenient, but I'd like to have full control over what I sign. Also this more resembles the meaning of a signature where I actually have to take action and write down my signature.

### Transfer the keys

As already said - transferring the keys to a smartcard is a one-way operation, so I checked my backup keys once more and then pushed the keys to the yubikey:

```shell
% gpg2 --edit-key EA0F1EEB5C0470BB
gpg (GnuPG) 2.2.4; Copyright (C) 2017 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/E36CA9675F6E096A
     created: 2018-04-25  expires: 2019-04-25  usage: S   
ssb  rsa2048/552B3CC161134ABD
     created: 2018-04-25  expires: 2019-04-25  usage: E   
ssb  rsa2048/A19F885AD690E214
     created: 2018-04-25  expires: 2019-04-25  usage: A   
[ultimate] (1). example <ex@mp.le>

gpg>key 1

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC  
     trust: ultimate      validity: ultimate
ssb* rsa2048/E36CA9675F6E096A
     created: 2018-04-25  expires: 2019-04-25  usage: S   
ssb  rsa2048/552B3CC161134ABD
     created: 2018-04-25  expires: 2019-04-25  usage: E   
ssb  rsa2048/A19F885AD690E214
     created: 2018-04-25  expires: 2019-04-25  usage: A   
[ultimate] (1). example <ex@mp.le>

gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

You need a passphrase to unlock the secret key for
user: "example <ex@mp.le>"
2048-bit RSA key, ID 0xE36CA9675F6E096A, created 2018-04-25

gpg> key 1
[...]
gpg> key 2

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/E36CA9675F6E096A
     created: 2018-04-25  expires: 2019-04-25  usage: S   
ssb* rsa2048/552B3CC161134ABD
     created: 2018-04-25  expires: 2019-04-25  usage: E   
ssb  rsa2048/A19F885AD690E214
     created: 2018-04-25  expires: 2019-04-25  usage: A   
[ultimate] (1). example <ex@mp.le>

gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2
[...]
gpg> key 2
[...]
gpg> key 3

sec  rsa4096/EA0F1EEB5C0470BB
     created: 2018-04-25  expires: never       usage: SC  
     trust: ultimate      validity: ultimate
ssb  rsa2048/E36CA9675F6E096A
     created: 2018-04-25  expires: 2019-04-25  usage: S   
ssb  rsa2048/552B3CC161134ABD
     created: 2018-04-25  expires: 2019-04-25  usage: E   
ssb* rsa2048/A19F885AD690E214
     created: 2018-04-25  expires: 2019-04-25  usage: A   
[ultimate] (1). example <ex@mp.le>

gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3
[...]

gpg> save
```

To check if everything went as expected, we list the secret keys:

```shell
% gpg2 --list-secret-keys
/home/sko/.gnupg/pubring.kbx
----------------------------
sec   rsa4096 2018-04-25 [SC]
      FEEE228F404A0C446DE5E918EA0F1EEB5C0470BB
uid           [ultimate] example <ex@mp.le>
ssb>  rsa2048 2018-04-25 [S] [expires: 2019-04-25]
ssb>  rsa2048 2018-04-25 [E] [expires: 2019-04-25]
ssb>  rsa2048 2018-04-25 [A] [expires: 2019-04-25]
```

The ">" indicates the subkeys are stub-keys which aren't physically stored on this system but point to the key in the smartcard.

#### Cleaning up the GPG key setup

After everything went well, it is time to make the public key should be made available e.g. via public keyservers via `gpg2 --send-key KEY_ID`
Now we can clean up and destroy the jail/zone/vm we used, removing any "online" trace of our master key. If we have to make changes later on like adding more subkeys or renewing the existing subkeys, we will need the master key, so make sure the backup is working.


### signing the new keys

To Build an initial chain of trust for the new Key, I imported the public key to a keyring where I already had all my old, still active gpg keys registered, signed it and all its identities with each of those keys and re-published the public key to the keyservers. 






## The bitter pill

As nice as the end result might sound/be; there are some rocks to climb:


We need yet another GPG ID, especially if all previously used/created ones were directly used for signing/encryption. The new key will be _exclusively_ used for creating/signing subkeys. This is absolutely essential if we want to regain access to the data we encrypted if the yubikey should get lost/stolen/broken and revoking the keys stored on it. This "master key" therefore becomes our crown-jewel we must protect and preserve with our life.
We also need to tell everbody we communicated with using our old keys that we are now using this new ID and get this new ID signed to build a new web of trust. Signing the new key with our old ones already builds an inital chain of trust - but self-signing isn't and shouldn't be considered a base of (high) trust in a key. Especially this transition phase might be cumbersome and tedious if we have keys that were already signed and trusted by a lot of people.  
[keybase.io](https://keybase.io) is a nice way of building a web of trust, also the kbfs is absolutely great for sharing the password store repository (or other private data) between systems.

Updating the authorized_hosts on all hosts we have access to can be quite a challange. If I had to count all my accounts in all hosts, jails, zones etc pp I might end up with a triple-digit number. Luckily a lot of them are managed via ansible, which also distributes the ssh-fingerprints, so for ~60-70% of the hosts this is a no-brainer. The remaining ~30% is a completely different story I still have nightmares about.
The most practical solution I've found is to leave the old SSH keys in place (if they were password protected!), so you will be prompted for the password of the SSH key if the server isn't recognizing your new key. The problem I had with this is the ssh-agent fighting with gpg-agent and if ssh-agent won I got prompted for the SSH-key password even if the server alredy knew about my new key. I also noticed a relatively long delay for ssh connections if the new key didn't work and the old keys needed to be probed.  








_Thanks_  
_sko_

----

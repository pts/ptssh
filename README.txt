ptssh: portable SSH client for Unix
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ptssh is and SSH client (with support for ssh, scp, sftp and rsync) for
Unix with a focus on portability and single-file configs: it has a single
config file (and it ignores ~/.ssh/*), and it works equivalently on any
Unix system and any OpenSSH version. After configuration, it can be used
as a fallback to connect if ssh(1) stops working because of a software
upgrade or a configuration change. It can also be used on public computers
or loaners where reading and writing OpenSSH config files is not desired.
ptssh is implemented as a Bourne shell script calling OpenSSH ssh(1) with
custom options. The config file can be embedded to the ptssh script and
the merged standalone script can be copied to a pen drive and used on
another computer (running Unix and having OpenSSH installed).

To start using ptssh, you need to create the config file ~/.ptssh first.
This config file contains the user identity (private key), and for each
server the server hostkey, server connection information (hostname, port,
username) and server X11 forwarding config. The easiest way to create a
config file is importing from ssh(1), e.g. `./ptssh import myserver1' and
`./ptssh import myuser@myserver2'. It possible to import multiple times
if the user identity (private key) is the same; in this case subsequent
imported server specifications will be appended to the config file
~/.ptssh. A sample config file is also provided in the file
`config.ptssh.sample'. (See details below.)

Design goals of ptssh (all met):

* It should work with old and new versions of OpenSSH. More specifically,
  OpenSSH 3.9 (released on 2004-08-18) or later, and
  Debian Etch (released on 2007-04-08) or later should work.
* It should work with many Bourne shells.
* The ptssh-keycat tool should work with old and new versions of Perl 5.
  More specifically, Perl 5.6.1 (released on 2001-04-08) or later should
  work.
* It should ignore OpenSSH system config files in /etc/ssh/* .
* It should ignore OpenSSH user config files in ~/.ssh/* . The reason for
  this is portability to other systems: by copying the ptssh shell script
  and the config file ~/.ptssh, it should work equivalently on the target
  system.
* It should ignore the ssh-agent(1). This is also part of portability.
* It should be easy to embed the config into the ptssh script, thus
  copying one file to the target system should be enough.
* It shouldn't attempt to run any command (other than ssh, scp, sftp, rsync
  and chmod) which isn't a shell builtin.

It's possible to embed the config into the ptssh script by just
concatenating them:

  $ cat ptssh ~/.ptssh >ptssh_merged
  $ chmod +x ptssh_merged
  $ ./ptssh_merged myserver1  # Example call.

However, it's better to use ptssh-keycat for concatenation, because it
also does the necessary key conversion. Native OpenSSH private key format
(-----BEGIN OPENSSH PRIVATE KEY-----) only works if the key is at the
beginning of the file (ssh(1) error message: `Load key "...": invalid
format'), and ptssh-keycat converts them to an OpenSSL-compatible format,
which works anywhere. Do it like this:

  $ ./ptssh-keycat -x ptssh_merged ptssh ~/.ptssh
  $ ./ptssh_merged myserver1  # Example call.

Then `./ptssh_merged ...' won't read ~/.ptssh, but it would use the
embedded user identity and server specifications.

A limitation of embedding: the user identity must be an RSA (ssh-rsa,
recommended with 4096 bits) or DSA (ssh-dss) private key in an
OpenSSL-compatible format. Other key algorithms (e.g. ssh-ed25519) won't
work for embedding, because OpenSSH can't read them in an
OpenSSL-compatible format.

Run this to generate a new passphrase-protected RSA private key (user
identity), and create and embedded ptssh using the new key:

   $ (openssl genrsa 4096; grep '^ptssh-hostkey-' ~/.ptssh) |
     ./ptssh-keycat --protect -x ptssh_merged

The equivalent commands without ptssh-keycat:

   $ openssl genrsa 4096 |
     openssl pkcs8 -out embed_id_rsa -topk8 -v2 aes-256-cbc -v2prf hmacWithSHA1 -iter 4200000
   $ (cat ptssh embed_id_rsa; grep '^ptssh-hostkey-' ~/.ptssh) >ptssh_merged
   $ rm -f emed_id_rsa
   $ chmod +x ./ptssh_merged

Run this to enable the newly created private key on the server:

   $ (PK="$(ssh-keygen -y -f ptssh_merged)"; echo "$PK" |
      ssh myserver1 'test -d .ssh || mkdir .ssh; cat >>.ssh/authorized_keys')

ptssh-keycat is the most convenient way to convert OpenSSH private keys to
an OpenSSL-compatible format:

  $ ./ptssh-keycat -x ptssh_merged ptssh ~/.ptssh

ptssh-keycat is a Perl script which implements most of the conversion
steps natively in Perl, and it uses ssh-keygen(1) and openssl(1) for
passphrase-protection.

You can also do the conversion without ptssh-keycat, like this:

   $ puttygen -P -O private-openssh -o embed.tmp ~/.ptssh
   Enter passphrase to load key:
   Enter passphrase to save key:    (specify empty passphrase)
   Re-enter passphrase to verify:   (specify empty passphrase)
   $ openssl pkcs8 -out embed.pem -in embed.tmp -topk8 -v2 aes-256-cbc -v2prf hmacWithSHA1 -iter 4200000
   Enter Encryption Password:
   Verifying - Enter Encryption Password:
   $ rm -f embed.tmp
   $ (cat ptssh embed.pem; grep '^ptssh-hostkey-' ~/.ptssh) >ptssh_merged
   $ chmod +x ./ptssh_merged

Please note that neither openssl(1) nor ssh-keygen(1) are able to do the
conversion above, puttygen(1) or ptssh-keycat is also needed. On Debian,
install puttygen(1) with `sudo apt-get install putty-tools'. Probably it's
more convenient to use ptssh-keycat, because it works without installation.

Please note that it's possible to have password-protected RSA and DSA
private keys in the traditional formats ( ----BEGIN RSA PRIVATE KEY-----;
and ----BEGIN DSA PRIVATE KEY-----), but those are insecure (weak),
because they generate the decryption key by doing a single MD5 digest
call, and thus the attacker can try (brute-force) millions of passphrases
per second. The alternative above (`openssl pkcs8
... -iter 4200000') uses PBKDF2 with a sufficiently large iteration count
(equivalent to `ssh-keygen -a 400') to protect against brute-force
attacks. The output private key is compatible with OpenSSH >=3.8 (or even
earlier).

ptssh is very portable: it uses RSA hostkeys by default, and it works with
OpenSSH >=3.8 (checked with OpenSSH 8.2, OpenSSH 7.4, OpenSSH 3.8.1, and
OpenSSH_4.3p2 Debian-9etch3, OpenSSL 0.9.8c 05 Sep 2006 on Debian Etch,
OpenSSL 0.9.7e on Debian Sarge), no matter what the system defaults and
user defaults for the SSH client are. It also works with various Bourne
shells: bash, zsh, dash, pdfksh, busybox sh (ash), posh.

ptssh is compatible with OpenSSH >=3.8 client (released 2004-02-34), but
it's recommended to upgrade to OpenSSH >=3.9 client (released 2004-08-18),
because recent OpenSSH server versions don't support any kex (key
exchange) algorithm compatible with the OpenSSH 3.8 client, with the
client failing with error `no kex alg'. ptssh is also compatible
with newer OpenSSH clients, e.g. 8.2 (released 2020-02-14).

ptssh ignores the ssh-agent, and asks for the identity file passphrase
each time. This is on purpose. The goal is to make to ptssh run
independently and identically on any system and environment (given the
same config file), and the use of ssh-agent would introduce variance.

ptssh is also compatible with Debian Sarge (2005-06-06) or later, but
recent OpenSSH server versions don't support any kex algorithm compatible
with the OpenSSH 3.8.1 client in Debian Sarge, so it's recommended to
upgrade to Debian Etch (2007-04-08) instead, which has OpenSSH 4.3.

ptssh is compatible with rsync >=2.6.4 client in Debian Sarge
(2005-06-06). Maybe it works with even older versions of rsync.

ptssh can't run from a FAT filesystem (or other non-Unix filesystems),
because `chmod' below isn't able to remove rwx permissions for group and
other there. This can be a problem if it's run from USB pen drives.

The format of the config file ~/.ptssh is the following:

* (Most users don't need to care about these details, because they have
  created the config file with `./ptssh import ...'.)
* A line of the form `-----BEGIN ... PRIVATE KEY-----' (including
  `-----BEGIN PRIVATE KEY-----') indicating the
  beginning of the user identity (private key). Any format (`...')
  supported by OpenSSH works. It can be protected (encrypted with a
  passphrase) or unprotected. Any key algorithm supported by OpenSSH (e.g.
  ssh-rsa, ssh-dss, ssh-ed25519) works.
* Optional `Key: value' lines if it's an encrypted traditional OpenSSL RSA
  or DSA private key. Example key: `Proc-Type' and `DEK-Info'.
* Base64-encoded private key material.
* A line of the form `-----END ... PRIVATE KEY-----'.
* Comment lines and server specification lines. At least 1 server
  specification line. Each line which doesn't start with `ptssh-hostkey-'
  is a comment line.
* A server specification line is a whitespace-separated (can be space, tab
  or combination) of these items:
  * `ptssh-hostkey-ID'. `ID' is an arbitrary unique ID within the file.
  * hostkey algorithm. Examples: ssh-rsa, ssh-dss, ssh-ed25519,
    ecdsa-sha2-nistp521, ecdsa-sha2-nistp384, ecdsa-sha2-nistp256.
  * hostkey public key, Base64-encoded. Starts with `AAAA'.
  * server username.
  * server port number. The default, 22, also has to be specified.
  * server hostname (or IP address).
  * X11 forwaring specification. One of: -x (disabled), -X (enabled,
    untrusted), -Y (enabled, trusted). If you don't want to run GUI programs
    on the server over the SSH connection, specify `-x'.
  * host aliases. Comma-separated list of aliases (`Host' lines in
    ~/.ssh/config) ptssh should recognize in the command-line. By default,
    there is only 1 alias: `ID' above.

The config file format was designed with the following requirements:

* It is a structured text file which is easy to generate and parse
  from shell scripts.
* It is easy to edit manually, including the copy-pasting of keys
  from other files. However, manual config file editing is only an expert
  use case: for most uses the config file created by tools ptssh and
  ptssh-keycat should be sufficient.
* It can be used for `ssh -o IdentityFile=...' (and `ssh -i ...') without
  modification, thus it starts with a private key understood by
  OpenSSH, thus starting with `-----BEGIN ... PRIVATE KEY-----'.
  Please note that if the config file is embedded within the ptssh script,
  then the script (starting with `#! /bin/sh --') comes first. This is
  not a problem, because for OpenSSL-compatible private key formats
  (i.e. anything other than ``-----BEGIN OPENSSH PRIVATE KEY-----'),
  OpenSSH accepts a private key starting anywhere in the file.
* It can be used for `ssh -o UserKnownHostsFile=...' without modification.
  This isn't an important restriction, because OpenSSH ignores lines
  not starting with with `...' specified in `ssh -o HostKeyAlias=...'.

__END__

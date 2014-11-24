GPG/PGP encrypted email alias service
========

This repository contains source codes and how-to installation guide for email alias service with GPG/PGP encryption.
p
Purpose of this solution is to encrypt incoming emails by PGP/GPG key and forward them to the final destination. The high level logic is following:

![](https://github.com/sl4dy/gpgalias/blob/master/samples/simple_flow.png)

Solution is using Ubuntu 12.04 and Postfix, Dovecot, MySQL, Amavis, Clam AntiVirus, SpamAssassin, Postgrey, Roundcube and Postfix Admin and [gpgit](https://github.com/mikecardwell/gpgit). All this software is installed from standard Ubuntu 12.04 repositories, unless mentioned otherwise.

## Detailed flow
![](https://github.com/sl4dy/gpgalias/blob/master/samples/detailed_flow.png)

## Installation
Use [this](https://www.exratione.com/2012/05/a-mailserver-on-ubuntu-1204-postfix-dovecot-mysql/) guide to install mail server based on software mentioned above. This will be the starting point for additional tweaking. If you decide to install web mail interface, Roundcube is recommended for it's simplicity.


## Tweaking
This sections mentions all files which need to be altered to make the solution working like mentioned on the pictures above. All files mentioned in this guide are available in the repository. All actions are performed as root unless mentioned otherwise. All files mentioned in this guide are available in the repository for reference.

#### Create user gpgmap

```
adduser --home /var/gpg gpgmap
mkdir -p /var/gpg/.gnupg
chown -R gpgmap /var/gpg
chmod 700 /var/gpg/.gnupg
```

#### Install gpgit and it's dependencies:
Get the gpgit and place it to required folder with required name.
```
cd /usr/local/bin
wget https://raw.githubusercontent.com/mikecardwell/gpgit/master/gpgit.pl
mv gpgit.pl gpgencmail.pl
```
Install required perl modules.
```
#cpan install MIME::Tools
#cpan install Mail::GnuPG
```

#### Create procmail recipe
Create **/etc/postfix/procmailrc.common** file and add following recipe. This recipe is used to pass the email thorough gpgit tool which will encrypt it by GPG key. 

```
TO=`egrep "^T[oO]:.*@gpgalias.com.*|for.*@gpgalias.com.*" | perl -wne'while(/[\w\.]+@[\w\.]+\w+/g){print "$&\n"}' | head -1`
:0 f
  |/usr/local/bin/gpgencmail.pl --encrypt-mode prefer-inline $TO | /usr/sbin/sendmail -G -i $RECIPIENT
```

#### Adjust master.cf file

Edit **/etc/postfix/master.cf** so mail the postfix passes the email to gpgfilter after receiving it back from amavis.

```
127.0.0.1:10025 inet    n       -       -       -       -       smtpd
  -o content_filter=gpgfilter # THIS IS THE REQUIRED CHANGE
```

Add following content to **/etc/postfix/master.cf** file:
```
gpgfilter    unix  -       n       n       -       10      pipe
    flags=Rq user=gpgmap:gpgmap null_sender=
    argv=/usr/bin/procmail RECIPIENT=$(recipient) /etc/postfix/procmailrc.common

procmail unix - n n - - pipe
  -o flags=RO user=vmail:mail argv=/usr/bin/procmail -t -m USER=${user} EXTENSION=${extension} RECIPIENT=$(recipient) /etc/postfix/procmailrc.common
```

The gpgfilter channel receives the email from postfix and passed it to procmail channel which passes it through **/etc/postfix/procmailrc.common**.
 

#### Disable postgrey
Disable postgrey as it is not desirable to delay any emails, destinations for the aliases are chosen by users. In **/etc/postfix/main.cf** comment out following line:

```
# "check_policy_service inet:127.0.0.1:10023" enables Postgrey.
```

#### Enable envelope rewriting
Enable envelope rewriting so MAIL FROM field always contains your domain when sending the email to it's final destination. In **/etc/postfix/main.cf** set following:

```
sender_canonical_classes = envelope_sender
sender_canonical_maps = regexp:/etc/postfix/sender_canonical
```

And create file **/etc/postfix/sender_canonical** with following content:

```
/^/ delivery@mail.example.com
```

This is required, otherwise SPF will be failing as mail.example.com is not permitted to send emails as MAIL FROM: **ann@here.com**; refering to the first picture.

#### Adjust header checks

Adjust the **/etc/postfix/header_checks** and keep the Received header there, it is required to keep the original recipient xxx@gpgalias.com address.

```
#/^Received:/                 IGNORE
/^User-Agent:/               IGNORE
/^X-Mailer:/                 IGNORE
/^X-Originating-IP:/         IGNORE
/^x-cr-[a-z]*:/              IGNORE
/^Thread-Index:/             IGNORE
```

#### Adjust amavisd config:

Edit **/etc/amavis/conf.d/50-user** so it contains following lines:

```
use strict;
$max_servers  = 3;
$sa_tag_level_deflt  = -9999;
$spam_quarantine_to  = undef;
$final_spam_destiny  = D_PASS;
$local_domains_re = new_RE( qr'.+@.+'i );
```
#### Configure postfix to deliver local mail
Edit **/etc/postfix/main.cf**:

```
smtp_host_lookup = dns, native
mydestination = mail, localhost
```

## Provisioning
It is required to provision email aliases, destinations of these aliases and GPG keys for these aliases. This can be done in gpg command line and Postfix Admin web interface. All GPG keys must be generated (or possibly imported) for the alias address and not for the destination address (e.g. the GPG key shall be generated for you@gpgalias.com and NOT for you@finaldestination.com).

### Command line / Postfix Admin provisioning
All GPG operations are performed as gpgmap user.

#### List all GPG keys
```
/usr/bin/gpg --list-keys
```
#### Import GPG key
```
/usr/bin/gpg --import /path/to/gpgkey
```

It is important to edit the imported key and set the trust to ultimate, otherwise gpgit will not work properly.
```
/usr/bin/gpg --edit-key alias@gpgalias.com
```
Type in "trust" and select "5 = I trust ultimately".

#### Generate GPG key
```
/usr/bin/gpg --genkey
```
**WARNING**: This process requires enough of random entropy. To easily fulfil this condition install RNG utils:

```
sudo apt-get install rng-tools
```

And run the generator; in separate terminal or as backround process:
```
sudo rngd -r /dev/urandom
```

#### Delete GPG key
```
/usr/bin/gpg --delete-key alias@gpgalias.com
```

The aliases can be created by using Postfix Admin CLI or using Postfix Admin web interface.

#### Create alias via Postfix Admin CLI

```
/var/www/postfixadmin/scripts/postfixadmin-cli alias add alias@gpgalias.com --goto final@destination.com
```

#### Delete alias via Postfix Admin CLI

```
/var/www/postfixadmin/scripts/postfixadmin-cli alias delete alias@gpgalias.com
```

## Sample encrypted email
You can see sample encrypted email delivered to its destination [here](https://github.com/sl4dy/gpgalias/blob/master/samples/encrypted_mail.txt).

## Virtualbox VDI image
You can download VDI image with working solution in samples directory. Just import the VDI to Virtualbox. Two network interfaces are used - one with Virtualbox only network and one with NAT to access the Internet. The memory requirement is 1 GB, but 512 MB should work also.

NOTE: This image includes the custom provisioning frontend which is not mentioned in this guide.

**Credentials and other useful info:**

Domain: example.com

Hostname: mail.example.com

Services:

192.168.56.101/frontend

192.168.56.101/postfixadmin

192.168.56.101/roundcube

UNIX credentials:

project/project
root/project

Postfixadmin credentials:

postfixadmin@example.com / project

Mysql root password:

project

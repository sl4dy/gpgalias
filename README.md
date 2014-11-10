GPG/PGP encrypted email alias service
========

This repository contains source codes and how-to installation guide for email alias service with GPG/PGP encryption.

Purpose of this solution is to encrypt incoming emails by PGP/GPG key and forward them to the final destination. The high level logic is following:
c
<img style="float: right" src="pic">

Solution is using Ubuntu 12.04 and Postfix, Dovecot, MySQL, Amavis, Clam AntiVirus, SpamAssassin, Postgrey, Roundcube and Postfix Admin and [gpgit](https://github.com/mikecardwell/gpgit). All this software is installed from standard Ubuntu 12.04 repositories, unless said otherwise.

## Detailed flow
<img style="float: right" src="pic">

## Installation
Use [this](https://www.exratione.com/2012/05/a-mailserver-on-ubuntu-1204-postfix-dovecot-mysql/) guide to install mailserver based on software mentioned above. This will be the starting point for additional tweaking. If you decide to isntall webmail interface Roundcube is recommended for it's simplicity.

## Tweaking

#### Create user gpgmap

```
adduser --home /var/gpg gpgmap
mkdir -p /var/gpg/.gnupg
chown -R gpgmap /var/gpg
chmod 700 /var/gpg/.gnupg
```

#### Install gpgit and it's dependencies:
```
wget https://raw.githubusercontent.com/mikecardwell/gpgit/master/gpgit.pl
```

```
#cpan install MIME::Tools
#cpan install Mail::GnuPG
```

#### Install expect

```
apt-get install expect
```

And create an expect script in **/var/gpg/trust_key.sh**

```
#!/usr/bin/expect -f
set key [lindex $argv 0]
set trust [lindex $argv 1]
spawn /usr/bin/gpg --homedir /var/gpg/.gnupg --edit-key $key trust
expect {
  -re "Your decision?" {
    exp_send "$trust\r"
    exp_continue
  }
  -re "Do you .*" {
    exp_send "y\r"
    exp_continue
  }
  -re "gpg>" {
    exp_send "quit\r"
  }
}
interact
```

Change the ownership and add executable flag:

```
chmod +x /var/gpg/trust_key.sh
chown gpgmap:gpgmap /var/gpg/trust_key.sh
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

The gpgfilter channel receives the email from postfix and passed it to procmail channel which runs **/etc/postfix/procmailrc.common**.
 

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


\#/^Received:/                 IGNORE
/^User-Agent:/               IGNORE
/^X-Mailer:/                 IGNORE
/^X-Originating-IP:/         IGNORE
/^x-cr-[a-z]*:/              IGNORE
/^Thread-Index:/             IGNORE

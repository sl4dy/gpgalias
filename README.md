GPG/PGP encrypted email alias service
========

This repository contains source codes and how-to installation guide for email alias service with GPG/PGP encryption.

Purpose of this solution is to encrypt incoming emails by PGP/GPG key and forward them to the final destination. The high level logic is following:

<img style="float: right" src="pic">

Solution is using Ubuntu 12.04 and Postfix, Dovecot, MySQL, Amavis, Clam AntiVirus, SpamAssassin, Postgrey, Roundcube and Postfix Admin and [gpgit](https://github.com/mikecardwell/gpgit). All this software is installed from standard Ubuntu 12.04 repositories, unless said otherwise.

## Detailed flow
<img style="float: right" src="pic">

## Installation
Use [this](https://www.exratione.com/2012/05/a-mailserver-on-ubuntu-1204-postfix-dovecot-mysql/) guide to install mailserver based on software mentioned above. This will be the starting point for additional tweaking. If you decide to isntall webmail interface Roundcube is recommended for it's simplicity.

## Tweaking

##### Disabling postgrey
Disable postgrey as 



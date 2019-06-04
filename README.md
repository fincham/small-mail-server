# small-mail-server

Configuration and a guide to build a small mail server in 2018.

## Unfinished note dump

```
1) Packages

These packages are needed from Debian (normal 'apt install' will do):

ii  dnsutils                             1:9.10.3.dfsg.P4-12.3+deb9u4      amd64        Clients provided with BIND
ii  postfix                              3.1.9-0+deb9u2                 amd64        High-performance mail transport agent
ii  postfix-pcre                         3.1.9-0+deb9u2                 amd64        PCRE map support for Postfix
ii  dovecot-core                         1:2.2.27-3+deb9u3              amd64        secure POP3/IMAP server - core files
ii  dovecot-imapd                        1:2.2.27-3+deb9u3              amd64        secure POP3/IMAP server - IMAP daemon
ii  dovecot-lmtpd                        1:2.2.27-3+deb9u3              amd64        secure POP3/IMAP server - LMTP server
ii  dovecot-sieve                        1:2.2.27-3+deb9u3              amd64        secure POP3/IMAP server - Sieve filters support

This package is needed from upstream, instructions:

ii  rspamd                               1.8.3-1~stretch                amd64        Rapid spam filtering system

apt-get install -y lsb-release wget # optional
CODENAME=`lsb_release -c -s`
wget -O- https://rspamd.com/apt-stable/gpg.key | apt-key add -
echo "deb [arch=amd64] http://rspamd.com/apt-stable/ $CODENAME main" > /etc/apt/sources.list.d/rspamd.list
echo "deb-src [arch=amd64] http://rspamd.com/apt-stable/ $CODENAME main" >> /etc/apt/sources.list.d/rspamd.list
apt-get update
apt-get --no-install-recommends install rspamd

This package is needed from stretch-backports, instructions:

ii  certbot                              0.28.0-1~deb9u1                all          automatically configure HTTPS using Let's Encrypt

echo "deb http://deb.debian.org/debian stretch-backports main" > /etc/apt/sources.list.d/backports.list
apt update
apt install -t stretch-backports certbot

2) DNS and hostnames

Make sure the server has a consistent idea of what its proper hostname is, and that that hostname is correctly hooked up in DNS.

Assuming your server is called "mx.example.com" and its IP address on the public Internet is 192.0.2.1

 * Edit /etc/hostname so it's the "short" version of the hostname, e.g. "mx"
 * Edit /etc/hosts so it has a line like this, the order is important:

     192.0.2.1 mx.example.com mx

   Replace "192.0.2.1" with some valid IP address for the server, either its Internet IP or the IP on the local network.
 * Reboot
 * "hostname -f" should output "mx.example.com"
 * Edit your DNS so that "mx.example.com" resolves to 192.0.2.1 (use an "A" record)
 * Edit the reverse DNS for 192.0.2.1 so that it resolves to (ideally) "mx.example.com"
 * "dig @8.8.8.8 mx.example.com A" should return 192.0.2.1 (this tests that the public DNS is right)
 * "dig @8.8.8.8 -x 192.0.2.1" should return "mx.example.com" (this tests that the reverse DNS is right)

3) Firewall

We're going to use port 80 for the Let's Encrypt validator service to connect to "mx.example.com" so that it can issue your TLS (aka SSL) certificate. This assumes you're not running a web server on port 80 as well. It is possible to run a web server alongside the mail server but that's out of scope of this process.

Ports you'll need to open in the firewall / map through your NAT:

  25 - for other mail servers to send mail to Postfix with SMTP on your mail server
  80 - for certbot and Let's Encrypt to do certificate validation every 90 days
 465 - for your e-mail clients to connect securely to Postfix with SMTP to submit mail to be delivered onwards
 993 - for your e-mail clients to connect securely to Dovecot with IMAP to retrieve mail 

4) TLS certificate

You'll need a valid TLS certificate (also sometimes called an "SSL" certificate). You can get this for free from Let's Encrypt. Once the above steps are in place you should be able to run certbot to issue the cert. Swap out the hostname for the name of your server.

sudo certbot certonly -d mx.example.com

Follow through the certbot process until you get to the Congratulations! message. Note down the path to the private key and certificate bundle. It'll be something like "/etc/letsencrypt/live/mx.example.com/fullchain.pem" and "/etc/letsencrypt/live/mx.example.com/privkey.pem"

Once the certificate is issued cerbot will renew it automatically every 90 days. If renewal fails then Let's Encrypt will e-mail the address given during setup to warn you.

5) Update dovecot and postfix configuration for proper hostnames

Edit etc/postfix/main.cf and change the smtpd_tls_* lines to point to the correct fullchain/privkey location.

Edit etc/dovecot/conf.d/10-ssl.conf and change the ssl_* lines to point to the correct fullchain/privkey location.

Edit etc/postfix/main.cf and change myhostname to the right hostname and all the various permutations in mydestination.

Edit etc/mailname to the same hsotname.

6) Update rspamd settings

Generate a hash for the rspamd worker password using "rspamadm pw", the hash is the line like $1$.... This password is the one you'll use for logging in to rspamd if you need to to look at statistics, train the classifier etc.

Edit etc/rspamd/local.d/worker-controller.inc and add the hash in the obvious place.

Optionally edit etc/rspamd/local.d/options.inc and set your own DNS server in place of 8.8.8.8 (8.8.8.8 is a Google DNS service).

7) Set up "vmail" user who owns the mail

The "vmail" user owns all the mail so that everyone who uses the system doesn't have to have their own UNIX user. To set up the vmail user:

addgroup --system vmail
adduser --system vmail
usermod -g vmail vmail

Then make the directory where the mail will go:

mkdir -p /srv/mail
chown vmail:vmail /srv/mail
chmod 0750 /srv/vmail

8) Deploy the config!

Run these commands to overwrite the default config in the packages:

sudo rsync -rv ./etc/* /etc
sudo mkdir -p /var/lib/rspamd/dkim/
sudo cp var/lib/rspamd/dkim/* /var/lib/rspamd/dkim

9) Restart postfix, dovecot and rspamd

sudo service dovecot restart
sudo service postfix restart
sudo service rspamd restart

10) Check everything is running properly

You will get useful info from /var/log/mail.* and /var/log/syslog.
```

## Operating system

This guide assumes you're using Debian Stretch.

## Internet connection

You will need an Internet connection where TCP port 25 isn't blocked, and where the IP address isn't marked as "residential" in any way. This can be a little difficult to determine. The safest bet is to not use a residential connection unless you have very careful control over it :(

Not all virtual machine and "cloud" providers are friendly to mail servers.

Some providers that are known to be viable for hosting mail servers:

* https://sitehost.nz/
* https://catalystcloud.nz/
* https://www.digitalocean.com/
* https://aws.amazon.com/

In the interest of transparency: the people who operate SiteHost and Catalyst Cloud are friends.

Notably several large cloud providers do not allow running mail servers. Both Google Cloud and Microsoft Azure specifically prevent users from running mail servers.

This guide assumes that any firewalls in front of your server will allow access to at least TCP ports 25, 80, 465 and 993. You may wish to later use a more restrictive firewall to add additional access control, and this will be discussed further in the Dovecot setup section.

## DNS and reverse DNS

You will need a domain name that you want to serve e-mail for. In this document we will use "example.com", but you will need to have your own domain name and a place to host the DNS for the domain name that lets you add and remove records. 

You may want to consider using a DNS host that has an API for making updates, as this will give you additional choices when it comes to generating a Let's Encrypt certificate.

You will need a DNS name for your mail server. In this document we will use "mail.example.com" but you can choose your own hostname. Once your mail server is working you will need to set the "MX" (Mail eXchanger) record for "example.com" to "0 mail.example.com" to tell other mail servers where to send mail for "example.com".

Being able to control the reverse DNS for the IP addresses of your mail server is important to ensure reliable mail delivery. A lot of receiving mail servers will check the reverse DNS for the connecting server.

It is usually enough to make sure that your IP addresses have a valid reverse DNS and that the thing the reverse DNS resolves to can also be forward resolved to a matching address. 

It is easier though to just make all your forward and reverse DNS the same. Avoid putting anything in the reverse DNS that might suggest your address is "dynamic" or "residential". We will use "mail.example.com" for all of our reverse and forward DNS.

## TLS certificate from Let's Encrypt

To allow mail clients and other mail servers to connect securely to your server you will need a TLS certificate. The best way to get this is to automatically issue one from [Let's Encrypt](https://letsencrypt.org/). We will prefer not to use STARTTLS where possible as generally direct TLS connections are implemently more reliably.

Follow the instructions at http://backports.debian.org/Instructions/ to enable the Stretch Backports repo, then install `certbot` with:

    sudo apt install -t stretch-backports certbot 

Once `certbot` is intalled create the file `/etc/letsencrypt/cli.ini` with these contents:

    max-log-backups = 0
    rsa-key-size = 4096
    authenticator = standalone
    agree-tos = true    
    email = you@example.com
    domains = mail.example.com

This guide assumes you will not also be running a web server on your mail server. If you do want to run a web server on your mail server, stop now and set it up. Once you are done, follow the instructions on https://certbot.eff.org to set up a Let's Encrypt certificate for your web server. We'll then re-use that certificate for the mail server.

If your're not running a web server and don't intend to you can use certbot in standalone mode to issue a certificate. Make sure your DNS has been set up for "mail.example.com" already and then run:

    sudo certbot certonly
   
Once `certbot` has completed you should now have the files `/etc/letsencrypt/live/mail.example.com/fullchain.pem` and `/etc/letsencrypt/live/mail.example.com/privkey.pem` which we will refer to later.

## Postfix

Postfix will be used as the MTA (Mail Transport Agent) in this setup. Its main functions are:

* Other mail servers will connect to it on port 25 to deliver mail to you.
* It'll run incoming messages through rspamd to try and identify spam and check DKIM (DomainKeys Identified Mail) and SPF and blacklists.
* Users will connect to port 587 to "submit" new messages to be relayed out to other mail servers.
* Incoming mail will be handed to Dovecot's LDA (Local Delivery Agent) using LMTP (Local Mail Transfer Protocol) for local delivery.
* Authentication for port 587 will also be handed off to Dovecot.

Postfix has many years of baggage and "features". We will only be using a fairly small subset of it here.

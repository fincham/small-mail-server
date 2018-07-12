# small-mail-server

Configuration and a guide to build a small mail server in 2018.

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

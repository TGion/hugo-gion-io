---

title: "Automatic Renewal of Wildcard Certificates"  
date: 2022-12-11T08:00:00+02:00  

draft: false  

description: "Automatically issue and renew wildcard certificates with acme.sh and the Namecheap API."  
keywords: [tls, ssl, freebsd, web, certificate, wildcard]  

tags: [bsd, security, web]  
toc: false

---

To automate the process of issuing and renewing TLS wildcard certificates we use [acme.sh](https://github.com/acmesh-official/acme.sh/) - A pure Unix posh script implementing ACME client protocol. To complete the DNS challenge without manually adding TXT records, we are supported by the [Namecheap API](https://www.namecheap.com/support/api/intro/) in this guide.

## Goals

After the completion of this guide, we will be able to:

1. Install and properly configure acme.sh as well as setting up a DNS challenge through the Namecheap API
2. Issue a wildcard certificate from a CA of our choice- we will use [Let's Encrypt](https://letsencrypt.org/) in this guide
3. Create an automatic renewal process for the certificate as well as reload our services after a successful renewal

## Installation

With FreeBSD, it basically boils down to two options when installing acme.sh: 
The installation via the FreeBSD ports collection or using the acme.sh installer.

Although I prefer the installation via the FreeBSD ports collection for maintenance reasons, it is of course possibly (and maybe preferred by others) to use the acme.sh installer.

### FreeBSD ports collection

Login as root, update the ports collection and configure the port (if desired) for acme.sh.

```bash
su
cd /usr/ports
make update
cd security/acme.sh
make config
```

If you want to change the default port configuration, now is your chance. I have disabled some default settings for which I have no use at the moment.

![](/img/Screenshot_2022-11-30_10-54-17.png)

When you are done, it is time to build and install the port.

```bash
make install clean
```

### acme.sh Installer

If you decide to not use the FreeBSD ports collection (or run another OS) you can check out the installation via the acme.sh installer [here](https://github.com/acmesh-official/acme.sh/#1-how-to-install).

Please be advised, when you use the acme.sh installer, your configuration files and certificates are in the home directory of the user you ran the installer with. This will be different from the paths used in the rest of this guide.

## Configuration

There are two types of configuration files used by acme.sh.

### account.conf

The first one is the global (or account) configuration. These settings are necessary (and valid) for every domain you issue. Lets have a look.

```bash
# /var/db/acme/.acme.sh/account.conf

# needed if one uses DNS alias mode
# more information here: https://github.com/acmesh-official/acme.sh/wiki/DNS-alias-mode
# NSUPDATE_SERVER="mydns.example.org"
# NSUPDATE_KEY="/var/db/acme/Kmydns.example.org.+165+59977.key"

DEFAULT_DNS_SLEEP="10"
CERT_HOME="/var/db/acme/certs"
LOG_FILE='/var/log/acme.sh.log'

# Namecheap API settings
NAMECHEAP_API_KEY='SECRET_API_KEY'
NAMECHEAP_USERNAME='USERNAME'
NAMECHEAP_SOURCEIP='WHITELISTED_IP_ADDRESS'

# Default CA Server
DEFAULT_ACME_SERVER='https://acme-v02.api.letsencrypt.org/directory'
```

The NSUPDATE settings were disabled since no DNS alias mode is used.

Also the Namecheap API credentials have been added. More information on setting up the Namecheap API are found [here](https://www.namecheap.com/support/api/intro/). acme.sh supports quite a lot different [DNS API's](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) if you use a different provider.

The last configuration is setting your default / preferred CA's server address. By default, acme.sh uses ZeroSSL. In our example we use Let's Encrypt instead. Changes can be made directly to the configuration file or by calling:

```bash
acme.sh --set-default-ca --server https://acme-v02.api.letsencrypt.org
```

acme.sh supports many [different CA's](https://github.com/acmesh-official/acme.sh/#supported-ca) as well.

### *\[domain.tld\]*.conf

The second configuration affects only the specific certificate / domain and will be created when we issue our certificate. It is located in the directory: ```$CERT_HOME/domain.tld/domain.tld.conf```
where *domain.tld* is the name of your domain you issued your certificate for.

### Logfile

Since FreeBSD creates a new user / group for acme which has no permissions to write into */var/log*, a log file must be manually created and the appropriate permissions must be set. This has to be only done once as user root.

```bash
su
touch /var/log/acme.sh.log
chown acme:acme /var/log/acme.sh.log
```

It is possible to setup *newsyslogd* for the logfiles. FreeBSD installs the necessary script. If desired, it can be enabled.

```bash
# /usr/local/etc/newsyslog.d/acme.sh

# logfilename         [owner:group]   mode count size when  flags [/pid_file] [sig_num]
/var/log/acme.sh.log  acme:acme       640  90    *    @T00   BC
```

Restart *newsyslog* after changes have been made.

```bash
service newsyslog restart
```

## Issue certificates

Now it is time to issue our certificate. Based on the issue command, acme.sh adds options to your domain.conf. Since some of those options will be encrypted in your configuration file, it is advised to set them with your issue command.

First, login as user acme.

```bash
su acme
```

For my configuration, I issued my certificates with the following command.

```bash
acme.sh --issue \
--dns dns_namecheap \
--domain domain.tld -d *.domain.tld \
--reloadcmd "/var/db/acme/services.restart" \
--preferred-chain "ISRG Root X1" 
```

### Parameters explained

To go briefly trough the parameters:

* ```issue``` - issues a new certificate
* ```dns dns_namecheap``` - use a DNS challenge to verify the domain and use the DNS hook *dns_namecheap*. It is necessary to enable the Namecheap API and also setup the credentials in the *account.conf* as mentioned in the configuration part of this guide.
* ```domain``` or ```-d``` - list of domains you want to issue certificates for. In this example a certificate for the root domain as well as every subdomain (wildcard) is issued.
* ```reloadcmd``` - list of commands / scripts to be executed after a certificate is issued or renewed.  
   I have used a script in my configuration to be more flexible for future applications. If you only use a webserver, a simple service restart will suffice here:  
   ```reloadcmd "doas service nginx restart"```

### Reload script

```bash
# /var/db/acme/services.restart

#!/bin/sh
doas /usr/sbin/service nginx reload
doas /usr/sbin/service adguardhome restart
```

I use doas to gain super user access to restart services. Of course you can use any other means.

* ```preferred-chain``` - select a specific certificate chain. In our case *ISRG Root X1* from Let's Encrypt.  
   To use my own private DNS on my Android Phone, i had to select this specific chain from the CA. Here is some more information on [certificate compatibility](https://letsencrypt.org/docs/certificate-compatibility/).

### Certificate files

The certificate(s) will be put in the previously defined ```$CERT_HOME``` directory:

* ```$CERT_HOME/cert/domain.tld/domain.tld.key```
* ```$CERT_HOME/cert/domain.tld/fullchain.cer```

Simply point your webserver or other services configuration to the appropriate path and reload / restart them manually.

## Automatic Renewal

To automatically renew all our previously issued certificates acme.sh delivers a crontab script. To be more consistent to the FreeBSD system, I decided to add a system crontab for the user acme.

### Crontab

As user acme edit the crontab file with the following command:

```bash
crontab -e
```

And add the following line to your crontab file:

```bash
10 0 * * * /usr/local/sbin/acme.sh --cron --home "/var/db/acme/.acme.sh" > /dev/null
```

It will run the command acme.sh in the cron mode every day at 0:10 system time. Dont worry about sending the outpout to */dev/null*. acme.sh still writes its logs to the predefined log directory.

### Renewal test run

You can test run the cron job with to confirm everything works as intended:

```bash
acme.sh --cron --test
```
 
acme.sh will go trough every certificate previously issued and will use the settings used to issue the certificate. For our example it will also use the Namecheap API for the DNS challenge and run our reload command.

## Conclusion

We are now able to issue certificates, including wildcard ones. The use of a DNS API helps us to automatically run a DNS challenge for our domains without the need to manually add those TXT records. We can furthermore streamline the renewal process with the help of crontab files and reload commands.
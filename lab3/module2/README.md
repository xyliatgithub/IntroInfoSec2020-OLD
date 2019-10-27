# Module 2: Certificate Authority Hierarchy

## Table of Contents

- [Introduction](#introduction)
- [Network Toppology](#network-topology)
- [Certificate Issuance](#certificate-issuance)
- [Server setups](#server-setups)
- [Certificate Issuance Test](#certificate-issuance-test)
- [(Optional) Setup GUI option (Firefox Web Browser) on GENI nodes](#optional-setup-gui-option-firefox-web-browser-on-geni-nodes)
- [Questions](#questions)

## Introduction

![Alt text](https://raw.githubusercontent.com/xyliatgithub/EN650.601/master/assignment2/module2/pic/Picture37.png)

As the picture shows, certificate authorities(CAs) are organized in a tree structure. Trust is propagated from the root to leaves. A certificate is trustworthy as long as it is signed by a trusted superior CA. Thus, when verifying a certificate, all certificates of the CAs involving in this path of trust are required.

In this lab, you will simulate the issuance of certificates within this structure, and test their validities through a SSL connection from client(**user**) to server(**ws**), which means most of the contents are similar to what you did in the first lab.

## Network topology

![Alt text](https://raw.githubusercontent.com/xyliatgithub/EN650.601/master/assignment2/module2/pic/Picture100.png)

1. **ca1** will be the root CA in this experiment.
2. **ca2** will be the intermediate CA, that is, the subordinate CA of **ca1**.
3. **ca3** will be the local CA, that is, the subordinate CA of **ca2**.
4. **ws** will be the web server where LAMP is installed.
5. **user** will be the client connecting to **ws**.

**_Notice: Not until all the GENI nodes turn green can you continue the following steps. This may take a while._**

## Certificate Issuance

### Generate keys and CSRs

#### Root CA

Similar to the first lab, you will have to initialize some files and directories. Connect to **ca1** and run the following commands.

```
sudo su
touch /etc/ssl/index.txt
echo 01 > /etc/ssl/serial
echo 01 > /etc/ssl/crlnumber
mkdir /etc/ssl/newcerts
```

Then, generate a key pair

```
openssl genrsa -des3 -out /etc/ssl/private/ca1.key 2048
```

You will be asked to set a 4 to 1023 characters passphrase. It will be used when signing **ca2**'s CSR.

```
Generating RSA private key, 2048 bit long modulus
....................................................................................................................................+++
..................+++
e is 65537 (0x10001)
Enter pass phrase for /etc/ssl/private/ca1.key:
Verifying - Enter pass phrase for /etc/ssl/private/ca1.key:
```

Generate a self-signed certificate with that private key.

```
openssl req -key /etc/ssl/private/ca1.key -new -x509 -days 365 -out /etc/ssl/ca1.crt
```

Fill in the information of that certificate. The common name does not matter if the node is not going to host any server. Thus, you can pick whatever name you want for CAs. However, the common name for web server should be consistent with your settings. For simplicity, the common names used in the example will correspond to the names of nodes, that is jhuca1.edu, jhuca2.edu, jhuca3.edu, and jhuws.edu

```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:MD
Locality Name (eg, city) []:Baltimore
Organization Name (eg, company) [Internet Widgits Pty Ltd]:JHU
Organizational Unit Name (eg, section) []:ISI
Common Name (e.g. server FQDN or YOUR name) []:jhuca1.edu
Email Address []:
```

Finally, edit the file "openssl.cnf" line 42 which is in "/etc/ssl" directory:

```
[ CA_default ]

 dir             = /etc/ssl              # Where everything is kept
 certs           = $dir/certs            # Where the issued certs are kept
 crl_dir         = $dir/crl              # Where the issued crl are kept
 database        = $dir/index.txt        # database index file.
 #unique_subject = no                    # Set to 'no' to allow creation of
```

#### Intermediate CA

In contrast to the first lab, there are multiple CA nodes in the second one. One of the extra CA nodes is intermediate CA. An intermediate CA, on the one hand, generates certificate signing request(CSR) and send it to its superior CA just like web server; on the other hand, it signs its subordinate CA's CSRs like root CA. Therefore, tasks to be done on **ca2** are the combination of those you did on **ws** and **ca** previously. First, connect to **ca2** and do the followings.

```
sudo su
touch /etc/ssl/index.txt
echo 01 > /etc/ssl/serial
echo 00 > /etc/ssl/crlnumber
mkdir /etc/ssl/newcerts
```

Then, generate a private key

```
openssl genrsa -des3 -out /etc/ssl/private/ca2.key 2048
```

Create a CSR of this key

```
openssl req -new -days 3650 -key /etc/ssl/private/ca2.key -out /etc/ssl/ca2.csr
```

Similarly, you will have to put in some information. Then, edit the file "openssl.cnf" line 42 in "/etc/ssl" directory:

```
[ CA_default ]

 dir             = /etc/ssl              # Where everything is kept
 certs           = $dir/certs            # Where the issued certs are kept
 crl_dir         = $dir/crl              # Where the issued crl are kept
 database        = $dir/index.txt        # database index file.
 #unique_subject = no                    # Set to 'no' to allow creation of
```

#### Local CA

The only difference between local CAs and the intermediate CAs is the CSR they sign. Local CAs sign CSR belonging to web server, while intermediate CAs sign CSRs from subordinate CAs. Thus, the setup is pretty much the same. Connect to **ca3** and do run these commands.

```
sudo su
touch /etc/ssl/index.txt
echo 01 > /etc/ssl/serial
echo 00 > /etc/ssl/crlnumber
mkdir /etc/ssl/newcerts
```

Generate a private key

```
openssl genrsa -des3 -out /etc/ssl/private/ca3.key 2048
```

Create CSR of this key

```
openssl req -new -days 3650 -key /etc/ssl/private/ca3.key -out /etc/ssl/ca3.csr
```

Similarly, you will have to put in some information. Then, edit the file "openssl.cnf" line 42 in "/etc/ssl" directory:

```
[ CA_default ]

 dir             = /etc/ssl              # Where everything is kept
 certs           = $dir/certs            # Where the issued certs are kept
 crl_dir         = $dir/crl              # Where the issued crl are kept
 database        = $dir/index.txt        # database index file.
 #unique_subject = no                    # Set to 'no' to allow creation of
```

#### Web Server

The web server won't function as a certificate authority, thus, just generate a private key and a corresponding CSR.

```
sudo openssl genrsa -des3 -out /etc/ssl/private/ws.key 2048
sudo openssl req -new -days 3650 -key /etc/ssl/private/ws.key -out /etc/ssl/ws.csr
```

### Sign certificates

#### Intermediate CA

First, move intermediate CA's CSR to the root CA, that is, use WinSCP/SFTP or other methods to transfer "ca2.csr" from **ca2** to **ca1** and sign "ca2.csr" with the following command

```
sudo openssl ca -extensions v3_ca -in ca2.csr -config /etc/ssl/openssl.cnf -days 3000 -out ca2.crt -cert /etc/ssl/ca1.crt -keyfile /etc/ssl/private/ca1.key
```

After it is signed, return the resulting certificate "ca2.crt" to **ca2**.

#### Local CA

Move local CA's CSR to the intermediate CA, that is, transfer "ca3.csr" from **ca3** to **ca2**, and sign "ca3.csr" with the following command

```
sudo openssl ca -extensions v3_ca -in ca3.csr -config /etc/ssl/openssl.cnf -days 3000 -out ca3.crt -cert ca2.crt -keyfile /etc/ssl/private/ca2.key
```

After it is signed, return the resulting certificate, "ca3.crt," to **ca3**.

#### Web Server

Signing the web server's certificate is a bit different if you want to test the certificate chain with Firefox. In order to not encounter a **_MOZILLA_PKIX_ERROR_CA_CERT_USED_AS_END_ENTITY_** error, you need to modify the openssl.cnf in **ca3**.

Open up openssl.cnf and modify line 236. Change CA:true to CA:false under [ v3_ca ].

```
[ v3_ca ]


# Extensions for a typical CA


# PKIX recommendation.

subjectKeyIdentifier=hash

authorityKeyIdentifier=keyid:always,issuer

#Change the following boolean to false
basicConstraints = critical,CA:true
```

Finally, move web server's CSR to the local CA, that is, transfer "ws.csr" from **ws** to **ca3**, and sign "ws.csr" with the following command

```
sudo openssl ca -extensions v3_ca -in ws.csr -config /etc/ssl/openssl.cnf -days 3000 -out ws.crt -cert ca3.crt -keyfile /etc/ssl/private/ca3.key
```

After it is signed, return the resulting certificate, "ws.crt," to **ws**. To enable **ws**’s certificate, the certificate should be put into "/etc/ssl."

```
sudo cp <ws_cert_location> /etc/ssl/certs
```

For example,

```
sudo cp ~/ws.crt /etc/ssl/certs/
```

## Server setups

Most of the contents are pretty much the same as the previous lab, for simplicity, detail explanations for installations are omitted and only commands are listed here.

### Install LAMP

```
sudo apt-get update
sudo apt-get install mysql-server
sudo apt-get install apache2
sudo apt-get install php-pear php-fpm php-dev php-zip php-curl php-xmlrpc php-gd php-mysql php-mbstring php-xml libapache2-mod-php
sudo apt-get install php-intl
sudo apt-get install php-imagick
sudo apt-get install php-imap
sudo apt-get install php-memcache
sudo apt-get install php7.2-ps //You probably will need to change the version number
sudo apt-get install php-pspell
sudo apt-get install php-recode php-snmp php7.2-sqlite php-tidy php7.2-xsl
```

### Write a simple PHP website

First change the permission of "/var/www."

```
sudo chmod 777 /var/www
```

Second, create a file named “info.php” under the “/var/www/html” folder, then copy these codes into this file:

```
<?php
phpinfo();
?>
```

Third, modify the directory file at '/etc/apache2/mods-enabled/', enter the following command:

```bash
vim /etc/apache2/mods-enabled/dir.conf
```

Then add info.php to the beginning:

```sh
<IfModule mod_dir.c>
        DirectoryIndex info.php index.html index.cgi index.pl index.php index.xhtml index.htm
</IfModule>
```

Finally, restart the Apache service.

```
sudo /etc/init.d/apache2 restart
```

### Configure certificates

It is more complicated to configure certificates in this lab since depth of the certificate chain is no longer one. Use SFTP/WinSCP or any other file transfer tools to put "ws.crt", "ca1.crt", "ca2.crt", "ca3.crt" onto **ws** node. Then, concatenating "ca1.crt", "ca2.crt", "ca3.crt" to create "ca.crt".

```
cat ca1.crt ca2.crt ca3.crt > ca.crt
```

Move "ws.crt" and "ca.crt" into "/etc/ssl/certs/"

```
mv ca.crt /etc/ssl/certs/
mv ws.crt /etc/ssl/certs/
```

### Enable Apache SSL connection

To enable SSL connection, copy "default-ssl.conf" from "sites-available" to "sites-enabled".

```
sudo cp /etc/apache2/sites-available/default-ssl.conf  /etc/apache2/sites-enabled/default-ssl.conf
```

Then, find its IP address

```
ifconfig
```

Modify "/etc/apache2/sites-enabled/default-ssl.conf" :

```
<IfModule mod_ssl.c>
        <VirtualHost 172.17.2.12:443>
                ServerAdmin webmaster@localhost
                ServerName www.jhuws.edu
                DocumentRoot /var/www/html

                # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
                # error, crit, alert, emerg.
                # It is also possible to configure the loglevel for particular
                # modules, e.g.
                #LogLevel info ssl:warn

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                # For most configuration files from conf-available/, which are
                # enabled or disabled at a global level, it is possible to
                # include a line for only one particular virtual host. For example the
                # following line enables the CGI configuration for this host only
                # after it has been globally disabled with "a2disconf".
                #Include conf-available/serve-cgi-bin.conf

                #   SSL Engine Switch:
                #   Enable/Disable SSL for this virtual host.
                SSLEngine on

                #   A self-signed (snakeoil) certificate can be created by installing
                #   the ssl-cert package. See
                #   /usr/share/doc/apache2/README.Debian.gz for more info.
                #   If both key and certificate are stored in the same file, only the
                #   SSLCertificateFile directive is needed.
                SSLCertificateFile      /etc/ssl/certs/ws.crt
                SSLCertificateChainFile /etc/ssl/certs/ca.crt
                SSLCertificateKeyFile /etc/ssl/private/ws.key
```

In line 2, replace "\_default\_" with the IP address you just found. Do not remove “:443” after the IP address. It is the port number for SSL connection. In line 4, insert a line and specify your ServerName. In line 32, 33, and 34, change the directory to where your certificate and private key are stored, and add a new line specifying **SSLCertificateChainFile**.

**_Notice: ServerName should be consistent with your previous setting while generating CSR._**

Check whether the configuration is correct:

```
sudo apachectl configtest
```

If so, use the following command

```
sudo a2enmod ssl
```

to enable the SSL module of Apache2. Then, restart apache2 server using

```
sudo service apache2 restart
```

Type in the passphrase for your private key.

```
Enter passphrase for SSL/TLS keys for pcvm2-12.geni.it.cornell.edu:443 (RSA):
```

## Certificate Issuance Test

:::info
Hint: Before you move on, do you still remember in Lab 1, we had to modify a certain type file for some reason? Do you think we need to do the same here?
:::

### Install the self-signed certificates from root CA

Put "ca1.crt" into /etc/ssl/certs on the **user** node, and create a symbolic link for it (more details of symbolic link can be found [here](http://gagravarr.org/writing/openssl-certs/others.shtml#ca-openssl)).

```
ln -s ca1.crt `openssl x509 -hash -noout -in ca1.crt`.0
```

Use OpenSSL to test the connection.

```
openssl s_client -showcerts -connect www.jhuws.edu:443
```

If everything went well, you will see messages like:

```
CONNECTED(00000003)
depth=3 C = US, ST = MD, L = Baltimore, O = JHU, OU = ISI, CN = jhuca1.edu
verify return:1
depth=2 C = US, ST = MD, O = JHU, OU = ISI, CN = jhuca2.edu
verify return:1
depth=1 C = US, ST = MD, O = JHU, OU = ISI, CN = jhuca3.edu
verify return:1
depth=0 C = US, ST = MD, O = JHU, OU = ISI, CN = jhuws.edu
verify return:1
```

If any of your certificate wasn't signed correctly or you did not installed the root certificate on the **user** node, you might see the following error messages:

```
verify error:num=19:self signed certificate in certificate chain
verify error:num=20:unable to get local issuer certificate
verify error:num=21:unable to verify the first certificate
```

## (Optional) Setup GUI option (Firefox Web Browser) on GENI nodes

First, we need to install Firefox onto the `Victim`'s system.

```bash
$ sudo apt-get install firefox
```

Unfortunately, we cannot use this yet since the GENI node's system doesn't have a display attached. We need to do the followings based on the different operating systems of your personal computer.

### Windows

If you were using Windows all along, there is a very high percent chance that you are using PuTTY for your ssh connection. We need to do some tweaking before we can launch Firefox remotely.

Go to [https://sourceforge.net/projects/xming/](https://sourceforge.net/projects/xming/) and download Xming.

Install the program and launch Xming on your Windows computer.

Now open up PuTTY and on the left-hand side, you can see a dropdown tree of SSH, click on X11 and check 'Enable X11 forwarding'.

![PuTTY X11](https://github.com/bguan-2018/BlockchainResearch/blob/master/Digital%20Certificate%20Lab%203/PuTTY%20X11.png?raw=true)

Login back into the GENI node and enter `firefox` to open up a Firefox browser.

### MacOS

Open your browser on your Mac and head to the link [https://www.xquartz.org/](https://www.xquartz.org/) and download XQuartz.

Install the application.

Now, open up a regular terminal and login to the GENI node by using this command (don't forget to modify connection address):

```bash
$ ssh -i ~/.ssh/id_geni_ssh_rsa -X robin@pc2.instageni.wisc.edu -p 26042
```

Did you notice the difference?

By including option -X, we are able to enable X11 connection forwarding and run graphical applications installed on GENI systems.

After that, you should be able to launch the Firefox browser by typing `firefox` in the GENI console.

### Linux

Fortunately it's easier to use the GUI option on Linux OS. SSH into the remote machine with the additional `-X` and `-Y` command

```bash
$ ssh -i ~/.ssh/id_geni_ssh_rsa -X -Y robin@pc3.instageni.cs.princeton.edu -p 25210
```

Install the Firefox on the remote node

```bash
$ sudo apt-get update

$ sudo apt-get install firefox
```

Then, launch the command with `firefox` command, and you should see the following Firefox window popup.

![linux-firefox-window-pic](https://i.imgur.com/kAUT5DT.png)

---

### Trusting the certificate

Before we can visit jhuws.edu using HTTPS, we have to import the root certificate (`ca1.crt`) so that the browser can trust that certificate. Launch Firefox and on the top right corner, click on the options icon and select 'Preferences'.

![Firefox Preference](https://github.com/bguan-2018/BlockchainResearch/blob/master/Digital%20Certificate%20Lab%203/preference.png?raw=true =200x)

After you open Preferences, we can search for the option `Certificates` like such:

![Firefox Certificate](https://github.com/bguan-2018/BlockchainResearch/blob/master/Digital%20Certificate%20Lab%203/certificate.png?raw=true =400x)

Click on `View Certificates`.
This popup will appear, click on 'Import'.

![Firefox Certificate Manager](https://github.com/bguan-2018/BlockchainResearch/blob/master/Digital%20Certificate%20Lab%203/certificate%20manager.png?raw=true =400x)

In this case, We need to import `ca1.crt`, `ca2.crt`, `ca3.crt`. You will see a pop-up asking for the purpose of the certificates. Select _Trust this CA to identify websites_.

We also can see the hierarchy of other organizations’ certificate authority in the display.

Then, visit https://jhuws.edu in the browser window!

![PHP page](https://i.imgur.com/gPgeFK6.png)

Click on the lock icon next to the URL, ">", "More information", "Security", "View Certificate", and you should be able to view the whole certificate hierarchy!
![hierarchy](https://i.imgur.com/shcpQ3R.png)

## Questions

1. Why **ws** has to concatenate "ca1.crt", "ca2.crt", "ca3.crt" into one file? Is there an alternative way to achieve the same goal?

2. In our experiment, which node performs the role of root CA? What will happen if the root CA revokes the digital certificate of the intermediate CA? Try this and screenshot your result.

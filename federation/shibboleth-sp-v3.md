# HOWTO Install and Configure a Shibboleth SP v3.x on Debian-Ubuntu Linux

## Table of Contents

01. [Requirements Hardware](#requirements-hardware)
02. [Software that will be installed](#software-that-will-be-installed)
03. [Other Requirements](#other-requirements)
04. [Installation Instructions](#installation-instructions)
    01. [Install software requirements](#install-software-requirements)
    02. [Configure the environment](#configure-the-environment)
    03. [Install Shibboleth Service Provider](#install-shibboleth-service-provider)
05. [Configuration Instructions](#configuration-instructions)
    01. [Configure SSL on Apache2](#configure-ssl-on-apache2)
    02. [Configure Shibboleth SP](#configure-shibboleth-sp)
    03. [Configure an example federated resource "secure"](#configure-an-example-federated-resource-secure)
    04. [Enable Attribute Support on Shibboleth SP](#enable-attribute-support-on-shibboleth-sp)
    05. [Connect SP to the Federation](#connect-sp-to-the-federation)
    06. [Connect SP directly to an IdP](#connect-sp-directly-to-an-idp)
06. [Test](#test)
07. [Enable Attribute Checker Support on Shibboleth SP](#enable-attribute-checker-support-on-shibboleth-sp)
08. [Increase startup timeout](#increase-startup-timeout)
09. [OPTIONAL - Maintain 'shibd' working](#optional---maintain-shibd-working)
10. [Utility](#utility)
11. [Authors](#authors)
12. [Thanks](#thanks)

## Requirements Hardware

- CPU: 2 Core
- RAM: 4 GB
- HDD: 20 GB
- OS: Debian 10

## Software that will be installed

- ca-certificates
- ntp
- vim
- libapache2-mod-php, php, libapache2-mod-shib, apache2 (>= 2.4)
- openssl

## Other Requirements

- SSL Credentials: HTTPS Certificate & Key

## Notes

You will use the sp-xx.ubuntunet.org servers for this exercise.

This HOWTO mostly uses`example.org` as domain name and `sp.example.org` as FQDN (Full Qualified Domain Name) to provide example values to this guide.

Please, remember to **replace all occurence** of `example.org` domain name, or part of it, with the SP domain name into the configuration files and also `sp.example.org` with the FQDN of your SP server.

I will also provide an example using my details below:
- Server FQDN: `sp-f01.cranecloud.africa`
- Server Domain Name: `cranecloud.africa`
-  Realm: `cranecloud.africa`

Please note that your machine FQDNs are `sp-xx.ubuntunet.org`. This is different from your institutional domain name or realm such as `botsren.org.bw` or `cranecloud.africa`. You will use your institutional domain name in the configuration files.

## Installation Instructions

### Install software requirements

01. Become ROOT:

    - `sudo su -`


02. Update packages:

    - `apt update && apt-get upgrade -y --no-install-recommends`
  
03. Install the packages required:

    - `apt install ca-certificates vim openssl`

### Configure the environment

01. Modify your `/etc/hosts`:

    - `vim /etc/hosts`
  
      ```bash
      127.0.1.1 sp.example.org sp
      ```

      (*Replace `sp.example.org` with your SP Full Qualified Domain Name for example sp-10.ubuntunet.org*)

      (*Replace `sp` with your SP Hostname*)


### Install Shibboleth Service Provider

01. Become ROOT:

    - `sudo su -`

02. Install Shibboleth SP:

    - `apt install apache2 libapache2-mod-shib ntp --no-install-recommends`

      From this point the location of the SP directory is: `/etc/shibboleth`

## Configuration Instructions

### Configure SSL on Apache2

> According to [NSA and NIST](https://www.keylength.com/en/compare/), RSA with 3072 bit-modulus is the minimum to protect up to TOP SECRET over than 2030.

01. Become ROOT:

    - `sudo su -`

02. Create the DocumentRoot:

    ```bash
    mkdir /var/www/html/$(hostname -f)
    
    sudo chown -R www-data: /var/www/html/$(hostname -f)
    
    echo '<h1>It Works!</h1>' > /var/www/html/$(hostname -f)/index.html
    ```

03. Create the Virtualhost file (**please pay attention: you need to edit this file and customize it, check the initial comment inside of it**):

    ```bash
    wget https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Service%20Provider/utils/sp.example.org.conf -O /etc/apache2/sites-available/000-$(hostname -f).conf
    ```

04.  Setting Up Let's Encrypt (TLS)
        ```bash
            sudo apt update
            sudo apt install -y certbot python3-certbot-apache
        ```

   Confirm it installed correctly:

   ```bash
    certbot --version
   ```

   Use the Apache plugin, which will automatically detect your vhost, obtain the certificate, and edit the Apache configuration to enable HTTPS.

   ```bash
   sudo certbot --apache -d sp-[participant_id].ubuntunet.org
   ```

   You will be prompted to:

   1. Enter an email address (used for renewal/expiry notifications).
   2. Agree to the Let's Encrypt Terms of Service.
   3. Optionally share your email with the EFF.
   4. Choose whether to redirect all HTTP traffic to HTTPS — **choose the redirect option (recommended)**, so `http://sp-[participant id].ubuntunet.org` automatically forwards to `https://`.

   If successful, Certbot will report where the certificate files were saved, typically:

      ```
      /etc/letsencrypt/live/sp-[participant_id].ubuntunet.org/fullchain.pem
      /etc/letsencrypt/live/sp-[participant_id].ubuntunet.org/privkey.pem
      ```

4. Put SSL credentials in the right place (Your FQDN is `idp-[participant_id].ubuntunet.org`):
   * HTTPS Server Certificate (Public Key) inside `/etc/ssl/certs/$(hostname -f).crt`
      ```bash
      ln -s /etc/letsencrypt/live/<FQDN>/fullchain.pem /etc/ssl/certs/$(hostname -f).crt
      ```

   * HTTPS Server Key (Private Key) inside `/etc/ssl/private/$(hostname -f).key`
      ```bash
      ln -s /etc/letsencrypt/live/<FQDN>/privkey.pem /etc/ssl/private/$(hostname -f).key
      ```
   * Add CA Cert into `/etc/ssl/certs`:
       ```bash
      ln -s /etc/letsencrypt/live/<FQDN>/cert.pem /etc/ssl/certs/$(hostname -f)-CA.pem
       ```

5. Configure the right privileges for the SSL Certificate and Key used by HTTPS:

   ```bash
   chmod 400 /etc/ssl/private/$(hostname -f).key

   chmod 644 /etc/ssl/certs/$(hostname -f).crt
   ```

6. Enable the following Apache2 modules and VirtualHost:
   * `a2enmod ssl` - To support SSL protocol
   * `a2enmod headers` - To control of HTTP request and response headers.
   * `a2enmod alias` - To manipulation and control of URLs as requests arrive at the server.
   * `a2enmod include` - To process files before they are sent to the client.
   * `a2enmod negotiation` - Essential Apache module
   * `a2ensite 000-$(hostname -f).conf` - Enable SSP IdP VirtualHost
   * `a2dissite 000-default.conf default-ssl 000-default-le-ssl.conf` - Disable HTTP & HTTPS & LE default VirtualHost
 

7. Configure and ensure Apache2 is working as expected:

   You need to edit the `/etc/apache2/sites-available/000-$(hostname -f).conf` file and replace all occurences of `sp.example.org` with your SP Full Qualified Domain Name.

   Edit the following line in the `/etc/apache2/sites-available/000-$(hostname -f).conf` file: **Remember to replace `GEANT_TLS_RSA_1.pem` with the CA certificate you are using for your SP server. For this exercise, it is `hostname -f`-CA.pem an example being `sp-01.ubuntunet.org-CA.pem`**:

   ```apache
      SSLCACertificateFile /etc/ssl/certs/sp-01.ubuntunet.org-CA.pem
   ```

    We can now go ahead and restart the Apache2 service to apply the changes:

  * `systemctl restart apache2.service`


8. Check that the SP works:
   * <span>https://sp.example.org/ <span> (*Replace `sp.example.org` with your SP Full Qualified Domain Name*)


### Configure Shibboleth SP

01. Become ROOT:

    - `sudo su -`

02. Change the SP entityID and technical contact email address:

    ```bash
    sed -i "s/sp.example.org/$(hostname -f)/" /etc/shibboleth/shibboleth2.xml

    sed -i "s/root@localhost/<TECH-CONTACT-EMAIL-ADDRESS-HERE>/" /etc/shibboleth/shibboleth2.xml

    sed -i 's/handlerSSL="false"/handlerSSL="true"/' /etc/shibboleth/shibboleth2.xml

    sed -i 's/cookieProps="http"/cookieProps="https"/' /etc/shibboleth/shibboleth2.xml

    sed -i 's/cookieProps="https">/cookieProps="https" redirectLimit="exact">/' /etc/shibboleth/shibboleth2.xml
    ```

03. Create SP metadata Signing and Encryption credentials:

    - Ubuntu:

      ```bash
      cd /etc/shibboleth

      shib-keygen -u _shibd -g _shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-signing -f

      shib-keygen -u _shibd -g _shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-encrypt -f

      /usr/sbin/shibd -t

      systemctl restart shibd.service

      systemctl restart apache2.service
      ```


04. Enable Shibboleth Apache2 configuration:

    ```bash
    a2enmod shib
    ```

05. Remove the `#` character from the `#Redirect ...` line on the Apache2 configuration to enable it (this is among the last items in that file):

    - `vim /etc/apache2/sites-available/000-$(hostname -f).conf`

      ```bash
      #Redirect "/shibboleth" "/Shibboleth.sso/Metadata"
      ```

06. Reload Apache2 service to apply changes:

    - `systemctl reload apache2.service`

07. Now you are able to reach your Shibboleth SP Metadata from its entityID:

    - `https://sp.example.org/shibboleth`

    or from its Metadata endpoint:

    - `https://sp.example.org/Shibboleth.sso/Metadata`

      ( *Replace `sp.example.org` with your SP Full Qualified Domain Name* )

### Configure an example federated resource "secure"

01. Create the Apache2 configuration for the application:

    - `sudo su -`

    - `vim /etc/apache2/conf-available/secure.conf`

      ```bash
      <Location /secure>
        Authtype shibboleth
        ShibRequireSession On
        require valid-user
      </Location>
      ```

    - `a2enconf secure`

02. Create the "`secure`" application into the DocumentRoot:

    ```bash
    mkdir -p /var/www/html/$(hostname -f)/secure

    wget https://github.com/ConsortiumGARR/idem-tutorials/raw/master/idem-fedops/HOWTO-Shibboleth/Service%20Provider/utils/index.php.txt -O /var/www/html/$(hostname -f)/secure/index.php
    ```

03. Install needed packages and restart Apache2:

    ```bash
    apt install libapache2-mod-php php

    systemctl restart apache2.service
    ```

### Enable Attribute Support on Shibboleth SP
>
> The Attribute Map file is used by the Service Provider to recognize and support new attributes released by an Identity Provider

Enable attribute support by removing comment from the related content into `/etc/shibboleth/attribute-map.xml` than restart `shibd` service with:

- `sudo systemctl restart shibd.service`

### Connect SP to the Federation
Later

### Connect SP directly to an IdP

> Follow these steps **IF** you need to connect one SP with only one IdP. It is useful for test purposes.

01. Edit `shibboleth2.xml` opportunely:

    - `vim /etc/shibboleth/shibboleth2.xml`

      ```bash

      <!-- If it is needed to manage the authentication on several IdPs
           install and configure the Shibboleth Embedded Discovery Service
           by following this HOWTO: https://url.garrlab.it/nakt7 
      -->
      <SSO entityID="https://idp.example.org/simplesaml/module.php/saml/idp/metadata">
         SAML2
      </SSO>
      <!-- ... other things ... -->
      <MetadataProvider type="XML" validate="true"
                        url="https://idp.example.org/simplesaml/module.php/saml/idp/metadata"
                        backingFilePath="idp-metadata.xml" maxRefreshDelay="7200" />
      ```

      ( *Replace `entityID` with the IdP entityID and `url` with an URL where it can be downloaded its metadata* )

      (`idp-metadata.xml` will be saved into `/var/cache/shibboleth`)

02. Restart `shibd` and `Apache2` daemon:

    - `sudo systemctl restart shibd`
    - `sudo systemctl restart apache2`


03. On the SimpleSAMLphp IdP, we need to add the SP metadata into the `metadata/saml20-sp-remote.php` file:

```bash
   cd /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/metadata/
   sudo cp saml20-sp-remote.php.dist /var/simplesamlphp/metadata/saml20-sp-remote.php
   cd /var/simplesamlphp/metadata/
   sudo vim saml20-sp-remote.php
   ```

   At the end of that `saml20-sp-remote.php` file, add the following content (as before, replace `https://sp.example.org/shibboleth` with your SP entityID - just change the example.org domain name with your SP FQDN):

```php
$metadata['https://sp.example.org/shibboleth'] = [
    'AssertionConsumerService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://sp.example.org/Shibboleth.sso/SAML2/POST',
        ],
    ],
    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://sp.example.org/Shibboleth.sso/SLO/POST',
        ],
    ],
    'NameIDFormat' => 'urn:oasis:names:tc:SAML:2.0:nameid-format:transient',
    'simplesaml.nameidattribute' => 'uid',
    'simplesaml.attributes' => true,
];
```

04. Jump to [Test](#test)

## Test

Open the `https://sp.example.org/secure` application into your web browser

(*Replace `sp.example.org` with your SP Full Qualified Domain Name*)

The above should redirect you to the IdP login page (please enter the username and password stored in the LDAP server). After successful authentication, you should be redirected back to the SP and see the attributes released by the IdP.


## Utility

- [The Mozilla Observatory](https://observatory.mozilla.org/):
  The Mozilla Observatory has helped over 240,000 websites by teaching developers, system administrators, and security professionals how to configure their sites safely and securely.

## Authors

### Original Author

- Marco Malavolti (<marco.malavolti@garr.it>)

## Thanks

- eduGAIN Wiki: For the original [How to configure Shibboleth SP attribute checker](https://wiki.geant.org/display/eduGAIN/How+to+configure+Shibboleth+SP+attribute+checker)
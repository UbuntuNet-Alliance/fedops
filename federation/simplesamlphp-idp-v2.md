# HOWTO Install and Configure a SimpleSAMLphp IdP v2.x on Debian-Ubuntu Linux with Composer

## Table of Contents

1. [Requirements](#requirements)
   1. [Hardware](#hardware)
   2. [Software](#software)
   3. [Other](#other)
2. [Notes](#notes)
3. [Configure the environment](#configure-the-environment)
4. [Configure APT Mirror](#configure-apt-mirror)
5. [Install Instructions](#install-instructions)
   1. [Install useful basic packages](#install-useful-basic-packages)
   2. [Install SimpleSAMLphp](#install-simplesamlphp)
6. [Configuration Instructions](#configuration-instructions)
   1. [Configure SSL on Apache2](#configure-ssl-on-apache2)
   2. [Configure SimpleSAMLphp](#configure-simplesamlphp)
   3. [Configure the Identity Provider](#configure-the-identity-provider)
      1. [Configure SAML Metadata Credentials](#configure-saml-metadata-credentials)
      2. [Configure Metadata](#configure-metadata)
      3. [Configure Attribute Release Policies](#configure-attribute-release-policies)
   4. [Configure the Directory (openLDAP or AD) Connection](#configure-the-directory-openldap-or-ad-connection)
   5. [Download IdP Metadata](#download-idp-metadata)
   6. [Register the IdP on the IDEM Test Federation](#register-the-idp-on-the-idem-test-federation)
7. [Appendix A - How to manage sessions with Memcached](#appendix-a---how-to-manage-sessions-with-memcached)
8. [Appendix B - How to collect useful statistics](#appendix-b---how-to-collect-useful-statistics)
9. [Appendix C - How to enable F-Ticks module](#appendix-c---how-to-enable-f-ticks-module)
10. [Appendix D - How to upgrade all modules](#appendix-d---how-to-upgrade-all-modules)
11. [Utility](#utility)
12. [Authors](#authors)
    1. [Original Author](#original-author)

## Requirements

### Hardware

* CPU: 2 Core (64 bit)
* RAM: 4 GB
* HDD: 20 GB
* OS: Ubuntu 24.04 (Noble)

[[TOC]](#table-of-contents)

### Software

* ca-certificates
* ntp
* vim
* apache2 (>= 2.4)
* php (>= 7.4 for SSP v2.0.x, >=8.0.0 for SSP v2.1.x)
* php extensions (date,dom,hash,intl,json,libxml,mbstring,openssl,pcre,SPL,zlib,ldap extensions)
* zip
* unzip
* composer
* memcached (optional)
* openssl
* cron
* curl
* git
* rsyslog
* logrotate

[[TOC]](#table-of-contents)

### Other

* SSL Credentials: HTTPS Certificate & Key (We will use Let's Encrypt)
* Logo (if you can):
  * size: 80x60 px (or other that respect the aspect-ratio)
  * format: PNG
  * style: with a transparent background
* Favicon (if you can):
  * size: 16x16 px (or other that respect the aspect-ratio)
  * format: PNG
  * style: with a transparent background

[[TOC]](#table-of-contents)

## Notes

This HOWTO uses `example.org` and `idp.example.org` to provide this guide with example values.

Please remember to **replace all occurencences** of the `example.org` value with the IdP domain name
and `idp.example.org` value with the Full Qualified Name of the Identity Provider.

For our Lab, if your participant ID is `01`, then your IdP hostname will be `idp-01` and your IdP Full Qualified Domain Name will be `idp-01.ubuntunet.org`. If your participant ID is `15`, then your IdP hostname will be `idp-15` and your IdP Full Qualified Domain Name will be `idp-15.ubuntunet.org`. And so on.

Your LDAP domain name is your institutional domain name. For example, if your institutional domain name is `botsren.org.bw`, then your LDAP domain name will be `botsren.org.bw` and your LDAP distinguished name will be `dc=botsren,dc=org,dc=bw`. If your institutional domain name is `zamren.zm`, then your LDAP domain name will be `zamren.zm` and your LDAP distinguished name will be `dc=zamren,dc=zm`. And so on.

[[TOC]](#table-of-contents)

## Configure the environment

1. Become ROOT:
   * `sudo su -`

2. Be sure that your firewall **is not blocking** the traffic on port **443** and **80** for the IdP server.

3. Set the IdP hostname:

   (**ATTENTION**: *Replace `idp.example.org` with your IdP Full Qualified Domain Name and `<HOSTNAME>` with the IdP hostname*)

   * `vim /etc/hosts`

     ```bash
     <YOUR SERVER IP ADDRESS> idp.example.org <HOSTNAME>
     ```

   * `hostnamectl set-hostname <HOSTNAME>`

[[TOC]](#table-of-contents)

## Configure APT Mirror

1.  Update packages:

    ``` text
    apt update && apt-get upgrade -y --no-install-recommends
    ```

[[TOC]](#table-of-contents)

## Install Instructions

### Install useful basic packages

1. Become ROOT:
   * `sudo su -`

2. Install useful packages:

   ```bash
   apt install vim wget ca-certificates openssl ntp fail2ban rsyslog logrotate --no-install-recommends
   ```

[[TOC]](#table-of-contents)

### Install SimpleSAMLphp

1. Become ROOT:
   * `sudo su -`

2. Prepare the environment:

   ```bash
   apt install git zip unzip apache2 php php-mbstring php-date php-intl php-xml php-curl libpcre3 libpcre3-dev zlib1g zlib1g-dev curl cron --no-install-recommends
   ```

3. Download Composer setup:
   * `wget "https://getcomposer.org/installer" -O /usr/local/src/composer-setup.php`

4. Install Composer:
   * `php /usr/local/src/composer-setup.php --install-dir=/usr/local/bin --filename=composer`

     **NOTE**: To update Composer use: `composer self-update`

5. Create the required directories:
   * `mkdir -p /var/simplesamlphp/cert /var/simplesamlphp/config /var/simplesamlphp/metadata /var/simplesamlphp/data`

6. Install SimpleSAMLphp:
   * `cd /var/simplesamlphp`
   * `composer require simplesamlphp/simplesamlphp --update-no-dev`
   * To the question "**Do you trust "simplesamlphp/composer-module-installer**" to execute code and wish to enable it now? (writes "allow-plugins" to composer.json) [y,n,d,?]" answer `y`

7. Load `config` and `metadata` configuration files into `/var/simplesamlphp`:
   ```bash
      cp /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/config/config.php.dist \
         /var/simplesamlphp/config/config.php

      cp /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/metadata/saml20-idp-hosted.php.dist \
         /var/simplesamlphp/metadata/saml20-idp-hosted.php
   ```

[[TOC]](#table-of-contents)

## Configuration Instructions

### Configure SSL on Apache2

1. Become ROOT:
   * `sudo su -`

2. Create the DocumentRoot:

   ```bash
   mkdir /var/www/html/$(hostname -f)

   chown -R www-data: /var/www/html/$(hostname -f)

   echo '<h1>It Works!</h1>' > /var/www/html/$(hostname -f)/index.html
   ```

3. Create the Virtualhost file (**please pay attention: you need to edit this file and customize it, check the internal initial comment**):

   ```bash
   curl -sL "https://api.github.com/repos/ConsortiumGARR/idem-tutorials/contents/idem-fedops/HOWTO-SimpleSAMLphp/Identity%20Provider/utils/idp.example.org.conf?ref=master" \
   | python3 -c "import sys,json,base64; print(base64.b64decode(json.load(sys.stdin)['content']).decode())" \
   | sudo tee /etc/apache2/sites-available/$(hostname -f).conf > /dev/null
   ```

   You need to edit the `/etc/apache2/sites-available/$(hostname -f).conf` file and replace all occurences of `idp.example.org` with your IdP Full Qualified Domain Name:

   ```apache
     DocumentRoot /var/www/html/idp.example.org
     ```

   ```apache
     SSLCertificateFile /etc/ssl/certs/idp.example.org.crt
     SSLCertificateKeyFile /etc/ssl/private/idp.example.org.key
   ```

   Uncomment the following line in the `/etc/apache2/sites-available/$(hostname -f).conf` file: **Remember to replace `ACME-CA.pem` with the CA certificate you are using for your IdP server. For this exercise, it is `hostname -f`-CA.pem an example being `idp-01.ubuntunet.org-CA.pem`**:

   ```apache
      SSLCACertificateFile /etc/ssl/certs/ACME-CA.pem
   ```

   Comment the following line in the `/etc/apache2/sites-available/$(hostname -f).conf` file:

   ```apache
      #SSLCACertificateFile /etc/ssl/certs/GEANT_TLS_RSA_1.pem
   ```



4. Setting Up Let's Encrypt (TLS)
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
   sudo certbot --apache -d idp-[participant_id].ubuntunet.org
   ```

   You will be prompted to:

   1. Enter an email address (used for renewal/expiry notifications).
   2. Agree to the Let's Encrypt Terms of Service.
   3. Optionally share your email with the EFF.
   4. Choose whether to redirect all HTTP traffic to HTTPS — **choose the redirect option (recommended)**, so `http://idp-[participant id].ubuntunet.org` automatically forwards to `https://`.

   If successful, Certbot will report where the certificate files were saved, typically:

      ```
      /etc/letsencrypt/live/idp-[participant_id].ubuntunet.org/fullchain.pem
      /etc/letsencrypt/live/idp-[participant_id].ubuntunet.org/privkey.pem
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
   * `a2ensite $(hostname -f).conf` - Enable SSP IdP VirtualHost
   * `a2dissite 000-default.conf default-ssl 000-default-le-ssl.conf` - Disable HTTP & HTTPS & LE default VirtualHost
   * `systemctl restart apache2.service`

7. Check that IdP works:
   * ht<span>tps://</span>idp.example.org/simplesaml

   There may be an error message, but we will fix it below:

   ```bash
      sudo mkdir -p /var/cache/simplesamlphp
      sudo chown -R www-data:www-data /var/cache/simplesamlphp
      sudo chmod -R 750 /var/cache/simplesamlphp
   ```

   The above command will create the cache directory and set the correct permissions for the web server to write to it. You can refresh the page and check if the error message is gone.

8. Verify the strength of your IdP's machine on:
   * [**https://www.ssllabs.com/ssltest/analyze.html**](https://www.ssllabs.com/ssltest/analyze.html)

[[TOC]](#table-of-contents)

### Configure SimpleSAMLphp

1. Become ROOT:
   * `sudo su -`

2. Generate secrets:
   * `<USER_ADMIN_PASSWORD>' (`auth.adminpassword`):
     * `php /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/bin/pwgen.php`

   * `<SECRET_SALT>` (`secretsalt`):
     * `tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null ; echo`

3. Change SimpleSAMLphp configuration:
   * `vim /var/simplesamlphp/config/config.php`

      ```php
      'baseurlpath' => 'simplesaml/',

      // ...other configuration settings...

      'loggingdir' => null,
      'datadir' => '/var/simplesamlphp/data/',
      'tempdir' => '/tmp/simplesaml',

      // ...other configuration settings...

      'certdir' => '/var/simplesamlphp/cert/',

      // ...other configuration settings...

      'technicalcontact_name' => 'Technical Contact',
      'technicalcontact_email' => 'technical.support@example.com',

      // ...other configuration settings...

      'secretsalt' => '<SECRET_SALT>',

      // ...other configuration settings...

      'auth.adminpassword' => '<USER_ADMIN_PASSWORD>',

      // ...other configuration settings...

      'logging.level' => 'SimpleSAML\Logger::NOTICE',
      'logging.handler' => 'syslog',

      // ...other configuration settings...

      'enable.saml20-idp' => true,

      // ...other configuration settings...

      'theme.header' = '<ORGANIZATION_NAME>',

      // ...other configuration settings...

      /*
       * Comment out line "50 => 'core:AttributeLimit'," into "authproc.idp" section
       * because we will use core:AttributeLimit into the "authproc" section on "metadata/saml20-idp-hosted.php"
      */
      
      // ...other configuration settings...

      'metadatadir' => '/var/simplesamlphp/metadata',

      // ...other configuration settings...

      'store.type' => 'phpsession',
      ```

   * `vim /etc/rsyslog.d/22-ssp-log.conf`

     ```bash
     # SimpleSAMLphp logging
     local5.*                        /var/log/simplesamlphp.log
     # Notice level is reserved for statistics only...
     local5.=notice                  /var/log/simplesamlphp.stat
     ```

   * `systemctl restart rsyslog.service`

4. Enable Log rotation for Statistics logs:

   * `sudo vim /etc/logrotate.d/simplesamlphp`

     ```bash
     /var/log/simplesamlphp.stat {
         monthly
         missingok
         rotate 12
         compress
         dateext
         dateformat .%Y-%m
         postrotate
             systemctl reload rsyslog
         endscript
     }
     ```

   * `sudo systemctl restart logrotate.service`

5. Create the `authsources.php` file:
   * `vim /var/simplesamlphp/config/authsources.php`

     ```php
     <?php

     $config = [

        // This is a authentication source which handles admin authentication.
        'admin' => [
            'core:AdminPassword',
        ],
     ];   
     ```

6. Install Consent module:
   * `composer require simplesamlphp/simplesamlphp-module-consent --update-no-dev`

7. Enable Consent module:
   * `vim /var/simplesamlphp/config/config.php`

     ```php
     // ...other configuration settings...
     
     'module.enable' => [
        'exampleauth' => false,
        'core' => true,
        'admin' => true,
        'saml' => true,
        'consent' => true,
     ],
     
     // ...other configuration settings...
     ```

[[TOC]](#table-of-contents)

### Configure the Identity Provider

#### Configure SAML Metadata Credentials

1. Become ROOT:
   * `sudo su -`

2. Generate `md-sign-enc-cert.crt` and `md-sign-enc-cert.key`:
   * `vim /var/simplesamlphp/cert/ssp-md-credentials.cnf`:

     (*Replace `idp.example.org` with your IDP Full Qualified Domain Name*)

     ```cnf
     [req]
     default_bits=4096
     default_md=sha256
     encrypt_key=no
     distinguished_name=dn
     # PrintableStrings only
     string_mask=MASK:0002
     prompt=no
     x509_extensions=ext

     # customize the "default_keyfile,", "CN" and "subjectAltName" lines below
     default_keyfile=md-sign-enc-cert.key

     [dn]
     CN=idp.example.org

     [ext]
     subjectAltName=DNS:idp.example.org, \
                    URI:https://idp.example.org/simplesaml/module.php/saml/idp/metadata
     subjectKeyIdentifier=hash
     ```

   * `cd /var/simplesamlphp/cert`
   * `openssl req -new -x509 -config ssp-md-credentials.cnf -out md-sign-enc-cert.crt -days 3650`
   * `chown -R www-data: /var/simplesamlphp/cert`
   * `chmod 400 /var/simplesamlphp/cert/md-sign-enc-cert.key`

[[TOC]](#table-of-contents)

#### Configure Metadata

1. Become ROOT:
   * `sudo su -`

2. Configure the IdP metadata:
   * `vim /var/simplesamlphp/metadata/saml20-idp-hosted.php`

     ```php
     <?php

      /**
      * SAML 2.0 IdP configuration for SimpleSAMLphp.
      *
      * See: https://simplesamlphp.org/docs/stable/simplesamlphp-reference-idp-hosted
      */

      $metadata['https://<IDP-FQDN>/simplesaml/module.php/saml/idp/metadata'] = [
         'host' => '__DEFAULT__',
         'privatekey' => 'md-sign-enc-cert.key',
         'certificate' => 'md-sign-enc-cert.crt',

         // Usually the scope(s) are the domain name(s) belonging to the institution
         'scope' => ['<IDP-SCOPE-1>', '<IDP-SCOPE-2>'],

         'OrganizationName' => [
            'en' => '<INSERT-HERE-THE-ENGLISH-ORGANIZATION-NAME>',
         ],
         'OrganizationDisplayName' => [
            'en' => '<INSERT-HERE-THE-ENGLISH-ORGANIZATION-DISPLAY-NAME>',
         ],
         'OrganizationURL' => [
            'en' => '<INSERT-HERE-THE-ENGLISH-ORGANIZATION-PAGE-URL>',
         ],

         // eduPersonTargetedID with oid NameFormat is a raw XML value
         'attributeencodings' => ['urn:oid:1.3.6.1.4.1.5923.1.1.1.10' => 'raw'],

         // The <LogoutResponse> message MUST be signed if HTTP-POST or Redirect binding is used
         'sign.logout' => true,

         'SingleLogoutServiceBinding' => [
            'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
         ],

         'NameIDFormat' => [
            'urn:oasis:names:tc:SAML:2.0:nameid-format:transient',
            'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent',
         ],

         'auth' => 'example-userpass',

         'authproc' => [
            // Generate the transient NameID.
            1 => [
                  'class' => 'saml:TransientNameID',
            ],

            // Generate the persistent NameID
            2 => [
                  'class' => 'saml:PersistentNameID',
                  'identifyingAttribute' => 'uid',
                  'NameQualifier' => true,
                  'SPNameQualifier' => true,
            ],

            // Add schacHomeOrganization for domain of entity
            10 => [
                  'class' => 'core:AttributeAdd',
                  'schacHomeOrganization' => '<INSERT-HERE-YOUR-DOMAIN-NAME>',
                  'schacHomeOrganizationType' => 'urn:schac:homeOrganizationType:eu:higherEducationalInstitution',
            ],

            // Add eduPersonPrincipalName
            11 => [
                  'class' => 'core:ScopeAttribute',
                  'scopeAttribute' => 'schacHomeOrganization',
                  'sourceAttribute' => 'uid',
                  'targetAttribute' => 'eduPersonPrincipalName',
            ],

            // Add eduPersonScopedAffiliation
            12 => [
                  'class' => 'core:ScopeAttribute',
                  'scopeAttribute' => 'eduPersonPrincipalName',
                  'sourceAttribute' => 'eduPersonAffiliation',
                  'targetAttribute' => 'eduPersonScopedAffiliation',
            ],

            // Add subject-id
            13 => [
                  'class' => 'saml:SubjectID',
                  'identifyingAttribute' => 'uid',
                  'scopeAttribute' => 'schacHomeOrganization',
            ],

            // Add pairwise-id
            14 => [
                  'class' => 'saml:PairwiseID',
                  'identifyingAttribute' => 'uid',
                  'scopeAttribute' => 'schacHomeOrganization',
            ],

            // Auto-generate eduPersonTargetedID/persistent nameID
            20 => [
                  'class' => 'saml:PersistentNameID2TargetedID',
                  'attribute' => 'eduPersonTargetedID',
                  'nameId' => true,
            ],

            // No attribute limiting — release everything the auth source provides.

            // Consent module, persisted via cookie
            90 => [
                  'class' => 'consent:Consent',
                  'identifyingAttribute' => 'uid',
                  'focus' => 'yes',
                  'checked' => true,
                  'store' => 'consent:Cookie',
                  'attributes.exclude' => ['uid'],
            ],

            91 => [
                  'class' => 'core:PHP',
                  'code' => 'unset($attributes["uid"]);',
            ],

            // If language is set in Consent module it's added as 'preferredLanguage'
            99 => 'core:LanguageAdaptor',

            // Convert LDAP names to oids needed to send attributes to the SP
            100 => ['class' => 'core:AttributeMap', 'name2oid'],
         ],
      ];
     ```

3. **NOTE**: Remember to Comment out the line "**50 => 'core:AttributeLimit',**" into "**authproc.idp**" section because we will use `core:AttributeLimit` into the "**authproc**" section on `metadata/saml20-idp-hosted.php` to limit the attributes released. If you keep the line *no attributes will be released*. You can comment out the line by adding a `//` at the beginning of this line in the file `/var/simplesamlphp/config/config.php`:
   ```bash
   sudo vim /var/simplesamlphp/config/config.php
   ```


   ```php
   // 50 => 'core:AttributeLimit',
   ```

[[TOC]](#table-of-contents)

### Configure the Directory (openLDAP) Connection

This builds on the OpenLDAP server that we did in the previous exercise. We will head straight to configuring the IdP to connect to the LDAP server.

1. Become ROOT:
   * `sudo su -`

2. Enable LDAP PHP module:
   * `apt install php-ldap --no-install-recommends`
   * `systemctl restart apache2.service`

3. Install the SimpleSAMLphp LDAP module:
   * `cd /var/simplesamlphp`
   * `composer require simplesamlphp/simplesamlphp-module-ldap --update-no-dev`

5. Add the `ldap:Ldap` Authentication Source:
   * `vim /var/simplesamlphp/config/authsources.php`

     **NOTE**:
     Replace the list provided into the `attributes` array with the attributes released by institutional LDAP/AD, <br/>
     and all `example` values with the correct one.

     ```php
     <?php

     $config = [

         // This is a authentication source which handles admin authentication.
         'admin' => [
             'core:AdminPassword',
         ],

         // LDAP authentication source.
         'ldap' => [
             'ldap:Ldap',
             'connection_string' => 'ldap://<LDAP-SERVER-FQDN>',
              'encryption' => 'none',
              'version' => 3,
              'ldap.debug' => true,
              'options' => [
                 /**
                  * Set whether to follow referrals.
                  * AD Controllers may require 0x00 to function.
                  * Possible values are 0x00 (NEVER), 0x01 (SEARCHING),
                  *   0x02 (FINDING) or 0x03 (ALWAYS).
                  */
                 'referrals' => 0x03,
                  'network_timeout' => 3,
             ],
             'connector' => '\SimpleSAML\Module\ldap\Connector\Ldap',
             // Pay attention on 'eduPersonTargetedID', 'eduPersonPrincipalName', 'eduPersonScopedAffiliation', 'schacHomeOrganization' and 'schacHomeOrganizationType'
             // Because they will be managed by the Authentication Process Filter inside metadata/saml20-idp-hosted.php
             // If you need to manage them directly on your Directory Service, remove the AuthProcFilter number 10,11,12,20 from metadata/saml20-idp-hosted.php
             'attributes' => ['uid','sn','givenName','cn','displayName','mail','eduPersonAffiliation','eduPersonEntitlement'],
             'search.filter' => '(&(objectClass=inetOrgPerson)(uid=%username%))',
             'dnpattern' => 'uid=%username%,ou=people,dc=example,dc=org',
             'search.enable' => false,
             'search.base' => [
                 'ou=people,dc=example,dc=org',
             ],
             'search.scope' => 'sub',
             'search.attributes' => ['uid'],
             'search.username' => '<LDAP-DN-OF-USER-THAT-PERFORMS-QUERIES-ON-DIRECTORY>', // cn=idpuser,ou=system,dc=example,dc=org
             'search.password' => '<QUERY-USER-PASSWORD>', // our bind/search password for LDAP
         ],
     ];
     ```

6. Connect LDAP to the IdP:
   * `vim /var/simplesamlphp/metadata/saml20-idp-hosted.php`

     ```php
        /* ...other things before end of file...*/
        'auth' => 'ldap',
     ];
     ```

7. Enable the SimpleSAMLphp LDAP module:
   * `vim /var/simplesamlphp/config/config.php`

     ```php
     /* ...other configuration settings...*/
     'module.enable' => [
        'exampleauth' => false,
        'core' => true,
        'admin' => true,
        'saml' => true,
        'consent' => true,
        'ldap' => true,
     ],
     /* ...other configuration settings...*/
     ```

8. Try the LDAP Authentication Source on:
   * `https://idp.example.org/simplesaml/module.php/admin/test`

     (*Replace `idp.example.org` with your IDP Full Qualified Domain Name*)

     You will be asked to the admin username and password if not logged in. After that, you will see the list of Authentication Sources. Click on the `ldap` link to test the LDAP authentication source. You will be prompted to enter a username and password. Enter the credentials of a user in your LDAP directory to test the connection. (*Remember our LDAP user with the password: hello-world-12*)

     When successful, you will see the attributes released by the LDAP server for that user. To this point, our IdP is working correctly and is able to authenticate users against the LDAP directory.

[[TOC]](#table-of-contents)

### Download IdP Metadata

* `https://idp.example.org/simplesaml/module.php/saml/idp/metadata`

  (*Replace `idp.example.org` with your IDP Full Qualified Domain Name*)

### Register the IdP on our Test Federation

We will come back to this step towards the end of the workshop. All we will need is the metadata URL for the IdP, the registry (Jaeger) and some extra configuration on the IdP to load the federation metadata.

## Authors

### Original Author

* Marco Malavolti (marco.malavolti@garr.it)

[[TOC]](#table-of-contents)
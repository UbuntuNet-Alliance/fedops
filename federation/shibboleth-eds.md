# HOWTO Install and Configure a Shibboleth Embedded Discovery Service

The Embedded Discovery Service (EDS) allows a Service Provider to run a discovery service within their own site. As such the discovery service can look like any other page on the site and thus not be as jarring to a user as being redirected to a totally different, third-party, discovery service site.
The EDS is a set of Javascript and CSS files, so installing it and using it is straight forward and does not require any additional software. Note: you must already have an installed and configured Shibboleth Service Provider, V2.4+, in order to use the EDS - We have already done this.

## Index

1. [Requirements](#requirements)
2. [Installation](#installation)
   1. [Debian/Ubuntu](#debianubuntu)
3. [Enable EDS on Shibboleth SP](#enable-eds-on-shibboleth-sp)
4. [Configuration](#configuration)
5. [Whitelist - How to allow IdPs to access the federated resource](#whitelist---how-to-allow-idps-to-access-the-federated-resource)
   1. [How to allow the access to IdPs by specifying their entityID](#how-to-allow-the-access-to-idps-by-specifying-their-entityid)
   2. [How to allow the access to IdPs that support a specific Entity Category](#how-to-allow-the-access-to-idps-that-support-a-specific-entity-category)
   3. [How to allow the access to IdPs that support SIRTFI](#how-to-allow-the-access-to-idps-that-support-sirtfi)
6. [Blacklist - How to disallow IdPs to access the federated resource](#blacklist---how-to-disallow-idps-to-access-the-federated-resource)
   1. [How to disallow the access to IdPs by specifying their entityID](#how-to-disallow-the-access-to-idps-by-specifying-their-entityid)
   2. [How to disallow the access to IdPs that support a specific Entity Category](#how-to-disallow-the-access-to-idps-that-support-a-specific-entity-category)
7. [Connect the SimpleSAMLphp IdP to the Federation](#connect-the-simplesamlphp-idp-to-the-federation)
8. [Testing](#testing)
9. [Authors](#authors)
10. [Credits](#credits)

## Requirements

* Apache Server (>= 2.4)
* A working Shibboleth Service Provider (>= 2.4)
* Tested on: Debian/Ubuntu

## Installation

### Debian/Ubuntu

1. ```bash
   sudo su -

   cd /usr/local/src

   wget https://shibboleth.net/downloads/embedded-discovery-service/latest/shibboleth-embedded-ds-1.4.0.tar.gz -O shibboleth-eds.tar.gz

   tar xzf shibboleth-eds.tar.gz

   cd shibboleth-embedded-ds-1.4.0

   apt install make

   make install
   ```

2. Enable Discovery Service Web Page
   * `mv /etc/shibboleth-ds/shibboleth-ds.conf /etc/apache2/conf-available/shibboleth-ds.conf`

3. Enable the Discovery Service Page:
   * `a2enconf shibboleth-ds.conf`

4. Restart Apache to load the new web site:
   * `systemctl restart apache2.service`

## Enable EDS on Shibboleth SP

1. Update "`shibboleth2.xml`" file to point to the new Discovery Service page:
   * `vim /etc/shibboleth/shibboleth2.xml`

     ```xml
     <SSO discoveryProtocol="SAMLDS"
          discoveryURL="https://###YOUR.SP.FQDN###/shibboleth-ds/index.html">
        SAML2
     </SSO>

     <!-- SAML and local-only logout. -->
     <Logout>SAML2 Local</Logout>

     <!-- ...other things ... -->

     <!-- JSON feed of discovery information. -->
     <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>
     ```

2. Restart "**shibd**" service:

   * `systemctl restart shibd.service`

## Configuration

The behaviour of Shibboleth Embedded Discovery Service is controlled by the `IdPSelectUIParms` class contained in `/etc/shibboleth-ds/idpselect_config.js`.

In most cases you only need to modify this file to change the behaviour of the Discovery Service.

Make sure to amend `this.redirectAllow` to reflect your server name. Replace `example.org` with the FQDN of your Shibboleth Service Provider. For example, my SP FQDN is `sp-f01.cranecloud.africa`, so I have set:

```js
this.redirectAllow = [ "^https:\/\/sp-f01\.cranecloud\.africa\/Shibboleth\.sso\/Login.*$" ];
```

If you need to allow redirects to more than one host or path (e.g. a second SP, or a separate logout endpoint), add additional patterns as extra entries in the array rather than repeating the same one - separated by a coma. 

Find here the EDS Configuration Options: https://wiki.shibboleth.net/confluence/display/EDS10/3.+Configuration

## Whitelist - How to allow IdPs to access the federated resource

I have put some sample IdPs in the whitelist/blacklist, but you can add/remove them as you wish. You can also add/remove IdPs by specifying their entityID.

### How to allow the access to IdPs by specifying their entityID

DOC: [https://shibboleth.atlassian.net/wiki/spaces/SP3/pages/2063696201/IncludeMetadataFilter](https://shibboleth.atlassian.net/wiki/spaces/SP3/pages/2063696201/IncludeMetadataFilter)

Now remember we had set up a 1-1 SP-IdP relationship, but this discovery service exercise allows us to relax the relationship for multiple entities. In the file `/etc/shibboleth/shibboleth2.xml` please comment out the `MetadataProvider` line we had earlier configured. An example from my side commented out:

```xml
<!--    <MetadataProvider type="XML" validate="true"
                url="https://idp-f01.cranecloud.africa/simplesaml/module.php/saml/idp/metadata"
                backingFilePath="idp-metadata.xml" maxRefreshDelay="7200" /> -->
```

We can now proceed with the new `MetadataProvider` configuration:

1. Modify "**shibboleth2.xml**":
   * `vim /etc/shibboleth/shibboleth2.xml`

      ```xml
      <MetadataProvider type="XML"
                        url="https://registry.eduid.africa/metadata/federation/Systems_Kampala/metadata.xml"
                        backingFilePath="sysk-metadata.xml">
         <MetadataFilter type="RequireValidUntil" maxValidityInterval="864000" />
         <MetadataFilter type="Include">
             <Include>https://idp-04.ubuntunet.org/simplesaml/module.php/saml/idp/metadata</Include>
             <Include>https://idp-dev.ethernet.edu.et/simplesaml/module.php/saml/idp/metadata</Include>
             <Include>https://idp-06.ubuntunet.org/simplesaml/module.php/saml/idp/metadata</Include>
             <Include>https://idp-f01.cranecloud.africa/simplesaml/module.php/saml/idp/metadata</Include>
         </MetadataFilter>
      </MetadataProvider>
      ```

2. Restart "**shibd**" service:

   * `systemctl restart shibd.service`

## Blacklist - How to disallow IdPs to access the federated resource

### How to disallow the access to IdPs by specifying their entityID

DOC: [https://shibboleth.atlassian.net/wiki/spaces/SP3/pages/2063696198/ExcludeMetadataFilter](https://shibboleth.atlassian.net/wiki/spaces/SP3/pages/2063696198/ExcludeMetadataFilter)

1. Modify "**shibboleth2.xml**":

   * `vim /etc/shibboleth/shibboleth2.xml`

       ```xml
       <MetadataProvider type="XML"
                         url="https://registry.eduid.africa/metadata/federation/Systems_Kampala/metadata.xml"
                         backingFilePath="sysk-metadata.xml">
          <MetadataFilter type="RequireValidUntil" maxValidityInterval="864000" />
          <MetadataFilter type="Exclude">
              <Exclude>https://idp-05.ubuntunet.org/simplesaml/module.php/saml/idp/metadata</Exclude>
          </MetadataFilter>
       </MetadataProvider>
       ```

   > Note: unlike the `Include` filter, the `Exclude` filter's child elements must be `<Exclude>`, not `<Include>` — using `<Include>` here will not exclude anything.

2. Restart "**shibd**" service:

   * `systemctl restart shibd.service`

## Connect the SimpleSAMLphp IdP to the Federation

The SimpleSAMLphp IdP can also load Service Provider metadata directly from the federation instead of maintaining a manually curated `saml20-sp-remote.php` file. The registry publishes an aggregate metadata file per federation rather than a per-entity query endpoint, so the standard way to consume it is the `metarefresh` module: it periodically downloads the aggregate, converts it into flatfile metadata, and SimpleSAMLphp loads it like any other metadata source.

We are using the unsigned aggregate for the **Systems_Kampala** federation, so no signature validation is configured below. If you later switch to a signed feed, add a `certificates` entry pointing at the federation's signing certificate.

### 1. Enable the metarefresh module

```bash
    sudo -i
    composer require simplesamlphp/simplesamlphp-module-metarefresh

```

Open/Edit the `/var/simplesamlphp/config/config.php` and enable the `cron` and `metarefresh` modules 

```bash
    vim /var/simplesamlphp/config/config.php
```

Notice the two lines added and as shown below:

```
    'module.enable' => [
        'exampleauth' => false,
        'core' => true,
        'admin' => true,
        'saml' => true,
        'consent' => true,
        'ldap' => true,
        'cron' => true,
        'metarefresh' => true,
    ],
```

### 2. Configure the metarefresh source

Copy the configuration template files to the main config directory.

```bash
    cd ~ 
    # Copy the Cron config file
    cp /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/modules/cron/config/module_cron.php.dist /var/simplesamlphp/config/module_cron.php

    # Copy the Metarefresh config file
    cp vendor/simplesamlphp/simplesamlphp/modules/metarefresh/config-templates/module_metarefresh.php /var/simplesamlphp/config/module_metarefresh.php

    cd /var/simplesamlphp/

```

* `vim  /var/simplesamlphp/config/module_metarefresh.php`

At the end of that file just before the last closing `];`, please insert the following:

  ```php
          'systems_kampala' => [
              'cron' => ['hourly'],
              'sources' => [
                  [
                      'src' => 'https://registry.eduid.africa/metadata/federation/Systems_Kampala/metadata.xml',
                      // No 'certificates' entry: we are not validating a signature on
                      // this feed, since we're using the unsigned aggregate.
                  ],
              ],
              'expireAfter'  => 60 * 60 * 24 * 4, // drop entities not seen for 4 days
              'outputDir'    => 'metadata/metarefresh-systems_kampala/',
              'outputFormat' => 'flatfile',
          ],
  ```

### 3. Point `config.php` at the metarefresh output

* `vim /var/simplesamlphp/config/config.php`

You can edit out the following section:

```php
    'metadata.sources' => [
    ['type' => 'flatfile'],
    ],
```

with the following:

  ```php
  'metadata.sources' => [
      ['type' => 'flatfile'],
      ['type' => 'flatfile', 'directory' => 'metadata/metarefresh-systems_kampala'],
  ],
  ```

### 4. Create the output directory

```bash
sudo mkdir -p /var/simplesamlphp/metadata/metarefresh-systems_kampala
sudo chown www-data /var/simplesamlphp/metadata/metarefresh-systems_kampala
```

### 5. Schedule the refresh

Metarefresh only updates when it's run. Enable the `cron` module, set a cron secret in `config.php`, and schedule a job on the same tag used above (`hourly`):

```bash
touch /var/simplesamlphp/modules/cron/enable
```

```
0 * * * * curl -s "https://idp-f01.cranecloud.africa/simplesaml/module.php/cron/cron.php?key=<your-cron-secret>&tag=hourly" > /dev/null
```

### 6. Clean up `/metadata`

Check whether `https://sp-f01.cranecloud.africa/shibboleth` is already published in the Systems_Kampala federation metadata. If it is, the manual entry in `saml20-sp-remote.php` becomes redundant — and having the same entityID in two metadata sources at once has undefined precedence. Keep both only long enough to confirm the federation copy resolves correctly, then remove the manual block:

```bash
mkdir /var/simplesamlphp/metadata.old
mv /var/simplesamlphp/metadata/saml20-sp-remote.php /var/simplesamlphp/metadata.old/
```

Leave `saml20-idp-hosted.php` in place — that defines your own IdP, not remote SP metadata.

## Testing

You can now visit `https://<SP-FQDN>/secure` to access the secured resource. You will be presented with the discovery service page that allows you to select an Identity Provider for use with the service provider.

## Authors

### Original Author

* Marco Malavolti (marco.malavolti@garr.it)

## Credits

* [Consortium Shibboleth](https://shibboleth.net/)

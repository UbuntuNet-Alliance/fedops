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
    cd /var/simplesamlphp/
    composer require simplesamlphp/simplesamlphp-module-metarefresh --update-no-dev

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

Copy the config template, then **replace its entire contents** — don't just append, and don't leave the module's sample `kalmar` set in there (it points at an unrelated feed and references certificate files that won't exist on your box, which will throw errors on every run once other sets are present):

```bash
cp /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/modules/metarefresh/config-templates/module_metarefresh.php /var/simplesamlphp/config/module_metarefresh.php
```

* `vim /var/simplesamlphp/config/module_metarefresh.php`

```php
<?php

$config = [
    'sets' => [
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
    ],
];
```

**Before doing anything else, check it actually parses:**

```bash
php -l /var/simplesamlphp/config/module_metarefresh.php
```

It must print `No syntax errors detected`. A broken array here fails silently from the outside (cron returns HTTP 200 either way) and only shows up as a `ConfigurationError` in the log — worth catching now rather than after several rounds of "why is the output directory still empty."

> ⚠️ **`outputDir` is relative to the core library's own directory, not `/var/simplesamlphp`.** On this install, `'outputDir' => 'metadata/metarefresh-systems_kampala/'` resolves to `/var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/metadata/metarefresh-systems_kampala/` — **not** `/var/simplesamlphp/metadata/metarefresh-systems_kampala/`. Confirm this on your own install (see the verification step below) rather than assuming — this is exactly the kind of thing that varies with how SimpleSAMLphp's `basedir` is resolved on a given Composer setup, and getting it wrong means the fetch succeeds but the write silently lands somewhere you're not looking.

### 3. Point `config.php` at the metarefresh output

* `vim /var/simplesamlphp/config/config.php`

```php
'metadata.sources' => [
    ['type' => 'flatfile'],
    ['type' => 'flatfile', 'directory' => 'metadata/metarefresh-systems_kampala'],
],
```

### 4. Create the output directory — using the *real* resolved path

```bash
mkdir -p /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/metadata/metarefresh-systems_kampala
chown -R www-data:www-data /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/metadata
```

Owning it as `www-data` matters because the refresh runs as the web server user when triggered over HTTP (see below) — a root-owned directory here will fail with "Error creating directory" the moment metarefresh tries to write to it.

### 5. Schedule the refresh

Metarefresh only updates when something actually triggers it — enabling the module doesn't schedule anything by itself. That job belongs to the `cron` module.

**a. Copy and configure the cron module's config file**

```bash
cp /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/modules/cron/config/module_cron.php.dist /var/simplesamlphp/config/module_cron.php
```

Generate a random key:

```bash
tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null ; echo
```

* `vim /var/simplesamlphp/config/module_cron.php`

```php
<?php

$config = [
    'key'           => '#_YOUR_RANDOM_CRON_KEY_#',
    'allowed_tags'  => ['daily', 'hourly', 'frequent'],
    'debug_message' => true,
    'sendemail'     => false,
];
```

Three things to get right here:
* `allowed_tags` must include the tag used on the `systems_kampala` set (`'cron' => ['hourly']`). If `hourly` isn't listed, cron fires but silently skips that set — no error, it just never updates.
* `key` authorizes the trigger over HTTP. Treat it like a password: long, random, never committed to version control.
* Set `sendemail` to **`false`** unless this box has a working local MTA. With it `true`, every run logs `Unable to send cron report; Could not instantiate mail function` — harmless, but noisy, and it doesn't affect whether the refresh itself works.

**b. Also raise the log level while getting this working**

Metarefresh's per-source status ("Executing set...", "loading source...", any fetch/write errors) logs at `DEBUG`, one level below what most default installs capture. Without this, cron can run cleanly with no visible error at all while metarefresh fails silently underneath it:

* `vim /var/simplesamlphp/config/config.php`

```php
'logging.level' => \SimpleSAML\Logger::DEBUG,
```

(Dial this back to something quieter, like `NOTICE`, once things are confirmed working — DEBUG is chatty for normal operation.)

**c. Schedule the trigger**

```bash
crontab -e
```

```
0 * * * * curl -sS "https://idp.example.org/simplesaml/module.php/cron/run/hourly/<YOUR_RANDOM_CRON_KEY>" > /dev/null
```

### 6. Verify it's actually working

Trigger it once by hand rather than waiting an hour:

```bash
curl -sS "https://idp.example.org/simplesaml/module.php/cron/run/hourly/<YOUR_RANDOM_CRON_KEY>"
```

An empty response here is normal — the cron endpoint doesn't render a page, so no output doesn't mean failure. The real checks:

```bash
ls -l /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/metadata/metarefresh-systems_kampala/
```

You should see `.php` files (e.g. `saml20-idp-remote.php`). If the directory is empty, check syslog/`journalctl` for `metarefresh` and `Cron - Summary` lines (this is why step 5b raised the log level):

```bash
grep -i -E 'metarefresh|cron - summary' /var/log/syslog | tail -n 30
```

**Check what entity types actually came through.** Metarefresh splits output by type — you may only get `saml20-idp-remote.php`, only `saml20-sp-remote.php`, or both, entirely depending on what the aggregate contains. On the Systems_Kampala feed, only IdP entities came through (`saml20-idp-remote.php`); no SPs. List what's in there:

```bash
grep -o "metadata\['[^']*'\]" /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/metadata/metarefresh-systems_kampala/saml20-idp-remote.php
```

Finally, confirm the entities are actually being picked up by SimpleSAMLphp itself (not just present as files) via **Admin → Federation**:

```
https://<idp.example.org>/simplesaml/module.php/admin/federation
```

## Testing

You can now visit `https://<SP-FQDN>/secure` to access the secured resource. You will be presented with the discovery service page that allows you to select an Identity Provider for use with the service provider.

## Authors

### Original Author

* Marco Malavolti (marco.malavolti@garr.it)

## Credits

* [Consortium Shibboleth](https://shibboleth.net/)

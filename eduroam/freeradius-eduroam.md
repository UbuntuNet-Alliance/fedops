# Lab 1 - Install & Configure FreeRADIUS for eduroam

In this lab, each group will set up an **institutional RADIUS** server using 
FreeRADIUS for a realm of your choice. You will first configure and test 
authentication for your institution's users locally.

You will then connect your RADIUS server to the workshop federation and work with 
another group to test **cross-institution authentication**. The federation uses 
two facilitator-managed **radsecproxy** servers to route authentication requests 
between participating institutions.

---

## Prerequisites

- One Linux server per group - **Ubuntu 24.04 (Noble)** is the target for this
  guide (the FreeRADIUS 3.2 repo in Part 1 is the `noble` build). On another
  release, swap `noble` in the repository URL for your codename.
- `sudo`/root access and outbound internet for package installation.
- **UDP ports 1812 and 1813** reachable between your server and the federation
  servers (check any local firewall / security group).
- Comfort with a terminal and a text editor (`nano`, `vim`).
- Work in **groups of two**, ideally two engineers from the same NREN.
- Software installed during the lab: `freeradius`, `freeradius-utils`, and the
  EAP test tools `eapol_test` / `rad_eap_test`.

---

## Lab Environment

**Two layers:**

- **Institution servers** - one per group, built by participants.
- **Federation servers (FLR1, FLR2)** - two radsecproxy relays run by the
  facilitators. They route requests between realms.

```
        Group A                 Group B
   institution RADIUS      institution RADIUS
   realm: renu.ac.ug        realm: mak.ac.ug
   (FreeRADIUS)             (FreeRADIUS)
        |                        |
        |   RADIUS/UDP 1812      |
        |   (shared secret)      |
        +-----------+------------+
                    |
             Federation layer
         radsecproxy FLR1 + FLR2
         (facilitators, pre-set)
```

Your server authenticates **your** realm locally and **proxies everything else**
to the federation (FLR1, with FLR2 as backup). The federation **routes by realm**
back to the correct institution server.

**Choose a realm** for your group — something plausible and unique, e.g.
`renu.ac.ug`, `mak.ac.ug`, `must.ac.ug`. It does not have to be a domain you own,
but it must be unique among the groups so routing does not collide.

**Parameters** - fill in the values the facilitators give you; every command
below refers back to these:

| Name | Meaning | Example | Your value |
|------|---------|---------|-----------|
| `MY_REALM` | Your group's realm | `renu.ac.ug` | |
| `MY_IP` | Your institution server's IP | `<YOUR_IP_ADDRESS` | |
| `FLR1_IP` | Federation server 1 | `196.32.212.213` | |
| `FLR2_IP` | Federation server 2 | `196.32.212.220` | |
| `FED_SECRET` | Shared secret to the federation | `eduroam-lab-2026` | |
| `TEST_USER` | Local test user (with your realm) | `testuser@renu.ac.ug` | |
| `TEST_PASS` | Test user password | `Test1234` | |

> `FED_SECRET` is the same on your server and on the federation servers - that is
> what "we share the secrets in the instructions" means. Lab-only; never reuse a
> workshop secret in production.

---

## Part 1: Install FreeRADIUS

We install **FreeRADIUS 3.2** from the official InkBridge Networks /
NetworkRADIUS repository (the Ubuntu archive ships an older build).

### 1.1 Add the NetworkRADIUS repository

Add the packages PGP public key:

```bash
sudo install -d -o root -g root -m 0755 /etc/apt/keyrings
curl -s 'https://packages.inkbridgenetworks.com/pgp/packages.networkradius.com.asc' | \
    sudo tee /etc/apt/keyrings/packages.networkradius.com.asc > /dev/null
```

Pin all `freeradius` packages to the InkBridge repository:

```bash
printf 'Package: /freeradius/\nPin: origin "packages.inkbridgenetworks.com"\nPin-Priority: 999\n' | \
    sudo tee /etc/apt/preferences.d/networkradius > /dev/null
```

Add the APT sources list (Ubuntu 24.04 "noble"):

```bash
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/packages.networkradius.com.asc] http://packages.inkbridgenetworks.com/freeradius-3.2/ubuntu/noble noble main" | \
    sudo tee /etc/apt/sources.list.d/inkbridge.list > /dev/null
```

### 1.2 Install the packages

```bash
sudo apt update
sudo apt install -y freeradius freeradius-utils freeradius-ldap
sudo apt install build-essential libssl-dev libnl-3-dev libnl-genl-3-dev
```

### 1.3 Stop the service

Confirm that the freeradius service is running before you stop it
```bash
sudo systemctl status freeradius
```

We run the server in the foreground (debug mode) instead, which is far easier to
troubleshoot:

```bash
sudo systemctl stop freeradius
```
Confirm that the freeradius service is not running before you proceed
```bash
sudo systemctl status freeradius
```

### 1.4 Verify

```bash
freeradius -v            # should report 3.2.x
ls /etc/freeradius/
```

You should see `clients.conf`, `proxy.conf`, `mods-enabled/`, `sites-enabled/`
and `mods-config/`.

> **Config path:** on the NetworkRADIUS 3.2 package the configuration lives
> directly under **`/etc/freeradius/`**. All paths in this guide use `/etc/freeradius/`. 
---

## Part 2: Configure FreeRADIUS

Before configuring eduroam, verify that the FreeRADIUS server can successfully authenticate a local user.

### 2.1 Add a local test user

Add a test user to the users file. Replace the example realm with your group's realm.
```bash
sudo sed -i '1i testuser@renu.ac.ug   Cleartext-Password := "Test1234"' /etc/freeradius/mods-config/files/authorize
```

> Cleartext is used here only so every EAP method works in the lab. In production
> you would authenticate against LDAP/AD or SQL instead — see the
> "Protocol & Password Compatibility" slide for why the storage format matters.

### 2.2 Set the default EAP type

FreeRADIUS ships with self-signed **test certificates**, which are all we need for
Lab 1. But on this package the default outer EAP type is **`md5`**, which will not
work for eduroam — we need **PEAP**. Check what is currently set:

```bash
grep -n "default_eap_type" /etc/freeradius/mods-enabled/eap
```

You will see something like:

```
27:      default_eap_type = md5        <-- the main (outer) type: change this
992:             default_eap_type = mschapv2   <-- inside peap{ }: leave as-is
```

Edit the file and change the **first / top-level** `default_eap_type` (the one in
the main `eap { ... }` block, near line 27) from `md5` to `peap`:

```bash
sudo nano /etc/freeradius/mods-enabled/eap
```

```
eap {
    default_eap_type = peap
    ...
}
```

Leave the `default_eap_type = mschapv2` **inside** the `peap { }` block unchanged —
that correctly selects MSCHAPv2 as the method carried inside the PEAP tunnel.

> The bundled certs are for labs only. Real eduroam requires a proper server
> certificate and clients that validate it.

### 2.3 Confirm the localhost test client

`clients.conf` already defines a `localhost` client with secret `testing123`:

```bash
grep -n -A3 "client localhost" /etc/freeradius/clients.conf
```
You will see something like:

```
49:client localhost {
50-     #  Only *one* of ipaddr, ipv4addr, ipv6addr may be specified for
51-     #  a client.
52-     #
--
335:client localhost_ipv6 {
336-    ipv6addr        = ::1
337-    secret          = testing123
338-}
```
We use it for local testing. No change needed yet.

### 2.4 Test Local Authentication
Start FreeRADIUS in debug mode:

```bash
sudo freeradius -X
```
Leave FreeRADIUS running in this terminal. Open a second terminal to perform the authentication test.

### 2.4 Test The Local User
From the second terminal, test the user you created earlier
```bash
radtest testuser@renu.ac.ug Test1234 127.0.0.1 0 testing123
```
Replace renu.ac.ug with your group's realm.

A successful authentication should return something like this 
```
Sent Access-Request Id 40 from 0.0.0.0:60486 to 127.0.0.1:1812 length 89
        User-Name = "testuser@renu.ac.ug"
        User-Password = "Test1234"
        NAS-IP-Address = 192.168.21.219
        NAS-Port = 0
        Message-Authenticator = 0x00
        Cleartext-Password = "Test1234"
Received Access-Accept Id 40 from 127.0.0.1:1812 to 127.0.0.1:60486 length 38
        Message-Authenticator = 0x13e082fe52af7540c55de8261f43cf23
```
**Checkpoint**: Do not proceed to the eduroam configuration until local authentication is successful.
---

## Part 3: Configure eduroam

### 3.1 Configure the eduroam Virtual Servers
Remove the existing virtual server configurations from the `sites-enabled` directory:

```bash
sudo rm -f /etc/freeradius/sites-enabled/*
```

#### 3.1.1 Create and open the `eduroam` virtual server configuration:
```bash
sudo nano /etc/freeradius/sites-available/eduroam
```
Copy the provided eduroam configuration into the file, then save and exit.

#### 3.1.2 Create and open the eduroam-inner-tunnel configuration file:
```bash
sudo nano /etc/freeradius/sites-available/eduroam-inner-tunnel
```
Copy the provided `eduroam-inner-tunnel` configuration into the file, then save and exit.

#### 3.1.3 Enable the virtual servers
Create symbolic links from `sites-available` to `sites-enabled` to enable both virtual servers:

```bash
sudo ln -s /etc/freeradius/sites-available/eduroam /etc/freeradius/sites-enabled/eduroam 
sudo ln -s /etc/freeradius/sites-available/eduroam-inner-tunnel  /etc/freeradius/sites-enabled/eduroam-inner-tunnel
```

### 3.2 Configure `proxy.conf`

Next, configure the RADIUS server to forward authentication requests for non-local realms to the **Federation-Level RADIUS (FLR) server**.

This allows the institutional RADIUS server to route authentication requests for visiting eduroam users to the federation.
```bash
sudo mv /etc/freeradius/proxy.conf /etc/freeradius/proxy.conf.original
sudo touch /etc/freeradius/proxy.conf
sudo nano /etc/freeradius/proxy.conf
```
Copy the contents of proxy.conf file to the new file created

### 3.3 Configure `clients.conf`

Next, configure the RADIUS clients that are allowed to communicate with your FreeRADIUS server.

Back up the existing `clients.conf` file:

```bash
sudo mv /etc/freeradius/clients.conf /etc/freeradius/clients.conf.original
```
Create and open a new clients.conf file:
```bash
sudo nano /etc/freeradius/clients.conf
```
Copy the configuration provided in the `clients.conf` lab file into this file, then save and exit.

### 3.4 Configure Certificates

FreeRADIUS includes default certificate configuration files during installation. Before modifying these files, create a backup of the certificate directory so that the original configuration can be restored if necessary.

Back up the existing certificate directory:

```bash
sudo cp -a /etc/freeradius/certs /etc/freeradius/certs.backup
```

For this lab, you will modify the following certificate configuration files:
`ca.cnf`
`server.cnf`
`inner-server.cnf`
`client.cnf`

#### 3.4.1 Configure `ca.cnf`
Open the CA configuration file:
```bash
sudo nano /etc/freeradius/certs/ca.cnf
```

Locate the following lines:
```
countryName             = FR
stateOrProvinceName     = Radius
localityName            = Somewhere
organizationName        = Example Inc.
emailAddress            = admin@example.org
commonName              = "Example Certificate Authority"
```
Update these values with the details of the institution your group is using for the lab.

For example:
```
countryName             = UG
stateOrProvinceName     = Kampala
localityName            = Kololo
organizationName        = RENU
emailAddress            = bnamuli@renu.ac.ug
commonName              = "RENU Certificate Authority"
```
Replace the example values as follows:
- `countryName` — use the two-letter country code for your institution.
- `stateOrProvinceName` — use your state, province, or region.
- `localityName` — use your city or locality.
- `organizationName` — use the name of your institution.
- `emailAddress` — use an appropriate email address for your institution.
- `commonName` — use a descriptive name for your institution's Certificate Authority.

Save the changes and exit the file:

#### 3.4.2 Configure `client.cnf`
```bash
sudo nano /etc/freeradius/certs/client.cnf
```

Locate the corresponding certificate information and update it with the details of your institution, 
following the same approach used for `ca.cnf`.

Save the changes and exit the file.

#### 3.4.3 Configure `inner-server.cnf`
```bash
sudo nano /etc/freeradius/certs/inner-server.cnf
```

Locate the corresponding certificate information and update it with the details of your institution, 
following the same approach used for `ca.cnf`.

Save the changes and exit the file.

#### 3.4.4 Configure `server.cnf`
Open the server certificate configuration file:
```bash
sudo nano /etc/freeradius/certs/server.cnf
```
Update the certificate information with your institution's details. For the `commonName`, use the hostname
 of your RADIUS server.

For example:
```
countryName             = UG
stateOrProvinceName     = Kampala
localityName            = Kololo
organizationName        = RENU
emailAddress            = bnamuli@renu.ac.ug
commonName              = "freeradius.renu.ac.ug"
```

Replace the example values with your group's institution details and use your RADIUS server hostname for `commonName`.

For example, if your group's realm is example.ac.ug and your RADIUS server is `radius.example.ac.ug`, use:
```
commonName = "radius.example.ac.ug"
```

Save the changes and exit the file.

#### 3.4.5 Configure the input and output password
The certificate configuration files use `input_password` and `output_password` to protect generated private keys. Replace the default password whatever with a randomly generated password.

Generate a random password
```bash
PASSWORD=$(openssl rand -base64 24)
```

Update the input_password and output_password values in all certificate configuration files:
```bash
sudo sed -i -e "s|^[[:space:]]*input_password[[:space:]]*=.*|input_password = \"$PASSWORD\"|" -e "s|^[[:space:]]*output_password[[:space:]]*=.*|output_password = \"$PASSWORD\"|" /etc/freeradius/certs/*.cnf
```

Verify the changes: 
```bash
grep -H -E '^[[:space:]]*(input_password|output_password)' /etc/freeradius/certs/*.cnf
```

#### 3.4.6 Generate the certificates
Once the certificate configuration is complete, update the ownership of the FreeRADIUS configuration directory:
```bash
sudo chown -R freerad:freerad /etc/freeradius/
```

Navigate to the FreeRADIUS certificate directory:
```bash
cd /etc/freeradius/certs/
```

Generate the certificates:

```bash
sudo make
```

Verify that the certificates were generated successfully:
```bash
ls -l *.pem
```
Ensure that the certificate generation completes without errors before proceeding.

#### 3.4.6 Configure the EAP Module
Update the EAP module so that FreeRADIUS uses the same private key password that was configured in the certificate `.cnf` files.

Open the EAP configuration file:
Update the private_key_password value in the eap module configuration file:
```bash
sudo sed -i "s|^[[:space:]]*private_key_password[[:space:]]*=.*|private_key_password = $PASSWORD|" /etc/freeradius/mods-available/eap
```

Next, update the virtual server used by the ttls and peap sections.
```bash
sudo sed -i 's|virtual_server = "inner-tunnel"|virtual_server = "eduroam-inner-tunnel"|g' /etc/freeradius/mods-available/eap
```

Verify the changes:
```bash
grep -E 'private_key_password|virtual_server' /etc/freeradius/mods-available/eap
```

Ensure that:
- `private_key_password` matches the password used when generating the certificates.
- Both the `ttls` and `peap` sections reference `eduroam-inner-tunnel`.


## Part 4: Configure LDAP
In this part, you will configure FreeRADIUS to use LDAP as the identity store for authenticating institutional users.

Before proceeding, ensure that you have the LDAP server address, Base DN, bind DN, and bind password.

### 4.1 Enable the LDAP Module
Enable the FreeRADIUS LDAP module by creating a symbolic link in `mods-enabled`:
```bash
sudo ln -s /etc/freeradius/mods-available/ldap /etc/freeradius/mods-enabled/ldap
```
### 4.2 Configure the LDAP Connection 
Open the LDAP module configuration:
```bash
sudo nano /etc/freeradius/mods-available/ldap
```
Locate and update the LDAP server connection settings with the details of your LDAP server:
```
server = "ldap.example.org" 
port = 389
identity = "cn=admin,dc=example,dc=org" 
password = "your-bind-password" 
base_dn = "dc=example,dc=org"
```
Replace the example values with the LDAP information provided for your group.
For example:
```
server = "ldap://idp-08.ubuntunet.org" # replace with the FQDN of your LDAP server 
port = 389
identity = "cn=idpuser,ou=system,dc=zimren,dc=ac,dc=zw" 
password = "password used for idpuser" 
base_dn = "ou=people,dc=zimren,dc=ac,dc=zw"
```
### 4.3 Configure the LDAP User Filter 
In the `user` section of the LDAP configuration, configure FreeRADIUS to search for users using either the `uid` or `cn` attribute:
```
user { base_dn = "${..base_dn}" 
filter = "(|(uid=%{%{Stripped-User-Name}:-%{User-Name}})(cn=%{%{Stripped-User-Name}:-%{User-Name}}))" }
```
This allows FreeRADIUS to locate an LDAP user whose username is stored in either the `uid` or `cn` attribute.

For example, a username of bnamuli results in a search equivalent to:
```
(|(uid=bnamuli)(cn=bnamuli))
```

### 4.4 Test the LDAP Connection 
Before testing authentication through FreeRADIUS, verify that the LDAP server can be queried using the configured bind account:
```bash
ldapsearch -x \
  -H ldap://idp-[x].ubuntunet.org \
  -D "cn=idpuser,ou=system,dc=example,dc=org" \
  -W \
  -b "ou=people,dc=example,dc=org" \
  '(|(uid=user1)(cn=user1))'
```
If this command fails to work install the ldap-utils then rerun the command 
```bash
apt install ldap-utils
```

Replace the LDAP server, bind DN, Base DN, and username with the values provided for your group.

A successful search should return the LDAP entry for the user.

### 4.5 Verify the FreeRADIUS Configuration
Check the FreeRADIUS configuration for errors:
```bash
sudo freeradius -XC
```
If the configuration check is successful, start FreeRADIUS in debug mode:
```bash
sudo freeradius -X
```
Check the debug output and confirm that the LDAP module initializes and successfully connects to the LDAP server.

**Checkpoint**: Do not proceed to EAP authentication testing until FreeRADIUS can successfully connect to LDAP and locate your test user.

## Part 5 Test EAP Authentication
In this part, you will test EAP authentication against your FreeRADIUS server using `eapol_test`. The test verifies that the EAP-TTLS tunnel can be established and that the user's credentials can be authenticated against the configured LDAP identity store.

### 5.1 Install eapol_test
Install the `eapoltest` package
```bash
sudo add-apt-repository universe 
sudo apt update 
sudo apt install -y eapoltest
```
Verify that `eapol_test` is installed:
```bash
which eapol_test
```

### 5.2 Download the EAP Test Configuration Files 
Create a directory for the EAP test configuration files:
```bash
mkdir -p ~/radius-debug 
cd ~/radius-debug
```

Download the test configurations:
```bash
wget https://networkradius.com/assets/scripts/eapol_test/peap-mschapv2.conf 
wget https://networkradius.com/assets/scripts/eapol_test/ttls-pap.conf 
wget https://networkradius.com/assets/scripts/eapol_test/ttls-chap.conf 
wget https://networkradius.com/assets/scripts/eapol_test/ttls-mschapv2.conf
```
For this lab, we will use the ttls-pap.conf configuration to test EAP-TTLS with PAP.

### 5.3 Configure the EAP Test
Open the TTLS-PAP test configuration:
```bash
nano ~/radius-debug/ttls-pap.conf
```
Update the `ssid`, `identity`, and `password` values.

For example:
```
network={ 
    ssid="eduroam" 
    key_mgmt=WPA-EAP 
    eap=TTLS identity="bnamuli" 
    anonymous_identity="anonymous" 
    password="your-user-password" 
    phase2="auth=PAP" 
}
```
Replace:
- `identity` with the username of a valid user in your identity store.
- `password` with the user's password.
- `ssid` with eduroam.

The `anonymous_identity` can remain as:
```bash
anonymous_identity="anonymous"
```
Save the changes and exit the file.

### 5.4 Start FreeRADIUS in Debug Mode
Start FreeRADIUS in debug mode:
```bash
sudo freeradius -X
```
Leave this terminal running so that you can observe the authentication process.

Open a second terminal to run the EAP test.

### 5.5 Run the EAP-TTLS Authentication Test
Navigate to the directory containing the test configuration:
```bash
cd ~/radius-debug
```
Confirm that the `ttls-pap.conf` file contains the correct `identity`, `password`, and `ssid` values:
```bash
cat ttls-pap.conf
```

Run the EAP-TTLS/PAP authentication test:
```bash
eapol_test -c ttls-pap.conf -a 127.0.0.1 -p 1812 -s testing123
```
Where:

- `-c` specifies the EAP test configuration file.
- `-a` specifies the FreeRADIUS server address.
- `-p` specifies the RADIUS authentication port.
- `-s` specifies the shared secret configured for the RADIUS client.

### 5.6 Verify Successful Authentication
Observe both the `eapol_test` output and the FreeRADIUS debug output.

A successful test should complete with:
```
SUCCESS
```
The FreeRADIUS debug output should show the EAP-TTLS tunnel being established, the inner authentication request being processed, and an Access-Accept returned to the client.

**Checkpoint**: Ensure that EAP-TTLS authentication is successful before proceeding to federation testing.

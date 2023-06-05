# getcert_vmware
Automate FreeIPA certificate deployments and renewals for vCenter and anything else that uses the vSphere REST API calls for certificate management.

See also: TBC

Adapted from https://github.com/dmgeurts/getcert_paloalto


# FreeIPA Certificates for vCenter

While free LetsEncrypt certificates are great, when you have an internal CA then this opens the door for using your own certificates. vCenter should never be publily exposed, so using an internal CA stands to reason.

Not using LetsEncrypt avoids problems with permitting ingress HTTP-01 traffic or setting up DNS-01 verification, both have their uses and while I use both, FreeIPA PKI is a useful local solution for a local problem. It also avoids exposing internal management dns records externally, all Let's Encrypt issued certificate domains are publicly listed.

The following will detail how to automate the ipa-getcert certificate process for VMware devices that support the vSphere REST API.


# Prerequisites

Image TBD

### Before getting started, the following are required:
+ A FreeIPA server with Dogtag CA.

+ A linux-based container, VM, or host with reachability to vCenter and the following packages installed:
  + ipa-getcert - part of ipa-client-install on a FreeIPA enrolled host.

+ A Service Principal for the web interface that needs a certificate:
  + Manually create the host with the FQDN of vCenter.
    + Edit the host to add the enrolled host under 'managed by'.
  + Create a Service Principal for vCenter: `HTTP/fqdn@DOMAIN.COM`
    + Edit it Service Principal to add the enrolled host under 'managed by'.

+ vCenter credentials:
  + With the following vCenter privileges:
    + Certificate Managent - Manage (below Admins priv)
    + Certificate Management - Administer (Admins priv)
  + If using FreeIPA as the LDAP server for vCenter, it's possible to use Keycloak as a SSO provider to vCenter. Keycloak supports SPNEGO, I'm curious if that can be used from a script to eradicate a user/pass combo stored in a file somewhere.
  + When using centralised authentication, consider whether the API user should be a user or dedicated API account.
    + Using a user account will mean the API calls stop working when the user is removed!


# The Scripts
There are two scripts:
+ "vmw_getcert" --> https://github.com/dmgeurts/getcert_vmware/edit/master/vmw_getcert
  + Obtains a certificate from FreeIPA and sets the post-save to use vmw_instcert for automatic installation of a renewed certificate.
+ "vmw_instcert" --> https://github.com/dmgeurts/getcert_vmware/edit/master/vmw_instcert
  + Installs the certificate on a vSphere appliance.

# How It Works
### vmw_getcert performs the following actions:
1. Uses the privileges set in FreeIPA (managed by) to call ipa-getcert and request a certificate from FreeIPA.
2. ipa-getcert will automatically renew a certificate when it's due, as long as the FQDN DNS record resolves, and the host and Service Principal still exist in FreeIPA.
3. Sets the post-save command to vmw_instcert with the same parameters as issued to vmw_getcert, for automated installation of renewed certificates.
    - Post-save will run on the first certificate save, using vmw_instcert for certificate installation.

### vmw_instcert performs the following actions:
1. Uploads the management certificate in json format using the REST API.
2. Logs all output to `/var/log/vmw_instcert.log`.

### Script options
Run vmw_getcert without sudo, doing so may block access to your keytab and then ipa-getcert can't authenticate to IPA. It will test for and run ipa-getcert with sudo privileges.

vmw_getcert uses the following options:
```
Usage: vmw_getcert [-hv] -c CERT_CN [-a CRED_FILE] [OPTIONS] [FQDN]
This script requests a certificate from FreeIPA using ipa-getcert and calls a
partner script to deploy the certificate to a VMware appliance via REST API.

    FQDN              Fully qualified name of the VMware appliance web interface.
                      Must be reachable from this host on port TCP/443. Defaults
                      to CERT_CN if omitted.
    -c CERT_CN        REQUIRED. Common Name (Subject) of the certificate (must be a
                      FQDN). Will also present in the certificate as a SAN.
    -a CRED_FILE      File with VMware credentials in plain text or base64 format.
                      Defaults to /etc/ipa/.vmwrc
OPTIONS:
    -i                Resolve CERT_CN to an IP address, for inclusion in the SAN.
    -I IP4            IPv4 address of the vCenter server, for inclusion in the SAN.
    -S SERVICE        Service of the Service Principal. Default: HTTP.
    -T CERT_PROFILE   Ask IPA to process the request using the named profile or
                      template.
    -G TYPE           Type of key to be generated if one is not already in place.
                      IPA CA uses RSA by default. (RSA, DSA, EC or ECDSA)
    -b BITS           In case a new key pair needs to be generated, this option
                      specifies the size of the key. Default: 2048 (RSA/DSA).
                      EC: 256, 384 or 512. See certmonger.conf for the default.

    -h                Display this help and exit.
    -v                Verbose mode.
```

At a minimum vmw_getcert can be run as: `sudo vmw_getcert -c vcenter.domain.com` This will look for API credentials in `/etc/api/.vmwrc` and will assume that vcenter can be reached via the common name given for the certificate. If you want to include the IPv4 address in the SAN add the `-i` option to have the command name resolved to the IPv4 address.

**Note:** FreeIPA only supports RSA keys. Hence the -G option is in preparation of future support of other keys. [More info](https://www.reddit.com/r/FreeIPA/comments/134puyw/freeipa_ca_pki_ecdsa_support/).

Run vmw_instcert with sudo, it will test if it's run with uid 0 (root). Ensure this command is added to sudo rules if it must be used manually.

vmw_instcert uses the same options, minus the FreeIPA specifics:
```
Usage: vmw_instcert [-hv] -c CERT_CN -a CRED_FILE FQDN
This script uploads a certificate issued by ipa-getcert to a VMware appliance
via REST API.

    FQDN          Fully qualified name of the VMware appliance web interface.
                  Must be reachable from this host on port TCP/443.
    -c CERT_CN    REQUIRED. Common Name (Subject) of the certificate, to find
                  the certificate and key files.
    -a CRED_FILE  File with credentials in plain text or base64 format.
                  Defaults to /etc/ipa/.vmwrc

    -h            Display this help and exit.
    -v            Verbose mode.
```

# Automated Renewal and Installation
ipa-getcert requests the certificate with the post-save option, thus no cronjob is needed to install renewed certificates and vmw_instcert does not need to be manually called.

The post-save process will upload the renewed certificates, and take the same actions against SSL/TLS Service Profiles as when the certificate was initially requested and deployed using vmw_getcert.

Check the configured post-save command with:
```
sudo ipa-getcert list [-i Request_ID | -f Path_to_cert_file]
```

# Prepare the Script for Execution
+ In an environment with centralised credentials it's best to not run code under a user that may be removed. Install the code to a common location: `/usr/local/bin` is assumed. When using a different location, change variable INST_CERT in vmw_getcert accordingly.
  + Create a new role with the following privileges in vCenter for the API user:
    + Certificate Managent - Manage (below Admins priv)
    + Certificate Management - Administer (Admins priv)
  + Assign the API user to the role
+ Install and ensure vmw_getcert and vmw_instcert are executable:
  ```
  sudo cp getcert_vmware/vmw_{get,inst}cert /usr/local/bin/
  sudo chmod +x /usr/local/bin/vmw_{get,inst}cert
  ```

# Expected CLI output
When requesting and deploying a certificate:
```
user@host:~$ sudo vmw_getcert -v -c vcenter.domain.local -i
Certificate Common Name: vcenter.domain.local
Credentials found in: /etc/ipa/.vmwrc
IPv4 will be included in the SAN.
New signing request "20230605141554" added.
Certificate requested for: vcenter.domain.local
  Certificate install and commit by the post-save process on: vcenter.domain.local took 1 seconds.
FINISHED: Check the VMware appliance to see if the certificate is in use.
```

# Crontab
Not required; ipa-getcert certificate monitoring takes care of running post-save after renewing certificates.


# Validate Automated Renewals
vmw_instcert will log to `/var/log/vmw_instcert.log`.

vmw_getcert doesn't log as it's expected to be run once (manually).
```
user@host:~$ sudo vmw_instcert -v -c vcenter.domain.local vcenter.domain.local
START of vmw_instcert.
Certificate Common Name: vcenter.domain.local
Credentials found in: /etc/ipa/.vmwrc => vmware-api-user-ssl
VMW_MGMT: vcenter.domain.local
curl -k -X POST https://vcenter.domain.local/api/session -H "Authorization: Basic **********"
Session-key: 2f6203f67dc59a7bc23771a5d986b746
Certmonger certificate state MONITORING, stuck: no.

XML API output for root certificate: "64049D739AA7928FAD89D786DCB079E177E965E4"
XML API output for certificate: 
Finished uploading certificate for: vcenter.domain.local
END - Finished certificate installation to: vcenter.domain.local
```

If successful, you should see something similar to the following in the log file:
```
[2023-06-05 23:31:05+02:00]: START of vmw_instcert.
[2023-06-05 23:31:05+02:00]: Certificate Common Name: vcenter.domain.local
[2023-06-05 23:31:05+02:00]: Credentials found in: /etc/ipa/.vmwrc => vmware-api-user-ssl
[2023-06-05 23:31:05+02:00]: VMW_MGMT: vcenter.domain.local
[2023-06-05 23:31:05+02:00]: Certmonger certificate state MONITORING, stuck: no.
[2023-06-06 00:05:11+02:00]: XML API output for root certificate: "64049D739AA7928FAD89D786DCB079E177E965E4"
[2023-06-05 23:31:06+02:00]: XML API output for certificate:
[2023-06-05 23:31:06+02:00]: Finished uploading certificate for: vcenter.domain.local
[2023-06-05 23:31:06+02:00]: END - Finished certificate installation to: vcenter.domain.local
```

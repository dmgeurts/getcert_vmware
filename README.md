# getcert_vmware
Automate FreeIPA certificate deployments and renewals for vCenter and anything else that uses the vSphere REST API calls for certificate management.

See also: TBC

Adapted from https://github.com/dmgeurts/getcert_paloalto


# FreeIPA Certificates for vCenter

While free LetsEncrypt certificates are great, when you have an internal CA then this opens the door for using your own certificates. vCenter should never be publily exposed, so using an internal CA stands to reason.

Not using LetsEncrypt avoids problems with permitting ingress HTTP-01 traffic or setting up DNS-01 verification, both have their uses and while I use both, FreeIPA PKI is a useful local solution for a local problem.

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
    + TBC
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
Usage: vmw_getcert [-hv] -c CERT_CN -a CRED_FILE [OPTIONS] FQDN
This script requests a certificate from FreeIPA using ipa-getcert and calls a
partner script to deploy the certificate to a VMware appliance via REST API.

    FQDN          Fully qualified name of the VMware appliance web interface.
                  Must be reachable from this host on port TCP/443.
    -c CERT_CN    REQUIRED. Common Name (Subject) of the certificate (must be a
                  FQDN). Will also present in the certificate as a SAN.
    -a CRED_FILE  File with VMware credentials in plain text or base64 format.
                  Defaults to /etc/ipa/.vmwrc
OPTIONS:
    -G TYPE       Type of key to be generated if one is not already in place.
                  IPA CA uses RSA by default. (RSA, DSA, EC or ECDSA)
    -b BITS       In case a new key pair needs to be generated, this option
                  specifies the size of the key. Default: 2048 (RSA/DSA).
                  EC: 256, 384 or 512. See certmonger.conf for the default.

    -h            Display this help and exit.
    -v            Verbose mode.
```

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
  + Create the privileges in vCenter for the API user:
    + TBC
      + ~Authentication Profile: None~
      + ~Set a strong password (only used to obtain the API key)~
      + ~Administrator Type: Role Based~
      + ~Profile: Create a new profile with the folowing rights (Device >> Admin Roles):~
        + ~Web UI - Disable everything~
        + ~XML API - Enable only:~
          + ~Configuration~
          + ~Operational Requests~
          + ~Commit~
          + ~Import~
        + ~Command Line - None~
        + ~REST API - Disable everything~
+ Install and ensure vmw_getcert and vmw_instcert are executable:
  ```
  sudo cp getcert_vmware/vmw_{get,inst}cert /usr/local/bin/
  sudo chmod +x /usr/local/bin/vmw_{get,inst}cert
  ```

# Expected CLI output
When requesting and deploying a certificate:
```
TBC
```

# Crontab
Not required; ipa-getcert certificate monitoring takes care of running post-save after renewing certificates.


# Validate Automated Renewals
vmw_instcert will log to `/var/log/vmw_instcert.log`.

vmw_getcert doesn't log as it's expected to be run once (manually).
```
cat /var/log/vmw_instcert.log
```

If successful, you should see something similar to the following in the log file:
```
TBC
```

# custodia.ipa — IPA vault plugin for Custodia

**WARNING** *custodia.ipa is a tech preview with a provisional API.*

custodia.ipa is a collection of plugins for
[Custodia](https://custodia.readthedocs.io/). It provides integration with
[FreeIPA](http://www.freeipa.org). The *IPAVault* plugin is an interface to
[FreeIPA vault](https://www.freeipa.org/page/V4/Password_Vault). Secrets are
encrypted and stored in [Dogtag](http://www.dogtagpki.org)'s Key Recovery
Agent. The *IPACertRequest* plugin creates private key and signed certificates
on-demand. Finally the *IPAInterface* plugin is a helper plugin that wraps
ipalib and GSSAPI authentication.
 

## Requirements

### Installation

* pip
* setuptools >= 18.0

### Runtime

* custodia >= 0.3.1
* ipalib >= 4.5.0
* ipaclient >= 4.5.0
* Python 2.7 (Python 3 support in IPA vault is unstable.)

custodia.ipa requires an IPA-enrolled host and a Kerberos TGT for
authentication. It is recommended to provide credentials with a keytab file or
GSS-Proxy. Furthermore *IPAVault* depends on Key Recovery Agent service
(``ipa-kra-install``).

### Testing and development

* wheel
* tox

### virtualenv requirements

custodia.ipa depends on several binary extensions and shared libraries for
e.g. python-cryptography, python-gssapi, python-ldap, and python-nss. For
installation in a virtual environment, a C compiler and several development
packages are required.

```
$ virtualenv venv
$ venv/bin/pip install --upgrade custodia.ipa
```

#### Fedora

```
$ sudo dnf install python2 python-pip python-virtualenv python-devel \
    gcc redhat-rpm-config krb5-workstation krb5-devel libffi-devel \
    nss-devel openldap-devel cyrus-sasl-devel openssl-devel
```

#### Debian / Ubuntu

```
$ sudo apt-get update
$ sudo apt-get install -y python2.7 python-pip python-virtualenv python-dev \
    gcc krb5-user libkrb5-dev libffi-dev libnss3-dev libldap2-dev \
    libsasl2-dev libssl-dev
```

---

## Example configuration

Create directories

```
$ sudo mkdir /etc/custodia /var/lib/custodia /var/log/custodia /var/run/custodia
$ sudo chown USER:GROUP /var/lib/custodia /var/log/custodia /var/run/custodia
$ sudo chmod 750 /var/lib/custodia /var/log/custodia
```

Create service account and keytab

```
$ kinit admin
$ ipa service-add custodia/client1.ipa.example
$ ipa service-allow-create-keytab custodia/client1.ipa.example --users=admin
$ mkdir -p /etc/custodia
$ ipa-getkeytab -p custodia/client1.ipa.example -k /etc/custodia/custodia.keytab
```

The IPA cert request plugin needs additional permissions

```
$ ipa privilege-add \
    --desc="Create and request service certs with Custodia" \
    "Custodia Service Certs"
$ ipa privilege-add-permission \
    --permissions="Retrieve Certificates from the CA" \
    --permissions="Request Certificate" \
    --permissions="Revoke Certificate" \
    --permissions="System: Modify Services" \
    "Custodia Service Certs"
# for add_principal=True
$ ipa privilege-add-permission \
    --permissions="System: Add Services" \
    "Custodia Service Certs"
$ ipa role-add \
    --desc="Create and request service certs with Custodia" \
    "Custodia Service Cert Adminstrator"
$ ipa role-add-privilege \
    --privileges="Custodia Service Certs" \
    "Custodia Service Cert Adminstrator"
$ ipa role-add-member \
    --services="custodia/client1.ipa.example" \
    "Custodia Service Cert Adminstrator"
```

Create ```/etc/custodia/custodia.conf```

```
[DEFAULT]
confdir = /etc/custodia
libdir = /var/lib/custodia
logdir = /var/log/custodia
rundir = /var/run/custodia

[global]
debug = true
server_socket = ${rundir}/custodia.sock
auditlog = ${logdir}/audit.log

[auth:ipa]
handler = IPAInterface
keytab = ${confdir}/custodia.keytab
ccache = FILE:${rundir}/ccache

[auth:creds]
handler = SimpleCredsAuth
uid = root
gid = root

[authz:paths]
handler = SimplePathAuthz
paths = /. /secrets

[store:vault]
handler = IPAVault

[store:cert]
handler = IPACertRequest
backing_store = vault

[/]
handler = Root

[/secrets]
handler = Secrets
store = vault

[/secrets/certs]
handler = Secrets
store = cert
```

Run Custodia server

```
$ custodia /etc/custodia/custodia.conf
```


## IPA cert request

The IPACertRequest store plugin generates or revokes certificates on the
fly. It uses a backing store to cache certs and private keys. The plugin can
create service principal automatically. However the host must already exist.

The request ```GET /secrets/certs/HTTP/client1.ipa.example``` generates a
private key and CSR for the service ```HTTP/client1.ipa.example``` with
DNS subject alternative name ```client1.ipa.example```. A DELETE request
removes the cert/key pair from the backing store and revokes the cert at
the same time.

Automatical renewal of revoked or expired certificates is not implemented yet.


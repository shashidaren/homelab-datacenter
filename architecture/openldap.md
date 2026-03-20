# 🔐 OpenLDAP Lab Guide

<p align="center">
  <b>Rocky Linux 9 • LDAPS • Identity Management Lab</b><br>
  Production-style LDAP deployment with full configuration and security hardening
</p>

<p align="center">
  <img src="https://img.shields.io/badge/OS-Rocky%20Linux%209-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/LDAP-OpenLDAP-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/Security-LDAPS%20Enabled-success?style=for-the-badge">
  <img src="https://img.shields.io/badge/Status-Lab%20Project-orange?style=for-the-badge">
</p>

---

## 🧠 Overview

This project documents a **complete OpenLDAP deployment** based on a real lab build.

---

## 🧱 Phase 1 — Installation

```bash
sudo dnf config-manager --set-enabled plus
sudo dnf install -y openldap-servers openldap-clients
sudo systemctl enable --now slapd
```

### Set Config Admin Password

```bash
slappasswd -s "Telco666"
```

```bash
cat <<EOF > chrootpw.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}PASTE_YOUR_CONFIG_HASH_HERE
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif
```

---

## 🌐 Phase 2 — Domain & Schema

```bash
cat <<EOF > rootdb.ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=lab,dc=local
-
replace: olcRootDN
olcRootDN: cn=admin,dc=lab,dc=local
-
add: olcRootPW
olcRootPW: {SSHA}PASTE_YOUR_DOMAIN_HASH_HERE
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f rootdb.ldif
```

### Load Schemas

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

---

## 🔐 Phase 3 — Security Hardening

### Firewall

```bash
firewall-cmd --add-service={ldap,ldaps} --permanent
firewall-cmd --reload
```

### Disable Anonymous + ACL

```bash
cat <<EOF > disable_anon.ldif
dn: cn=config
changetype: modify
add: olcDisallows
olcDisallows: bind_anon
-
add: olcRequires
olcRequires: authc

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage by * break
olcAccess: {1}to * by self read by dn.base="cn=admin,dc=lab,dc=local" write by * none
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f disable_anon.ldif
```

---

## 🔒 Phase 4 — TLS / LDAPS

```bash
mkdir -p /etc/openldap/certs

openssl req -new -x509 -nodes \
-out /etc/openldap/certs/ldapcert.pem \
-keyout /etc/openldap/certs/ldapkey.pem \
-days 3650

chown ldap:ldap /etc/openldap/certs/*.pem
```

### Apply TLS

```bash
cat <<EOF > certs.ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldapcert.pem
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldapkey.pem
-
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ldapcert.pem
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f certs.ldif
```

---

## 👥 Phase 5 — Directory Structure

```bash
cat <<EOF > base_structure.ldif
dn: dc=lab,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: lab
dc: lab

dn: ou=People,dc=lab,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=lab,dc=local
objectClass: organizationalUnit
ou: Groups
EOF

ldapadd -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -f base_structure.ldif
```

---

### Users & Groups (FULL)

```bash
cat <<EOF > expansion.ldif

dn: uid=shashi,ou=People,dc=lab,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Shashi
sn: Shashi
uid: shashi
uidNumber: 2000
gidNumber: 2000
homeDirectory: /home/shashi
loginShell: /bin/bash
userPassword: YourSecretPassword123
mail: shashi@lab.local

dn: uid=aiman,ou=People,dc=lab,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Aiman
sn: Admin
uid: aiman
uidNumber: 2001
gidNumber: 2000
homeDirectory: /home/aiman
loginShell: /bin/bash
userPassword: TempPassword123
mail: aiman@lab.local

dn: uid=siva,ou=People,dc=lab,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Siva
sn: Monitor
uid: siva
uidNumber: 2002
gidNumber: 2001
homeDirectory: /home/siva
loginShell: /bin/bash
userPassword: TempPassword123
mail: siva@lab.local

dn: cn=admins,ou=Groups,dc=lab,dc=local
objectClass: posixGroup
cn: admins
gidNumber: 2000
memberUid: shashi
memberUid: aiman

dn: cn=viewers,ou=Groups,dc=lab,dc=local
objectClass: posixGroup
cn: viewers
gidNumber: 2001
memberUid: siva

EOF

ldapmodify -x -D "cn=admin,dc=lab,dc=local" -w "" -a -f expansion.ldif
```

---

## ⚡ Phase 6 — Indexing

```bash
cat <<EOF > indexing.ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcDbIndex
olcDbIndex: objectClass eq
olcDbIndex: uid pres,eq
olcDbIndex: mail pres,eq
olcDbIndex: cn pres,eq,sub
olcDbIndex: memberUid eq
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f indexing.ldif
```

---

Configure Linux clients to authenticate against OpenLDAP using SSSD.

📦 Install Required Packages
sudo dnf install -y sssd sssd-ldap oddjob oddjob-mkhomedir authselect-compat
⚙️ Configure SSSD

Edit:

vim /etc/sssd/sssd.conf
```bash
[sssd]
services = nss, pam
config_file_version = 2
domains = default

[domain/default]
id_provider = ldap
auth_provider = ldap

ldap_uri = ldap://192.168.0.112
ldap_search_base = dc=lab,dc=local

ldap_tls_reqcert = never

cache_credentials = true
enumerate = true

i🔒 Set Permissions
chmod 600 /etc/sssd/sssd.conf
🏠 Enable Auto Home Directory Creation
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
▶️ Start SSSD
systemctl enable --now sssd
🔍 Test Authentication
id shashi
getent passwd shashi

## 💾 Maintenance & Backup

```bash
slapcat -b "dc=lab,dc=local" -l /var/backups/ldap.ldif

# 1. Backup Script Location
# /usr/local/bin/ldap_backup.sh
# 2. Crontab (Daily at 02:00)
00 02 * * * /usr/local/bin/ldap_backup.sh > /dev/null 2>&1

```

---

## 🔍 Verification

```bash
openssl x509 -in /etc/openldap/certs/ldapcert.pem -text -noout | grep -E "Subject:|Not After"
```

```bash
openssl x509 -noout -modulus -in /etc/openldap/certs/ldapcert.pem | openssl md5
openssl rsa -noout -modulus -in /etc/openldap/certs/ldapkey.pem | openssl md5
```

```bash
ldapsearch -x -D "cn=admin,dc=lab,dc=local" -w -b "dc=lab,dc=local" dn
```

```bash
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://localhost
```

```bash
journalctl -u slapd -f
```

---

## 🚀 Summary

* Full OpenLDAP deployment
* Secure LDAPS configuration
* Structured directory with users & groups
* Performance indexing enabled
* Backup and verification included

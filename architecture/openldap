# 🔐 OpenLDAP Lab Guide

<p align="center">
  <b>Rocky Linux 9 • LDAPS • Identity Management Lab</b><br>
  Production-style LDAP deployment with security hardening and structured directory design
</p>

<p align="center">
  <img src="https://img.shields.io/badge/OS-Rocky%20Linux%209-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/LDAP-OpenLDAP-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/Security-LDAPS%20Enabled-success?style=for-the-badge">
  <img src="https://img.shields.io/badge/Status-Lab%20Project-orange?style=for-the-badge">
</p>

---

## 🧠 Overview

This project documents a **secure OpenLDAP deployment** built in a home lab environment.

It demonstrates:

* 🔐 LDAP over TLS (LDAPS)
* 🚫 Disabled anonymous authentication
* 🧱 Structured identity tree (Users & Groups)
* ⚡ Performance optimization (indexing)
* 💾 Backup strategy with `slapcat`

---

## 🏗 Architecture

```text
                +----------------------+
                |     LDAP Client      |
                |  (Apps / Servers)    |
                +----------+-----------+
                           |
                           |  LDAPS (636)
                           |
                 +---------v----------+
                 |     ldap01         |
                 |  OpenLDAP Server   |
                 |  Rocky Linux 9     |
                 +--------------------+
```

---

## 📌 Environment

| Component  | Value           |
| ---------- | --------------- |
| OS         | Rocky Linux 9   |
| Hostname   | ldap01          |
| IP Address | 192.168.0.112   |
| Domain     | dc=lab,dc=local |
| Protocol   | LDAPS (636)     |

---

## 🧱 Phase 1 — Installation

```bash
sudo dnf config-manager --set-enabled plus
sudo dnf install -y openldap-servers openldap-clients
sudo systemctl enable --now slapd
```

### 🔑 Set Admin Password

```bash
slappasswd -s "Telco666"
```

```bash
cat <<EOF > chrootpw.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}PASTE_HASH
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif
```

---

## 🌐 Phase 2 — Domain Setup

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
olcRootPW: {SSHA}PASTE_HASH
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f rootdb.ldif
```

### 📦 Load Schemas

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

---

## 🔐 Phase 3 — Security Hardening

### 🔥 Firewall

```bash
firewall-cmd --add-service={ldap,ldaps} --permanent
firewall-cmd --reload
```

### 🚫 Disable Anonymous Bind

```bash
cat <<EOF > disable_anon.ldif
dn: cn=config
changetype: modify
add: olcDisallows
olcDisallows: bind_anon
-
add: olcRequires
olcRequires: authc
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

```bash
cat <<EOF > certs.ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldapcert.pem
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldapkey.pem
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f certs.ldif
```

---

## 👥 Phase 5 — Directory Structure

```bash
cat <<EOF > base.ldif
dn: dc=lab,dc=local
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

ldapadd -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -f base.ldif
```

---

## ⚡ Performance Optimization

```bash
cat <<EOF > indexing.ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcDbIndex
olcDbIndex: uid eq
olcDbIndex: cn eq
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f indexing.ldif
```

---

## 💾 Backup Strategy

```bash
slapcat -b "dc=lab,dc=local" -l /var/backups/ldap.ldif
```

---

## 🔍 Verification

```bash
ldapsearch -x -b "dc=lab,dc=local"
```

```bash
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://localhost
```

```bash
journalctl -u slapd -f
```

---

## 🧠 Key Takeaways

* Security first: disable anonymous bind
* Always use TLS (LDAPS)
* LDAP is strict — syntax matters
* Indexing improves performance
* Logs are your best debugging tool

---

## 🚀 Final Thoughts

This lab demonstrates a **real-world approach to identity management** using OpenLDAP.

It forms a strong foundation for:

* Centralized authentication
* Infrastructure integration
* Enterprise directory services

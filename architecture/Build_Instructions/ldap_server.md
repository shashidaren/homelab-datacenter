OpenLDAP Lifecycle Management: Rocky Linux 9
Server IP: 192.168.0.112 | Domain: dc=lab,dc=local

Phase 1: Installation & Initial Config
Enable the Plus repository to access the standard OpenLDAP server packages.

Bash
# 1. Enable Repo & Install
sudo dnf config-manager --set-enabled plus
sudo dnf install -y openldap-servers openldap-clients
sudo systemctl enable --now slapd

# 2. Set Admin Password for cn=config
# Generate hash first: slappasswd -s "Telco666"
cat <<EOF > chrootpw.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}PASTE_YOUR_CONFIG_HASH_HERE
EOF

sudo ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif
Phase 2: Domain & Schema Setup
Define the base DN and load core schemas required for users and groups.

Bash
# 1. Define dc=lab,dc=local
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

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f rootdb.ldif

# 2. Load Core Schemas
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
Phase 3: Networking & Security Hardening
Enable LDAPS (636), open firewalls, and disable anonymous binds.

Bash
# 1. Firewall Open
sudo firewall-cmd --add-service={ldap,ldaps} --permanent
sudo firewall-cmd --reload

# 2. Enable LDAPS Listener
# Edit /etc/sysconfig/slapd and set: SLAPD_URLS="ldapi:/// ldap:/// ldaps:///"
sudo systemctl restart slapd

# 3. Disable Anonymous Binds
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

sudo ldapadd -Y EXTERNAL -H ldapi:/// -f disable_anon.ldif
Phase 4: TLS Configuration
Secure communication using self-signed certificates.

Bash
# 1. Generate Certs
sudo mkdir -p /etc/openldap/certs
sudo openssl req -new -x509 -nodes -out /etc/openldap/certs/ldapcert.pem \
-keyout /etc/openldap/certs/ldapkey.pem -days 3650
sudo chown ldap:ldap /etc/openldap/certs/*.pem

# 2. Apply to LDAP
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

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f certs.ldif
Phase 5: Identity & Group Population
Build the tree structure and add users/groups.

Bash
# 1. Build OUs
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

# 2. Bulk User/Group Expansion (Add Shashi, Aiman, Siva)
# [Insert your Expansion LDIF content here]
ldapmodify -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -a -f expansion.ldif
Phase 6: Maintenance & Backups
Automated hot-backups via slapcat.

Bash
# 1. Backup Script Location: /usr/local/bin/ldap_backup.sh
# 2. Crontab (Run daily at 02:00)
00 02 * * * /usr/local/bin/ldap_backup.sh > /dev/null 2>&1
Verification Commands
Local Search: ldapsearch -x -b "dc=lab,dc=local" "(objectClass=*)" dn

Secure Search: LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://localhost -D "cn=admin,dc=lab,dc=local" -w "Telco666" -b "dc=lab,dc=local"

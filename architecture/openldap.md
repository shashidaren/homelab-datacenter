🔐 OpenLDAP Lab Guide (Verified Build)<p align="center"><b>Rocky Linux 9 • LDAPS • Identity Management Lab</b>Production-style LDAP deployment with full configuration and security hardening</p>🧱 Phase 1 — InstallationBashsudo dnf config-manager --set-enabled plus
sudo dnf install -y openldap-servers openldap-clients
sudo systemctl enable --now slapd
Set Config Admin PasswordGenerate hash: slappasswd -s "Telco666"Apply the hash below:Bashcat <<EOF > chrootpw.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}PASTE_YOUR_CONFIG_HASH_HERE
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif
🌐 Phase 2 — Domain & SchemaBashcat <<EOF > rootdb.ldif
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
Load SchemasBashldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
🔐 Phase 3 — Security Hardening (Working ACLs)FirewallBashfirewall-cmd --add-service={ldap,ldaps} --permanent
firewall-cmd --reload
Disable Anonymous & Apply Fixed ACLsNote: This ACL allows "anonymous auth" which is required for users to login.Bashcat <<EOF > disable_anon.ldif
dn: cn=config
changetype: modify
add: olcDisallows
olcDisallows: bind_anon
-
add: olcRequires
olcRequires: authc

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage by * break
olcAccess: {1}to attrs=userPassword by self write by dn.base="cn=admin,dc=lab,dc=local" write by anonymous auth by * none
olcAccess: {2}to * by dn.base="cn=admin,dc=lab,dc=local" write by self read by * read
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f disable_anon.ldif
🔒 Phase 4 — TLS / LDAPSBashmkdir -p /etc/openldap/certs
openssl req -new -x509 -nodes -out /etc/openldap/certs/ldapcert.pem -keyout /etc/openldap/certs/ldapkey.pem -days 3650
chown ldap:ldap /etc/openldap/certs/*.pem
Apply TLSBashcat <<EOF > certs.ldif
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
👥 Phase 5 — Directory Structure & UsersBashcat <<EOF > base_structure.ldif
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
Add SSHA Hashed UsersGenerate hash: slappasswd -s "Telco777"Bashcat <<EOF > expansion.ldif
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
userPassword: {SSHA}PASTE_HASH_HERE
mail: shashi@lab.local

dn: cn=admins,ou=Groups,dc=lab,dc=local
objectClass: posixGroup
cn: admins
gidNumber: 2000
memberUid: shashi
EOF

ldapadd -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -f expansion.ldif
⚙️ Phase 6 — Client Configuration (SSSD)Bashsudo dnf install -y sssd sssd-ldap oddjob oddjob-mkhomedir authselect-compat

cat <<EOF > /etc/sssd/sssd.conf
[sssd]
services = nss, pam
config_file_version = 2
domains = lab.local

[domain/lab.local]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldaps://192.168.0.112
ldap_search_base = dc=lab,dc=local
ldap_tls_reqcert = never
cache_credentials = true
enumerate = true
ldap_default_bind_dn = cn=admin,dc=lab,dc=local
ldap_default_authtok = Telco666
EOF

chmod 600 /etc/sssd/sssd.conf
authselect select sssd with-mkhomedir --force
systemctl enable --now sssd oddjobd
🔍 VerificationTaskCommandVerify Local Bindldapwhoami -x -D "uid=shashi,ou=People,dc=lab,dc=local" -WCheck SSSD Userid shashiVerify TLSLDAPTLS_REQCERT=never ldapsearch -x -H ldaps://localhost

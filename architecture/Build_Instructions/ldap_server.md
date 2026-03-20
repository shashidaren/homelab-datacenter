# 1. Enable the Plus repository
sudo dnf config-manager --set-enabled plus

# 2. Install the server and client tools
sudo dnf install -y openldap-servers openldap-clients

# 3. Start and enable the service
sudo systemctl enable --now slapd

# 4. generate a hashed password for the LDAP "root" administrator
slappasswd -s "Telco666"

# 5. Create a temporary file: chrootpw.ldif
# Replace THE_HASH_BELOW with the output from the previous command
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}your_generated_hash_here

# 3. Apply the password to the config database
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif

Define Domain & Manager
This sets up your dc=lab,dc=local base and the cn=admin manager account.

Create a file named rootdb.ldif:

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=lab,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=lab,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}your_generated_hash_here


Apply it:

Bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f rootdb.ldif

Gemini said
Since you are rebuilding on Rocky Linux 9, the instructions differ slightly from older versions because the package is now in the Plus repository and uses the OLC (Online Configuration) method.

Here are the precise, step-by-step standalone LDAP instructions for your documentation:

Phase 1: Repository & Installation
Rocky 9 moves openldap-servers to the "Plus" repo.

Bash
# 1. Enable the Plus repository
sudo dnf config-manager --set-enabled plus

# 2. Install the server and client tools
sudo dnf install -y openldap-servers openldap-clients

# 3. Start and enable the service
sudo systemctl enable --now slapd
Phase 2: Administrative Password Setup
You need to generate a hashed password for the LDAP "root" administrator.

Bash
# 1. Run this and COPY the output (looks like {SSHA}...)
slappasswd -s "ChangeMe123!"

# 2. Create a temporary file: chrootpw.ldif
# Replace THE_HASH_BELOW with the output from the previous command
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}your_generated_hash_here

# 3. Apply the password to the config database
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif
Phase 3: Define Domain & Manager
This sets up your dc=lab,dc=local base and the cn=admin manager account.

Create a file named rootdb.ldif:

LDIF
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=lab,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=lab,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}your_generated_hash_here
Apply it:

Bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f rootdb.ldif

Phase 4: Load Basic Schemas
Without these, your LDAP server won't understand common attributes like mail, uid, or homeDirectory.

sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif


Phase 5: Networking & Firewall

Ensure the server listens on all interfaces (port 389).

# 1. Open the Firewall
sudo firewall-cmd --add-service=ldap --permanent
sudo firewall-cmd --reload

# 2. Verify it is listening on 0.0.0.0:389
ss -tulnp | grep 389

Phase 6: Final Verification
ldapsearch -x -D "cn=admin,dc=lab,dc=local" -w "ChangeMe123!" -b "dc=lab,dc=local"


Create the base_structure.ldif file

cat <<EOF > base_structure.ldif
# Define the root of the domain
dn: dc=lab,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: lab
dc: lab

# Define the People container (where Shashi lives)
dn: ou=People,dc=lab,dc=local
objectClass: organizationalUnit
ou: People

# Define the Groups container (for permissions)
dn: ou=Groups,dc=lab,dc=local
objectClass: organizationalUnit
ou: Groups
EOF

Apply it using your Admin password
ldapadd -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -f base_structure.ldif

Adding the User & Admin Group
This step creates the shashi user and ensures they belong to the admins group. This is the "Key" that unlocks the door for your AI-Ops stack later.

Create users_groups.ldif

cat <<EOF > users_groups.ldif
# Create Shashi User
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

# Create Admins Group
dn: cn=admins,ou=Groups,dc=lab,dc=local
objectClass: posixGroup
cn: admins
gidNumber: 2000
memberUid: shashi
EOF


Apply the LDIF

ldapadd -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -f users_groups.ldif


Phase 9: The Final "Senior Admin" Audit
To confirm everything is perfect for your documentation, run this comprehensive search
ldapsearch -x -b "dc=lab,dc=local" "(objectClass=*)" dn

Phase 11: Disable Anonymous Binds (Security Hardening)
By default, anyone can run ldapsearch against your server without a password. Let's lock it down so only authenticated users (like your Grafana bind user) can see the directory.

Create disable_anon.ldif:

cat <<EOF > disable_anon.ldif
dn: cn=config
changetype: modify
add: olcDisallows
olcDisallows: bind_anon

dn: cn=config
changetype: modify
add: olcRequires
olcRequires: authc

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage by * break
olcAccess: {1}to * by self read by dn.base="cn=admin,dc=lab,dc=local" write by * none
EOF


Apply the security policy:

sudo ldapadd -Y EXTERNAL -H ldapi:/// -f disable_anon.ldif

Enable Database Indexing (Performance)

Create indexing.ldif:

cat <<EOF > indexing.ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcDbIndex
olcDbIndex: objectClass eq
olcDbIndex: uid pres,eq
olcDbIndex: mail pres,eq
olcDbIndex: cn pres,eq,sub
olcDbIndex: sn pres,eq,sub
olcDbIndex: memberUid eq
EOF

Apply the indexes:

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f indexing.ldif

Phase 13: Final Connectivity Test (The "External" Check)
Now, test if your admin user can still search, but an anonymous user cannot.

Test 1 (Should Fail - Anonymous):
ldapsearch -x -b "dc=lab,dc=local"
(Result should be: "Inappropriate authentication")

Test 2 (Should Pass - Authenticated):
ldapsearch -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -b "dc=lab,dc=local"
(Result should be your 5 entries)

Configure SSL/TLS (LDAPS)

1. Generate the Certificates
We'll create a dedicated directory to store your keys.

# 1. Create directory and set permissions
sudo mkdir -p /etc/openldap/certs
sudo chown ldap:ldap /etc/openldap/certs
sudo chmod 700 /etc/openldap/certs

# 2. Generate a Self-Signed Certificate (Valid for 10 years)
# Note: When prompted for "Common Name", use your server's FQDN or IP (192.168.0.112)
sudo openssl req -new -x509 -nodes -out /etc/openldap/certs/ldapcert.pem \
-keyout /etc/openldap/certs/ldapkey.pem -days 3650

# 3. Set proper ownership so the 'ldap' user can read the private key
sudo chown ldap:ldap /etc/openldap/certs/*.pem
sudo chmod 600 /etc/openldap/certs/ldapkey.pem

Apply TLS Configuration to LDAP
Now we tell the running LDAP server where to find these files
# 1. Create certs.ldif
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

# 2. Apply to the config database
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f certs.ldif

Enable the LDAPS Port (636)
By default, Rocky 9 might only be listening on port 389. We need to tell the service to start the secure listener.

# 1. Edit the sysconfig file
sudo vi /etc/sysconfig/slapd

# 2. Ensure SLAPD_URLS includes ldaps:///
# It should look like this:
SLAPD_URLS="ldapi:/// ldap:/// ldaps:///"

# 3. Restart the service
sudo systemctl restart slapd

# 4. Open the firewall for LDAPS
sudo firewall-cmd --add-service=ldaps --permanent
sudo firewall-cmd --reload

Verify the Secure Connection

ss -tulnp | grep 636


Then, run a secure search.

Note: Because it's a self-signed cert, we use LDAPTLS_REQCERT=never to tell the client "I trust this specific certificate even if it's not from a major CA."

Bash
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://localhost -D "cn=admin,dc=lab,dc=local" -w "Telco666" -b "dc=lab,dc=local"

"To make clients trust this certificate permanently (so you don't need REQCERT=never), copy /etc/openldap/certs/ldapcert.pem to the client machine and add it to /etc/openldap/ldap.conf using the TLS_CACERT directive."

Phase 12: Automated Daily Backups (Cron Job)
1. Create the Backup Directory
We'll store the backups in a dedicated folder and ensure the ldap user has access.

sudo mkdir -p /var/backups/ldap
sudo chown ldap:ldap /var/backups/ldap
sudo chmod 700 /var/backups/ldap

Create the Backup Script

We will create a script that exports both the Config database and the User database, then compresses them with a timestamp.

cat << 'EOF' | sudo tee /usr/local/bin/ldap_backup.sh
#!/bin/bash
# LDAP Backup Script - Shashi's Lab
BACKUP_DIR="/var/backups/ldap"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# 1. Export the User Database (dc=lab,dc=local)
/sbin/slapcat -n 2 -l ${BACKUP_DIR}/ldap_data_${TIMESTAMP}.ldif

# 2. Export the Config Database (cn=config)
/sbin/slapcat -n 0 -l ${BACKUP_DIR}/ldap_config_${TIMESTAMP}.ldif

# 3. Compress the files
gzip ${BACKUP_DIR}/*.ldif

# 4. Housekeeping: Delete backups older than 7 days
find ${BACKUP_DIR} -name "*.gz" -mtime +7 -delete
EOF

# Make the script executable
sudo chmod +x /usr/local/bin/ldap_backup.sh

3. Schedule the Cron Job
We will set this to run every night at 2:00 AM.

# Open the crontab for the root user
sudo crontab -e

# Add this line to the bottom:
00 02 * * * /usr/local/bin/ldap_backup.sh > /dev/null 2>&1

Bulk Identity & Group Expansion

Create expansion.ldif
We’ll add a "Lead Admin" and a "Viewer" user to test different permission levels.


cat <<EOF > expansion.ldif
# --- Additional Users ---

# User 2: Senior Admin (Aiman)
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

# User 3: Junior Monitor (Siva)
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

# --- Additional Groups ---

# Group 2: Viewers (Read-only access for Grafana)
dn: cn=viewers,ou=Groups,dc=lab,dc=local
objectClass: posixGroup
cn: viewers
gidNumber: 2001
memberUid: siva

# Update: Add Aiman to the existing Admin Group
dn: cn=admins,ou=Groups,dc=lab,dc=local
changetype: modify
add: memberUid
memberUid: aiman
EOF

Apply the Expansion
ldapmodify -x -D "cn=admin,dc=lab,dc=local" -w "Telco666" -a -f expansion.ldif

Note: I used ldapmodify -a (add) here because one of the entries is an update to an existing group, while the others are new.

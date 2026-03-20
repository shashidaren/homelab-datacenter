🔐 OpenLDAP Lab Guide (Verified Build)
<p align="center">
<b>Rocky Linux 9 • LDAPS • Identity Management Lab</b>


Production-style LDAP deployment with full configuration and security hardening
</p>

🧱 Phase 1 — Installation
Bash
sudo dnf config-manager --set-enabled plus
sudo dnf install -y openldap-servers openldap-clients
sudo systemctl enable --now slapd

# OpenLDAP-Based Centralised Identity Management on AlmaLinux 9
This homelab project sets up a centralised user authentication using OpenLDAP with AlmaLinux 9 VM as the server and Lubuntu VM as the client. It’s optimized for low-resource environments, ideal for single-laptop homelabs and virtual setups.


1. [AlmaLinux 9 Server VM Setup in VirtualBox](#almalinux-9-server-vm-setup-in-virtualbox)
2. [OpenLDAP Installation and Configuration](#openldap-installation-and-configuration)
3. [Lubuntu Client VM Integration](#lubuntu-client-vm-integration)
4. [Testing and Validation](#testing-and-validation)
5. [Multi-Master Replication for High Availability](#multi-master-replication-for-high-availability)
6. [Automated LDIF Backups and Restores](#automated-ldif-backups-and-restores)


## AlmaLinux 9 Server VM Setup in VirtualBox
- Download AlmaLinux OS from [https://almalinux.org/get-almalinux/](https://almalinux.org/get-almalinux/). Since this entire project is done on a single laptop, we will go with downloading AlmaLinux OS 9.5 Minimal ISO to ensure minimal resources are used <br />
  ![image](https://github.com/user-attachments/assets/a411d1d8-e14e-4368-af52-df7ba3aee008)
  
- After downloading is completed, open VirtualBox and click New to create a VM. Select the downloaded ISO of AlmaLinux 9 and name it. Skip unattended installation to configure disk size and click Next <br />
  ![image](https://github.com/user-attachments/assets/6677f79e-5a83-4648-bac2-8083a0cd9804)

- Set the base memory to 2GB and number of processors to 2 CPUs <br />
  ![image](https://github.com/user-attachments/assets/76db5dbb-0723-4d4f-98d3-e9cef69eb3c3)
  
- Set the disk size to 8GB <br />
  ![image](https://github.com/user-attachments/assets/2e3d50ca-8ee7-4e4c-9c3e-84346a93b6e0)

- After choosing the right settings, the summary is shown as below. Click Finish to end the VM setup <br />
  ![image](https://github.com/user-attachments/assets/bd81148c-5972-4325-a8ec-0a33406cf275)

- In the VM network settings, change NAT to Bridged Adapter <br />
  ![image](https://github.com/user-attachments/assets/1039f90b-fac8-4a39-8eba-7eb493ab6387)

- Start AlmaLinux 9 VM. Choose the preferred language <br />
  ![image](https://github.com/user-attachments/assets/a84c8218-5ca7-4966-80bd-504933f5af9f)

- The Installation Summary page is shown below <br />
  ![image](https://github.com/user-attachments/assets/f8107d40-53de-4f33-ba3b-7f03a73ca47c)

- In the Installation Destination, select the disk <br />
  ![image](https://github.com/user-attachments/assets/5cce5527-e0b2-40bf-85e5-7d67ae7661f7)

- For Software Selection, choose the default settings, as we don't need to install unnecessary things for the sake of this homelab project
  ![image](https://github.com/user-attachments/assets/33f024a5-5a43-4609-b4ce-49684475d256)

- Create a new root password <br />
  ![image](https://github.com/user-attachments/assets/fadd0027-451c-4dca-a952-8d10ffd2c687)

- Create a new user. Add in the password of the new user too. After this step is completed, click Begin Installation <br />
  ![image](https://github.com/user-attachments/assets/87229d0a-f5fd-4458-9ec7-19d4ee8a79d8)

- Once the installation is complete, reboot the system <br />
  ![image](https://github.com/user-attachments/assets/f56a2b89-d9ac-4372-a348-d08435057a6c)

- AlmaLinux 9 is now ready for use after logging in with the user credentials created earlier <br />
  ![image](https://github.com/user-attachments/assets/eb9f8805-794c-4b5c-80e1-be30297f35df)

  

## OpenLDAP Installation and Configuration
- Install and enable the required packages
  ```
  sudo dnf install -y openldap-servers openldap-clients migrationtools
  ```
  `openldap-servers` and `openldap-clients` are core LDAP daemons and tools
  
- Enable and start the OpenLDAP server
  ```
  sudo systemctl enable --now slapd
  sudo systemctl status slapd
  ```
  
- Set the LDAP admin password
  ```
  slappasswd
  ```
  Save the hashed password output for the next step

- Configure LDAP root DN by creating a file named `chrootpw.ldif`
  ```
  dn: olcDatabase={2}mdb,cn=config
  changetype: modify
  replace: olcRootPW
  olcRootPW: <hashed_password_here>
  ```
  Apply the config
  ```
  sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif
  ```

- Set the base domain by replacing `example.com` with the actual domain
  ```
  export BASEDN="dc=example,dc=com"
  export ADMIN="cn=admin,$BASEDN"
  ```

- Create `basedomain.ldif`
  ```
  dn: $BASEDN
  objectClass: top
  objectClass: dcObject
  objectclass: organization
  o: Example Org
  dc: example
  
  dn: $ADMIN
  objectClass: organizationalRole
  cn: admin
  description: LDAP Admin
  ```
  Apply
  ```
  ldapadd -x -D "$ADMIN" -W -f basedomain.ldif
  ```

- Configuring LDAP TLS is optional but highly recommended. Generate or obtain a certificate and configure TLS in `/etc/openldap/slapd.d/cn=config.ldif` or via `ldapmodify`



## Lubuntu Client VM Integration
- Install LDAP client packages
  ```
  sudo apt update
  sudo apt install -y libnss-ldap libpam-ldap ldap-utils nslcd
  ```
  During setup prompts:
  - LDAP server URI: ldap://<AlmaLinux IP>
  - Base DN: dc=example,dc=com
  - LDAP version: 3
  - Make local root Database admin: Yes
  - LDAP account for root: cn=admin,dc=example,dc=com
- Update NSS to use LDAP. Edit `/etc/nsswitch.conf`
  ```
  passwd:         files ldap
  group:          files ldap
  shadow:         files ldap
  ```

- Enable home directory creation
  ```
  sudo pam-auth-update
  ```
  Select "Create home directory on login"

- Restart necessary services
  ```
  sudo systemctl restart nslcd
  ```



## Testing and Validation
- Add LDAP users by creating `newuser.ldif`
  ```
  dn: uid=john,ou=People,dc=example,dc=com
  objectClass: inetOrgPerson
  objectClass: posixAccount
  objectClass: shadowAccount
  cn: John Doe
  sn: Doe
  uid: john
  uidNumber: 10001
  gidNumber: 10001
  homeDirectory: /home/john
  loginShell: /bin/bash
  userPassword: {SSHA}<hashed_pass_here>
  ```

- Add user
  ```
  ldapadd -x -D "$ADMIN" -W -f newuser.ldif
  ```

- Test the login from Lubuntu. Use the following command to see if the user exists
  ```
  getent passwd john
  ```
  Try logging in as john
  
  


## Multi-Master Replication for High Availability
- This task requires a second AlmaLinux LDAP server (let's call it ldap2). Enable syncprov overlay by creating `syncprov.ldif`
  ```
  dn: olcOverlay=syncprov,olcDatabase={2}mdb,cn=config
  objectClass: olcOverlayConfig
  objectClass: olcSyncProvConfig
  olcOverlay: syncprov
  olcSpCheckpoint: 100 10
  olcSpSessionLog: 100
  ```

  Apply it
  ```
  ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov.ldif
  ```
  
- Configure replication consumer on ldap2. Replace the values as necessary
  ```
  dn: olcDatabase={2}mdb,cn=config
  changetype: modify
  add: olcSyncRepl
  olcSyncRepl: rid=001
    provider=ldap://ldap1.example.com
    bindmethod=simple
    binddn="cn=admin,dc=example,dc=com"
    credentials=yourpassword
    searchbase="dc=example,dc=com"
    type=refreshAndPersist
    retry="60 +"
    timeout=1
  ```

  Also
  ```
  add: olcMirrorMode
  olcMirrorMode: TRUE
  ```

- Restart slapd on both
  ```
  sudo systemctl restart slapd
  ```

- Test by adding a user on ldap1 and verifying it appears on ldap2





## Automated LDIF Backups and Restores
- To automate backups on AlmaLinux, create `/opt/ldap_backup.sh`
  ```
  #!/bin/bash
  DATE=$(date +%F)
  BACKUP_DIR="/var/backups/openldap"
  mkdir -p $BACKUP_DIR
  slapcat -v -l "$BACKUP_DIR/ldap-backup-$DATE.ldif"
  ```
  Make it executable
  ```
  chmod +x /opt/ldap_backup.sh
  ```

- Add the script to cron
  ```
  crontab -e
  ```
  And add
  ```
  0 2 * * * /opt/ldap_backup.sh
  ```

- To restore from LDIF, use
  ```
  sudo systemctl stop slapd
  sudo mv /var/lib/ldap /var/lib/ldap.bak
  sudo mkdir /var/lib/ldap
  sudo chown ldap:ldap /var/lib/ldap
  sudo slapadd -v -l /path/to/backup.ldif
  sudo systemctl start slapd
  ```
  

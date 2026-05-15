# SOP: DRC EMR/OpenMRS Bare-Metal Installation on Ubuntu

## 1. Purpose

This SOP describes how to install and bring up the bundled DRC EMR/OpenMRS distribution for the first time on a bare-metal Ubuntu server.

## 2. Scope

Use this SOP for a fresh Ubuntu server installation using this package. The bundled release directory still carries its upstream name on the release section:

```text
DRCEMR_19.3.4-Release_14_MAY_2026
```

The procedure installs and configures:

- Java 11
- Tomcat 9
- MySQL Server
- OpenMRS runtime directory
- Blank OpenMRS database
- DRC EMR modules, frontend assets, configuration, and WAR file
- Backup scripts and scheduled backup cron job

## 3. Recommended Environment

Use:

- Ubuntu 22.04 LTS
- 32 GB RAM
- 1 TB storage
- Stable internet access during package installation
- A sudo-capable Linux user

The installer is tuned for this server profile:

- Tomcat JVM heap: 4 GB initial, 8 GB maximum
- Tomcat Metaspace: 512 MB initial, 1 GB maximum
- MySQL InnoDB buffer pool: 12 GB

This leaves memory for Ubuntu, filesystem cache, OpenMRS module startup, scheduled backups, and maintenance commands.

Avoid Ubuntu 24.04 

## 4. Pre-Installation Checklist

Confirm the server is ready:

```bash
lsb_release -a
free -h
df -h
ip addr
sudo -v
```

Confirm Java alternatives before installation or troubleshooting Tomcat startup:

```bash
java -version
javac -version
sudo update-alternatives --display java
sudo update-alternatives --display javac
```

If another Java version is selected, switch to Java 11:

```bash
sudo update-alternatives --config java
sudo update-alternatives --config javac
```

For a non-interactive update on Ubuntu 22.04 amd64, use:

```bash
sudo update-alternatives --set java /usr/lib/jvm/java-11-openjdk-amd64/bin/java
sudo update-alternatives --set javac /usr/lib/jvm/java-11-openjdk-amd64/bin/javac
java -version
```

Expected Java major version: `11`.

Confirm the installation package contains the required files:

```bash
ls -l Prerequisites/installsetup.bash
ls -l Prerequisites/db/Blankdb_19.3.4.sql
ls -l DRCEMR_19.3.4-Release_14_MAY_2026/webapp/openmrs.war
ls -l DRCEMR_19.3.4-Release_14_MAY_2026/modules.tar.gz
ls -l DRCEMR_19.3.4-Release_14_MAY_2026/frontend.tar.gz
```

## 5. Installation Variables

Default values:

```bash
MYSQL_USER=root
MYSQL_PASSWORD=test
MYSQL_DATABASE=openmrs
```

For a standard first-time installation, keep the defaults because the bundled scripts and database are prepared for them.

To use a different MySQL password:

```bash
sudo MYSQL_PASSWORD='your-password' bash Prerequisites/installsetup.bash
```

## 6. Access the Deployment Package

Copy or extract the deployment package to any working directory on the Ubuntu server. Then enter the package root, which is the directory that contains `Prerequisites/`, `DRCEMR_19.3.4-Release_14_MAY_2026/`, `README.md`, and this SOP.

Example:

```bash
cd /path/to/DRC_EMR_3.X_BAREMETAL_SETUP_MAY_13_2026
ls
```

Expected package-root contents include:

```text
Prerequisites/
DRCEMR_19.3.4-Release_14_MAY_2026/
README.md
SOP_DRC_EMR_BAREMETAL_UBUNTU_INSTALL.md
```

## 7. First-Time Installation Procedure

Before running the DRC EMR installer, update the Ubuntu package lists and apply OS package upgrades. This step can take time on a fresh server, so run it separately before timing or troubleshooting the application installer.

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

After the server comes back up, return to the package root and run the installer:

```bash
cd /path/to/DRC_EMR_3.X_BAREMETAL_SETUP_MAY_13_2026
sudo bash Prerequisites/installsetup.bash
```

The installer performs these steps:

1. Installs Ansible if missing.
2. Installs Java 11 and Tomcat 9.
3. Creates `/var/lib/OpenMRS`.
4. Installs and starts MySQL.
5. Configures MySQL for OpenMRS.
6. Creates the `openmrs` database.
7. Restores the bundled blank OpenMRS SQL dump.
8. Writes `/var/lib/OpenMRS/openmrs-runtime.properties`.
9. Installs backup scripts.
10. Deploys the DRC EMR/OpenMRS WAR, modules, frontend assets, and configuration.
11. Starts Tomcat.

## 8. Expected Successful Result

At the end of the installer, this URL should be available from the server:

```text
http://localhost:8080/openmrs/
```

From another machine on the network, use:

```text
http://SERVER_IP:8080/openmrs/
```

The first startup can take several minutes while OpenMRS initializes modules and database changes.

## 9. Verification Steps

Check Tomcat:

```bash
sudo systemctl status tomcat9
```

Check MySQL:

```bash
sudo systemctl status mysql
```

Check the OpenMRS database:

```bash
mysql -uroot -ptest -e 'SHOW DATABASES;'
mysql -uroot -ptest openmrs -e 'SHOW TABLES LIMIT 10;'
```

Watch OpenMRS startup logs:

```bash
sudo tail -f /var/log/tomcat9/catalina.out
```

Confirm OpenMRS runtime files:

```bash
sudo ls -l /var/lib/OpenMRS
sudo ls -l /var/lib/OpenMRS/modules
sudo ls -l /var/lib/OpenMRS/frontend
sudo ls -l /var/lib/OpenMRS/configuration
```

## 10. Post-Upgrade SQL Fixes

After OpenMRS has started successfully and module initialization has settled, run:

```bash
sudo bash DRCEMR_19.3.4-Release_14_MAY_2026/post_upgrade.sh
```

Then restart Tomcat:

```bash
sudo service tomcat9 restart 
```

Watch the logs again:

```bash
sudo tail -f /var/log/tomcat9/catalina.out
```

## 11. Redeploy DRC EMR Only

Use this when the server already has prerequisites installed and you only need to redeploy the bundled DRC EMR release:

```bash
sudo bash DRCEMR_19.3.4-Release_14_MAY_2026/setup_script.sh
```

This redeploys:

- OpenMRS WAR file
- OMOD modules
- Frontend assets
- Configuration files
- CSRF guard properties

## 12. Reinstall or Recover a Partial First Install

If the database already exists and has tables, the installer skips database restore by default.

To force a blank database restore:

```bash
sudo FORCE_DB_RESTORE=1 bash Prerequisites/installsetup.bash
```

Warning: this drops and recreates the configured OpenMRS database.

If `/var/lib/OpenMRS` already exists, the installer moves it to a timestamped backup path such as:

```text
/var/lib/OpenMRS.preinstall.YYYYMMDDHHMMSS
```

## 13. Backup Verification

The installer places backup tools here:

```text
/usr/share/openmrs-backup-tools
```

Backup destination:

```text
/var/backups/DRC_EMR
```

Confirm cron entry:

```bash
sudo crontab -l
```

Expected schedule:

```text
00 11,16 * * * /usr/share/openmrs-backup-tools/openmrs_backup.sh > /dev/null 2>&1
```

Run a manual backup test:

```bash
sudo /usr/share/openmrs-backup-tools/openmrs_backup.sh
sudo ls -lh /var/backups/DRC_EMR
```

## 14. Service Operations

Start Tomcat:

```bash
 sudo service tomcat9 start
```

Stop Tomcat:

```bash
 sudo service tomcat9 stop
```

Restart Tomcat:

```bash
sudo systemctl restart tomcat9
```

Enable Tomcat on boot:

```bash
sudo systemctl enable tomcat9
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

## 15. Troubleshooting

If OpenMRS does not load, check Tomcat status:

```bash
sudo systemctl status tomcat9
```

Check logs:

```bash
sudo tail -n 200 /var/log/tomcat9/catalina.out
```

If MySQL login fails:

```bash
mysql -uroot -ptest -e 'SELECT 1;'
```

If the WAR is not deployed:

```bash
sudo ls -l /var/lib/tomcat9/webapps
```

Expected:

```text
openmrs.war
openmrs/
```

If Tomcat logs show `openmrs.war (Permission denied)`, remove the failed extracted app, give Tomcat ownership of the full `webapps` directory, and restart Tomcat:

```bash
 sudo service tomcat9 stop
sudo rm -rf /var/lib/tomcat9/webapps/openmrs
sudo chown -R tomcat:tomcat /var/lib/tomcat9/webapps
sudo chmod 0755 /var/lib/tomcat9/webapps
sudo chmod 0644 /var/lib/tomcat9/webapps/openmrs.war
sudo systemctl start tomcat9
sudo tail -f /var/log/tomcat9/catalina.out
```

If modules are missing:

```bash
sudo ls -l /var/lib/OpenMRS/modules
```

If frontend assets are missing:

```bash
sudo ls -l /var/lib/OpenMRS/frontend
```

If permissions are wrong:

```bash
sudo chown -R tomcat:tomcat /var/lib/OpenMRS
sudo chmod -R 755 /var/lib/OpenMRS
sudo chmod 640 /var/lib/OpenMRS/openmrs-runtime.properties
sudo systemctl restart tomcat9
```

## 16. Firewall

If the server firewall is enabled, allow Tomcat port 8080:

```bash
sudo ufw allow 8080/tcp
sudo ufw status
```

## 17. Handover Checklist

Before handing over the server, confirm:

- OpenMRS loads in the browser.
- Tomcat is enabled and running.
- MySQL is enabled and running.
- `/var/lib/OpenMRS/openmrs-runtime.properties` exists.
- Modules exist in `/var/lib/OpenMRS/modules`.
- Frontend assets exist in `/var/lib/OpenMRS/frontend`.
- Configuration files exist in `/var/lib/OpenMRS/configuration`.
- Backup cron job exists.
- A manual backup has been tested.
- Server IP address and OpenMRS URL are documented.

## 18. Key Paths

```text
/var/lib/OpenMRS
/var/lib/OpenMRS/openmrs-runtime.properties
/var/lib/OpenMRS/modules
/var/lib/OpenMRS/frontend
/var/lib/OpenMRS/configuration
/var/lib/tomcat9/webapps
/var/log/tomcat9/catalina.out
/usr/share/openmrs-backup-tools
/var/backups/DRC_EMR
```

## 19. Standard Installation Command

For a normal first-time installation:

```bash
cd /path/to/DRC_EMR_3.X_BAREMETAL_SETUP_MAY_13_2026
sudo bash Prerequisites/installsetup.bash
```

Then open:

```text
http://SERVER_IP:8080/openmrs/
```

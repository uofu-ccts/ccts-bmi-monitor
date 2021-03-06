# Icinga2/Icinga Web 2 Setup Guide

This guide was made assuming a clean install of CentOS 7 and the use of firewalld instead of iptables. Instructions for other distros are available online; here are some of the better resources I was able to find. 

- [Official Icinga Installation Guide](https://icinga.com/docs/icinga2/latest/doc/02-installation/)
- [Debian/Ubuntu (Computing for Geeks)](https://computingforgeeks.com/how-to-install-icinga2-monitoring-tool-on-ubuntu-18-04-lts/)

For the purposes of CHPC monitoring, we chose (at this time) to do agentless monitoring via SNMP. As such, we don't make use of many key Icinga features, i.e. Icinga Agent/Icinga Director.

## Script Installation
I've created a script which can be run using the following commands (requires root). Again, this script assumes the use of firewalld. If you are using iptables instead, the script will fail and you will need to go through the manual setup; right now I don't do any error checking in the script, so if it fails at any point it's pretty hard to tell. This should be added in the future, but for now it's been tested and shown to work on clean installs of CentOS 7 with firewalld as the firewall; other configurations haven't been tested and may or may not work (but iptables is known to cause failure).
```
git clone https://github.com/andrewtakeshi/bmi.git ./bmi
cd bmi/icinga.d/initial
./icinga_setup
```

## Icinga2 Manual Installation
If for whatever reason the script fails or you want to manually follow the steps, do the following (requires root/sudo access): 

1. Disable SELinux; with SELinux enabled, we are unable to run most CheckCommands. The sed command replaces the line 'SELINUX=...' with 'SELINUX=disabled' in /etc/selinux/config. You can change this manually if you would prefer to do so.
```
sed -i 's/^SELINUX=[[:alpha:]]\+$/SELINUX=disabled/' /etc/selinux/config
setenforce 0
```
2. Add the epel-release repository.
```
yum install -y epel-release
```
3. Add the Icinga2 Official Repository, install Icinga2, and install the latest Nagios plugins package.  
```
rpm --import https://packages.icinga.com/icinga.key
yum install -y https://packages.icinga.com/epel/icinga-rpm-release-7-latest.noarch.rpm
yum install -y icinga2 nagios-plugins-all
```
4. Start Icinga2. 
```
systemctl enable icinga2 --now
```
5. Install and start mariaDB (MySQL).
```
yum install -y mariadb-server mariadb
systemctl enable mariadb --now

# Optional, but highly recommended
mysql_secure_installation
```
6. Install the Icinga2 IDO Modules for MySQL.
```
yum install -y icinga2-ido-mysql
```
7. Create a database for the Icinga2 IDO Modules and import the Icinga2 IDO Schema into the database. 
```
mysql -u root -p
# Inside MariaDB (MySQL)
# Feel free to substitute your own database names/passwords
MariaDB> CREATE DATABASE icinga2;
MariaDB> GRANT ALL PRIVILEGES ON icinga2.* TO icinga2@localhost IDENTIFIED BY 'icinga123';
MariaDB> EXIT;
# Outside of MariaDB
mysql icinga2 < /usr/share/icinga2-ido-mysql/schema/mysql.sql
```
8. Enable necessary features in Icinga2.
```
icinga2 feature enable ido-mysql command
```
9. Configure the firewall.
```
firewall-cmd --permanent --add-port=5665/tcp --zone=public
firewall-cmd --reload
```
10. Set up IDO MySQL connection.
```
# Run this, assuming no changes to database names/passwords were made. 
printf "object IdoMysqlConnection \"ido-mysql\" {\n\tuser = \"icinga2\",\n\tpassword=\"icinga123\",\n\thost=\"localhost\",\n\tdatabase = \"icinga2\"\n}" > /etc/icinga2/features-enabled/ido-mysql.conf

# You can also use this to replace the contents of /etc/icinga2/features-enabled/ido-mysql.conf. 
# This again assumes no changes to database names/passwords were made. 

# Copy this to /etc/icinga2/features-enabled/ido-mysql.conf: 
object IdoMysqlConnection "ido-mysql" {
  user="icinga2"
  password="icinga123"
  host="localhost"
  database="icinga2"
}
```
11. (CHPC/SNMP Specific) Copy files to correct locations to allow for better SNMP-based monitoring. These allow us to easily monitor things like RAM, CPU, and swap usage via SNMP. There are also scripts for checking the number of running processes, and for checking if specific processes are running. 
```
# Clone the git repo, if not already done.
git clone https://github.com/andrewtakeshi/bmi.git

# Go to the correct directory
cd bmi/icinga.d/initial

# Copy the scripts over. 
cp -r custom_scripts/* /usr/lib64/nagios/plugins
```
12. Restart Icinga2.
```
systemctl restart icinga2
```

At this point, Icinga2 is set up. The setup of IcingaWeb2 is completely optional, as IcingaWeb2 is just a way of visualizing Icinga2. As a side note, IcingaWeb2 is set up to communicate with Icinga2 through the Icinga2 API. There are other methods of communication available, but the API is the recommended method of communication. 

## IcingaWeb2 Manual Installation

If you wish to install IcingaWeb2, do the following:

1. Enable the SCL Repository.
```
yum install -y centos-release-scl
```
2. Install required PHP packages (used for visualization in IcingaWeb2).
```
yum install -y rh-php71-php-json rh-php71-php-pgsql rh-php71-php-xml rh-php71-php-intl rh-php71-php-common rh-php71-php-pdo rh-php71-php-mysqlnd rh-php71-php-cli rh-php71-php-mbstring rh-php71-php-fpm rh-php71-php-gd rh-php71-php-zip rh-php71-php-ldap rh-php71-php-imagick
```
3. Modify php.ini to have the correct time zone. A list of supported time zones can be found [here](https://www.php.net/manual/en/timezones.php). The command below sets the time zone to America/Denver, but this can easily be modified to the correct time zone. 
```
sed -in 's/;date.timezone=/date.timezone = America\/Denver/g' /etc/opt/rh/rh-php71/php.ini
```
4. Enable and start php-fpm. 
```
systemctl enable rh-php71-php-fpm --now
```
5. Install icingaweb, icingacli, and httpd, and enable httpd. 
```
yum install -y icingaweb2 icingacli httpd
systemctl enable httpd --now
```
6. Open the firewall to allow web access.
```
firewall-cmd --permanent --add-service=http --zone=public
firewall-cmd --reload
```
7. Modify group settings and do some initial setup. 
```
groupadd -r icingaweb2
usermod -a -G icingaweb2 apache
icingacli setup config directory --group icingaweb2
systemctl restart httpd rh-php71-php-fpm
```
9. Create the IcingaWeb2 database. 
```
mysql -u root -p
# Inside MariaDB (MySQL)
# Feel free to substitute your own database names/passwords
MariaDB> CREATE DATABASE icingawebdb;
MariaDB> GRANT ALL PRIVILEGES ON icingawebdb.* TO icingaweb@localhost IDENTIFIED BY 'icinga123';
MariaDB> FLUSH PRIVILEGES;
MariaDB> EXIT;
```
10. Setup Icinga2 API to allow communication to IcingaWeb2.
```
icinga2 api setup
systemctl restart icinga2
PASS=$(grep password /etc/icinga2/conf.d/api-users.conf -m 1 | sed 's/.*"\(.*\)".*/\1/g')
TOKEN=$(icingacli setup token create)
```
12. Assuming that no default settings were changed, run the following to get the settings required for finishing setup of IcingaWeb2. IcingaWeb2 is accessible at [http://localhost/icingaweb2](http://localhost/icingaweb2). 
```
printf "Setup Token:\n\t$TOKEN\n" &&                                                                                                                       
printf "Database Resource:\n\tdb name = icingawebdb\n\tuser = icingaweb\n\tpassword = icinga123\n" &&                                                 
printf "IDO Resource:\n\tdb name = icinga2\n\tuser = icinga2\n\tpassword = icinga123\n" &&                                                        
printf "API Configuration:\n\thost = localhost\n\tuser = root\n        password = $PASS\n" 
```

## Remote Device Setup (SNMP)
This section contains instructions for setting up SNMP to work on remote nodes; this section doesn't apply to the master (device on which Icinga2 is installed), unless you're also monitoring the master from itself. 

Setup/Install using the script (assuming the repo has already been cloned)
```
git clone https://github.com/andrewtakeshi/bmi.git ./bmi
cd bmi/icinga.d/initial
./snmp_setup
```
Alternatively, install manually: 

1. Install SNMP tools, if not already installed.
```
sudo yum install -y net-snmp net-snmp-utils
```
2. Start and enable the SNMP daemon.
```
sudo systemctl enable snmpd --now
```
3. For CHPC, we need access to the host-resources-mib and the ucd-snmp-mib. These MIBs should be installed along with net-snmp-utils, but they need to be enabled for query with the following command:
```
sudo echo "view systemview included .1.3.6.1.2.1.25" >> /etc/snmp/snmpd.conf
sudo echo "view systemview included .1.3.6.1.4.1.2021" >> /etc/snmp/snmpd.conf
sudo systemctl restart snmpd 
```
4. Check to see if the changes have been applied successfully with the following command. If the ucd-snmp-mib is loaded properly, it will print 'SUCCESS'. Otherwise, it will print 'FAIL'. This assumes the use of SNMP V2, with 'public' as the community string. 
```
snmpget -v 2c -c public localhost laLoad.1 | grep -q "STRING" && echo "SUCCESS" || echo "FAIL"
```
5. Open the firewall to allow for access from the Icinga master (or other SNMP agents):
```
sudo firewall-cmd --add-service=snmp --zone=public --permanent
sudo firewall-cmd --reload
```

# Nagios Installation and Configuration Guide
# Current Date: April 04, 2025

## PREREQUISITE
### Instance Specifications
- Machine Type: e2-standard-2 (2 vCPUs, 8GB memory) or higher
- Disk Size: Minimum 30GB, preferably 50GB for production
- OS: Ubuntu 22.04 LTS or 24.04 LTS
- Root or sudo access with PORT 80 (HTTP) allowed

## INSTALLATION

### Part 1: Setting up Nagios Core on First VM
#### Install Dependencies and Nagios Core
- sudo su -
- sudo apt update
- sudo apt-get install -y autoconf gcc libc6 make wget unzip apache2 php libapache2-mod-php libgd-dev ufw
- sudo apt-get install -y openssl libssl-dev
- cd /tmp
- wget -O nagioscore.tar.gz $(wget -q -O - https://api.github.com/repos/NagiosEnterprises/nagioscore/releases/latest | grep '"browser_download_url":' | grep -o 'https://[^"]*')
- tar xzf nagioscore.tar.gz
- cd /tmp/nagios-*
- sudo ./configure --with-httpd-conf=/etc/apache2/sites-enabled
- sudo make all
- sudo make install-groups-users
- sudo usermod -a -G nagios www-data
- sudo make install
- sudo make install-daemoninit
- sudo make install-commandmode
- sudo make install-config
- sudo make install-webconf
- sudo a2enmod rewrite
- sudo a2enmod cgi
- sudo ufw allow apache
- sudo ufw reload
- sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin # Need to set password

# Note: Skip password step for default "nagios:nagios" or set password for "nagiosadmin:your_password"

#### Install Nagios Plugins
- cd ..
- sudo apt-get install -y autoconf gcc libc6 libmcrypt-dev make libssl-dev wget bc gawk dc build-essential snmp libnet-snmp-perl gettext
- wget -O nagios-plugins.tar.gz $(wget -q -O - https://api.github.com/repos/nagios-plugins/nagios-plugins/releases/latest | grep '"browser_download_url":' | grep -o 'https://[^"]*')
- tar zxf nagios-plugins.tar.gz
- cd /tmp/nagios-plugins-*
- sudo ./configure
- sudo make
- sudo make install

#### Manage Nagios Service
- sudo systemctl start nagios.service
- sudo systemctl stop nagios.service
- sudo systemctl restart nagios.service
- sudo systemctl status nagios.service

#### Access Nagios Web Interface
# Use: http://your_server_external_ip/nagios
# Example: http://1.2.3.4/nagios

### Additional Requirements
#### Install NRPE Plugin on Nagios Server
- cd /tmp
- wget -O nrpe.tar.gz https://github.com/NagiosEnterprises/nrpe/archive/nrpe-4.1.0.tar.gz
- tar xzf nrpe.tar.gz
- cd nrpe-4.1.0/
- sudo ./configure --enable-command-args
- sudo make all
- sudo make install-plugin

#### Configure NRPE Monitoring
- sudo mkdir -p /usr/local/nagios/etc/servers
- sudo nano /usr/local/nagios/etc/servers/nrpe-server.cfg  # Define host and service for second server
- sudo nano /usr/local/nagios/etc/nagios.cfg              # Edit: cfg_dir=/usr/local/nagios/etc/servers
- sudo nano /usr/local/nagios/etc/objects/commands.cfg    # Define NRPE command:
# Add:
 define command {
    command_name check_nrpe
   command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
 }

#### Verify and Restart
- sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
- sudo systemctl restart nagios

### Part 2: Setting up NRPE on Second VM
#### Step 1: Install NRPE and Plugins
- sudo apt update
- sudo apt-get install -y nagios-nrpe-server nagios-plugins
- sudo apt-get install -y nagios-plugins-basic nagios-plugins-standard

#### Step 2: Configure NRPE
sudo nano /etc/nagios/nrpe.cfg
# Modify:
- allowed_hosts=127.0.0.1,NAGIOS_SERVER_IP
- Configure check commands:
- command[check_users]=/usr/lib/nagios/plugins/check_users -w 5 -c 10
- command[check_load]=/usr/lib/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
- command[check_disk]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /
- command[check_total_procs]=/usr/lib/nagios/plugins/check_procs -w 150 -c 200
- command[check_ssh]=/usr/lib/nagios/plugins/check_ssh -p 22 localhost

#### Step 3: Manage NRPE Service
- sudo systemctl restart nagios-nrpe-server
- sudo systemctl enable nagios-nrpe-server
- sudo systemctl status nagios-nrpe-server
- sudo ufw allow 5666/tcp
- sudo ufw reload

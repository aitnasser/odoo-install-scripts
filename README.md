# odoo-install-scripts
# [Odoo](https://www.odoo.com "Odoo's Homepage") Install Script

This script is based on the install script from André Schenkels (https://github.com/aschenkels-ictstudio/openerp-install-scripts)
but goes a bit further and has been improved. This script will also give you the ability to define an xmlrpc_port in the .conf file that is generated under /etc/
This script can be safely used in a multi-odoo code base server because the default Odoo port is changed BEFORE the Odoo is started.

## Installation procedure

##### 1. Download the script odoo_install.sh or clone the repo:
```
git clone  https://github.com/aitnasser/odoo-install-scripts.git
```
##### 2. Modify the parameters as you wish.
There are a few things you can configure, this is the most used list:<br/>
```OE_USER``` will be the username for the system user.<br/>
```INSTALL_WKHTMLTOPDF``` set to ```False``` if you do not want to install Wkhtmltopdf, if you want to install it you should set it to ```True```.<br/>
```OE_PORT``` is the port where Odoo should run on, for example 8069.<br/>
```OE_VERSION``` is the Odoo version to install, for example ```11.0``` for Odoo V11.<br/>
```IS_ENTERPRISE``` will install the Enterprise version on top of ```11.0``` if you set it to ```True```, set it to ```False``` if you want the community version of Odoo 11.<br/>
```OE_SUPERADMIN``` is the master password for this Odoo installation.<br/>

#### 3. Make the script executable
```
sudo chmod +x odoo_install.sh
```
##### 4. Execute the script:
```
sudo ./odoo_install.sh
```

##### 5. Upgrade From Community to Enterprise edition:
Download the deb file from the odoo partner space
Backup your community database
Stop the odoo service
```
 sudo service odoo stop
```
Install the enterprise .deb (it should install over the community package)
```
 sudo dpkg -i <path_to_enterprise_deb>
```

Update your database to the enterprise packages using
```
python3 /usr/bin/odoo-bin -d <database_name> -i web_enterprise --stop-after-init
```
You should be able to connect to your Odoo Enterprise instance using your usual mean of identification. You can then link your database with your Odoo Enterprise Subscription by entering the code you received by e-mail in the form input

## Setup NGINX as HTTPS Proxy for Odoo ##

### Install NGINX ###


    sudo apt-get install nginx -y

### create a new configuration file for the odoo site ###

    sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/odoo.conf
    sudo nano /etc/nginx/sites-available/odoo.conf
    


paste the following code and replace `server_name odoo11.your.domain` by our own name:  

    upstream odoo {
    		server 127.0.0.1:8069;
    }
    
    upstream odoochat {
    		server 127.0.0.1:8072;
    }
    
    server {
    		listen 80;
    		server_name odoo11.your.domain;
    
    		proxy_read_timeout 720s;
    		proxy_connect_timeout 720s;
    		proxy_send_timeout 720s;
    
    		# add headers for odoo proxy mode
    		proxy_set_header X-Forwarded-Host $host;
    		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    		proxy_set_header X-Forwarded-Proto $scheme;
    		proxy_set_header X-Real-IP $remote_addr;
    
    		# log
    		access_log /var/log/nginx/odoo.access.log;
    		error_log /var/log/nginx/odoo.error.log;
    
    		# redirect requests to odoo backend server
    		location /{
    				proxy_redirect off;
    				proxy_pass http://odoo;
    		}
    		location /longpolling {
    				proxy_pass http://odoochat;
    		}
    
    		# common gzip
    		gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
    		gzip on;
    
    }

### Enable the new site and remove the default site ###

    sudo ln -s /etc/nginx/sites-available/odoo.conf /etc/nginx/sites-enabled/odoo.conf
    sudo rm /etc/nginx/sites-enabled/default
    
    sudo nginx -t
    sudo nginx -s reload

### Enable Odoo's Proxy Mode ###

in /etc/odoo-server.conf add the line: `proxy_mode = True` 

then restart odoo server: `sudo service odoo-server restart` 

### Test the HTTP Connection via Proxy ###

in your Browser connect to the Odoo Url
http://odoo11.your.domain

=> now we have a working Odoo instance proxied via NGINX on HTTP Port 80

### Install and configure Letsencrypt ACME client for NGINX ###

We use the certbot NGINX Plugin (https://github.com/certbot/certbot) to request and auotmatically renew the SSL Certificate for our odoo url

install certbot from its official Ubuntu PPA:

    sudo add-apt-repository ppa:certbot/certbot
    sudo apt-get update
    sudo apt-get install python-certbot-nginx
    
    sudo certbot --nginx -d odoo11.your.domain
    -> when asked enter your email address
    -> when asked press A to agree the terms of service
    -> when asked to share your email address enter (Y)es/(N)o:
    -> when asked to choose whether or not to redirect HTTP traffic to HTTPS
    	answer 2 to add redirection
    


### Test the HTTPS Connection via Proxy ###

in your Browser connect to the Odoo Url
http://odoo11.your.domain
-> should redirect to https://odoo11.your.domain
=> now we have a working Odoo instance proxied via NGINX on HTTPS Port 443 with a Public SSL Certificate from Letsencrypt


### configure Letsencrypt Certificate Autorenewal ###

    sudo crontab -e
    
    	0 3 * * * /usr/bin/certbot renew --quiet
    
This runs the command every day at 03:00. The command will check to see if the certificate on the server is expired, and renew it if it is.

We now have a working Odoo 11 instance, available via HTTPS.

### Optional: install ssmtp and configure it for sending alerts ###

    sudo apt-get install ssmtp
    
    sudo nano /etc/ssmtp/ssmtp.conf
    	root=alerts@your.domain
    	mailhub=smtp.office365.com:587
    	AuthUser=alerts@your.domain
    	AuthPass=VerySecret
    	UseTLS=YES
    	UseSTARTTLS=YES
    	rewriteDomain=mexnet.ch
    	hostname=odoo11.your.domain
    	FromLineOverride=YES
    
    
    sudo nano /etc/ssmtp/revaliases
    	root:alerts@your.domain:smtp.office365.com:587
    	ubuntu:alerts@your.domain:smtp.office365.com:587
    

simple Test:

    echo "Test message from Linux server using ssmtp" | sudo ssmtp -vvv joerg.peter@your.domain

advanced Test:

	sudo ssmtp joerg.peter@your.domain
		To: joerg.peter@your.domain
		From: alerts@your.domain
		Subject: Info from odoo

		Hello Joerg

		<CTRL><D>


## Resources ##

https://www.odoo.yenthevg.com/installing-odoo-11-enterprise-ubuntu/
https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/
https://github.com/certbot/certbot
https://help.ubuntu.com/community/EmailAlerts

make https connections more secure (tbd):
https://arashmilani.com/post?id=95
https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

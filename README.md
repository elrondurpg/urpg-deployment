# URPG Deployment with Ansible

The purpose of this repository is to provide a single, easy-to-understand and easy-to-use method for deploying URPG's technology resources. 
The primary function of this repository is provided in the Ansible playbook `plays/deploy.yml`. 

This playbook installs all resources on your local host.

Please read on to learn what it does and how to use it. 

## Prerequisites

This deployment playbook makes the following assumptions: 
- You are running it on a Linux server using CentOS 7 architecture. 
- You have permissions to use the 'sudo' command. 
- You have permission to install your own Apache HTTPD webserver, or you have access to edit the httpd.conf file of an existing one. 
- You have a domain name. 

## Description

This deployment playbook runs eight installation steps:
- URPG DB
- URPG Server 
- URPG HTTPD
- URPG Let's Encrypt
- URPG Middleware
- URPG Webapps
- URPG Wordpress
- URPG Wordpress Theme

### URPG DB

The urpg-db role installs a MariaDB (MySQL) database on your host. It performs basic security and ease-of-use configurations on the database. Additionally, this role creates the users and databases that will be used by the URPG Server application and the Wordpress website. 

### URPG Server

The urpg-server role installs the Java-based URPG server application, which provides the APIs that other applications use to access and modify information in the database. By default, it operates as follows:
- The urpg-server JAR file will be downloaded to /opt/urpg-server
- The application's logs will be output to /var/log/urpg-server
- The urpg-server application will be configured as a systemctl service which starts automatically when the host reboots.
- The urpg-server application will run on port 8080 and will not use HTTPS security. 

The deployment playbook **will** later configure URPG Server to use port 8443 over secure HTTPS. If the URPG Server role is run independently, you will not have security features installed by default. 

### URPG HTTPD

The urpg-httpd role installs an Apache HTTPD web server and updates its httpd.conf configuration file. 

Initially, the HTTPD application will be installed and started without HTTPS support. This is necessary in order to support the challenge mechanism that will eventually allow the playbook to create a self-signed SSL certificate and **then** enable HTTPS support on your HTTPD web server.

Please make note of the following important pieces of functionality in the httpd.conf file:
- A VirtualHost directive that redirects all HTTP requests to HTTPS will be installed after HTTPS is enabled. 
- A number of server environment variables are installed for use by the URPG Middleware libraries. 
- Any requests made to paths that don't start with /info or /urpg-webapps will be redirected to the /info homepage. 
- In general, requests made to paths beginning with /urpg-webapps will be routed by the URPG Webapps application.
- In general, requests made to paths beginning with /info will be routed by the Wordpress application. 

### URPG Let's Encrypt

The urpg-letsencrypt role creates a self-signed SSL certificate and installs it on the URPG Server application and the HTTPD web server, then updates their configuration to support HTTPS requests. The certificate will be created with a 90-day expiration window. You can renew it using the deployment playbook after 80 days. 

### URPG Middleware

The urpg-middleware role installs a set of PHP files that are used by URPG Webapps to log into and send authenticated requests to the URPG Server APIs. It installs PHP version 8.2 and the PHP modules that are required by the middleware files. 

### URPG Wordpress

The urpg-wordpress role installs a standard Wordpress application, including required PHP modules, and configures it for use by URPG Infohub.

Here is a summary of the custom configurations: 
- We change the permalink structure of our pages to use the post name. 
- We make Wordpress pages accessible at /info. Thanks to our redirect rules, these pages are also accessible at /. Therefore, both /info/wp-login.php and /wp-login.php will take you to the Wordpress login page. 

### URPG Wordpress Theme

The urpg-wordpress-theme role installs the URPG Wordpress theme, makes it active, and injects information that allows the theme to use the site header defined in the URPG Webapps application. 

Namely, this role updates the theme's header.php file to insert scripts derived from the URPG Webapps directory. Additionally, it copies one of those scripts and makes a few changes to allow Wordpress independent access to certain Angular-based components. 

Among other functionality, this theme turns off the Wordpress toolbar for Wordpress users by default. 

## Usage

1. Log into your host and run the following commands: 
```
	sudo yum update -y
	python3 -m pip install --user ansible-core>=2.2
	sudo pip3 install pymysql
		[ Note: Need to confirm whether I need 'sudo', or whether it needs to be run once with AND once without ]
		[ Note: I am also curious whether this can be done through Ansible ]
	sudo pip3 install jmespath
		[ Note: Need to confirm whether I need 'sudo', or whether it needs to be run once with AND once without ]
		[ Note: I am also curious whether this can be done through Ansible ]
	sudo yum install git -y
	git clone https://github.com/elrondurpg/urpg-deployment.git
```	
2. Create a file with the following contents. Insert your own values and note the file name for use in the next step:
```
	mysql_root_password: 
	urpg_db_password: 
	artifact_download_token: 
	urpg_server_client_id: 
	urpg_server_client_secret: 
	urpg_server_redirect_uri: 
	keystore_password: 
	wp_admin_password: 
```
3. Run the following command to encrypt the Ansible vault you created in the previous step: `ansible-vault encrypt <vault_filename>`
4. Open the file `plays/host_vars/localhost.yml` and update the value of the `domain_name` variable to your domain name. 
5. \*Run the following command to deploy all URPG resources using the urpg-deployment playbook: `cd ~/urpg-deployment; ansible-playbook plays/deploy.yml -e @<vault_filename> --ask-vault-pass`
6. On the live URPG host, run the following command to obtain a copy of the contents of the URPG DB: `sudo mysqldump --add-drop-table -u root -p URPG_DB > urpg_db.sql`
7. Place the .sql file in an accessible location on your new host, e.g. `~/sql/urpg-db.sql`
8. Navigate to the directory where you placed the .sql file containing the contents of the URPG DB, e.g. `cd ~/sql`
9. Log into MySQL, e.g. `mysql -u root -p`
10. Run the following commands in the MySQL shell to upload the DB contents stored in the .sql file to the DB itself:
```
	use URPG_DB
	source urpg_db.sql
```	
11. Obtain a copy of the contents of the live Wordpress site.
	a. On the live URPG host, log into Wordpress and navigate to the following page: https://pokemonurpg.com/wp/wp-admin/export.php
	b. Click **Download Export File**
12. Import the contents of the Wordpress site to the new host.
	a. On the new URPG host, log into Wordpress and navigate to the following page: https://<your domain here>/wp-admin/import.php
	b. Click **Install Now** under **Wordpress**
	c. Click **Run Importer**
	d. Click **Choose file** and browse to the file you exported from the live site
	e. Click **Upload file and import**
	f. Click **Submit**
	g. Go to **Posts** and delete any dummy posts that were included with the Wordpress installation.

\* You can add tags to the end of the ansible-playbook command in order to run one or more roles in isolation. See `plays/deploy.yml` for a list of tags that correspond to each role.
# Steps

This guide is written for Linux, specifically for a compute instance running Oracle Linux 7.7 image. Navigate to your home directory.

This guide is heavily based off of https://docs.oracle.com/en/cloud/paas/atp-cloud/atpdg/php-application.html#GUID-F6EC7814-5C62-4BA9-AE9C-F5DC21AD5FF1.

This guide is, admittedly, a little vague at some places in that it does not tell you what your current working directory is. Unless otherwise specified, assume that the directory is on a compute instance and in either /home/opc or /home/opc/app. If the command to run is one that installs stuff (e.g. "yum install packagename otherpackagename") then the directory does not matter.

## Installing PHP
- Run the following commands:
	- `sudo yum install -y oracle-release-el7 oracle-php-release-el7`
	- `sudo yum install -y php php-devel php-xml dtrace-utils`
	- `wget http://pear.php.net/go-pear.phar`
	- `sudo php go-pear.phar` (press enter for all prompts)

## Installing Oracle instant client:
- Run the following command:
	- `sudo yum -y install oracle-instantclient19.3-basic oracle-instantclient19.3-devel`

## Install PHP OCI8
- Run the following commands:
	- `sudo PHP_DTRACE=yes pecl install oci8` (press enter when prompted)
	- `sudo sh -c "echo extension=oci8.so > /etc/php.d/20-oci8.ini"`
	- `sudo sh -c "echo oci8.events = On > /etc/php.d/20-oci8.ini"`

## Installing Client Credentials for ATP Database
- Create an ATP instance
	- Link: https://oracle.github.io/learning-library/workshops/autonomous-transaction-processing/?page=LabGuide100ProvisionAnATPDatabase.md
	- Remember the following information:
		- ADMIN password
- (optional) Create a user
	- Download SQL developer if you don't have it already
	- Configure a connection using the Wallet zip file and with the admin credentials; save the password, test the connection, and if the connection gives you the "200 OK" status then click Connect
	- Run the following code:
	```
	create user demo_user identified by "HelloWorld12345#";
	grant dwrole to demo_user;
	```
		- Username: "demo_user"
		- Password: "HelloWorld12345#"
- Download the wallet
	- Click into your ATP instance, and then click on the Service Console button
	- On the left sidebar menu click "Administration"
	- Select "Download Client Credentials (Wallet)" and create a password for the wallet
		- You will not need the wallet password for this app
	- If you are working on a compute instance and you downloaded the wallet to your downloads folder, you might need to copy it to the compute instance via this command:
		- `scp ~/Downloads/Wallet_nameofyourATPinstance.zip opc@123.456.789.012:~` (replace the wallet name and the ip address)
- Process the wallet
	- Make a folder called "app": `mkdir app`
	- Move the wallet into the app folder: `mv Wallet_nameofyourATPinstance.zip app`
	- Navigate into the app folder: `cd app`
	- Unzip the wallet file: `unzip Wallet_nameofyourATPinstance.zip`
		- It should contain these files: `cwallet.sso` `tnsnames.ora` `truststore.jks` `ojdbc.properties` `sqlnet.ora` `ewallet.p12` `keystore.jks`
	- Get the current directory: `pwd`
		- It should return something like "/home/opc/app" on the compute instance
	- Edit the file sqlnet.ora:
		- Open the file: `vim sqlnet.ora`
		- Press capital-"A" to enter INSERT mode and navigate cursor to the end of the line
		- Use arrow keys to navigate left to the DIRECTORY variable
		- Replace the directory with the one returned above. For instance, you would replace `?/network/admin` with `/home/opc/app`
			- For Windows/Linux users, "CTRL+V" will not paste in a bash terminal. You might have to type out the URL (safe) or right-click and paste (this might not appear for all users). For Mac users you can safely paste with "COMMAND+V"
		- Press the escape key to exit INSERT mode
		- Type ":wq" to execute the vim commands to "write" and "quit" (save and quit)
	- Set the global variable TNS_ADMIN to the directory where you put your wallet files: for instance `export TNS_ADMIN=/home/opc/app`
		- For instance: `export TNS_ADMIN=/home/opc/app`
		- To ensure that TNS_ADMIN is always exported whenever you start a new bash session, you will need to edit the .bash_profile file. Run `echo "export TNS_ADMIN=/home/opc/app" >> ~/.bash_profile` so that the command will be executed whenever you start a new bash session.
- Get the service name
	- Paste the contents of the "tnsnames.ora" file to stdout (the terminal's output): `cat tnsnames.ora`
	- You can see a variety of different service names, for instance "atpDatabaseName_high". Copy the name of the "high" one for now; you will need it in a future step.

## Install the PHP CLI
- Run the following command:
	- `sudo yum install php-cli`
		- It is highly likely that your compute instance will already have the php-cli package installed
- Link the OCI8 libraries to the PHP command
	- (optional) Run ``setcurr(){ currdir=`pwd` ; }; getcurr(){ echo $currdir ; }; curr(){ cd $currdir ; }`` to set commands that allow you to navigate back to a saved directory. Then run `setcurr` so that you can run `curr` later to come back to this folder.
		- If you want to see where `curr` will take you, run `getcurr`
	- Run the following commands:
		- `cd /etc/php.d`
		- `sudo su`
		- `echo "extension=oci8.so" > oci8.ini`
		- `exit`
		- (optional) `curr`
	- You can test to see if you have to proper libraries via `php -i | grep oci8`. Successful linking means you should see something similar to the following:
	```
	/etc/php.d/oci8.ini,

	oci8
	oci8.connection_class => no value => no value
	oci8.default_prefetch => 100 => 100
	oci8.events => Off => Off
	oci8.max_persistent => -1 => -1
	oci8.old_oci_close_semantics => Off => Off
	oci8.persistent_timeout => -1 => -1
	oci8.ping_interval => 60 => 60
	oci8.privileged_connect => Off => Off
	oci8.statement_cache_size => 20 => 20
	```

## Testing
- Download the example application for testing called "Example.php" via https://github.com/OracleMichael/PHP-ATP/Example.php
	- Via command line: `curl https://raw.githubusercontent.com/OracleMichael/PHP-ATP/master/Example.php > Example.php`
- Modify the file:
	- Open the file: `vim Example.php`
	- (optional) Show the line numbers: `:set number`
	- Navigate directly to line 28: `:28`
	- Press "i" to enter INSERT mode at the current location
	- Replace the username, password, and service name with their corresponding information.
	- Press the escape key to exit INSERT mode
	- Type ":wq" to save and quit
- Run the file
	- Execute `php -f Example.php`
	- Here is an example of proper output:
	```
	PHP Warning:  oci_execute(): ORA-01950: no privileges on tablespace 'DATA' in /home/opc/app/Example.php on line 75
	PHP Warning:  oci_execute(): ORA-01950: no privileges on tablespace 'DATA' in /home/opc/app/Example.php on line 75
	PHP Warning:  oci_execute(): ORA-01950: no privileges on tablespace 'DATA' in /home/opc/app/Example.php on line 75
	<table border='1'>
	<tr>
	<th><b>ID</b></th>
	<th><b>DATA</b></th>
	</tr>
	</table>
	```
TODO fix the warnings

## That's it, you're done!

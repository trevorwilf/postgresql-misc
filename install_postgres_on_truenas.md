
This is a tutorial for installing a PostgreSQL server in a jail on TrueNAS.  My original motivation was to give myself a SQL-capable back-end that I could use for my data science lessons.  I now use it for that as well as to maintain various databases I need (like personal finances, etc.)  

As of this writing, I'm working with the following versions of the software:

* TrueNAS:		TrueNAS-13.0-U6.1  (latest stable version)
* TrueNAS Jail:		13.3-Release  (latest stable version)
* PostgreSQL:  	16.3     (latest non-beta FreeBSD version available in the repository)
	
#### Prequisites
* A working TrueNAS installation, connected to the Internet, with some spare disk space available

#### Follow these 8 steps to install PostgreSQL in an iocage jail:

1. Carve out some space to hold your databases - create a new dataset dedicated to that purpose.

	* Go to **Storage / Pools**
	* Select your pool, click on the 3 dots and select [Add Dataset]
		* Name:  (whatever - I use postgresql)
		* Comment:  (whatever)
		* Keep all remaining defaults and click **Save**

2. Set up a new jail for running the PostgreSQL server:

	- Go to **Jails**
		- (If you don't have any jails yet, select a pool for jail storage)
	- Click on **Add** and use the sequence of Basic pages
		- Name:  (whatever - I use postgresql)
		- Jail type:  (I use basejail - see [here](https://www.truenas.com/community/threads/iocage-jail-type-base-jail-vs-clone-which-to-choose.82639/) for great discussion of jail types)
		- Release:  might as well use the latest (13.3-RELEASE as of this writing)
		- Networking: (whatever you like; I use DHCP Autoconfigure IPv4, VNET, and Berkeley PacketFilter, and then my router hands the jail a fixed lease.)
		- Check Auto-start so the jail will always  be running
		- Keep all remaining defaults and click **Save**

3. Set up a mount point so the jail from step 2 can access the dataset in step 1.

	- Select the jail you just created (if the state is not up, Select **START** to turn it on)
 	- Select **> SHELL**
 	- Create the directory you want to mount the data drive too. (I use /mnt/postgresql) **mkdir /mnt/postgresql**
 	- Go back to the jails and turn off the jail by selecting **STOP**
  	- click the pull down at the far right, click on **Mount Points**
 	- Select **ACTIONS** and then select **ADD**
	- For Source, choose the dataset you created in step 1.
	- For Destination, choose something within the jail (I use /mnt/postgresql)
	- Click **Save**

4. Install PostgreSQL in the jail, using the `pkg` command in the shell.

	* Go back to the jails and turn on the jail by selecting **START**
 	* Click on the jail, select the pulldown, and click on **Shell**
	* At the command prompt, type `pkg` and hit enter
		* If not yet installed, answer y to install pkg
	* Install the following packages:
	
		`pkg install postgresql16-server`
		
		`pkg install postgresql16-contrib`
		
		`pkg install postgresql16-docs`
	
	Note:  A search on the [ports page](https://www.freebsd.org/cgi/ports.cgi) for something like 'postgresql16' will show you what you're installing.
	Note2:  All Databases [Databases page](https://cgit.freebsd.org/ports/tree/databases]https://cgit.freebsd.org/ports/tree/databases)

	While you're at it, install a basic editor for some editing that you'll be doing shortly.

	`pkg install vim`
	`pkg install vim-colorschemes-legacy`

Or use nano:

	`pkg install nano`

6. While still at the jail shell prompt, adjust some settings, set up proper ownership, and initialize and run postgresql

	* Make sure postgresql starts up when jail is launched; at the prompt type
	
		`sysrc postgresql_enable=YES`

		(Note that this just adds a line **postgresql_enable="YES"** to the end of the /etc/rc.conf file.)		

	* Tell postgres the location to hold databases by referring to the mount the point you set up in step 3

		`sysrc postgresql_data=/mnt/postgresql`
		
		(Similarly, this adds a line **postgresql_data="/mnt/postgresql"** to /etc/rc.conf)

	* Change user and group ownership of that location to the **postgres** user and group:

		`chown -R postgres:postgres /mnt/postgresql`
		
	(Note that the **postgres** user and group were created when you installed postgres.)


6) Initialize postgres and ensure that you can access it.

	* Initialize postgres

		`service postgresql initdb`

	* Make modifications to permit access of the server from other hosts. By default, all connections will be refused. To change that, we need to edit **postgresql.conf** and **pg_hba.conf**, files that were produced from the initdb.

		`vim /mnt/postgresql/postgresql.conf`
		
		Where  you see **listen_addresses**, add this line if it's okay for any host to try to connect:

		`listen_addresses = '*'`

		Alternatively, if you want to lock it down more where you see **listen_addresses**, change it to a comma seperated list.  For example:

		`listen_addresses = '127.0.0.1, 10.10.1.23, 10.10.1.24'`

		Save and exit (e.g., with vim: esc then : (to enter the submenu).  Then rq<enter>.

		Then edit pg_hba.conf:

		`vim /mnt/postgresql/pg_hba.conf`

		To the end of the file, add a line to specify how hosts on your LAN can access postgresql.  For example, if your LAN is 192.168.1.0/24 and you don't require any special security, you would add a line like
		
		`host all all 192.168.1.0/24 trust`

		Save and exit.

	* Now start up postgresql and make sure it's running

		`service postgresql start`
		
		`service postgresql status`

7. Create a database and a user so you can test access to the system
			
	`su postgres`  (take over as the postgres superuser; prompt will change to $)

	`createuser --interactive <nameofuser>`

	Provide a name (I use **stu**) and make it a superuser.  Then create a new database 'new_db':

	`createdb new_db`

	Launch an interactive SQL session to configure new user permissions
	
	`psql`  (prompt will change to postgres=#)

	`ALTER USER stu WITH ENCRYPTED PASSWORD 'INSERT_YOUR_PASSWORD_HERE';`  (need the quotes around the password)

	`GRANT ALL PRIVILEGES ON DATABASE new_db TO stu;`

	Exit psql and your postgres shell
	
	`\q`
	
	`exit`

	Restart the service
	
	`service postgresql restart`

8. Connect to the server using your favorite tool (e.g. pgAdmin or DBeaver), using the jail's IP address and the default port of 5432. Check to make sure you can access the new_db database.


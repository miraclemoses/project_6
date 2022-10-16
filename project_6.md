Awesome Documentation of project_6
Web Solution with WordPress
WordPress is a free and open-source content management system written in PHP and paired with MySQL or MariaDB as its backend Relational Database Management System (RDBMS). WordPress software can be used to create beautiful websites, blogs, or apps with beautiful designs, powerful features, and the freedom to build anything you want. 
WordPress is both free and priceless at the same time. 40% of the web uses WordPress, from hobby blogs to the biggest news sites online. WordPress can be extended with over 55,000 plugins to help your website meet your needs.
This project demonstrates how to prepare storage infrastructure on two Linux servers and implement a basic web solution using WordPress.


Presentation Layer (PL): This is the user interface such as the client server or browser on your laptop.
Business Layer (BL): This is the backend program that implements business logic. Application or Web Server
Data Access or Management Layer (DAL): This is the layer for computer data storage and data access. Database Server or File System Server such as FTP server, or NFS Server
This project consists of two parts:
Configuring storage subsystem for Web and Database servers based on Linux OS. 
Installing WordPress and connect it to a remote MySQL database server. 
Two CentOS servers from Microsoft Azure will be used for this project, with each server having three additional volumes attached.

Step 1: Prepare a Web Server
Create a CentOS VM that will serve as a Web Server named web-server. Create 3 volumes (disks) for the VM of the same size.

SSH to the VM after creation (using Putty in a Windows OS).
Use lsblk command to inspect what block devices (disks) are attached to the server. Notice names of the newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure all 3 newly created block devices are seen there

 

Use df -h command to see all mounts and free space on the server


Use gdisk utility to create a single GPT partition on each of the 3 disks.
To create a partition on the first disk i.e. /dev/sdc, run the command and follow the prompt:
$ sudo gdisk /dev/xvdf
$ sudo gdisk /dev/xvdg
$ sudo gdisk /dev/xvdh

Use lsblk to view the newly configured partition on /dev/xvd



Create a partition each on the other two disks i.e. /dev/xvd and /dev/xvd
Use lsblk to view the newly configured partition on the 3 disks


Run sudo lvmdiskscan command to check for available partitions:

From the partition created for each disk, create a Physical Volume (PV).
Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by the LVM (Logical Volume):
$ sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
Run sudo pvdisplay to view the info about the created PVs. 
Run sudo pvs to view the PVs summary.


Physical volumes are combined into volume groups (VGs). It creates a pool of disk space out of which logical volumes can be allocated.
Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg
$ sudo vgcreate webdata-vg /dev/sdc1 /dev/sdd1 /dev/sde1


Run sudo vgdisplay to view the info about the created VG. 

Run sudo vgs to view the VG summary.
Run sudo vgdisplay –v to view all info about a VG including the PVs and LVs of a VG.

Use lvcreate utility to create 2 logical volumes: apps-lv (Use half of the PV size), and logs-lv (use the remaining space of the PV size). apps-lv will be used to store data for the website while logs-lv will be used to store data for logs.
$ sudo lvcreate -n apps-lv -L 24G  webdata-vg
$ sudo lvcreate -n logs-lv -L 20G webdata-vg

Run sudo lvdisplay to view the info about the created LVs. 
Run sudo lvs to view the LVs summary.

Verify the entire setup
$ sudo vgdisplay -v (view complete setup - VG, PV, and LV)
$ sudo lsblk
On running sudo lsblk, it now shows that the LVM apps-lv took spaces from two PV to create its 14gb size while the LVM logs-lv also took spaces from two PV to create its 10gb size.

Use mkfs.ext4 (an ext4 file system) to format the logical volumes with ext4 filesystem
$ sudo mkfs.ext4 /dev/webdata-vg/apps-lv
Or
$ sudo mkfs -t ext4 /dev/webdata-vg/apps-lv


$ sudo mkfs.ext4 /dev/webdata-vg/logs-lv
Or
$ sudo mkfs -t ext4 /dev/webdata-vg/logs-lv


Create /var/www/html directory to store website files
Create /home/recovery/logs to store backup of log data

Mount /var/www/html on apps-lv logical volume
$ sudo mount /dev/webdata-vg/apps-lv /var/www/html
Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)
$ sudo rsync -av --delete /var/log /home/recovery/logs

The code above will synchronize the contents of /var/log to /home/recovery/logs, and leave no differences between the two. If rsync finds that /home/recovery/logs has a file that /var/log does not, it will delete it. If rsync finds a file that has been changed, created, or deleted in /var/log, it will reflect those same changes to /home/recovery/logs.
Here is what the aforementioned code tells rsync to do with the backups:
1. -a = Recursive (recurse into directories), links (copy symlinks as symlinks), perms (preserve permissions), times (preserve modification times), group (preserve group), owner (preserve owner), preserve device files, and preserve special files.
2. -v = Verbose, to see exactly what rsync is backing up.
3. --delete = This tells rsync to delete any files that are in /home/recovery/logs that aren’t in /var/log. This is optional in this context.
Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. This is why the rsync step above is very important).
$ sudo mount /dev/webdata-vg/logs-lv /var/log
Restore log files back into /var/log directory
$ sudo rsync -av /home/recovery/logs/ /var/log
Run df –h to confirm that the logical volumes were mounted on their correct paths

Note that if the server is restarted, the mount configurations will be lost.
Update /etc/fstab file so that the mount configurations will persist after restart of the server.
Add this two lines at the end of the contents of the /etc/fstab with any editor of choice
$ sudo vi /etc/fstab
/dev/mapper/webdata--vg-apps--lv    /var/www/html    ext4    defaults    0 0
/dev/mapper/webdata--vg-logs--lv    /var/log    ext4    defaults    0 0

Restart the server and run df – h  to confirm if the mount configurations persisted.


Step 2: Prepare the Database Server
Create a second CentOS instance that will serve as a Database Server named database-server. Repeat the same steps as for the Web Server, but instead of app-lv create db-lv and mount it to /db directory instead of /var/www/html/.
Below is for the database-server VM’s /etc/fstab file to enable persistence of the mount configurations after restarting the server:
/dev/mapper/dbdata--vg-db--lv    /db    ext4    defaults    0 0
/dev/mapper/dbdata--vg-logs--lv    /var/log    ext4    defaults    0 0


Step 3: Install WordPress on the Web Server
Install WordPress and other necessary packages
$ sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
$ cd /var/www/html
$ sudo wget http://wordpress.org/latest.tar.gz
$ sudo tar xzvf latest.tar.gz
$ cp -r wordpress /var/www/html
$ sudo systemctl restart httpd
$ sudo systemctl enable httpd

Confirm that the Apache server is active:
$ sudo systemctl status httpd


Step 4: Install MySQL on the Database Server
$ sudo yum update
$ sudo yum install mysql-server
Verify that the service is up and running by using sudo systemctl status mysqld. If it is not running, restart the service and enable it so it will be running even after reboot:
$ sudo systemctl restart mysqld
$ sudo systemctl enable mysqld




Step 5 — Configure Database to work with WordPress
NOTE: Kindly ensure to type in the quotation marks in the MySQL cli commands, instead of copying and pasting. Sometimes the quotation marks copied are different.
$ sudo mysql
mysql>  CREATE DATABASE wordpress;
mysql>  CREATE USER 'miracle'@'%' IDENTIFIED WITH mysql_native_password BY 'Password.1#';
mysql>  GRANT ALL ON wordpress.* TO 'miracle'@'%';
mysql>  FLUSH PRIVILEGES;
mysql>  SHOW DATABASES;

mysql>  Exit
Test if the new user has the proper permissions by logging in to the MySQL console again, this time using the custom user credentials:
$ mysql -u miracle -p
Notice the -p flag in this command, which will prompt for the password used when creating the ernesto user. After logging in to the MySQL console, confirm access to the example_database database:
mysql> SHOW DATABASES;
This will give the output:

Exit the MySQL shell with:
mysql> exit


Step 6: Configure WordPress to connect to Database server
MySQL server uses TCP port 3306 by default. Add an inbound security rule to the Network Security Group of the ‘database-server’ server to open inbound connection through port 3306. For extra security, do not allow all IP addresses to reach the ‘database-server’ server. Allow access only to the IP address of the ‘web-server’ server.

Install MySQL client in the web-server VM and test the connection from the Web server to the Database server by using mysql-client.
$ sudo yum install mysql
$ mysql -u miracle -p -h <Database-Server-IP-address>
Verify if SHOW DATABASES; command can successfully execute and see a list of existing databases.
Exit the database and change permissions and configuration so Apache could use WordPress (VERY IMPORTANT):
$ sudo chown -R apache:apache /var/www/html/wordpress
$ sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
$ sudo setsebool -P httpd_can_network_connect=1

Enable TCP port 80 in Inbound Rules configuration for the Web Server VM (enable from everywhere 0.0.0.0/0)
Try to access from a browser the link to the WordPress website
http://<Web-Server-Public-IP-Address>/wordpress/



Fill out the Database credentials:


If this message is seen, it means the WordPress has successfully connected to the remote MySQL database.

Proceed with the installation of WordPress


Conclusion
In non-geek speak, WordPress is the easiest and most powerful blogging and website builder in existence today.
WordPress is an excellent website platform for a variety of websites. From blogging to e-commerce to business and portfolio websites, WordPress is a versatile CMS. Designed with usability and flexibility in mind, WordPress is a great solution for both large and small websites.


Credits
Darey.io
https://wordpress.org/
https://ithemes.com/tutorials/what-is-wordpress/
https://linuxconfig.org/how-to-list-create-delete-partitions-on-mbr-and-gpt-disks-rhcsa-objective-preparation
https://linoxide.com/linux-how-to/lvm-configuration-linux/
https://www.linuxquestions.org/questions/linux-newbie-8/pvcreate-device-dev-sda-excluded-by-a-filter-4175657557/
https://www.thegeekdiary.com/centos-rhel-how-to-find-logical-volumes-lvs-that-are-part-of-a-physical-volume-pv-in-lvm/
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_administration/s1-lvmsetupnfs-haaa
https://www.cyberciti.biz/faq/linux-mount-an-lvm-volume-partition-command/
https://www.howtogeek.com/135533/how-to-use-rsync-to-backup-your-data-on-linux/
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/deployment_guide/s1-filesystem-ext4-mount
https://www.howtogeek.com/444814/how-to-write-an-fstab-file-on-linux/
https://linuxconfig.org/how-fstab-works-introduction-to-the-etc-fstab-file-on-linux
https://linuxhint.com/lvm_home_directories/



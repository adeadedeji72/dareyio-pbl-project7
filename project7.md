# **DEVOPS TOOLING WEBSITE SOLUTION** #
This project implements these technologies:
1.  Infrastructure: AWS
1.  Webserver Linux: Red Hat Enterprise Linux 8
1.  Database Server: Ubuntu 20.04 + MySQL
1.  Storage Server: Red Hat Enterprise Linux 8 + NFS Server
1.  Programming Language: PHP
1.  Code Repository: GitHub

We will implement the infrastructure design illustrated below:

![image](nfs-webservers-db.png)

### **STEP 1 – PREPARE NFS SERVER** ###

Lunch a new EC2 Instance of Red Hat Linux 8 OS

Configure LVM with 3 EBS volumes

Instead of formating the disks as ext4 you will have to format them as *xfs*
Ensure there are 3 Logical Volumes. lv-opt lv-apps, and lv-logs

You should have a similar output to below when you are done 

![](lsblk-df-h.jpg)

Create mount points on */mnt* directory for the logical volumes as follow:
Mount lv-apps on */mnt/apps* – To be used by webservers
Mount lv-logs on */mnt/logs* – To be used by webserver logs
Mount lv-opt on */mnt/opt* – To be used by Jenkins server in the next project

Install NFS server, configure it to start on reboot and make sure it is u and running
~~~
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
~~~

Export the mounts for webservers’ **subnet cidr** to connect as clients. For simplicity, you will install your all three Web Servers inside the same subnet, but in production set up you would probably want to separate each tier inside its own subnet for higher level of security.

Make sure we set up permission that will allow our Web servers to read, write and execute files on NFS:
~~~
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
~~~

Configure access to NFS for clients within the same subnet (example of Subnet CIDR – 172.31.80.0/20 ):
~~~
sudo vi /etc/exports

/mnt/apps <172.31.80.0/20>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <172.31.80.0/20>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <172.31.80.0/20>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!

sudo exportfs -arv
~~~

Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
~~~
rpcinfo -p | grep nfs
~~~
![image](rpcinfo-nfs-ports.jpg)

In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049
![](nfs-incoming-rules.jpg)

### **STEP 2 — CONFIGURE THE DATABASE SERVER** ###

**Install MySQL server**
1. Create a database and name it tooling
1. Create a database user and name it webaccess
1. Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr (172.31.80.0/20)


### **STEP 3 — PREPARE THE WEB SERVERS** ###

We need to make sure that our Web Servers can serve the same content from shared storage solutions, in our case – NFS Server and MySQL database.
You already know that one DB can be accessed for reads and writes by multiple clients. For storing shared files that our Web Servers will use – we will utilize NFS and mount previously created Logical Volume lv-apps to the folder where Apache stores files to be served to the users (/var/www).

During the next steps we will do following:

During the next steps we will do following:

* Configure NFS client (this step must be done on all three servers)
* Deploy a Tooling application to our Web Servers into a shared NFS folder
* Configure the Web Servers to work with a single MySQL database


1. Launch a new EC2 instance with RHEL 8 Operating System
1. Install NFS client
~~~
sudo yum install nfs-utils nfs4-acl-tools -y
~~~
3. Mount /var/www/ and target the NFS server’s export for apps
~~~
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <172.31.92.111>:/mnt/apps /var/www
~~~
Verify that NFS was mounted successfully by running df -h. Make sure that the changes will persist on Web Server after reboot:
~~~
sudo vi /etc/fstab
~~~
add following line
~~~
<172.31.92.111>:/mnt/apps /var/www nfs defaults 0 0
~~~

5. Install Remi’s repository, Apache and PHP
~~~
sudo yum install httpd -y
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo dnf module reset php
sudo dnf module enable php:remi-7.4
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
~~~
**Repeat steps 1-5 for another 2 Web Servers.**

6. Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. If you see the same files – it means NFS is mounted correctly. You can try to create a new file touch test.txt from one server and check if the same file is accessible from other Web Servers.

7. Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. Repeat step №4 to make sure the mount point will persist after reboot.

8. Fork the tooling source code from Darey.io Github Account to your Github account. (Learn how to fork a repo here)

9. Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html

10. check permissions to your /var/www/html folder and also disable SELinux sudo setenforce 0
To make this change permanent – open following config file 
~~~
sudo vi /etc/sysconfig/selinux
~~~
and set SELINUX=disabledthen restart httpd.

11. Update the website’s configuration to connect to the database (in /var/www/html/functions.php file). Apply tooling-db.sql script to your database using this command 
~~~
mysql -h <172.31.09.97> -u webaccess -p tooling < tooling-db.sql
~~~
Run the above command from the folder where tooling-db.sql file is located 

Create in MySQL a new admin user with username: myuser and password: password:
~~~
INSERT INTO ‘users’ (‘id’, ‘username’, ‘password’, ’email’, ‘user_type’, ‘status’) VALUES
-> (1, ‘myuser’, ‘5f4dcc3b5aa765d61d8327deb882cf99’, ‘user@mail.com’, ‘admin’, ‘1’);
~~~

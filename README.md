# Project-8

# **DEVOPS TOOLING WEBSITE SOLUTION.**

In this project a solution that consists of following components will be implemented:

* Infrastructure: AWS

* Webserver Linux: Red Hat Enterprise Linux 8

* Database Server: Ubuntu 20.04 + MySQL

* Storage Server: Red Hat Enterprise Linux 8 + NFS Server

* Programming Language: PHP

* Code Repository: Github

![image](https://github.com/kalkah/Project-8/assets/95209274/112d55be-388c-4200-829d-2181d2450ee5)

 ## STEP 1 – PREPARE NFS SERVER

1. Spin up a new EC2 instance with RHEL Linux 8 Operating System.

2. Based on your LVM experience from Project 6, LVM was configured on the Server.

* create volumes for the NFS server

  ![image](https://github.com/kalkah/Project-8/assets/95209274/9f81b3bd-97d8-4c2d-a70c-532b15d159e5)

![image](https://github.com/kalkah/Project-8/assets/95209274/1442c065-d6d5-4b22-af04-2926884777f2)

* using the gdisk utility to create partitions

  `sudo gdisk /dev/nvme1n1` `sudo gdisk /dev/nvme2n1` `sudo gdisk /dev/nvme3n1`

 ![image](https://github.com/kalkah/Project-8/assets/95209274/d4453642-5d1d-4dfd-b71e-5006dfb5aec6)

 The 3 partitions is shown below:

 ![image](https://github.com/kalkah/Project-8/assets/95209274/fa1548c8-9abb-4ba6-a6d0-25432fadf903)

* logical volume management was installed using `sudo yum install lvm2` command

* Other processes as in project 7 was carried out: lvmdiscscan, pvcreate, vgcreate, lv create

  ![image](https://github.com/kalkah/Project-8/assets/95209274/c7cc35ee-4ed9-49d8-8a46-c977941a6f36)

* Instead of formating the disks as `ext4` they will be formated them as `xfs`

```
sudo mkfs -t xfs /dev/webdata-vg/apps-lv
sudo mkfs -t xfs /dev/webdata-vg/logs-lv
sudo mkfs -t xfs /dev/webdata-vg/opt-lv
```
![image](https://github.com/kalkah/Project-8/assets/95209274/4b67c32a-afa2-4d92-8548-1fd34e8a3b0e)

3. Mount points was created on /mnt directory for the logical volumes as follow:

Mount apps-lv on /mnt/apps – To be used by webservers

Mount logs-lv on /mnt/logs – To be used by webserver logs

Mount opt-lv on /mnt/opt – To be used by Jenkins server in Project 8

```
sudo mkdir /mnt/apps

sudo mkdir /mnt/logs

sudo mkdir /mnt/opt

sudo mount /dev/webdata-vg/apps-lv /mnt/apps

sudo mount /dev/webdata-vg/logs-lv /mnt/logs

sudo mount /dev/webdata-vg/opt-lv /mnt/opt
```

The /etc/fstab file was updated so that the mount configuration will persist after restart of the server. The UUID of the device will be used to update the /etc/fstab file.

`sudo blkid`

![image](https://github.com/kalkah/Project-8/assets/95209274/e5ecc484-c175-4f78-a6c8-4a0bb9942c88)

'sudo nano /etc/fstab' to edit the fstab file

![image](https://github.com/kalkah/Project-8/assets/95209274/4072965e-2613-4411-b99f-bb0da32b55c0)

`sudo mount -a` was use to test the configuration `sudo systemctl daemon-reload` command was use to reload. `df -h` command was use to verify the setup

![image](https://github.com/kalkah/Project-8/assets/95209274/057af59c-967b-40c4-b147-cf95d469f7c8)

4. Install NFS server, configure it to start on reboot and make sure it is up and running

```
sudo yum -y update

sudo yum install nfs-utils -y

sudo systemctl start nfs-server.service

sudo systemctl enable nfs-server.service

sudo systemctl status nfs-server.service
```

![image](https://github.com/kalkah/Project-8/assets/95209274/bdf71ba9-5b6e-4095-aeb0-02a4795d0698)

5. Export the mounts for webservers’ subnet cidr to connect as clients. For simplicity, I install all three Web Servers inside the same subnet, but in production set up you would probably want to separate each tier inside its own subnet for higher level of security.
   
To check your subnet cidr – open your EC2 details in AWS web console and locate ‘Networking’ tab and open a Subnet link:

  * Make sure we set up permission that will allow our Web servers to read, write and execute files on NFS:

```
sudo chown -R nobody: /mnt/apps

sudo chown -R nobody: /mnt/logs

sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps

sudo chmod -R 777 /mnt/logs

sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

![image](https://github.com/kalkah/Project-8/assets/95209274/c1abacae-a02b-4be0-af1f-80a0ae081bf9)

* Configure access to NFS for clients within the same subnet (172.31.16.0/20):

`sudo nano /etc/exports`
```
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```

![image](https://github.com/kalkah/Project-8/assets/95209274/6b3820cd-f0ed-4731-80b3-5a7a48b36420)

`sudo exportfs -arv`

![image](https://github.com/kalkah/Project-8/assets/95209274/769bbb53-1158-48f8-9a1b-a0010cd8b053)

6. Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
`rpcinfo -p | grep nfs`

![image](https://github.com/kalkah/Project-8/assets/95209274/fcd8569c-d4d7-4d99-8b13-f1a3cdee9cdb)

**Important note: In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049**

![image](https://github.com/kalkah/Project-8/assets/95209274/691c85b4-a188-4b50-a1ff-e6a8453ab81e)


## STEP 2 –  CONFIGURE THE DATABASE SERVER

1. Install MySQL server

**`sudo apt update -y`**

**`sudo apt install mysql`**

2. Create a database and name it `tooling`
   
```
sudo mysql
create database tooling;
```

3. Create a database user and name it `webaccess`

**`create user 'webaccess'@'172.31.16.0/20' identified by 'password';`**

4. Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr

**`grant all privileges on tooling.* to 'webaccess'@'172.31.16.0/20';`**
`FLUSH PRIVILEGES;`
`SHOW DATABASES;`
`exit`

![image](https://github.com/kalkah/Project-8/assets/95209274/e73159c9-ee61-44f2-a8a8-532128d6fd49)

## STEP 3 - PREPARE THE WEB SERVERS

During the next steps we will do following:

* Configure NFS client (this step will be implemented on all three servers)

* Deploy a Tooling application to our Web Servers into a shared NFS folder

* Configure the Web Servers to work with a single MySQL database


1. Launch a new EC2 instance with RHEL 8 Operating System

2. Install NFS client

`sudo yum update -y'
`sudo yum install nfs-utils nfs4-acl-tools -y`

3. Mount /var/www/ and target the NFS server’s export for apps: 'sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www'

`sudo mkdir /var/www`
`sudo mount -t nfs -o rw,nosuid 172.31.28.109:/mnt/apps /var/www`

4. Verify that NFS was mounted successfully by running `df -h`. Make sure that the changes will persist on Web Server after reboot:
sudo nano /etc/fstab

Add following line: `<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0`

`172.31.28.109:/mnt/apps /var/www nfs defaults 0 0`

`sudo systemctl daemon-reload` to update the system
webserver1
![image](https://github.com/kalkah/Project-8/assets/95209274/b3be5953-f04d-4416-897f-e90a10ae3db6)

webserver2
![image](https://github.com/kalkah/Project-8/assets/95209274/427acee8-90ed-41ef-af70-624ade2bb4b6)

webserver3
![image](https://github.com/kalkah/Project-8/assets/95209274/14135c2d-ee6d-49d6-8feb-8613e6e34e7f)

Install Remi’s repository, Apache and PHP

```
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

setsebool -P httpd_execmem 1
```

webserver1 
![image](https://github.com/kalkah/Project-8/assets/95209274/99c9bcd4-e633-47f4-9f3e-3436e3605115)

webserver2
![image](https://github.com/kalkah/Project-8/assets/95209274/c47e3b8b-1d5a-4814-b35a-5bbae486fb7a)

webserver3
![image](https://github.com/kalkah/Project-8/assets/95209274/ec132b9f-c9b4-46c2-b958-0c4a7addf000)

5. Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. If you see the same files – it means NFS is mounted correctly. You can try to create a new file touch test.txt from one server and check if the same file is accessible from other Web Servers.

test.txt files created on the NFS server as shown below:
![image](https://github.com/kalkah/Project-8/assets/95209274/83cdab3b-20b5-424d-bb1b-ec54cb0dd1cd)

The test.txt file can be seen on the web servers:
![image](https://github.com/kalkah/Project-8/assets/95209274/61b799fe-4762-474a-9821-2f50e0a9fb66)
![image](https://github.com/kalkah/Project-8/assets/95209274/0c907fd0-2427-4234-b967-31de2df1b832)
![image](https://github.com/kalkah/Project-8/assets/95209274/eb27753b-e12b-4406-b466-b8b8a3076ed6)

6. Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. Repeat step 4 under the 'prepare web servers' to make sure the mount point will persist after reboot.

`sudo mount -t nfs -o rw,nosuid 172.31.28.109:/mnt/logs /var/log/httpd`

`sudo nano /etc/fstab`
Add following line: `<NFS-Server-Private-IP-Address>:/mnt/logs /var/logs/httpd nfs defaults 0 0`

`172.31.28.109:/mnt/logs /var/log/httpd nfs defaults 0 0`

![image](https://github.com/kalkah/Project-8/assets/95209274/71e5533f-867d-4ced-8d78-8b813863a2d0)
![image](https://github.com/kalkah/Project-8/assets/95209274/d2535850-4cf0-433d-8e07-70eeb385da52)
![image](https://github.com/kalkah/Project-8/assets/95209274/da2f85ff-9c54-43d7-9f96-80875c43ddc4)

8.  Fork the tooling source code from Darey.io Github Account to your Github account

```
sudo yum install git -y
git init
sudo clone https://github.com/darey-io/tooling.git
```

![image](https://github.com/kalkah/Project-8/assets/95209274/8d7dbf0d-3a76-4810-ae14-cf68c2c17d56)

9. Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to `/var/www/html`
![image](https://github.com/kalkah/Project-8/assets/95209274/eb175618-3870-4dab-bdba-985916257df9)

Note 1: Do not forget to open TCP port 80 on the Web Server.

Note 2: If you encounter 403 Error – check permissions to your /var/www/html folder and also disable SELinux `sudo setenforce 0`

To make this change permanent – open following config file `sudo nano /etc/sysconfig/selinux` and set `SELINUX=disabled` then restart httpd.
   ![image](https://github.com/kalkah/Project-8/assets/95209274/52dedf60-922b-40d9-ab9e-377b6f40cd01)
 
`sudo systemctl restart httpd`   `sudo systemctl status httpd`

10. Update the website’s configuration to connect to the database (in /var/www/html/functions.php file). `sudo nano /var/www/html/functions.php'
    ![image](https://github.com/kalkah/Project-8/assets/95209274/222737c1-d6b0-4369-b447-b35b590d3ba4)

* Install MySQL on the web servers using **`sudo yum install mysql -y`** then cd into the tooling directory to connect to thhe database.
    Apply tooling-db.sql script to your database using this command     `mysql -h <databse-private-ip> -u <db-username> -p <db-pasword> < tooling-db.sql>`
    **`mysql -h 172.31.19.35 -u webaccess -p tooling < tooling-db.sql**
    
If you can't connect to it and there is an error simply move to the DB server to edit the inbound security group.
![image](https://github.com/kalkah/Project-8/assets/95209274/3bf25368-16ea-4ba8-8155-5f5c9f3157fc)


Then edit the mysqld.cnf file (on the DB server)

```
sudo nano /etc/mysql/mysql.conf.d//mysqld.cnf

sudo systemctl restart mysql

sudo systemctl status mysql
```
![image](https://github.com/kalkah/Project-8/assets/95209274/95172014-f93c-451b-8e31-7356790852a5)

![image](https://github.com/kalkah/Project-8/assets/95209274/99754f94-4a5e-4fa6-8611-7ae310a93547)

11. Create in MySQL a new admin user with username: myuser and password: password:

**`INSERT INTO ‘users’ (‘id’, ‘username’, ‘password’, ’email’, ‘user_type’, ‘status’) VALUES
-> (1, ‘myuser’, ‘5f4dcc3b5aa765d61d8327deb882cf99’, ‘user@mail.com’, ‘admin’, ‘1’);`**

12. Open the website in your browser http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php and make sure you can login into the website with myuser user.

webserver1
![image](https://github.com/kalkah/Project-8/assets/95209274/11e0e21c-1078-4b99-b7a0-d6701961d0dc)
![image](https://github.com/kalkah/Project-8/assets/95209274/dd527128-6021-4f5c-9021-40793bfbc759)

webserver2
![image](https://github.com/kalkah/Project-8/assets/95209274/65444a8f-8463-4a00-952d-8e25dbf96ff4)

webserver3
![image](https://github.com/kalkah/Project-8/assets/95209274/c91c0e83-0129-4867-b243-2eaa5ecf5266)

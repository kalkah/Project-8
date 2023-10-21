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

* Configure NFS client (this step must be done on all three servers)

* Deploy a Tooling application to our Web Servers into a shared NFS folder

* Configure the Web Servers to work with a single MySQL database


1. Launch a new EC2 instance with RHEL 8 Operating System

2. Install NFS client

`sudo yum install nfs-utils nfs4-acl-tools -y`

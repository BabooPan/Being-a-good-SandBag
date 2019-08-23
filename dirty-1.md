# 髒東西
![](https://i.imgur.com/lKJsEbo.png)

### 測試EC2當中的Application如何運作
- 依據需求在單機上安裝相關套件
- 寫成UserData

#### UserData
- 自動 mount efs, 安裝 redis 與 MySQL
```
#!/bin/bash
sleep 30
	    
# EFS Setting.
mkdir -p /mnt/efs
echo "<efs-id>:/ /mnt/efs efs tls,_netdev" >> /etc/fstab
mount -a -t efs defaults

# Enable EPEL Repository and install and start Redis locally.
sed -i 's/enabled=0/enabled=1/g' /etc/yum.repos.d/epel.repo
yum -y install wget redis
sleep 5
service redis start
chkconfig redis on

# Download sample applicaiton files.
wget <Application_Endpoint>
mv <Package> <Target_Path>

# Download the latest version of the applicaiton files.
wget <Application_Endpoint>
mv <Package> <Target_Path>

# Configure and deploy MySQL
yum install mysql
service mysql start
chkconfig mysql on

# Rename and execute files 

mv <Target_Path> server
chmod +x server
./server

# Reboot if the server application crashes
shutdown -h now
```
###


# 髒東西 V2

![](https://i.imgur.com/lKJsEbo.png)

## 重點提示
- 不要用ECS run application
- AutoScaling
- CloudWatch監控體系：Log, Metric, Alarm、看要不要拉個Dashboard
- Dashboard分數為評比項目之一，並不代表真正成績，會再review架構做整體評分：
    * **Security**
        * NACL
        * SG
        * WAF
        * Endpoint
        * Bocket Policy
    * **Automation**
        * 不管你是要自己寫成一包部署下去
        * 或是Github找template直接deploy之後再改設定
        * ***架構有的設定都要呈現在CloudFormation Resources那邊，才會做為評判標準***

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

# Path最好是直接指絕對路徑：/home/ec2-user

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

### CloudFormation部署
- VPC先建，再弄ASG，接著再Deploy其他的
- 記得改裡面的參數為貼近實際狀況
- 其他的就從Sample裡面Deploy出來，再改設定符合情境
- 飯粒們  
    - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html
    - https://github.com/awslabs/aws-cloudformation-templates
    - https://github.com/widdix/aws-cf-templates
- Container
    - [pahud/ecs-cfn-refarch](https://github.com/pahud/ecs-cfn-refarch)
    - [aws-samples/amazon-eks-refarch-cloudformation](https://github.com/aws-samples/amazon-eks-refarch-cloudformation)

### ALB
- Listener在HTTP (TCP/80)
- Routing Rule導兩個Target Group
    - / → Root
    - /look (path) → Look

### ElastiCache
- Memcached engine
- 配置檔
    ```
    MemcacheHost" = "ELASTICACHE_HOSTNAME"
    "MemcachePort" = "ELASTICACHE_SERVICE_PORT"
    ```

### RDS
- PostgreSQL engine
    - Multi-AZ
- 倒SQL檔進去DB
    ```
    # 第一種
    psql -h hostname -d databasename -U username -f file.sql
    # 第二種
    pg_restore -h hostname -d databasename -U username filename.sql
    ```
- 倒完之後檢查裡面有沒有兩個Table
    - 一個管理用
    - 一個存資料用

- 配置檔
    ```
    "PgsqlHost" = "RDS_HOSTNAME"
    "PgsqlPort" = "RDS_SERVICE_PORT"
    "PgsqlUser" = "db-admin" #不要用Root
    "PgsqlPass" = "PASSWORD"
    "PgsqlDb" = "unicorndb"
    ```
### WAF
- Flooding, XSS, SQL Injection, Bad Bot, 黑白名單都可以透過[awslabs/aws-waf-security-automations](https://github.com/awslabs/aws-waf-security-automations)實現
- 部署上去來檔SQL Injection
- Request設高一點

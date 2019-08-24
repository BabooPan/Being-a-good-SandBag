# 髒東西 V2

![](https://i.imgur.com/VTNgSBr.png)


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

### CloudFormation部署
- VPC-v2先建，再弄ASG-v2
    > 注意AMI ID, Subnet ID
    > Subnet ID兩個都先選在Public Subnet比較保險
- 接著再Deploy其他的
    - CloudFront
    - S3
    - RDS-PostgreSQL
    - elasticache-memcached
- 記得改裡面的參數為貼近實際狀況
- 其他的就從Sample裡面Deploy出來，再改設定符合情境
- 飯粒們
    - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html
    - https://github.com/awslabs/aws-cloudformation-templates
    - https://github.com/widdix/aws-cf-templates

### 測試EC2當中的Application如何運作
- 依據需求在單機上安裝相關套件
- 寫成UserData

#### UserData
- 部署Application
- Application設定檔案，建議放在S3
    - 方便修改檔案後，重啟機器就能吃到最新的設定
    - 要注意S3那一段，Bucket Policy & Endpoint
    - 要記得修改Route Table

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
- 倒完之後檢查裡面有沒有兩個Table，不然就自己建
    - 一個管理用
    - 一個存資料用

- 配置檔
    ```
    "PgsqlHost" = "RDS_HOSTNAME"
    "PgsqlPort" = "RDS_SERVICE_PORT"
    "PgsqlUser" = "db-admin" #不要用Root
    "PgsqlPass" = "PASSWORD"
    "PgsqlDb" = "DB_NAME"
    ```
### WAF
- Flooding, XSS, SQL Injection, Bad Bot, 黑白名單都可以透過[awslabs/aws-waf-security-automations](https://github.com/awslabs/aws-waf-security-automations)實現
- 部署上去來檔SQL Injection
- Request Rate可以設高一點 ex. 10000/5mins

### S3 Challenge
- Account層級的Block Public Access要關掉
- Bucket層級的Block Public Access看情況處理
- 調整Bucket Policy，限定來源是Endpoint
- 如果有明確要Deny的IP，可以用Bucket Policy鎖掉Public IP
    > Private的不能
- 開Access Log

### ECS Challenge
- 確認EC2 Instance OS是什麼，決定base image from哪個OS
- 先在local寫dockerfile，驗證Application/UserData有辦法包成Container並順利執行
    > 在每一行bash前面加上`RUN`，若有需要開機運行則是`CMD`，以下為以amazon linux為例，在dockerhub上面可以找到相對應得
    ```
    # 指定Base Image, 從docker hub找
    FROM amazonlinux:1

    COPY app.py /home/ec2-user

    # 打包package
    RUN yum update -y
    RUN yum install python34 git curl -y
    RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
    RUN python get-pip.py
    RUN pip install flask
    RUN pip install boto3

    # 宣告Container對外的Service port，沒指定也沒差，給別人看的
    EXPOSE 80

    # 指定Container開啟後要執行的指令，跟機器開機腳本差不多概念
    CMD echo helloworld
    CMD python --version
    CMD python app.py
    ```
- 把Image推上去ECR
- 部署ECS Service，mount到Target上面
- ***這邊很重要，因為不知道要部署到ALB上面或者是NLB+API GW***
    - ALB
        > 應該是這個就可以了
        1. Target Group掛上去
        2. 設定ALB Routing
        3. 給Endpoint測試看看
        4. 失敗的話就是API Gateway
    - NLB + API GW
        1. Target Group掛上去
        2. 設定NLB Routing
        3. 給Endpoint測試看看
        4. 新增API Gateway
        5. 設定VPC Link，讓API GW指過去NLB，可以參考[這篇文件](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-private-integration.html)
        ![](https://i.imgur.com/jrDptj7.png)

# Project-27: AWS Stack With Cloud Formation

[*Project Source*](https://www.udemy.com/course/devopsprojects/learn/lecture/23902918#overview)


![](Images/Architecture%20setup.png)

## Pre-requisites

* AWS Account
* S3 Bucket backup in project 20
* Any IDE (VS Code, IntelliJ, etc)

## Step-1: Clone the source code repository in GitHub

- Create a workspace on any IDE and name it `CloudFormation-project`. 
- Create an s3 bucket to store the state in AWS.
```sh
vprofile-cicd-templates25
```

- Create a folder from the s3 bucket 
```sh
stack-templates
```

![](Images/s3%20bucket%20folder.png)

- Create a key pair
```sh
cicd-stack-key
```

## Step-2: Master or Root Template

- Create `cicdtemp.yaml` file under `CloudFormation-project` repo with below content:
```sh
Parameters:
  Keypair:
    Description: cicd-stack-Key
    Type: "AWS::EC2::KeyPair::KeyName"
  MyIp:
    Description: Assigning IP 
    Type: String 
    Default: 102.67.21.239/32
Resources:
  S3RoleforCiCd:
    Type: AWS::CloudFormation::Stack 
    Properties: 
      TemplateURL: https://s3.amazonaws.com/vprofile-cicd-templates25/stack-template/cicds3role.yaml
  JenkinsInst: 
    Type: AWS::CloudFormation::Stack
    DependsOn: S3RoleforCiCd 
    Properties: 
      TemplateURL: https://s3.amazonaws.com/vprofile-cicd-templates25/stack-template/jenk.yaml
      Parameters: 
        Keypair: !Ref Keypair
        MyIP: !Ref MyIp
  App01qa: 
    Type: AWS::CloudFormation::Stack
    DependsOn: JenkinsInst
    Properties: 
      TemplateURL: https://s3.amazonaws.com/vprofile-cicd-templates25/stack-template/app01qa.yaml
      Parameters: 
        Keypair: !Ref Keypair 
        MyIP: !Ref MyIp
  NexusServer: 
    Type: AWS::CloudFormation::Stack
    DependsOn: JenkinsInst
    Properties: 
      TemplateURL: https://s3.amazonaws.com/vprofile-cicd-templates25/stack-template/nexus.yaml
      Parameters: 
        Keypair: !Ref Keypair 
        MyIP: !Ref MyIp
  SonarServer: 
    Type: AWS::CloudFormation::Stack
    DependsOn: JenkinsInst
    Properties: 
      TemplateURL: https://s3.amazonaws.com/vprofile-cicd-templates25/stack-template/sonar.yaml
      Parameters: 
        Keypair: !Ref Keypair 
        MyIP: !Ref MyIp
  db01qa: 
    Type: AWS::CloudFormation::Stack
    DependsOn: JenkinsInst
    Properties: 
      TemplateURL: https://s3.amazonaws.com/vprofile-cicd-templates25/stack-template/db01qa.yaml
      Parameters: 
        Keypair: !Ref Keypair 
        MyIP: !Ref MyIp
  WintestServer:
    Type: AWS::CloudFormation::Stack
    DependsOn: JenkinsInst
    Properties: 
      TemplateURL: https://s3.amazonaws.com/vprofile-cicd-templates25/stack-template/wintest.yaml
      Parameters: 
        Keypair: !Ref Keypair 
        MyIP: !Ref MyIp
```

## Step-3: IAM Role Template

- Create `cicds3role.yaml` file under `CloudFormation-project` repo with below content:
```sh
AWSTemplateFormatVersion: 2010-09-09
Resources:
  VPS3Role:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName: vpr-cicd-data-s3
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
              - ec2.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: vps3fullaccess
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action: '*'
                Resource: 'arn:aws:s3:::*'
  VPInstanceProfile:
    Type: 'AWS::IAM::InstanceProfile'
    Properties:
      InstanceProfileName: vpr-cicd-data-s3
      Path: /
      Roles:
        - !Ref VPS3Role
Outputs:
  VPs3rolDetails:
    Description: VP CICD Pro s3 role info
    Value: !Ref VPInstanceProfile
    Export:
      Name: cicds3role-VPS3RoleProfileName
#        Fn::Sub: "${AWS::StackName}-VPS3RoleProfileName"
```

## Step-4: Jenkins Template

- Create `jenk.yaml` file under `CloudFormation-project` repo with below content:
```sh
Parameters:
  RoleTempName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: cicds3role
  MyIP:
    Type: String
  KeyName: 
    Type: String
  InstanceType:                                        
    Type: String
    Default: t2.micro

Mappings:
  AmiRegionMap:
    us-east-2:
      AMI: ami-0bbe28eb2173f6167
Resources:
  JenkinsInst:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap
#        Fn::FindInMap:
        - AmiRegionMap
        - !Ref AWS::Region
        - AMI
      Tags:
        - Key: "Name"
          Value: !Join 
            - ""
            - - "Jenkins in "
              - !Ref AWS::Region
      SecurityGroups: 
        - !Ref JenkinsSG
      IamInstanceProfile: 
        Fn::ImportValue:
            Fn::Sub: "${RoleTempName}-VPS3RoleProfileName"
      UserData:
        Fn::Base64:                                # YAML makes userdata much cleaner
          !Sub |
              #!/bin/bash
              sudo apt update
              sudo apt install openjdk-8-jdk -y
              sudo apt install maven git wget unzip -y
              sudo apt install awscli -y
              wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
              sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
              sudo apt-get update
              sudo apt-get install jenkins -y
              sleep 10
              systemctl stop jenkins
              sleep 10
              aws s3 cp s3://vprofile-cicd-templates25/jenkins_cicdjobs.tar.gz /var/lib/
              cd /var/lib/
              tar xzvf jenkins_cicdjobs.tar.gz
              chown jenkins.jenkins /var/lib/jenkins -R
              reboot 
  JenkinsSG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: JenkinsSG
      GroupDescription: Allow SSH & HTTP from myip
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: !Ref MyIP
          
        - IpProtocol: tcp
          FromPort: '8080'
          ToPort: '8080'
          CidrIp: 0.0.0.0/0
        
Outputs:
  MyEC2InstancePublicIP:
    Value: !GetAtt JenkinsInst.PublicIp
  MyEC2InstancePrivateIP:
    Value: !GetAtt JenkinsInst.PrivateIp
  MyEC2InstanceID:
    Value: !Ref JenkinsInst
  JenkSecurityGroupId:
    Description: Security Group 1 ID
    Value:
      Fn::GetAtt:
      - JenkinsSG
      - GroupId
    Export:
      Name: jenk-SGID
#        Fn::Sub: "${AWS::StackName}-SecurityGroupID" 
```

## Step-5: Nexus Template
- Create `nexus.yaml` file under `CloudFormation-project` repo with below content:
```sh
Parameters:
  RoleTempName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: cicds3role
  JenkStackName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: jenk
  MyIP:
    Type: String
  KeyName: 
    Type: String
  InstanceType:                                        
    Type: String
    Default: t2.medium
Mappings:
  AmiRegionMap:
    us-east-2:
      AMI: ami-0a75b786d9a7f8144
    
Resources:
  NexusInst:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap
#        Fn::FindInMap:
        - AmiRegionMap
        - !Ref AWS::Region
        - AMI
      Tags:
        - Key: "Name"
          Value: "Nexus Server "
      SecurityGroups: 
        - !Ref nexusSG
      IamInstanceProfile: 
        Fn::ImportValue:
            Fn::Sub: "${RoleTempName}-VPS3RoleProfileName"
      UserData:
        Fn::Base64:                                # YAML makes userdata much cleaner
          !Sub |
              #!/bin/bash
              yum install java-1.8.0-openjdk.x86_64 wget -y   
              yum install epel-release -y
              yum install awscli -y
              mkdir -p /opt/nexus/   
              mkdir -p /tmp/nexus/                           
              cd /tmp/nexus
              NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
              wget $NEXUSURL -O nexus.tar.gz
              EXTOUT=`tar xzvf nexus.tar.gz`
              NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
              rm -rf /tmp/nexus/nexus.tar.gz
              rsync -avzh /tmp/nexus/ /opt/nexus/
              useradd nexus
              chown -R nexus.nexus /opt/nexus 
              cat <<EOT>> /etc/systemd/system/nexus.service
              [Unit]                                                                          
              Description=nexus service                                                       
              After=network.target                                                            
                                                                                
              [Service]                                                                       
              Type=forking                                                                    
              LimitNOFILE=65536                                                               
              ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
              ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
              User=nexus                                                                      
              Restart=on-abort                                                                
                                                                                
              [Install]                                                                       
              WantedBy=multi-user.target                                                      

              EOT

              echo 'run_as_user="nexus"' > /opt/nexus/$NEXUSDIR/bin/nexus.rc
              systemctl daemon-reload
              systemctl start nexus
              systemctl enable nexus
              aws s3 cp s3://cicd-data-vprofile/nexus-cicd-vpro-pro.tgz  /opt/
              cd /opt/
              sleep 5
              systemctl stop nexus
              sleep 10
              tar xzvf nexus-cicd-vpro-pro.tgz 
              chown nexus.nexus /opt/nexus -R
              systemctl daemon-reload
              systemctl start nexus
              systemctl enable nexus
              reboot

  nexusSG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: nexusSG
      GroupDescription: Allow SSH & HTTP from myip
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: !Ref MyIP
          
        - IpProtocol: tcp
          FromPort: '8081'
          ToPort: '8081'
          CidrIp: !Ref MyIP
  vproappSGIngress:
   Type: 'AWS::EC2::SecurityGroupIngress'
   Properties:
      GroupId: 
        Fn::GetAtt:
        - nexusSG
        - GroupId
      IpProtocol: tcp
      FromPort: 8081
      ToPort: 8081
      SourceSecurityGroupId:
        Fn::ImportValue:
            Fn::Sub: "${JenkStackName}-SGID"
```

## Step-6: sonar Template

-Create `sonar.yaml` file under `CloudFormation-project` repo with below content:
```sh
Parameters:
  RoleTempName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: cicds3role
  JenkStackName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: jenk
  MyIP:
    Type: String
  KeyName: 
    Type: String
  InstanceType:                                        
    Type: String
    Default: t2.medium
Mappings:
  AmiRegionMap:
    us-east-2:
      AMI: ami-0bbe28eb2173f6167
  
Resources:
  SonarInst:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap
#        Fn::FindInMap:
        - AmiRegionMap
        - !Ref AWS::Region
        - AMI
      Tags:
        - Key: "Name"
          Value: !Join 
            - ""
            - - "SonarServer in "
              - !Ref AWS::Region
      SecurityGroups: 
        - !Ref sonarSG
      IamInstanceProfile: 
        Fn::ImportValue:
            Fn::Sub: "${RoleTempName}-VPS3RoleProfileName"
      UserData:
        Fn::Base64:                                # YAML makes userdata much cleaner
          !Sub |
              #!/bin/bash
              cp /etc/sysctl.conf /root/sysctl.conf_backup
              cat <<EOT> /etc/sysctl.conf
              vm.max_map_count=262144
              fs.file-max=65536
              ulimit -n 65536
              ulimit -u 4096
              EOT
              cp /etc/security/limits.conf /root/sec_limit.conf_backup
              cat <<EOT> /etc/security/limits.conf
              sonarqube   -   nofile   65536
              sonarqube   -   nproc    409
              EOT

              sudo apt-get update -y
              sudo apt-get install openjdk-11-jdk -y
              sudo update-alternatives --config java
              java -version

              sudo apt update
              wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -

              sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
              sudo apt install postgresql postgresql-contrib -y
              #sudo -u postgres psql -c "SELECT version();"
              sudo systemctl enable postgresql.service
              sudo systemctl start  postgresql.service
              sudo echo "postgres:admin123" | chpasswd
              runuser -l postgres -c "createuser sonar"
              sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
              sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
              sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
              systemctl restart  postgresql
              #systemctl status -l   postgresql
              netstat -tulpena | grep postgres
              sudo mkdir -p /sonarqube/
              cd /sonarqube/
              sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
              sudo apt-get install zip -y
              sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
              sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
              sudo groupadd sonar
              sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
              sudo chown sonar:sonar /opt/sonarqube/ -R
              cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
              cat <<EOT> /opt/sonarqube/conf/sonar.properties
              sonar.jdbc.username=sonar
              sonar.jdbc.password=admin123
              sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
              sonar.web.host=0.0.0.0
              sonar.web.port=9000
              sonar.web.javaAdditionalOpts=-server
              sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
              sonar.log.level=INFO
              sonar.path.logs=logs
              EOT

              cat <<EOT> /etc/systemd/system/sonarqube.service
              [Unit]
              Description=SonarQube service
              After=syslog.target network.target

              [Service]
              Type=forking

              ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
              ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

              User=sonar
              Group=sonar
              Restart=always

              LimitNOFILE=65536
              LimitNPROC=4096

              [Install]
              WantedBy=multi-user.target
              EOT
              
              systemctl daemon-reload
              systemctl enable sonarqube.service
              systemctl start sonarqube.service
              apt-get install nginx -y
              rm -rf /etc/nginx/sites-enabled/default
              rm -rf /etc/nginx/sites-available/default
              cat <<EOT> /etc/nginx/sites-available/sonarqube
              server{
                  listen      80;
                  server_name sonarqube.groophy.in;
              
                  access_log  /var/log/nginx/sonar.access.log;
                  error_log   /var/log/nginx/sonar.error.log;
              
                  proxy_buffers 16 64k;
                  proxy_buffer_size 128k;
              
                  location / {
                      proxy_pass  http://127.0.0.1:9000;
                      proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
                      proxy_redirect off;
                            
                      proxy_set_header    Host            \$host;
                      proxy_set_header    X-Real-IP       \$remote_addr;
                      proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
                      proxy_set_header    X-Forwarded-Proto http;
                  }
              }
              EOT
              ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
              systemctl enable nginx.service
              #systemctl restart nginx.service
              sudo ufw allow 80,9000,9001/tcp
              systemctl stop sonarqube
              systemctl stop postgresql
              sleep 10
              apt install awscli -y
              aws s3 cp s3://cicd-data-vprofile/sonarqube-vpro-pro-data.tgz /opt/              
              cd /opt/
              tar xzvf sonarqube-vpro-pro-data.tgz
              sudo chown sonar:sonar /opt/sonarqube/ -R
              aws s3 cp s3://cicd-data-vprofile/postgresql_sonar.tgz  /var/lib/
              cd /var/lib
              tar xzvf postgresql_sonar.tgz 
              chown postgres.postgres /var/lib/postgresql -R
              systemctl start postgresql
              systemctl start sonarqube
              sleep 10              
              reboot

  sonarSG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: sonarSG
      GroupDescription: Allow SSH & HTTP from myip
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: !Ref MyIP
          
        - IpProtocol: tcp
          FromPort: '80'
          ToPort: '80'
          CidrIp: !Ref MyIP
  sonarSGIngress:
   Type: 'AWS::EC2::SecurityGroupIngress'
   Properties:
      GroupId: 
        Fn::GetAtt:
        - sonarSG
        - GroupId
      IpProtocol: -1
      SourceSecurityGroupId:
        Fn::ImportValue:
            Fn::Sub: "${JenkStackName}-SGID"
  JenkinsSGIngress:
   Type: 'AWS::EC2::SecurityGroupIngress'
   Properties:
      GroupId: 
        Fn::ImportValue:
            Fn::Sub: "${JenkStackName}-SGID"
      IpProtocol: -1
      SourceSecurityGroupId:
        Fn::GetAtt:
        - sonarSG
        - GroupId
```

## Step-7: Wintest Template

- Create `Wintest.yaml` file under `CloudFormation-project` repo with below content:
```sh
Parameters:
  RoleTempName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: cicds3role
  JenkStackName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: jenk
  MyIP:
    Type: String
  KeyName: 
    Type: String
  InstanceType:                                        
    Type: String
    Default: t2.small
Mappings:
  AmiRegionMap:
    us-east-1:
      AMI: ami-032c2c4b952586f02
    us-east-2:
      AMI: ami-0239d3998515e9ed1
    eu-west-1:
      AMI: ami-0b5271aea7b566f9a
    us-west-1:
      AMI: ami-08bcc13ad2c143073
    ap-south-1:
      AMI: ami-07f7b791cbd0812bf 
    us-west-2:
      AMI: ami-029e27fb2fc8ce9d8 
Resources:
  WintestInst:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap
#        Fn::FindInMap:
        - AmiRegionMap
        - !Ref AWS::Region
        - AMI
      Tags:
        - Key: "Name"
          Value: !Join 
            - ""
            - - "Wintest in "
              - !Ref AWS::Region
      SecurityGroups: 
        - !Ref wintestSG
      IamInstanceProfile: 
        Fn::ImportValue:
            Fn::Sub: "${RoleTempName}-VPS3RoleProfileName"
      UserData:
        Fn::Base64:                                # YAML makes userdata much cleaner
          !Sub |
              <powershell>
              Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
              choco install jdk8 -y 
              choco install mvn -y 
              choco install googlechrome -y
              choco install git.install -y
              mkdir C:\jenkins
              </powershell>
  wintestSG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: wintestSG
      GroupDescription: Allow SSH & HTTP from myip
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '3389'
          ToPort: '3389'
          CidrIp: !Ref MyIP
  wintestSGIngress:
   Type: 'AWS::EC2::SecurityGroupIngress'
   Properties:
      GroupId: 
        Fn::GetAtt:
        - wintestSG
        - GroupId
      IpProtocol: -1
      SourceSecurityGroupId:
        Fn::ImportValue:
            Fn::Sub: "${JenkStackName}-SGID"
  JenkinsSGIngress:
   Type: 'AWS::EC2::SecurityGroupIngress'
   Properties:
      GroupId: 
        Fn::ImportValue:
            Fn::Sub: "${JenkStackName}-SGID"
      IpProtocol: -1
      SourceSecurityGroupId:
        Fn::GetAtt:
        - wintestSG
        - GroupId
```

## Step-8: Tomcat Template

- Create `app01qa.yaml` file under `CloudFormation-project` repo with below content:
```sh
Parameters:
  RoleTempName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: cicds3role
  JenkStackName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: jenk
  MyIP:
    Type: String
  KeyName: 
    Type: String
  InstanceType:                                        
    Type: String
    Default: t2.micro
Mappings:
  AmiRegionMap:
    us-east-2:
      AMI: ami-0bbe28eb2173f6167
  
Resources:
  App01qaInst:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap
#        Fn::FindInMap:
        - AmiRegionMap
        - !Ref AWS::Region
        - AMI
      Tags:
        - Key: "Name"
          Value: !Join 
            - ""
            - - "app01-qa-vpro in "
              - !Ref AWS::Region
      SecurityGroups: 
        - !Ref vproappSG
      IamInstanceProfile: 
        Fn::ImportValue:
            Fn::Sub: "${RoleTempName}-VPS3RoleProfileName"
      UserData:
        Fn::Base64:                                # YAML makes userdata much cleaner
          !Sub |
              #!/bin/bash
              sudo apt update
              sudo apt install openjdk-8-jdk -y
              sudo apt install git wget unzip -y
              sudo apt install awscli -y
              TOMURL="https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.37/bin/apache-tomcat-8.5.37.tar.gz"
              cd /tmp/
              wget $TOMURL -O tomcatbin.tar.gz
              EXTOUT=`tar xzvf tomcatbin.tar.gz`
              TOMDIR=`echo $EXTOUT | cut -d '/' -f1`
              useradd --shell /sbin/nologin tomcat
              rsync -avzh /tmp/$TOMDIR/ /usr/local/tomcat8/
              aws s3 cp s3://cicd-data-vprofile/tomcat-users.xml /usr/local/tomcat8/conf/tomcat-users.xml
              aws s3 cp s3://cicd-data-vprofile/context.xml /usr/local/tomcat8/webapps/manager/META-INF/context.xml
              chown -R tomcat.tomcat /usr/local/tomcat8 
              cat <<EOT>> /etc/systemd/system/tomcat.service
              [Unit]
              Description=Tomcat
              After=network.target

              [Service]
              User=tomcat
              WorkingDirectory=/usr/local/tomcat8
              Environment=CATALINA_HOME=/usr/local/tomcat8
              Environment=CATALINE_BASE=/usr/local/tomcat8
              ExecStart=/usr/local/tomcat8/bin/catalina.sh run
              ExecStop=/usr/local/tomcat8/bin/shutdown.sh
              SyslogIdentifier=tomcat-%i

              [Install]
              WantedBy=multi-user.target
              EOT

              systemctl daemon-reload
              systemctl start tomcat
              systemctl enable tomcat

  vproappSG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: vproappSG
      GroupDescription: Allow SSH & HTTP from myip
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: !Ref MyIP
          
        - IpProtocol: tcp
          FromPort: '8080'
          ToPort: '8080'
          CidrIp: 0.0.0.0/0
  vproappSGIngress:
   Type: 'AWS::EC2::SecurityGroupIngress'
   Properties:
      GroupId: 
        Fn::GetAtt:
        - vproappSG
        - GroupId
      IpProtocol: tcp
      FromPort: 8080
      ToPort: 8080
      SourceSecurityGroupId:
        Fn::ImportValue:
            Fn::Sub: "${JenkStackName}-SGID"
Outputs:
  appSecurityGroupId:
    Description: Security Group 1 ID
    Value:
      Fn::GetAtt:
      - vproappSG
      - GroupId
    Export:
      Name: app01qa-SGID
```

## Step-9:DB Template

- Create `db01qa.yaml` file under `CloudFormation-project` repo with below content:
```sh
Parameters:
  appStackName:
    Description: Name of the base stack with all infra resources
    Type: String
    Default: app01qa
  MyIP:
    Type: String
  KeyName: 
    Type: String
  InstanceType:                                        
    Type: String
    Default: t2.micro
Mappings:
  AmiRegionMap:
    us-east-2:
      AMI: ami-0a75b786d9a7f8144
   
Resources:
  DB01qaInst:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap
#        Fn::FindInMap:
        - AmiRegionMap
        - !Ref AWS::Region
        - AMI
      Tags:
        - Key: "Name"
          Value: !Join 
            - ""
            - - "db01-qa-vpro in "
              - !Ref AWS::Region
      SecurityGroups: 
        - !Ref vprodbSG
      UserData:
        Fn::Base64:                                # YAML makes userdata much cleaner
          !Sub |
              #!/bin/bash
              DATABASE_PASS='admin123'
              yum update -y
              yum install epel-release -y
              yum install mariadb-server -y
              yum install wget git unzip -y

              #mysql_secure_installation
              sed -i 's/^127.0.0.1/0.0.0.0/' /etc/my.cnf

              # starting & enabling mariadb-server
              systemctl start mariadb
              systemctl enable mariadb

              #restore the dump file for the application
              cd /tmp/
              wget https://raw.githubusercontent.com/devopshydclub/vprofile-repo/vp-rem/src/main/resources/db_backup.sql
              mysqladmin -u root password "$DATABASE_PASS"
              mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
              mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
              mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
              mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
              mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
              mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
              mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
              mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
              mysql -u root -p"$DATABASE_PASS" accounts < /tmp/db_backup.sql
              mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

              # Restart mariadb-server
              systemctl restart mariadb
              # SETUP MEMCACHE
              yum install memcached -y
              systemctl start memcached
              systemctl enable memcached
              systemctl status memcached
              memcached -p 11211 -U 11111 -u memcached -d
              sleep 30
              yum install socat -y
              yum install wget -y
              wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.10/rabbitmq-server-3.6.10-1.el7.noarch.rpm
              rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
              yum update
              rpm -Uvh rabbitmq-server-3.6.10-1.el7.noarch.rpm
              systemctl start rabbitmq-server
              systemctl enable rabbitmq-server
              systemctl status rabbitmq-server
              echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config
              rabbitmqctl add_user test test
              rabbitmqctl set_user_tags test administrator
              systemctl restart rabbitmq-server

  vprodbSG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: vprodbSG
      GroupDescription: Allow SSH & HTTP from myip
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: !Ref MyIP
  vprodbSGIngress:
   Type: 'AWS::EC2::SecurityGroupIngress'
   Properties:
      GroupId: 
        Fn::GetAtt:
        - vprodbSG
        - GroupId
      IpProtocol: -1
      SourceSecurityGroupId:
        Fn::ImportValue:
            Fn::Sub: "${appStackName}-SGID"
```

## Step-10: Create Nested Stack

- Create a folder `stack-template` under `CloudFormation-project` and move all child templates into the stack directory.
```sh
ls
mv * stack-template/
ls
mv stack-template/cicd
mv stack-template/cicdtemp.yaml
```

- Upload child's template into s3 bucket
![](Images/child's%20template%20upload.png)

![](Images/succsessful%20upload.png)

- Navigate to CloudFormation service and upload the `cicdtemp.yaml` file 
- View the stack in designer to avoid syntax error and validate the stack

![](Images/designer.png)

- Create a nested stack using below properties
```sh
stack name: vprofile-cicd-stack
key name: cicd-stack-key
```

![](Images/stack%20creation.png)

## Step-11: Validate

- Validate the stack by logging into the servers using their public IP
```sh
Jenkins server: publicIP:8080
Nexus server:   publicIP:8081
sonar server:   pubnlicIP
```
![](Images/jenkins%20validation.png)

## Step-12: Clean up

- Clean up all resources by deleting the root template
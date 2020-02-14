# Solution Build Guide

This `Solution Build Guide` steps how to solve [REA Systems Engineer practical task](https://github.com/rea-cruitment/simple-sinatra-app) and deploys it to an AWS EC2 instance. 

## Before start

* Prepare a linux host (physical or virtual) with docker up and running

```
[fli@192-168-1-10 ~]$ sudo systemctl is-active docker
[sudo] password for fli: 
active
[fli@192-168-1-10 ~]$ docker info
Client:
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 14
 Server Version: 19.03.5
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: b34a5c8af56e510852c35414db4c1f4fa6172339
 runc version: 3e425f80a8c931f88e6d94a8c831b9d5aa481657
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 3.10.0-1062.9.1.el7.x86_64
 Operating System: CentOS Linux 7 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 8
 Total Memory: 7.586GiB
 Name: 192-168-1-10.tpgi.com.au
 ID: 45NJ:KYEP:OZMC:QMQD:PKAM:7VVO:NG6J:NUQU:O7R2:JWBE:4TKV:HHBZ
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Username: fen9li
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false

[fli@192-168-1-10 ~]$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[fli@192-168-1-10 ~]$ 
```

* Prepare an aws account with enough provileges

> Note: I created a new user `readmin` for this project and configure it as default profile. 

```
[fli@192-168-1-10 ~]$ aws2 configure list-profiles | grep default
default
[fli@192-168-1-10 ~]$  
```

* Get your Internet facing IP address

> Note: you need this ip address should you want to limit your SSH Location for your new EC2 instance.

```
[fli@192-168-1-10 ~]$ curl http://checkip.amazonaws.com/
203.221.39.155
[fli@192-168-1-10 ~]$ 
```

## Step 1: clone `https://github.com/rea-cruitment/simple-sinatra-app.git`
```
[fli@192-168-1-10 ~]$ git clone https://github.com/rea-cruitment/simple-sinatra-app.git
Cloning into 'simple-sinatra-app'...
remote: Enumerating objects: 37, done.
remote: Total 37 (delta 0), reused 0 (delta 0), pack-reused 37
Unpacking objects: 100% (37/37), done.
[fli@192-168-1-10 ~]$

[fli@192-168-1-10 ~]$ cd simple-sinatra-app/
[fli@192-168-1-10 simple-sinatra-app]$ 

[fli@192-168-1-10 simple-sinatra-app]$ ll -a
total 24
drwxrwxr-x.  3 fli fli  106 Feb 10 15:26 .
drwx-----x. 55 fli fli 4096 Feb 10 15:26 ..
-rw-rw-r--.  1 fli fli   50 Feb 10 15:26 config.ru
-rw-rw-r--.  1 fli fli   43 Feb 10 15:26 Gemfile
drwxrwxr-x.  8 fli fli  163 Feb 10 15:26 .git
-rw-rw-r--.  1 fli fli   18 Feb 10 15:26 .gitignore
-rw-rw-r--.  1 fli fli   50 Feb 10 15:26 helloworld.rb
-rw-rw-r--.  1 fli fli 2058 Feb 10 15:26 README.md
[fli@192-168-1-10 simple-sinatra-app]$ 
```

## Step 2: prepare your working repository

* Create new repository `rea-sinatra-aws` in your GitHub account 

Here is the guide on how to [Create a repo](https://help.github.com/en/github/getting-started-with-github/create-a-repo) on GitHub.

* Change remote url to your new repository

```
[fli@192-168-1-10 rea-sinatra-aws]$ git remote set-url origin git@github.com:fen9li/rea-sinatra-aws.git
[fli@192-168-1-10 rea-sinatra-aws]$ git remote -v
origin  git@github.com:fen9li/rea-sinatra-aws.git (fetch)
origin  git@github.com:fen9li/rea-sinatra-aws.git (push)
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

* Create, change to new working branch `develop` and push to GitHub repo

```
[fli@192-168-1-10 rea-sinatra-aws]$ git status
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
[fli@192-168-1-10 rea-sinatra-aws]$ git checkout -b develop
Switched to a new branch 'develop'
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

## Step 3: prepare docker image

* Create `Dockerfile`
```
[fli@192-168-1-10 rea-sinatra-aws]$ cat Dockerfile
FROM ruby:2.7.0-slim

RUN apt-get update -qq && apt-get install -y build-essential

ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

ADD Gemfile* $APP_HOME/
RUN bundle config set without 'development test'
RUN bundle install

ADD . $APP_HOME

EXPOSE 9292

CMD ["bundle", "exec", "rackup", "--host", "0.0.0.0", "-p", "9292"]
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

* Build docker image `sinatra-test:v1.0.0` and test it locally

```
docker build --no-cache -t sinatra-test:v1.0.0 .

[fli@192-168-1-10 rea-sinatra-aws]$ docker run -d --rm --name sinatra -p 80:9292 sinatra-test:v1.0.0
726ec1a3bf86c03bd36f3b66f9cb5842ac36ecb71e8ced2b960c22bed36920fa
[fli@192-168-1-10 rea-sinatra-aws]$ docker container ls
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                  NAMES
726ec1a3bf86        sinatra-test:v1.0.0   "bundle exec rackup …"   12 seconds ago      Up 10 seconds       0.0.0.0:80->9292/tcp   sinatra
[fli@192-168-1-10 rea-sinatra-aws]$ docker port 726
9292/tcp -> 0.0.0.0:80
[fli@192-168-1-10 rea-sinatra-aws]$ curl localhost
Hello World![fli@192-168-1-10 rea-sinatra-aws]$ docker logs 726
[2020-02-12 03:39:30] INFO  WEBrick 1.6.0
[2020-02-12 03:39:30] INFO  ruby 2.7.0 (2019-12-25) [x86_64-linux]
[2020-02-12 03:39:30] INFO  WEBrick::HTTPServer#start: pid=1 port=9292
172.17.0.1 - - [12/Feb/2020:03:39:57 +0000] "GET / HTTP/1.1" 200 12 0.0346
[fli@192-168-1-10 rea-sinatra-aws]$  
```

* Tag and push this docker image to DockerHub

```
[fli@192-168-1-10 ~]$ docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
sinatra-test         v1.0.0              03243cafe1e3        3 days ago          390MB
ruby                 2.7.0-slim          b913dc62d63c        11 days ago         149MB
[fli@192-168-1-10 ~]$ docker tag 03243cafe1e3 fen9li/sinatra-test:v1.0.0
[fli@192-168-1-10 ~]$

[fli@192-168-1-10 ~]$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: fen9li
Password: 
WARNING! Your password will be stored unencrypted in /home/fli/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[fli@192-168-1-10 ~]$ 

[fli@192-168-1-10 ~]$ docker push fen9li/sinatra-test:v1.0.0
The push refers to repository [docker.io/fen9li/sinatra-test]
605fb97e726f: Pushed 
b95527fc9403: Pushed 
1d19c9ac67c5: Pushed 
a80e93cfbf55: Pushed 
07fa219dbca3: Pushed 
93eb6e3867f4: Pushed 
a5b96a33a938: Mounted from library/ruby 
95ab215e27f8: Mounted from library/ruby 
29d6d7a2eb3a: Mounted from library/ruby 
6bab58ebc554: Mounted from library/ruby 
488dfecc21b1: Mounted from library/ruby 
v1.0.0: digest: sha256:64a1d2267cb726ec0b84beecb4c9198d7b7e16696cfea2b7b285a818c5749aef size: 2621
[fli@192-168-1-10 ~]$ 
```

## Step 4: deploy `simple-sinatra-app` to aws

* Create CloudFormation template `simple-sinatra-app-stack-dev.yaml` & `simple-sinatra-app-stack-prod.yaml`

> Note: CloudFormation template `simple-sinatra-app-stack-dev.yaml` is designed to be used at developing stage. It supports SSH login to the EC2 instance. CloudFormation template `simple-sinatra-app-stack-prod.yaml` is designed to be used at production stage. No SSH login is allowed to the EC2 instance in production.

```
[fli@192-168-1-10 rea-sinatra-aws]$ vim simple-sinatra-app-stack-dev.yaml 
[fli@192-168-1-10 rea-sinatra-aws]$ cat simple-sinatra-app-stack-dev.yaml 
---
AWSTemplateFormatVersion: '2010-09-09'

Description: 'AWS CloudFormation Template to create AWS resources to deploy simple sinatra app.'

Parameters: 
  ImageId: 
    Description: as the name
    Type: String
    Default: ami-0b8b10b5bf11f3a22 
  KeyName: 
    Description: Name of an existing EC2 KeyPair to enable SSH access to the instance
    Type: String
    Default: rea-sinatra 
  InstanceType: 
    Description: as the name
    Type: String
    Default: t3.nano 
  SSHLocation: 
    Description: The IP address range that can be used to SSH to the EC2 instances
    Type: String
    Default: "203.221.39.155/32"
    AllowedPattern: "(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})/(\\d{1,2})"
    ConstraintDescription: "must be a valid IP CIDR range of the form x.x.x.x/x."
    
Resources:
  
  Vpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags: 
        - Key: Name
          Value: rea-test
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: 
        - Key: Name
          Value: rea-test
  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref Vpc
  PublicSubnetAZ1:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Select 
        - 0
        - Fn::GetAZs: {Ref: 'AWS::Region'}
      VpcId: !Ref Vpc
      CidrBlock: '10.0.10.0/24'
      MapPublicIpOnLaunch: true
      Tags: 
        - Key: Name
          Value: rea-test
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref Vpc
      Tags: 
        - Key: Name
          Value: rea-test
  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  PublicSubnetAZ1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnetAZ1
  Ec2:
    Type: AWS::EC2::Instance
    Properties: 
      KeyName: !Ref KeyName
      InstanceType: !Ref InstanceType
      ImageId: !Ref ImageId
      SubnetId: !Ref PublicSubnetAZ1
      SecurityGroupIds:  
        - !Ref InstanceSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          yum update -y
          amazon-linux-extras install docker
          service docker start
          docker run -d --rm --name sinatra -p 80:9292 fen9li/sinatra-test:v1.0.0
      Tags: 
        - Key: Name
          Value: rea-test
  InstanceSecurityGroup: 
    Type: AWS::EC2::SecurityGroup
    Properties: 
      GroupDescription: Access to EC2 instance
      VpcId: !Ref Vpc
      Tags: 
        - Key: Name
          Value: rea-test
  InstanceSecurityGroupIngressFromSSHLocation:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Ingress from SSHLocation
      GroupId: !Ref InstanceSecurityGroup
      IpProtocol: tcp
      FromPort: 22
      ToPort: 22
      CidrIp: !Ref SSHLocation
      Tags: 
        - Key: Name
          Value: rea-test
  InstanceSecurityGroupIngressFromPublic:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Ingress for http
      GroupId: !Ref InstanceSecurityGroup
      IpProtocol: tcp
      FromPort: 80
      ToPort: 80
      CidrIp: '0.0.0.0/0'
      Tags: 
        - Key: Name
          Value: rea-test
 
Outputs:
  PublicIp:
    Description: public ip address of this ec2 instance
    Value: !GetAtt Ec2.PublicIp
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

* spin up the `rea-sinatra` stack
```
aws2 cloudformation create-stack --stack-name rea-sinatra --template-body file://simple-sinatra-app-stack-dev.yaml
```

* Wait till the Stack is fully up and get the output -- it is an ip address
```
[fli@192-168-1-10 rea-sinatra-aws]$ aws2 cloudformation describe-stacks --stack-name rea-sinatra --query 'Stacks[].Outputs[].OutputValue'
[
    "52.62.237.212"
]
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

## Step 5: test it

* test it by using curl command

```
[fli@192-168-1-10 rea-sinatra-aws]$ curl 52.62.237.212
Hello World![fli@192-168-1-10 rea-sinatra-aws]$ 
```

> The screenshot    
![test-sinatra-01](images/test-sinatra-01.png)

* test it by using browser

> The screenshot    
![test-sinatra-02](images/test-sinatra-02.png)

## Step 6: dont forget git housekeeping
> Note: you can merge it to `master` branch if you want.    

```
git add .
git commit -am 'publish solution'
git push
```

## Q & A
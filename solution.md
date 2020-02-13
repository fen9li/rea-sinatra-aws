## setup
### clone original git repository
```
[fli@192-168-1-10 ~]$ mkdir rea-sinatra-aws
[fli@192-168-1-10 ~]$ cd rea-sinatra-aws
[fli@192-168-1-10 ~]$ git clone https://github.com/rea-cruitment/simple-sinatra-app.git .
Cloning into 'simple-sinatra-app'...
remote: Enumerating objects: 37, done.
remote: Total 37 (delta 0), reused 0 (delta 0), pack-reused 37
Unpacking objects: 100% (37/37), done.
[fli@192-168-1-10 ~]$

[fli@192-168-1-10 ~]$ cd rea-sinatra-aws/
[fli@192-168-1-10 rea-sinatra-aws]$ 

[fli@192-168-1-10 rea-sinatra-aws]$ ll -a
total 24
drwxrwxr-x.  3 fli fli  106 Feb 10 15:26 .
drwx-----x. 55 fli fli 4096 Feb 10 15:26 ..
-rw-rw-r--.  1 fli fli   50 Feb 10 15:26 config.ru
-rw-rw-r--.  1 fli fli   43 Feb 10 15:26 Gemfile
drwxrwxr-x.  8 fli fli  163 Feb 10 15:26 .git
-rw-rw-r--.  1 fli fli   18 Feb 10 15:26 .gitignore
-rw-rw-r--.  1 fli fli   50 Feb 10 15:26 helloworld.rb
-rw-rw-r--.  1 fli fli 2058 Feb 10 15:26 README.md
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

### create new repository `rea-sinatra-aws` in github `fen9li` account 
```
[fli@192-168-1-10 rea-sinatra-aws]$ git remote set-url origin git@github.com:fen9li/rea-sinatra-aws.git
[fli@192-168-1-10 rea-sinatra-aws]$ git remote -v
origin  git@github.com:fen9li/rea-sinatra-aws.git (fetch)
origin  git@github.com:fen9li/rea-sinatra-aws.git (push)
[fli@192-168-1-10 rea-sinatra-aws]$ 

[fli@192-168-1-10 rea-sinatra-aws]$ git status
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
[fli@192-168-1-10 rea-sinatra-aws]$ git checkout -b develop
Switched to a new branch 'develop'
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

## prepare docker image
### create `Dockerfile`
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

### build docker image `sinatra-test:v1.0.0` and test it locally

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

### push this docker image from local docker host to DockerHub

```
[fli@192-168-1-10 ~]$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: fen9li
Password: 
WARNING! Your password will be stored unencrypted in /home/fli/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[fli@192-168-1-10 ~]$ 

[fli@192-168-1-10 ~]$ docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
sinatra-test         v1.0.0              03243cafe1e3        3 days ago          390MB
ruby                 2.7.0-slim          b913dc62d63c        11 days ago         149MB
[fli@192-168-1-10 ~]$ docker tag 03243cafe1e3 fen9li/sinatra-test:v1.0.0
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

## deploy `simple-sinatra-app` to aws

### spin up the `rea-sinatra` stack
```
aws2 cloudformation create-stack --stack-name rea-sinatra --template-body file://simple-sinatra-app-stack.yaml
```

### get the output - it is an ip address
```
[fli@192-168-1-10 rea-sinatra-aws]$ aws2 cloudformation describe-stacks --stack-name rea-sinatra --query 'Stacks[].Outputs[].OutputValue'
[
    "52.62.237.212"
]
[fli@192-168-1-10 rea-sinatra-aws]$ 
```

### test it by using curl

```
[fli@192-168-1-10 rea-sinatra-aws]$ curl 52.62.237.212
Hello World![fli@192-168-1-10 rea-sinatra-aws]$ 
```

![test-sinatra-01](images/test-sinatra-01.png)

### test it by using chrome

![test-sinatra-02](images/test-sinatra-02.png)

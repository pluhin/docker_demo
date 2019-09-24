# Efficient docker containers 

> Many thanks to [@ric__harvey](https://twitter.com/ric__harvey) (Ric Harvey) and [AWS Dev Day organizers](https://aws.amazon.com/events/devdays-benelux/) for knowledge

[Slides](https://www.slideshare.net/siarheipishchyk/efficient-containers-120325715)

Created just for demo
## Repositories
+ [this repo](https://github.com/pluhin/docker_demo)
+ [microscanner-wrapper](https://github.com/pluhin/microscanner-wrapper)

## Security testing of docker containers
Requirements:
- Linux
- docker daemon installed and up
- git, wget

Get token key here: https://microscanner.aquasec.com/signup
Cone repo: 
```
$ git clone git@github.com:pluhin/microscanner-wrapper.git
$ cd microscanner-wrapper
```
Create any image (you can do in another shell window/tab): 
```
$ vim Dockerfile
```
```
FROM debian:jessie


RUN apt-get update
RUN apt-get install -y samba nginx openssl git wget curl
```
Build it:
```
$ docker build -t sp:v1 .
```
Before run microscanner-wrapper you need apply secutity tocken
```
$ export MICROSCANNER_TOKEN=<your_tocken>
```
or: add to the environment file 
``` 
$ sudo vim /etc/environment && source /etc/environment
$ sudo ./grabhtml.sh sp:v1 > v1.html
```
Analyse result
```
$ firefox v1.html
```

## Make efficient your docker containers
### Unstable repository
Create working folder:
```
$ $ mkdir demo && cd demo
```
Clone repositories
```
$ git clone git@github.com:pluhin/microscanner-wrapper.git
$ git clone git@github.com:pluhin/docker_demo.git
```
First Dockerfile1:
```
FROM debian:jessie

RUN apt-get update
RUN apt-get install -y samba nginx openssl git wget curl
```
Build it:
```
$ docker build -t sp:v1 -f Dockerfile1 .
```
Display this image
```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              660b4a078cb1        13 minutes ago      265MB
sp                  v1                  d87521c9b7ea        20 minutes ago      238MB
debian              wheezy              884b80eb370d        5 days ago          88.3MB
```

Check security vulnerability  
```
$ cd microscanner-wrapper/
$ sudo ./grabhtml.sh sp:v1 > v1.html; firefox file://$PWD/v1.html
```
### Stable repository
Go up ../
Second  Dockerfile2:
```
FROM debian:stretch

RUN apt update
RUN apt upgrade -y
RUN apt install -y samba nginx openssl git wget curl
```
Build it:
```
$  docker build -t sp:v2 -f Dockerfile2 .
```
Display this image
```bash
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
sp                  v2                  c79325ef58a8        9 seconds ago       407MB
<none>              <none>              660b4a078cb1        21 minutes ago      265MB
sp                  v1                  d87521c9b7ea        27 minutes ago      238MB
debian              wheezy              884b80eb370d        5 days ago          88.3MB
debian              stretch             be2868bebaba        5 days ago          101MB
```
Check security vulnerability  
```
$ cd microscanner-wrapper/
$ sudo ./grabhtml.sh sp:v2 > v2.html; firefox file://$PWD/v2.html
```

## User permission
Check permission for default user (spoiler: it will be root by default)
```
$ sudo docker run -it sp:v2 bash
root@babd5ba47d39:/# apt-get install mc
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  libglib2.0-0 libglib2.0-data libslang2 mc-data shared-mime-info unzip xdg-user-dirs
Suggested packages:
  arj catdvi | texlive-binaries dbview djvulibre-bin genisoimage gv imagemagick libaspell-dev
  links | w3m | lynx odt2txt poppler-utils python-boto python-tz xpdf | pdf-viewer zip
The following NEW packages will be installed:
  libglib2.0-0 libglib2.0-data libslang2 mc mc-data shared-mime-info unzip xdg-user-dirs
0 upgraded, 8 newly installed, 0 to remove and 0 not upgraded.
Need to get 8444 kB of archives.
After this operation, 29.3 MB of additional disk space will be used.
Do you want to continue? [Y/n] ^C
root@babd5ba47d39:/# exit
```
Let’s make more secure container:
Go up ../
Second  Dockerfile3:
```
FROM debian:stretch

RUN apt update
RUN apt upgrade -y
RUN apt install -y samba nginx openssl git wget curl

RUN useradd -ms /bin/bash test
USER test
```
Build it:
```
$ docker build -t sp:v3 -f Dockerfile3 .
```
And try to install anything:
```
$ sudo docker run -it sp:v3 bash
test@1d45453bcfe9:/$ apt install mc
E: Could not open lock file /var/lib/dpkg/lock - open (13: Permission denied)
E: Unable to lock the administration directory (/var/lib/dpkg/), are you root?
```
## More secure!  
You can pass security parameters using seccomp (https://docs.docker.com/engine/security/seccomp/)
Create file ```policy.json``` with the following content:
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "name": "mkdir",
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "name": "chmod",
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

And run last one container with new security policy
```
$ sudo docker run -it --security-opt seccomp:policy.json sp:v3 bash
test@c54c60626e93:/$ cd ~  
test@c54c60626e93:~$ mkdir ttt
mkdir: cannot create directory 'ttt': Operation not permitted
```
## To be slim
Let’s create more slim/light container:

Next Dockerfile:
```
FROM debian:stretch-slim

RUN apt update
RUN apt upgrade -y
RUN apt install -y samba nginx openssl git wget curl

RUN useradd -ms /bin/bash test
USER test
```

Build it:
```
$ sudo docker build -t sp:v4 -f Dockerfile4 .
```
Now let’s look how many it takes 
```
$ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
sp                  v4                  3d6b467a06d9        36 seconds ago      331MB
sp                  v3                  98390390c9b0        25 minutes ago      407MB
sp                  v2                  c79325ef58a8        31 minutes ago      407MB
<none>              <none>              660b4a078cb1        About an hour ago   265MB
sp                  v1                  d87521c9b7ea        About an hour ago   238MB
debian              wheezy              884b80eb370d        5 days ago          88.3MB
debian              stretch-slim        4b4471f624dc        5 days ago          55.3MB
debian              stretch             be2868bebaba        5 days ago          101MB
```
### Layers experiments
For first demo let’s create the following Dockerfile5:
```
FROM debian:stretch-slim

RUN apt update
RUN apt upgrade -y
RUN apt install -y samba nginx openssl git wget curl


RUN apt remove -y samba openssl git wget curl
# everything but nginx

RUN useradd -ms /bin/bash test
USER test
```
And build it:
```
$ sudo docker build -t sp:v5 -f Dockerfile5 .
```
Display size:
```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
sp                  v5                  01dbbf8b3ff0        20 seconds ago      333MB
sp                  v4                  3d6b467a06d9        6 minutes ago       331MB
sp                  v3                  98390390c9b0        31 minutes ago      407MB
sp                  v2                  c79325ef58a8        37 minutes ago      407MB
<none>              <none>              660b4a078cb1        About an hour ago   265MB
sp                  v1                  d87521c9b7ea        About an hour ago   238MB
debian              wheezy              884b80eb370d        5 days ago          88.3MB
debian              stretch-slim        4b4471f624dc        5 days ago          55.3MB
debian              stretch             be2868bebaba        5 days ago          101MB
```
You can see last image bigger than previous images.
Try to check how many layers this image has
```
$ docker inspect sp:v5
```
Let’s build dockerfile, but we make a chain from our commands
Dockerfile6:
```
FROM debian:stretch-slim

RUN apt update \
    && apt upgrade -y \
    && apt install --no-install-recommends --no-install-suggests -y samba nginx openssl git wget curl \
    && apt remove --purge --auto-remove -y samba openssl git wget curl \
    && rm -rf /var/lib/apt/lists/*
    # Clear up the cache also

USER www-data
```
And build it:
```
$ sudo docker build -t sp:v6 -f Dockerfile6 .
```
Display size:
```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
sp                  v6                  a34bba56ef09        2 minutes ago       107MB
sp                  v5                  01dbbf8b3ff0        8 minutes ago       333MB
sp                  v4                  3d6b467a06d9        14 minutes ago      331MB
sp                  v3                  98390390c9b0        39 minutes ago      407MB
sp                  v2                  c79325ef58a8        45 minutes ago      407MB
<none>              <none>              660b4a078cb1        About an hour ago   265MB
sp                  v1                  d87521c9b7ea        About an hour ago   238MB
debian              wheezy              884b80eb370d        5 days ago          88.3MB
debian              stretch-slim        4b4471f624dc        5 days ago          55.3MB
debian              stretch             be2868bebaba        5 days ago          101MB
```
Try to check how many layers this image has:
```
$ docker inspect sp:v6
```
### Change basic OS

Final step will contain minimisation of size by using alpine OS, and expose ngnix outside with default settings. For this I’ve created folder v7 with the following files:

- Dockerfile7  
- index.html  
- nginx.conf

Content of dockerfile:
```
FROM alpine:latest

RUN apk update \
    && apk upgrade \
    && apk add --no-cache nginx bash\
    && adduser -D -g 'www' www \
    && mkdir /www \
    && mkdir -p /run/nginx/ \
    && chown -R www:www /var/lib/nginx \
    && chown -R www:www /www

COPY nginx.conf /etc/nginx/nginx.conf
COPY index.html /www

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
Make it
```
$ sudo docker build -t sp:v7 -f Dockerfile7 .
```
And run it
```
$ sudo docker run -d -p 80:80 sp:v7
```

Docker images size
```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
sp                  v7                  77729ab972b8        4 minutes ago       9.63MB
sp                  v6                  a34bba56ef09        24 minutes ago      107MB
sp                  v5                  01dbbf8b3ff0        30 minutes ago      333MB
sp                  v4                  3d6b467a06d9        37 minutes ago      331MB
sp                  v3                  98390390c9b0        About an hour ago   407MB
sp                  v2                  c79325ef58a8        About an hour ago   407MB
<none>              <none>              660b4a078cb1        About an hour ago   265MB
sp                  v1                  d87521c9b7ea        2 hours ago         238MB
debian              wheezy              884b80eb370d        5 days ago          88.3MB
debian              stretch-slim        4b4471f624dc        5 days ago          55.3MB
debian              stretch             be2868bebaba        5 days ago          101MB
alpine              latest              196d12cf6ab1        5 weeks ago         4.41MB

```
## Appendix

Remove all images 
 ```
 $ docker rmi -f $(docker images -a -q)
 ```
Remove all containers
```
$ docker rm $(docker ps -a -q)
```


[HowToDo](https://docs.google.com/document/d/1KvR6kOcp5RqPVxBEatTa0Ad-xXEHImf2TAjBN8RZEh4/edit#heading=h.kl7go06fq85k) |

[Presentation](https://docs.google.com/presentation/d/1P3_YFFHU-sJOgduQYDATqSi1aTTXw31DaawYB1reJ2U/edit?usp=sharing)

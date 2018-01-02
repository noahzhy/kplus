---
layout: post
current: post
cover: False
navigation: True
title: CentOS7 + Apache + Tomcat8 연동(mod_jk)
date: 2017-06-24 10:18:00
tags: develop
class: post-template
subclass: 'post tag-develop'
logo: assets/images/slaysd.png
author: slaysd
---
# 개요

이번에는 CentOS7 환경에서 아파치 `httpd`와 `tomcat8`을 연동하는 방법에 대해서 알아보도록 하겠습니다. 평소 웹 환경을 구축해야하는 경우가 많은데 매번 할 때마다 까먹고 찾고 하는 시간이 너무 오래 걸리는 것같아 이번에 다시 환경을 만들면서 설치 순서를 기록으로 남겼습니다. 기본적인 환경 구축만을 목표로 하기 때문에 간단하고 추가되는 설정은 없습니다. 환경을 구축하고 난 뒤에 필요하신 옵션을 추가해서 사용하세요~! 지금 부터 알아보도록 하겠습니다.

<br/>

*   *   *

<br/>

## 환경을 구축하기 위한 패키지 및 Apache Httpd 설치

우선 환경을 구축하기 위해서 필요로 하는 패키지를 한번에 설치하도록 하겠습니다.

~~~ bash
yum install -y gcc gcc-c++ httpd-devel java-1.8.0-openjdk-devel.x86_64 wget libtool make
~~~

기본적으로 `libtool`, `gcc`환경과 `java jdk`가 설치 되어 있어야하며, `wget`이나 `make`는 설치가 안되 있으신 분들은 위해 넣었습니다. 이와 함께 제일 설치하기 쉬운 아파치 설치는 `httpd-devel`을 통해서 설치가 가능합니다. 위 명령어를 이용하면 환경을 구축하는데 있어 필요로하는 패키지들과 `apache`가 설치가 완료 하셨습니다.

<br/>

## Tomcat8 설치

저는 `tomcat8`을 설치 하게 되었는데요. 이는 http://tomcat.apache.org/download-80.cgi에 들어가시면 최신버전의 톰캣 `tar.gz`압축 파일을 받으실 수 있습니다.

~~~ bash
wget http://mirror.navercorp.com/apache/tomcat/tomcat-8/v8.5.23/bin/apache-tomcat-8.5.23.tar.gz
tar zxvf apache-tomcat-8.5.23.tar.gz
mv apache-tomcat-8.5.23.tar.gz /opt/tomcat
~~~

위와 같은 명령어를 통해서 현재 폴더에 톰캣 압축파일을 받아옵니다.  그 후 `tar`명령어를 통해 압축을 풀어주고 난 뒤에 해당 폴더를 그대로 `/opt/tomcat`으로 이동을 시켜줍니다.

<br/>

~~~ bash
sudo groupadd tomcat
sudo useradd -M -s /bin/nologin -g tomcat -d /opt/tomcat tomcat
sudo chown -R tomcat:tomcat /opt/tomcat
~~~

그 후 톰캣을 사용하는 유저를 지정해주셔야합니다. `tomcat`그룹과 `tomcat`유저를 생성하면서 구동파일들을 옮겨놨던 `/opt/tomcat`을 지정해주고, 해당 폴더의 소유자를 `tomcat`으로 변경해주시면 됩니다.

<br/>

그 다음에는 톰캣을 서비스에서 등록하고 서버가 부팅될 때 바로 켜지도록 설정을 할 것입니다. 수동으로 켜고 끄시는 것만 하실 경우에는 서비스만 등록하시고 `chkconfig` 명령어만 입력 안하셔도 괜찮습니다.

### Tomcat 서비스 파일 생성

~~~ bash
vim /etc/systemd/system/tomcat.service
~~~

~~~ vim
[Unit]
Description=Apache Tomcat Web Application Container
After=syslog.target network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-1.b12.el7_4.x86_64
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/bin/kill -15 $MAINPID

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
~~~

`JAVA_HOME`과 같은 경우에는 혹시나 다를까 싶어 해당하는 경로를 가져오는 방법까지 첨부합니다.

~~~ bash
which javac
readlink -f {which javac로 나온 경로를 입력}
~~~

위와 같은 명령어로 `java`가 불려지는 파일 링크의 절대경로를 읽어 올 수 있습니다. 명령어로 얻은 주소에서 `bin`경로 전까지의 경로를 `JAVA_HOME`에 입력하시면됩니다.



### Tomcat 서버 부팅시 자동시작

~~~ bash
chkconfig tomcat on
~~~

`chkconfig`를 통해서 `tomcat`서비스가 자동으로 시작되도록 추가해주시면 됩니다.

<br/>

## Apache Httpd와 Tomcat 연동(mod_jk)

`Apache`와 `tomcat`을 연동하기 위해서는 모듈이 필요한데요. 가장 많이 사용하는 `mod_jk`를 이용해서 연동해보도록 하겠습니다. 우선 `mod_jk`를 설치파일을 다운 받아야합니다. 파일은 http://tomcat.apache.org/download-connectors.cgi에서 다운로드 받으실 수 있습니다. 

~~~ bash
wget http://apache.mirror.cdnetworks.com/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.42-src.tar.gz
tar zxvf tomcat-connectors-1.2.42-src.tar.gz
cd tomcat-connectors-1.2.42-src/native/
~~~

다운로드를 받으신 후 압축을 푸신다음에 해당 폴더에 `native`폴더로 들어가셔서 설치를 하셔야 합니다.

~~~ bash
./buildconf.sh
./configure --with-apxs=/usr/bin/apxs
make
make install
~~~

위와 같은 순서대로 설치를 진행해주세요. 혹시나 다른 리눅스 환경에서 설치를 하시는 경우에는 `--with-apxs`의 폴더 위치가 다를 수 있습니다. 다른 환경에서 진행 시 `apxs`를 찾지 못하는 오류가 있으실 경우에는 리눅스 환경에 맞는 경로를 따로 찾으셔서 적용해주세요. CentOS의 경우에는 위 명령어대로 설치하시면 문제 없이 설치가 완료를 할 수 있습니다.

<br/>

### 연동 설정

설치를 모두 완료 하셨으니 인제 `httpd`와 `tomcat`간 설정을 해주셔야합니다. 우선 아파치 설정을 잡을 것입니다. CentOS를 사용하시는 경우 아파치 관련 폴더인  `/etc/httpd/`에서 폴더 구조가 `conf`, `conf.d`, `conf.modules.d`와 같이 나뉘어져 있습니다. `conf`는 아파치의 설정파일만 존재하도록 구성되어있고 추가적인 모듈별 설정 같은 것을 `conf.d`에 생성하셔서 사용하시면 자동으로 불러와 적용되어집니다. `conf.modules.d`와 같은 경우에는 사용할 모듈을 불러오는 설정만 적용하는 파일들을 관리하는 폴더입니다.

~~~ bash
cd /etc/httpd/conf.modules.d/
vim 00-jk.conf
~~~

~~~ vim
LoadModule jk_module modules/mod_jk.so
~~~

우선 `conf.module.d`폴더에 `mod_jk`관련 모듈을 불러오기 위해 `00-jk.conf`파일을 생성하고 `LoadModule ~~`을 입력하고 저장해주세요. 이는 방금전 설치한 `mod_jk`모듈을 불러와 `jk_module`이라는 이름으로 명명하는 구문입니다.

<br/>

~~~ bash
cd /etc/httpd/conf.d/
vim httpd-jk.conf
~~~

~~~ vim
<IfModule jk_module>

    # We need a workers file exactly once
    # and in the global server
    JkWorkersFile conf.d/workers.properties

    JkLogStampFormat "[%a %b %d %H:%M:%S %Y] "
    # Our JK error log
    # You can (and should) use rotatelogs here
    JkLogFile logs/mod_jk.log

    # Our JK log level (trace,debug,info,warn,error)
    JkLogLevel info

    # Our JK shared memory file
    JkShmFile logs/mod_jk.shm

    # If you want to put all mounts into an external file
    # that gets reloaded automatically after changes
    # (with a default latency of 1 minute),
    # you can define the name of the file here.
    JkMountFile conf.d/uriworkermap.properties

</IfModule>
~~~

그 후 불러온 모듈에 대한 설정을 진행해 줍니다. 주의해서 보셔야 할 것은 `JkWorkersFile`과 `JkMountFile`입니다. 전자는 톰캣 서버들을 각각 설정하는 역할을 해주며, 후자는 특정 패턴을 가지는 파일을 요청했을 때 서비스하고자 하는 톰캣 서버를 지정하는 역할을 합니다.

~~~ bash
cd /etc/httpd/conf.d/
vim workers.properties
~~~

~~~ vim
worker.list=instance1,instance2,instance3

worker.instance1.port=8109
worker.instance1.host=localhost
worker.instance1.type=ajp13
worker.instance1.lbfactor=1

worker.instance2.port=8209
worker.instance2.host=localhost
worker.instance2.type=ajp13
worker.instance2.lbfactor=1

worker.instance3.port=8309
worker.instance3.host=localhost
worker.instance3.type=ajp13
worker.instance3.lbfactor=1
~~~

`workers.properties`파일은 위와같이 톰캣서버가 3개일 경우 톰캣서버의 이름들을 설정하고 해당 서버에 대한 설정을 이루도록 되어있습니다. 구축하시려는 환경에 맞게 옵션을 설정해주시면 됩니다.

<br/>

~~~ bash
vim uriworkermap.properties
~~~

~~~ vim
/*.jsp=instance1
/*.png=instance2
/*.css=instance3
~~~

`uriworkermap.properties`에는 특정 파일에 대해서 어떤 톰캣서버가 서비스할 것인지 맵핑하실 수 있습니다. 해당하는 파일들을 따로 나누어 관리하는 이유는 톰캣이 기본 10분마다 다시불러와 적용하기 때문에 톰캣 설정 변경을 위해 서비스를 재시작하지 않으셔도 적용이 가능하다는 이점이있기 때문입니다.

<br/>

## 서비스 재시작

인제 모든 설정을 마무리했습니다. `httpd`서비스와 `tomcat`서비스를 재시작하셔서 설정된 내용들을 적용하시면 됩니다.

~~~ bash
service httpd restart
service tomcat restart
~~~

<br/>

### 팁1: 연동 설정이 정상적으로 적용이 되는지 확인하는 방법

아파치 `httpd`에는 정상적으로 설정파일들이 정상적인 Syntax를 가지고 있는지 확인하는 명령어를 내장하고 있습니다.

~~~ bash
apachectl configtest
~~~

### 팁2: 톰캣 인코딩설정

톰캣에서 한글을 사용할 때 인코딩 문제로 깨지는 문제가 발생하실 수 있기 때문에 인코딩을 설정을 하셔야 합니다.

~~~ bash
vim /opt/tomcat/conf/server.xml
~~~

~~~ vim
(생략)
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" URIEncoding="UTF-8" />
(생략)
~~~

위와 같이 AJP 커넥터에 `URIEncoding="UTF-8"`을 적용하시면 됩니다.

<br/>

*   *   *

<br/>

# 마치며

매번 환경을 구축할 때마다 괜한 곳에서 헤매는 경우가 많아 제가 많이 사용하는 서버환경에서 구축하는 방법을 정리해보았습니다. 여러번 시도하면서 군더더기 없는 설치를 위해 여러번을 엎고 다시 기록했습니다. 많은 분들께서 도움이 되셨으면 좋겠습니다. 감사합니다!
---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: false
---

# maven

### 1、安装maven

```
cp -rf /opt/apache-maven-3.6.3-bin.tar.gz /home/jenkins_home/
```

### 2、进入容器将安装包进行解压

```
docker exec -it jenkins bash
tar -zxvf /var/jenkins_home/apache-maven-3.6.3-bin.tar.gz -C .
mv apache-maven-3.6.3/ /usr/local/maven
```

### 3、配置环境变量及开机source

<pre><code><strong>1、maven环境变量
</strong><strong>vim /etc/profile
</strong>  export M2_HOME=/usr/local/maven
  export PATH=$PATH:$M2_HOME/bin
2、开机source
vi /root/.bashrc
  source /etc/profile
</code></pre>

### 4、退出容器重新进入并且查看maven的版本

```
root@9d6566c29ccb:/# mvn -v 
Error: Could not find or load main class org.codehaus.plexus.classworlds.launcher.Launcher                                  
Caused by: java.lang.ClassNotFoundException: org.codehaus.plexus.classworlds.launcher.Launcher 
```

报错了

错误原因：

由于容器环境并未安装java环境

解决办法：重新部署java环境

```
1、将下载好的java包解压
tar -zxvf /var/jenkins_home/openjdk-11_linux-x64_bin.tar.gz -C /usr/local
2、配置java的环境变量
vim /etc/profile
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    export PATH=$JAVA_HOME/bin:$PATH
3、查看java版本
java -version
    openjdk version "11" 2018-09-25
    OpenJDK Runtime Environment 18.9 (build 11+28)
    OpenJDK 64-Bit Server VM 18.9 (build 11+28, mixed mode)
```

### 5、部署完成后查看maven的版本

<pre><code>mvn -v
<strong> Apache Maven 3.9.9 (8e8579a9e76f7d015ee5ec7bfcdc97d260186937)
</strong> Maven home: /usr/local/maven
 Java version: 17.0.12, vendor: Eclipse Adoptium, runtime: /opt/java/openjdk
 Default locale: en, platform encoding: UTF-8
 OS name: "linux", version: "4.18.0-408.el8.x86_64", arch: "amd64", family: "unix"
</code></pre>

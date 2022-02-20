# Linux安装sonar服务

(注官方说明，从Sonar7.9版本，不再支持Mysql)，以下链接
**End of Life of MySQL Support : SonarQube 7.9 and future versions do not support MySQL.
Please migrate to a supported database. Get more details at **
[https://community.sonarsource.com/t/end-of-life-of-mysql-support](https://community.sonarsource.com/t/end-of-life-of-mysql-support)
[https://jira.sonarsource.com/browse/SONAR-11963](https://jira.sonarsource.com/browse/SONAR-11963)

## **安装步骤**

#### 1.安装sonarqube:

安装的是windows 7.4 community社区版
https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.4.zip

#### 2.选择数据库

```shell
vim sonar.properties
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false
sonar.jdbc.username=root
sonar.jdbc.password=root
sonar.sorceEncoding=UTF-8
```

#### 3. 新增用户

```shell
useradd sonar #新增用户
passwd sonar #设置密码
chmod -R 777 /opt/sonarqube-7.5 #目录授权
chown -R sonar:sonar /opt/sonar-scanner-4.4.0.2170-linux #目录授权指定用户
```



#### 4.测试地址

http://192.xx.xx.xx:9000  默认用户名密码:admin,admin

### sonar-scanner安装配置

#### 1.**配置环境变量**

```shell
SONAR_HOME=/opt/sonarqube-7.5
export SONAR_HOME
SONAR_RUNNER_HOME=/opt/sonar-scanner-4.4.0.2170-linux
PATH=$SONAR_RUNNER_HOME/bin:$PATH
export SONAR_RUNNER_HOME
export JAVA_HOME=/opt/local/jdk1.8.0_221
PATH=$JAVA_HOME/bin:$PATH:$HOME/.local/bin:$HOME/bin
```

#### 2.**在项目根目录下面创建sonar-project.properties配置文件**

```shell
# must be unique in a given SonarQube instance
sonar.projectKey=项目名称
# this is the name displayed in the SonarQube UI
sonar.projectName=项目名称
sonar.projectVersion=1.0
sonar.java.binaries=target/classes
sonar.sources=扫描文件目录
```

#### 3.**执行扫描**

./sonar-scanner

### 插件安装

#### 1.汉化

https://github.com/jbocc/sonar-l10n-zh/releases 

选择合适的sonar版本

#### 2.P3C代码检查（PMD）

https://github.com/rhinoceros/sonar-p3c-pmd/releases/tag/pmd-3.2.0-beta-with-p3c1.3.6-pmd6.10.0

https://github.com/mrprince/sonar-p3c-pmd

### 本地扫描

sonar-scanner.bat -Dsonar.projectKey=yunpei -Dsonar.sources=. -Dsonar.host.url=http://172.16.105.140:9000 -Dsonar.login=6d7406952de15105e550dd6db604e18638edc2ff -Dsonar.java.binaries=.

### IDE扫描

mvn sonar:sonar \ -Dsonar.projectKey=yunpei   \ -Dsonar.host.url=http://172.16.105.140:9000\ -Dsonar.login=6d7406952de15105e550dd6db604e18638edc2ff 

### maven扫描

#### 1.添加插件

直接在根工程的pom文件里添加如下内容

```xml
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.sonarsource.scanner.maven</groupId>
                    <artifactId>sonar-maven-plugin</artifactId>
                    <version>3.5.0.1254</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```

#### 2.添加连接属性配置

在pom.xml文件中添加

```xml
<properties>
    <sonar.login>9129bbeb9b95ea497d14761ffaf55d19a2d63ce3</sonar.login>
    <sonar.password></sonar.password>
    <sonar.host.url>
            http://IP:9000
    </sonar.host.url>
</properties>
```

#### 3.排除指定模块

在pom.xml文件中添加以下事列代码即可，

`在根项目的pom里使用 **sonar.exclusions** 简单配置一下就可以的，但一直不生效，官方也只是指明了该属性的作用，并没有做进一步的说明，也没有给出用例，即使是GitHub的Maven用例也没有出现相关用法，所以就只能自己去尝试。最后参考 StackOverflow 上的回答，并且测试可行的做法是，到对应子模块工程里添加 **sonar.exclusions**属性配置`

```xml
<properties>
<sonar.exclusions>
   src/main/java/com/.../domain/model/**/*,
   src/main/java/com/.../exchange/**/*
</sonar.exclusions>
</properties>
```


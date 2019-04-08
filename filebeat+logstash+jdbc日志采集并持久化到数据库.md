# **filebeat+logstash+jdbc日志采集并持久化到数据库** #


## PowerShell安装 Filebea ##
此处只介绍 Windows10环境的安装，至于其他系统大同小异


###1.注册为 Windows 服务###
将文件解压（或将文件夹放置该目录）

![](https://i.imgur.com/Dnvxruw.jpg)


以 管理员 身份运行 PowerShell。
在 PowerShell 中运行以下命令:

`cd 'C:\Program Files\filebeat-6.3.2-windows-x86_64'`

`C:\Program Files\filebeat-6.3.2-windows-x86_64> .\install-service-filebeat.ps1`

注：
如果此处提示你没有权限，请运行以下的命令注册 Filebeat 服务 ：
`PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-service-filebeat.ps1`

观察PowerShell是否输出如下信息，如果是表示安装成功，此时还未启动，接下来进行配置文件修改。
    
    Status Name DisplayName
    ------ ---- -----------
    Stopped filebeat filebeat
### 2.修改配置文件 ###
在目录里有个配置文件如下（filebeat.yml）

    filebeat.inputs:
    # info ⽇日志，请求修改 "paths" 路路径为本机实际路路径
    - type: log //类型为log
    enabled: true //配置生效true
    paths:
    - C:\java\logOject\*.json //文件路径
    fields:
    level: info //日志级别
    output.logstash:
    hosts: ["172.16.75.203:12044"] //监听端口，等下配置logstash的时候会用到

到这，已经将 Filebeat 成功注册为系统服务，当下一次开机时它会自动启动，当然你也可以手动通过服务控制面板手动启动它。

## 安装并配置logstash ##
安装之前，首先一定要确保 JDK 的版本是 1.8，并配置好相关环境变量。

这里我们使用的版本是logstash-6.5.0(各个版本的配置可能会有所不同)，解压文件放至任意目录如：

    F:\logstash-6.5.0

![](https://i.imgur.com/diNrBAR.jpg)
### 安装logstash-output-jdbc插件 ###
这里默认断网环境的安装
首先做好文件或配置的准备


1.找到以下文件：
> logstash-output-jdbc-master.zip

2.解压至  

    F:\logstash-6.5.0
3.然后在此目录下找到以下文件
> Gemfile（无任何后缀名）

打开并编辑，在末尾最后一行写入：

    gem "logstash-output-jdbc", :path => "logstash-output-jdbc-master"

4.在此目录下打开cmd `F:\logstash-6.5.0\bin`

执行命令：`logstash-plugin install --no-verify`
安装成功后会有相应的提示

5.检查是否安装成功，执行命令l`ogstash-plugin list`可以查看所有的插件列表

至此logstash-output-jdbc插件安装完成。
### 添加依赖的jar包 ###
按以下路径创建文件夹目录

    F:\logstash-6.5.0\logstash-output-jdbc-master\vendor\jar-dependencies\runtime-jars\

将以下文件放入此目录（这是logstash会去默认查找的jdbc驱动路径，也可以在配置文件中指定，稍后会提到）
> mysql-connector-java-8.0.12.jar

此插件还依赖另外一个jar包
> HikariCP-2.7.2.jar

将此文件放入到如下目录：

    F:\logstash-6.5.0\logstash-core\lib\jars
### 创建并编辑配置文件 ###
进入到目录：

    F:\logstash-6.5.0\config
创建并编辑文件：
> myConfig.conf

文件内容如下：

    input{
    	beats { port => "12044" }//输入：在此配置的就是之前filebeat的监听端口号，filebeat会监听文件的改动，并传给logstash
    }
    filter{
        json{
            source => "message"// 过滤器指定你所需要的数据域或过滤不想要的数据
        }
    }
    output{
    	stdout { codec => rubydebug }// 在控制台输出
                jdbc {
                 driver_jar_path => "F:\logstash-6.5.0\logstash-output-jdbc-master\vendor\jar-dependencies\runtime-jars\mysql-connector-java-8.0.12.jar"
                driver_class => "com.mysql.cj.jdbc.Driver"
                connection_string => "jdbc:mysql://localhost:3306/test?user=root&password=123456&useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8"
                statement => [ "INSERT INTO student(age, name) VALUES(?, ?)",  "age","name" ]
                 } 
     }


jdbc配置详解

    driver_jar_path：jdbc驱动路径
    driver_class：指定驱动类//这里我用的是8.0.12的jdbc所以是"com.mysql.cj.jdbc.Driver"
    connection_string：指定数据库连接以及可配参数、用户名、密码等
    statement：sql语句，注意它获取的值是("age","name") 这表示在message中的json，key为"age","name"的值，取值时一定要对应数据库的字段


总之，配置文件包括输入、输出、过滤器 .conf可以很灵活的配置，这里是最简单的配置demo,插入的表也是个很简单的库

## 通过已配置好的软件包在Linux下的部署Logstash##


**找到以下文件，放入/opt目录下**
> logstash-6.5.0.zip

**在此目录下解压该文件**

    unzip logstash-6.5.0.zip

**修改文件夹权限**

    chmod -R 777 logstash-6.5.0


**进入到bin目录，重新安装jdbc插件**

**检查配置文件,修改所需要修改的地方，比如数据库链接等**
> myConfig.conf


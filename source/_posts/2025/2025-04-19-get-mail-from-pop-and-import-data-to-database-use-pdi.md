---
title: 使用 Kettle 实现收取邮件并将数据导入数据库
date: 2025-04-19 16:41:44
updated: 2025-04-19 16:41:44
tags: [Kettle]
---

Kettle 是 [PDI](https://docs.hitachivantara.com/r/en-us/pentaho-data-integration-and-analytics/9.4.x/mk-95pdia003) 的旧称，在这里我们会不加区别地使用它们。我们将模拟一套生产环境用来实现以下需求

1. 使用数据库保存作业和转换；
2. 在 Windows 上编辑作业和转换，在 Linux 上执行作业和转换；
3. 使用 POP 收信下载 Excel 附件；
4. 将 Excel 数据导入数据库；
5. 把 Excel 文件备份到另一个目录。

<!-- more -->

## 使用数据库作为仓库

首先，准备一个 PostgreSQL 数据库实例，创建一个名为 *kettle* 的数据库用来作为数据库仓库。在 Windows 打开 Kettle 应用，点击右上角 *Connect* 按钮。如果已经有了一个或多个仓库这里将显示下拉框，此时选择 *Repository Manager* 选项

{% asset_img repo-1.png %}

在 *Repositories* 对话框点击 *Add* 按钮，在弹出的菜单项中选择 *Database Repository*

{% asset_img repo-2.png %}

在 *Database Repository* 对话框点击 *Create* 按钮

{% asset_img repo-3.png %}

在*数据库连接*对话框选择*一般*，*连接名称*填 *postgresql* 或者你认为合适的名称，*连接类型*选 *PostgreSQL*，*连接方式*选 *Native (JDBC)*，在*设置*里填写数据库连接信息。这一步有两点需要注意，一是要记一下*连接名称*的值，马上就会用到它；二是要注意填写的*数据库名称*，要与前面准备的数据库名称一致

{% asset_img repo-4.png %}

点击*确认*按钮回到 *Database Repository* 对话框，填写 *Display name*，在 *Create* 下拉框选择刚创建的数据库连接 *postgresql*

{% asset_img repo-5.png %}

点击 *Save* 按钮后回到 *Repositories* 对话框，可以看到刚刚创建的 *Database* 仓库

{% asset_img repo-6.png %}

如果是第一次创建数据库仓库，PDI 会在 *kettle* 数据库中初始化一系列的表。可以在这个对话框选中 *Database* 仓库，然后点击右侧的 *Connect* 按钮连接仓库，也可以稍后在主界面右上角点击 *Connect* 下拉框连接 *Database* 仓库

{% asset_img repo-7.png %}

Kettle 默认会创建两个用户：*admin* 和 *guest*，它们的密码分别为 *admin* 和 *guest*

{% asset_img repo-8.png %}

## 在 Linux 安装 PDI

首先在 Linux 上安装 JDK，可以使用 `java -version` 检查 JDK 是否可用，不再赘述。把下载好的 *pdi-ce-9.4.0.0-343.zip* 文件上传到用户主目录 */home/acomma*，执行 `unzip pdi-ce-9.4.0.0-343.zip -d .` 命令将文件解压到当前目录，解压完成后在当前目录可以看到 *data-integration* 子目录

```text
drwxr-xr-x 15 acomma acomma      4096 2022年11月 9日 data-integration
-rw-r--r--  1 acomma acomma 168621798  4月19日 21:29 jdk-11.0.21_linux-x64_bin.tar.gz
-rw-r--r--  1 acomma acomma 385517733  4月19日 21:29 pdi-ce-9.4.0.0-343.zip
```

执行 `./data-integration/kitchen.sh` 命令，如果显示以下信息则表示 PDI 安装成功

```sh
$ ./data-integration/kitchen.sh 
#######################################################################
WARNING:  no libwebkitgtk-1.0 detected, some features will be unavailable
    Consider installing the package with apt-get or yum.
    e.g. 'sudo apt-get install libwebkitgtk-1.0-0'
#######################################################################
Options:
  -rep            = Repository name
  -user           = Repository username
  -trustuser      = !Kitchen.ComdLine.RepUsername!
  -pass           = Repository password
  -job            = The name of the job to launch
  -dir            = The directory (dont forget the leading /)
  -file           = The filename (Job XML) to launch
  -level          = The logging level (Basic, Detailed, Debug, Rowlevel, Error, Minimal, Nothing)
  -logfile        = The logging file to write to
  -listdir        = List the directories in the repository
  -listjobs       = List the jobs in the specified directory
  -listrep        = List the available repositories
  -norep          = Do not log into the repository
  -version        = show the version, revision and build date
  -param          = Set a named parameter <NAME>=<VALUE>. For example -param:FILE=customers.csv
  -listparam      = List information concerning the defined parameters in the specified job.
  -export         = Exports all linked resources of the specified job. The argument is the name of a ZIP file.
  -custom         = Set a custom plugin specific option as a String value in the job using <NAME>=<Value>, for example: -custom:COLOR=Red
  -maxloglines    = The maximum number of log lines that are kept internally by Kettle. Set to 0 to keep all rows (default)
  -maxlogtimeout  = The maximum age (in minutes) of a log line while being kept internally by Kettle. Set to 0 to keep all rows indefinitely (default)
```

这个命令同时会在用户主目录创建 *.kettle* 和 *.pentaho* 两个隐藏目录。为了让 Linux 上的 PDI 连接上数据库仓库，需要将 Windows 上的配置好的 *repositories.xml* 文件上传到 Linux 的 *.kettle* 目录，在 Windows 上这个文件的位置为 *C:\Users\nuc\\.kettle\repositories.xml*，上传好的效果如下所示

```sh
$ ls -l .kettle/
总计 8
-rw-r--r-- 1 acomma acomma  274  4月19日 22:58 kettle.properties
-rw-r--r-- 1 acomma acomma 1763  4月19日 23:14 repositories.xml
```

从 Windows 上传 *repositories.xml* 文件最简单。如果 Linux 有 GUI 也可以执行 `./data-integration/spoon.sh` 命令启动 PDI，然后采用和 Windows 类似的方式创建数据库仓库。另一种方式是手工创建 *repositories.xml* 文件，只是目前还不知道 *repositories > connection > password* 项该如何填写。无论采用哪种方式，最终的目标是让 Windows 和 Linux 使用相同的数据库仓库。目前经过格式化处理后的 *repositories.xml* 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<repositories>
    <connection>
        <name>postgresql</name>
        <server>192.168.1.4</server>
        <type>POSTGRESQL</type>
        <access>Native</access>
        <database>kettle</database>
        <port>5432</port>
        <username>postgres</username>
        <password>Encrypted 2be98afc86aa7f2e4cb79ff228dc6fa8c</password>
        <servername/>
        <data_tablespace/>
        <index_tablespace/>
        <attributes>
            <attribute>
                <code>FORCE_IDENTIFIERS_TO_LOWERCASE</code>
                <attribute>N</attribute>
            </attribute>
            <attribute>
                <code>FORCE_IDENTIFIERS_TO_UPPERCASE</code>
                <attribute>N</attribute>
            </attribute>
            <attribute>
                <code>IS_CLUSTERED</code>
                <attribute>N</attribute>
            </attribute>
            <attribute>
                <code>PORT_NUMBER</code>
                <attribute>5432</attribute>
            </attribute>
            <attribute>
                <code>PRESERVE_RESERVED_WORD_CASE</code>
                <attribute>Y</attribute>
            </attribute>
            <attribute>
                <code>QUOTE_ALL_FIELDS</code>
                <attribute>N</attribute>
            </attribute>
            <attribute>
                <code>SUPPORTS_BOOLEAN_DATA_TYPE</code>
                <attribute>Y</attribute>
            </attribute>
            <attribute>
                <code>SUPPORTS_TIMESTAMP_DATA_TYPE</code>
                <attribute>Y</attribute>
            </attribute>
            <attribute>
                <code>USE_POOLING</code>
                <attribute>N</attribute>
            </attribute>
        </attributes>
    </connection>
    <repository>
        <id>KettleDatabaseRepository</id>
        <name>Database</name>
        <description/>
        <is_default>false</is_default>
        <connection>postgresql</connection>
    </repository>
</repositories>
```

## 在 Linux 开启 Carte 服务

目前我们只是想让 Carte 执行作业和转换，并不是搭建一套完整的 Carte 集群，因此我们只启动一个 Carte 服务。

使用命令 `vi ./data-integration/pwd/carte-config-master-8080.xml` 编辑 *carte-config-master-8080.xml* 文件，修改 *hostname* 为 Linux 服务器的 IP 地址 *192.168.56.101*，其他内容保持不变，修改后的文件内容如下所示

```xml
<slave_config>
  <!-- 
     Document description...
     
     - masters: You can list the slave servers to which this slave has to report back to.
                If this is a master, we will contact the other masters to get a list of all the slaves in the cluster.

     - report_to_masters : send a message to the defined masters to let them know we exist (Y/N)

     - slaveserver : specify the slave server details of this carte instance.
                     IMPORTANT : the username and password specified here are used by the master instances to connect to this slave.

  --> 

  <slaveserver>
    <name>master1</name>
    <hostname>192.168.56.101</hostname>
    <port>8080</port>
    <master>Y</master>
  </slaveserver>


</slave_config>
```

使用命令 `./data-integration/carte.sh pwd/carte-config-master-8080.xml` 启动 CCarte 服务，没什么问题的话将输出如下内容

```text
#######################################################################
WARNING:  no libwebkitgtk-1.0 detected, some features will be unavailable
    Consider installing the package with apt-get or yum.
    e.g. 'sudo apt-get install libwebkitgtk-1.0-0'
#######################################################################
2025/04/20 00:18:52 - Carte - Installing timer to purge stale objects after 1440 minutes.
2025/04/20 00:18:52 - Carte - 创建 web 服务监听器 @ 地址: 192.168.56.101:8080
```

## 添加子服务器

在菜单栏依次选择*工具 > 资源库 > 探索资源库*

{% asset_img carte-1.png %}

在弹出的对话框选择*子服务器*标签页，点击 *+* 按钮

{% asset_img carte-2.png %}

在*子服务器对话框*选择*服务*标签页，填写先前启动的 Carte 服务的信息，*服务名称*随便填；*主机名称或IP地址*填启动 Carte 服务的 Linux 的 IP 地址；*端口号*填 Carte 服务的端口号；*用户名*和*密码*都是 *cluster*；勾选*是否主服务器吗？*，因为 Carte 服务配置了 `<master>Y</master>`

{% asset_img carte-3.png %}

## 新建数据库连接

在 PostgreSQL 创建一个名为 *example* 的数据库用来保存我们的业务数据。在菜单栏依次选择工具 > 资源库 > 探索资源库

{% asset_img carte-1.png %}

在弹出的对话框选择*连接*标签页，点击 *+* 按钮

{% asset_img db-1.png %}

填写数据库连接信息，注意*数据库名称*要与刚刚创建的数据库一致

{% asset_img db-2.png %}

## 开启邮箱客户端协议

我使用网易邮箱作为邮件服务器，因此参考[如何开启客户端协议？](https://help.mail.163.com/faqDetail.do?code=d7a5dc8471cd0c0e8b4b8f4f8e49998b374173cfe9171305fa1ce630d7f67ac2a5feb28b66796d3b)开启邮箱的客户端协议，开通完成后需要记住授权码

{% asset_img mail-1.png %}

同时也要记住邮箱服务器的地址信息，我当前使用的是 *yeah.net* 邮箱，地址信息如下所示

{% asset_img mail-3.png %}

至于端口号信息可以参考网易 163 邮箱

{% asset_img mail-2.png %}

这些信息在配置 PDI 的 *POP 收信*功能时需要用到。

## 准备 Excel 文件和事件表

要导入的 Excel 文件样例如下所示

{% asset_img excel-1.png %}

有三个地方需要注意，一是*开始时间*列是日期格式，而*结束时间*列是文本格式；二是*组织者*列使用了英文 *ORGANIZER*；三是时间是乱序的，而 Excel 表没有标识事件顺序的列。

在 *example* 数据库创建一个事件表 `event_excel` 用来保存 Excel 的原始数据

```sql
create table event_excel
(
    "seq_no"    serial4,
    "事件名称"  varchar,
    "事件类型"  varchar,
    "详细描述"  varchar,
    "开始时间"  varchar,
    "结束时间"  varchar,
    "事件地点"  varchar,
    "ORGANIZER" varchar,
    "优先级"    int2
);
```

有三个地方需要注意，一是 `event_excel` 表的列名和顺序和 Excel 文件一致（也可以不一致，只是会影响后面*文件导入*，因此这里保持一致是有意为之的），在第一列添加了 `seq_no` 列来表示事件在 Excel 文件中的顺序；二是虽然在 Excel 文件中*开始时间*列的类型是日期，但是在 `event_excel` 表中`开始时间`列的类型是 `varchar`；三是在 `event_excel` 表中`优先级`列的类型是 `int2`。

再创建一个事件表 `event` 用来保存原始数据经过转换后符合数据库表设计规范且便于业务方使用的事件数据

```sql
create table event
(
    event_id    serial8,     --事件ID
    event_name  varchar,     --事件名称
    event_type  varchar,     --事件类型
    description varchar,     --详细描述
    start_time  timestamptz, --开始时间
    end_time    timestamptz, --结束时间
    location    varchar,     --事件地点
    organizer   varchar,     --组织者
    priority    int2,        --优先级
    seq_no      int4,        --顺序号
    report_date varchar,     --报告日期
    sync_time   timestamptz, --同步时间
    primary key (event_id)
);
```

## 创建作业

先来看看创建好的作业长什么样子，然后再看看每一步的构成和设置是什么样的

{% asset_img job-1.png %}

### 设置报告日期

{% asset_img job-2.png %}

这里需要注意的是*数据库连接*

{% asset_img job-3.png %}

这里需要注意的是*变量名* `REPORT_DATE`，后面会多次使用它

{% asset_img job-4.png %}

### POP 收信

从前面我们知道 *yeah.net* 邮箱的 IMAP 地址是 *imap.yeah.net*，但是我的虚拟电脑暂时无法解析域名，所以*源主机*填写的是 IP 地址；*密码*是先前生成的授权码；*附件通配符*填写的是*每日事件信息${REPORT_DATE}.xlsx*，其中 `${REPORT_DATE}` 作为变量会被替换

{% asset_img job-5.png %}

*IMAP文件夹*可以通过右边的*选择一个文件选择*来选择，几乎所有邮箱服务器都有 *INBOX* 文件夹；*收取邮件*选择的是*获取未读邮件*

{% asset_img job-6.png %}

*主题*填写的是*每日事件报告*

{% asset_img job-7.png %}

通过这些配置我们实现了收取主题中包含*每日事件报告*的未读邮件，下载附件中模式为*每日事件信息${REPORT_DATE}.xlsx* 的 Excel 附件，并把附件保存到 */home/acomma/excel* 目录。

### 检查是否正确下载附件

注意*文件名*的值，我们添加了一个变量 `${REPORT_DATE}`，这个变量在运行时会被替换，即我们只处理这个日期的文件

{% asset_img job-8.png %}

### 清空表

{% asset_img job-9.png %}

### 文件导入

{% asset_img job-10.png %}

注意*选中的文件*表格里的值，我们添加了一个变量 `${REPORT_DATE}`，这个变量在运行时会被替换，即我们只处理这个日期的文件

{% asset_img job-11.png %}

注意工作表的名称

{% asset_img job-12.png %}

这里要注意两个地方，一是*去除空格类型*由于 PDI 的 BUG 是无法选择*去除两边空格*的；二是*开始时间*的格式 *MM-dd-yy*，这是为了测试目的特殊选择的

{% asset_img job-13.png %}

为了解决无法选择*去除两边空格*问题，我们需要在一个 *File Repository* 中创建一个转换，将 *Microsoft Excel input* 这一项拷贝这个转换里，然后在文件夹中找到这个转换，用文本编辑器打开这个转换，将 *none* 值修改为 *both*，参考 [kettle输入“去除空格类型”设置不上的办法](https://blog.csdn.net/sptoor/article/details/24427627)

{% asset_img job-14.png %}

这里要注意*数据库字段*标签页下 *"ORGANIZER"* 字段添加了英文双引号

{% asset_img job-15.png %}

### 备份文件

如果备份目录存在同名文件则删除

{% asset_img job-16.png %}

将文件移动到备份目录

{% asset_img job-17.png %}

### 转移数据

{% asset_img job-18.png %}

## 添加运行配置

选中作业下的 *Run Configuration*，在右键菜单中点击 *New*

{% asset_img job-19.png %}

在弹出的对话框中 *Name* 随便填写，这里填写的名称和*子服务器*名称一样 *vbox-8080*，在 *Settings* 选中 *Slave server*，在 *Location* 选择*子服务器*

{% asset_img job-20.png %}

## 运行作业

在 *Run Configuration* 处选择刚刚添加的运行配置

{% asset_img job-21.png %}

## 运行结果

进入 `event_excel` 表的内容如下所示

{% asset_img result-1.png %}

进入 `event` 表的内容如下所示

{% asset_img result-2.png %}

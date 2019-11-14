# Linux 进程管理工具SystemD

# 一、简介

Linux内核加载启动后，用户空间的第一个进程就是初始化进程，这个程序的物理文件约定位于/sbin/init，当然也可以通过传递内核参数来让内核启动指定的程序。这个进程的特点是进程号为1，代表第一个运行的用户空间进程。不同发行版采用了不同的启动程序，主要有以下几种主流选择：

- 以Ubuntu为代表的Linux发行版采用upstart。

- 以7.0版本之前的CentOS为代表的System V init。

- CentOS 7.0版本开始的Systemd。

历史上，[Linux 的启动](http://www.ruanyifeng.com/blog/2013/08/linux_boot_process.html)一直采用[init](https://en.wikipedia.org/wiki/Init)进程。这种方法有两个缺点

1. 启动时间长。init进程是串行启动，只有前一个进程启动完，才会启动下一个进程。
2. 启动脚本复杂。init进程只是执行启动脚本，不管其他事情。脚本需要自己处理各种情况，这往往使得脚本变得很长。

作为系统初始化系统，systemd 的最大特点有两个：

- 令人惊奇的激进的并发启动能力，极大地提高了系统启动速度；
- 用 CGroup 统计跟踪子进程，干净可靠。

此外，和其前任不同的地方在于:

- systemd 已经不仅仅是一个初始化系统了

- Systemd 出色地替代了 sysvinit 的所有功能。因为 init 进程是系统所有进程的父进程这样的特殊性，systemd 非常适合提供曾经由其他服务提供的功能，比如定时任务 (以前由 crond 完成) ；会话管理 (以前由 ConsoleKit/PolKit 等管理) 

- 有助于标准化 Linux 的管理！如果所有的 Linux 发行版都采纳了 systemd，那么系统管理任务便可以很大程度上实现标准化
- 此外 systemd 有个很棒的承诺：接口保持稳定，不会再轻易改动

- 根据 Linux 惯例，字母d是守护进程（daemon）的缩写。 Systemd 这个名字的含义，就是它要守护整个系统

- systmed是一个用户空间的程序，属于应用程序，不属于Linux内核范畴，Linux内核的主要特征在所有发行版中是统一的，厂商可以自由改变的是用户空间的应用程序


# 二、基本概念

## 1. Unit单元

系统初始化要做很多工作，如挂在文件系统，启动sshd服务，配置交换分区，这都可以看做是一个配置单元，Systemd 可以管理所有系统资源。不同的资源统称为 Unit（单位）。systemd把配置单元分成分成12种

- Service unit：系统服务
- Target unit：多个 Unit 构成的一个逻辑分组，可以当成是SystemV中的运行级。
- Device Unit：硬件设备
- Mount Unit：文件系统的挂载点，systemd据此进行自动挂载，为了与SystemV兼容，目前systemd自动处理/etc/fstab并转化为mount
- Automount Unit：自动挂载点
- Path Unit：文件或路径
- Scope Unit：不是由 Systemd 启动的外部进程
- Slice Unit：进程组
- Snapshot Unit：Systemd 快照，可以切回某个快照
- Socket Unit：进程间通信的 socket
- Swap Unit：配置swap交换分区文件
- Timer Unit：定时器。用来定时触发用户定义的操作，它可以用来取代传统的atd，crond等。

每一个配置单元都有一个对应的配置文件，系统管理员的任务就是编写和维护这写不同的配置文件，比如一个MySql服务对应一个mysql.service文件。

## 2. 依赖关系

systemd并不能完全解除各个单元之间的依赖关系，如物理设备单元准备就绪之前，不可能执行挂载单元。为此需要定义各个单元之间的依赖关系。有依赖的地方就会有出现死循环的可能，比如A依赖于B，B依赖于C，C依赖于A，那么导致死锁。systemd为此提供了两种不同程度的依赖关系，一个是require，一个是want，出现死循环时，systemd会尝试忽略want类型的依赖，如仍不能解锁，那么systemd报错。

## 3. Target和runlevel

systemd使用target取代了systemV的运行级的概念

Sysvinit 运行级别和 systemd 目标的对应表

| Sysvinit 运行级别 |                     Systemd 目标                      |                            备注                             |
| :---------------: | :---------------------------------------------------: | :---------------------------------------------------------: |
|         0         |           runlevel0.target, poweroff.target           |                         关闭系统。                          |
|   1, s, single    |            runlevel1.target, rescue.target            |                        单用户模式。                         |
|       2, 4        | runlevel2.target, runlevel4.target, multi-user.target |           用户定义/域特定运行级别。默认等同于 3。           |
|         3         |          runlevel3.target, multi-user.target          |    多用户，非图形化。用户可以通过多个控制台或网络登录。     |
|         5         |          runlevel5.target, graphical.target           | 多用户，图形化。通常为所有运行级别 3 的服务外加图形化登录。 |
|         6         |            runlevel6.target, reboot.target            |                            重启                             |
|     emergency     |                   emergency.target                    |                         紧急 Shell                          |

# 三、Systemd包含的命令

Systemd 是一个完整的软件包，Systemd 并不是一个命令，而是一组命令，涉及到系统管理的方方面面。安装完成后有很多物理文件组成，大致分布为，配置文件位于/etc/systemd这个目录下，配置工具命令位于/bin，和/sbin这两个目录下，预先准备的备用配置文件位于/lib/systemd目录下，还有库文件和帮助手册等等。这是一个庞大的软件包。详情使用rpm -ql systemd即可查看。

## 1. systemctl

Systemd 的主命令，用于管理系统

```bash
# 重启系统
$ systemctl reboot
# 关闭系统，切断电源
$ systemctl poweroff
# CPU停止工作
$ systemctl halt
# 暂停系统
$ systemctl suspend
# 让系统进入冬眠状态
$ systemctl hibernate
# 让系统进入交互式休眠状态
$ systemctl hybrid-sleep
# 启动进入救援状态（单用户状态）
$ systemctl rescue
```

## 2. systemd-analyze

systemd-analyze命令用于查看启动耗时

```json
# 查看启动耗时
$ systemd-analyze
# 查看每个服务的启动耗时
$ systemd-analyze blame
# 显示瀑布状的启动过程流
$ systemd-analyze critical-chain
# 显示指定服务的启动流
$ systemd-analyze critical-chain atd.service
```

## 3. hostnamectl

hostnamectl命令用于查看当前主机的信息。

```json
# 显示当前主机的信息
$ hostnamectl
# 设置主机名。
$ hostnamectl set-hostname rhel7
```

## 4. localectl

localectl命令用于查看本地化设置。

```json
# 查看本地化设置
$ localectl
# 设置本地化参数。
$ localectl set-locale LANG=en_GB.utf8
$ localectl set-keymap en_GB
```

## 5. timedatectl

timedatectl命令用于查看当前时区设置。

```json
# 查看当前时区设置
$ timedatectl
# 显示所有可用的时区
$ timedatectl list-timezones
# 设置当前时区
$ timedatectl set-timezone America/New_York
$ timedatectl set-time YYYY-MM-DD
$ timedatectl set-time HH:MM:SS
```

## 6. loginctl

loginctl命令用于查看当前登录的用户。

```json
# 列出当前session
$ loginctl list-sessions
# 列出当前登录用户
$ loginctl list-users
# 列出显示指定用户的信息
$ loginctl show-user ruanyf
```



# 三、Unit

- 每一个 Unit 都有一个配置文件，告诉 Systemd 怎么启动这个 Unit 。

- Systemd 默认从目录`/etc/systemd/system/`读取配置文件。但是，里面存放的大部分文件都是符号链接，指向目录`/usr/lib/systemd/system/`，真正的配置文件存放在那个目录。

- 配置文件的后缀名，就是该 Unit 的种类，比如sshd.socket。如果省略，Systemd 默认后缀名为.service，所以sshd会被理解成sshd.service。

## 1. Unit配置文件结构

配置文件分成几个区块。每个区块的第一行，是用方括号表示的区别名，比如[Unit]。注意：

- 配置文件的区块名和字段名，都是大小写敏感的。

- 每个区块内部是一些等号连接的键值对，键值对的等号两侧不能有空格。

```pro
[Unit]
Description=ATD daemon

[Service]
Type=forking
ExecStart=/usr/bin/atd

[Install]
WantedBy=multi-user.target
```

Unit 配置文件的完整字段清单，请参考[官方文档](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)。

### [Unit]区块

[Unit]区块通常是配置文件的第一个区块。用来定义 Unit 的元数据，以及配置与其他 Unit 的关系。它的主要字段如下。

 - Description：简短描述
 - Documentation：文档地址
 - Requires：当前 Unit 依赖的其他 Unit，如果它们没有运行，当前 Unit 会启动失败
 - Wants：与当前 Unit 配合的其他 Unit，如果它们没有运行，当前 Unit 不会启动失败
 - BindsTo：与Requires类似，它指定的 Unit 如果退出，会导致当前 Unit 停止运行
 - Before：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之后启动
 - After：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之前启动
 - Conflicts：这里指定的 Unit 不能与当前 Unit 同时运行
 - Condition...：当前 Unit 运行必须满足的条件，否则不会运行
 - Assert...：当前 Unit 运行必须满足的条件，否则会报启动失败

### [Service]区块

[Service]区块用来定义如何启动当前服务，只有 Service 类型的 Unit 才有这个区块。它的主要字段如下。

 - Type：定义启动时的进程行为。它有以下几种值。
 - Type=simple：默认值，执行ExecStart指定的命令，启动主进程
 - Type=forking：以 fork 方式从父进程创建子进程，创建后父进程会立即退出
 - Type=oneshot：一次性进程，Systemd 会等当前服务退出，再继续往下执行
 - Type=dbus：当前服务通过D-Bus启动
 - Type=notify：当前服务启动完毕，会通知Systemd，再继续往下执行
 - Type=idle：若有其他任务执行完毕，当前服务才会运行
 - ExecStart：启动当前服务的命令
 - ExecStartPre：启动当前服务之前执行的命令
 - ExecStartPost：启动当前服务之后执行的命令
 - ExecReload：重启当前服务时执行的命令
 - ExecStop：停止当前服务时执行的命令
 - ExecStopPost：停止当其服务之后执行的命令
 - RestartSec：自动重启当前服务间隔的秒数
 - Restart：定义何种情况 Systemd 会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog
 - TimeoutSec：定义 Systemd 停止当前服务之前等待的秒数
 - Environment：指定环境变量

### [Install]区块

[Install]通常是配置文件的最后一个区块，用来定义如何启动，以及是否开机启动。它的主要字段如下。

 - WantedBy：它的值是一个或多个 Target，当前 Unit 激活时（enable）符号链接会放入/etc/systemd/system目录下面以 Target 名 +.wants后缀构成的子目录中
 - RequiredBy：它的值是一个或多个 Target，当前 Unit 激活时，符号链接会放入/etc/systemd/system目录下面以 Target 名 + .required后缀构成的子目录中
 - Alias：当前 Unit 可用于启动的别名
 - Also：当前 Unit 激活（enable）时，会被同时激活的其他 Unit

## 2. Unit管理

### 3.2.1 systemctl enable

命令用于在上面两个目录之间，建立符号链接关系。如果配置文件里面设置了开机启动，systemctl enable命令相当于激活开机启动。

```bash
$ systemctl enable clamd@scan.service 
# 等同于
$ ln -s '/usr/lib/systemd/system/clamd@scan.service' '/etc/systemd/system/multi-user.target.wants/clamd@scan.service'
```

### 3.2.2 systemctl disable

用于在两个目录之间，撤销符号链接关系，相当于撤销开机启动。

```bash
systemctl disable clamd@scan.service
```

### 3.2.3 systemctl list-unit-files

systemctl list-unit-files会显示每个配置文件的状态。而每个配置文件的状态，一共有四种

- **enabled**：已建立启动链接
- **disabled**：没建立启动链接
- **static**：该配置文件没有[Install]部分（无法执行），只能作为其他配置文件的依赖
- **masked**：该配置文件被禁止建立启动链接

```bash
# 列出所有配置文件 
$ systemctl list-unit-files

# 列出指定类型的配置文件
$ systemctl list-unit-files --type=service
```

### 3.2.4 systemctl list-units

查看当前系统的所有 Unit 

```bash
# 列出正在运行的 Unit
$ systemctl list-units

# 列出所有Unit，包括没有找到配置文件的或者启动失败的
$ systemctl list-units --all

# 列出所有没有运行的 Unit
$ systemctl list-units --all --state=inactive

# 列出所有加载失败的 Unit
$ systemctl list-units --failed

# 列出所有正在运行的、类型为 service 的 Unit
$ systemctl list-units --type=service
```

### 3.2.5 systemctl status

systemctl status命令用于查看系统状态和单个 Unit 的状态

```json
# 显示系统状态 
$ systemctl status 

# 显示单个 Unit 的状态 
$ sysystemctl status httpd
httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
   Active: active (running) since 金 2014-12-05 12:18:22 JST; 7min ago
 Main PID: 4349 (httpd)
   Status: "Total requests: 1; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─4349 /usr/sbin/httpd -DFOREGROUND
           ├─4350 /usr/sbin/httpd -DFOREGROUND
           ├─4351 /usr/sbin/httpd -DFOREGROUND
           ├─4352 /usr/sbin/httpd -DFOREGROUND
           ├─4353 /usr/sbin/httpd -DFOREGROUND
           └─4354 /usr/sbin/httpd -DFOREGROUND

12月 05 12:18:22 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
12月 05 12:18:22 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
12月 05 12:22:40 localhost.localdomain systemd[1]: Started The Apache HTTP Server.



# 显示远程主机的某个 Unit 的状态 
$ systemctl -H root@rhel7.example.com status httpd.service
```

### 3.2.6 查询Unit状态

```bash
# 显示某个 Unit 是否正在运行
$ systemctl is-active application.service

# 显示某个 Unit 是否处于启动失败状态
$ systemctl is-failed application.service

# 显示某个 Unit 服务是否建立了启动链接
$ systemctl is-enabled application.service
```

### 3.2.7 Unit状态管理

```bash
# 立即启动一个服务
$ systemctl start apache.service

# 立即停止一个服务
$ systemctl stop apache.service

# 重启一个服务
$ systemctl restart apache.service

# 杀死一个服务的所有子进程
$ systemctl kill apache.service

# 重新加载一个服务的配置文件
$ systemctl reload apache.service

# 重载所有修改过的配置文件
$ systemctl daemon-reload

# 显示某个 Unit 的所有底层参数
$ systemctl show httpd.service

# 显示某个 Unit 的指定属性的值
$ systemctl show -p CPUShares httpd.service

# 设置某个 Unit 的指定属性
$ systemctl set-property httpd.service CPUShares=500
```

### 3.2.8 查询Unit间的依赖关系

Unit 之间存在依赖关系：A 依赖于 B，就意味着 Systemd 在启动 A 的时候，同时会去启动 B。

```bash
# 列出一个 Unit 的所有依赖。
$ systemctl list-dependencies nginx.service
# 上面命令的输出结果之中，有些依赖是 Target 类型（详见下文），默认不会展开显示。如果要展开 Target，就需要使用--all参数。
$ systemctl list-dependencies --all nginx.service
```

# 四、Unit的日志管理

Systemd 统一管理所有 Unit 的启动日志。带来的好处就是，可以只用journalctl一个命令，查看所有日志（内核日志和应用日志）。日志的配置文件是/etc/systemd/journald.conf。journalctl功能强大，用法非常多。

```bash
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ journalctl

# 查看内核日志（不显示应用日志）
$ journalctl -k

# 查看系统本次启动的日志
$ journalctl -b
$ journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ journalctl -b -1

# 查看指定时间的日志
$ journalctl --since="2012-10-30 18:17:16"
$ journalctl --since "20 min ago"
$ journalctl --since yesterday
$ journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ journalctl -n

# 显示尾部指定行数的日志
$ journalctl -n 20

# 实时滚动显示最新日志
$ journalctl -f

# 查看指定服务的日志
$ journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
$ journalctl _PID=1

# 查看某个路径的脚本的日志
$ journalctl /usr/bin/bash

# 查看指定用户的日志
$ journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ journalctl -u nginx.service
$ journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ journalctl --no-pager

# 以 JSON 格式（单行）输出
$ journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ journalctl -b -u nginx.serviceqq -o json-pretty

# 显示日志占据的硬盘空间
$ journalctl --disk-usage

# 指定日志文件占据的最大空间
$ journalctl --vacuum-size=1G

# 指定日志文件保存多久
$ journalctl --vacuum-time=1years
```

# 五、Target

启动计算机的时候，需要启动大量的 Unit。如果每一次启动，都要一一写明本次启动需要哪些 Unit，显然非常不方便。Systemd 的解决方案就是 Target。简单说，Target 就是一个 Unit 组，包含许多相关的 Unit 。启动某个 Target 的时候，Systemd 就会启动里面所有的 Unit。从这个意义上说，Target 这个概念类似于"状态点"，启动某个 Target 就好比启动到某种状态。

传统的init启动模式里面，有 RunLevel 的概念，跟 Target 的作用很类似。不同的是，RunLevel 是互斥的，不可能多个 RunLevel 同时启动，但是多个 Target 可以同时启动。

## 1. 与传统 RunLevel 的对应关系如下:

| Traditional runlevel | New target name  -->  Symbolically linked to |
| :------------------- | :------------------------------------------- |
| Runlevel 0           | runlevel0.target -> poweroff.target          |
| Runlevel 1           | runlevel1.target -> rescue.target            |
| Runlevel 2           | runlevel2.target -> multi-user.target        |
| Runlevel 3           | runlevel3.target -> multi-user.target        |
| Runlevel 4           | runlevel4.target -> multi-user.target        |
| Runlevel 5           | runlevel5.target -> graphical.target         |
| Runlevel 6           | runlevel6.target -> reboot.target            |

## 2. 与init进程的主要差别如下:

- 默认的 RunLevel（在/etc/inittab文件设置）现在被默认的 Target 取代，位置是/etc/systemd/system/default.target，通常符号链接到graphical.target（图形界面）或者multi-user.target（多用户命令行）。
- 启动脚本的位置，以前是/etc/init.d目录，符号链接到不同的 RunLevel 目录 （比如/etc/rc3.d、/etc/rc5.d等），现在则存放在/lib/systemd/system和/etc/systemd/system目录。

## 3. Target管理

```bash
# 查看当前系统的所有 Target
$ systemctl list-unit-files --type=target

# 查看一个 Target 包含的所有 Unit
$ systemctl list-dependencies multi-user.target

# 查看启动时的默认 Target
$ systemctl get-default

# 设置启动时的默认 Target
$ sudo systemctl set-default multi-user.target

# 切换 Target 时，默认不关闭前一个 Target 启动的进程，
# systemctl isolate 命令改变这种行为，
# 关闭前一个 Target 里面所有不属于后一个 Target 的进程
$ systemctl isolate multi-user.target
```

# 六、常见服务的Unit配置

## 1. Zookeeper

```bash
bash -c ' cat > /usr/lib/systemd/system/zookeeper.service <<EOF
[Unit]
Description=Zookeeper server daemon
Before=syslog.target network.target

[Service]
Type=forking
Environment=ZOO_LOG_DIR=/data/zookeeper/logs
ExecStart=/opt/zookeeper/bin/zkServer.sh start
ExecStop=/opt/zookeeper/bin/zkServer.sh stop
PIDFILE=/data/zookeeper/data/zookeeper.pid

[Install]
WantedBy=multi-user.target
EOF' ;\
  systemctl daemon-reload ;\
  systemctl enable zookeeper ;\
  systemctl start zookeeper ;\
  systemctl status zookeeper.service  -l ;\
  zkCli.sh
```

## 2. Kafka

```bash
bash -c ' cat > /usr/lib/systemd/system/kafka.service <<EOF
[Unit]
Description=Kafka server daemon
Wants=zookeeper.service
After=zookeeper.service

[Service]
Type=forking
ExecStart=/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

[Install]
WantedBy=multi-user.target
EOF' ;\
  systemctl daemon-reload ;\
  systemctl enable kafka ;\
  systemctl start kafka ;\
  systemctl status kafka.service  -l ;\
  jps -l
```

## 3. Tomcat

```bash
bash -c ' cat > /usr/lib/systemd/system/tomcat.service <<EOF
[unit]
Description=Tomcat
After=network.target

[Service]
Type=forking
PIDFile=/opt/tomcat/tomcat.pid
ExecStart=/opt/tomcat/bin/catalina.sh start
ExecReload=/opt/tomcat/bin/catalina.sh restart
ExecStop=/opt/tomcat/bin/catalina.sh stop

[Install]
WantedBy=multi-user.target' && \
	sed -i '1a CATALINA_PID=/opt/tomcat/tomcat.pid' /opt/tomcat/bin/catalina.sh && \
	systemctl daemon-reload && \
	systemctl enable tomcat && \
	systemctl start tomcat && \
	systemctl status tomcat
```



# 参考

1. http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html

2. http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html


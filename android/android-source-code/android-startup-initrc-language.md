# Android启动之initrc语法

> 译自system/core/init/readme.txt

## Android init.rc语法

Android init.rc包含四种类型的声明，分别是 **Actions**，**Commands**，**Services**，和 **Options**。 其语法规则如下：

* init.rc所有的语句都是以行为单位，行间使用空格分隔符号，类似C语言，使用""续行，类似shell脚本以"#"进行行注释。
* Actions和Services默认声明一个新的section，所有该section下的command与options都属于它，在section前申明的command和options将会被其忽略。
* Actions与Services的命名是唯一的,重复的定义被视为无效。

### Actions

Action是由一系列命令组成并由"on"开头的语句,通过trigger来决定什么时候执行("on"后面跟的字符串即为ation的trigger).当trigger事件发生时，如果action不在to-be-executed中，将被加入到to-be-executed的队尾等待执行。

在to-be-executed队列中的每个action按FIFO顺序执行，同样action中的command也是按定义的先后顺序依次执行。init除了执行action的command，还负责设备节点的创建及销毁，属性设置，进程重启等工作。

Actions的格式如下：

```
on <trigger>
   <command>
   <command>
   <command>
```

### Services

services,以“service”字符串开头，由init进程启动并且可以在退出时重启(可选)，services的格式如下：

```
service <name> <pathname> [ <argument> ]*
   <option>
   <option>
   ...
```

### Options

options是services的可选项，它决定service的执行行为.

* critical 表示关键的service，如果service 4分钟内异常退出4次将会重启进入recovery模式。
* disabled 指定disable的service，不能通过class service(class\_start)启动，只有通过`start service_name`启动时才能启动
* setenv <\<name> \<value> service启动时设置环境变量值
* socket \<name> \<type> \<perm> \[ \<user> \[ \<group> \[ \<seclabel> ] ] ] 为要启动的进程创建名为/dev/socket/\<name>的socket,\<type>可选"dgram"或"stream",\<user> \[ \<group>默认为0.
* user \<username> 在service启动时为其设置user，默认为root
* group \<groupname> \[ \<groupname> ]\* 在service启动时为其设置group，默认为root
* seclabel \<seclabel> SELinux security context
* oneshot 定义oneshot的service，退出后不再重启
* class \<name> 指服务所属的class，同一class的服务可以同时启动或停止.默认为default
* onrestart 服务重启时执行命令

### Triggers

Triggers是on后面的字符串，匹配某类事件的发生来执行action。

* boot init执行后触发的第一个trigger.
* \<name>=\<value> 设置属性值时触发Trigger

### Commands

Commands即Action或Services onrestart中所执行的命令,下面是常用的命令。

* bootchart\_init 启动bootcharting
* chmod \<octal-mode> \<path> chmod权限
* chown \<owner> \<group> \<path> chown用户/组
* class\_start \<serviceclass> 启动serviceclass下所有未启动的服务服务,disable的除外
* class\_stop \<serviceclass> 停止serviceclass下所有已启动的服务服务,disable的除外
* class\_reset \<serviceclass> Stop all services of the specified class if they are currently running, without disabling them. They can be restarted later using class\_start.
* copy \<src> \<dst> copy操作
* domainname \<name> 设置domain name.
* enable \<servicename> enable服务，该服务可以随class service启动或停止
* exec \[ \<seclabel> \[\<user> \[ \<group> ]\* ] ] -- \<command> \[ \<argument> ]\* fork并执行command，将阻塞init直到执行完毕
* export \<name> \<value> export环境变量值
* hostname \<name> 设置hostname.
* ifup \<interface> 启用网络接口
* import \<filename> import其他initrc文件
* insmod \<path> 加载指定驱动模块
* load\_all\_props Loads properties from /system, /vendor, et cetera. This is included in the default init.rc.
* load\_persist\_props Loads persistent properties when /data has been decrypted. This is included in the default init.rc.
* loglevel \<level> 设置log级别
* mkdir \<path> \[mode] \[owner] \[group] 创建dir
* mount\_all \<fstab> mount所有fstab.
* mount \<type> \<device> \<dir> \[ \<flag> ]\* \[\<options>] mount文件系统到执行目录
* powerctl Internal implementation detail used to respond to changes to the "sys.powerctl" system property, used to implement rebooting.
* restart \<service> restart服务
* restorecon \<path> \[ \<path> ]\* 重新存储\<path>到由file\_contexts配置的安全上下文.
* restorecon\_recursive \<path> \[ \<path> ]\* Recursively restore the directory tree named by \<path> to the security contexts specified in the file\_contexts configuration.
* rm \<path> rm文件
* rmdir \<path> rmdir
* setprop \<name> \<value> 设置属性
* setrlimit \<resource> \<cur> \<max> 设置进程占用的资源大小，包括可打开文件数目，大小
* start \<service> 启动服务
* stop \<service> 停止服务
* swapon\_all \<fstab> Calls fs\_mgr\_swapon\_all on the given fstab file.
* symlink \<target> \<path> 创建连接到target的path符号链接
* sysclktz \<mins\_west\_of\_gmt> Set the system clock base (0 if system clock ticks in GMT)
* trigger \<event> trigger的事件
* verity\_load\_state Internal implementation detail used to load dm-verity state.
* verity\_update\_state \<mount\_point> Internal implementation detail used to update dm-verity state and set the partition.\<mount\_point>.verified properties used by adb remount because fs\_mgr can't set them directly itself.
* wait \<path> \[ \<timeout> ] Poll文件，默认timeout为5s
* write \<path> \<content> 打开指定文件并写入字符串

### Properties

Init进程运行过程中,以下的属性值会随时更新(可通过adb shell getprop查看)：

* init.action 当前执行的action
* init.command 将被执行的命令
* init.svc.\<name> 服务的运行状态("stopped", "running", "restarting")

## Android Init Language

The Android Init Language consists of four broad classes of statements, which are Actions, Commands, Services, and Options.

All of these are line-oriented, consisting of tokens separated by whitespace. The c-style backslash escapes may be used to insert whitespace into a token. Double quotes may also be used to prevent whitespace from breaking text into multiple tokens. The backslash, when it is the last character on a line, may be used for line-folding.

Lines which start with a # (leading whitespace allowed) are comments.

Actions and Services implicitly declare a new section. All commands or options belong to the section most recently declared. Commands or options before the first section are ignored.

Actions and Services have unique names. If a second Action or Service is declared with the same name as an existing one, it is ignored as an error. (??? should we override instead)

### Actions

Actions are named sequences of commands. Actions have a trigger which is used to determine when the action should occur. When an event occurs which matches an action's trigger, that action is added to the tail of a to-be-executed queue (unless it is already on the queue).

Each action in the queue is dequeued in sequence and each command in that action is executed in sequence. Init handles other activities (device creation/destruction, property setting, process restarting) "between" the execution of the commands in activities.

Actions take the form of:

```
on <trigger>
   <command>
   <command>
   <command>
```

### Services

Services are programs which init launches and (optionally) restarts when they exit. Services take the form of:

```
service <name> <pathname> [ <argument> ]*
   <option>
   <option>
   ...
```

### Options

Options are modifiers to services. They affect how and when init runs the service.

critical This is a device-critical service. If it exits more than four times in four minutes, the device will reboot into recovery mode.

disabled This service will not automatically start with its class. It must be explicitly started by name.

setenv \<name> \<value> Set the environment variable \<name> to \<value> in the launched process.

socket \<name> \<type> \<perm> \[ \<user> \[ \<group> \[ \<seclabel> ] ] ] Create a unix domain socket named /dev/socket/\<name> and pass its fd to the launched process. \<type> must be "dgram", "stream" or "seqpacket". User and group default to 0. 'seclabel' is the SELinux security context for the socket. It defaults to the service security context, as specified by seclabel or computed based on the service executable file security context.

user \<username> Change to username before exec'ing this service. Currently defaults to root. (??? probably should default to nobody) Currently, if your process requires linux capabilities then you cannot use this command. You must instead request the capabilities in-process while still root, and then drop to your desired uid.

group \<groupname> \[ \<groupname> ]\* Change to groupname before exec'ing this service. Additional groupnames beyond the (required) first one are used to set the supplemental groups of the process (via setgroups()). Currently defaults to root. (??? probably should default to nobody)

seclabel \<seclabel> Change to 'seclabel' before exec'ing this service. Primarily for use by services run from the rootfs, e.g. ueventd, adbd. Services on the system partition can instead use policy-defined transitions based on their file security context. If not specified and no transition is defined in policy, defaults to the init context.

oneshot Do not restart the service when it exits.

class \<name> Specify a class name for the service. All services in a named class may be started or stopped together. A service is in the class "default" if one is not specified via the class option.

onrestart Execute a Command (see below) when service restarts.

writepid \<file...> Write the child's pid to the given files when it forks. Meant for cgroup/cpuset usage.

### Triggers

Triggers are strings which can be used to match certain kinds of events and used to cause an action to occur.

boot This is the first trigger that will occur when init starts (after /init.conf is loaded)

\<name>=\<value> Triggers of this form occur when the property \<name> is set to the specific value \<value>.

&#x20;One can also test multiple properties to execute a group of commands. For example:

&#x20;on property:test.a=1 && property:test.b=1 setprop test.c 1

&#x20;The above stub sets test.c to 1 only when both test.a=1 and test.b=1

### Commands

bootchart\_init Start bootcharting if configured (see below). This is included in the default init.rc.

chmod \<octal-mode> \<path> Change file access permissions.

chown \<owner> \<group> \<path> Change file owner and group.

class\_start \<serviceclass> Start all services of the specified class if they are not already running.

class\_stop \<serviceclass> Stop and disable all services of the specified class if they are currently running.

class\_reset \<serviceclass> Stop all services of the specified class if they are currently running, without disabling them. They can be restarted later using class\_start.

copy \<src> \<dst> Copies a file. Similar to write, but useful for binary/large amounts of data.

domainname \<name> Set the domain name.

enable \<servicename> Turns a disabled service into an enabled one as if the service did not specify disabled. If the service is supposed to be running, it will be started now. Typically used when the bootloader sets a variable that indicates a specific service should be started when needed. E.g.

```
    on property:ro.boot.myfancyhardware=1
      enable my_fancy_service_for_my_fancy_hardware
```

exec \[ \<seclabel> \[ \<user> \[ \<group> ]\* ] ] -- \<command> \[ \<argument> ]\* Fork and execute command with the given arguments. The command starts after "--" so that an optional security context, user, and supplementary groups can be provided. No other commands will be run until this one finishes. \<seclabel> can be a - to denote default.

export \<name> \<value> Set the environment variable \<name> equal to \<value> in the global environment (which will be inherited by all processes started after this command is executed)

hostname \<name> Set the host name.

ifup \<interface> Bring the network interface \<interface> online.

import \<filename> Parse an init config file, extending the current configuration.

insmod \<path> Install the module at \<path>

load\_all\_props Loads properties from /system, /vendor, et cetera. This is included in the default init.rc.

load\_persist\_props Loads persistent properties when /data has been decrypted. This is included in the default init.rc.

loglevel \<level> Sets the kernel log level to level. Properties are expanded within \<level>.

mkdir \<path> \[mode] \[owner] \[group] Create a directory at \<path>, optionally with the given mode, owner, and group. If not provided, the directory is created with permissions 755 and owned by the root user and root group. If provided, the mode, owner and group will be updated if the directory exists already.

mount\_all \<fstab> Calls fs\_mgr\_mount\_all on the given fs\_mgr-format fstab.

mount \<type> \<device> \<dir> \[ \<flag> ]\* \[\<options>] Attempt to mount the named device at the directory \<dir> \<device> may be of the form mtd@name to specify a mtd block device by name. \<flag>s include "ro", "rw", "remount", "noatime", ... \<options> include "barrier=1", "noauto\_da\_alloc", "discard", ... as a comma separated string, eg: barrier=1,noauto\_da\_alloc

powerctl Internal implementation detail used to respond to changes to the "sys.powerctl" system property, used to implement rebooting.

restart \<service> Like stop, but doesn't disable the service.

restorecon \<path> \[ \<path> ]\* Restore the file named by \<path> to the security context specified in the file\_contexts configuration. Not required for directories created by the init.rc as these are automatically labeled correctly by init.

restorecon\_recursive \<path> \[ \<path> ]\* Recursively restore the directory tree named by \<path> to the security contexts specified in the file\_contexts configuration.

rm \<path> Calls unlink(2) on the given path. You might want to use "exec -- rm ..." instead (provided the system partition is already mounted).

rmdir \<path> Calls rmdir(2) on the given path.

setprop \<name> \<value> Set system property \<name> to \<value>. Properties are expanded within \<value>.

setrlimit \<resource> \<cur> \<max> Set the rlimit for a resource.

start \<service> Start a service running if it is not already running.

stop \<service> Stop a service from running if it is currently running.

swapon\_all \<fstab> Calls fs\_mgr\_swapon\_all on the given fstab file.

symlink \<target> \<path> Create a symbolic link at \<path> with the value \<target>

sysclktz \<mins\_west\_of\_gmt> Set the system clock base (0 if system clock ticks in GMT)

trigger \<event> Trigger an event. Used to queue an action from another action.

verity\_load\_state Internal implementation detail used to load dm-verity state.

verity\_update\_state \<mount\_point> Internal implementation detail used to update dm-verity state and set the partition.\<mount\_point>.verified properties used by adb remount because fs\_mgr can't set them directly itself.

wait \<path> \[ \<timeout> ] Poll for the existence of the given file and return when found, or the timeout has been reached. If timeout is not specified it currently defaults to five seconds.

write \<path> \<content> Open the file at \<path> and write a string to it with write(2). If the file does not exist, it will be created. If it does exist, it will be truncated. Properties are expanded within \<content>.

### Properties

Init updates some system properties to provide some insight into what it's doing:

init.action Equal to the name of the action currently being executed or "" if none

init.command Equal to the command being executed or "" if none.

init.svc.\<name> State of a named service ("stopped", "running", "restarting")

### Bootcharting

This version of init contains code to perform "bootcharting": generating log files that can be later processed by the tools provided by [www.bootchart.org](https://www.bootchart.org).

On the emulator, use the -bootchart \<timeout> option to boot with bootcharting activated for \<timeout> seconds.

On a device, create /data/bootchart/start with a command like the following:

&#x20;adb shell 'echo $TIMEOUT > /data/bootchart/start'

Where the value of $TIMEOUT corresponds to the desired bootcharted period in seconds. Bootcharting will stop after that many seconds have elapsed. You can also stop the bootcharting at any moment by doing the following:

&#x20;adb shell 'echo 1 > /data/bootchart/stop'

Note that /data/bootchart/stop is deleted automatically by init at the end of the bootcharting. This is not the case with /data/bootchart/start, so don't forget to delete it when you're done collecting data.

The log files are written to /data/bootchart/. A script is provided to retrieve them and create a bootchart.tgz file that can be used with the bootchart command-line utility:

&#x20;sudo apt-get install pybootchartgui `# grab-bootchart.sh uses $ANDROID_SERIAL.` $ANDROID\_BUILD\_TOP/system/core/init/grab-bootchart.sh

One thing to watch for is that the bootchart will show init as if it started running at 0s. You'll have to look at dmesg to work out when the kernel actually started init.

### Debugging init

By default, programs executed by init will drop stdout and stderr into /dev/null. To help with debugging, you can execute your program via the Android program logwrapper. This will redirect stdout/stderr into the Android logging system (accessed via logcat).

For example service akmd /system/bin/logwrapper /sbin/akmd

For quicker turnaround when working on init itself, use:

&#x20;mm -j m ramdisk-nodeps m bootimage-nodeps adb reboot bootloader fastboot boot $ANDROID\_PRODUCT\_OUT/boot.img

Alternatively, use the emulator:

&#x20;emulator -partition-size 1024 -verbose -show-kernel -no-window

You might want to call klog\_set\_level(6) after the klog\_init() call so you see the kernel logging in dmesg (or the emulator output).

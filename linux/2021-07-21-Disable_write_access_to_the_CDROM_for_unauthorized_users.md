
[index](./index.md)

# Disable write access to the CDROM for unauthorized users 

---

Author:sandylaw

Email :freelxs@gmail.com

Date  :2021-07-21

tags  : linux

---

## udev

udev 是 Linux 内核的设备管理器。总的来说，它取代了 devfs 和 hotplug，负责管理 /dev 中的设备节点。同时，udev 也处理所有用户空间发生的硬件添加、删除事件，以及某些特定设备所需的固件加载。

## udev 规则

udev 规则以管理员身份编写并保存在 /etc/udev/rules.d/ 目录，其文件名必须以 .rules 结尾。各种软件包提供的规则文件位于 /lib/udev/rules.d/。如果 /usr/lib 和 /etc 这两个目录中有同名文件，则 /etc 中的文件优先。

参考wiki： https://wiki.archlinux.org/title/Udev_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

要想学习写udev规则，请访问编写 udev 规则 http://www.reactivated.net/writing_udev_rules.html

译注：这里有一篇转载的该文简体中文译本: http://www.cnitblog.com/luofuchong/archive/2007/12/18/37831.html

## acl

Acl(Access Control List)就是访问控制列表。

```bash
sudo apt install acl
```

### getfacl，命令名，获取文件访问控制列表,即ACL规则。

```bash
 [root@localhost ~]# getfacl --omit-header ./test.txt
 user::rw-
 user:john:rw-
 group::rw-
 mask::rw-
 other::r--
```

### setfacl，命令名，设置文件访问控制列表,即ACL规则。

```bash
用法: setfacl [-bkndRLP] { -m|-M|-x|-X ... } file ...

  -m, --modify=acl 更改文件的访问控制列表

  -M, --modify-file=file 从文件读取访问控制列表条目更改

  -x, --remove=acl 根据文件中访问控制列表移除条目

  -X, --remove-file=file 从文件读取访问控制列表条目并删除

  -b, --remove-all 删除所有扩展访问控制列表条目

  -k, --remove-default 移除默认访问控制列表

      --set=acl 设定替换当前的文件访问控制列表

      --set-file=file 从文件中读取访问控制列表条目设定

      --mask 重新计算有效权限掩码

  -n, --no-mask 不重新计算有效权限掩码

  -d, --default 应用到默认访问控制列表的操作

  -R, --recursive 递归操作子目录

  -L, --logical 依照系统逻辑，跟随符号链接

  -P, --physical 依照自然逻辑，不跟随符号链接

      --restore=file 恢复访问控制列表，和“getfacl -R”作用相反

      --test 测试模式，并不真正修改访问控制列表属性

  -v, --version           显示版本并退出

  -h, --help              显示本帮助信息
```
### ACL的名词定义

ACL是由一系列的Access Entry所组成的，每一条Access Entry定义了特定的类别可以对文件拥有的操作权限。
Access Entry有三个组成部分：
- Entry tag type
- qualifier (optional)
- permission。

Entry tag type它有以下几个类型：
类型	说明
ACL_USER_OBJ： 	相当于Linux里file_owner的permission
ACL_USER： 	定义了额外的用户可以对此文件拥有的permission
ACL_GROUP_OBJ： 	相当于Linux里group的permission
ACL_GROUP： 	定义了额外的组可以对此文件拥有的permission
ACL_MASK： 	定义了ACL_USER,ACL_GROUP_OBJ和ACL_GROUP的最大权限
ACL_OTHER： 	相当于Linux里other的permission

 举例例子说明一下

下面我们就用getfacl命令来查看一个定义好了的ACL文件：
```bash
[root@localhost ~]# getfacl ./test.txt
#file: test.txt
#owner: root
#group: admin
user::rw-
user:john:rw-
group::rw-
group:dev:r--
mask::rw-
other::r--
```

前面三个以#开头的定义了文件名，fi
 le owner和group。这些信息没有太大的作用，接下来我们可以用--omit-header来省略掉。
类型	说明
user::rw- 	 定义了ACL_USER_OBJ, 说明file owner拥有read and write permission
user:john:rw-  	定义了ACL_USER,这样用户john就拥有了对文件的读写权限,实现了我们一开始要达到的目的
group::rw-  	定义了ACL_GROUP_OBJ,说明文件的group拥有read and write permission
group:dev:r--  	定义了ACL_GROUP,使得dev组拥有了对文件的read permission
mask::rw-  	定义了ACL_MASK的权限为read and write
other::r-- 	 定义了ACL_OTHER的权限为read

## 禁用CDROM的未授权用户的写入功能

Not having the DVD mounted in the user interface doesn't mean the cannot write to it using a CD recording tool such as Brasero.
To avoid unauthorized users to write to a device, a modification to the standard udev rules must be performed.
Technically, the udev rules in /usr/lib/udev/rules.d/70-uaccess.rules tag devices which should be available to users with the uaccess tag.
Because of that, upon logging onto the graphical interface, the systemd-logind service will set up an ACL to let the user write to the device, as shown below:
Raw

```bash
# getfacl /dev/sr0
getfacl: Removing leading '/' from absolute path names
# file: dev/sr0
# owner: root
# group: cdrom
user::rw-
user:amd:rw-
group::rw-
mask::rw-
other::---
```
In the example above, we can see that even though /dev/sr0 is owned by root:cdrom and only root:cdrom has read-write capabilities, a new ACL user:test:rw- has been automatically created to allow the test user to read and write to the device. This is what we want to avoid.
Unfortunately for now, the only solution is to duplicate the /usr/lib/udev/rules.d/70-uaccess.rules file into /etc/udev/rules.d/70-uaccess.rules and modify the file to remove tagging of cdrom devices with the uaccess tag, as shown below (unified diff):
Raw

复制`/usr/lib/udev/rules.d/70-uaccess.rules`到`/etc/udev/rules.d/70-uaccess.rules`，修改后者，屏蔽CDROM的uaccess非授权用户（普通用户）的访问规则。

```bash
# diff -u /usr/lib/udev/rules.d/70-uaccess.rules /etc/udev/rules.d/70-uaccess.rules
--- /usr/lib/udev/rules.d/70-uaccess.rules  2017-04-21 09:03:53.000000000 +0200
+++ /etc/udev/rules.d/70-uaccess.rules  2017-08-01 15:27:34.291000000 +0200
@@ -21,8 +21,9 @@
 ENV{ID_HPLIP}=="1", TAG+="uaccess"

 # optical drives
-SUBSYSTEM=="block", ENV{ID_CDROM}=="1", TAG+="uaccess"
-SUBSYSTEM=="scsi_generic", SUBSYSTEMS=="scsi", ATTRS{type}=="4|5", TAG+="uaccess"
+# Prevent automatic rw access for users
+#SUBSYSTEM=="block", ENV{ID_CDROM}=="1", TAG+="uaccess"
+#SUBSYSTEM=="scsi_generic", SUBSYSTEMS=="scsi", ATTRS{type}=="4|5", TAG+="uaccess"

 # Sound devices
 SUBSYSTEM=="sound", TAG+="uaccess" \
```

By doing so, after a reboot of the machine, the optical drives will not be tagged with uaccess tag, causing systemd-logind not to create the ACL when the user logs in.
Testing that it works

    create the file
    reboot
    login as a regular user
    from a terminal as root, verify that no ACL has been created: getfacl /dev/sr0

再次查看ACL访问控制权限发现，没有普通用户的访问权限

```bash
# ls -l /dev/sr0
brw-rw---- 1 root cdrom 11, 0 7月  20 13:59 /dev/sr0
# getfacl /dev/sr0
getfacl: Removing leading '/' from absolute path names
# file: dev/sr0
# owner: root
# group: cdrom
user::rw-
group::rw-
other::---
```
## udev规则调用blockdev设置光驱只读

```bash
cat << EOF | sudo tee /etc/udev/rules.d/71-cdrom-ro.rules 
SUBSYSTEMS=="block", ACTION=="add", KERNEL=="sr*", RUN+="/sbin/blockdev --setro /dev/%k"
SUBSYSTEM=="scsi_generic", SUBSYSTEMS=="scsi", ATTRS{type}=="4|5", ACTION=="add", KERNEL=="sr*", RUN+="/sbin/blockdev --setro /dev/%k"
EOF

sudo udevadm control --reload

```



### 使用刻录机进行测试



参考：

- https://access.redhat.com/articles/3148751	

- https://sleeplessbeastie.eu/2015/05/11/how-to-enforce-read-only-mode-on-every-connected-usb-storage-device/



> 本笔记为安全体系与 Android 平台权限/签名机制速记。**最近一次校对：2025-09（基于 Android 15 / API 35 现状）**。文末与相关章节已补充近年重要变更：APK 签名 v2/v3/v4、分区存储、包可见性、运行时权限调整、前台服务限制、精确闹钟权限等。

扎绳篇：
非对称密码 公钥体系
证书 
包含公钥，公钥是利用证书来传递的

利用签名来保护数字证书本身

人的信任关系：一个信任人的列表
数字时代的信任关系：一个受信任者的证书列表

人的信任链：孔子-》孔子的徒弟-》孔子的徒弟的徒弟
数字时代的信任链：证书链
证书签名的不同点：根证书自签名，非根证书父签名
证书的限制：
    约束
    用途
    有效期
PKI的概念

基于可信任证书的认证方式被广泛的应用在现代安全领域，比如WIFI，HTTPS
在HTTPS中，典型的Client对Server的认证和鉴别基于可信任列表
织网篇:
# 进程和进程边界
## 进程和线程
可执行文件：不活动就是废物
进程：可执行文件的活动表现，一次生命的历练，资源的最小单位
线程：CPU(核)的调度单位,并发的执行序列，进程的多管齐下
资源和调度
## 手机操作系统的发展
Feature Phone 时代的实时简单的单进程多任务非智能系统
Smart Phone 时代的多进程多任务智能系统
## 进程的地址空间边界
每个进程的虚拟地址空间0-4G（32位），每个进程的物理地址空间是独立的

## 进程边界的安全围栏：Crash的不可扩展性
## 进程边界的安全围栏：全局数据和服务的不可访问性
回顾和总结

# 多用户和多用户边界
## 需求背景
资源缺乏
中央统一管理
## 多用户的边界：独立的工作目录
独立的老巢
/home/nuoen
## 多用户的边界：可操作/访问的资源
资源分类
权限管理
## 多用户的边界：可执行的操作
操作分类
权限管理
## 多用户的边界：UID和GID
Name只是供看的
Identifier才是系统层面的标识
用户的行为是一系列进程的行为
特性标识其实是进程的UID/GID

# 进程和文件的UID/GID
## 文件资源的权限力度：UID/GID
* 文件是一种资源
* 在Linux中，甚至一切皆是文件，Socket,Driver
* 文件资源对不同Target(用户)的不同操作权限的需求应运而生
* 如何描述和区分不同的target? ID=====》UID===》唯一的
* 某些场景下，允许多个不同的Target/用户 (而不是一个)具有一致的操作权限，怎么办？ID---->GID---->多个用户可以属于一个GID，一个用户可以属于多个GIDs
* 所以文件权限的管理力度区分为3类群体：属于特定UID的用户，属于特定GID的用户（们），其他用户
* 一个上帝用户存在：ROOT，其UID=0，上帝用户永远满足属于任何UID
```
ps -eo pid,uid,gid,user,args
```
## 文件的可操作权限
文件/文件夹的可读
文件/文件夹的可写
文件/文件夹的可执行
```
ls -l 
是否是文件夹|Owner用户权限|非Owner用户但是相同的组用户权限|既不是Owner用户也不是相同组的用户权限
*|***|***|***   UID  GID
drwxr-xr-x  2 root  root        4096 Apr 22  2024  mnt
drwxrwxr-x 29 nuoen nuoen       4096 Jun  7 04:55  msm
-rw-------  1 nuoen nuoen          0 May 20 11:20  nohup.out
```
## 进程的标识：PID,UID,GID,GIDs
PID:进程的Unique Identifier。每次Running的PID可能相同，或者不同，由系统分配
UID:进程的身份标识。每次运行，即便重启后默认都相同。不同进程允许有相同的UID(用户身份标识)
GID:进程的（组）身份标识。每次运行，即便重启后默认都相同。不同进程允许有相同的GID(组用户身份标识)。同一进程允许属于多个GID
GIDs:进程所属的全部GID

## Name 和 ID的映射
android_filesystem_config.h
```c
/* This is the master Users and Groups config for the platform.
 * DO NOT EVER RENUMBER
 */

#define AID_ROOT 0 /* traditional unix root user */
/* The following are for LTP and should only be used for testing */
#define AID_DAEMON 1 /* traditional unix daemon owner */
#define AID_BIN 2    /* traditional unix binaries owner */

#define AID_SYSTEM 1000 /* system server */

#define AID_RADIO 1001           /* telephony subsystem, RIL */
#define AID_BLUETOOTH 1002       /* bluetooth subsystem */
#define AID_GRAPHICS 1003        /* graphics devices */
#define AID_INPUT 1004           /* input devices */
#define AID_AUDIO 1005           /* audio devices */
#define AID_CAMERA 1006          /* camera devices */
#define AID_LOG 1007             /* log devices */


/* The 3000 series are intended for use as supplemental group id's only.
 * They indicate special Android capabilities that the kernel is aware of. */
#define AID_NET_BT_ADMIN 3001 /* bluetooth: create any socket */
#define AID_NET_BT 3002       /* bluetooth: create sco, rfcomm or l2cap sockets */
#define AID_INET 3003         /* can create AF_INET and AF_INET6 sockets */
#define AID_NET_RAW 3004      /* can create raw INET sockets */

#define AID_APP 10000       /* TODO: switch users over to AID_APP_START */
/** android apk 的uid都是10000开始的 **/

```
📌 Android UID 分配机制概览
UID 范围	用途	示例
0 – 999	内核 / root / 守护进程	0=root, 100=logd
1000 – 1999	系统服务 / 框架进程	1000=system
2000 – 2999	硬件守护程序	2000=radio, 2001=bluetooth
3000 – 9999	其他核心服务（AID_*）	3004=audioserver, 1073=networkstack
10000 以上	普通 APK 应用	10123=com.x.app


📦 普通应用为什么是 10000+？
	•	安装 APK 时，如果没有 <sharedUserId>：
	•	PackageManager 自动从 10000 开始为应用动态分配 UID。
	•	多用户模式下会按：

app_uid = user_id * 100000 + app_id


	•	如果有 <sharedUserId>，且不是 AID 预定义的 UID：
	•	必须满足条件：系统签名 + 安装在 system/priv-app。
	•	否则系统拒绝安装。

⸻
## chmod 和 chown 命令介绍
文件R/W/X的系统内部采用3Bit表示，R为最高位比特，置位为0x04,W为中间比特，置位为0x02,X为最低位比特，置位为0x01
shell中表示时，置位使用相应R/W/X表示，未置位使用-
操作文件面向群体的操作权限时，使用Chomd，可以直接使用数字，也可以使用助记符
(a:all ,u:owner user,g:group, +:add one permission, -:remove one permission)

chown 命令用于改变文件的所有者和所属组(UID和GID)
SHELL命令中通常采用Name方式修改，而不是ID方式
一般格式： chown newUID : newGID FileName

```shell
chown system:system nohup.out
```
## UID/GID的衔接
Linux一切皆是文件
文件基于UID/GID来划分它的面向群体，对它的面向群体定义不同的操作权限
用户的行为映射为进程的运行
进程的运行使用进程的UID/GID来标识自己的身份
进程的UID/GID<=======>文件的UID/GID 完美衔接！！
进程的UID/GID除了被授予可操作文件的范畴外，非文件范畴的需要进行权限控制的操作（如重启系统等特权操作）继续通过进程的UID/GID身份来进行控制和授权
比如，对于Reboot这个API,其入口处可以check calling的Process的UID,如果不是Root，则Reject

 # 进程的 Real UID 和 Effective UID
## 身份的标识：Real UID
* 进程的UID只是泛称，其实有很多种不同的UID
* 进程的Real UID是进程的身份的标识，用来说明 Who am I
* 仅仅说明Who am I,但是没有“实权”是不行的
* Linux中，进程能做什么事情不是由Real UID决定的
* Real UID仅仅是身份，有身份没有权利是无用的
## 权利的标识：Effective UID
* 有身份无权利是不行的
* Effective UID是进程的权利的标识，标识了该进程的“权利”
* Linux中的进程的授权（即，当前进程具有的操作权限）是靠Effective UID来识别的
* 有权利就能做一切，Linux中具有“特权”Effective UID的进程能为所欲为
* 之前课时说明的，文件，资源以及特权API操作时对进程是否有权限的识别的UID,即是指Effective UID
## 身份和权利的关系
* 一般情况下，身份和权利是一致的，即 Real UID = Effective UID
* 所以，默认PS CMD 输出的UID指的是Effective UID,而没有输出Real UID
```shell
ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0 170644 14180 ?        Ss   Jun28   0:51 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    Jun28   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   Jun28   0:00 [rcu_gp]
root           4  0.0  0.0      0     0 ?        I<   Jun28   0:00 [rcu_par_gp]
```
* 我们也可以显式的显示完整的Effective UID 和 Real UID
```shell
# 查看所有进程的真实/有效 UID、GID
ps -eo pid,ruid,euid,rgid,egid,user,comm

# 仅看某个 PID（示例：1234）
ps -o pid,ruid,euid,rgid,egid,cmd -p 1234

# 从 /proc 读取（同时可见 Capability）
cat /proc/1234/status | egrep 'Uid:|Gid:|Cap(Inh|Prm|Eff)'
```
## ROOT用户的特权
* 我们所指的root用户，均是Effective UID = root的进程，尽管一般情况下，Real UID = Effective UID
* Effective UID = root的进程具有“皇权”，他不受任何限制，可以为所欲为
* 它可以为自己正身：如果自己的身份（Real UID）不是root，它可以将自己的身份名正言顺的改为root(调用SetXUID),从而使身份和权利均是root
* 它也可以自降为庶民：出家皈依，将自己的Real UID 和 Effective UID 都设置（降低，也是调用SetXUID）为庶民（非特权普通的Real UID,Effective UID），从而失去特权.
## UID的世袭
* 在linux世界里，为了安全考虑，UID的世袭遵循：身份世袭而权利不世袭的准则
* 子进程的Real UID = Effective UID = 父进程的Real UID
* 这使得临时夺权且尚位正身（普通Real UID 而特权Effective UID）的进程的子嗣不能继承其特权而仅能继承其的正身 （Real UID）

# 文件setUID标识
## 平民身份、皇族特权的需求背景
* Linux中的password是一个可执行程序（CMD），用于修改用户的密码
* Passwd需要操作多用户的账号文件（该文件高度安全，仅有ROOT用户可以读写）
* 普通用户难道不能修改自己的密码？
* Passwd进程虽然是平民身份（普通用户启动它），却需要皇族的权利。身份和权利不一致。
## 如何解决
临时提升特权（Effective UID = root）而维持身份不变（Real UID = 普通用户），使得其利用特权行使职责时可避免世袭的安全问题。
## Linux的文件setUID标识
* 文件的Owner UID设置为特殊用户，比如ROOT
* 文件面向Owner UID的群体（第一组标志）的操作权限增加额外的setUID标识
* Linux系统保证，任何用户（进程）执行该可执行文件（会Fork一个新的进程来加载该可执行文件running）时，该可执行文件所在的子进程的Real UID仍然继承其父进程的Real UID，但是其Effective UID不再等于其父的Real UID而是被提升到该可执行文件的Owner UID
```shell
ls -l | grep passwd
-rwsr-xr-x 1 root root         88464 Feb  6  2024 gpasswd
-rwxr-xr-x 1 root root        254160 Dec  2  2022 grub-mkpasswd-pbkdf2
-rwsr-xr-x 1 root root         68208 Feb  6  2024 passwd
```
## Chomd设置setUID的方式
* 和基本的RWX设置类似，有助记符和直接数字设置。直接数字设置时，采用4位数字，其中第一位标志setUID
```shell
chmod 4775 xxx.file //设置setUID
chmod 0775 xxx.file //取消setUID
```
## setUID的安全问题
* sUID的进程的EUID提升了
* sUID的进程的RUID默认没有提升
* 在sUID进程的RUID没有正身（也设置为ROOT）之前，其子民的RUID/EUID只是平民,此时是安全的
* 要依据实际场景，有限制的决定是否要在sUID进程中为自己正身，需要明确知道其后果是，其任何子民的RUID/EUID均会提升至贵族（ROOT）
* 两个例子，passwd（未正身），Android的su（正身）
## 有RealGID,EffecitveGID,setGID吗
* 存在，待看

## 文件 Capability（setcap/getcap）
* 除了 setUID 之外，现代 Linux/Android 更推荐使用 **文件 Capability** 赋权，避免一刀切的 root 特权。
* 常用命令：
```shell
# 赋予二进制绑定低端口能力（示例）
sudo setcap 'cap_net_bind_service=+ep' /path/to/bin
# 查看文件能力
getcap /path/to/bin
```
* 在 Android 上，系统分区的二进制能力通常由构建系统打包时附加；第三方 App 无法随意为自身可执行文件设置 capability。

如此管理权限颗粒度太粗，不够精细，所以引入了Capability机制

# Linux的Capability机制
## UID怎么了
* 权限颗粒太粗
* 容易引起权利过剩（溢出）
* 权利溢出/过剩引起的安全问题
## Capability:细粒度的权限控制
* 我们需要细粒度的权限
* 除了皇帝，我们也需要不同的地方官
* Linux引入了Capability:每个Capability系统内以一位Bit代表，OS内部使用64bit存储
```c
//android_filesystem_capability.h
#define CAP_CHOWN 0
#define CAP_DAC_OVERRIDE 1
#define CAP_DAC_READ_SEARCH 2
#define CAP_FOWNER 3
#define CAP_FSETID 4
#define CAP_KILL 5
#define CAP_SETGID 6
#define CAP_SETUID 7
#define CAP_SETPCAP 8
#define CAP_LINUX_IMMUTABLE 9
#define CAP_NET_BIND_SERVICE 10
#define CAP_NET_BROADCAST 11
#define CAP_NET_ADMIN 12
#define CAP_NET_RAW 13
#define CAP_IPC_LOCK 14
#define CAP_IPC_OWNER 15
#define CAP_SYS_MODULE 16
#define CAP_SYS_RAWIO 17
#define CAP_SYS_CHROOT 18
#define CAP_SYS_PTRACE 19
#define CAP_SYS_PACCT 20
```
## 进程的Capability
* Permitted Capability Sets
  * 当前进程的权利的围栏，最大权利范围，是Effecitve Capability Sets的超集
* Effective Capability Sets
  * 当前进程的时机使用（支配）的权利集，该集内的Capability必须从属于Permitted Capability Sets。该集合与Effective UID类似，是实际的权利标识
* Inheritable Capability Sets
  * 子进程唯一可以直接继承的Capability Sets。在Capability模式下，只有子进程的Inheritable Capability Sets = 父进程的Inheritable Capabiltity Sets。其他皆是NO
## 文件的Capability
* Permitted Capability Sets
  * 该可执行文件可以为其进程带来的Permitted Capability Sets
* Effective Capability Set
  * 仅1bit,Enable or Disable ,标识该可执行文件running所在的进程的Permitted Capability Sets是否自动全部Assign到其Effective Capability Sets。通常用于与传统的Root-setUID可执行文件向下兼容。
* Inheritable Capability Sets
  * 与进程的Inheritable Capability Sets一起作用（位与）以决定新的进程的Permitted Capability Sets
## Capability BoundSet
* Capability BoundSet是进程的超集
* 是进程自己为自己设定的安全围栏（Capability Sets），限制可执行文件的Permitted Capability Sets仅有局部能转化为进程的Permitted Capability Sets
* Capability BoundSet能够被子进程继承
* Init进程默认Capability BoundSet 为全1
## Spawn进程的Capability
P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & cap_bset)
P'(effective) = F(effective)?P'(permitted):0
P'(inheritable) = P(inheritable)
```shell
cat /proc/[pid]/status | grep Cap
```
# Capability与UID的兼容
* 新技术必须向下（前）兼容，要保证旧的实体（应用）工作正常
* 旧的应用：Capability - dumb进程
  Permitted Capability Sets = Effective Capability Sets = Inheritable Capability Sets = 0x000000000000000,我不知道，我怎么设置？
* 旧的可执行文件：Capability - dumb文件:
  Root-setUID可执行文件，系统转变为Capability的方式为：File Effective Bit = True;File 's Permitted和Inheritable Capability Sets = 0xFFFFFFFFFFFFFFFF
* 旧的 Root EUID 的进程：对使用 v1（JAR）签名时代的 setUID 兼容，**File Effective Bit = true** 时仍会将文件的 Permitted 能力集赋予进程（受 BoundSet 限制）。
* 注意：公式中的常量应为 **PR_CAPBSET**（capability bound set），不是 PR_CAPBEST。
# 高级特性
## 被ROOT了怎么办
* 手机被ROOT了怎么办
* 恶意应用获取了ROOT权限怎么办？
* 现在无能为力
* 我们想怎么样？即便ROOT EUID的应用，仍然不能为所欲为
## SELinux
* 拯救英雄
* DAC和MAC的策略区别
* DAC(Discretionary Access Control)自主访问控制：传统U6 m0k9noijnix/Linux安全管理模型；主体对它所属的对象和运行的程序拥有全部的控制权
* MAC(Mandatory Access Control)强制访问控制：SELinux基于的安全策略；管理员管理访问控制。管理员指定策略，用户不能改变它。策略定义了哪个主体能访问哪个对象。采用最小特权方式，默认情况下应用程序和用户没有任何权限
* Web服务器的假想例子
DAC模式下，Web服务器进程具有ROOT权限，当恶意病毒攻击成功并注入Web服务器进程后，则可以利用Web服务器进程的ROOT权限，做任何事情。
MAC模式下：Web服务器进程所能操作的对象和权限均在安全策略中明确列出，比如，只允许访问网络和访问特定文件等。即便Web服务器被恶意病毒攻击注入了，你仍然无法借由Web服务器进程为所欲为，所有安全策略没有授权的行为仍然是不允许的。
## SEAndroid
* 与SELinux的关系
SEAndroid (Security-Enhanced Android) 将原本运用在Linux上的SELinux技术移植到了Android平台上
* 与SELinux的区别
  除了移植SELinux以外，还做了很多针对Android的安全提高，比如把Binder IPC、Socket、Properites访问控制加入到了SEAndroid的控制中。
* SEAndroid的核心理念
  即使恶意应用篡得了ROOT权限，恶意应用仍然被优先的控制着仍然不能为所欲为。

## Jelly Bean MR2（Android 4.3）的补丁（setUID 提权收紧）
* 在 Android 4.3 之前，APK 进程能通过 `Runtime.exec()` 执行 **root-setUID** 二进制并提升 EUID。
* 4.3 起，Zygote 在 fork 应用进程时**清空 Capability Bound Set**，使后续执行 setUID 文件时即使 EUID 改变，也**无实际 capability**可用。
* 关键逻辑位于 Zygote 分支（Dalvik/ART 不同版本位置不同），等价于：
```c
// 伪代码：清空 PR_CAPBSET（capability bound set）
for (int i = 0; prctl(PR_CAPBSET_READ, i, 0, 0, 0) >= 0; i++) {
    prctl(PR_CAPBSET_DROP, i, 0, 0, 0);
}
```
* 结论：普通 APK 进程执行 root-setUID 二进制不再具备有效的 capability，无法借此获取特权。

### JB MR2 细节（扩展版）
**动机**  
早期（4.2 及之前）APK 进程可通过 `Runtime.exec()`/`fork+exec` 执行带 **setUID-root** 位的二进制，从而把子进程的 **EUID 提升为 0**，再借助传统 root 权限做特权操作。为阻断这一途径，Android 4.3 在 Zygote 派生应用进程后，对 **Capability Bound Set（BSET）** 进行清空，使随后即便执行 setUID 文件，进程也**拿不到任何有效的 capability**。

**内核前提**  
* 依赖 Linux 的 **Capability Bounding Set** 机制（`prctl(PR_CAPBSET_DROP, cap)`）。  
* 当某 capability 被从 BSET 中 drop 后，后续**不可再添加**到该进程的 Permitted 集（除非通过 exec 进入 `init`/更高权限环境，但普通 APK 不具备）。

**Zygote 侧变化（伪代码）**  
```c
// 在 fork 普通应用进程后（进入 app uid/gid 环境前后），遍历所有 capability：
for (int cap = 0; prctl(PR_CAPBSET_READ, cap, 0, 0, 0) >= 0; ++cap) {
    prctl(PR_CAPBSET_DROP, cap, 0, 0, 0);  // 将 BSET 清空
}
// 随后再 setresgid / setresuid 至应用 uid/gid，清空附属组等。
// 注意：JB MR2 当时并**未**依赖 PR_SET_NO_NEW_PRIVS（较新内核才有），主要手段就是清空 BSET。
```

**能力传播公式回顾**  
对一次 `execve(file)`，新的进程 capability 计算（简化）为：
```
P' = (P_inh & F_inh) | (F_perm & BSET)
E' = F_eff ? P' : 0
I' = P_inh   // 仅 Inheritable 可直接继承
```
* 其中 `F_*` 来自**目标可执行文件**的 capability，`P_*` 为**当前进程**；`BSET` 为 **Capability Bound Set**。
* 当 **BSET = 0** 时，无论 `F_perm` 多大、`F_eff` 是否置位，`P'` 都只能来源于 `P_inh & F_inh`。而普通 APK 的 `P_inh` 默认为 0，故 `P' = 0`，`E' = 0`。

**对 setUID-root 的具体影响**  
* setUID 仍会令子进程的**EUID 变为 0**（身份层面），但由于 `P'=0`、`E'=0`，进程**没有任何有效 capability**，无法执行需要 capability 的特权操作（例如 `CAP_SYS_ADMIN`、`CAP_SYS_MODULE` 等）。
* 这等价于：**“看起来像 root，实则无权”**，从根本上堵住了通过 setUID 提权的常见路径。

**前/后对比（示例）**  
* *JB MR2 之前（BSET=全 1）*：
  - `F_eff=1`、`F_perm` 含若干能力 → `P'` 至少获得 `F_perm & BSET`，再赋给 `E'`，子进程具备相应特权。
* *JB MR2 之后（BSET=0）*：
  - 即便 `F_eff=1`，也因 `F_perm & 0 = 0` → `P'=0`、`E'=0`。

**如何验证**  
1. 在 APK 内部 `exec` 一个 setUID-root 的小程序：
   ```shell
   id; cat /proc/$$/status | egrep 'Uid:|Gid:|Cap(Inh|Prm|Eff)'
   ```
   可见 `Uid:` 的 `EUID` 为 0，但 `CapEff:` 为全 0。  
2. 改为在 `adb shell`（root 环境）直接运行同一程序（不经 Zygote app 沙盒）对比其 `CapEff`，能观察到差异。

**补充说明**  
* 这一设计与后续版本中的 **SELinux（Enforcing）**、**权限拆分/前台服务限制** 等共同组成多层防护。  
* 后续 Android/内核版本也引入了 `no_new_privs` 等手段，但 **JB MR2 的关键变化点就是清空 BSET**。

捕鱼篇
# 签名和权限
## 移动平台中的主流签名作用：自签名的完整性鉴别
* 什么是自签名
  证书是用来证明公钥拥有者的身份信息，一般由可信的第三方机构颁发，三方机构用自己的私钥签名拥有者的公钥，从而证明该证书的有效性
  证书的签名一般是由证书的颁发者的私钥签名
  自签名指证书的签名者（机构）和证书拥有者（个体）是同一个实体
  自签名即我自己的私钥签名我自己的公钥
* 自签名的作用1：作为信任链的根证书
  比如一个银行的公钥证书，由银行自己签名，银行自己颁发给自己，访问网上银行服务时，需要吧银行的根证书加入到系统中，从而信任该证书
* 自签名的作用2：完整性鉴别 

## 移动平台中的主流签名作用：信任模式
* 何为可信任？
  我写的应用？
  我信任的哥们写的应用？

* 如何鉴别可信任
  签名了？
  签名是可信的？（证书、证书回溯）

* 可信任和普通应用的权利差异
  人为的把一些Operation归类
  某类Operation对于可信任应用和普通应用的表现不一样
* 一些例子
  系统应用使用系统特权不需要用户弹窗授权
## 移动平台中的主流签名作用：限制安装/运行
* 应用安装时
  是否包含签名？----》没有？禁止安装
  提取证书进行验证----》证书是有效且可信的？----》不是？禁止安装
  基于证书的公钥对签名进行验证----》签名验证通过？----》不是？禁止安装
* 应用运行某些特殊代码（特权）时
  是否包含签名？----》没有？禁止运行
  提取证书进行验证----》证书是有效且可信的？----》不是？禁止运行
  基于证书的公钥对签名进行验证----》签名验证通过？----》不是？禁止运行
## 权限的作用：细粒度的特权管理
* 权限是一个ID或者一个字符串
* 权限用来细分权利（类似Capability）
* 通常一个权限与一类操作绑定
* 权限首先需要申请
* 但是申请后是否被批准由平台策略决定
* 例子1：应用读写SDCARD
* 例子2：应用reset手机
## 权限的安全性保护：通过签名
* 权限的完整性保护：防篡改
  例子（通过认证并获得签名后再加policy权限）
  实现方式：签名对（code+权限）完整性保护 ，如果权限被篡改，签名校验失败，拒绝执行
* 权限的授权安全策略：防Escalate
  例子（普通应用申请Inject Event权限）
  实现方式，即使用户申请了，也不能获得，因为签名不是系统的签名，所以不授予权限
  
## 回顾和总结

# Android中的签名
## Android的签名作用：完整性鉴别
* 支持自签名用于完整行鉴别
* 不做信任模式
* 不做安装和运行时的限制,即不限制证书是否是可信的
## Android的签名作用：Signature Permissions 和 ShareUID
* Signature Protected Level Permissions
  用于特权Permission
  只有特定签名的APK才被授权
```xml
      <!-- @SystemApi Allows access to hardware peripherals.  Intended only for hardware testing.
         <p>Not for use by third-party applications.
         @hide
    -->
    <permission android:name="android.permission.HARDWARE_TEST"
        android:protectionLevel="signature" />
 ```
* Share Process UID android:sharedUserId="xxxx"
  Process间Share UID的目的是共享资源等
  Android中两个APK Share相同的UID必须其签名所用的Private Key一样（为什么）
  如果shareUID相同，A可以访问B中的data/data下的资源
  android11 shareUID不再支持，因为它破坏了应用沙盒的安全模型，使得应用间的隔离变得复杂。
## Android的签名作用：身份ID和升级的匹配
* Android中的自签名只是代表了身份，但不代表身份是否可信任
* Android的应用的Identifier是Package Name:
  Package Name 不一样，互相不影响，允许同时存在（安装）
  Package Name 一样，只能存在一个，允许做升级处理
* 升级的安全性考虑
  必须签名的证书一致（防假冒，防侵入隐私）
  如果不一致，则用户要么放弃新的应用，要么先卸载旧的，再安装新的。但这属于安装，不属于升级
  正常的升级不擦除应用的工作目录数据，以保证历史数据的持续性
## AndroidAPK之META INF
* APK结构
* META INF的组成
*   •	Android 7.0 (Nougat) 开始支持 v2 签名（APK Signature Scheme v2），之后又有 v3/v4。
	•	使用 v2/v3/v4 签名时，可以不再包含 CERT.RSA、CERT.SF、MANIFEST.MF 文件，因为签名数据存储在 APK 尾部的 Signing Block 里。
**APK Signature Scheme 现状（2025）**
* **v1（JAR 签名）**：只保护单个条目；Android 7.0+ 仍支持但已不推荐，易被 zipalign 等修改破坏。
* **v2**：签名数据位于 APK **Signing Block**；显著加速安装校验，允许不再包含 `CERT.RSA/ CERT.SF/ MANIFEST.MF`。
* **v3**：在 v2 基础上支持**密钥轮换（Proof-of-rotation）**；升级时可更换签名密钥并建立信任链。
* **v4**：为**增量安装/分发**设计，搭配 v2/v3 使用，主要由安装器消费。
* 校验命令：
```shell
apksigner verify -v --print-certs your.apk
```
* Play 商店上架要求：应使用 v2+（新应用一般要求 v3），仅 v1 可能被拒。
* 签名流程
  PrivateKey(hash(CERT.SF)) => CERT.RSA
## 回顾和总结

 # Android中的权限
## 近年权限与隐私重大变化（Android 10~15 摘要）
1) **分区存储 Scoped Storage**（Android 10/11）
* 外部存储访问被沙箱化；`READ/WRITE_EXTERNAL_STORAGE` 逐步被弱化。
* Android 11 起提供 **MANAGE_EXTERNAL_STORAGE**（极度敏感，仅少量场景允许）。
* 媒体访问细分为：`READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO`（Android 13+），以及 **选择性访问** `READ_MEDIA_VISUAL_USER_SELECTED`（Android 14/15）。

2) **包可见性（Package Visibility）**（Android 11）
* 默认**无法枚举**系统上安装的其他包。
* 需要在 `AndroidManifest.xml` 中声明：
```xml
<queries>
    <package android:name="com.example.target" />
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="https" />
    </intent>
</queries>
```

3) **运行时权限拆分与新增**
* Android 12：蓝牙权限改为运行时 `BLUETOOTH_CONNECT/SCAN/ADVERTISE`，并引入 **NEARBY_DEVICES** 场景限制。
* Android 12：位置支持**近似/精确**；用户可仅授予近似位置。
* Android 13：**通知权限** `POST_NOTIFICATIONS` 改为运行时授权。
* Android 14/15：健康/传感器、后台权限进一步收紧（例如 `BODY_SENSORS_BACKGROUND` 需额外审查）。

4) **前台服务（FGS）与后台限制**
* Android 12 起要求声明 **前台服务类型**（`type="location|mediaPlayback|connectedDevice|dataSync|camera|microphone|health"` 等）。
* Android 14/15 对**后台启动 FGS**、超时与并发数量更严格；建议改用 **WorkManager + 明确的 FGS 类型**。

5) **精确闹钟**
* Android 13 引入 `USE_EXACT_ALARM`；Android 14/15 对非闹钟类 App 更**严格限制**，需合规理由与设置跳转。

6) **照片选择器（Photo Picker）**（Android 13+）
* 无需存储权限即可让用户选择媒体；优先使用系统 Photo Picker 替代自建文件选择。

7) **PendingIntent 可变性**
* Android 12 要求显式设置 `FLAG_IMMUTABLE` 或 `FLAG_MUTABLE`，默认不再安全。

8) **签名权限与 privapp 权限**
* 大量历史 `signatureOrSystem` 权限被收紧为 `signature`；OEM 侧须通过 `privapp-permissions` 白名单并配套签名。
## Android的权限作用：细粒度特权管理
* 权限与操作关联
  android不支持隐式申请权限，即使是系统应用，也需要显式申请权限
* 应用需要显式申请权限
* 用户对权限可知（不可控），
  现在可控（厂商处理）
* 对特权权限单独控制
## Android的不同权限类别
在 Android 中，**权限（Permission）**按作用范围和敏感程度可以分为几个主要类别。下面给为一个体系化梳理：
⸻
1. 普通权限 (Normal Permissions)
	•	特点：涉及用户隐私风险较小，系统会在安装时自动授予。
	•	示例：
	•	INTERNET：访问网络
	•	ACCESS_NETWORK_STATE：获取网络状态
	•	SET_WALLPAPER：设置壁纸
⸻
2. 危险权限 (Dangerous Permissions)
	•	特点：涉及用户隐私或设备安全，必须 运行时动态申请（Android 6.0+）。
	•	按权限组分类（常见组）：
	•	日历 (CALENDAR)
	•	READ_CALENDAR、WRITE_CALENDAR
	•	相机 (CAMERA)
	•	CAMERA
	•	联系人 (CONTACTS)
	•	READ_CONTACTS、WRITE_CONTACTS、GET_ACCOUNTS
	•	位置 (LOCATION)
	•	ACCESS_FINE_LOCATION、ACCESS_COARSE_LOCATION
	•	麦克风 (MICROPHONE)
	•	RECORD_AUDIO
	•	电话 (PHONE)
	•	READ_PHONE_STATE、CALL_PHONE、READ_CALL_LOG、WRITE_CALL_LOG、ADD_VOICEMAIL、USE_SIP、PROCESS_OUTGOING_CALLS
	•	传感器 (SENSORS)
	•	BODY_SENSORS
	•	短信 (SMS)
	•	SEND_SMS、RECEIVE_SMS、READ_SMS、RECEIVE_WAP_PUSH、RECEIVE_MMS
	•	存储 (STORAGE)
	•	READ_EXTERNAL_STORAGE、WRITE_EXTERNAL_STORAGE
⸻
3. 签名权限 (Signature Permissions)
	•	特点：只有当 请求权限的应用 和 声明权限的应用 使用相同的签名证书时，才能授予。
	•	应用场景：系统应用间的私有接口调用。
	•	示例：
	•	INSTALL_PACKAGES（安装 APK）
	•	DELETE_PACKAGES（卸载 APK）
⸻
4. 签名或系统权限 (SignatureOrSystem Permissions)
	•	特点：Android 5.0 以前允许系统预装应用（放在 /system/app）或相同签名应用使用。
	•	状态：Android 6.0 开始逐渐废弃，很多权限被收紧为 Signature-only。
	•	示例（老系统才有效）：
	•	WRITE_SETTINGS
	•	CHANGE_CONFIGURATION
⸻
5. 特殊权限 (Special Permissions)
这些不属于普通危险权限，需要通过 特殊 API 或系统设置界面 授权。
	•	SYSTEM_ALERT_WINDOW（悬浮窗权限）
	•	通过 Settings.canDrawOverlays() 检查
	•	WRITE_SETTINGS（修改系统设置）
	•	通过 Settings.System.canWrite() 检查
	•	REQUEST_INSTALL_PACKAGES（允许安装未知来源的 APK）
	•	Android 8.0 引入
	•	MANAGE_EXTERNAL_STORAGE（管理所有文件）
	•	Android 11 引入，替代传统的存储权限
	•	USE_FULL_SCREEN_INTENT（高优先级通知）
	•	Android 10 引入
⸻
6. 受限制权限 (Restricted Permissions, Android 10+)
	•	特点：Google 逐步收紧一些权限的使用，需要 Google 审核/声明特殊用途 才能上架 Play。
	•	示例：
	•	READ_SMS、SEND_SMS（敏感通讯权限）
	•	READ_CALL_LOG、WRITE_CALL_LOG
	•	ACCESS_BACKGROUND_LOCATION（后台定位权限，Android 10+ 严格限制）
⸻
✅ 总结：Android 权限主要分为：
	•	普通权限（自动授予）
	•	危险权限（需要运行时授权）
	•	签名/签名或系统权限（系统或签名一致才可用）
	•	特殊权限（需要通过系统设置授权）
	•	受限制权限（Google 审核，敏感程度高）
## Android的平台权限定义
  定义在AndroidManifest.xml中
Android 的权限本质上就是字符串常量，统一定义在 frameworks/base/core/res/AndroidManifest.xml 里。
这是系统框架的清单文件，其中包含了所有系统预定义权限，例如：
```xml
<permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:protectionLevel="dangerous" />
<permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:protectionLevel="dangerous" />
```
当在Application中 AndroidManifest.xml 里写：
```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```
就使用了这个权限，但是必须动态申请。

> 提示：从 Android 13 起，媒体读取权限按类型细分；从 Android 11 起，**`WRITE_EXTERNAL_STORAGE` 基本被废弃**，请改用 `MediaStore` + 具体媒体权限或 Photo Picker。

## Android的运行时权限控制方式：通过PM的CheckPermission
* Android独有的Service（底层平台不具有）
* 所以需要在Android本身Framework中控制
* 主流的Service一般都是基于Binder IPC或者其他IPC提供服务
* 所以在最低层控制（Service所在的service可执行文件 run中）以避免逃逸控制
  绕开Utility Function直接invoke Remote Service
  所以要在service run中进行控制
* 例子 DayDream
```java
void startDockOrHome(){
  awakenDreams();
  Intent dock = createDockIntent();
  if(dock != null){
    try{
      mContext.startActivityAsUser(dock,UserHandle.CURRENT);
      return;
    }catch(ActivityNotFoundException e){
    }

    private static void awakenDreams(){
      IDreamManager dreamManager = getDreamManager();
      if(dreamManager != null){
        try{
          dreamManager.awaken();
        }catch(RemoteException e){
          //fine,stay asleep then
        }
      }    
      }
  }
}

static IDreamManager getDreamManager(){
  return IDreamManager.Stub.asInterface(ServiceManager.checkService(DreamService.DREAM_SERVICE));
}

@Override //Binder call
public void awaken(){
  checkPermission(android.Manifest.permission.WRITE_DREAM_STATE);
  final long ident = Binder.clearCallingIdentity();
  try{}
}

private void checkPermission(String permission){
  if(mContext.checkCallingOrSelfPermission(permission) != PackageManager.PERMISSION_GRANTED){
    throw new SecurityException("Access denied to process: " + Binder.getCallingPid()+", must have permission " + permission);
  }
}
```

> 生产代码建议：在 **Binder 端（服务实现）**做二次校验（`checkCallingPermission()` / `enforceCallingPermission()`），避免调用方绕过客户端封装直接发起 IPC。

## Android的运行是权限控制方式：映射为OS的特定属性
* 非Android特有的Service（底层平台已经提供，如File访问，TCPIP数据首发等）
* 多个入口访问：Android API , Java API ,NDK C API ,Shell ,etc
* 底层控制准则，会聚口在底层，所以在底层（OS层面）统一控制，这样可以避免逃逸控制
* 所以复用OS的一些安全控制特性，比如GID
* 所以需要把Android空间的Permission Mapping到OS的GID
* 例子：访问SDCard
  sdcard目录下的文件权限如下所示：
```shell
# 外部存储挂载示例（不同设备存在差异）
marlin:/ $ ls -l /storage
total 8
drwx--x--x 3 root sdcard_rw 4096 2025-09-01 21:59 emulated
drwxr-xr-x 2 root root        60 1974-12-07 14:35 self
# 进程通过所属 GID（如 sdcard_rw / media_rw）与 FUSE/SDCardFS/FS-verity 等层配合受控访问
```

## Android的Permission与UID/GID的mapping

# Android 专题补充（2025）
## AID 与多用户 UID 计算
* `appId` 从 10000 起分配；多用户下 `uid = userId * 100000 + appId`。
* 典型系统 AID：`AID_SYSTEM=1000`、`AID_RADIO=1001`、`AID_BLUETOOTH=1002`、`AID_GRAPHICS=1003` 等（详见 `android_filesystem_config.h`）。

## SELinux（SEAndroid）要点
* 强制访问控制（MAC）默认 **Enforcing**；检查：`getenforce`。
* App 进程运行在诸如 `u:object_r:app_data_file:s0:cXX,cYY`（文件）与 `u:r:untrusted_app:s0:cXX,cYY`（进程域）之下；越权访问由策略阻断。
* 调试常用：`adb shell dmesg | grep avc:`、`adb logcat | grep denied` 定位策略拒绝。

## 调试与自检清单
* 查看进程身份与能力：`ps -eo pid,ruid,euid,rgid,egid,user,comm`; `cat /proc/$PID/status`。
* 查看签名方案与证书：`apksigner verify -v --print-certs your.apk`。
* 遵循现代权限：优先 Photo Picker / MediaStore；声明 `queries`；细化 FGS 类型；声明 PendingIntent 可变性；最小权限原则。

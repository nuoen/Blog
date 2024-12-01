Android Studio导入源码相关
1、编译
source lunch
 mmm development/tools/idegen/
sh ./development/tools/idegen/idegen.sh
生成的产物有android.ipr , android.iml

2、使用Android Studio打开android.ipr之前的配置

可更改下文件权限，chmod 777等
adnroid.ipr: Android Studio打开选择此文件

android.iml: 用来描述modules，一般只看framework相关代码，可将其他无用modules exclude掉，减少index时间

在android.iml中加入excludeFolder  exclude_modules

3、配置SDKs
通过右击project的 根节点，选择“open module settings”打开

先Java SDK（删除class path），再Android API（删除source path）
![alt text](../pic/javasdk.png)

4、删除不必要的依赖

只保留<Module source>和Android API Platform即可
![alt text](../pic/module_source.png)
5、可将framebases目录添加为目录依赖，如上图

注意，frameworks 要在Module Source上面，要不然代码可能会跳到生成目录的文件里

6、在Project视图中点击设置按钮，取消勾选Show Exclude Files

7、导航条中 File→ Settings → Version Control 中删除不必要的目录管理，保留fw和 fw


## android.bp讲解
https://gityuan.com/2018/06/02/android-bp/

源码调试：
https://www.cnblogs.com/yongfengnice/p/18246075

打开文件/etc/sysctl.conf，在这个文件加入一行：

fs.inotify.max_user_watches=524288
保存文件后执行：

sudo sysctl -p

最后可以执行：

cat /proc/sys/fs/inotify/max_user_watches

来检查这个值是否被成功修改了。
2 增加Android Studio的堆内存
使用Android Studio浏览这么大的项目会存在堆内存不够的问题。

在Android Studio中点击File → Settings → Appearance & Behavior → System Settings → Memory Settings，将其中所有heap size调到最大然后点击ok。
3 编译idegen模块
cd 源码目录

source build/envsetup.sh

lunch 机型名-userdebug

如果没有编译过任何模块：

make idegen -j12

或

mmm development/tools/idegen -j12

如果编译过任何模块（参考增量编译单个模块）：

ninjabuild idegen -j12
4 生成android.ipr和android.iml文件
编译完成后执行：

./development/tools/idegen/idegen.sh

执行完成可以看到在源码目录出现了android.ipr和android.iml文件

android.iml文件中包括了太多的源码目录和jar包，如果此时直接用Android Studio打开ipr文件，扫描会非常耗时。通常如果只看framework层代码，只需要扫描framewroks, miui, packages, tools这几个目录就够了，并且jar包也是完全不需要的。而直接手动修改android.iml文件又十分麻烦，因此我写了一个python脚本来处理。
执行完成后即完成了对android.iml文件的删减，在我的Ubuntu主机上实测可以将后面的3个小时扫描时间缩短到10分钟。

最近经常碰到执行idegen.sh脚本卡死的情况，目前的解决方案是按ctrl+c取消执行脚本，然后将其它项目的android.ipr和android.iml文件复制到源码目录使用即可，这里提供一份已经完成裁剪的android.ipr和android.iml文件：
5 用Android Studio打开项目
在Android Studio中点File → Open选择刚刚生成的android.ipr文件；

等待右下角的文件扫描任务完成就可以看代码并且可以有代码跳转了。

如果你之前没有执行过cutiml.py脚本，此时代码跳转会跳到.class文件里面而不是.java文件中，修复方法：

点击File → Project Structure... → Modules → Dependencies将这里面的jar包全部删除然后点击OK。

6 调试安卓源码
在Android Studio中点File → Project Structure... → Project → Project SDK选择Android API XX Platform，其中XX与你的安卓版本一致，例如R版本就是30。（如果在这里没有选择Android SDK会导致Android Studio中不显示连接的设备）

调试方法：

1 插入手机并打开USB调试；

2 在需要调试的代码中打断点；

3 点击Attach Debugger to Android Process看到Choose Process对话框出现；（点击按钮没反应不出现对话框的修复方法：点击Add Configuration...然后点击左上角的+，选择Android App并点击OK）

4 勾选Show all processes；

5 选择要调试的进程然后点击OK即可开始调试。（注：这里system_process就是SystemServer进程）
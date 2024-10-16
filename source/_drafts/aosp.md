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
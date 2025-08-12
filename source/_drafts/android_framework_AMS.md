# AMS

ActivityManagerService 启动

1. SystemServer 在boot的时候启动
frameworks/base/services/java/com/android/server/SystemServer.java
```java
 private void startBootstrapServices() {
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(mSystemServiceManager, atm);
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
 }

```
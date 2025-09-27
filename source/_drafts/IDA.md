ida调试 https://bbs.kanxue.com/thread-259633.htm

ida 调试，调试arm运行android_service32
调试arm64 运行android_service64
1. ida 的 android_service 文件push 到手机中
2. 开启 android_service
3. ida导入将要调试的so文件
4. 端口转发 
   adb forward tcp:23946 tcp:23946
5. 以调试模式开启app
   adb shell am start -D -n com.nuoen.attacker/.MainActivity
6. 获取attacker的进程号31142,jdwp转端口（教程中的ddms）
   adb forward tcp:8700 jdwp:31142

   jdb -connect com.sun.jdi.SocketAttach:hostname=localhost,port=8700 ?
   无反应
   为什么无反应：因为开了android studio (吐血)

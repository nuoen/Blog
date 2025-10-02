native攻防
## 1.使用软件壳工具对so进行加壳
加壳：pux
`github.com/upx/upx`
## 2.编译OLLVM工具链套件
## 3.使用OLLVM进行Native层混淆
## 4.环境检测与反调试检测

### 反IDA
https://zhuanlan.zhihu.com/p/642925410
1. IDA端口检测
   更改默认端口
   root@phone:/data/local/tmp # ./as_64 -p12346
2. 特征文件检测
   android_server 特征文件名检测
3. TracePid检测
   安卓的native下，通过读取进程的status或stat来检测Tracepid ，它主要原理是调试状态下的进程Tracepid不为0
   当检测到Tracepid 不为0时，app会kill掉进程，从而达到反调试的目的。
   一般情况下当 app 只有一个进程时，附加后程序不会立即退出，而是等待运行到相关退出函数时才会退出，原因很简单，只有一个进程时程序会断下但没办法继续往下执行，便不能够杀死自己，针对这种情况一般会 fork 出一个子进程，让子进程负责 kill 父进程，再优化便是父子进程相互检测是否被调试。
反反调试：
    让Tracepid在调试的状态下，Tracepid 仍然为0，并且以为万一让kill失效，但能让 app 退出的函数并不止这一个。
exp:




5.so文件符号隐藏与精简方法

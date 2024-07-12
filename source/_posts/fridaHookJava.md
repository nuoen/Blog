title: frida hook java
author: nuoen
tags: []
categories:
  - Frida
date: 2023-05-05 14:15:28
---
## objection 
* objection 附加模式hook
		在启动时hook,避免错过hook时机
		objection -g packageName explore --startup-command "android hooking watch"

## Frida 启动

###  attach 启动
直接附加到指定包名的应用中
``` bash
frida -U com.kevin.android -l hook.js
```
直接附加到当前应用中
``` bash
frida -UF -l hook.js
```

```python
import sys
import time
import frida

def on_message(message,data):
    print("message",message)
    print("data",data)

device = frida.get_usb_device()
session = device.attach("com.kevin.demo1")

with open("./demo1.js","r") as f:
    script = session.create_script(f.read())

script.on("message",on_message)
script.load()
sys.stdin.read()
```

        
``` Javascript
function printJavaStack(tag) {
    Java.perform(function () {
        console.log(tag + "\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));
    });
}
```
```
Java.choose("com.mitv.videoplayer.fragment.InnerVideoFragment" , {
    onMatch : function(instance){ //This function will be called for every instance found by frida
    console.log("Found instance: "+instance);
    console.log("mOnInfoListener : " + instance.mOnInfoListener.value);
    var runnable=DisabelSecureRunable.$new(instance);
    instance.runOnUiThread(runnable);
    console.log("stack: "+Java.use("java.lang.Exception").$new("Exception").getStackTrace());            },
    onComplete:function(){}
});
```
java.choose ,只有具体的类名才会生效， 比如InnerVideoFragment 继承 VideoFragment ,找VideoFragment会找不到。

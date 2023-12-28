## 米兔儿童
1.拉起米兔儿童
* adb shell am start -n com.xiaomi.mitukid/com.xiaomi.children.SplashActivity
* adb shell am start -a android.intent.acton.VIEW -d "mitu://mitu.com/splash"(调不起来)
* adb shell am start -a android.intent.acton.VIEW -d "mitu://mitu.com/splash com.xiaomi.mitukid/com.xiaomi.children.SplashActivity"
* adb shell am start -n "com.xiaomi.mitukid/com.xiaomi.children.SplashActivity" -a android.intent.action.MAIN -c android.intent.category.LAUNCHER
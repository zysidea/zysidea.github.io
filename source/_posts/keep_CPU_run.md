---
title: 关于Android黑屏待机之后保持CPU运作执行任务
categories:
  - Android
id: 21
date: 2015-12-10 22:36:50
tags:
---

今天遇到一个问题，要求开启一个异步线程做一个任何（暂且称呼为Task）,一切都很正常，但是无意间发现如果关闭屏幕，手机CPU就不会运作了，同时线程也停止运行，这个问题一般出现在后台下载，静默安装都会涉及到。
百度了一下，结果都是利用**PowerManager**和**WakeLock**，但是这个解决办法有个很大的弊端，就是虽然是activity级别的，但是一旦黑屏，Activity进入后台，Task还是会中断运行。最后google了一下需要用服务来运行耗时的任务，在Android Developer发现了**WakefulBroadcastReceiver**和**IntentService**的组合：

```java
public class MyWakefulReceiver extends WakefulBroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {

        // Start the service, keeping the device awake while the service is
        // launching. This is the Intent to deliver to the service.
        Intent service = new Intent(context, MyIntentService.class);
        startWakefulService(context, service);
    }
}

public class MyIntentService extends IntentService {
    public static final int NOTIFICATION_ID = 1;
    private NotificationManager mNotificationManager;
    NotificationCompat.Builder builder;
    public MyIntentService() {
        super("MyIntentService");
    }
    @Override
    protected void onHandleIntent(Intent intent) {
        Bundle extras = intent.getExtras();
        // Do the work that requires your app to keep the CPU running.
        // ...
        // Release the wake lock provided by the WakefulBroadcastReceiver.
        MyWakefulReceiver.completeWakefulIntent(intent);
    }
}
```



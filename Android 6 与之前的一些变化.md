## Android 6.0的一些变化   

 
本文列出开发过程中遇到的一些Android6.0的变化

1。 **<font size = 5>Mac地址的获取</font>**　　　


　　android 6.0之前，通过WifiManager-WifiIno-getMacAddress()就可以获取到设备的ma地址

```
 public static String getWiFiMac(Context context)
    {
        WifiInfo wifiInfo = getWifiInfo(context);
        if (wifiInfo == null)
            return null;
        String wifiMac = wifiInfo.getMacAddress();     
        return wifiMac;
    }

    public static WifiInfo getWifiInfo(Context context)
    {
        WifiManager wifiManager = getWifiManager(context);
        if (wifiManager != null)
        {
            WifiInfo wifiInfo = wifiManager.getConnectionInfo();
            return wifiInfo;
        }
        else
            return null;
    }

    public static WifiManager getWifiManager(Context context)
    {
        WifiManager wifiManager = (WifiManager) context
                .getSystemService(Context.WIFI_SERVICE);
        return wifiManager;
    }

```
　　　但是Android6.0通过该方法只能获得一个固定的字符串“02:00:00:00:00:00”  
　
　　解决方法：

```
 public static String getMac()
    {
        String str = "";
        String macSerial = "";
        try
        {
            Process pp = Runtime.getRuntime().exec("cat /sys/class/net/wlan0/address ");
            InputStreamReader ir = new InputStreamReader(pp.getInputStream());
            LineNumberReader input = new LineNumberReader(ir);

            for (; null != str;)
            {
                str = input.readLine();
                if (str != null)
                {
                    Log.d(tag, "cat sys/class/net/wlan0/address");
                    macSerial = str.trim();// 去空格  
                    break;
                }
            }
        }
        catch (Exception ex)
        {
            Log.e(tag, "error = " + ex.getMessage().toString());
        }

        if (macSerial == null || "".equals(macSerial))
        {
            try
            {
                Log.d(tag, "sys/class/net/eth0/address");
                return loadFileAsString("/sys/class/net/eth0/address").toUpperCase(
                    Locale.getDefault()).substring(0, 17);
            }
            catch (Exception e)
            {
                Log.e(tag, "error = " + e.getMessage().toString());
            }

        }
        return macSerial;
    }

 public static String loadFileAsString(String fileName) throws Exception
    {
        FileReader reader = new FileReader(fileName);
        String text = loadReaderAsString(reader);
        reader.close();
        return text;
    }

    public static String loadReaderAsString(Reader reader) throws Exception
    {
        StringBuilder builder = new StringBuilder();
        char[] buffer = new char[4096];
        int readLength = reader.read(buffer);
        while (readLength >= 0)
        {
            builder.append(buffer, 0, readLength);
            readLength = reader.read(buffer);
        }
        return builder.toString();
    }
``` 



2。  **<font size = 5>动态权限的申请</font>**

　　Android 6.0之前，开发者只需要在AndroidManifesxt.xml中配置自己想要的权限就可以，当你安装应用程序的时候，会有一个页面提示你当前的应用需要你哪些权限。当前手机的ROM一般的都有管理应用程序权限的设置。  
　　Android6 Google推出了动态权限的获取，如下危险的权限需要动态的症的用户的同意，比如：身体传感器、日历、摄像头、通讯录、地理位置、麦克风、电话、短信和存储空间。 Android6.0默认的为targetSdkVersion<23的应用程序默认授权了所申请的所有权限，所以如果你以前 APP设置的targetSdkVersion<23，在运行时也不会崩溃，但是这只是一个临时的救急策略。    

　　下面以申请SdCard的写权限为例说明：    

```
    private void requestPermission()
    {
        if (checkSelfPermission(Manifest.permission.WRITE_EXTERNAL_STORAGE) != PackageManager.PERMISSION_GRANTED)
        {
            if (shouldShowRequestPermissionRationale(Manifest.permission.WRITE_EXTERNAL_STORAGE))
            { //如果你上次拒绝了该权限
                Toast.makeText(this, "必须具有sdcard权限", Toast.LENGTH_LONG).show();
                requestPermissions(
                    new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE},
                    EXTERNAL_STORAGE_REQ_CODE);
            }
            else
            {
                requestPermissions(
                    new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE},
                    EXTERNAL_STORAGE_REQ_CODE);
            }
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, String permissions[],
            int[] grantResults)
    {
        if (grantResults.length <= 0)
            return;

        if (grantResults[0] == PackageManager.PERMISSION_DENIED)
        {//用户拒绝了该权限，需要进行界面逻辑处理。
        }
        else if (grantResults[0] == PackageManager.PERMISSION_GRANTED)
        {
            switch (requestCode)
            {
                case EXTERNAL_STORAGE_REQ_CODE :
                {
                    createFile("hello.txt");
                }
            }
        }

    }
```  

3。  **<font size = 5>悬浮窗的处理</font>**  
　　Android 6.0之(这时候你的targetSdkVersion<23)，创建悬浮窗是需要申请权限：**android.permission.SYSTEM_ALERT_WINDOW**。 让后通过WindowManager就可以创建并显示一个悬浮窗（当前你必须授权该权限，MIUI这种Rom默认的是关闭的，需要在设置中打开）。  
　　在Android6.0之后，如果你修改了targetSdkVersion为23（这个只能增大，不能变小回退）。  你再次通过WindowManager来创建悬浮窗，你的APP就直接崩溃了。  
　　解决办法： 如果你将targetSdkVersion设置为23或者更高，在使用SYSTEM_ALERT_WINDOW权限是，需要先调用**Setting.canDrawOverlays()**来判断是否允许创建悬浮窗。允许直接创建；如果不允许需要发送一个action值为**ACTION_MANAGER_OVERLAY_PERMISSION**的Intent来让用户同意创建悬浮窗。 具体你代码如下所示：  

```
if(Build.VERSION.SDK_INT >=23) {
	if(Seetings.canDrawOverlays(context) {
		//显示你的的悬浮窗
	}
	else {
		Intent inent = new Intent(Setting.ACTION_MANAGER_OVERLAT_PERMISSION);
		startActivity(intent);
	}
}
```

　　**“通过将WindowManager.LayoutParams的type设置为TYPE_TOAST”，是否还管用？**  
　　之前有人通过逆向的方法得出结论：[将WindowManager.LayoutParams的type设置为TYPE_TOAST，可以不申请SYSTEM_ALERT_WINDOW权限就可以显示悬浮窗](http://www.jianshu.com/p/634cd056b90c)，但是这种方法需要处兼容问题，比如：在MIUI下还是需要权限（同理其他ROM可能也会面临同样的问题），并且需要API level>=19（Android4.4）（老版本不响应触摸事件）。 具体的结论如下所示：  

```
在4.0.1以前, 当我们使用TYPE_TOAST, Android会偷偷给我们加上FLAG_NOT_FOCUSABLE和FLAG_NOT_TOUCHABLE, 4.0.1开始, 会额外再去掉FLAG_WATCH_OUTSIDE_TOUCH, 这样真的是什么事件都没了. 而4.4开始, TYPE_TOAST被移除了, 所以从4.4开始, 使用TYPE_TOAST的同时还可以接收触摸事件和按键事件了, 而4.4以前只能显示出来, 不能交互。  
API level 18及以下使用TYPE_TOAST无法接收触摸事件的原因也找到了.
文／Shawon（简书作者）
原文链接：http://www.jianshu.com/p/634cd056b90c
```
　　那么这种方法在Androi6.0中(排除MIUI这种ROM)，还可以吗？   
　　我们拿一个Android 6.0的手机(天机AXON)，经过测试（[demo](https://github.com/liaohuqiu/android-UCToast)）：**<font size = 5>是可以的</font>**   
　　当然，最可靠的方案还是：**read  the fucking sourecode !!!**  

4。 **<font size = 5>Apache HttpClient的移除</font>**   
　　早在Android2.3 Android就建议使用HttpURLConnection来进行网络开发。而且Google退出的一些开源都是这么做的，如：Volley等。  

5。 **<font size = 5>WiFi和网络变化</font>**  
　　1.  你的app智能修改自己的创建的WifiConfiguration对象的状态，不能修改其他App创建的WifiConfiguration对象。  
　　2.  Android 6.0之前，你可以通过enableNetwork（），设置disableAllOthers = true
，来是的设备软开其他网络，如蜂窝网络，而强制连接指定的Wifi网络。此版本上设备将不会从其他网络断开。  

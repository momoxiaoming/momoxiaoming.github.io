### Android实现钉钉自动打卡功能(AccessibilityService版本)

===============================================

**目录**
=================
-----------
[TOC]



### 为什么要做这个项目?

有天早晨下大雨,小编虽然出门早却还是路上堵的迟到了,心中一句XXX崩腾而过啊,这月全勤又没了,无奈之余想起既然技术能解决一切,那能不能搞个自动打卡的功能(好像有点作弊的嫌疑...哈O(∩_∩)O哈哈~),这样以后就不用在考虑会迟到了!于是一个邪恶的程序就诞生了.


### 一. 项目需求

##### 项目功能:

1. 程序启动后,一直后台运行,自动启动钉钉,并进入相应的打卡页面进行打卡(需要用到模拟点击功能).
2. 程序的执行时间段为上午8-9点为上班打卡,18-19点为下班打卡(时间段根据需求即可).
3. 确认打卡成功之后程序进入休眠状态,等待下次指令.
4. 程序必须24小时处于激活状态,避免被系统清理

##### 项目流程:

![WX20180727-192204@2x.png](https://upload-images.jianshu.io/upload_images/1978245-d4c5c2b846f4bc39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 二. 资源准备

大致需要准备以下东西:

1. 一台空闲的andorid手机,能root最好.
2. 下载钉钉,登陆账号
3. 手机设置充电不锁屏,并且连接了相应的打卡wifi.

### 三. 核心代码架构

项目的核心在于利用程序模拟人工打卡操作,需要用到android模拟点击功能的相关api,目前比较常用的黑科技主要是以下两种:

##### AccessibilityService

AccessibilityService本来是做一些辅助功能的，提供了一系列的事件回调，帮助我们指示一些用户及界面的状态变化，主要给残障人群提供帮助.手机上的所有操作都会通过onAccessibilityEvent方法返回,我们可以利用该原理做到模拟点击我们需要的操作程序.
不过，现在AccessibilityService已经基本偏离了它设计的初衷，至少在国内是这样，越来越多的App借用AccessibilityService来实现了一些其它功能，甚至是灰色产品。

##### UiAutomator

基于UIAutomation的用户界面自动化测试框架，可以跨应用工作，谷歌亲生的.
UIAutomation在Android4.3发布时有了新版本，[官方简介](http://blog.csdn.net/zhubaitian/article/details/40504827。)
Android4.3之前：使用inputManager或者更早的WindowsManager来注入KeyEvent

-----------------
当然,除了以上两种,还有其他的一些能实现模拟点击的框架,这里我就不一一赘述了,今天我们要用的就是利用AccessibilityService 辅助功能来实现我们的自动打卡功能.

------------

### 四. 功能实现
------------

#### 4.1 配置AccessibilityService,监听手机操作


1.继承AccessibilityService类,监听手机运行状态信息

```java
public class MainAccessService extends AccessibilityService {

 	@Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
    	//手机的所有操作信息都会通过这个方法回调
    	
    }
    
    @Override
    public void onInterrupt() {

    }

    @Override
    protected void onServiceConnected() {
        super.onServiceConnected();

    }
}


```
2.配置AccessibilityService,创建accessibility\_service\_config.xml文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"

					   android:accessibilityEventTypes="typeAllMask" //过滤所有时间
					   android:accessibilityFlags="flagReportViewIds" //辅助服务额外的flag信息
					   android:accessibilityFeedbackType="feedbackSpoken"//事件的反馈类型
					   android:notificationTimeout="100" //通知超时时间
					   android:canRetrieveWindowContent="true" //是否可以获取窗口内容

/>
```

3.AndroidManifest引用创建的配置文件(以下是配置必须)

```
<service android:name=".MainAccessService"
				 android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
			<intent-filter>
				<action android:name="android.accessibilityservice.AccessibilityService" />
			</intent-filter>

			<meta-data
				android:name="android.accessibilityservice"
				android:resource="@xml/accessibility_service_config"/>
		</service>
```

4.在设置中打开辅助功能服务

------
检查辅助服务是否开启

```java
 private void openAccessSettingOn(){
        if (!isAccessibilitySettingsOn(getApplicationContext())) {
            Toast.makeText(getApplicationContext(), "请开启辅助服务", Toast.LENGTH_SHORT).show();
            Intent intent = new Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS);
            startActivity(intent);
        }
    }


private boolean isAccessibilitySettingsOn(Context mContext) {
        int accessibilityEnabled = 0;
        // TestService为对应的服务
        final String service = getPackageName() + "/" + MainAccessService.class.getCanonicalName();
        // com.z.buildingaccessibilityservices/android.accessibilityservice.AccessibilityService
        try {
            accessibilityEnabled = Settings.Secure.getInt(mContext.getApplicationContext().getContentResolver(),
                    android.provider.Settings.Secure.ACCESSIBILITY_ENABLED);
        } catch (Settings.SettingNotFoundException e) {
            e.printStackTrace();
        }
        TextUtils.SimpleStringSplitter mStringColonSplitter = new TextUtils.SimpleStringSplitter(':');

        if (accessibilityEnabled == 1) {
            String settingValue = Settings.Secure.getString(mContext.getApplicationContext().getContentResolver(),
                    Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES);
            if (settingValue != null) {
                mStringColonSplitter.setString(settingValue);
                while (mStringColonSplitter.hasNext()) {
                    String accessibilityService = mStringColonSplitter.next();

                    if (accessibilityService.equalsIgnoreCase(service)) {
                        return true;
                    }
                }
            }
        }
        return false;

    }

```

--------


#### 4.2 实现自动化打卡流程

---------
配置好AccessibilityService服务后,接下来我们就可以在onAccessibilityEvent方法中写我们自动化脚本的逻辑了.具体流程看第一节中的图.
##### 4.2.1 保证手机处于桌面(以下是部分核心代码)

```java
AccessibilityNodeInfo node=getRootInActiveWindow();
if (node == null || !Comm.launcher_PakeName.equals(node.getPackageName().toString())) {
                throw new Exception("程序不在初始化启动器页面,抛出异常");
            }

```
注意上面的手动异常和下面所有的手动抛出异常到最后是会有大作用的,后面会讲到.


##### 4.2.2 启动钉钉

```java
AccessibilityNodeInfo node=getRootInActiveWindow();
int m = 10;
            while (m > 0) {
                LogUtil.D("循环--" + node);
                if (node != null && Comm.dingding_PakeName.equals(node.getPackageName().toString())) {
                    node = getRootInActiveWindow(); //刷新根页面节点
                    LogUtil.D("已进入app" + node);                
                    break;
                } else {
                    startApplication(getApplicationContext(), Comm.dingding_PakeName);
                }
                sleepT(1000);  //1秒钟启动一次
                if (node != null) {
                    node = refshPage();
                }
                m--;
            }
            if (m <= 0) {
                throw new Exception("进入钉钉主页异常");
            }
```
这里我用了10次循环去尝试启动钉钉,,假如10次之后都没有进入钉钉或者已进入钉钉,都将抛出异常,此次脚本终止.(目的是防止出现启动时卡死,导致脚本也卡死)

##### 4.2.3 判断是否位于钉钉主页面

通过Android SDK的uiautomatorviewer工具(在tools文件夹下,需要手机root,studio的sdk可能和elipse的不同),查看页面的节点信息,如下图:

![12.jpg](https://upload-images.jianshu.io/upload_images/1978245-e078ac5e4d301079.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以得到底部绝对布局的资源id是com.alibaba.android.rimet:id/home\_bottom\_tab\_root,而且这个id是唯一的,也就是说我们只要找到这个节点的资源id,就代表已经进入了钉钉程序的主页了.

具体代码:

```java
String resId="com.alibaba.android.rimet:id/home_bottom_tab_root";
AccessibilityNodeInfo info=getRootInActiveWindow();
List<AccessibilityNodeInfo> list = info.findAccessibilityNodeInfosByViewId(resId);
if(list==null||list.size()==0){
     throw new Exception("已进入app,未找到主页节点");

}

```

##### 4.2.4 进入工作页面

--------
到这一步,我们程序已进入钉钉主页,接下来需要进入考勤打卡所在的工作页面
在底部选项卡中,找到工作按钮布局所在的资源id(com.alibaba.android.rimet:id/home\_bottom\_tab\_button\_work),点击工作页按钮,进入工作页,如下图:

![CB989BA02A7AB74285EA28D92A998E19.jpg](https://upload-images.jianshu.io/upload_images/1978245-c4a1665cd2226120.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体代码

```java
String resId="com.alibaba.android.rimet:id/home_bottom_tab_button_work";
AccessibilityNodeInfo info=getRootInActiveWindow();
List<AccessibilityNodeInfo> list = info.findAccessibilityNodeInfosByViewId(resId);
if(list==null||list.size()==0){
     throw new Exception("已进入主页,未找到工作页按钮");
}else{
	  list.get(0).performAction(AccessibilityNodeInfo.ACTION_CLICK);
}

```

##### 4.2.5 已进入工作页,查找考勤打卡按钮,进行点击操作,进入考勤打卡页面

到这一步,我们程序默认已经在工作页面了,接下来需要做的就是点击考勤打卡选项,进入考勤页面.
这里有些许的复杂,因为不能直接找到考勤打卡所在布局的id,只能先查找其所在的父布局的id(com.alibaba.android.rimet:id/oa\_fragment\_gridview),然后再找到考勤打卡的节点.

![33.jpg](https://upload-images.jianshu.io/upload_images/1978245-b0e83cc15f3f1dbb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


具体代码:

```java
        String resId="com.alibaba.android.rimet:id/oa_fragment_gridview";
        AccessibilityNodeInfo info=getRootInActiveWindow();
        List<AccessibilityNodeInfo> list = info.findAccessibilityNodeInfosByViewId(resId);
        if(list!=null||list.size()!=0){
            AccessibilityNodeInfo node = list.get(0);
            if (node != null || node.getChildCount() >= 8) {
                node = node.getChild(7);
                if (node != null) {  //已找到考勤打卡所在节点,进行点击操作
                    node.performAction(AccessibilityNodeInfo.ACTION_CLICK);
                }else{
                    throw new Exception("已进入工作页,但未找到考勤打卡节点");
                }
            }else{
                throw new Exception("已进入工作页,但未找到考勤打卡节点");
            }
        }else{
            throw new Exception("已进入工作页,但未找到相关节点");
        }
```


##### 4.2.6 确认已考勤打卡页面

到这一步,我们程序认为已经进入了考勤打卡页面了,接下来我们需要再确认一下目前所在节点是不是考勤打卡页面的节点.
这个页面是一个webview页面,所以判断是否已进入考勤打卡界面,我们只要找到了webview布局的一个唯一资源id标识即可(com.alibaba.android.rimet:id/webview_frame),

![55.jpg](https://upload-images.jianshu.io/upload_images/1978245-fff37638fc62fe25.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

代码:

```java
String resId="com.alibaba.android.rimet:id/webview_frame";
AccessibilityNodeInfo info=getRootInActiveWindow();
List<AccessibilityNodeInfo> list = info.findAccessibilityNodeInfosByViewId(resId);
if(list==null||list.size()==0){
     throw new Exception("进入考勤打卡页面异常");
}

```

##### 4.2.7 执行打卡操作
------------

到这一步,程序已确认进入考勤打卡页面,可以开始执行打卡操作.按照我们一些的步骤,打卡操作只需要你找到相应的打卡按钮节点,然后通过节点的点击操作接口,但是很不幸的是,由于考勤打卡页面时webview页面,我们不能定位到详细的打卡按钮所在的节点(准确来说有时可以,有时不可以,而且这情况发生在同一台手机上,差点把小编折腾死,只能用最坏情况操作了),因为我们根本找不到他的资源id,我们唯一能找到的只能是他的父节点(com.alibaba.android.rimet:id/webview_frame),然后并没卵用!

---------
不过方法总是有的!
既然我们不能定位节点,但我们可以定位坐标啊,刚好tap命令可以模拟点击屏幕坐标!!!瞬间感觉自己是个天才!!

-----
我们只需要找到上班打卡和下班打卡两个按钮所在的坐标(不同分辨率的手机会有不同),然后使用adb命令直接模拟点击即可!


点击坐标方法

```java

    public static void clickXy(String x,String y){
        String cmd = "input tap "+x+" "+y ;

        try {

            execRootCmdSilent( cmd);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 执行命令但不关注结果输出
     */
    private  static int execRootCmdSilent(String cmd) {
        int result = -1;
        DataOutputStream dos = null;

        try {
            Process p = Runtime.getRuntime().exec("su");
            dos = new DataOutputStream(p.getOutputStream());

            dos.writeBytes(cmd + "\n");
            dos.flush();
            dos.writeBytes("exit\n");
            dos.flush();
            p.waitFor();
            result = p.exitValue();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (dos != null) {
                try {
                    dos.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return result;
    }

```







##### 4.2.8 确定打卡成功

模拟点击了打卡界面之后,如果操作成功,默认会出现一个打卡成功的弹窗,我们可以根据这个弹窗来判断是否打卡成功

由于这个弹窗也不能找到相关的id的详细节点,而且也不能通过text去查找,所以这里先通过递归方法拿到所有的几点,然后判断每个节点的content-desc是否包含打卡成功的字样,如果有,我们就默认打卡成功!

![66.jpg](https://upload-images.jianshu.io/upload_images/1978245-f61b4cbce3e7b5fd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


首先找出所有节点

```json
//递归获取所有节点
    private List<AccessibilityNodeInfo> getAllNode(AccessibilityNodeInfo node, List<AccessibilityNodeInfo> list) {
        if (list == null) {
            list = new ArrayList<>();
        }
        if (node != null && node.getChildCount() != 0) {
            for (int i = 0; i < node.getChildCount(); i++) {
                AccessibilityNodeInfo info = node.getChild(i);
                if (node != null) {
                    list.add(info);
                    node = info;
                }
            }

        } else {
            return list;
        }
        return getAllNode(node, list);
    }
```

--------

判断节点是否包含打卡成功字样

```java
//检查是否打卡成功
        AccessibilityNodeInfo node = getRootInActiveWindow();

        //查询所有的根节点,假如有弹窗,说明打卡成功
        List<AccessibilityNodeInfo> list = getAllNode(node, null);
        LogUtil.D("所有节点个数-->" + list.size());
        if (list != null) {
            for (AccessibilityNodeInfo info : list) {
                String className = info.getClassName().toString();
                if ("android.app.Dialog".equals(className)) {
                    //说明可能是打卡导致的成功弹窗
                    AccessibilityNodeInfo nodeInfo = info.getChild(0);
                    if (nodeInfo != null) {
                        nodeInfo = nodeInfo.getChild(1);
                        if (nodeInfo != null) {
                            String des = nodeInfo.getContentDescription().toString();
                            if (des.contains("打卡成功")) {
									//这里做你想做的事,比如发个邮件通知一下
                                return;
                            }
                        }
                    }

                }
            }
        }
```
-----------

每次模拟点击之后,都要判断一下是否有打卡成功弹窗,最多尝试10次

```java
//已进入打卡页面,执行打卡操作
        int j = 10;
        while (j >= 0) {
            LogUtil.D("尝试打卡操作->" + j);
            if(DoDaKa(order)){  //这里封装了一下,这是模拟点击之后,判断弹窗打卡成功的方法
                //这里可以发送邮件
                return;
            }
            sleepT(2000);

            j--;
        }

```

##### 4.2.9 异常处理

在上述流程中,基本每一步都抛出了大量异常,出现异常,即代表程序没有按照我们设定的流程走,这时我们就需要去修正.一旦出现异常,我们让脚本回到初始状态,也就是最初的桌面状态.android可以通过回退键来恢复到桌面.

代码:

```java
//程序异常时的操作方法
    private void AppCallBack() {
        int i = 10; //最多尝试10次回退操作
        while (true) {
            //执行回退操作
            AccessibilityNodeInfo node = getRootInActiveWindow();
            if (i < 0) { //10次还未到桌面
                //说明可能卡住了,无法回退,强行停止程序进程
                CMDUtil.stopProcess(node.getPackageName().toString());
                break;
            }
            LogUtil.D("执行回退操作");
            performGlobalAction(AccessibilityService.GLOBAL_ACTION_BACK);
            if (node != null && Comm.launcher_PakeName.equals(node.getPackageName().toString())) {
                //已回退到启动页,退出循环
                LogUtil.D("桌面");
                break;
            }
            i--;
            sleepT(1000); //睡眠一秒
        }

    }
```



TIPS:
上溯所有流程的每一步,我们最好都加上1-2秒的延迟时间,毕竟页面跳转是需要时间的,对于手机性能差的手机相应的时间可以再延迟一些.

--------------

五. 功能测试

到这里,我们的自动打卡程序基本就已经实现了,当然,上面只是实现自动打卡的核心代码.还有很多的拓展空间,比如可以加上一个任务请求线程,实现在特定时间,来实现打上班卡还是打下班卡,以及打卡成功之后及时的邮件通知到手机上.也可以通过服务器来定时启动程序,控制脚本程序啥时候运行,啥时候不运行.发挥你的想象吧!

------



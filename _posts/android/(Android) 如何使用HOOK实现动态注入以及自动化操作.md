
##  (Android) 如何使用HOOK实现动态注入以及自动化操作

===============================================

**目录**
=================
-----------
[TOC]

### 为什么会有这边博文?

最近一直在搞一些apk破解以及自动化方面的东西.觉得有必要记录一下.也是为了修改下自己懒的毛病(很久很久很久没更新博客了)


### 一. 项目需求

#### 项目功能:

本文将使用QQ为例,利用Hook实现自动登录,自动退出登录功能

#### 项目流程:
流程比较简单
![WX20190621-113232@2x.png](https://upload-images.jianshu.io/upload_images/1978245-40bb1d85ed32176b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 二. 资源准备

大致需要准备以下东西:
1. 一台空闲的**安卓**手机,能root最好.
2. 安装VirtualXposed([传送门](https://vxposed.com/)),安装QQ([传送门](https://pan.baidu.com/s/1hk9TG4zEKr7SN-uytsjMqQ),提取码: ty58)(注意QQ是8.0.0.4000版本,不同版本可能会影响hook点),[对于不懂怎么使用hook的请参考](https://www.jianshu.com/p/b1cec452d1b7)
3. 将QQ安装到VirtualXposed
4. QQ脱壳获取源代码([不懂脱壳操作请点这里](https://zhuanlan.zhihu.com/p/45591754))

### 三. 功能实现

####   3.1 实现自动从QQ新用户页跳转到登陆页
第一次启动qq的时候,qq会停留在,显然要实现登陆的话我们需要让其自动进入登陆页面,如图:
![登陆图片.png](https://upload-images.jianshu.io/upload_images/1978245-4d4251e05831372b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 3.1.1 获取新用户页的堆栈信息(手机需要连接电脑)

```java
adb shell dumpsys activity >>输出路径
```

下面是导出的堆栈信息:
![堆栈信息.png](https://upload-images.jianshu.io/upload_images/1978245-88f2b925ae66fb84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以得到登陆界面的**RegisterGuideView**.以及登陆按钮的16进制值**7f0b0da9**

##### 3.1.2 获取登陆按钮逻辑的Hook点

通过**PowerGREP**工具对脱壳后的QQdex源码(脱壳后可得到十几个dex文件)进行字符串搜索找到**RegisterGuideView**所在的dex,然后通过jeb工具反编译该dex, 以下是RegisterGuideView类的源码

```java
package com.tencent.mobileqq.activity.registerGuideLogin;

import android.annotation.SuppressLint;
import android.content.Intent;
import android.graphics.drawable.BitmapDrawable;
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.View$OnClickListener;
import android.view.View;
import android.view.ViewGroup;
import android.widget.Button;
import android.widget.ImageView;
import com.tencent.mobileqq.activity.RegisterPhoneNumActivity;
import com.tencent.mobileqq.app.QQAppInterface;
import com.tencent.mobileqq.statistics.ReportController;
import com.tencent.qphone.base.util.QLog;

public class RegisterGuideView extends GuideBaseFragment implements View$OnClickListener {
    private View a;
    private Button a;
    private Button b;

    public RegisterGuideView() {
        super();
    }

    @SuppressLint(value={"ValidFragment"}) public RegisterGuideView(QQAppInterface arg1) {
        super(arg1);
    }

    public void onClick(View arg14) {
        Intent v0;
        switch(arg14.getId()) {
            case 2131430824: {
                ReportController.b(this.a, "CliOper", "", "", "0X8007576", "0X8007576", 0, 0, "", "", "", "");
                v0 = new Intent(this.a, RegisterPhoneNumActivity.class);
                v0.putExtra("key_register_from", 2);
                v0.putExtra("leftViewText", this.a.getString(2131499116));
                v0.addFlags(67108864);
                this.a.startActivity(v0);
                break;
            }
            case 2131430825: {
                ReportController.b(this.a, "CliOper", "", "", "0X8007575", "0X8007575", 0, 0, "", "", "", "");
                v0 = this.a.getIntent();
                v0.putExtra("from_register_guide", true);
                v0.putExtra("is_need_show_logo_animation", true);
                GuideBaseFragment v0_1 = GuideHandler.a(this.a, this.a);
                if(this.a == null) {
                    return;
                }

                this.a.a(v0_1);
                break;
            }
        }
    }

    public View onCreateView(LayoutInflater arg7, ViewGroup arg8, Bundle arg9) {
        View v1 = arg7.inflate(2130903607, arg8, false);
        this.a = v1.findViewById(2131428759);
        this.a.setVisibility(0);
        this.a = v1.findViewById(2131430825);
        this.b = v1.findViewById(2131430824);
        this.a.setOnClickListener(((View$OnClickListener)this));
        this.b.setOnClickListener(((View$OnClickListener)this));
        View v0 = v1.findViewById(2131430823);
        try {
            ((ImageView)v0).setImageDrawable(new BitmapDrawable(this.getResources(), this.getActivity().getAssets().open("splash.jpg")));
        }
        catch(Throwable v0_1) {
            QLog.e("LoginActivity.RegisterGuideView", 1, "onCreateView error:" + v0_1.getMessage());
        }

        return v1;
    }
}


```

通过我们得到的登陆按钮的16进制值7f0b0da9可以确定10进制值2131430825,从而确定变量a是登陆按钮.

##### 3.1.3 通过Hook 实现登陆按钮点击

由于我们已经找到了登陆按钮相关的类和按钮变量对象,接下来我们只需要通过hook拿到该按钮对象,实现点击即可自动进入登陆页面
以下是核心代码:
```java
/**
     * hook新用户页,拿到登陆按钮,模拟点击登陆
     */
    public static void hookRegisterGuideLogin()
    {
        XposedHelpers.findAndHookMethod("com.tencent.mobileqq.activity.registerGuideLogin.RegisterGuideView", context.getClassLoader(), "onCreateView", LayoutInflater.class, ViewGroup.class, Bundle.class, new XC_MethodHook()
        {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable
            {
                super.afterHookedMethod(param);

                final Object obj = param.thisObject;

                
                doMainLooper(context, 1000, new Runnable()
                {
                    @Override
                    public void run()
                    {
                        //得到按钮对象
                        Object viewObj = ReflectUtils.getFieldValueForFiledClsNm(obj, "a", "android.widget.Button");

                        if (viewObj instanceof Button)
                        {
                            //点击按钮
                            ((Button) viewObj).performClick();
                        }
                    }
                });

            }
        });
    }
```

目前为止,我们就已经完成了长征的第一步,从新用户页自动点击到登陆页. 下面是成果演示:

![前往登陆界面.gif](https://upload-images.jianshu.io/upload_images/1978245-d0d3cd5a6b027090.gif?imageMogr2/auto-orient/strip)

可以看到我们已经实现了从新用户页点击到登陆页了.

#### 3.2 实现登陆页自动填充登陆账号和密码并自动登陆

##### 3.2.1 打印登陆页的堆栈信息
如下图,我们可以找到登陆页面所在的类名,以及按钮输入框等信息
![登陆图片-1.png](https://upload-images.jianshu.io/upload_images/1978245-7ed1376083ca3bde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 3.2.2 找到源码中登陆相关hook点

LoginView的重要源码如下,我们基本可以定位到账号输入框,密码输入框,登陆按钮等对象

```java
private void a(View arg12) {
        int v0_6;
        ImageView v1_1;
        int v3;
        int v1;
        int v10 = -16578534;
        int v9 = 2;
        int v8 = 8;
        this.i = arg12.findViewById(2131430866);
        arg12.findViewById(2131438684).setVisibility(v8);
        this.a = arg12.findViewById(2131430867);
        this.a = arg12.findViewById(2131430735);
        this.a.setHeadBorder(2130839513);
        this.a = this.a.a();
        this.a.a = ((NewStyleDropdownView$DropdownCallback)this);
        this.a.setContentDescription(this.a.getString(2131492973));
        Bundle v0 = this.a.getInputExtras(true);
        if(v0 != null) {
            v0.putInt("INPUT_TYPE_ON_START", 1);
        }

        this.a = arg12.findViewById(2131430736);
        this.a.setCustomClearButtonCallback(new abgm(this, arg12.findViewById(2131430871), ((int)(43f * this.a + 0.5f))));
        this.a.setContentDescription(this.a.getString(2131492974));
        SpannableString v0_1 = new SpannableString("输入密码");
        v0_1.setSpan(new AbsoluteSizeSpan(17, true), 0, v0_1.length(), 33);
        this.a.setHint(((CharSequence)v0_1));
        if(Build$VERSION.SDK_INT >= 26) {
            try {
                View.class.getMethod("setImportantForAutofill", Integer.TYPE).invoke(this.a, Integer.valueOf(8));
            }
            catch(Exception v0_2) {
                QLog.w("LoginActivity.LoginView", v9, "disable auto fill error", ((Throwable)v0_2));
            }
        }

        this.a = arg12.findViewById(2131430150);
        this.a.setContentDescription(this.a.getString(2131492976));
        arg12.findViewById(2131430876).setOnClickListener(((View$OnClickListener)this));
        View v0_3 = arg12.findViewById(2131430875);
        ViewCompat.setImportantForAccessibility(v0_3, v9);
        v0_3.setContentDescription(this.a.getString(2131498620) + this.a.getString(2131500954));
        this.a.setOnClickListener(((View$OnClickListener)this));
        this.a.a();
        this.g();
        this.a = arg12.findViewById(2131430740);
        this.a.setContentDescription(this.a.getString(2131492978));
        this.a.setOnClickListener(((View$OnClickListener)this));
        this.a = arg12.findViewById(2131430864);
        this.f = arg12.findViewById(2131430874);
        this.g = arg12.findViewById(2131430733);
        this.a.setOnSizeChangedListenner(((InputMethodRelativeLayout$onSizeChangedListenner)this));
        this.a.setOnTouchListener(((View$OnTouchListener)this));
        this.e = arg12.findViewById(2131430865);
        this.e.setOnTouchListener(new abgn(this));
        this.a = arg12.findViewById(2131430868);
        if(this.j) {
            this.a.setVisibility(4);
        }

        this.b = arg12.findViewById(2131430738);
        this.b = arg12.findViewById(2131430739);
        this.b.setContentDescription(this.a.getString(2131498674));
        arg12.findViewById(2131438685).setOnClickListener(((View$OnClickListener)this));
        this.a = this.a.getSystemService("input_method");
        this.b = this.a.a();
        this.b.setOnClickListener(this.a);
        if(this.a == null) {
            this.a = new ArrayList();
            goto label_183;
        }

        try {
            this.a.clear();
        }
        catch(Exception v0_2) {
            QLog.d("LoginActivity.LoginView", 1, "initViews crash: ", ((Throwable)v0_2));
            this.a = new ArrayList();
        }

    label_183:
        List v0_4 = BaseApplicationImpl.sApplication.getAllAccounts();
        if(v0_4 != null) {
            this.a.addAll(((Collection)v0_4));
        }

        if(this.a != null) {
            if(this.a.size() <= 0) {
                goto label_470;
            }

            while(this.a.size() > v8) {
                this.a.remove(this.a.size() - 1);
            }

            this.a.setAdapter(new abgo(this, this.a));
            if((this.g) && !this.f) {
                goto label_273;
            }

            if(this.i) {
                goto label_273;
            }

            if(this.k) {
                goto label_273;
            }

            String v4 = this.a.getIntent().getStringExtra("uin");
            String v5 = this.a.getIntent().getStringExtra("befault_uin");
            if((this.f) && v4 != null && v4.length() > 0) {
                v1 = 0;
                v3 = -1;
            }
            else {
                this.a(this.a.get(0));
                this.a = 0;
                goto label_273;
            }

            while(v1 < this.a.size()) {
                Object v0_5 = this.a.get(v1);
                if(v0_5 != null && ((SimpleAccount)v0_5).getUin() != null) {
                    if(v5 != null && (v5.equals(((SimpleAccount)v0_5).getUin()))) {
                        v3 = v1;
                    }

                    if(!v4.equals(((SimpleAccount)v0_5).getUin())) {
                        goto label_255;
                    }

                    this.a(((SimpleAccount)v0_5));
                    this.a = v1;
                }

            label_255:
                ++v1;
            }

            if(v3 == -1) {
                goto label_273;
            }

            this.a.remove(v3);
        }
        else {
        label_470:
            this.a.b().setVisibility(v8);
        }

    label_273:
        this.a.addTextChangedListener(this.a);
        this.a.addTextChangedListener(this.b);
        this.a.setOnFocusChangeListener(this.a);
        this.a.setOnFocusChangeListener(this.a);
        this.a.setLongClickable(false);
        this.c = arg12.findViewById(2131430872);
        this.c.setOnClickListener(((View$OnClickListener)this));
        if(this.a) {
            this.a.setTransformationMethod(PasswordTransformationMethod.getInstance());
            v1_1 = this.c;
            v0_6 = (this.g) || (this.f) || (this.i) ? 2130844987 : 2130842965;
            v1_1.setImageResource(v0_6);
            this.c.setContentDescription("隐藏密码");
        }
        else {
            this.a.setTransformationMethod(HideReturnsTransformationMethod.getInstance());
            v1_1 = this.c;
            v0_6 = (this.g) || (this.f) || (this.i) ? 2130844988 : 2130842968;
            v1_1.setImageResource(v0_6);
            this.c.setContentDescription("显示密码");
        }

        this.c.setVisibility(v8);
        this.b.setOnClickListener(((View$OnClickListener)this));
        if(this.a.mSystemBarComp != null && ImmersiveUtils.isSupporImmersive() == 1) {
            this.a.mSystemBarComp.init();
        }

        this.c = arg12.findViewById(2131430734);
        this.d = arg12.findViewById(2131430869);
        this.a.clearFocus();
        this.a.clearFocus();
        this.a.setClearButtonVisible(false);
        this.a.setTextClearedListener(((ConfigClearableEditText$OnTextClearedListener)this));
        this.a.addTextChangedListener(this.c);
        this.b(arg12);
        if(this.a.getIntent().getBooleanExtra("reason_for_upgrade", false)) {
            this.a.showDialog(v9);
        }

        if((this.a.getIntent().getBooleanExtra("key_req_by_contact_sync", false)) && (this.a.getIntent().getBooleanExtra("IS_ADD_ACCOUNT", false))) {
            this.a.setText(this.a.getIntent().getStringExtra("key_uin_to_login"));
        }

        this.a.setVisibility(0);
        this.i.setVisibility(v8);
        this.f.setVisibility(0);
        this.b.setVisibility(0);
        this.a.a(false, null);
        this.a.setTextColor(v10);
        this.a.setHintTextColor(-5196865);
        this.a.setFocusable(true);
        this.a.setFocusableInTouchMode(true);
        this.a.setTextColor(v10);
        this.a.setHintTextColor(-5196865);
        this.a.setVisibility(0);
        this.b.findViewById(2131430877).setVisibility(0);
        this.a.setVisibility(0);
        this.b.setVisibility(0);
        this.d(this.a.isInMultiWindow());
        if((this.g) || (this.f) || (this.i)) {
            this.c.setVisibility(0);
            this.a.setVisibility(v8);
            this.i.setVisibility(0);
            this.i.setOnClickListener(((View$OnClickListener)this));
            if(this.i) {
                String v0_7 = this.a.getIntent().getStringExtra("uin");
                if(!TextUtils.isEmpty(((CharSequence)v0_7))) {
                    v1 = v0_7.length();
                    this.a.setText(v0_7.substring(0, v9) + "****" + v0_7.substring(v1 - 2, v1));
                    this.b(v0_7);
                    this.a.setFocusable(false);
                    this.a.setFocusableInTouchMode(false);
                }

                this.b.findViewById(2131430877).setVisibility(v8);
                this.a.setVisibility(v8);
                RelativeLayout$LayoutParams v0_8 = new RelativeLayout$LayoutParams(-2, -2);
                v0_8.addRule(13);
                this.b.setLayoutParams(((ViewGroup$LayoutParams)v0_8));
                goto label_454;
            }

            this.a.a(false, null);
            this.a.setFocusable(true);
            this.a.setFocusableInTouchMode(true);
        }
        else {
            if(!this.l && !this.a.isInMultiWindow()) {
                goto label_454;
            }

            this.c.setVisibility(0);
        }

    label_454:
        this.a = this.a.getLayoutParams();
        this.b = this.g.getLayoutParams();
        this.c = this.a.getLayoutParams();
        this.e();
    }
```

####  3.2.3 使用hook实现自动登陆

以下是核心代码

```java
private static void hookAutoLogin()
    {
        String loginviewClsNm = "com.tencent.mobileqq.activity.registerGuideLogin.LoginView";
        final String loginBtnClsNm = "com.tencent.mobileqq.activity.registerGuideLogin.LoginAnimBtnView"; //登陆按钮类名
        final String loginBtnFiledNm = "a"; //登陆按钮变量名
        final String pwdEdTextClsNm = "com.tencent.mobileqq.widget.CustomSafeEditText"; //密码输入框类名
        final String pwdEdTextFiledNm = "a"; //密码输入框变量名
        final String newStyleDropdownViewClsNm = "com.tencent.mobileqq.widget.NewStyleDropdownView"; //输入框外view类名
        final String newStyleDropdownViewFiledNm = "a"; //输入框外view类名
        final String actInputViewClsNm = "aqsw"; //输入框类名
        final String actInputViewFiledNm = "a"; //输入框的变量名

        //hook登录页节点
        XposedHelpers.findAndHookMethod(loginviewClsNm, context.getClassLoader(), "a", View.class, new XC_MethodHook()
        {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable
            {
                super.afterHookedMethod(param);

                //得到登陆所需对象
                Object loginViewObj = param.thisObject;
                Object newStyleDropdownViewOBj = ReflectUtils.getFieldValueForFiledClsNm(loginViewObj, newStyleDropdownViewFiledNm, newStyleDropdownViewClsNm);
                final Object loginBtnObj = ReflectUtils.getFieldValueForFiledClsNm(loginViewObj, loginBtnFiledNm, loginBtnClsNm);
                final Object pwdEdTextObj = ReflectUtils.getFieldValueForFiledClsNm(loginViewObj, pwdEdTextFiledNm, pwdEdTextClsNm);
                final Object actInputViewobj = ReflectUtils.getFieldValueForFiledClsNm(newStyleDropdownViewOBj, actInputViewFiledNm, actInputViewClsNm);


                // 开始输入账号密码登陆

                final String account="";
                final String pwd="";

                doMainLooper(context, 3000, new Runnable()
                {
                    @Override
                    public void run()
                    {

                        if (actInputViewobj instanceof AutoCompleteTextView)
                        {

                            ((AutoCompleteTextView) actInputViewobj).setText(account);
                        }

                        if (pwdEdTextObj instanceof EditText)
                        {

                            ((EditText) pwdEdTextObj).setText(pwd);
                        }
                        if (loginBtnObj instanceof View)
                        {

                            ((View) loginBtnObj).performClick();
                        }


                    }
                });


            }
        });
    }

```

至此我们的自动登陆功能就算是做好了,下面是演示:

![自动登陆.gif](https://upload-images.jianshu.io/upload_images/1978245-99d57ac16b5d1909.gif?imageMogr2/auto-orient/strip)



#### 3.3 实现退出登录

和上面步骤一样,我们需要从账号管理界面界面入手

![账号管理.png](https://upload-images.jianshu.io/upload_images/1978245-29d6528c664fab0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



##### 3.3.1 获取堆栈信息

拿到堆栈信息如下,找到所在的类AccountManageActivity

![账号管理堆栈信息.png](https://upload-images.jianshu.io/upload_images/1978245-cee74c4451c0040a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 3.3.2 查找源码中退出登录的相关hook点

```java
public void a(int arg11, boolean arg12) {
        String v9 = null;
        int v5 = 7000;
        int v6 = 2;
        Object v0 = this.a.get(arg11);
        if(v0 == null) {
            this.a.dismiss();
            if(QLog.isColorLevel()) {
                QLog.w("Switch_Account", v6, "onItemLongClick simple account = null");
            }
        }
        else {
            String v2 = ((SimpleAccount)v0).getUin();
            String v3 = this.app.getCurrentAccountUin();
            this.a = v2;
            if(v2.equals(v3)) {
                AccountManageActivity.a(((Activity)this), this.app);
            }

            if(QLog.isColorLevel()) {
                QLog.d("hunter", v6, "++++++++++");
            }

            this.a(this.a, arg12);
            HistoryChatMsgSearchKeyUtil.a(v2);
            CrmUtils.a(this.getBaseContext(), v3);
            this.a.remove(v0);
            Manager v1 = this.app.getManager(60);
            if(v1 != null && (((SubAccountManager)v1).a(v2))) {
                SubAccountControll.a(this.app, 0, v2);
                ((SubAccountManager)v1).e(v2);
                ((SubAccountManager)v1).a(v2, v9, true);
                ((SubAccountManager)v1).a(v2, v6);
                SubAccountControll.a(this.app, v2, 7);
                int v1_1 = 1 - this.app.a().a(v2, v5);
                if(v1_1 != 0) {
                    this.app.a().c(v2, v5, v1_1);
                }

                if(!QLog.isColorLevel()) {
                    goto label_68;
                }

                QLog.d("SUB_ACCOUNT", v6, "deleteAccount() hint need to verify,msg num=1, subUin=" + v2);
            }

        label_68:
            GesturePWDUtils.clearGestureData(this.getActivity(), ((SimpleAccount)v0).getUin());
            if(v2.equals(v3)) {
                this.app.getApplication().refreAccountList();
                List v0_1 = this.getAppRuntime().getApplication().getAllAccounts();
                if(v0_1 != null && v0_1.size() > 0) {
                    v0 = v0_1.get(0);
                    if(((SimpleAccount)v0).isLogined()) {
                        this.getAppRuntime().startPCActivePolling(((SimpleAccount)v0).getUin(), "delAccount");
                    }
                }
            }

            ThreadManager.post(new unx(this, v2, arg12, arg11), 8, ((ThreadExcutor$IThreadListener)v9), true);
        }
    }
```


##### 3.3.3 hook注入退出方法,实现自动退出登录

以下是核心代码:

```java
 private static void loginOut()
    {
        //hook 账号管理页
        XposedHelpers.findAndHookMethod(accountManageActivityClsNm, context.getClassLoader(), "a", Bundle.class, new XC_MethodHook()
        {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable
            {
                super.afterHookedMethod(param);
                final Object accountManageActivityObj = param.thisObject;

                doMainLooper(context, 3000, new Runnable()
                {
                    @Override
                    public void run()
                    {
                        ReflectUtils.invokeMethod(accountManageActivityObj, "a", new Class[]{int.class, boolean.class}, 0, true);


                    }
                });

            }
        });
    }
```



###  四. 最终效果

已经实现了完全的自动登录,自动退出登录操作,完整展示:

![退出登录.gif](https://upload-images.jianshu.io/upload_images/1978245-050585bf12f39e05.gif?imageMogr2/auto-orient/strip)


由于篇幅原因,部分代码已省略,完整代码已上传github([传送门](https://github.com/momoxiaoming/QQAutoLoginDemo/tree/master))

实际上,利用hook动态注入可以无所不能,大家可以发挥想象.(不过希望用在正途哦,手动滑稽)
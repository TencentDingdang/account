# 一、概述
接入腾讯云小微的设备端，需要通过用户授权，获取云小微访问令牌(TVSToken)，该令牌随每个请求一起发送到云小微服务。为了完成用户授权，我们在云小微设备平台上搭建了账号体系。
设备端需要接入云小微账号，实现用户授权。

另外，腾讯云小微与QQ音乐深度合作，您可以使用QQ音乐授权功能，绑定用户QQ音乐账号到云小微账号，实现QQ音乐账号资源互通，体验深度定制的音乐服务。
<br/>
# 二、账号接入
云小微账号体系支持以下几种账号接入方式：
|接入方式|描述|
|-|-|
|QQ/微信账号|通过QQ/微信客户端授权云小微访问用户信息，获取账号。|
|访客账号|通过云小微SDK自动获取设备信息，生成唯一账号，实现授权。此方式账号与设备唯一对应。|

您可以通过开发伴生APP，利用适用于Android或iOS的 **DMSDK** 来实现用户账户授权。您的伴生APP负责获取授权码(ClientId)并将其安全地传输到您的设备端。您的设备端负责使用ClientId获取访问和刷新令牌，用于调用云小微服务。
如果您的设备不适用伴生APP控制，也可以直接在设备端以访客身份获取访问和刷新令牌，调用云小微服务。

## 1  QQ/微信账号
### 1.1 前期准备

#### ① 在 QQ互联平台/微信开放平台 申请 appId

> QQ互联平台：[https://connect.qq.com/](https://connect.qq.com/) 
> 微信开放平台：[https://open.weixin.qq.com/](https://open.weixin.qq.com/)

#### ② 在 [云小微开放平台 - 设备平台](https://dingdang.qq.com/tvs#/projects) 打开相关应用，选择接入方案，填写上一步申请的 AppId/AppSecret
![](https://3gimg.qq.com/trom_s/dingdang/upload/20190909/a5206e1ca5d63999617bf32ced7d0cee.png)

### <a name="1.2"></a>1.2 TVS设备SDK方案
授权流程如下：
![](https://3gimg.qq.com/trom_s/dingdang/upload/20190909/a47ccbfbbc1e34e20d78a80116edc127.jpg)
#### ① 建立手机APP与智能设备间的传输通道

> 常用设备间信息传递方案有：扫码、wifi热点、蓝牙等，厂商需自行选择方案建立通道

#### ② 将智能设备DSN、ProductId信息传递到APP

> Product ID： 在 [云小微开放平台 - 设备平台](https://dingdang.qq.com/tvs#/projects) 打开相关应用，**应用概览** 部分查看  
> DSN： 设备唯一序列码，需保证每个设备不同，具体规则由厂商自行制定

#### ③ 判断QQ/微信是否已授权，若未授权，拉起QQ/微信登录、授权

> 此步需保证QQ/微信已安装

Android：

	LoginProxy loginProxy = LoginProxy.getInstance();
    if(!loginProxy.isTokenExist()){
        loginProxy.tvsLogin(ELoginPlatform.WX, null, new TVSCallback(){
            @Override
            public void onSuccess() {
	            //授权成功
            }
            @Override
            public void onError(int i) {
	            //授权失败
            }
        });
    }
iOS：

```
[[TVSAuthManager shared]wxLoginWithViewController:self handler:^(TVSAuthResult result){
    if (result == TVSAuthResultSuccess) {
        // 授权成功
    } else {
        // 授权失败
    }
}];
```

#### <a name="1.24"></a>④ 验证QQ/微信票据有效性，若无效，需跳转到第③步重新登录。      

Android：

    loginProxy.tvsTokenVerify(new TVSCallback() {
        @Override
        public void onSuccess() {
	        //票据有效
        }
        @Override
        public void onError(int i) {
	        //票据无效
        }
    });
iOS：

```
[[TVSAuthManager shared]wxTokenRefreshWithHandler:^(TVSAuthResult result){
    if (result == TVSAuthResultSuccess) {
        // 票据有效
    } else {
        // 票据无效
    }
}];
```

#### ⑤ 传入ProductId/DSN绑定账号及设备

Android：

    TVSDevice tvsDevice = new TVSDevice();
    tvsDevice.productID = "此处填写Product ID";
    tvsDevice.dsn = "此处填写DSN";
    // bindType、pushIDExtra填写内容请参考DMSDK接口文档，此处仅作示例！
    tvsDevice.bindType = TVSDeviceBindType.TVS_SPEAKER;
    tvsDevice.pushIDExtra = "TVSSpeaker";
    loginProxy.bindPushDevice(tvsDevice, new TVSCallback() {
        @Override
        public void onSuccess() {
        }
        @Override
        public void onError(int i) {
        }
    });
iOS：

```
TVSDeviceInfo* device = [TVSDeviceInfo new];
device.productId = @"此处填写Product ID";
device.dsn = @"此处填写DSN";
// bindType、pushIDExtra填写内容请参考DMSDK接口文档，此处仅作示例！
device.bindType = TVSDeviceBindTypeTVSSpeaker;
device.pushIdExtra = PUSH_ID_EXTRA_TVS_SPEAKER;
[[TVSDeviceManager shared]bindDevice:device handler:^(BOOL success) {
    if (success) {
        // 绑定成功
    } else {
        // 绑定失败
    }
}];
```

#### ⑥ 获取ClientId
Android：
```
AccountInfoManager.getInstance().getClientId("Product ID", "DSN")
```
iOS：

```
TVSAccountInfo* ai = [TVSAuthManager shared].accountInfo;
NSString* clientId = [TVSAccountInfo clientIdWithProductId:@"Product ID" dsn:@"DSN"];
```

#### ⑦ 传递ClientId到智能设备
#### ⑧ 智能设备调用TVS SDK，传入ClientId，完成云端鉴权
```
TVSApi.getInstance().getAuthManager().setClientId("ClientId"); 
```

### 1.3 TVS云端API方案

授权流程如下：
![](https://3gimg.qq.com/trom_s/dingdang/upload/20191125/cb273b722874ce95218fe8778d0a06d1.png)

#### ①-⑦伴生APP授权绑定设备
同 <a href="#1.2">**[1.2 TVS设备SDK方案 ①-⑦]**</a>

#### <a name="1.3-⑧⑨"></a>⑧⑨设备鉴权，获取**访问票据**

 **请求URL：** 
> 正式环境：POST https://tvs.html5.qq.com/auth/o2/token
> 测试环境：POST https://tvstest.html5.qq.com/auth/o2/token

**请求参数：**

```
{
	"grant_type":"authorization_code",
	"client_id":"{{STRING}}",
	"code":"authCode",
	"code_verify":"{{STRING}}",
	"redirect_uri":"redirectUri"
}
```

参数名|类型|是否必选|描述
-|-|-|-
grant_type|string|是|固定为`authorization_code`
client_id|string|是|上文获取的ClientId
code|string|是|固定为`authCode`
code_verify|string|是|固定为`codeVerify`
redirect_uri|string|是|固定为`redirectUri`

**返回参数：**

```
{
	"token_type":"Tvser",
	"access_token":"{{STRING}}",
	"refresh_token":"{{STRING}}",
	"expires_in":{{LONG}}
}
```

参数名|类型|描述
-|-|-
token_type|string|固定为`Tvser`
access_token|string|**访问票据**。设备端调用TVS云端API需要携带该票据，为空时表示授权失败
refresh_token|string|**刷新票据**。用于刷新**访问票据**
expires_in|string|**访问票据**过期时间，单位：秒

#### <a name="1.3-⑩⑪"></a>⑩⑪定时刷票，根据获取到的票据过期时间，及时更新票据

**请求URL：**

> 正式环境：POST https://tvs.html5.qq.com/auth/o2/token 
> 测试环境：POST https://tvstest.html5.qq.com/auth/o2/token

**请求参数：**
```
{
	"grant_type":"refresh_token",
	"refresh_token":"{{STRING}}",
	"client_id":"{{STRING}}"
}
```
参数名|类型|是否必选|描述
-|-|-|-
grant_type|string|是|固定为`refresh_token`
refresh_token|string|是|上一步获取到的refresh_token
client_id|string|是|上文获取的ClientId

**返回参数：**

```
{
	"token_type":"Tvser",
	"access_token":"{{STRING}}",
	"refresh_token":"{{STRING}}",
	"expires_in":{{LONG}}
}
```
参数名|类型|描述
-|-|-
token_type|string|固定为`Tvser`
access_token|string|**访问票据**
refresh_token|string|**刷新票据**
expires_in|string|**访问票据**过期时间，单位：秒

#### ⑫带着访问票据，请求tvs云端API接口
[云端API接口文档](https://dingdang.qq.com/doc/page/285)


## 2 访客账号
访客账号适用于不需要伴生APP的智能设备。接入方根据设备信息，生成唯一的账号信息，绑定当前设备。

**优点：**
>  1、接入简单 
>  2、不需要伴生APP 

**缺点：** 
> 1、账号唯一绑定设备，无法解绑、关联其它设备。 
> 2、部分账号相关的技能不可用。
> 3、无法使用DMSDK特有功能：多端互动、TTS音色设置、闹钟管理等。

### 2.1 TVS设备SDK方案
设备端直接调用TVS SDK 完成访客账号绑定
```
TVSApi.getInstance().getAuthManager().setGuestClientId();
```

### 2.2 TVS云端API方案
授权流程如下：
![](https://3gimg.qq.com/trom_s/dingdang/upload/20191126/6f563cdea941745f70058bcbf49bf2f3.png)

#### ① 生成ClientId

> Product ID： 在 [云小微开放平台 - 设备平台](https://dingdang.qq.com/tvs#/projects) 打开相关应用，**应用概览** 部分查看
> DSN：设备唯一序列码，需保证每个设备不同，具体规则由厂商自行制定

```
// Product ID  
string strProductId;
//DSN
string strDsn;
//拼接文本
string strPlain = strProductId + strDsn;

/*生成终端ClientId
"ENCRYPT:0001,"，"0001"，"MD5"是固定值，不能改变（包括大小写）
md5表示采用MD5算法，生成32字节的字符串
upper表示将字符串转换成大写
*/
string strClientId = "ENCRYPT:0001," + upper(md5(upper(md5(strPlain + "0001")) + "MD5")) + "," + strProductId + "," + strDsn;
```

#### ②③设备鉴权，获取tvsRefreshToken/authorization
同 <a href="#1.3-⑧⑨">**[1.3 TVS云端API方案 - ⑧⑨]**</a>

#### ⑤⑥定时刷票，根据获取到的票据过期时间，及时更新票据
同 <a href="#1.3-⑩⑪">**[1.3 TVS云端API方案 - ⑩⑪]**</a>

#### ⑦带着授权票据，请求tvs云端API接口
[云端API接口文档](https://dingdang.qq.com/doc/page/285)

<br/>

# 三、音乐授权
云小微无法通过现有账号体系直接获取用户的QQ音乐账号。因此，用户使用云小微访问QQ音乐时，需通过QQ音乐账号授权操作，绑定音乐账号到云小微账号。

**重要：实现音乐授权前，请务必先接入云小微账号！**

QQ音乐授权支持以下方式：
授权方式|描述
-|-
扫码授权|适用于有屏设备，设备端打开音乐授权二维码页面，用户通过QQ/微信客户端扫码授权
QQ音乐授权|适用于无屏设备，手机App授权，需下载QQ音乐App，通过拉起QQ音乐APP实现授权

## 1 扫码授权
### 1.1 授权流程
![音乐扫码授权.png](https://3gimg.qq.com/trom_s/dingdang/upload/20191121/7ce3c94b742c6ca46e2bebe37fb875bc.png)

#### ① 获取TVSToken
调用TVS SDK获取 TVSToken

```
TVSApi.getInstance().getAuthManager().getAccessToken();
```
#### ② TVSToken 传入 WebviewSDK
引入TVSWebView组件

```
<com.tencent.ai.tvs.web.TVSWebView />
```
传入TVSToken
```
tvsWebView.getController().setTVSToken("TVSToken");
```
#### ③ 传入音乐授权页面url，打开授权页面
```
tvsWebView.getController().loadURL("https://ddsdk.html5.qq.com/v2/opt/music/login");
```

**音乐授权页面url：**
> 正式环境：https://ddsdk.html5.qq.com/v2/page/qqmusic_qrcode
> 体验环境：https://ddsdkgray.html5.qq.com/v2/page/qqmusic_qrcode
> 测试环境：https://sdk.sparta.html5.qq.com/v2/page/qqmusic_qrcode

#### ④ 打开手机QQ音乐客户端，扫码授权

## 2 拉起QQ音乐APP授权

### 2.1 前期准备

#### ① 申请QQ音乐Appid
[申请流程](https://github.com/TencentDingdang/QQMusic/blob/master/QQ%E9%9F%B3%E4%B9%90AppId%E7%94%B3%E8%AF%B7.pdf)

### 2.2 授权流程
![](https://3gimg.qq.com/trom_s/dingdang/upload/20191115/703294562e46d15bf3a24dfa24b7954f.png)


#### ① 创建授权业务类，注册到SDK
创建类 QQMusicAuthAgent (类名可自定义)，实现 CpAuthAgent 接口，并实例化对象注册到SDK
Android：
```
 // 初始化DMSDK
LoginProxy.getInstance().registerApp(this, "wx-app-id", "qq-app-id");
// 使WebViewSDK具备第三方CP授权能力
TVSThirdPartyAuth.setupWithWebViewSDK();
// 在这里将您的实现注入到TSKM模块，注意参数中填入您申请的QQ音乐AppID、密钥和配置对应的回调URL
TVSThirdPartyAuth.setCpAuthAgent(ThirdPartyCp.QQ_MUSIC, new QQMusicAuthAgent());
```
#### ② 刷新QQ/微信票据，确保票据有效
参见 <a href="#1.2">**[QQ/微信票据验证]**</a>

#### ③ 拉起授权H5页面
Android：

```
tvsWebView.getController().loadPresetURLByPath(TVSThirdPartyAuth.getPresetUrlPathByCp(ThirdPartyCp.QQ_MUSIC));
```
#### ④ 用户点击授权页面授权按钮后，SDK会回调 QQMusicAuthAgent对象相关方法检查/下载QQ音乐
在类 QQMusicAuthAgent 中实现以下方法：
Android：

```
boolean checkCpAppInstallation(){
	// 判断QQ音乐是否已经安装、以及版本是否满足要求
	//若QQ音乐已经安装且版本满足则返回true，否则返回false
}

void jumpToAppDownload(){
	// 引导用户下载QQ音乐
	// 当checkCpAppInstallation方法返回false时调用该方法
}
```
#### ⑤ 当QQ音乐版本满足要求时，SDK会回调 QQMusicAuthAgent对象相关方法请求授权
在类 QQMusicAuthAgent 中实现以下方法：
Android：

```
void requestCpAuthCredential(ThirdPartyAuthCallback callback){
  // ⑥ 拉起QQ音乐APP请求授权，接入方需自行实现该逻辑。
	
  ......
  
  // ⑦ 回调授权信息
  if(success) {
	  // 若授权成功，需回调授权信息
	  callback.onSuccess(credential);
  } else {
	  // 若授权失败，需回调错误码、错误描述
      callback.onSuccess(errCode, errMessage);
  }
		    
}
```
#### ⑧ 授权结果回调
添加监听器，监听回调结果
Android：
```
tvsWebView.getController().setBusinessEventListener(new TVSWebController.BusinessEventListener() {
	@Override
    public void onReceiveProxyData(JSONObject data) {
	// 监听data，当授权成功时返回以下格式数据：
	//{
        //    "action": "cpAuthResult", 
        //    "data": {
        //        "cp": "qq_music",
        //        "code": 0   //0标识成功，错误码参见 CpAuthError
        //    }
        //}
    }
});
```
### 2.2 获取授权状态
查询音乐授权状态，需要调用TVS uniAccess接口。
Android：

```
TVSTSKM#requestTSKMUniAccess(productID, dsn, guid, domain, intent, blobInfo, callback)
```
使用方法参见DMSDK接口文档


## 3 音乐服务接入

音乐服务接入详见：[https://dingdang.qq.com/doc/page/248](https://dingdang.qq.com/doc/page/248)
<br/>

# 四、附录
## <a name="qua"></a>QUA

QUA是用于标识客户端信息的key-value对，key-value之间以&连接，服务端可根据QUA信息给出响应的适配内容。
终端每次请求腾讯后台时，都需要在请求结构体中带上QUA信息。

**字段说明：**

属性|类型|默认值|必填|说明
-|-|-|-|-
QV|int|3|是|QUA版本号，**默认填3，不能更改**
VN|string||是|终端版本号，格式必须为四段：[ **主版本.子版本.修正版本.Build** ]，且新版本的版本号必须比旧版本大（按字母排序）。例如：1.0.1.1000。
PP|string||是|终端软件包名，例如：com.tencent.ai.tvs。
VE|string||否|终端版本名。 P - 预览版；GA - 正式版；RC - 发布候选；BN - BetaN
CHID|int||否|渠道号，用于区分不同的渠道，如：线上渠道，线下渠道。

>**示例:** QV=3&VE=GA&VN=1.0.1000&PP=com.tencent.ai.tvs&CHID=10020

## SDK下载链接： 
> **DMSDK：**[https://github.com/TencentDingdang/dmsdk](https://github.com/TencentDingdang/dmsdk)
> **WebViewSDK：**[https://github.com/tencentdingdang/webviewsdk](https://github.com/tencentdingdang/webviewsdk)

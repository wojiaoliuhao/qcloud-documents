## 文档导读
本文主要介绍如何基于 TRTC SDK 实现一个既支持视频连麦，又支持上万人高并发观看的在线直播功能：

- 本文仅提及了最基本的几个功能，如果您希望了解更多高级功能，请参见 [高级功能](https://cloud.tencent.com/document/product/647/32227)。
- 本文仅罗列了最常用的几个接口，如果您希望了解更多的接口函数，请参见 [API 文档](https://cloud.tencent.com/document/product/647/32228)。

## 示例代码

| 所属平台 | 示例代码 | 
|---------|---------|
| Windows（MFC） | [TRTCMainViewController.cpp](https://github.com/tencentyun/TRTCSDK/blob/master/Windows/MFCDemo/TRTCMainViewController.cpp) |
| Windows（Duilib） | [TRTCMainViewController.cpp](https://github.com/tencentyun/TRTCSDK/blob/master/Windows/DuilibDemo/TRTCMainViewController.cpp) |


## 在线直播
### 1. 初始化 SDK

使用 TRTC SDK 的第一步，是先获取 `TRTCCloud` 的单例对象，并注册监听 SDK 事件的回调。

- 继承 `ITRTCCloudCallback` 事件回调接口类，重写关键事件的回调接口，包括本地用户进房/退房事件、远端用户加入/退出事件、错误事件和警告事件等。
- 调用 `addCallback` 接口注册监听 SDK 事件。

>!如果 `addCallback` 注册 N 次，同一个事件， SDK 就会触发 N 次回调，建议只调用一次 `addCallback`。

```c++
// TRTCMainViewController.h

// 继承 ITRTCCloudCallback 事件回调接口类
class TRTCMainViewController : public ITRTCCloudCallback
{
public:
	TRTCMainViewController();
	virtual ~TRTCMainViewController();

    virtual void onError(TXLiteAVError errCode, const char* errMsg, void* arg);
    virtual void onWarning(TXLiteAVWarning warningCode, const char* warningMsg, void* arg);
    virtual void onEnterRoom(uint64_t elapsed);
    virtual void onExitRoom(int reason);
    virtual void onUserEnter(const char* userId);
    virtual void onUserExit(const char* userId, int reason);
...
private:
	ITRTCCloud * m_pTRTCSDK = NULL；
...
}

// TRTCMainViewController.cpp

TRTCMainViewController::TRTCMainViewController()
{
    // 创建 TRTCCloud 实例
    m_pTRTCSDK = getTRTCShareInstance();
    
    // 注册 SDK 回调事件
    m_pTRTCSDK->addCallback(this);
}

TRTCMainViewController::~TRTCMainViewController()
{
    // 取消监听 SDK 事件
    if(m_pTRTCSDK) {
        m_pTRTCSDK->removeCallback(this);
    }
    
    // 释放 TRTCCloud 实例
	  if(m_pTRTCSDK != NULL) {
       destroyTRTCShareInstance();
        m_pTRTCSDK = null;
    }
}

// 错误通知是要监听的，错误通知意味着 SDK 不能继续运行了
virtual void TRTCMainViewController::onError(TXLiteAVError errCode, const char* errMsg, void* arg)
{
    if (errCode == ERR_ROOM_ENTER_FAIL) {
        LOGE(L"onError errorCode[%d], errorInfo[%s]", errCode, UTF82Wide(errMsg).c_str());
		    exitRoom();
	 }
}
```

### 2. 组装 TRTCParams

TRTCParams 是 SDK 最关键的一个参数，它包含如下四个必填的字段：sdkAppId、userId、userSig 和 roomId。

- **sdkAppId**
进入腾讯云实时音视频 [控制台](https://console.cloud.tencent.com/rav)，如果您还没有应用，请创建一个，即可看到 sdkAppId。
![](https://main.qcloudimg.com/raw/e42c76fd9d4fd3e3e5d80e8fb2763134.png)

- **userId**
  您可以随意指定，由于是字符串类型，可以直接跟您现有的账号体系保持一致，但请注意，**同一个音视频房间里不应该有两个同名的 userId**。

- **userSig**
  基于 sdkAppId 和 userId 可以计算出 userSig，计算方法请参见 [如何计算 UserSig](https://cloud.tencent.com/document/product/647/17275)。

- **roomId**
  房间号是数字类型，您可以随意指定，但请注意，**同一个应用里的两个音视频房间不能分配同一个 roomId**。

### 3. 主播预览摄像头画面
TRTC SDK 并不会默认打开本地的摄像头采集，`startLocalPreview` 可以开启本地的摄像头并显示预览画面，`stopLocalPreview` 则会关闭。

启动本地预览前，可调用 `setLocalViewFillMode` 指定视频显示模式为 `Fill` 或 `Fit` 模式。两种模式下视频尺寸都是等比缩放，区别在于：`Fill` 模式优先保证视窗被填满。如果缩放后的视频尺寸与显示视窗尺寸不一致，多出的视频将被截掉； `Fit` 模式则优先保证视频内容全部显示。如果缩放后的视频尺寸与显示视窗尺寸不一致，未被填满的视窗区域将使用黑色填充。

```c++
void TRTCMainViewController::onEnterRoom(uint64_t elapsed)
{
	// 获取渲染窗口的句柄。
    CWnd *pLocalVideoView = GetDlgItem(IDC_LOCAL_VIDEO_VIEW);
    HWND hwnd = pLocalVideoView->GetSafeHwnd();
    
    if(m_pTRTCSDK)
    {
        // 调用SDK接口设置渲染模式和渲染窗口。
        m_pTRTCSDK->setLocalViewFillMode(TRTCVideoFillMode_Fit);
        m_pTRTCSDK->startLocalPreview(hwnd);
    }
    
}
```

### 4. 主播开启麦克风采集
TRTC SDK 并不会默认打开本地的麦克风采集，主播调用 `startLocalAudio` 可以开启本地的声音采集并将音视频数据广播出去，`stopLocalAudio` 则会关闭。您可以在 `startLocalPreview` 之后继续调用 `startLocalAudio`。

>`startLocalAudio` 会检查麦克风使用权限，如果没有麦克风权限，SDK 会向用户申请开启。

### 5. 主播创建新房间开播

主播可以使用 `enterRoom` 创建一个音视频房间，参数 `TRTCParams` 中的 `roomId` 用于指定房间号，同时，我们还需要将 `role` 字段指定为 `TRTCRoleAnchor`（主播）。

`appScene` 参数指定 SDK 的应用场景，本文档中我们使用 `TRTCAppSceneLIVE`（在线直播）。

- 如创建成功，SDK 会回调 `onEnterRoom` 接口，参数：`elapsed` 代表进入耗时，单位：ms。
- 如创建失败，SDK 会回调 `onError` 接口，参数：`errCode`（错误码 `ERR_ROOM_ENTER_FAIL`，错误码可参考 `TXLiteAVCode.h`）、`errMsg`（错误原因）、`extraInfo`（保留参数）。

```c++
// TRTCMainViewController.cpp

void TRTCMainViewController::startBroadCasting()
{
    // TRTCParams 定义参考头文件TRTCCloudDef.h
    TRTCParams params;
    params.sdkAppId = sdkappid;
    params.userId   = userid;
    params.userSig  = usersig;
    params.roomId   = 908; // 输入您想进入的房间
    params.role     = TRTCRoleAnchor; //主播
    if(m_pTRTCSDK)
    {
    	m_pTRTCSDK->enterRoom(params, TRTCAppSceneLIVE);
    }
}

void TRTCMainViewController::onError(TXLiteAVError errCode, const char* errMsg, void* arg)
{
    if(errCode == ERR_ROOM_ENTER_FAIL)
    {
        LOGE(L"onError errorCode[%d], errorInfo[%s]", errCode, UTF82Wide(errMsg).c_str());
        // 检查userSig是否合法、网络是否正常等
    }
}

...

void TRTCMainViewController::onEnterRoom(uint64_t elapsed)
{
    LOGI(L"onEnterRoom elapsed[%lld]", elapsed);
    
	// 启动本地的视频预览，请参考下面 设置视频编码参数 和 预览本地摄像头画面 的内容
}
```

### 6. 主播开关隐私模式
直播过程中，主播可能出于隐私目的希望屏蔽本地的音视频数据，可以调用 `muteLocalVideo` 屏蔽本地的视频采集，调用 `muteLocalAudio` 屏蔽本地的音频采集。

### 7. 观众加入房间观看

观众调用 `enterRoom` 可以进入一个音视频房间，参数 TRTCParams 中的 `roomId` 用于指定房间号。
`appScene` 同样填写 `TRTCAppSceneLIVE`（在线直播），但 `role` 字段需要指定为 `TRTCRoleAudience`（观众）。

```c++

void TRTCMainViewController::startPlaying()
{
    // TRTCParams 定义参考头文件TRTCCloudDef.h
    TRTCParams params;
    params.sdkAppId = sdkappid;
    params.userId   = userid;
    params.userSig  = usersig;
    params.roomId   = 908; // 输入您想进入的房间
    params.role     = TRTCRoleAudience; //观众
    if(m_pTRTCSDK)
    {
    	m_pTRTCSDK->enterRoom(params, TRTCAppSceneLIVE);
    }
}
```

如果主播在房间里，观众会通过 TRTCCloudDelegate 中的 `onUserVideoAvailable` 回调获知主播的 userid。之后，观众可以调用 `startRemoteView` 方法来显示主播的视频画面。

通过 `setRemoteViewFillMode` 可以指定视频显示模式为 `Fill` 或 `Fit` 模式。两种模式下视频尺寸都是等比缩放，区别在于：
- `Fill` 模式：优先保证视窗被填满。如果缩放后的视频尺寸与显示视窗尺寸不一致，多出的视频将被截掉。
- `Fit` 模式：优先保证视频内容全部显示。如果缩放后的视频尺寸与显示视窗尺寸不一致，未被填满的视窗区域将使用黑色填充。

```c++
void TRTCMainViewController::onUserVideoAvailable(const char* userId, bool available){
    if (available) {
        // 获取渲染窗口的句柄。
        CWnd *pRemoteVideoView = GetDlgItem(IDC_REMOTE_VIDEO_VIEW);
        HWND hwnd = pRemoteVideoView->GetSafeHwnd();  
        
        // 设置远端用户视频的渲染模式。
        m_pTRTCSDK->setRemoteViewFillMode(TRTCVideoFillMode_Fill);
        // 调用SDK接口播放远端用户流。
        m_pTRTCSDK->startRemoteView(userId， hwnd);
    } else {
        m_pTRTCSDK->stopRemoteView(userId);
    }    
}
```

> ! 在 TRTCAppSceneLIVE 模式下，同一个房间中的观众（TRTCRoleAudience）人数没有限制。

### 8. 观众跟主播连麦
不管是主播还是观众，都可以通过 TRTCCloud 提供的 `switchRole` 进行角色间的相互切换，最常见的场景是观众跟主播连麦：观众可以通过该接口切换成“小主播”，然后跟房间里原来的“大主播”进行连麦互动。
  
### 9. 退出房间
调用 `exitRoom` 方法退出房间。无论当前是否还在通话中，调用该方法会把视频通话相关的所有资源释放掉。在您调用 `exitRoom` 之后，SDK 会进入一个复杂的退房握手流程，当 SDK 回调 `onExitRoom` 方法时才算真正完成资源的释放。

```c++
// TRTCMainViewController.cpp

void TRTCMainViewController::exitRoom()
{
    if(m_pTRTCSDK)
    {
    	m_pTRTCSDK->exitRoom();
    }
}
....
void TRTCMainViewController::onExitRoom(int reason)
{
	// 退房成功，reason参数保留，暂未使用。

}
```



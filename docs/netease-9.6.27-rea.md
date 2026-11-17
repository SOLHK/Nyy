# 网易云音乐 9.6.27 REA 静态分析日志

分析日期为 2026-11-17。对象是 iOS `com.netease.cloudmusic` 9.6.27（7684），最低版本 iOS 13.3。IPA 未修改。

| 对象 | 标识 |
| --- | --- |
| IPA SHA-256 | `2bc6ee9a61a6e9877d6a3906527b3f1caf3951a088dac1a406062a8f2ddede62` |
| 主程序 SHA-256 | `977c6ca599e3fefa00a18d9e4dc068337478ff5ece7e85d0fedeca55fc06ca54` |
| Mach-O UUID | `BA7D18E7-E465-3175-B727-23DE387BC3D2` |
| Mach-O | thin arm64，PIE，基址 `0x100000000`，`LC_ENCRYPTION_INFO_64.cryptid = 0` |

## REA 记录

使用开源 [REA](https://github.com/morluto/rea) 的 `rea-agents` 6.9.0 和 Rizin 0.9.1。REA 的 IPA 检查、二进制识别和以下定点反汇编均成功退出。用主程序 Objective-C 方法表、Capstone 和 LLVM 交叉核对方法所属类、类型编码和分支。本文地址为静态虚拟地址。

| 查询 | REA 证据 ID |
| --- | --- |
| `inspect-artifact <IPA> --json` | `ev_1a1313d845635ae5eb2a72862f555ddfc23d9284d9bd927d558cbaa76cba77ab` |
| Rizin `iI` | `ev_eeba7175bc0687540c301d54f951736a6581204eec4e6a82623537d9fc228322` |
| 开屏加载和前台路径 | `ev_990c655d995ac4eaed63cd3543516536a9244a093c621150af57d4ce0a0ef3fb` |
| 开屏判断分支 | `ev_51f5afbf82d757c65d93f2feae23f7231934284295a6c17240a544156c7df9e3` |
| 激励广告资格和结果模型 | `ev_ad066afc344e76f5ed2624ab2aec189ff8103de1c29928f1193068241e3d958b` |
| 领奖请求，Rizin `pd 235 @ 0x1027d0b00; pd 45 @ 0x10271b880` | `ev_0696f7a5eccb0f01c3583482c445f78671dc90e33ba8823d800d9e05ce3d0518` |

## 开屏广告

广告判断集中在 `NMLaunchAdViewManager`。以下四个实例方法均为 `B16@0:8`。

| 方法 | 静态分支和插件行为 |
| --- | --- |
| `interceptWhenNoLogin` | 冷启动调用点 `0x100f1485c` 和 `0x1042c4228` 均反转返回值。取 `YES` 进入宿主无广告分支，分别到 `0x100f14a8c` 和 `0x1042c4420` 处理启动占位界面并退出。 |
| `shouldAdNoRequestOnLaunch` | 预加载在 `0x1006b8590` 查询该值。取 `YES` 走预加载方法的返回分支。 |
| `checkoutIfShouldShowAdWhenBecomeActive` | 回前台在 `0x100f18e7c` 查询，在 `0x100f1903c` 根据结果跳转。取 `NO` 跳过 `0x100f19058` 的广告加载，继续处理原生口令分享检查。 |
| `shouldRequestVIPStartupRedAdWhenBecomeActive` | 回前台在 `0x100f19220` 单独检查 VIP 开屏扩展。取 `NO` 后跳过 `0x100f19248` 的加载调用。 |

冷启动在注册广告完成回调之前选择无广告分支。主界面继续使用原有的退出启动页逻辑。

## 看广告免费听

付费提示的静态入口包括 `NMRewardAdsNoticeView rewardStyle` 和 `NMRewardAdsRequest initWithSongId:`，分别可见领取按钮事件 `btn_ads_audition_getfree` 和资格路径 `/ad/listening/rights/qualification`。广告控制器 `NMAdvertisementIncentiveVideoVC loadAd:` 从路由参数读取 `adPosition` 与 `resourceId`。这两个值由运行时配置提供。

截图中的「狠心离开」有 `0x1027b5144` 和 `0x1027d022c` 的交叉引用，分别落在 `NMAdvertisementIncentiveDetainmentPopView leaveBtn` 和 `NMAdvertisementIncentiveVideoView closeBtnClick`。同一挽留弹窗在 `0x1027b4ff8` 引用「换一个」。

1. `NMAdvertisementIncentiveVideoView imageProgress` 在 `0x1027d1608` 和 `0x1027d16a0` 调用 `onImpressWithProgress:duration:`。后者在 `0x1027d5214` 读取 `generalRightsInfo.validVideoInterval`，保存真实进度，并在 `0x1027d5270` 比较当前进度和门槛。
2. `canGetRights`（`0x1027c988c`）按 `rightsGainMethod` 判断观看进度、点击或其他动作，并检查当前领奖状态。`closeAdWithEndToast:` 在 `0x1027ca6a8` 查询资格，随后使用当前进度在 `0x1027ca710` 调用 `getRights:complation:`。
3. `getRights:complation:` 对进度有额外分支。测试开关 `incentive_ad_gain_without_progress_check` 在 `0x1027d0b0c` 出现。进入请求分支后，`0x1027d0bac` 设置请求中状态，`0x1027d0ce8` 发送请求。请求路径是 `/ad/listening/rights/gain`。原生请求参数包含广告模型、资格参数和曝光、点击相关字段。
4. 响应回调在 `0x1027d0dd0` 检查业务 `code == 200`，在 `0x1027d0e30` 读取 `gainFlag`。成功分支在 `0x1027d0e40` 设置已领取状态，随后走原生权益通知。模型中可见 `rightsUpperLimit` 与 `rightsEndTime`。仅凭静态代码无法确定服务端接受提前请求的条件或账户次数上限。

### 插件测试版

目标广告取得第一笔正向进度后，插件暂停播放，并在该广告视图的原生关闭调用中放行客户端资格检查。由宿主发送领奖请求，使用真实进度和原始服务器响应。单个视图中的同一广告发起一次提前请求，插件没有跨广告的计数器。

本版按 `generalRightsInfo.rightType == 5` 定位宿主收银台相关路径。`closeCashierIfNeeded` 在 `0x1027d2734` 检查该值。截图入口在账户上实际收到的类型与广告配置仍需真机比对。旧播放器使用 `listeningRightsInfo` 并显示分钟数，该代码位于另一条路径。

插件已交叉编译并链接为 iOS arm64 dylib，检查了平台、最低版本、初始化器、UUID 和临时签名的页哈希。冷启动、回前台、联网领奖、实际播放和连续领取尚无真机结果。

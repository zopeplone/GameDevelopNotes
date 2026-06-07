# 目标
混淆js
```js
"use strict";
(Object.defineProperty(exports, "__esModule", { value: !0 }),
  (exports.AppLinks = void 0));
const UE = require("ue"),
  BaseConfigController_1 = require("./BaseConfig/BaseConfigController"),
  HotPatchKuroSdk_1 = require("./HotPatchKuroSdk/HotPatchKuroSdk"),
  HotPatchLogReport_1 = require("./HotPatchLogReport"),
  Platform_1 = require("./Platform/Platform"),
  LauncherLog_1 = require("./Util/LauncherLog"),
  APP_LINKS_HOST = "mc.kurogame.com",
  APP_LINKS_HOST_GLOBAL = "wutheringwaves.kurogame.com";
class AppLinks {
  static Init() {
    var e;
    AppLinks.S_e ||
      (Platform_1.Platform.IsMobilePlatform() &&
        ((e =
          "CN" !==
          BaseConfigController_1.BaseConfigController.GetPublicValue(
            "SdkArea",
          )),
        Platform_1.Platform.IsIOSPlatform() && e
          ? UE.KuroLauncherLibrary.IsFirstIntoLauncher() &&
            (LauncherLog_1.LauncherLog.Info("派发IOS缓存的事件"),
            UE.KuroiOSDelegateStaticLibrary.DispatchCacheMessage())
          : HotPatchKuroSdk_1.HotPatchKuroSdk.CanUseSdk() &&
            (UE.KuroSDKManager.Get().OnActivatedByApplinksDelegate.Clear(),
            UE.KuroSDKManager.Get().OnActivatedByApplinksDelegate.Add(
              (e, t, o) => {
                AppLinks.ReportApplinksActivation(e, t, o);
              },
            ),
            Platform_1.Platform.IsAndroidPlatform()
              ? UE.KuroSDKManager.CheckApplinksActivation()
              : Platform_1.Platform.IsIOSPlatform() &&
                UE.KuroLauncherLibrary.IsFirstIntoLauncher() &&
                (LauncherLog_1.LauncherLog.Info("派发IOS缓存的事件"),
                UE.KuroiOSDelegateStaticLibrary.DispatchCacheMessage()),
            (AppLinks.S_e = !0))));

  }

  static Destroy() {
    AppLinks.S_e &&
      (UE.KuroSDKManager.Get().OnActivatedByApplinksDelegate.Clear(),
      this.u2n.clear(),
      (AppLinks.S_e = !1));
  }
  static ReportApplinksActivation(e, t, o) {
    ((e !== APP_LINKS_HOST && e !== APP_LINKS_HOST_GLOBAL) ||
      (HotPatchLogReport_1.HotPatchLogReport.ReportAppLinksEvent(t, o),
      "" !== t && this.u2n.has(t) && this.u2n.get(t)(t, o)),
      LauncherLog_1.LauncherLog.Info(
        `Application Activated By AppLinks: ${e}, ${t}, ` + o,
      ));
  }

  static SetDeepValueHandle(e, t) {
    this.u2n.has(e)
      ? LauncherLog_1.LauncherLog.Error(
          "Try to set duplicated handle for deepvalue: " + e,
        )
      : this.u2n.set(e, t);
  }

  static RemoveDeepValueHandle(e) {
    this.u2n.has(e)
      ? this.u2n.delete(e)
      : LauncherLog_1.LauncherLog.Error(
          "Can't find handle for deepvalue: " + e,
        );
  }
}

(((exports.AppLinks = AppLinks).S_e = !1), (AppLinks.u2n = new Map()));
```
目标ts
```ts
"use strict";

/**
 * 推测后的变量命名：
 * S_e  -> isInitialized
 * u2n  -> deepValueHandlers
 * e/t/o -> host / deepValue / payload
 */

import * as UE from "ue";

import { BaseConfigController } from "./BaseConfig/BaseConfigController";
import { HotPatchKuroSdk } from "./HotPatchKuroSdk/HotPatchKuroSdk";
import { HotPatchLogReport } from "./HotPatchLogReport";
import { Platform } from "./Platform/Platform";
import { LauncherLog } from "./Util/LauncherLog";

const APP_LINKS_HOST_CN = "mc.kurogame.com";
const APP_LINKS_HOST_GLOBAL = "wutheringwaves.kurogame.com";

export class AppLinks {

  static isInitialized: boolean = false;
  static deepValueHandlers: Map<
    string,
    (deepValue: string, payload: string) => void
  > = new Map();
  static Init(): void {
    // 已初始化则直接返回
    if (AppLinks.isInitialized) {
      return;
    }
    if (!Platform.IsMobilePlatform()) {
      return;
    }
    const isGlobal =
      BaseConfigController.GetPublicValue("SdkArea") !== "CN";

    if (Platform.IsIOSPlatform() && isGlobal) {
      if (UE.KuroLauncherLibrary.IsFirstIntoLauncher()) {
        LauncherLog.Info("派发IOS缓存的事件");
        UE.KuroiOSDelegateStaticLibrary.DispatchCacheMessage();
      }
      return;
    }

    if (!HotPatchKuroSdk.CanUseSdk()) {
      return;
    }

    UE.KuroSDKManager.Get()
      .OnActivatedByApplinksDelegate.Clear();
    UE.KuroSDKManager.Get()
      .OnActivatedByApplinksDelegate.Add(
        (
          host: string,
          deepValue: string,
          payload: string,
        ) => {
          AppLinks.ReportApplinksActivation(
            host,
            deepValue,
            payload,
          );
        },
      );
    if (Platform.IsAndroidPlatform()) {
      UE.KuroSDKManager.CheckApplinksActivation();
    } else if (Platform.IsIOSPlatform()) {
      if (UE.KuroLauncherLibrary.IsFirstIntoLauncher()) {
        LauncherLog.Info("派发IOS缓存的事件");

        UE.KuroiOSDelegateStaticLibrary.DispatchCacheMessage();
      }
    }

    AppLinks.isInitialized = true;
  }
  static Destroy(): void {
    if (!AppLinks.isInitialized) {
      return;
    }
    UE.KuroSDKManager.Get()
      .OnActivatedByApplinksDelegate.Clear();
    AppLinks.deepValueHandlers.clear();
    AppLinks.isInitialized = false;
  }

  /**
   * 上报 AppLinks 激活事件
   */
  static ReportApplinksActivation(
    host: string,
    deepValue: string,
    payload: string,
  ): void {
    const isValidHost =
      host === APP_LINKS_HOST_CN ||
      host === APP_LINKS_HOST_GLOBAL;

    if (isValidHost) {
      // 上报日志
      HotPatchLogReport.ReportAppLinksEvent(
        deepValue,
        payload,
      );
      // 分发 deepValue 回调
      if (
        deepValue !== "" &&
        this.deepValueHandlers.has(deepValue)
      ) {
        const handler =
          this.deepValueHandlers.get(deepValue);

        if (handler) {
          handler(deepValue, payload);
        }
      }
    }
    LauncherLog.Info(
      `Application Activated By AppLinks: ${host}, ${deepValue}, ${payload}`,
    );
  }
  static SetDeepValueHandle(
    deepValue: string,
    handler: (
      deepValue: string,
      payload: string,
    ) => void,
  ): void {
    if (this.deepValueHandlers.has(deepValue)) {
      LauncherLog.Error(
        "Try to set duplicated handle for deepvalue: " +
          deepValue,
      );

      return;
    }

    this.deepValueHandlers.set(
      deepValue,
      handler,
    );
  }

  static RemoveDeepValueHandle(
    deepValue: string,
  ): void {
    if (!this.deepValueHandlers.has(deepValue)) {
      LauncherLog.Error(
        "Can't find handle for deepvalue: " +
          deepValue,
      );

      return;
    }

    this.deepValueHandlers.delete(deepValue);
  }
}
```


# 步骤

## 静态分析代码转换
1. 在文件头部的 `require` 全部替换为 `import` ，对于xxx_1，xxx_2的导入，合并并全部都通过解构的方式导入模块内变量
2. 对于动态导入的require，如果是同步的则不需要管，如果是`new Promise`包起来的require，用import()函数来代替
3. 对于UE，cpp，puerts，全部都使用命名空间导入
4. 对于纯js模块，如 `EventSystem_1.EventSystem.RemoveWithTarget` 则使用解构导入，将`EventSystem_1`去掉，解构导入`EventSystem`
5. 对于defult导入的js，如`TsAiController.js`，这种在其他地方使用时，都是 `TsAiController_1.default` 来使用，这种很好判断，直接default导入即可
6. 在文件末尾，的`(((exports.AppLinks = AppLinks).S_e = !1), (AppLinks.u2n = new Map()));`，可以识别出导出了哪些东西，以及初始化了这个类中的哪些static变量，识别出的static变量必须要放到class中定义
7. 在文件头部的require下，用逗号还声明了许多模块内的常量，这些也需要用另外的const去处理，每个常量一个const
8. `(Object.defineProperty(exports, "__esModule", { value: !0 })` 对于文件顶部的这个defineProperty不需要
9. 将`void 0` 改为 undefined，将`!0`改为true, `!1`改为false

对于静态分析代码转换的工具，还需要有地方能够配置一些库文件和库
比如 `Protocol_1 = require("../../../Core/Define/Net/Protocol"),`
这里的 `Protocol`工具我们不想去分析，只希望将它当作一个外部库来使用，并且将`Protocol_1`改为`Protocol`并进行默认导入
## AI代码转换

### AI能力
1. 给代码中的所有混淆后的类成员变量/方法，局部变量，函数进行解混淆
	- 对于成员变量/方法和export的函数，解混淆之后，需要调用工具来告诉工具哪个变量解混淆成了哪个名称（在整个源码文件中，每个混淆后的变量类成员变量全局唯一！）
	- 对于局部变量和未export的函数，可以直接进行解混淆，ai不需要调用工具
2. 根据代码，生成每个变量的ts类型，在输出的ts中需要包含类型
3. 将这种代码改为宜读的代码，删去括号，将这些逗号改为分号，将不易读的三目运算符, && || 改为if else
```js
// 删去括号，逗号改为分号
(super(t, i),
	(this.UsePool = !1),
	(this.m6 = void 0),
	(this._W = void 0),
	(this.InitStatTdType = void 0),
	(this.uW = void 0)
)
// 将大多数不易读的三目运算符和 &&, || 改为if/else
((this.qzo = 0 === this.$4a.size && 0 === this.X4a.size),
      this.PW && (this.qzo = this.qzo && this.PW.Active),
      this.qzo !== t &&
        (this.qzo
          ? this.qW && this.FW(() => (this.OnEnable(), !0), this.OnEnable.name)
          : this.GW &&
            this.FW(() => {
              var t =
                "string" == typeof i ? i : "EEntityDisableKey." + i?.toString();
              return (this.OnDisable(t), !0);
            }, this.OnDisable.name)));
	
```

### 如何调用AI以及AI如何输出ts
AI Agent 我想选择 OpenCode SDK，在OpenCodeSDK可以在js工具中进行调用
我的工具需要可以有一个供AI进行工具调用的接口，在OpenCodeSDK中注册该工具
这个工具需要有一个简易的数据库，能够持久化
1. 解混淆了哪些类成员变量/方法，export的函数，要支持混淆的符号名和解混淆后的符号名双向查找
2. 哪些文件AI已经生成完毕
3. 当工具想让AI开始解混淆输出ts时，会给ai哪个文件让他进行工作？这里我想先生成一个依赖图，然后工具根据依赖图来给AI某个js文件进行工作
4. 支持有限并发（1-5个），由依赖图来决定并发，每个Agent只能生成其对应的文件（而且需要保证生成的ts和源js的目录匹配），不可修改其它文件和生成其它文件，但是可以查找所有的js源码和已生成的ts
5. 需要给AI一个提示词，里面存放解混淆的标准提示词
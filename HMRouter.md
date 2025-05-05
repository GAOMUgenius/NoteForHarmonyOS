# 1. 概述

HMrouter是HarmonyOS上页面跳转的场景解决方案，主要解决页面间互相跳转的问题，本文主要介绍HMRouter路由框架的使用。HMrouter路由框架提供了下列功能特性：

- 使用自定义注解实现路由跳转。
- 支持HAR/HSP。
- 支持路由拦截、路由生命周期。
- 简化自定义动画配置：配置全局动画，单独指定某个页面的切换动画。
- 支持不同的页面类型：单例页面、Dialog页面。

该框架底层对Navigation相关能力进行了封装，帮助开发者减少对Navigation相关细节内容的关注、提高开发效率，同时该框架对页面跳转能力进行了增强，例如其中的路由拦截、单例页面等。

# 2. 下载安装并配置

## 系统依赖版本

SDK: Ohos_sdk_public 5.0.0.71 (API 12 Release)

## 使用ohpm安装依赖

```
ohpm install @hadss/hmrouter
```

> 或者按需在模块中配置运行时依赖，修改`oh-package.json5`

```json
{
  "dependencies": {
    "@hadss/hmrouter": "^1.0.0-rc.11"
  }
}
```

## 使用配置

### 编译插件配置

1. 修改工程的`hvigor/hvigor-config.json`文件，加入路由编译插件

   ```json
   {
     "dependencies": {
       "@hadss/hmrouter-plugin": "^1.0.0-rc.11"
       // 使用npm仓版本号
     },
     // ...其他配置
   }
   ```

   2.在使用到HMRouter的模块中引入路由编译插件，修改`hvigorfile.ts`

   示例：

   ```typescript
   // ./hvigorfile.ts  工程根目录的hvigorfile.ts
   import { appTasks } from '@ohos/hvigor-ohos-plugin';
   
   export default {
     system: appTasks,
     plugins:[]
   }
   
   // entry/hvigorfile.ts  entry模块的hvigorfile.ts
   import { hapTasks } from '@ohos/hvigor-ohos-plugin';
   import { hapPlugin } from '@hadss/hmrouter-plugin';
   
   export default {
     system: hapTasks,
     plugins: [hapPlugin()] // 使用HMRouter标签的模块均需要配置，与模块类型保持一致
   }
   
   // libHar/hvigorfile.ts  libHar模块的hvigorfile.ts
   import { harTasks } from '@ohos/hvigor-ohos-plugin';
   import { harPlugin } from '@hadss/hmrouter-plugin';
   
   export default {
     system: harTasks,
     plugins:[harPlugin()]  // 使用HMRouter标签的模块均需要配置，与模块类型保持一致
   }
   
   // libHsp/hvigorfile.ts  libHsp模块的hvigorfile.ts
   import { hspTasks } from '@ohos/hvigor-ohos-plugin';
   import { hspPlugin } from '@hadss/hmrouter-plugin';
   
   export default {
     system: hspTasks,
     plugins: [hspPlugin()]  // 使用HMRouter标签的模块均需要配置，与模块类型保持一致
   }
   ```

   > 如果模块是Har则使用`harPlugin()`, 模块是Hsp则使用`hspPlugin()`, 模块是Hap则使用`hapPlugin()`

   3.在项目根目录创建路由编译插件配置文件`hmrouter_config.json`（可选）

   ```json
   {
     // 如果不配置则扫描src/main/ets目录，对代码进行全量扫描，如果配置则数组不能为空，建议配置指定目录可缩短编译耗时
     "scanDir": [
       "src/main/ets/components",
       "src/main/ets/interceptors"
     ],
     // 默认为false，调试排除错误时可以改成true，不删除编译产物
     "saveGeneratedFile": false,
     // 默认为false，不自动配置混淆规则，只会生成hmrouter_obfuscation_rules.txt文件帮助开发者配置混淆文件；如果设置为true，会自动配置混淆规则，并删除hmrouter_obfuscation_rules.txt文件
     "autoObfuscation": false,
     // 默认模板文件，不配置时使用插件内置模板
     "defaultPageTemplate": "./templates/defaultTemplate.ejs",
     // 特殊页面模版文件，匹配原则支持文件通配符
     "customPageTemplate": [
       {
         "srcPath": [
           "**/component/Home/**/*.ets"
         ],
         "templatePath": "templates/home_shopping_template.ejs"
       },
       {
         "srcPath": [
           "**/live/**/*.ets"
         ],
         "templatePath": "templates/live_template.ejs"
       }
     ]
   }
   ```

   > 配置文件读取规则为 模块 > 工程 > 默认
   >
   > 优先使用本模块内的配置，如果没有配置，则使用工程目录中的配置，若找不到则使用默认配置

### 工程配置

由于拦截器、生命周期和自定义转场动画会在运行时动态创建实例，因此需要进行如下配置，使得HMRouter路由框架可以动态导入项目中的模块

1.在工程目录下的`build-profile.json5`中，配置`useNormalizedOHMUrl`属性为true

```json
{
  "app": {
    "products": [
      {
        "name": "default",
        "signingConfig": "default",
        "compatibleSdkVersion": "5.0.0(12)",
        "runtimeOS": "HarmonyOS",
        "buildOption": {
          "strictMode": {
            "useNormalizedOHMUrl": true
          }
        }
      }
    ],
    // ...其他配置
  }
}
```

2.在`oh-package.json5`中配置对Har和Hsp的依赖，这里需要注意**依赖的模块名称需要与module.json5中moduleName、oh-package.json5中name保持一致**。

动态import实现方案介绍

```json
{
  "dependencies": {
    "AppHar": "file:../AppHar",
    // AppHar库可以正确动态创建拦截器、生命周期和自定义转场动画对象
    "@app/har": "file:../AppHar"
    // 错误使用方式，无法动态创建对象
  }
}
```

# 3. 快速开始

## 3.1 在UIAbility初始化路由框架

```typescript
export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    HMRouterMgr.init({
      context: this.context
    })
  }
}
```

## 3.2 在启动框架AppStartup中初始化路由框架

**AppStartup**提供了一种简单高效的应用启动方式，可以支持任务的异步启动，加快应用启动速度。同时，通过在一个配置文件中统一设置多个启动任务的执行顺序以及依赖关系，让执行启动任务的代码变得更加简洁清晰、容易维护。

1. **定义启动框架配置**

   在应用主模块的“resources/base/profile”路径下，新建启动框架配置文件。文件名可以自定义，本文以“startup_config.json”为例，此处注意，**runOnThread的值必须配置为mainThread，配置为taskPool会导致页面无法正常加载。**,**waitOnMainThread推荐配置为true，配置为false可能会导致首页异常**

   ```json
   {
     "startupTasks": [
       {
         "name": "HMRouterInitStartupTask",
         "srcEntry": "./ets/startup/HMRouterInitStartupTask.ets",
         "runOnThread": "mainThread",
         "waitOnMainThread": true
       }
     ],
     "configEntry": "./ets/startup/StartupConfig.ets"
   }
   ```

2. **设置启动参数**

   在“ets/startup/StartupConfig.ets”中设置启动参数，可参考官网启动参数配置

   ```typescript
   import { StartupConfig, StartupConfigEntry, StartupListener } from '@kit.AbilityKit';
   import { hilog } from '@kit.PerformanceAnalysisKit';
   import { BusinessError } from '@kit.BasicServicesKit';
   
   export default class MyStartupConfigEntry extends StartupConfigEntry {
     onConfig() {
       hilog.info(0x0000, 'testTag', `onConfig`);
       let onCompletedCallback = (error: BusinessError<void>) => {
         hilog.info(0x0000, 'testTag', `onCompletedCallback`);
         if (error) {
           hilog.info(0x0000, 'testTag', 'onCompletedCallback: %{public}d, message: %{public}s', error.code,
             error.message);
         } else {
           hilog.info(0x0000, 'testTag', `onCompletedCallback: success.`);
         }
       };
       let startupListener: StartupListener = {
         'onCompleted': onCompletedCallback
       };
       let config: StartupConfig = {
         'timeoutMs': 10000,
         'startupListener': startupListener
       };
       return config;
     }
   }
   ```

3. **为HMRouter添加启动任务**

   在"src/main/ets/startup/HMRouterInitStartupTask.ets"中添加启动任务，初始化HMRouter

   ```typescript
   import { HMRouterMgr } from '@hadss/hmrouter';
   import { StartupTask, common } from '@kit.AbilityKit';
   import { hilog } from '@kit.PerformanceAnalysisKit';
   
   @Sendable
   export default class HMRouterInitStartupTask extends StartupTask {
     constructor() {
       super();
     }
   
     async init(context: common.AbilityStageContext) {
         // add task to init HMRouter
       HMRouterMgr.init({
         context: context
       })
       HMRouterMgr.openLog('DEBUG')
     }
   
     onDependencyCompleted(dependence: string, result: Object): void {
       hilog.info(0x0000, 'testTag', 'StartupTask_001 onDependencyCompleted, dependence: %{public}s, result: %{public}s',
         dependence, JSON.stringify(result));
     }
   }
   ```

## 3.3 定义路由入口

HMRouter以来系统Navigation能力，**所以必须在页面中定义一个HMNaviagtion容器**，并设置相关参数，具体代码如下：

```typescript
import { HMDefaultGlobalAnimator, HMNavigation } from '@hadss/hmrouter';
import { AttributeUpdater } from '@kit.ArkUI';

@Entry
@Component
struct Index {
  modifier: MyNavModifier = new MyNavModifier();

  build() {
    // @Entry中需要再套一层容器组件,Column或者Stack
    Column() {
      // 使用HMNavigation容器
      HMNavigation({
        navigationId: 'mainNavigation', homePageUrl: 'MainPage',
        options: {
          standardAnimator: HMDefaultGlobalAnimator.STANDARD_ANIMATOR,
          dialogAnimator: HMDefaultGlobalAnimator.DIALOG_ANIMATOR,
          modifier: this.modifier
        }
      })
    }
    .height('100%')
    .width('100%')
  }
}

class MyNavModifier extends AttributeUpdater<NavigationAttribute> {
  initializeModifier(instance: NavigationAttribute): void {
    instance.hideNavBar(true);
  }
}
```

Navigation的系统属性通过`modifier`传递，部分`modifier`不支持的属性使用`options`设置

## 3.4 定义拦截器

使用@HMInterceptor标签定义拦截器，并实现`IHMInterceptor`接口。

```typescript
@HMInterceptor({ interceptorName: 'JumpInfoInterceptor', global: true })
export class JumpInfoInterceptor implements IHMInterceptor {
  handle(info: HMInterceptorInfo): HMInterceptorAction {
    let connectionInfo: string = info.type === 'push' ? 'jump to' : 'back to';
    Logger.info(`${info.srcName} ${connectionInfo} ${info.targetName}`)
    return HMInterceptorAction.DO_NEXT;
  }
}
```

## 3.5 定义生命周期

使用`@HMLifecycle`标签定义生命周期处理器，并实现`IHMLifecycle`接口

```typescript
@HMLifecycle({ lifecycleName: 'PageDurationLifecycle' })
export class PageDurationLifecycle implements IHMLifecycle {
  private time: number = 0;

  onShown(ctx: HMLifecycleContext): void {
    this.time = new Date().getTime();
  }

  onHidden(ctx: HMLifecycleContext): void {
    const duration = new Date().getTime() - this.time;
    Logger.info(`Page ${ctx.navContext?.pathInfo.name} stay ${duration}`);
  }
}
```

## 3.6 自定义转场动画

### 3.6.1 IHMAnimator,自定义转场动画接口

自定义动画接口，所有`@HMAnimator`申明的自定义动画需要实现此接口，当页面指定了自定义动效的页面将不在执行HMNavgaion中指定的默认动效；

页面定义动画对应页面2类情况：

1. 页面入场动画，针对路由跳转的目标页面
   - 主动入场，A页面push/replace到B页面，B页面属于主动入场，B页面执行动画过程：`enterHandle.start,enterHandle.finish,enterHandle.onFinish`或者`enterHandle.customAnimation`
   - 被动入场，B页面pop到A页面，A页面属于被动入场，A页面执行的动画过程: `enterHandle.passiveStart,enterHandle.passiveFinish,enterHandle.passiveOnFinish`或者`enterHandle.passiveCustomAnimation`
2. 页面出场动画，针对路由跳转时源页面执行的动画
   - 主动出场，B页面pop到A页面，B页面属于主动出场，B页面执行的动画过程：`exitHandle.start,exitHandle.finish,exitHandle.onFinish`或者`exitHandle.customAnimation`
   - 被动出场，A页面push/replace到B页面，A页面属于被动出场，A页面执行动画过程：`exitHandle.passiveStart,exitHandle.passiveFinish,exitHandle.passiveOnFinish`或者`exitHandle.passiveCustomAnimation`

| 接口        | 参数                                                        | 返回值 | 接口描述                         |
| ----------- | ----------------------------------------------------------- | ------ | -------------------------------- |
| effect      | enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle | void   | 声明入场和出场动画的定义         |
| interactive | handle: HMAnimatorHandle                                    | void   | 跟手转场（可交互式转场）动画定义 |

#### 3.6.1.1 HMAnimatorHandle类

动画处理器

| 参数                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| start                  | 定义组件在动画开始前的状态                                   |
| finish                 | 定义组件在动画结束后的状态                                   |
| onFinish               | 定义组件在动画结束后保持的最终状态                           |
| customAnimation        | 自定义组件动画过程，设置之后start,finish,onFinish不再生效，页面动画通过自定义动画控制 |
| passiveStart           | 定义组件被动触发动画开始前的状态                             |
| passiveFinish          | 定义组件被动触发动画结束后的状态                             |
| passiveOnFinish        | 定义组件被动触发动画结束后保持的最终状态                     |
| passiveCustomAnimation | 自定义组件被动触发动画过程，设置之后passiveStart,passiveFinish,passiveOnFinish不再生效，页面动画通过自定义动画控制 |
| timeout                | 定义动画超时时间                                             |
| curve                  | 动画曲线                                                     |
| duration               | 动画持续时间                                                 |
| interactive            | 是否可交互式转场                                             |
| actionStart            | 手势触发的回调                                               |
| updateProgress         | 更新转场进度的回调                                           |
| actionEnd              | 手势结束的回调                                               |

动画处理回调

| 参数            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| translateOption | TranslateOption对象，定义页面的位移参数，只支持x，y轴        |
| scaleOption     | ScaleOption对象，定义页面的缩放参数 ，只支持x，y轴           |
| opacityOption   | OpacityOption对象，定义页面的透明度参数，属性：opacity取值范围为0到1，1表示不透明，0表示完全透明 |
| extendOption    | Any对象，开发者自定义参数，通过模版传入                      |
| proxy           | 系统NavigationTransitionProxy对象                            |

#### 3.6.1.2 IHMAnimator.Effect类

内置转场动画定义

| 构造函数     | 说明                                   |
| ------------ | -------------------------------------- |
| effectOption | IHMAnimator.EffectOptions 动画参数定义 |

| 函数       | 说明                   |
| ---------- | ---------------------- |
| toAnimator | 转换成内置转场动画对象 |

#### 3.6.1.3 IHMAnimator.EffectOptions 类

| 参数      | 说明             |
| --------- | ---------------- |
| direction | 转场方向定义     |
| opacity   | 透明度初始值定义 |
| scale     | 缩放初始值定义   |

### 3.6.2 原理介绍

通过实现`IHMAnimator`接口，覆盖`effect`和`interactive`方法，来对动画效果进行自定义

动画效果处理器`HMAnimatorHandle`在页面加载`aboutToAppear`时进行初始化，对动画效果进行预设，并在页面跳转时进行调用

一次性动画的处理器会在页面跳转时进行重新覆盖

因此需注意:

- 全局动画的变更不会影响到已经加载的页面(包括单例页面)，只有页面重新加载才会生效

#### 3.6.2.1 动画状态设置

`effect(enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle)`方法中提供了入场动画`enterHandle`和出场动画`exitHandle`2组动画状态设置

通过对`HMAnimatorHandle`回调中的页面位移/缩放/透明度等状态进行设置，来实现动画效果

- 入场动画分为主动入场`enterHandle.start/finish/onFinish`和被动入场`enterHandle.passiveStart/passiveFinish/passiveOnFinish`
- 出场动画分为主动出场`exitHandle.start/finish/onFinish`和被动出场`exitHandle.passiveStart/passiveFinish/passiveOnFinish`

每种动画可以设置3种页面状态:

- 动画开始状态`start`
- 动画完成状态`finish`
- 动画结束状态`onFinish`

通过设置动画曲线让页面在开始和完成状态之间进行过渡，默认动画曲线为`Curve.EaseIn`，结束状态用于将页面还原，释放资源等

**A页面push/replace到B页面**：B页面为主动入场，A页面为被动出场

![push](https://gitee.com/hadss/hmrouter/raw/dev/docs/assets/CustomTransitionExplain-Push.jpg)

**B页面pop到A页面**：B页面为主动出场，A页面为被动入场

![pop](https://gitee.com/hadss/hmrouter/raw/dev/docs/assets/CustomTransitionExplain-Pop.jpg)

动画曲线、持续时间、超时时间等也需要通过`HMAnimatorHandle`进行设置

`HMAnimatorHandle`也提供了自定义动画回调`customAnimation`和`passiveCustomAnimation`让开发者对动画进行完全控制

**自定义动画类型**:

- 全局自定义动画通过`HMNavigation`初始化时定义，并且可以通过`HMRouterMgr`进行修改
- 页面自定义动画通过`@HMRouter`标签进行定义，页面跳转时分别执行2个页面定义的动画，如果有页面未定义则使用全局动画
- 一次性动画通过`HMRouterMgr`进行页面跳转时定义，同时执行一次性动画中的入场动画和出场动画

一次性动画的覆盖原则:

> - 路由push/replace时传入一次性动画，目标页面执行一次性动画的主动入场，源页面执行一次性动画的被动离场；
> - push/replace时传入的一次性动画根据目标页面生存周期，当目标页面pop时未传入一次性动画，则执行push/replace时的一次性动画，pop目标页面执行一次性动画的被动入场，源页面执行主动离场
> - 当pop时传入一次性动画，目标页面的被动入场和源页面的主动离场均查找一次性动画
> - 上述场景描述的主动入场，被动入场，主动离场，被动离场动画，如果一次性动画内未定义，则会执行页面定义的动画或者全局动画定义

#### 3.6.2.2 交互事件配置

`interactive?(handle: HMAnimatorHandle)`方法中提供了交互式事件配置

通过对`HMAnimatorHandle`回调中的手势事件回调来进行路由操作，实现动画状态的跟随手势变化

手势事件：

- `actionStart`: 手势开始事件，在事件中触发路由跳转，通过对手势位移判断来决定页面是push/replace还是pop
- `updateProgress`: 手势更新事件，在事件中通过设置转场进度百分比来触发动画曲线的跟随手势变化
- `actionEnd`: 手势结束事件，在事件中对手势位移判断来调用`NavigationTransitionProxy`决定转场状态是完成转场还是取消转场

交互式转场方法`interactive`不设置动画效果，只是通过手势事件对转场行为进行处理并控制动画曲线的进度，动画效果需要配合`effect`方法进行定义

## 3.7 路由跳转使用

使用`@HMRouter`标签定义页面，绑定拦截器、生命周期及自定义转场动画

```typescript
@HMRouter({ pageUrl: '/pageB', lifecycle: 'pageLifecycle', animator: 'pageAnimator' })
@Component
export struct PageB {
  // 获取生命周期中定义的状态变量
  @State model: ObservedModel | null = (HMRouterMgr.getCurrentLifecycleOwner().getLifecycle() as PageLifecycle).model
  @State param: HMPageParam = HMRouterMgr.getCurrentParam(HMParamType.all)
  
  build() {
    Column() {
      Text(`${this.model?.property}`)
      Text(`${this.param.paramsMap?.get('msg')}`)
    }
  }
}
```

定义页面PageA，使用`HMRouterMgr.push`执行路由跳转至PageB

```typescript
const PAGE_URL: string = '/pageA'

@HMRouter({ pageUrl: PAGE_URL })
@Component
export struct PageA {

build() {
  Column() {
    Button('Push')
      .onClick(() => {
        HMRouterMgr.push({ pageUrl: 'pageB?msg=abcdef' })
      })
  }
}
}
```

> 路由跳转支持URL带参数的方式，例如定义的页面pageUrl: `/pages1/users`，跳转时可以指定pageUrl为: `/pages1/users?msg=1234`
>
> 通过HMRouterMgr.getCurrentParam传入HMParamType.all获取URL的参数内容

##  3.8 服务路由使用

服务路由由于类似服务提供发现机制（Service Provider Interface），通过不依赖实现模块的方式获取接口实例并调用方法，当前仅提供方法级的调用

```typescript
export class CustomService {
  @HMService({ serviceName: 'testConsole' })
  testConsole(): void {
    Logger.info('调用服务 testConsole')
  }

  @HMService({ serviceName: 'testFunWithReturn' })
  testFunWithReturn(param1: string, param2: string): string {
    return `调用服务 testFunWithReturn:${param1} ${param2}`
  }

  @HMService({ serviceName: 'testAsyncFun', singleton: true })
  async asyncFunction(): Promise<string> {
    return new Promise((resolve) => {
      resolve('调用异步服务 testAsyncFun')
    })
  }
}

@HMRouter({ pageUrl: 'test://MainPage' })
@Component
export struct Index {

build() {
  Row() {
    Column({ space: 8 }) {
      Button('service').onClick(() => {
        HMRouterMgr.request('testConsole')
        Logger.info(HMRouterMgr.request('testFunWithReturn', 'home', 'service').data)
        HMRouterMgr.request('testAsyncFun').data.then((res: string) => Logger.info(res))
      })
    }
    .width('100%')
  }
  .height('100%')
}
}
```

> 当前不支持同时和其他注解混用，也不支持静态方法

```typescript
// 不支持类与类方法同时添加 @HM* 装饰器
@HMLifecycle({ serviceName: 'lifecycleName' })
export class CustomServiceErr1 {
  @HMService({ serviceName: 'testConsole' }) // 类已经添加 @HMLifecycle 装饰器，@HMService 无法识别
  testConsole(): void {
    Logger.info('调用服务 testConsole')
  }
}

// 不支持在静态方法上添加 @HMService 装饰器
export class CustomServiceErr2 {
  @HMService({ serviceName: 'testConsole' }) // 静态方法添加 @HMService 装饰器，调用时会报错
  static testConsole(): void {
    Logger.info('调用服务 testConsole')
  }
}
```

# 4. 混淆配置说明

`@hadss/hmrouter-plugin(1.0.0-rc.6)`版本之后HMRouter支持混淆自动配置白名单

在`build-profile.json5`中配置混淆选项enable为true(开启混淆)，如下所示，并且在当前模块`hmrouter_config.json` 中配置`autoObfuscation`为true(默认为false)。

HMRouter会自动生成HMRouter必须的白名单配置。将其保存在当前模块`hmrouter_obfuscation_rules.txt` 文件中，并在编译阶段将该文件自动加入到混淆配置文件`files`列表中，实现混淆自动配置效果。

```json
// build-profile.json5
{
  "buildOptionSet": [
    {
      "name": "release",
      "arkOptions": {
        "obfuscation": {
          "ruleOptions": {
            "enable": true,
            "files": [
              "./obfuscation-rules.txt"
            ]
          }
        }
      }
    },
  ],
}
```

```json
// hmrouter_config.json
{
  "saveGeneratedFile": true,
  "autoObfuscation": true
}
```

> 如果将`autoObfuscation`改为false，则只会生成混淆规则文件，但不会自动修改模块的混淆配置。
>
> 需要自行将生成的混淆文件`hmrouter_obfuscation_rules.txt`文件加入到混淆配置文件`files`列表中。

# 5. HMRouter动态加载原理介绍

## 5.1 动态加载

### 5.1.1 动态加载介绍

目前提供两种动态加载能力：

- 动态import。
- napi_load_module_with_info/napi_load_module。

这两种方式都是在native侧动态加载arkts的模块，由于napi_load_module_with_info适用范围更广，HMRouter使用napi_load_module_with_info去动态加载

#### napi_load_module_with_info/napi_load_module函数说明

napi_load_module_with_info的函数说明如下所示

```c++
napi_status napi_load_module_with_info(napi_env env,
                                       const char* path,
                                       const char* module_info,
                                       napi_value* result);
```

| 参数        | 说明                            |
| ----------- | ------------------------------- |
| env         | 当前的虚拟机环境                |
| path        | 加载的文件路径或者模块名        |
| module_info | bundleName/moduleName的路径拼接 |
| result      | 加载的模块                      |

### 5.1.2 动态加载（napi_load_module_with_info）

#### 5.1.2.1 定义需要导出的文件

**此处需要注意，必须要export才能运行动态加载**

```typescript
//./src/main/ets/Test.ets
let value = 123;
function test() {
  console.log("Hello HarmonyOS");
}
export {value, test};
```

#### 5.1.2.2 build-profile.json5配置

必须配置才能保证运行态可以找到对应的字节码段

```json
{
    "buildOption" : {
        "arkOptions" : {
            "runtimeOnly" : {
                "sources": [
                    "./src/main/ets/Test.ets"
                ]
            }
        }
    }
}
```

#### 5.1.2.3 运行时加载

- path 由模块名称+文件的路径拼接
- module_info 由应用名称+模块名称的路径拼接

```c++
static napi_value loadModule(napi_env env, napi_callback_info info) {
    napi_value result;
    //1. 使用napi_load_module_with_info加载Test文件中的模块
    napi_status status = napi_load_module_with_info(env, "entry/src/main/ets/Test", "com.example.application/entry", &result);

    napi_value testFn;
    //2. 使用napi_get_named_property获取test函数
    napi_get_named_property(env, result, "test", &testFn);
    //3. 使用napi_call_function调用函数test
    napi_call_function(env, result, testFn, 0, nullptr, nullptr);

    napi_value value;
    napi_value key;
    std::string keyStr = "value";
    napi_create_string_utf8(env, keyStr.c_str(), keyStr.size(), &key);
    //4. 使用napi_get_property获取变量value
    napi_get_property(env, result, key, &value);
    return result;
}
```

### 5.1.3 HMRouter动态加载

此处以示例代码中LoginStatusInterceptor为例，开发者在使用HMRouter的时候就是自行实现的拦截器、生命周期、动画、功能路由等。此处需要注意，类必须要export

#### 5.1.3.1 定义需要导出的文件

```typescript
@HMInterceptor({ interceptorName: 'LoginStatusInterceptor', global: true })
export class LoginStatusInterceptor implements IHMInterceptor {
  handle(info: HMInterceptorInfo): HMInterceptorAction {
    console.log(`Login status is ${!!AppStorage.get('isLogin') ? 'Y' : 'N'}`);
    return HMInterceptorAction.DO_NEXT;
  }
}
```

#### 5.1.3.2 build-profile.json5配置

当前HMRouter插件已经帮我们在编译期自动配置，开发者不需要额外配置

#### 5.1.3.3 运行时加载

当前HMRouter已经帮我们自动封装，此处介绍当前HMRouter如何封装

1. **HMRouter编译期**

   hmrouter-plugin插件会在编译期自动解析我们配置的装饰器，并生成系统路由表，在rawfile中生成一份路由表，方便HMRouter初始化的时候读取路由表，解析路由，此处以Sample为例，打开生成好的hap/hsp，在resources/rawfile中查看是否存在hm_router_map.json文件

   ![输入图片说明](https://foruda.gitee.com/images/1725700012938306436/2b67d3bc_5119830.png)

   搜索LoginStatusInterceptor会存在下面的JSON字符串，我们在开发态定义的LoginStatusInterceptor会被编译期解析为下面的配置，此处中点关注**ohmurl、bundleName、moduleName**

   ```json
     {
         "name": "__interceptor__LoginStatusInterceptor",
         "pageSourceFile": "src/main/ets/interceptor/LoginStatusInterceptor.ets",
         "buildFunction": "",
         "customData": {
           "interceptorName": "LoginStatusInterceptor",
           "global": true,
           "name": "LoginStatusInterceptor",
           "module": "entry"
         },
         "ohmurl": "@normalized:N&&&entry/src/main/ets/interceptor/LoginStatusInterceptor&",
         "bundleName": "com.huawei.hadss.hmrouter",
         "moduleName": "entry"
       }
   ```

2. ##### HMRouter运行期

   运行期使用napi_load_module_with_info去动态加载，napi_load_module_with_info所需如下参数，由上面的JSON解析而成,感兴趣可以参考源码，源码路径HMRouterLibrary/src/main/ets/store/entity/HMComponent.ets。native侧路径HMRouterLibrary/src/main/cpp/napi_init.cpp

   - path 通过ohmurl解析而成，解析的结果为entry/src/main/ets/interceptor/LoginStatusInterceptor
   - module_info 由bundleName和moduleName拼接而成，拼接结果为com.huawei.hadss.hmrouter/entry

## 5.2 useNormalizedOHMUrl

一个ets文件在编译后会成为安装包的一部分，这个ets文件对应的字节码称为一个字节码段，OHMUrl是用来定位一个字节码段的标识。标准化的OHMUrl格式，标准化的OHMUrl统一了原有OHMUrl的格式。使用集成态HSP和字节码HAR需使用标准化的OHMUrl格式。

HMRouter使用动态加载的能力，可以在运行态动态加载指定的模块，使开发者可以在运行态加载hap、hsp、har的代码。开启useNormalizedOHMUrl是为了满足更多场景，如果不开启，会导致动态加载失效，最终导致拦截器、生命周期、动画、功能路由等失效。使用HMRouter必须配置useNormalizedOHMUrl为true，如下所示

```json
{
  "app": {
    "products": [
      {
        "name": "default",
        "signingConfig": "default",
        "compatibleSdkVersion": "5.0.0(12)",
        "runtimeOS": "HarmonyOS",
        "buildOption": {
          "strictMode": {
            "useNormalizedOHMUrl": true
          }
        }
      }
    ],
    // ...其他配置
  }
}
```

# 6. HMRouter标签的使用规则

## 6.1 路由标签@HMRouter

`@HMRouter(pageUrl, isRegex, regexPriority, dialog, singleton, interceptors, lifecycle, animator)` 标签使用在自定义组件`struct`上，且该自定义组件需要添加`export`关键字

- pageUrl: string, 用来表示NavDestination，必填

  > 1.支持使用本文件或者本模块定义的常量，或者Class中定义的静态变量
  >
  > 2.pageUrl配置支持的格式说明：
  >
  > - 支持普通字符串定义，路由跳转采用全路径方式匹配，例如定义demo://xxxx，路由跳转时pageUrl=demo://xxx可以使用全路径匹配
  >
  > 3.pageUrl路由匹配优先级说明：优先全路径匹配，然后匹配正则格式路由，正则内的路由匹配优先级通过regexPriority属性设置；例如定义了两个路由：pageUrl/detail, pageUrl/.* ；当路由跳转时传入pageUrl=/pages/detail，将匹配第一个/pages/detail，当路由跳转时传入pageUrl=/pages/abcdef时，将匹配/pages/.*定义的路由页面

- isRegex：boolean, 标识配置的pageUrl是否是正则表达式，如果配置为正则，会影响页面跳转效率，配置为true时，需要确保pageUrl为正确的正则表达式格式，非必填

- regexPriority: number, pageUrl正则匹配优先级，数字越大越先匹配，默认值为0，优先级相同时，不保证先后顺序，非必填，默认为0

- dialog: boolean, 是否是Dialog类型页面，非必填，默认为false

- singleton: boolean, 是否是单例页面，单例页面即表示在一个HMNavigation容器中，只有一个此页面，非必填，默认为false

- interceptors: string[], `@HMInterceptor`标记的拦截器名称列表，非必填

- lifecycle: string, `@HMLifecycle`标记的生命周期处理实例，非必填

- animator: string, `@HMAnimator`标记的自定义转场实例，非必填

**示例**：

```typescript
@HMRouter({
  pageUrl: 'pageOne',
  interceptors: ['LoginInterceptor'],
  lifecycle: 'pageLifecycle',
  animator: 'pageAnimator'
})
@Component
export struct PageOne {

  build() {
  }
}

// constants.ets
export class Constants {
  static readonly PAGE: string = 'pageTwo'
}

@HMRouter({ pageUrl: Constants.PAGE })
@Component
export struct PageOne {

  build() {
  }
}
```

## 6.2 拦截器标签@HMInterceptor

标记在实现了`IHMInterceptor`的对象上，声明此对象为一个拦截器

- interceptorName: string, 拦截器名称，必填
- priority: number, 拦截器优先级，数字越大优先级越高，非必填，默认为9；
- global: boolean, 是否为全局拦截器，当配置为true时，所有跳转均过此拦截器；默认为false，当为false时需要配置在@HMRouter的interceptors中才生效。

**执行时机：**

在路由栈发生变化前，转场动画发生前进行回调。 1.当发生push/replace路由时，pageUrl为空时，拦截器不会执行，需传入pageUrl路径；

2.当跳转pageUrl目标页面不存在时，执行全局以及发起页面拦截器，当拦截器未执行DO_REJECT时，然后执行路由的onLost回调

3.当跳转pageUrl目标页面存在时，执行全局，发起页面和目标页面的拦截器；

**拦截器执行顺序：**

1. 按照优先级顺序执行，不区分自定义或者全局拦截器，优先级相同时先执行@HMRouter中定义的自定义拦截器
2. 当优先级一致时，先执行srcPage>targetPage>global

> srcPage表示跳转发起页面。
>
> targetPage表示跳转结束时展示的页面。

示例：

```typescript
@HMInterceptor({
  priority: 9,
  interceptorName: 'LoginInterceptor'
})
export class LoginInterceptor implements IHMInterceptor {
  handle(info: HMInterceptorInfo): HMInterceptorAction {
    if (isLogin) {
      // 跳转下一个拦截器处理
      return HMInterceptorAction.DO_NEXT;
    } else {
      HMRouterMgr.push({
        pageUrl: 'loginPage',
        param: { targetUrl: info.targetName },
        skipAllInterceptor: true
      })
      // 拦截结束，不再执行下一个拦截器，不再执行相关转场和路由栈操作
      return HMInterceptorAction.DO_REJECT;
    }
  }
}
```

## 6.3 生命周期标签@HMLifecycle

```
@HMLifecycle(lifecycleName, priority, global)
```

标记在实现了IHMLifecycle的对象上，声明此对象为一个自定义生命周期处理器

- lifecycleName: string, 自定义生命周期处理器名称，必填
- priority: number, 生命周期优先级，数字越大优先级越高，非必填，默认为9；
- global: boolean, 是否为全局生命周期，当配置为true时，所有页面生命周期事件会转发到此对象；默认为false

**生命周期触发顺序：**

按照优先级顺序触发，不区分自定义或者全局生命周期，优先级相同时先执行@HMRouter中定义的自定义生命周期

示例：

```typescript
@HMLifecycle({ lifecycleName: 'exampleLifecycle' })
export class ExampleLifecycle implements IHMLifecycle {
}
```

## 6.4  转场动画标签 @HMAnimator

标记在实现了IHMAnimator的对象上，声明此对象为一个自定义转场动画对象

- animatorName: string, 自定义动画名称，必填。

示例：

```typescript
@HMAnimator({ animatorName: 'exampleAnimator' })
export class ExampleAnimator implements IHMAnimator {
  effect(enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle): void {
  }
}
```

## 6.5  服务标签 @HMServiceProvider

标记在类上，声明此类为一个服务

- serviceName: string，服务名称，必填。
- singleton: boolean，是否是单例，非必填，默认为false

示例：

```typescript
@HMServiceProvider({ serviceName: ServiceConstants.CLASS_SERVICE, singleton: true })
export class CustomService implements IService {
  testConsole(): void {
    Logger.info('Calling service testConsole');
  }
}
```

使用`HMRouterMgr.getService()`进行调用

```typescript
const res = HMRouterMgr.getService<IService>(ServiceConstants.CLASS_SERVICE).testFunWithReturn()
```

## 6.6  服务标签 @HMService

标记在类的方法上，声明此方法为一个服务

- serviceName: string，服务名称，必填。
- singleton: boolean，是否是单例，非必填，默认为false

示例：

```typescript
export class ExampleClass {
  @HMService({ serviceName: 'ExampleService', singleton: true })
  exampleFun(params: string): void {
  }
}
```

# 7. MRouter接口和属性列表

## 7.1 HMNavigation, 路由容器组件

| 参数             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| navigationId     | 需要开发者指定，并确保全局唯一，否则运行时输出报错日志       |
| homePageUrl      | 需要与@HMRouter(pageUrl）中的pageUrl一致，表示HMNavigation默认加载的页面。 |
| navigationOption | 指定该HMNavigation的全局参数选项NavigationOption，保留可扩展性。 |

### 7.1.1 HMNavigationOption

| 参数                                | 说明                   |
| ----------------------------------- | ---------------------- |
| modifier:AttributeUpdater           | Navigation动态属性设置 |
| standardAnimator:IHMAnimator.Effect | 页面全局动画配置       |
| dialogAnimator:IHMAnimator.Effect   | 弹窗全局动画配置       |
| title : NavTitle                    | 系统Title设置          |
| menus: Array \| CustomBuilder       | 系统Menu设置           |
| toolbar: Array \| CustomBuilder     | 系统Toolbar设置        |
| systemBarStyle: Optional            | 系统SystemBar设置      |

## 7.2 HMRouterMgr类

核心类，提供初始化方法，日志使能方法，路由能力

| 接口                               | 参数                                                         | 返回值                                               | 接口描述                                                     |
| ---------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------ |
| static init                        | config: HMRouterConfig                                       | void                                                 | 初始化HMRouter，采用多线程初始化，将标签里面的页面与拦截器、转场动画、生命周期的映射加载到内存 |
| static openLog                     | level : 'DEBUG'\|'INFO'                                      | void                                                 | 使能HMRouter日志。level为info时，表示打开Info日志，level为debug时，表示打开info和debug日志。论述是否调用openLog，warnning和error日志均正常打印。 |
| static push                        | pathInfo: HMRouterPathInfo, callback? HMRouterPathCallback   | void                                                 | 实现路由跳转，提供跳转回调                                   |
| static replace                     | pathInfo: HMRouterPathInfo, callback?: HMRouterPathCallback  | void                                                 | 实现路由跳转，提供跳转回调                                   |
| static pop                         | pathInfo?: HMRouterPathInfo, skipedLayerNumber?: number      | void                                                 | 实现路由返回, skipedLayerNumber返回是跳跃的页面层数，默认为0（表示返回上级页面，1表示跳过一级页面返回，即同时两个页面出栈），以HMRouterPathInfo.pageUrl为首选，skipedLayerNumber为次选 |
| static getPathStack                | navigationId: string                                         | NavPathStack                                         | 根据navigationId获取对应的路由栈，如果没有就生成一个。HMRouter中需要保存navigationId与路由栈映射关系。 |
| static getCurrentParam             | type?: HMParamType                                           | HMPageParam \| Map<string, Object> \| Object \| null | 获取当前页面路由参数, type参数传值对应四种可能性： 1. 当传入type为空时，则返回Object对象，为路由跳转时param参数内容； 2. 当传入参数type=HMParamType.all时，返回HMPageParam， 3. 当传入参数type=HMParamType.urlParam时,返回Map<string, Object> 4. 当传入type=HMParamType.routeParam时，返回对象和情形1一样 |
| static getCurrentLifecycleOwner    |                                                              | IHMLifecycleOwner                                    | 获取当前页面的生命周期托管者实例                             |
| static registerGlobalInterceptor   | interceptor: IHMInterceptor                                  | void                                                 | 注册全局拦截器，interceptor对象包含IHMInterceptor、interceptorName、priority；等同于@HMInterceptor申明全局拦截器 |
| static unRegisterGlobalInterceptor | interceptorName                                              | boolean                                              | 注销全局拦截器，参数为拦截器名称，返回是否注销成功           |
| static registerGlobalLifecycle     | lifecycle: IHMLifecycle                                      | void                                                 | 注册全局生命周期，lifecycle对象包含IHMLifecycle、lifecycleName、priority；等同于@HMLifecycle申明全局生命周期 |
| static unRegisterGlobalLifecycle   | lifecycleName                                                | boolean                                              | 注销全局生命周期，参数为生命周期名称，返回是否注销成功       |
| static registerGlobalAnimator      | navigationId: string, key: 'standard'\|'dialog', animator: IHMAnimator | void                                                 | 注册全局动画，key为路由页面类型，standard时注册动画针对标准路由页面生效，dialog时注册动画针对dialog类型路由页面生效 |
| static unRegisterGlobalAnimator    | navigationId: string, key: 'standard'\|'dialog'              | boolean                                              | 注销全局动画，key为路由页面类型，standard为普通路由页面，dialog为弹窗类型路由页面 |
| static registerPageBuilder         | HMPageInstance                                               | boolean                                              | 动态注册路由信息，等同于@HMRouter申明                        |
| static generatePageLifecycleId     |                                                              | string                                               | 生成页面生命管理实例唯一标识                                 |
| static getPageLifecycleById        | pageLifecycleId                                              | HMPageLifecycle                                      | 页面生命周期实例                                             |
| static request                     | serviceName, ...args: Object[]                               | HMServiceResp                                        | 调用@HMService声明的服务                                     |

### 7.2.1 HMRouterConfig类

初始化配置

| 参数                       | 说明                                          |
| -------------------------- | --------------------------------------------- |
| context:UIAbilityContext   | UIAbility上下文，用于初始化时读取路由配置信息 |
| initWithTaskPool?: boolean | 是否使用taskpool进行多线程初始化，默认为开启  |

### 7.2.2 HMRouterPathInfo类

路由参数，用于路由跳转/返回参数。

| 参数                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| navigationId?        | 根页面名称，对应的是Navigation的名称/ID，当navigationId为null时，表示为对最近一次navigation组件内进行路由跳转。 |
| pageUrl?             | 路由页面名称，对应的是NavDestination的名称，@HMRouter(pageUrl, isRegex, regexPriority, dialog, interceptor, animator, lifecycle)中的pageUrl，push时必须传入，否则会打印错误日志 |
| param?               | 传递的参数，当调用push时表示传递给下页面的参数对象，当调用pop时表示回传给上一页面的返回参数对象。 |
| animator?            | 自定义动画，传入使用此自定义动画执行转场，不再使用原先定义的转场。如果为false时，则不触发动画 |
| interceptors?        | 自定义拦截器，最高优先级执行。                               |
| lifecycle?           | 自定义生命周期，最高优先级执行。                             |
| skipAllInterceptors? | 是否跳过所有拦截器，boolean类型                              |

### 7.2.3 HMRouterPathCallback类

提供路由完成回调

| 参数      | 说明                                |
| --------- | ----------------------------------- |
| onResult  | 页面返回回调，回调参数类型HMPopInfo |
| onArrival | 目标页面跳转完成回调                |
| onLost    | 目标页面找不到回调                  |

> onResult回调可以在每次返回该页面时触发

### 7.2.4 HMPopInfo类

| 参数        | 说明                                 |
| ----------- | ------------------------------------ |
| srcPageInfo | name:返回的源页面，param: 源页面参数 |
| info        | 系统NavPathInfo对象                  |
| result      | 返回携带的参数                       |

### 7.2.5 HMParamType枚举

获取页面参数接口传参类型

| 枚举值     | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| all        | 接口返回HMPageParam对象，包含页面所有参数，HMPageParam.data为路由跳转时传入的param参数, HMPageParam.paramsMap为解析url路径参数，包含pathParam和queryParam内容 |
| urlParam   | 接口返回Map<string, Oject>对象，内容通过解析url获取，例如pageUrl定义/path/id,路由跳转时参数/path/id?name=xxx，则map包含name=123 |
| routeParam | 接口返回Object对象，接口返回内容为路由时传入的HMRouterPathInfo的param对象 |

### 7.2.6 HMPageParam类

获取页面参数接口返回值类型

| 属性     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| data     | Object或者null类型，为路由时传入的HMRouterPathInfo的param对象 |
| urlParam | 接口返回Map<string, Object>对象，解析路由时pageUrl路径获取参数 |

### 7.2.7 HMPageInstance类

用于动态注册路由信息

| 参数              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| builder           | 路由页面内容，需要使用NavDestination组件包裹，WrappedBuilder类型 |
| pageUrl           | 路由页面名称， 同@HMRouter pageUrl                           |
| interceptorArray? | 页面对应拦截器，同@HMRouter interceptors                     |
| singleton?        | 页面是否单例，同@HMRouter singleton                          |

### 7.2.8 HMPageLifecycle类

@Entry页面生命周期管理类

| 方法          | 说明                                       |
| ------------- | ------------------------------------------ |
| onDisAppear   | 页面销毁时触发，绑定页面销毁时回调         |
| onShown       | 页面显示时触发，绑定页面显示时回调         |
| onHidden      | 页面隐藏时触发，绑定页面隐藏时回调         |
| onBackPressed | 页面侧滑返回时触发，绑定页面侧滑返回时回调 |

## 7.3 IHMInterceptor, 拦截器接口

拦截器接口，所有`@HMInterceptor`申明的拦截器需要实现此接口。

| 接口   | 参数              | 返回值              | 接口描述                                               |
| ------ | ----------------- | ------------------- | ------------------------------------------------------ |
| handle | HMInterceptorInfo | HMInterceptorAction | 执行时机：在路由栈发生变化前进行回调，转场动画发生前。 |

### 7.3.1 HMInterceptorInfo类

拦截器获取到的数据对象，包含如下信息

| 参数               | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| srcName            | 发起页面名称                                                 |
| targetName         | 目标页面名称                                                 |
| isSrc              | 是否是发起页面                                               |
| type               | 路由跳转类型，push，replace，pop                             |
| routerPathInfo     | 路由跳转信息，HMRouterPathInfo                               |
| routerPathCallback | 路由跳转回调，HMRouterPathCallback                           |
| context            | UIContext，用来对UI界面进行操作，系统提供，每个navigationId对应一个UIContext，在路由跳转时提供给拦截器。 |

### 7.3.2 HMInterceptorAction枚举

handle方法返回值，表示此拦截器完成后的下一步动作。

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| DO_NEXT       | 继续执行下一个拦截器。                                       |
| DO_REJECT     | 停止执行下一个拦截器，并且不执行路由跳转动画，不执行路由栈操作 |
| DO_TRANSITION | 跳过后续拦截器，直接执行路由转场动画，执行路由栈操作         |

## 7.4 IHMLifecycle, 自定义生命周期接口

自定义生命周期接口，所有`@HMLifecycle`申明的自定义生命周期处理器需要实现此接口。

| 接口            | 参数                    | 返回值  | 接口描述                                                     |
| --------------- | ----------------------- | ------- | ------------------------------------------------------------ |
| onPrepare       | ctx: HMLifecycleContext | void    | 触发时机：在拦截器执行后，路由栈真正push前触发               |
| onAppear        | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onAppear`事件中触发回调。       |
| onDisAppear     | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onDisAppear`事件中触发回调。    |
| onShown         | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onShown`事件中触发回调。        |
| onHidden        | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onHidden`事件中触发回调。       |
| onWillAppear    | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onWillAppear`事件中触发回调。   |
| onWillDisappear | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onWillDisappear`事件中触发回调。 |
| onWillShow      | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onWillShow`事件中触发回调。     |
| onWillHide      | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onWillHide`事件中触发回调。     |
| onReady         | ctx: HMLifecycleContext | void    | 触发时机：在NavDestination的`onReady`事件中触发回调。        |
| onBackPressed   | ctx: HMLifecycleContext | boolean | 触发时机：在NavDestination的`onBackPressed`事件中触发回调    |

### 7.4.1 HMLifecycleContext类

提供context扩展

> onWillAppear声明周期回调中无法获取到navContext

| 参数       | 说明                            |
| ---------- | ------------------------------- |
| uiContext  | 提供UIContext上下文             |
| navContext | 提供NavDestinationContext上下文 |

### 7.4.2 IHMLifecycleOwner接口

生命周期托管者实例

| 接口         | 参数                                                         | 返回值                    | 描述 ｜                        |
| ------------ | ------------------------------------------------------------ | ------------------------- | ------------------------------ |
| getLifecycle |                                                              | IHMLifecycle \| undefined | 获取开发者定义的生命周期实例   |
| addObserver  | state:HMLifecycleState,callback: (ctx: HMLifecycleContext) => boolean \| void | void                      | 注册观察者跟随指定生命周期触发 |

> 由于addObserver添加生命周期观察的时机，可能在方法调用前相关的生命周期已回调结束，需要开发者注意注册观察者的时机

## 7.5 IHMAnimator, 自定义转场动画接口

自定义动画接口，所有`@HMAnimator`申明的自定义动画需要实现此接口，当页面指定了自定义动效的页面将不在执行HMNavigation中指定的默认动效；

页面定义动画对应页面2类情况：

1.页面入场动画，针对路由跳转的目标页面

- 主动入场，A页面push/replace到B页面，B页面属于主动入场，B页面执行动画过程：`enterHandle.start,enterHandle.finish,enterHandle.onFinish`或者`enterHandle.customAnimation`
- 被动入场，B页面pop到A页面，A页面属于被动入场，A页面执行的动画过程: `enterHandle.passiveStart,enterHandle.passiveFinish,enterHandle.passiveOnFinish`或者`enterHandle.passiveCustomAnimation`

2.页面出场动画，针对路由跳转时源页面执行的动画

- 主动出场，B页面pop到A页面，B页面属于主动出场，B页面执行的动画过程：`exitHandle.start,exitHandle.finish,exitHandle.onFinish`或者`exitHandle.customAnimation`
- 被动出场，A页面push/replace到B页面，A页面属于被动出场，A页面执行动画过程：`exitHandle.passiveStart,exitHandle.passiveFinish,exitHandle.passiveOnFinish`或者`exitHandle.passiveCustomAnimation`

| 接口        | 参数                                                        | 返回值 | 接口描述                         |
| ----------- | ----------------------------------------------------------- | ------ | -------------------------------- |
| effect      | enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle | void   | 声明入场和出场动画的定义         |
| interactive | handle: HMAnimatorHandle                                    | void   | 跟手转场（可交互式转场）动画定义 |

### 7.5.1 HMAnimatorHandle类

动画处理器

| 参数                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| start                  | 定义组件在动画开始前的状态                                   |
| finish                 | 定义组件在动画结束后的状态                                   |
| onFinish               | 定义组件在动画结束后保持的最终状态                           |
| customAnimation        | 自定义组件动画过程，设置之后start,finish,onFinish不再生效，页面动画通过自定义动画控制 |
| passiveStart           | 定义组件被动触发动画开始前的状态                             |
| passiveFinish          | 定义组件被动触发动画结束后的状态                             |
| passiveOnFinish        | 定义组件被动触发动画结束后保持的最终状态                     |
| passiveCustomAnimation | 自定义组件被动触发动画过程，设置之后passiveStart,passiveFinish,passiveOnFinish不再生效，页面动画通过自定义动画控制 |
| timeout                | 定义动画超时时间                                             |
| curve                  | 动画曲线                                                     |
| duration               | 动画持续时间                                                 |
| interactive            | 是否可交互式转场                                             |
| actionStart            | 手势触发的回调                                               |
| updateProgress         | 更新转场进度的回调                                           |
| actionEnd              | 手势结束的回调                                               |

动画处理回调

| 参数            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| translateOption | [TranslateOption对象](https://gitee.com/link?target=https%3A%2F%2Fdeveloper.huawei.com%2Fconsumer%2Fcn%2Fdoc%2Fharmonyos-references-V5%2Fts-universal-attributes-transformation-V5%23translateoptions%E5%AF%B9%E8%B1%A1%E8%AF%B4%E6%98%8E)，定义页面的位移参数，只支持x，y轴 |
| scaleOption     | [ScaleOption对象](https://gitee.com/link?target=https%3A%2F%2Fdeveloper.huawei.com%2Fconsumer%2Fcn%2Fdoc%2Fharmonyos-references-V5%2Fts-universal-attributes-transformation-V5%23scaleoptions%E5%AF%B9%E8%B1%A1%E8%AF%B4%E6%98%8E)，定义页面的缩放参数 ，只支持x，y轴 |
| opacityOption   | OpacityOption对象，定义页面的透明度参数，属性：opacity取值范围为0到1，1表示不透明，0表示完全透明 |
| extendOption    | Any对象，开发者自定义参数，通过模版传入                      |
| proxy           | [系统NavigationTransitionProxy对象](https://gitee.com/link?target=https%3A%2F%2Fdeveloper.huawei.com%2Fconsumer%2Fcn%2Fdoc%2Fharmonyos-references-V5%2Fts-basic-components-navigation-V5%23navigationtransitionproxy-11) |

### 7.5.2 IHMAnimator.Effect类

内置转场动画定义

| 构造函数     | 说明                                   |
| ------------ | -------------------------------------- |
| effectOption | IHMAnimator.EffectOptions 动画参数定义 |

| 函数       | 说明                   |
| ---------- | ---------------------- |
| toAnimator | 转换成内置转场动画对象 |

### 7.5.3 IHMAnimator.EffectOptions 类

| 参数      | 说明             |
| --------- | ---------------- |
| direction | 转场方向定义     |
| opacity   | 透明度初始值定义 |
| scale     | 缩放初始值定义   |

# 8. 基于HMRouter路由框架的页面跳转开发实践

## 8.1 页面跳转场景

###  8.1.1 页面跳转与返回

HMRouter提供了基于自定义注解的页面跳转与返回功能，使用步骤如下:

1. 为需要跳转的页面添加@HMRouter注解，并配置其中的pageUrl参数，例如此处配置为ProductContent。

   ```typescript
   @HMRouter({ pageUrl: 'ProductContent' })
   @Component
   export struct ProductContent {
     // ...
   }
   ```

2. 在需要进行页面跳转的位置，使用HMRouterMgr提供的push/replace方法进行页面跳转，在参数中配置目标页面的pageUrl，param参数等，例如下述代码配置pageUrl为ProductContent，并传递了相关参数。此处也可以配置页面栈唯一标识navigationId，当使用多个HMNavigation时建议开发者手动指定，当使用单个HMNavigation时，开发者可以不传递navigationId参数，系统会默认处理。同时HMRouter对push/replace方法还做了增强，可以传递第二个参数，在其中配置返回到当前页面时数据接受的回调函数onResult，只要有页面返回到该页面时都会触发该函数，该函数会接收一个参数，可以通过该参数上的srcPageInfo.name获取到由哪个页面跳转到当前页，还可以从该参数上的result属性获取到其他页面pop到当前页面时传递的参数。

   ```typescript
   HMRouterMgr.push({
     navigationId: "mainNavigationId",
     pageUrl: 'ProductContent',
     param: { a: 1, b: 2 },
     animator: new CustomAnimator(),
   }, {
     onResult(popInfo: HMPopInfo) {
       const pageName = popInfo.srcPageInfo.name;
       const params = popInfo.result;
       console.log(`page name is ${pageName}, params is ${JSON.stringify(params)}`);
     }
   })
   ```

3. 在跳转的目标页面使用HMRouterMgr.getCurrentParam()获取到传递的页面参数。

   ```typescript
   @Component
   export struct ProductContent {
     // ...
     @State param: ParamsType | null = null;
   
     aboutToAppear(): void {
       this.param = HMRouterMgr.getCurrentParam() as ParamsType;
     }
   
     // ...
   }
   ```

4. 如需使用页面返回功能，在对应的业务逻辑位置使用HMRouterMgr提供的pop方法实现页面返回，同样的pop方法支持传入navigationId，同时HMRouter还支持在返回时通过配置param参数向其所返回的页面传递参数。

   ```typescript
   HMRouterMgr.pop({ navigationId: 'mainNavigationId', param: this.param })
   ```

### 8.1.2 多次页面跳转，返回指定页面

当页面跳转路径如HomePage->PageA->PageB->PageC，开发者希望在PageC的页面逻辑中直接返回到HomePage并携带参数，开发者仅需使用HMRouterMgr提供的pop方法，传入要返回目标页面的pageUrl、传递的参数param，即可直接带参返回到指定页面。

```typescript
HMRouterMgr.pop({ navigationId: 'mainNavigationId', pageUrl: 'HomePage', param: this.param })
```

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtyPub/011/111/111/0000000000011111111.20250430151054.96137689936291440121477664508481:50001231000000:2800:C4ABF2F17F2B64E4D81FCF36115F8A46CD857F1878772893CB246ABE8C3F21DD.png)

### 8.1.3 应用未登录，点击跳转登录页的校验场景

应用中经常会有当用户未登录应用时，点击某些应用内容会自动跳转到登录页面的场景，在使用HMRouter对此场景进行实现时，可以采用以下步骤：

1. 定义拦截器类LoginCheckInterceptor实现IHMInterceptor接口。

2. 为定义的拦截器类添加@HMInterceptor注解，通过interceptorName配置拦截器名称LoginCheckInterceptor。

3. 实现IHMInterceptor的handle方法，在该方法中根据当前的登录状态来控制页面跳转的目标。

   - 当用户已登录，通过返回HMInterceptorAction.DO_NEXT，正常执行后续页面跳转逻辑。

   - 当用户未登录，通过Toast弹窗向用户提示登录，然后跳转到登录页面，最后通过HMInterceptorAction.DO_REJECT来拦截此次跳转请求。

     ```typescript
     @HMInterceptor({ interceptorName: 'LoginCheckInterceptor' })
     export class LoginCheckInterceptor implements IHMInterceptor {
       handle(info: HMInterceptorInfo): HMInterceptorAction {
         // ...
           if (!!AppStorage.get('isLogin')) {
             return HMInterceptorAction.DO_NEXT;
           } else {
             info.context.getPromptAction().showToast({ message: '请先登录' })
             HMRouterMgr.push({
               pageUrl: 'loginPage',
               param: info.targetName,
               skipAllInterceptor: true
             })
             return HMInterceptorAction.DO_REJECT;
           }
           // ...
       }
     }
     ```

4. 在需要进行拦截的页面中配置@HMRouter的interceptors参数即可，由于一个页面可以配置多个拦截器，所以需要将关联的拦截器名称封装为一个数组进行传入。

   ```typescript
   @HMRouter({
     pageUrl: 'shoppingBag',
     singleton: true,
     interceptors: ['LoginCheckInterceptor'],
     lifecycle: 'requestLifecycle'
   })
   @Component
   export struct ShoppingBagContent {
     // ...
   }
   ```

### 8.1.4 实现单例页面的跳转

当应用中存在初始化加载资源消耗大且有复用需求的页面时，就可以使用单例页面。典型的业务场景如视频类应用中的视频播放页面，此欸页面通常需要加载视频解码器资源并对其初始化，且该页面在视频类应用中会频繁出现。实现上只需要配置@HMRouter注解参数中的singleton参数为true即可。

```typescript
@HMRouter({
  pageUrl: 'liveHome',
  singleton: true,
  animator: 'liveInteractiveAnimator',
  lifecycle: 'liveHomeLifecycle'
})
@Component
export struct LiveHome {
  // ...
}
```

## 8.2 弹窗提示场景

### 8.2.1 实现弹窗类型的页面

在HMRouter路由框架中，开发者只需要设置@HMRouter注解的dialog配置为ture即可将当前页面作为弹窗使用。

```typescript
@HMRouter({ pageUrl: 'privacyDialog', dialog: true })
@Component
export struct PrivacyDialogContent {
  // ...
}
```

### 8.2.2 返回时弹窗，提示用户是否确认返回

当从某些页面返回时，应用希望通过弹窗方式让用户确认是否要执行返回操作，例如在订单支付页面中用户执行返回操作时，通常会弹窗提示用户是否确认退出，当用户点击确认后才会执行页面退出逻辑，此场景下就可以考虑使用弹窗类型页面加上自定义生命周期来实现。操作步骤如下：

1. 开发者首先需要根据自己的业务需求，来进行自定义弹窗的开发。

   ```typescript
   @HMRouter({ pageUrl: 'PayCancel', dialog: true })
   @Component
   export struct PayCancel {
     // ...
     build() {
       Stack({ alignContent: Alignment.Center }) {
         // ...
         ConfirmDialog({
           title: '取消订单',
           content: '您确认要取消此订单吗?',
           leftButtonName: '再看看',
           rightButtonName: '取消订单',
           leftButtonFunc: () => {
             HMRouterMgr.pop({
               navigationId: this.queryNavigationInfo()?.navigationId
             })
           },
           rightButtonFunc: () => {
             // ...
           }
         })
       }
       .width('100%')
       .height('100%')
     }
   }
   ```

2. 定义ExitPayLifecycle类来实现IHMLifecycle接口，为ExitPayLifecycle加上@HMLifecycle注解，传入生命周期名称ExitPayLifecycle，在类的内部，重写onBackPressed回调函数，当用户执行返回操作时，该回调函数触发，弹出刚刚定义的PayCancel弹窗。

   ```typescript
   @HMLifecycle({ lifecycleName: 'ExitPayLifecycle' })
   export class ExitPayLifecycle implements IHMLifecycle {
     model: ObservedModel = new ObservedModel();
   
     onBackPressed(): boolean {
       HMRouterMgr.push({ pageUrl: 'PayCancel', param: this.model.pageUrl });
       return true;
     }
   }
   ```

3. 将定义的生命周期与支付页面绑定，只需要将刚刚定义的生命周期传入对应组件@HMRouter注解的lifecycle参数即可。

   ```typescript
   @HMRouter({
     pageUrl: 'PayDialogContent',
     dialog: true,
     lifecycle: 'ExitPayLifecycle',
     interceptors: ['LoginCheckInterceptor']
   })
   @Component
   export struct PayDialogContent {
     // ...
   }
   ```

### 8.2.3 首页两次返回退出应用

该场景下用户第一次触发应用返回退出时向用户提示“再次返回退出”，第二次用户触发返回操作时应用真正退出。实现上可参考以下步骤：

1. 定义一个生命周期类ExitAppLifecycle实现IHMLifecycle接口。

2. 使用@HMLifecycle注解传入生命周期名称参数lifecycleName为ExitAppLifecycle。

3. 重写其中的onBackPressed方法（此处是由于上述业务场景需要，实际开发中根据实际业务场景按需重写方法），通过判断上次返回操作与当前返回操作的时间间隔，按如下逻辑处理：

   1. 当两次返回操作的时间间隔大于设置值时（此处为1000ms），重新弹窗对用户进行提示，此处返回true，表示不执行默认返回逻辑。
   2. 当两次返回操作的时间间隔小于设置值时（此处为1000ms），返回为false表示执行默认返回逻辑，退出应用。

   ```typescript
   @HMLifecycle({ lifecycleName: 'ExitAppLifecycle' })
   export class ExitAppLifecycle implements IHMLifecycle {
     lastTime: number = 0;
   
     onBackPressed(ctx: HMLifecycleContext): boolean  {
       let time = new Date().getTime();
       if (time - this.lastTime > 1000) {
         this.lastTime = time;
         ctx.uiContext.getPromptAction().showToast({
           message: '再次返回退出应用',
           duration: 1000,
         });
         return true;
       } else {
         return false;
       }
     }
   }
   ```

4. 将定义好的生命周期类与页面进行关联，开发者只需在@HMRouter注解中配置lifecycle为要关联的生命周期名称即可。

   ```typescript
   @HMRouter({ pageUrl: 'HomeContent', singleton: true, lifecycle: 'ExitAppLifecycle' })
   @Component
   export struct HomeContent {
     // ...
   }
   ```

## 8.3 转场动效场景

### 8.3.1 全局自定义转场动效

- 定义全局页面转场效果。开发者只需要创建出IHMAnimator.Effect实例，在参数中按照业务需求对动画方向direction，透明度opacity，横纵方向页面缩放效果scale进行配置即可。

  ```typescript
  const globalPageTransitionEffect: IHMAnimator.Effect = new IHMAnimator.Effect({
    direction: IHMAnimator.Direction.BOTTOM_TO_TOP,
    opacity: { opacity: 0.5 },
    scale: { x: 0.5, y: 0.2 }
  })
  ```

  定义完成后，只需要将实例传入HMNavigation组件的standarAnimator参数即可

  ```typescript
  HMNavigation({
    navigationId: 'mainNavigationId', homePageUrl: 'HomeContent', options: {
      standardAnimator: globalPageTransitionEffect,
    }
  })
  ```

- 定义全局弹窗效果。同样的，开发者也只需要按照业务需求创建出对应的IHMAnimator.Effect实例，代码示例如下。

  ```typescript
  const globalDialogTransitionEffect: IHMAnimator.Effect = new IHMAnimator.Effect({
    direction: IHMAnimator.Direction.BOTTOM_TO_TOP,
    opacity: { opacity: 1 },
    scale: { x: 1, y: 1 }
  })
  ```

  将创建好的实例作为dialogAnimator的参数进行传入即可。

  ```typescript
  HMNavigation({
    navigationId: 'mainNavigationId', homePageUrl: 'HomeContent', options: {
      dialogAnimator: globalDialogTransitionEffect,
    }
  })
  ```

### 8.3.2 特定页面设置自定义转场

可以自定义动画类并实现IHMAnimator接口中的effect方法，该方法会将页面进出场的效果对象enterHandle与exitHandle作为参数传入，可通过参数对象上的start、finish方法，设置对应效果的起止状态，支持设置的常用属性还有：

- curve：设置动画速度曲线，支持通过Curve枚举传入值，默认Curve.EaseInOut。
- duration：动画持续时长，单位ms。

start/finish方法参数说明如下：

- translateOption：坐标位置，以屏幕左上角为原点，水平向右为x轴正方向，竖直向下为y轴正方向。百分比相对于屏幕宽度。例如希望从右侧进入可以设置translateOption.x从100%变到0。
- scaleOption：页面缩放，可通过scaleOption.x、scaleOption.y单独设置横纵方向的缩放比例。
- opacityOption：跳转页面的透明度。

以下代码示例表示入场时由屏幕底部以线性速度向屏幕顶部运动，入场动画持续时长为400ms。出场时从屏幕顶部以线性速度向屏幕底部运动，出场动画持续时长也为400ms。

```typescript
@HMAnimator({ animatorName: 'CustomAnimator' })
export class CustomAnimator implements IHMAnimator {
  effect(enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle): void {
    // 入场动画
    enterHandle.start((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = '100%'
      scaleOption.x = 0.7;
      opacityOption.opacity = 0.3;
    })
    enterHandle.finish((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = '0'
      scaleOption.x = 1;
      opacityOption.opacity = 1;
    })
    enterHandle.duration = 400;
    enterHandle.curve = Curve.Linear;

    // 出场动画
    exitHandle.start((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = '0'
      scaleOption.x = 1;
      opacityOption.opacity = 1;
    })
    exitHandle.finish((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = '100%'
      scaleOption.x = 0.7;
      opacityOption.opacity = 0.3;
    })
    exitHandle.duration = 400;
    enterHandle.curve = Curve.Linear;
  }
}
```

自定义动画定义完成后，其实例可以作为push/replace方法的animator参数进行传入。

```typescript
HMRouterMgr.push({ pageUrl: 'ProductContent', animator: new CustomAnimator() })
```

### 8.3.3 根据条件呈现不同转场动效

相同的页面可能在不同情况下出现不同的转场效果，常见的有短视频播放时的评论页面弹出时的转场：

- 当短视频横屏播放时，评论页面由右至左弹出，视频向左缩放。
- 当短视频竖屏播放时，评论页面由下至上弹出，视频向上缩放。

此处以评论区组件打开的视角进行动画定义，定义竖屏播放时评论区进出场动画如下：

```typescript
@HMAnimator({ animatorName: 'myAnimator1' })
export class MyAnimator1 implements IHMAnimator {
  effect(enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle): void {
    enterHandle.start((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = '100%';
    }).finish((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = 0;
    })

    exitHandle.start((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = 0;
    }).finish((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.y = '100%';
    })
  }
}
```

定义短视频横屏播放时评论区进出场动画如下：

```typescript
@HMAnimator({ animatorName: 'myAnimator2' })
export class MyAnimator2 implements IHMAnimator {
  effect(enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle): void {
    enterHandle.start((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.x = '100%';
      translateOption.y = 0;
    }).finish((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.x = 0;
    })

    enterHandle.duration = 500;
    exitHandle.start((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.x = 0;
    }).finish((translateOption: TranslateOption, scaleOption: ScaleOption,
      opacityOption: OpacityOption) => {
      translateOption.x = '100%';
    })
    exitHandle.duration = 500;
  }
}
```

最后根据条件选择不同的动效，例如此处根据视频播放方向是否为横向，在页面跳转时使用不同的animator值。

```typescript
@Component
export struct CommentInput {
  // ...
  build() {
    Row() {
      // ...
      Image($r('app.media.icon_comments'))
        .width(24)
        .height(24)
        .margin({ right: 16 })
        .onClick(() => {
          if (this.isLandscape) {
            HMRouterMgr.push({
              navigationId: this.queryNavigationInfo()?.navigationId,
              pageUrl: 'liveComments',
              param: {
                commentRenderNode: this.commentRenderNode,
              },
              animator: myAnimator2
            }, {
              onResult: (paramInfo: PopInfo) => {
                this.videoWidth = '100%';
              }
            })
            this.videoWidth = '50%';
          } else {
            HMRouterMgr.push({
              navigationId: this.queryNavigationInfo()?.navigationId,
              pageUrl: 'liveComments',
              param: {
                commentRenderNode: this.commentRenderNode,
              },
              animator: myAnimator1
            }, {
              onResult: (paramInfo: PopInfo) => {
                this.videoHeight = '100%'
              }
            })
            this.videoHeight = '30%'
          }
        })//
        })// ... 
    }
    // ...
  }
}
```

### 8.3.4 交互式转场

当应用中有页面的进出场效果与用户手势操作同步的诉求时，即当用户手指在屏幕上移动时，页面跟随用户手势移动，可以参考以下实现，通过IHMAnimator的interactive函数控制动画播放进度，在actionStart中判断向右移动执行页面返回操作，在updateProgress更新动画进度，在actionEnd中获取到动画的最终状态，根据最终状态判断是继续执行动画与页面返回还是关闭动画取消页面返回。

```typescript
@HMAnimator({ animatorName: 'liveInteractiveAnimator' })
export class LiveInteractiveAnimator implements IHMAnimator {
  effect(enterHandle: HMAnimatorHandle, exitHandle: HMAnimatorHandle): void {
    // ...
  }

  interactive(handle: HMAnimatorHandle): void {
    handle.actionStart((event: GestureEvent) => {
      if (event.offsetX > 0) {
        HMRouterMgr.pop()
      }
    })
    handle.updateProgress((event, proxy, operation, startOffset) => {
      if (!proxy?.updateTransition || !startOffset) {
        return
      }
      let offset = event.fingerList[0].localX - startOffset;
      if (offset < 0) {
        proxy?.updateTransition(0)
        return;
      }
      let rectWidth = event.target.area.width as number
      let rate = offset / rectWidth
      proxy?.updateTransition(rate)
    })
    handle.actionEnd((event, proxy, operation, startOffset) => {
      if (!startOffset) {
        return
      }
      let rectWidth = event.target.area.width as number
      let rate = (event.fingerList[0].localX - startOffset) / rectWidth
      if (rate > 0.4) {
        proxy?.finishTransition()
      } else {
        proxy?.cancelTransition?.()
      }
    })
  }
}
```

## 8.4 数据加载场景

### 8.4.1 数据请求预加载，与页面跳转并行化

该场景下，希望提前网络请求的位置并在其他线程中执行网络请求而不阻塞主线程，代码实现参考如下步骤。

1. 定义网络请求参数，可使用TaskPool再其他线程执行网络请求并返回请求结果

   ```typescript
   @Concurrent
   async function networkRequest(lifecycle: string): Promise<string> {
     // ...
   }
   ```

2. 定义生命周期，在onPrepare回调函数中，执行对应的网络请求函数，该回调触发时机为拦截器执行后，路由栈真正push前。

   ```typescript
   @HMLifecycle({ lifecycleName: 'requestLifecycle' })
   export class ExampleLifecycle implements IHMLifecycle {
     requestModel: RequestModel = new RequestModel()
   
     onPrepare(): void {
       console.log(this.requestModel.data);
       let task: taskpool.Task = new taskpool.Task(networkRequest, 'onPrepare');
       taskpool.execute(task).then((res: Object) => {
         console.log(res + '');
       })
     }
   
     // ...
   }
   ```

3. 关联生命周期与对应组件。将生命周期的lifecycleName作为@HMRouter注解的lifecycle参数进行传入完成关联。

   ```typescript
   @HMRouter({
     pageUrl: 'shoppingBag',
     singleton: true,
     interceptors: ['LoginCheckInterceptor'],
     lifecycle: 'requestLifecycle'
   })
   @Component
   export struct ShoppingBagContent {
     // ...
   }
   ```

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtyPub/011/111/111/0000000000011111111.20250430151055.79024289896197383284794869039302:50001231000000:2800:03030BB23FF592574B2C1E5820A987A6D74F038113BC4247BE49C6A39E9A2307.png)

### 8.4.2 页面重开数据恢复

该场景下当页面关闭时，之前浏览的相关记录依然存在，典型的场景例如短视频评论，当用户打开评论区页进行翻阅后停留在某处，此时关闭评论区再打开，评论内容会任然停留在上一次浏览的位置。实现上可以参考如下步骤。

1. 使用BuilderNode构造出评论区组件，在makeNode函数中，若评论区不存在则创建，存在便直接返回。

   ```typescript
   @Builder
   function buildComment(liveComments: LiveCommentsProduct[]) {
     // ...
   }
   
   
   
   export class CommentNodeController extends NodeController {
     commentArea: BuilderNode<[LiveCommentsProduct[]]> | null = null;
     commentListData: LiveCommentsProduct[] = new LiveCommentsModel().getLiveCommentsList()
   
     constructor() {
       super();
     }
   
     makeNode(context: UIContext): FrameNode | null {
       if (this.commentArea == null) {
         this.nodeBuild(context)
       }
       return this.commentArea!.getFrameNode();
     }
   
     nodeBuild(context: UIContext) {
       this.commentArea = new BuilderNode(context);
       if (this.commentArea !== null) {
         this.commentArea.build(wrapBuilder<[LiveCommentsProduct[]]>(buildComment), this.commentListData)
       }
     }
   
     dispose() {
       if (this.commentArea !== null) {
         this.commentArea.dispose();
       }
     }
   }
   ```

2. 通过在@HMLifecycle生命周期中，将CommentNodeController类的实例跟随视频播放页面的生命周期创建与释放，而非跟随评论区组件的生命周期创建与释放，使得当用户处在视频播放页时，内存中保存着评论区组件的BuilderNode，从而达成当用户关闭评论区再打开，浏览进度与关闭前一致的诉求。

   ```typescript
   @HMLifecycle({ lifecycleName: 'liveHomeLifecycle' })
   export class liveHomeLifecycle implements IHMLifecycle {
     // ...
     commentRenderNode: CommentNodeController = new CommentNodeController();
   
     onAppear(ctx: HMLifecycleContext): void {
       this.commentRenderNode.makeNode(ctx.uiContext);
     }
   
     onDisAppear(ctx: HMLifecycleContext): void {
       this.commentRenderNode.dispose();
     }
   
     // ...
   }
   ```

3. 在对应的UI组件处获取到生命周期内的commentRenderNode，并在后续业务逻辑中使用NodeContainer进行挂载。

   ```typescript
   @Component
   export struct CommentInput {
     @State commentRenderNode: CommentNodeController =
       (HMRouterMgr.getCurrentLifecycle() as liveHomeLifecycle).commentRenderNode;
   
     // ...
   }
   ```

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtyPub/011/111/111/0000000000011111111.20250430151055.01496212685990482909620114919880:50001231000000:2800:74AA7AA16DE3985363EDED560AA40EFD8DEC4277843076A212A1C959A779159C.png)

## 8.5 维测场景

### 8.5.1 页面埋点开发

当需要统计类似于页面加载耗时等数据，或者有其他自定义打点数据需要统计时，可以使用生命周期回调，在对应的位置进行打点，以下示例为页面停留时长的数据打点统计，实现上参考以下步骤：

1. 定义一个类PageDurationLifecycle实现IHMLifecycle接口。
2. 为该类添加@HMLifecycle注解，并配置global为true，将该生命周期配置到全局，所有页面都会执行该生命周期。
3. 在页面显示时（onShown）记录当前的时间戳，在页面隐藏时（onHidden）计算页面停留时长。

```typescript
@HMLifecycle({ lifecycleName: 'PageDurationLifecycle', global: true })
export class PageDurationLifecycle implements IHMLifecycle {
  private time: number = 0;

  onShown(): void {
    this.time = new Date().getTime();
  }

  onHidden(ctx: HMLifecycleContext): void {
    const duration = new Date().getTime() - this.time;
    console.log(`Page ${ctx.navContext?.pathInfo.name} stay ${duration}`);
  }
}
```


# Stage 模型进阶应用分析报告：AbilityStartMode

## 1. 项目选择

本次选择的开源项目是 OpenHarmony 官方样例仓库中的
`AbilityStartMode`，路径为：

`applications_app_samples/code/BasicFeature/ApplicationModels/AbilityStartMode`

该样例不是只包含单页 UI 的入门程序，而是围绕 Stage 模型中的
`UIAbility` 启动模式展开，包含主页、详情页、多个 Ability、Want 参数传递、
Ability 生命周期日志、指定实例复用逻辑等内容。它适合作为“高级 App”分析对象，
因为它能直接对应课程 PPT 中关于应用包结构、应用组件、任务管理模型和生命周期的
核心概念。

本仓库中的代码副本位于：

`advanced-stage-app/AbilityStartMode`

## 2. 运行环境与验证结果

本机环境：

| 项目 | 内容 |
| --- | --- |
| IDE | DevEco Studio |
| SDK / 系统镜像 | HarmonyOS 6.0.2 / API 22 |
| 模拟器 | Pura 80 Pro |
| HDC 目标 | `127.0.0.1:5555 TCP Connected localhost hdc` |
| Bundle Name | `ohos.samples.startmode` |
| HAP 路径 | `entry/build/default/outputs/default/entry-default-unsigned.hap` |

实际执行结果：

```powershell
ohpm install
hvigor assembleHap --stacktrace
hdc install -r entry/build/default/outputs/default/entry-default-unsigned.hap
hdc shell aa start -b ohos.samples.startmode -a MainAbility
```

验证结论：

- `ohpm install` 成功。
- `hvigor assembleHap --stacktrace` 成功生成 HAP。
- `hdc install -r ...entry-default-unsigned.hap` 返回
  `install bundle successfully`。
- `aa start` 返回 `start ability successfully`。
- `pidof ohos.samples.startmode` 返回进程号 `8022`。
- `aa dump -a` 显示当前任务为
  `#ohos.samples.startmode:entry:MainAbility`，状态为 `FOREGROUND`。
- `bm dump -n ohos.samples.startmode` 显示该应用为
  `isStageBasedModel: true`，并注册了 `MainAbility`、`StandardAbility`、
  `SingletonAbility`、`SpecifiedAbility`。

## 3. 与 PPT 中 Stage 模型概念的对应关系

课程 PPT 将 Stage 模型放在应用程序框架和应用模型的关系中理解：应用模型用于描述
应用组件、生命周期、进程模型、线程模型、任务管理和包管理等方面，而 Stage 是当前
长期演进策略。PPT 还强调 Stage 模型通过 `AbilityStage`、`WindowStage` 等“舞台”
概念来承载应用组件和窗口。

该样例的代码正好覆盖这些点：

| PPT 概念 | 样例中的对应实现 |
| --- | --- |
| 应用包结构 | `AppScope/app.json5`、`entry/src/main/module.json5`、`entry/src/main/resources/` |
| Module 级组件容器 | `entry/src/main/ets/Application/MyAbilityStage.ts` |
| UIAbility 组件 | `MainAbility`、`StandardAbility`、`SingletonAbility`、`SpecifiedAbility` |
| WindowStage | 各 Ability 的 `onWindowStageCreate(windowStage)` |
| 页面与 ArkUI | `pages/Home.ets`、`pages/FoodDetail.ets`、`pages/component/FoodListItem.ets` |
| 任务管理与实例模式 | `module.json5` 中的 `launchType` 配置 |
| UI 与 Ability 分离 | Ability 负责生命周期和启动，ArkUI 页面负责状态驱动的展示 |

PPT 中提到 Stage 模型希望“多设备复用一套生命周期”，并把 UI 和 Ability 进行分离。
在本样例中，`UIAbility` 生命周期方法负责应用组件级逻辑，例如创建窗口、加载页面、
进入前台、进入后台；页面状态和交互逻辑则放在 ArkUI 组件中，通过 `@State`、
`@Link` 和数据模型驱动界面刷新。

## 4. 代码结构分析

关键源码位置如下：

```text
entry/src/main/ets/
|-- Application/MyAbilityStage.ts
|-- MainAbility/MainAbility.ts
|-- StandardAbility/StandardAbility.ts
|-- SingletonAbility/SingletonAbility.ts
|-- SpecifiedAbility/SpecifiedAbility.ts
|-- common/Util.ts
|-- model/DataModels.ts
|-- model/DataUtil.ts
|-- model/MokeData.ts
|-- pages/Home.ets
|-- pages/FoodDetail.ets
`-- pages/component/FoodListItem.ets
```

`MyAbilityStage.ts` 是 Module 级容器。它的 `onCreate()` 在 HAP 初始化时触发；
`onAcceptWant(want)` 用于 `specified` 指定实例启动模式，样例根据
`want.parameters.foodItemId` 返回类似 `SpecifiedAbility0` 的实例 key。
当 key 已存在时，系统复用该实例；不存在时创建新实例。

`MainAbility.ts` 是入口 Ability。它在 `onCreate()` 中通过 `eventHub` 暴露
Ability context 和启动 Want，在 `onWindowStageCreate()` 中调用
`windowStage.loadContent('pages/Home')` 加载主页。

`StandardAbility.ts`、`SingletonAbility.ts` 和 `SpecifiedAbility.ts` 都继承
`UIAbility`，核心区别来自 `module.json5` 中的 `launchType`：

| Ability | launchType | 行为 |
| --- | --- | --- |
| `StandardAbility` | `standard` | 每次启动创建新实例，适合多窗口、多文档一类场景。 |
| `SingletonAbility` | `singleton` | 系统中只保留一个实例，再次启动时触发 `onNewWant()` 并复用页面。 |
| `SpecifiedAbility` | `specified` | 由 `AbilityStage.onAcceptWant()` 返回 key，key 相同则复用，key 不同则新建。 |

`Home.ets` 展示三个食物分类，每个分类对应一种启动模式。用户点击食物后，
`FoodListItem.ets` 通过 `startMode()` 构造 Want 并启动目标 Ability。`FoodDetail.ets`
读取 Want 中的 `foodItemId`、`modeName` 和 `foodListIndex`，再从模拟数据中取出详情。
对于 singleton 模式，`SingletonAbility.onNewWant()` 使用 `emitter` 通知页面刷新数据。

## 5. Stage 生命周期观察

PPT 中列出的生命周期包括 `AbilityStage.onCreate()`、`UIAbility.onCreate()`、
`onWindowStageCreate()`、`onForeground()`、`onBackground()`、`onDestroy()` 等。
样例在这些回调里都写了日志，便于通过 `hilog` 或 `aa dump` 观察实际运行。

在本机模拟器启动入口 Ability 时，运行链路可以概括为：

1. 安装 HAP 后，系统识别 `AppScope/app.json5` 中的 bundle 信息。
2. 启动 `MainAbility` 时，系统读取 `entry/src/main/module.json5` 的 `mainElement`。
3. `MyAbilityStage.onCreate()` 作为模块容器初始化。
4. `MainAbility.onCreate()` 保存 Ability 上下文和启动 Want。
5. `MainAbility.onWindowStageCreate()` 创建窗口舞台并加载 `pages/Home`。
6. `MainAbility.onForeground()` 进入前台。

这与 PPT 中“AbilityStage、WindowStage 作为应用组件和窗口的舞台”的描述一致。

## 6. 启动模式实验理解

该样例的主页把三种启动模式设计成三组食物：

- standard：点击番茄、黄瓜等会不断创建新的详情 Ability。
- singleton：点击冰淇淋、螃蟹等会复用同一个 Ability，并通过 `onNewWant()` 更新页面。
- specified：点击核桃、蓝莓等时，以 `foodItemId` 生成实例 key；同一个 key 复用，
  不同 key 新建。

从任务管理角度看，standard 更接近“每次拉起都是新任务实例”；singleton 更接近
“应用内固定入口复用”；specified 则提供介于二者之间的粒度，由开发者决定实例 key。
这正好对应 PPT 中对 multiton/singleton/specified 启动模式的比较。

## 7. 本次适配修改

原样例较早，README 中说明面向 API 9 / DevEco Studio 3.1。为了在本机
DevEco Studio + HarmonyOS 6.0.2 / API 22 上编译运行，我做了兼容性整理：

- 将工程配置更新到 HarmonyOS 6.0.2 的 `modelVersion`、`targetSdkVersion` 和
  `compatibleSdkVersion`。
- 将旧字段 `srcEntrance` 更新为当前工程使用的 `srcEntry`。
- 为 ArkTS 严格检查补齐组件属性默认值，避免未初始化属性报错。
- 将 `Resource` 类型调整为 `resourceManager.Resource` 或 `ResourceStr`。
- 为 `ForEach`、Want 参数、Emitter 事件数据补充类型。
- 修正主页中的 `sinleton` 拼写为 `singleton`，避免模式名和逻辑不一致。
- 移除页面中把 `null` 传给文本资源参数的写法，改为空字符串。

这些修改不改变样例主题，只是让旧样例在当前 SDK 下可以稳定构建和运行。

## 8. 总结

`AbilityStartMode` 用一个小型食品详情应用展示了 Stage 模型的关键运行机制：
应用由 `AppScope` 和 `module.json5` 描述包结构；`AbilityStage` 是 Module 级入口；
多个 `UIAbility` 承载不同启动模式；`WindowStage` 负责加载 ArkUI 页面；
页面通过 Want 参数和事件机制与 Ability 协作。

通过实际编译、安装、启动和 `aa` / `bm` 检查，可以确认该样例不仅能说明 PPT 中的
概念，也能在 DevEco Studio 模拟器上真实运行。它体现了 Stage 模型相较简单单页
应用更重要的部分：生命周期、任务实例、窗口舞台和 UI/业务逻辑分离。

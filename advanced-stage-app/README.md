# Advanced Stage App Study

本目录补充一个可编译运行的 HarmonyOS Stage 模型进阶样例：
OpenHarmony 官方样例 `AbilityStartMode`。

- 源码目录：`AbilityStartMode/`
- 分析报告：`REPORT.md`
- 原始样例来源：`https://github.com/openharmony/applications_app_samples`
- 样例主题：`standard`、`singleton`、`specified` 三种 `UIAbility`
  启动模式，以及 `AbilityStage`、`WindowStage`、`Want` 参数传递和
  ArkUI 页面协作。

## Build And Run

```powershell
cd advanced-stage-app/AbilityStartMode
ohpm install
hvigor assembleHap --stacktrace
hdc install -r entry/build/default/outputs/default/entry-default-unsigned.hap
hdc shell aa start -b ohos.samples.startmode -a MainAbility
```

已在 DevEco Studio 模拟器 `Pura 80 Pro`（HarmonyOS 6.0.2 / API 22）上验证。

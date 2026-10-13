# Cocos Creator Skills

面向 Cocos Creator 开发与排障的 Codex skills 集合。每个 skill 自包含，结论均注明实测来源（Creator 3.8.8 为主，引擎侧结论另对 4.0-alpha 源码核验过一遍）。

## 按任务选 skill

先按「要动的是哪一层」分流：

| 任务 / 症状 | 用哪个 skill |
|---|---|
| 写、扩展、排查**编辑器扩展**：`package.json` 贡献点（asset-db / inspector / scene / panel / messages）、自定义资源导入器、拖拽进场景或 Inspector、原生 C++ 插件、wasm 交付、模拟器（原生 exe）预览黑屏 | **cocos-creator-plugin** |
| 手写或解析**资源文件格式**：`.anim` / `.prefab` / `.scene` 共用序列化容器、AnimationClip JSON、Spine `skeleton.json` + `.atlas`（3.8 / 4.2 双版本互不兼容）、`.plist` 两代键名、SpriteFrame trim / offset | **cocos-asset-formats** |
| **不开编辑器**创建/修改场景、节点、组件：手写 `.scene` / `.meta`、压缩 uuid `__type__`、静态 UI 序列化进场景、`cc.ClickEvent`、运行时零资源搭 UI | **cocos-creator-scene-authoring** |
| **微信小游戏多线程 Worker**：分包约束、V2 灰度（OffscreenCanvas / SharedArrayBuffer / worker 内 wx API）、wasm 实例化真机铁律、devtools / 真机 A/B 验证 | **wechat-minigame-worker** |

边界口诀：「**插件怎么写**」看编辑器壳、归 plugin skill；「**资源长什么样 / 能不能播**」看引擎、归 formats skill；要直接产出的**场景文件**归 scene-authoring；**微信小游戏的 worker / wasm 通道**归 wechat-minigame-worker。plugin 与 formats 的完整判界见 cocos-creator-plugin 的 SKILL.md 开头路由表。

## Skills

- [`cocos-creator-plugin`](skills/cocos-creator-plugin/SKILL.md) — Creator 3.8.x 编辑器扩展开发全流程：三进程模型、贡献点骨架（照内置扩展抄，附 schema 位置）、asset-db 实测事实（UUID/importer 争抢/刷新时序/自激循环）、静默失败与「报错误地方」排查、包体与物理裁剪、JSB + embind 双后端、模拟器预览 staging 管线与黑屏双根因、消息路由无权限门槛的自动化杠杆、CDP 直连模拟器/预览 V8。
- [`cocos-asset-formats`](skills/cocos-asset-formats/SKILL.md) — 3.8.x 资源数据格式逐字段规则：序列化容器（`__id__` / 内联规则）、手写 AnimationClip、Spine 双版本 atlas 与 vendored spine-cpp 4.3 解析器规则、`.plist` 两代键名、SpriteFrame trim 几何与 offset 的 y 方向、SpriteAtlas 摊平序列化、`compressSettings` 保留。
- [`cocos-creator-scene-authoring`](skills/cocos-creator-scene-authoring/SKILL.md) — 免编辑器场景操作：`.scene` / `.meta` 手写、场景挂脚本组件、静态 UI / 图片资产直接序列化、场景内 `cc.ClickEvent` 点击绑定、运行时零资源搭 UI（含动态图集 / `packable` 与 builtin 白图的 WebGL 崩溃链）、节点定位与移动。
- [`wechat-minigame-worker`](skills/wechat-minigame-worker/SKILL.md) — 微信小游戏 Worker 完整参考：旧版约束与分包、V2 灰度、Cocos wasm 解码器 worker 化约束表、devtools 模拟器与真机「全局存在 ≠ 通道可用」验证法、真机 wasm 实例化铁律、错误链直报 collector 的定位法。

## 目录结构

```text
skills/
└── <skill-name>/
    ├── SKILL.md          # 触发条件与工作流程
    ├── agents/           # Codex UI 元数据
    ├── references/       # 按需加载的详细资料（可选）
    ├── scripts/          # 可重复执行的工具（可选）
    └── assets/           # 输出模板和静态资源（可选）
```

每个 skill 都应保持自包含；仅在实际需要时创建可选目录。

## 使用

将需要的 skill 目录复制或链接到 Codex skills 目录，例如：

```powershell
Copy-Item -Recurse .\skills\cocos-creator-plugin "$env:USERPROFILE\.codex\skills\"
```

随后通过 `$cocos-creator-plugin` 显式调用，或让 Codex 根据任务自动匹配。

## 开发

修改 skill 后运行官方快速校验脚本：

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" `
  .\skills\cocos-creator-plugin
```

Windows 中文（GBK）系统上要先 `$env:PYTHONUTF8 = '1'`，否则校验脚本读含中文的 SKILL.md 会报 `UnicodeDecodeError`。

本仓库不内嵌大型上游源码。Cocos Creator 引擎或 Ultralight WebCore 等代码库应保持独立检出，只作为调查与交叉验证的事实来源。

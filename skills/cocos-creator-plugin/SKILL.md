---
name: cocos-creator-plugin
description: Use when scaffolding, extending, or debugging a Cocos Creator (3.8.x) editor extension/plugin under a project's extensions/ folder — especially one that needs a custom asset type + asset-db importer, generated engine assets (AnimationClip / SpriteAtlas / Spine), drag-and-drop into the scene or Inspector, a custom editor panel, extension-to-extension messages, or a native C++ runtime shared between JSB (native) and Emscripten/embind (wasm). Triggers on requests like "make a Cocos Creator plugin", "custom asset importer for Creator", "editor extension for Cocos", "generate an AnimationClip from code", "add drag-and-drop support to my plugin", "native/wasm binding for a Cocos Creator component", "why doesn't my asset-handler register", "why is my set-property not taking effect", or any work touching contributions.asset-db / contributions.scene / contributions.inspector / contributions.messages in a Creator package.json.
---

# Cocos Creator 插件开发

这份 skill **自包含**：所有骨架代码可以直接抄，所有结论都注明了来源。凡是没验证过的地方都明确写着「未验证」——那种地方要按「猜错也不会碎」的方式写代码，别当成事实。

结论基于 **Creator 3.8.8** 实测。换大版本前先按 §0 把版本确认一遍。

引擎侧的结论额外对着 **Cocos 4.0.0-alpha.31** 源码核过一遍，标了「3.8 / 4.0 一致」的那些可以跨这两个大版本用；没标的只保证 3.8.8。

## 路由：这些问题不在本 skill

本 skill 只管「编辑器扩展怎么写」。同一生态里相邻、但不属于插件开发的问题，按这张表去对应的 skill：

| 问题 | 去哪 |
|---|---|
| **资源数据格式**：手写/解析 `.anim`/`.prefab`/`.scene`（`__id__` 容器、内联规则）、AnimationClip JSON、Spine `skeleton.json` + `.atlas`（双版本格式、vendored spine-cpp 4.3 逐字段规则）、`.plist` 两代键名、SpriteFrame trim/offset、`library/*.json` 里的产物形状 | **cocos-asset-formats** skill |
| **微信小游戏多线程 Worker**：`wx.createWorker`、worker 分包、V2 灰度（OffscreenCanvas / SharedArrayBuffer / worker 内 wx API / 多 worker）、「把解码搬进 worker」行不行 | **wechat-minigame-worker** skill |
| **不开编辑器写场景**：手写 `.scene`/`.meta`、压缩 uuid `__type__`、静态 UI 序列化进场景、`cc.ClickEvent` 点击绑定、运行时建节点/UI | **cocos-creator-scene-authoring** skill |

判断口诀沿用 §0 那条分界线：「插件怎么写」看编辑器版本、在本 skill；「资源长什么样 / 能不能播」是引擎侧格式问题、在上表第一个 skill。

## 章节地图：按任务直达

| 任务 | 直达 |
|---|---|
| 确认编辑器/引擎版本、挖编辑器内部事实 | §0 |
| 想清楚这次改动落在哪个进程 | §1 |
| 选贡献点 / 写 package.json / 主进程 / 面板 | §2–3 |
| 自定义资源类型、导入器、UUID、刷新、引用 | §3.4、§4 |
| 「没反应」「报错误地方」「黑屏 / 卡住」 | §5 |
| 装进目标工程、免 GUI 构建、CLI typecheck | §6 |
| 包体估算、功能裁剪、tree-shaking | §7 |
| 原生 C++（JSB）与 wasm 双后端 | §8 |
| 模拟器（原生 exe）预览 | §9 |

## 0. 动手前先确认版本（两分钟，能省几小时）

### 编辑器壳的版本 ≠ 引擎的版本

这是最容易在毫不知情的情况下踩到的坑。3.8.8 的编辑器可以被配置成加载**别的引擎**，而这个配置是**全局**的、不属于任何工程：

```bash
cat ~/.CocosCreator/profiles/v2/packages/engine.json
# engine.<版本号去掉点>.javascript.custom 有值 → 加载的不是内置引擎
```

确认实际加载了什么：

```bash
head -20 <工程>/temp/logs/project.log
# "Load engine in <路径>" 和 "[Scene] Cocos Creator vX.Y.Z"
# 后者是引擎自己报的版本，可能和编辑器壳的版本号完全不同
```

分界线记牢，它决定一个结论会不会因为换引擎而失效：

| 归属 | 住在哪 | 受自定义引擎影响 |
|---|---|---|
| 导入器、asset-db、`.meta`、扩展消息、面板、菜单 | 编辑器 `app.asar` | **不受影响** |
| 资源反序列化、组件属性名、各种 enum 取值、spine 运行时 | 引擎 | **受影响** |

所以「插件写法」类问题看编辑器版本，「生成的资源能不能播」类问题看引擎版本。

### 引擎的 `package.json` 被换掉会引发一整套看似无关的报错

编辑器靠**引擎目录里的 `package.json`**（`name` / `version` / `uuid`）识别这是哪个引擎。这个文件如果被别的东西覆盖（比如另一个工具误把插件清单写到了引擎目录），编辑器会认不出引擎 → 判定缓存作废 → 重建整个 `library/`。

真正的坑在重建的时序上：**55 个内置 asset handler 是靠 `engine-extends:afterStartDB` 这个钩子懒注册的**，如果重建抢在这个钩子前面跑，所有资源就会被 `*` 兜底导入器（原样拷文件）认领，`.meta` 被就地改写（`importer` 变 `*`、`ver` 掉回 `0.0.1`、`subMetas` 清空）。表现出来的症状和真正原因完全对不上号：`effect` 的 `.json` 产物下载失败、渲染管线读到 null、材质属性「非法」——一套连锁反应，没人会一上来就怀疑到 `package.json`。

判据是 `<工程>/temp/asset-db/log/*.log` 里的两个计数，跟前面《四条自己挖事实的通道》配套用：

```bash
L=$(ls -t <工程>/temp/asset-db/log/*.log | head -1)
grep -c 'lazy register asset handler' "$L"   # 健康 = 55
grep -c 'Can not find the importer'    "$L"  # 健康 = 0
```

**`Can not find the importer <name> in editor` 不是某个导入器单独出问题的信号**——handler 注册整体失败时，这句话会对每一个资源都打一遍，几百次都长一个样，别被引导去查那个具体的 `<name>`。

好消息是这类损坏能自愈：下一次编辑器启动如果没再撞上时序竞态，`.meta` 会被重新写回正确值。所以先看上面两个数再决定要不要动手改文件——数正常说明已经自愈，剩下的 diff 大概率只是无害的键序变动。

### 四条自己挖事实的通道

遇到本 skill 没覆盖的细节，按这个顺序挖，都不需要联网：

1. **引擎源码**（TS，可读）：
   `/Applications/Cocos/Creator/<版本>/CocosCreator.app/Contents/Resources/resources/3d/engine/cocos/`
   组件属性名、enum 取值、序列化字段一律在这里逐字段核对，不要凭记忆。
   **用了自定义引擎就要去自定义引擎那个目录读**，读内置的等于白读。不确定路径时，工程里这两个文件会直接把它写出来：`temp/declarations/cc.d.ts`（一行 `/// <reference path="<引擎>/bin/.declarations/cc.d.ts"/>`）和 `temp/tsconfig.cocos.json`（`paths` 里的 `db://internal/*`）。引擎自己的版本号在它的 `package.json` 里。
2. **`package.json` 的写法：asar 里有明文 JSON schema 和官方模板。** 这是写贡献点时的第一站，见 §3.1 开头。
   ```bash
   npx @electron/asar extract <app.asar> /tmp/ax     # 1.2G，整包解出来最省事
   ls /tmp/ax/modules/editor-extensions/extensions/extension/static/extension-template/
   #   template/@types/schema/package/{index.json,base/panels.json}  ← schema
   #   blank/ html-panel/ vue2.x-panel/ vue3.x-panel/                ← 官方模板
   cat /tmp/ax/builtin/*/package.json                # 19 个内置扩展，明文，贡献点的唯一事实来源
   ```
   **schema 只管顶层字段和 `panels`；`contributions` 的 schema 是故意空的**，所以贡献点形态只能照内置扩展抄（写法依据集中在 §3.1）。
3. **编辑器的类型声明**（在 `app.asar` 里，但 `.d.ts` 是明文）：
   ```bash
   npx @electron/asar list  /Applications/Cocos/Creator/<版本>/CocosCreator.app/Contents/Resources/app.asar | grep '\.d\.ts'
   npx @electron/asar extract-file <app.asar 路径> 'builtin/asset-db/@types/protected/asset-handler.d.ts'
   ```
   `node_modules/@cocos/creator-types/editor/**` 下面是全套编辑器 API 声明。
   ⚠️ **实现是 `.ccc`，加密的**（`asset-handler-manager.ccc`、`libs/importer.ccc`），解不开。所以导入器的内部行为只能靠观测，没有静态替代路径。
   **但别把「编辑器内部读不了」扩大化**（2026-09 实测）：加密的只有 `.ccc`；`@editor/creator` 各模块的 `.jsg` 是 **gzip**（`gunzip` 即明文 JS，消息路由就是这样反编译确认无权限门槛的，见 §9.3）；**builtin 扩展的 `dist/**/*.js` 干脆就是明文**（`builtin/preview/dist/browser/simulator.js` 等直接可读）。查编辑器行为时先看这三层里哪层可读，再谈观测。
4. **真实工程的产物**：`<工程>/library/**/*.json` 是**已被当前引擎序列化过**的资源，是最可靠的格式样本；`<工程>/temp/asset-db/log/*.log` 是导入/注册的真实时间线；`<工程>/temp/logs/project.log` 是扩展加载和面板注册的真实时间线（`warn:` 行往往是唯一的线索，见 §5）。

**`extract-file` 只往 stdout 写、不落盘**，用它取多个文件时记得重定向；一次要看好几个的话直接 `extract` 整包更省事。

## 1. 心智模型：三个不共享内存的进程

| 进程 | 跑什么 | 代码位置 |
|---|---|---|
| 主进程 (main) | `main` 入口、菜单、消息路由、开面板 | `contributions.main` 指向的文件 |
| 场景进程 (scene) | 场景 / 层级 / Inspector 交互，能拿到 `cce.*` | `contributions.scene.script` |
| asset-db worker | 资源导入、解析 | `contributions.asset-db.script` / `asset-handler` |

跨进程只能发消息：

```js
await Editor.Message.request('scene', 'execute-scene-script', {
    name: 'my-extension',        // 必须是 package.json 的 name
    method: 'doThing',           // scene script 导出的方法名
    args: [uuid],
});
```

**动手前先想清楚这次改动落在哪个进程**——最常见的返工就是把该在场景进程做的事写进了主进程。

判断依据很简单：要碰节点 / 组件实例 → 场景进程；要碰文件和 `.meta` → asset-db；只是转发和 UI → 主进程。

## 2. 按需求勾选贡献点

多余的贡献点是纯粹的复杂度，按实际要的能力勾：

| 想要的能力 | 需要的贡献点 |
|---|---|
| 自定义资源类型（`.xxx` → 自定义 Asset 类） | `asset-db.script` + `asset-db.asset-handler` |
| 把 TS 运行时代码交付给游戏（组件 / 资源类） | `asset-db.mount`（只读挂载；**挂进去的每个脚本都进包，见 §7.1**） |
| 资源面板右键菜单 | `assets.menu` |
| 参数弹窗 / 工具窗口 | `panels` |
| 存一份用户配置（偏好 / 工程设置） | `contributions.profile` 或 `contributions.preferences` + `Editor.Profile`（见 §3.9） |
| 拖资源到已有节点的 Inspector 自动加组件 | `inspector.drop` + `messages` + 主进程转发 |
| 拖资源进层级面板自动建节点 | `scene.script` |
| 组件 / 资源自定义 Inspector 面板 | `inspector.section` |
| 被别的扩展或批处理脚本调用 | `messages` 里 `"public": true` |
| 监听资源变动 | `messages` 里订阅 `asset-db:asset-add` / `asset-db:asset-change` |
| 原生 C++ 运行时 | `native.plugins` + `native/cc_plugin.json`（**仅 android / windows / ios / mac**，见 §8.1） |
| Web / 小游戏也要同一套原生逻辑 | 独立 wasm（Emscripten，embind 可选）+ `builder` 贡献复制产物 + `main.js` 拷进引擎 `native/external/`（见 §8.3–8.4） |

**决策规则**：
- 只要涉及「自定义资源类型」，`asset-db.script` + `asset-handler` 是最小闭环。
- **只是要生成引擎已认识的资源（`.anim` / `.plist` / spine 三件套）就不要注册 asset-handler。** 输入后缀（`.png` / `.mp4`）早已被内置导入器认领，抢过来只会制造冲突；参数存在源资源的 `.meta` 里根本不需要自己的 importer。
- 不需要原生运行时就别碰 §8。

## 3. 骨架代码

### 3.1 package.json

**先说依据，因为这一节以前是靠记忆写的、错了三处。** 权威材料就是 §0 通道 2 那两处：`extension-template` 里的 JSON schema（只覆盖 package.json **顶层字段**和 `panels`）与 19 个内置扩展的明文 `package.json`（`builtin/*/package.json`）。**`contributions` 的 schema 是故意留空的**（`"properties": {}`，每个宿主包自定义 key），所以贡献点形态没有 schema，唯一事实来源是内置扩展——下面每一条都注明了照的是哪一个。

```json
{
  "package_version": 2,
  "name": "my-extension",
  "version": "1.0.0",
  "author": "me",
  "editor": ">=3.8.0",
  "main": "./dist/main.js",

  "panels": {
    "my-panel": {
      "title": "我的面板",
      "type": "simple",
      "main": "./dist/panel/index.js",
      "size": { "min-width": 480, "min-height": 560, "width": 520, "height": 640 }
    }
  },

  "contributions": {
    "messages": {
      "open-panel":        { "methods": ["openPanel"] },
      "do-thing":          { "methods": ["doThing"], "public": true },
      "asset-db:asset-add":    { "methods": ["onAssetAdd"] },
      "asset-db:asset-change": { "methods": ["onAssetChange"] }
    },
    "assets": {
      "menu": { "methods": "./dist/assets-menu.js", "assetMenu": "assetMenu" }
    },
    "inspector": {
      "drop": {
        "node": [ { "type": "cc.AnimationClip", "message": "inspector-drop" } ]
      }
    },
    "scene": { "script": "./dist/scene.js" }
  }
}
```

**`panels` 是顶层字段，不在 `contributions` 里面。** 这是 schema 里写明的（`index.json` 的 `properties.panels`，和 `contributions` 平级），也是 19 个内置扩展的一致做法——有面板的那 10 个（animator / scene / project / preferences / shortcuts / builder / preview / messages / tester / animation-graph）全部写在顶层，`contributions.panels` **一个都没有**。

放错了的症状极具误导性，实测踩过：**扩展正常加载、消息正常收发、右键菜单正常出，只有 `Editor.Panel.open()` 报一行 `warn: Panel(<pkg>.<key>) is not defined`。** 因为看起来「扩展是好的」，很容易去查面板脚本、查 `module.exports`、查路径——而错的地方在 `package.json` 的缩进层级里。

其余易错点：

- **`"package_version": 2` 漏了**的症状同样怪：扩展照常加载、消息照常收发，但它在「扩展 → 扩展管理器」列表里**根本不出现**，于是看起来像装失败了。日志里只有一行 `warn: [Package] <name>: package_version is not defined`。实测踩过。
- schema 的 `required` 是 `["author", "name", "package_version", "version"]`——**`author` 是必填的**，容易漏。
- `panels.<key>.type` 的 enum 只有 `"dockable"` 和 `"simple"`，**缺省是 `dockable`**。`required` 是 `["title", "main"]`。
- **`contributions.assets.menu` 是个对象，不是字符串**：`{ "methods": "<js 路径>", "assetMenu": "<导出的函数名>" }`。key `assetMenu` 是钩子名，value 是你在那个文件里导出的函数名，两者可以不同名。照 `builtin/preview/package.json`。
- **`contributions.inspector.drop` 是 `{ node: [...] }`，数组元素是 `{ type, message }`，`type` 是被拖资源的 `@ccclass` 名**（`cc.AnimationClip` / `sp.SkeletonData`），**不是** `"node"`，也**没有 `extname` 字段**——`extname` 在 19 个内置 `package.json` 里出现 **0 次**，是我编的。照 `builtin/animator/package.json`，它是 3.8.8 里唯一用这个贡献点的内置扩展。
- `messages` 的 `methods` 里写的是**主进程模块导出的方法名**，不要加 `default.` 前缀（那是面板方法专属语法，见官方 `html-panel` 模板里的 `"default.hello"`）。
- 订阅别的包广播的消息，key 就写 `<包名>:<消息名>`。
- `"public": true` 才能被 `Editor.Message.request('my-extension', 'do-thing', …)` 从外部调到。

### 3.2 主进程

```ts
// TS 编译到 CommonJS 后，导出必须落在 exports 上。
export const methods = {
    async openPanel(source?: string) {
        await Editor.Panel.open('my-extension', { source });   // 参数会进面板的 update()
    },
    async doThing(payload: { url: string }) {
        // 返回值会成为 Editor.Message.request 的 resolve 值
        return { ok: true };
    },
    async onAssetAdd(uuid: string) { /* ... */ },
};

export function load() {}
export function unload() {}
```

**CPU 密集的活要 fork 出去**，否则编辑器 UI 会冻住：

```ts
import { fork } from 'child_process';
const child = fork(scriptPath, argv, {
    execPath: process.execPath,                       // Electron 自己
    env: { ...process.env, ELECTRON_RUN_AS_NODE: '1' } // 让它当普通 Node 跑
});
```

这样不要求用户系统里另装 Node。

### 3.3 面板

```ts
// 这一行是 module.exports，不是 export default。
// TS 的 `export default` 在 CommonJS 下编译成 exports.default，面板加载器读不到，
// 表现是面板打开一片空白且没有任何报错。
module.exports = Editor.Panel.define({
    template: `<div><ui-button id="go">走</ui-button><div id="out"></div></div>`,
    style: `#out { margin-top: 8px; }`,
    $: { go: '#go', out: '#out' },     // this.$.go / this.$.out
    methods: {
        async run() {
            const r = await Editor.Message.request('my-extension', 'do-thing', { url: '' });
            this.$.out!.innerHTML = String(r.ok);
        },
    },
    ready() {
        // 注意：ready() **不接收任何参数**。
        // 这里重新初始化所有可变状态 —— methods 这个对象字面量在扩展的整个生命周期里
        // 只有一份，面板关掉再打开时上一次的残留还在。
        this.$.go!.addEventListener('confirm', () => void this.run());
    },
    update(args) {
        // Editor.Panel.open(name, ...args) 传的东西**据说**到这里。
        // 但这条没有依据，别指望它：官方模板（见下面）的 define 里没有 update 钩子，
        // 类型声明里 open 也只是 `open(name, ...args: any[]): any`，没说 args 去哪。
    },
});
```

**`Editor.Panel.define` 到底收哪些字段，有官方模板可抄**（比任何记忆都可靠）：

```bash
# asar 里的官方模板，明文 TS
cat /tmp/ax/modules/editor-extensions/extensions/extension/static/extension-template/html-panel/source/panels/default/index.ts
# → listeners{show,hide} / template / style / $ / methods / ready / beforeClose / close
#   没有 update。@editor/panel 的实现是 .ccc，读不了，所以模板就是上限。
```

**要把参数送进已打开的面板，用消息路由，不要靠 `Panel.open` 的实参。** `contributions.messages` 里把 `methods` 写成 `"<面板 key>.<方法名>"` 就直接进面板方法：

```json
"panels": { "baker": { "title": "…", "main": "./dist/panel.js" } },
"contributions": {
  "messages": { "baker:load-source": { "methods": ["baker.loadSource"] } }
}
```

依据是两个内置扩展：`builtin/messages` 用 `["debug.broadcast"]`、`builtin/shortcuts` 用 `["default.changeTab"]`，两个包顶层 `panels` 里都确实有 `debug` / `default` 这个 key。于是主进程侧写成两步：

```ts
await Editor.Panel.open('my-extension.baker');
Editor.Message.send('my-extension', 'baker:load-source', url);   // 开完再送，不赌 open 的实参
```

`ui-*` 控件的值读写走 `el.value`，事件用 `change`（连续变化）和 `confirm`（敲回车 / 失焦），按钮只有 `confirm`。

**藏区域藏不到区域外面的按钮。** 面板常见布局是「表单一大块 + footer 三个按钮」，而「这个资源不能处理」的处置通常是把表单 `display:none`。footer 在表单外面，于是**按钮照样能点**，点了就走进一个参数为空的流程。这类 bug 的报错会精准地指向被点的那个功能（「source 是空的」），而错在**它压根不该点得到**——禁用按钮要单独写一句。

### 3.4 自定义资源导入器（asset-handler）

3.8.8 的 handler 是**结构化**的，`importer` 是个独立子对象——不是一堆平铺字段（来源：`builtin/asset-db/@types/protected/asset-handler.d.ts` 的 `AssetHandlerBase`）：

```js
module.exports = {
    name: 'my-asset',
    assetType: 'MyAsset',            // 运行时类的 @ccclass 名，缺省是 cc.Asset
    // 同后缀被多个 handler 争时靠这个区分，看内容而不是看后缀
    async validate(asset) {
        const text = await fs.promises.readFile(asset.source, 'utf8');
        return text.includes('"myMagicKey"');
    },
    importer: {
        version: '1.0.0',            // 改了会触发全量重导
        async import(asset) {
            const payload = { __type__: 'MyAsset', myField: 1 };
            await asset.saveToLibrary('.json', JSON.stringify(payload));
            return true;             // false = 这次导入没产生变化
        },
    },
};
```

- `payload.__type__` 必须和运行时类的 `@ccclass()` 名字**精确一致**，字段名和 `@property` 一一对应。
- **大段二进制走 `_native` sidecar，不要塞进 JSON 字段**：`asset.copyToLibrary('.bin', srcPath)` + payload 里 `_native: '.bin'`，运行时类重写 `_nativeAsset` getter/setter 接收 `ArrayBuffer`。把二进制编码进 JSON 字符串字段是错的。

### 3.5 资源右键菜单

声明和导出的函数名要对上（`{ "methods": "<文件>", "assetMenu": "<导出名>" }`，见 §3.1）：

```ts
// package.json: "assets": { "menu": { "methods": "./dist/assets-menu.js", "assetMenu": "assetMenu" } }
export function assetMenu(assetInfo: { uuid: string; url: string; isDirectory?: boolean }) {
    if (!canHandle(assetInfo)) return [];   // 返回空数组 = 不出菜单项
    return [{
        label: '烘焙…',
        async click() {
            await Editor.Message.request('my-extension', 'open-panel', assetInfo.url);
        },
    }];
}
```

### 3.6 拖到 Inspector（`inspector.drop`）

贡献点的**声明**形态照 `builtin/animator/package.json`，形态与易错点见 §3.1。

⚠️ 但**回调实参形态在 3.8.x 没有公开签名**（内置 animator 走的是同一条通道，实现编译在 `app.asar` 里读不到）。历史上见过 `(dropItem, dumps, uuidList)` 这种三参形态，但**不要当成事实**。

稳妥写法是不解构，把所有实参广度优先扫一遍找 uuid：

```ts
export const methods = {
    async inspectorDrop(...args: unknown[]) {
        const uuids = collectUuids(args);            // BFS 扫 {uuid} / {value} 字段
        const nodeUuid = uuids.find(isNodeUuid)
            ?? (await Editor.Message.request('scene', 'query-node-tree'), currentSelection());
        // ...
    },
};
```

真实形态在具体工程里验一次之后再考虑简化，**防御代码不要凭猜删掉**。

### 3.7 往节点上挂组件、填属性

```ts
await Editor.Message.request('scene', 'create-component', { uuid: nodeUuid, component: 'cc.Sprite' });
await Editor.Message.request('scene', 'set-property', {
    uuid: nodeUuid,
    path: '__comps__.0.spriteFrame',
    dump: { type: 'cc.SpriteFrame', value: { uuid: frameUuid } },
});
```

**`set-property` 对不存在的属性会静默成功。** 这是 Cocos 插件开发里最常见的一类静默失败：日志一片正常，属性没赋上。所以每次赋值后**必须回读**：

```ts
const dump = await Editor.Message.request('scene', 'query-node', nodeUuid);
// 从 dump.__comps__ 里核对刚写的值，不一致就换下一个候选属性名
```

属性名候选（`spriteFrame` / `_spriteFrame` 之类）要一个个试并回读，全都失败才退到场景脚本，并且要在结果里明确告诉用户「这个属性走了备路、不在 undo 栈里」。

**但候选只放引擎里确实存在的名字。** 「反正多试一个不亏」是错的——`set-property` 对不存在的属性静默成功，多一个假名字就是多一个假阳性的机会。花两分钟去引擎源码里确认，别猜。实测在一份代码里抓到三个凭猜写的备候选（3.8 / 4.0 都不存在）：

| 猜的 | 引擎里的实情 |
|---|---|
| `cc.Sprite._trim` | `trim` 是 getter，背后的字段叫 **`_isTrimmedMode`** |
| `cc.Animation._playOnLoad` | `playOnLoad` 是普通公开字段，**没有**下划线版本 |
| `sp.Skeleton._loop` | `loop` 是普通公开字段，**没有**下划线版本 |

规律：**getter/setter 才有下划线版本的私有字段，`@serializable public xxx` 这种平铺公开字段没有。** 先看清是哪一种再决定要不要写备候选。

### 3.8 场景脚本（备路 / 建节点）

```ts
// scene.ts —— 跑在场景进程，能拿到 cce.* 和引擎的 cc
export function createNode(name: string) {
    // 必须走编辑器的门面，不要用裸 node.addComponent：
    // 裸调用撞不上 undo/redo 和场景保存（改完看着对，一保存就没了）
    const node = cce.Node.createNode ? cce.Node.createNode({ name }) : null;
    // cce.* 是未公开内部 API，大版本升级要重新验证路径。
    // 多路径探测 + try/catch 降级是必须的，不是过度设计。
    return node?.uuid ?? null;
}
```

### 3.9 存一份配置：三个宿主，先按「存在哪」选

三条路都能给用户一个 checkbox，区别只在**值存到哪个文件、什么时候能读到**。先想清楚这个，再选 UI：

| 宿主 | 值落在哪 | 什么时候读得到 | 适合 |
|---|---|---|---|
| `contributions.preferences` | 默认全局（`~/.CocosCreator/`），偏好设置面板每项有「将配置存储到项目」的开关 | 编辑期任意时刻 | 个人习惯（日志开关、默认路径） |
| `contributions.profile` + `Editor.Profile` | `'project'` → `<工程>/settings/v2/packages/<name>.json`（**随工程提交**）；`'local'` → 工程内不提交；`'global'` → 跨工程 | 编辑期任意时刻 | 团队要共享、要可复现的决策（包体、输出格式） |
| `IBuildPluginConfig.options` | 构建配置，随构建平台参数走 | **只有构建时**，经 `options.packages['<扩展名>']` 进钩子 | 只影响本次产物的参数 |

- 声明式 checkbox 的写法：`contributions.preferences.properties` 里是**扁平的** `{ ui, attributes, items }`（照 `builtin/asset-db` / `builtin/builder` 的 `package.json`）。**这和 `builtin/preferences/@types/index.d.ts` 里写的 `render: { ui }` 不一致 —— 以 builtin 的实际写法为准**，类型声明那份是旧的。
- `contributions.profile` 的形态是 `{ editor: {...}, project: { <key>: { default, label?, message? } } }`（`builtin/engine/package.json` 同时用了两半）。`message` 是值变化时编辑器广播的那条消息名，用它做「改了就立刻生效」。
- 读写就是 `Editor.Profile.getProject(pkg, key, 'project')` / `setProject(pkg, key, value, 'project')`，以及 `getConfig` / `setConfig(…, 'local' | 'global')`。`getProject` 在没存过时**返回 `contributions.profile.project.<key>.default`**，所以不用自己写「首次运行」分支 —— 但仍然要把读到的值过一遍归一化函数（老配置缺 key、有人手改成字符串）。
- **随工程存 ⇒ 扩展必须装在 `<工程>/extensions/` 下。** 装在全局目录时一份扩展目录被所有工程共用，而配置是每个工程一份，任何「按配置改扩展自己目录里的文件」的设计都会在切工程时来回打架。这种扩展应当在 `load()` 里用 `path.relative(Editor.Project.path, __dirname)` 检测自己在不在工程内，不在就打一条 warning。

**时序结论（决定了架构，不只是选型）**：构建面板的 `options` **管不了 asset-db 里有什么**。构建开始时脚本清单已经定了，最早的钩子 `onBeforeBuild` 也已经太晚。所以任何「哪些脚本进包」的开关都**必须是编辑期配置**，放构建面板会做出「勾了但这次构建没生效」这种最坏的交互。

## 4. 资源侧的 3.8.8 实测事实

### 4.1 UUID 只能由 asset-db 分配

这条决定了架构分工。任何引用别的资源的资源（`.anim` 引用 SpriteFrame、prefab 引用材质）里存的都是 **UUID**，而 UUID 只能由 asset-db 在导入时分配。所以：

> **纯 CLI 能产出的只有「无引用」资源**（spine 三件套、图片、纯文本）。凡是要引用别的资源的，序列化必须发生在编辑器侧。

正确的分工是：CLI 做全部**像素 / 计算**工作并把引用位置留空，扩展导入之后查 UUID 填上再写盘。

子资源 UUID 形如 `<父资源uuid>@<hash>`（实测 hash 是 5 位 hex，**别假设长度**，整串原样用）。查法：

```js
const info = await Editor.Message.request('asset-db', 'query-asset-info', 'db://assets/x.plist');
// info.importer  → 实际认领它的 handler 名
// info.subAssets → { [key]: { uuid, name, type } }
```

**子资源的 key 实测是一串 5 位 hex hash，不是名字**（3.8.8 + `sprite-atlas` 导入 12 帧图集）：

| | 值 |
|---|---|
| key | `adb25` —— hash，和资源名无关 |
| `name` | `char_0000` —— 真名，而且**后缀被去掉了**（plist 里写的是 `char_0000.png`） |
| `uuid` | `<父uuid>@adb25` —— key 就是 `@` 后那一段 |

所以**按 `name` 查，不要按 key 查**。顺带一条说明这个 hash 是什么的观测：13 个不同的 png，子资源 key 全都是同一对 `6c48a` / `f9941`——**hash 是子资源名的哈希，不掺父资源 UUID**。

写代码时仍然把 key、去后缀的 key、`name`、去后缀的 `name` 四种都建进索引：上面这些值读的是磁盘上的 `.meta`，而 `query-asset-info` 返回的 `subAssets` 是不是同一个形状要另外验，多建三个索引的成本是零。

### 4.2 同后缀的导入器争抢

`.json` 被 **4 个** handler 争（spine-data / dragonbones / dragonbones-atlas / json），`.plist` 被 **2 个**争（sprite-atlas / particle）。区分全靠 `validate()` 看内容：粒子 plist 顶层是 `emitterType` / `angle`，图集 plist 顶层是 `frames` + `metadata`。

tie-break 规则是「**最后注册的先试**」（handler manager 反向遍历注册数组），**但项目扩展和引擎内置扩展谁先注册没有保证**——不能假设「我是项目扩展所以我更晚所以我赢」。

排查顺序：

1. 看资源已生成的 `.meta` 里的 `"importer"` 字段——它直接告诉你现在是谁在管这个文件。**`.meta` 一旦存在，asset-db 会直接复用记录的 importer，不会重新走优先级判定**，所以怀疑注册冲突时先删 `.meta` 再触发导入。
2. `package.json` 的 `contributions.asset-db.asset-handler[].extnames` 和 asset-db 脚本里兜底注册用的 extnames 列表必须**两处同步**。只改一处（尤其只改脚本）会导致某条后缀实际从未被认领——这是实际犯过的错。
3. 两处都对但顺序还是不听话时，不要指望「重启换个顺序」。注册完之后把 `extname2registerInfo[ext]` 里自己的条目挪到数组末尾，并 monkey-patch 一次 `mgr.register`，让**之后任何人**注册同一后缀时都被重新排到自己后面。这是确定性修复。
4. 用 `<工程>/temp/asset-db/log/*.log` 里的 `run package(...) handler(enable)`、`lazy register asset handler <name>`、`Importer exec failed` 还原真实时间线，比猜靠谱得多。

### 4.3 asset-db **不监听**文件系统

3.8.8 的 asset-db 只在**编辑器窗口重新获得焦点**时跑 `refresh-all-database`。用外部工具往 `assets/` 里写文件，编辑器不会自己发现（观测证据：`temp/asset-db/log/*.log` 里只有焦点时刻的 refresh 记录）。

所以扩展写完文件必须**显式**通知：

```ts
await Editor.Message.request('asset-db', 'refresh-asset', url);   // 或 import-asset
```

并且**导入有严格顺序**：被引用的先导。`.plist` 引用 `.png`、`.anim` 引用 `.plist` 的子资源。顺序错了 asset-db 会先建出一堆坏引用，之后即使文件到位也不会自动修。

顺带一条：**「这个目录建了吗」不要问 asset-db，问磁盘。** asset-db 既不监听文件系统、又对某些 url 形态直接抛异常（下一节），而 `db://assets/a/b` 到磁盘路径的映射是确定的：

```ts
const fsPath = path.join(Editor.Project.path, 'assets', url.replace(/^db:\/\/assets\/?/, ''));
if (!fs.existsSync(fsPath)) await Editor.Message.request('asset-db', 'create-asset', url, null);   // null → 建文件夹
```

**`import-asset(source, target)` 的 `target` 是新资源自己的完整 url，不是「放进哪个目录」**：

```ts
// 对：
await Editor.Message.request('asset-db', 'import-asset', '/tmp/out/char.png',
    'db://assets/art/char_baked/char.png', { overwrite: true });
// 错：传目录 → Cannot overwrite directory '<target>' with non-directory '<source>'
```

依据是 `builtin/asset-db/@types/message.d.ts` 里 `import-asset` 和 `copy-asset` 的 `params` **逐字段相同**，而 `copy-asset` 的第二参显然是目标路径本身。这条错误信息**难得地把两个参数分别打了出来**（`directory '<target>'` / `non-directory '<source>'`），照着读就能定位——容易读错的是把它当成「目录被占用、要先删目录」然后去改建目录的代码，而那边是对的。`overwrite: true` 也要开，否则第二次导入会被 asset-db 改名成 `char-001`，场景里的引用还指着旧资源（表现为「重导了但画面没变」）。

顺带一条**关于类型注释的**：我们自己 `.d.ts` 里那句「target 可以是目录」是凭直觉写的，正好写反，于是 IDE 一路给出错误的提示。**凭印象写的类型注释会把错误固化下来，比没有注释更糟**——照抄 asar 里的明文 `.d.ts`。

### 4.3.1 `query-asset-info('db://assets')` 会抛 `parameter error`

**数据库根不是一个可查询的资源。** 而且 asset-db 区分得很清楚：

| 情况 | 行为 |
|---|---|
| 资源不存在 | **返回 `null`** |
| 入参形态不接受（根、空串、`undefined`） | **抛 `Error: parameter error`** |

所以看到抛异常就说明是**入参问题，不是「找不到」**——这个判据能直接省掉一轮排查。

踩法都一样：写一个「沿 db url 逐级往上找祖先」的循环，条件写成 `url.startsWith('db://assets')`，最后一步正好落在根上。加个**末尾斜杠**（`'db://assets/'`）就没了。

**麻烦的不是这个错，是它的报错完全不可定位**：

```
Error: parameter error
  at AssetManager.queryAssetInfo (…/builtin/asset-db/dist/worker/manager/asset-manager.ccc:1:2181)
  (Target: asset-db, Message: query-asset-info)
  (Target: cocos-anim-baker, Message: bake)   ← 只告诉你是哪个「消息」，不是哪个调用
```

不说实参、不说调用点，`.ccc` 是加密的所以行号也没用。一个流程里 `query-asset-info` 调七八次，长得一模一样。**所以先包一层带标签的封装，再去猜原因**：

```ts
async function assetDb<T = any>(message: string, ...args: unknown[]): Promise<T> {
    try {
        return await (Editor.Message.request as any)('asset-db', message, ...args);
    } catch (e) {
        throw new Error(`asset-db 的 ${message}(${args.map((a) => JSON.stringify(a)).join(', ')}) 失败：${(e as Error).message}`);
    }
}
```

加标签是确定有收益的，猜原因不是。同理，凡是 asset-db 返回的 `uuid` 要拿去做下一次查询的，**当场检查它有没有值**——`query-asset-info(undefined)` 换来的又是一句一模一样的 `parameter error`，而这次错误发生在几十行之外。

还有一个陷阱：`void somethingAsync().then(...)` 里抛出来的是**无主 rejection**，控制台里只剩一句没有任何上下文的 `parameter error`，连是哪个消息触发的都看不出来。异步回调里调 asset-db 记得自己 catch。

### 4.3.2 「写状态回资源」和「监听资源变动」会互相触发

只要扩展同时做这两件事，**写操作自己就是一个事件源**：

```
处理成功 → 把参数写回【源资源自己的】.meta（save-asset-meta）
        → asset-db 广播 asset-change(同一个 uuid)
        → 你的 onAssetChange 命中刚写的记录 → 又处理一次 → 又写 → …
```

症状是几行日志每隔一个防抖周期重复一次、永不停止，而且**每轮都是一次完整的真实工作**（子进程、重新导入一批资源），不只是刷屏。防「产物目录变动」的那种 url 前缀判断**挡不住这条**——这次变的是源资源自己的 meta，uuid 就是源资源的。

修法是给自己的写打个短时标记，并且**消费掉**（别让它长期有效，否则会吞掉用户的真实编辑）：

```ts
const SELF_WRITE_GRACE_MS = 3000;              // 要宽于你的防抖周期
const selfWrites = new Map<string, number>();

export function consumeSelfWrite(uuid: string): boolean {
    const t = selfWrites.get(uuid);
    if (t === undefined) return false;
    selfWrites.delete(uuid);
    return Date.now() - t < SELF_WRITE_GRACE_MS;
}
// save-asset-meta 之前：selfWrites.set(uuid, Date.now())
// onAssetChange 第一行：if (consumeSelfWrite(uuid)) return;
```

**所以「不会自激循环」这个检查项要按方向拆开验**，写成一条必然漏——我就是先修好了产物方向、以为完事，第一次成功烘焙立刻撞上写-meta 方向。它还有个额外的误导性：循环的第一轮日志里常常还留着别的报错，于是很容易以为循环是**失败重试**造成的，而它其实是**成功**路径触发的。

### 4.3.3 外部工具拷入资源后「预览找不到」：三层状态各自为政

外部拷文件进 `assets/`（编辑器失焦期），焦点时刻 `refresh-all-database` 会补账导入，但**导入成功 ≠ 预览能加载**。三层状态，任何一层滞后都表现成同一个报错（`Bundle resources doesn't contain <path>`），排查要逐层验：

| 层 | 验法（全部 HTTP/文件，不需要碰编辑器 UI） | 滞后时的症状 |
|---|---|---|
| ① asset-db 登记 | `library/.assets-info.json` 的 **`map` 键**：`{map:{'resources\\x\\y.json':{time,uuid}}}`。注意值是 `time/uuid`——按 `value.source` 之类字段查必得 0 条，形状要看清 | map 没条目 = 导入没跑成 |
| ② library 产物 | `library/<uuid前2>/<uuid>.json`（payload）+ 源文件旁 `.meta` | 有 meta 没 payload 或反之 = 半途 |
| ③ 预览的 bundle config | `GET http://localhost:7456/assets/resources/config.json`，看 `paths` 里有没有目标路径；项目脚本新旧看 `GET /scripting/x/import-map.json` 里模块映射指向的 chunk 内容 | db 有而 config 没有 = builder 的「查询 Asset Bundle」缓存没失效 |

关键机制：**builder 的 bundle 查询缓存只在收到 `asset-db:asset-add/change/delete` 广播时失效**，而失焦补账式的 refresh-all-database 路径不发这个广播。让缓存失效最简单的办法是制造一次**真实**的资源增删（新文件进场 → 焦点循环），而不是只 touch mtime——mtime 变化常常连重新导入都不触发。
**改工程 TS 后预览吃的可能是旧代码**（2026-09 实证）：工程 TS 只在编辑器窗口**获得焦点**时才重编译。外部工具改完 `assets/scripts/**.ts` 失焦放着，预览页永远跑旧包——URL 参数都在 `location.search`、行为却纹丝不动，极像「参数没生效」的 bug，其实是代码没进包；聚焦后编译还可耗时 >14s。确认编译落地：日志出现 `%cImport%c: …\.ts`，或轮询 `temp/programming/packer-driver/targets/preview/chunks/` 的 chunk 出现新函数名；**判定页面吃的是新包还是旧包，最快探针是 CDP `Runtime.evaluate` 读页面上版本敏感的 Label 文本**（编译失败时 console 什么都不打，日志探针失效）。需要聚焦用 `(New-Object -ComObject WScript.Shell).AppActivate(<编辑器PID>)`，自动化跑批前务必先确认编译落地。改动同时跨 `assets/scripts` 与扩展 `runtime/`（如 spine worker 交付）时，把两边文件全部 touch 后再聚焦一次搞定。

**watcher 会整个卡死**（长会话实测）：症状三连——新文件不生成 `.meta`、touch 源文件后日志无任何 `%cImport%` 行、失焦/聚焦循环后没有新的 `refresh-all-database` 行。到这一步焦点循环救不回来，**重启编辑器是唯一恢复手段**（`taskkill /PID <主进程>` 再带 `--project` 拉起，1.5 分钟全恢复；预览服务器随之重启，collector 等外部进程不受影响）。

### 4.3.4 预览里 `resources.load(path, 扩展自定义资产类)` 必失败：ctor 急切绑定

引擎 `config.ts` 的 `_initPath` 在 **bundle 初始化时**就急切执行 `assetInfo.ctor = js.getClassById(type)`。预览页的时序是 game.init 预加载 bundles → 场景脚本（含扩展 runtime、`@ccclass('xx.yyData')` 注册）才执行，于是**扩展自定义资产类型的条目 ctor 永远绑成 undefined**。之后 `resources.load(path, MyClass)` 里的类型过滤 `isChildClassOf(undefined, MyClass)` 恒 false，报出来的是**误导性的** `Bundle resources doesn't contain <path>`——资产明明就在 bundle 里。

判别（CDP `Runtime.evaluate` 一条就够）：`cc.assetManager.bundles.get('resources').config.assetInfos.get('<uuid>')` —— `.path` 有值而 `.ctor === undefined` 即此坑。规避：**load 不传 type**（`resources.load(path, null, cb)`），路径查找不做 ctor 过滤，反序列化按载荷 `__type__` 实例化，那时类已注册。构建产物没有这个时序问题（插件脚本先于 game.init 加载），所以该写法在预览/构建两端都成立。

### 4.4 生成引擎能直接播的资源，路只有这么几条

| 想要 | 唯一 / 首选路径 | 注意 |
|---|---|---|
| `cc.SpriteAtlas` + 逐帧 SpriteFrame 子资源 | 写 TexturePacker **`.plist`** | 3.8.8 的 55 个 asset-handler 里只有两个能产出 SpriteAtlas，另一个是 `auto-atlas` 的 `.pac`，它在**构建期重新打包**，不给你稳定的子资源 |
| `cc.AnimationClip` | 手写 `.anim` | 关键帧引用 UUID，必须在编辑器侧组装；**序列化格式见 cocos-asset-formats skill** |
| Spine 动画 | `skeleton.json` + `.atlas` + `.png` | 三个普通文件，**无 UUID 概念**，纯 CLI 就能产出，拖进工程即可。格式细节（双版本 atlas / vendored spine-cpp 解析器规则）见 **cocos-asset-formats** skill |

## 5. 调试与验证

- **静默失败优先怀疑五个地方**（都不报错，或只报一行指向错方向的 warn）：`set-property` 写了不存在的属性（§3.7）、`.meta` 里 importer 被复用（§4.2）、面板主文件用了 `export default`（§3.3）、`package.json` 缺 `package_version`（§3.1，症状是扩展管理器里看不见但其实已加载）、**`panels` 被写进了 `contributions` 里**（§3.1，症状是扩展一切正常、只有开面板报 `Panel(...) is not defined`）。根源对策：贡献点**照 §3.1 的 schema 和内置扩展抄，不要凭记忆写**——凭记忆写的那份骨架三处全错（`panels` 位置、`assets.menu` 形态、`inspector.drop` 形态），症状全都指向别处（查脚本、查 `module.exports`、查路径），错的地方其实在 JSON 的缩进层级里。
- **Cocos 报错的位置和真正的错误位置几乎从不重合**：`Panel(...) is not defined` 指向面板脚本而错在 `package.json`；`parameter error` 指向 asset-db 内部而错在你某个循环的边界条件（§4.3.1）；`.effect` 下载失败 / 渲染管线为 null / 材质属性非法这一整串场景报错，错在**引擎目录的 `package.json`** 被换掉了（§0）。栈还都在 `.ccc` 里看不了。**所以顺序是「先给每个跨进程调用加上带实参的标签，再开始分析」**——加标签的收益是确定的，分析的收益不是。
- **不是所有失败都安静**：也有反过来的——**扩展在后台无限重复干活**。如果它既写资源又监听资源变动，先怀疑自激循环（§4.3.2），判据是日志里同样几行按固定周期重复。这个不看日志基本发现不了，因为 UI 上什么都不弹。
- **「卡住不动、既不成功也不报错」几乎一定是 wasm 路径错了**：`pal/wasm` 的失败路径是 noop，给你一个永不 settle 的 Promise（§8.3）。别找报错，去查文件在不在。
- **自动化开预览 URL**：CDP `Page.navigate` 到与当前页**只差 hash** 的 URL 是同文档导航，而 preview **不响应 hashchange 切场景** → 页面原地跑旧场景、新场景的采样永远等不到。自动化逐场景打开时先跳 `about:blank` 再导航目标 URL。
- **「扩展装了没生效」先去看 `<工程>/temp/logs/project.log`**，别在扩展管理器里瞪着看。加载成功的话你自己 `load()` 里打的日志会出现在里面，`[Package] <name>: …` 和 `Panel(...) is not defined` 那几行 warn 也在，一眼就能分清「没加载」「加载了但没显示」「加载了但某个贡献点没注册」。
- `<工程>/temp/asset-db/log/*.log` 是导入问题的第一现场。
- 引擎侧的行为（资源能不能播、属性名对不对）**没法靠 mock 单测**——mock 出来的断言等于把实现抄一遍。把纯计算部分抽成零依赖模块去单测，剩下的消息往返写一份**带症状→成因对照表**的人工验证清单，比假装有覆盖率诚实得多。
- **能力探测必须按真实 API 形态写**（2026-09 实证）：探测浏览器「fetch 文本 → Blob URL → Worker」链时写过 `typeof globalThis.createObjectURL === 'function'`——它是 **`URL` 的静态方法**、不是全局函数，恒 `undefined` → worker 档静默回退主线程，A/B 的 worker 组悄悄全是基线（「小游戏里验过、浏览器其实从未真跑过 worker」）。防线：探测函数在真实页面里手验一次，API 名对着 MDN 抄，不凭印象。
- **降级链任何一级静默 return null 都必须留 warn**（静默降级是 A/B 的天敌）：worker 档取脚本失败曾静默返回 null——对照组与实验组悄悄变成同一实现，「无差异」的假数据比崩溃更难发现。会话级缓存失败结果的场合，warn 天然只打一次，不必担心刷屏。
- 格式类输出（`.atlas` / `.plist` / `.anim` / `skeleton.json`）用 **golden 文件逐字节比对**。这几种格式写错的症状都是「能导入但不显示」，没有 golden 守着基本查不出来。**插件自带解码器（wasm / JSB）时还有更真的一招**：在 Node 里把生成的数据喂给真实解码器端到端跑一遍（见 §8 清单第 7 条），golden 只证明「和上次一样」，真实解码器才证明「真的能播」。

## 6. 集成到工程 & 发布前检查

- 插件文件夹（含 `package.json` 的那一层）整个放进目标工程 `extensions/` 即可被识别；`asset-db.mount` 的 `runtime/` 自动只读挂载为项目脚本。
- 扩展目录里的 `node_modules` 要**在目标工程里装**（`cd extensions/<name> && npm install`），别指望能借到工程或编辑器的依赖。
- 编译产物尽量**自包含**：让扩展的 `tsconfig.json` 把共享的源码目录一起编译进 `dist`（`rootDir` 指到共同父目录），比引 workspace 软链靠得住。
- TS 运行时代码 `import from 'cc'` **不需要本地 tsconfig**——类型由挂载它的工程环境提供。编辑器侧脚本是 CommonJS（`require`、全局 `Editor` / `cce`）。**两条线不要混。**
- 涉及原生：目标引擎必须导出插件用到的引擎能力（例如 `cc.UIMesh` + 公开的 `cc.wasm`）——集成前先确认目标是不是这样的自定义引擎，不是的话原生功能装了也用不了。可执行判据：`typeof (cc as any).wasm?.instantiateWasm === 'function'`，或者直接在引擎源码里看 `exports/webassembly.ts` 在不在。**判断不满足时优雅降级（少一个格式 / 少一个特性 + 一条 warning）比装不上要好** —— 插件能在任何 3.8.x 上装得进去，用户才愿意试。另外编辑器用哪个引擎是**全局配置**（§0 的 engine.json），所以「插件在这个工程里能不能跑」很可能取决于用户的全局引擎配置而不是工程本身——集成后先 `head -20 <工程>/temp/logs/project.log` 确认 `Load engine in` 指向的是不是要求的那个引擎。
- 工程脚本消费挂载运行时（比如给插件写测试场景），导入路径是 **`db://<package.json 的 name>/<runtime 内相对路径去后缀>`** —— db 名取包名，不是扩展目录名（两者一致是惯例不是保证）。实测：`import { Spine, loadSpineRuntime } from 'db://spine-runtime/index'`。
- 插件新装 / 更新后，工程脚本要 import 它的挂载运行时**必须先重启编辑器**（扩展加载和 `asset-db.mount` 都在启动期）；`temp/logs/project.log` 里能看到扩展加载日志确认。
- 要在 CLI 里对「引用了挂载运行时的工程脚本」跑 tsc：paths 映射兜住 db:// 导入（`"db://<pkg>/*": ["../extensions/<pkg>/runtime/*"]`）；引擎声明用自定义引擎自己的 `<引擎>/bin/.declarations/cc.d.ts`（没有就 `npm run build:declaration` 生成，别引用工程 `temp/declarations/cc.d.ts` 那个一行桩）。**声明生成器有缺口**：实测 `_decorator.editable/override` 被漏掉（构建日志有 `Don't know how to handle element enable of interface`），加一个 `declare module 'cc'` 的模块增强 d.ts 补齐，别改引擎。完整管线样板：`D:\projects\performance\tools\tsconfig.check.json` + `cc-shim.d.ts`。
- **改了挂载运行时 / worker bundle 后的免 GUI 重建**（3.8.8 实测）：`CocosCreator.exe --project <工程> --build "logDest=<日志路径>"` —— `--build` **不带构建参数 = 直接用「构建发布面板」保存任务的完整 options**（含 `overwriteProjectSettings` 的 includeModules / macroConfig 覆盖；平台 packages 配置如微信 appid 也会 merge 进来），不需要手工拼 configPath JSON。退出码：36 成功、32 参数不合法、34 构建失败。同工程编辑器 GUI 开着时实测**不撞项目锁**（担心挂死可配 360s 日志监控，无日志即杀）。可比性验证：日志里 `Use cache, md5String: ... includeModules=[...]` 命中引擎构建缓存 = 引擎裁剪与上次构建逐字节同源。⚠ 构建会**重置 `build/` 里的 `project.config.json`**（urlCheck 等手工修改被模板覆盖，出预览码前要重改）。
- 交付「可裁剪的可选功能」时，README 里写的「不需要就不进包」必须是真的 —— 见 §7.1，Cocos 不做 tree-shaking，这句话在多数人的直觉实现下是假的。

## 7. 参数暴露与包体

把重活挪到编辑期，换掉的是**运行时成本，不是存储成本**。小游戏 500KB 首包之类的约束一点没变。

所以生成资源的插件**必须在界面上摆一个实时的体积 / 容量估算**，否则用户会稳定产出几 MB 的资源，而且要到上线前才发现。

配套的一条：**校验函数只放硬错误。** 决定「烘焙按钮能不能点」的那个函数里混进「能跑但不划算」的建议，会把用户拦在一个其实完全合法的参数上。建议另开一条通道显示、不禁用按钮。

### 7.1 Cocos 3.8 不做未引用脚本的 tree-shaking

**脚本目录下的每个脚本都是 bundle 入口。** 场景和 prefab 按 UUID 引用组件，构建器无法证明哪个脚本是死的，所以它一个都不敢删。

测量方法（比读构建器源码快得多）：数产物 `assets/<bundle>/index.js` 里 `_RF.push` 的个数，和源码 `.ts` 的个数比。实测 13 个 `.ts` → **正好 13 个** `_RF.push`，其中包括**没有任何文件 import 的 `index.ts`**、场景引用 0 次的一个 demo 组件，以及编译后本该什么都不剩的**纯 `interface` 文件**。

由此得到一条对功能开关设计的硬约束：

- **「注释掉一行 import / 改一个配置常量」式的裁剪在 Cocos 里是假的。** 它省的是运行时开销（不解析、不加载 wasm、不占内存），**包体一个字节都不少**。给用户一个这样的开关，再在 README 里写「不需要的格式会被裁掉」，是在写一句不成立的话 —— 这个坑我踩过，而且是在自己写的 README 里踩的。
- 要真的减包体，**文件必须在构建时物理不在挂载目录里**。可行做法：把整组文件（**连同 `.meta`**，UUID 才不变）在 `runtime/` 和一个同级 `trimmed/` 之间**移动**，按编辑期配置（§3.9）在扩展 `load()` 时幂等对齐一次。移动而非「删除 + 从 pristine 拷回」：只有一份拷贝，不会漂移，用户裁剪后提交工程文件仍在他仓库里。
- 组间的引用集中到一两个**生成文件**里（一个注册表、一个 barrel `index.ts`），文件头写明「生成物，手改会被覆盖」。仓库里提交的是默认全开态，所以默认情况下重新生成是零 diff。
- 生成文件的「内容没变就不写」判断要**忽略行尾**，并保持文件原有的行尾。`core.autocrlf` 打开的 Windows 检出会把这些文件变成 CRLF，而生成器吐 LF —— 不处理的话每次加载都判定「不一致」并全部重写，churn 掉 mtime 引发无意义的重新导入，正好是这个优化想避免的事。
- 移完文件要按 §4.3 显式 `refresh-asset`（asset-db 不监听文件系统）。**只读挂载能不能接受 `refresh-asset` 未验证**，所以正确性不要建立在它成功上：给一句「若资源管理器或构建结果没跟着更新，重启一次编辑器」的兜底提示。
- 一个格式 / 功能往往有**多条互不相干的入包路径**，每条都要单独关。实例：同一个 WebP 支持要关三处 —— 挂载的 TS（文件移走）、`.wasm`（构建钩子跳过复制）、原生 C++（`cc_plugin.json` 的 `platforms` 改成 `[]`，见 §8.1）。漏掉任何一条，「已裁剪」就是假的。**在构建日志里打一行当前生效的功能集合** —— 这通常是产物里到底含什么的唯一信号。

⚠️ 上面 13 → 13 那次是在工程 `assets/` 下的真实目录测的。**`asset-db.mount` 出来的 bundle 是否同样不 tree-shake 尚未单独验证**（没有理由认为不同，但没实测）。要依赖这个结论时先按上面的方法量一次。

## 8. 原生 C++ / wasm（只在真要跨语言共享同一份逻辑时读）

这一节的复杂度（命名空间隔离、AOT embind、JSB 手动绑定）只在**真的要让 native 和 web 跑同一份 C++ 逻辑**时才值得。纯 TS 组件 + 自定义资源导入，做到 §1–6 就完整可用。

核心结构：**一份 C++ facade 头文件**，JSB（`se::Object` / `se::Class`，`SE_BIND_FUNC`）和 embind（`EMSCRIPTEN_BINDINGS`）各写一层薄绑定，TS 侧按 `NATIVE` 宏**严格二选一**（不是「没有 JSB 就 fallback 到 wasm」）。

### 8.1 `cc_plugin.json` → CMake 全自动接入链路

文件布局与命名约定（出处 `native/cmake/scripts/plugins_parser.js`）：

- `package.json` → `contributions.native.plugins.<任意key>.cc_plugin.json` → `./native/cc_plugin.json`
- 字段：`name / version / engine-version / author / description / modules / platforms`
- `native/<平台>/<target>-config.cmake` —— **文件名必须是 `<target>-config.cmake`**，因为生成的是 `find_package(<target> REQUIRED NAMES "<target>")`
- 平台目录名是固定的（`add_search_path_suffix`，`plugins_parser.js:173`）：`android/`（先找 `android/${ANDROID_ABI}`）、`windows/`（先找 `windows/x86_64`）、`ios/`、`mac/`（先找 `mac/${CMAKE_SYSTEM_PROCESSOR}`）；`iphonesimulator` 回落到 `ios/`
- **linux / ohos / harmonyos 没有 suffix 映射**（else 分支只 warn 一句 `Don't knowm suffix`）→ 搜索路径为空 → `find_package(... REQUIRED)` 失败。**原生插件机制实际只覆盖 android / windows / ios / mac 四个平台。**
- `platforms` 的值是**精确字符串匹配** CMake 平台名（`indexOf(PLATFORM_NAME_FROM_CMAKE) < 0` → 跳过），而目录 suffix 那边是正则匹配 —— **两套规则不一样**，`platforms` 里写 `win` 不会生效。反过来用：`platforms: []` 是一个干净的「对所有平台都不接入」开关（比删清单靠错误被吞掉可靠，见下条），适合做 §7.1 那种功能裁剪。

**验证点**：构建后看工程 `native/engine/<平台>/Pre-AutoLoadPlugins.cmake` 里有没有你的 target。

**为什么只能靠这个验证点**：`cc_plugin.json` 的必填字段校验是**死代码**。`plugins_parser.js:141` 的 `field_required` 写的是 `Object.hasOwnProperty(obj, field_name)` —— 在 `Object` 构造器上取 `hasOwnProperty`、把 `obj` 当 key（第二个实参被忽略），键是 `"[object Object]"`，恒 false，于是**恒返回 true，校验永不失败**。漏字段拿不到「field 'x' is not set」，而是后面某处炸掉、被 `for` 循环外层的 try/catch 吞成一行 `console.error`，**插件被整个跳过而构建照常成功**。典型的「症状指向错地方」——你会以为是 CMake 或 C++ 写错了。

### 8.2 `CC_PLUGIN_ENTRY`：JSB 自动注册，不用改工程的 `Game.cpp`

完整链路：`cc_plugin.json` → `Pre-AutoLoadPlugins.cmake` → `find_package` → `<target>-config.cmake` → 你的 `.cmake` 建 STATIC 库 → `CC_PLUGIN_ENTRY` → `templates/cmake/common.cmake:14` 的 `cc_plugin_entry()` 生成 `plugin_registry_autogen.cpp` → `cc_load_all_plugins()` ← `native/cocos/application/BaseGame.cpp:47` 调。**装上插件出普通原生包即自动编译、自动注册，零手工步骤。**

「原生绑定要改工程的 `Game.cpp`」是个很自然但**错**的推断 —— `jsb_module_register.cpp` 里所有 `addRegisterCallback` 都是硬编码的、看不到插件钩子，很容易就此下结论。钩子在 `cocos/plugins/Plugins.h`，不在 bindings 那边。

三个能让你卡住的细节：

- **`CC_PLUGIN_ENTRY(name, fn)` 的 `name` 必须逐字等于 `modules[].target`**，不是 `cc_plugin.json` 的 `name` 字段。因为 autogen 拿 `CC_REGISTERED_PLUGINS`（存的是 target 名）拼 `cc_load_plugin_<target>()`（`common.cmake:28,35`）。
- 于是 **target 名必须是合法 C 标识符**，只能 `_` 不能 `-`。所以会看到 `name: "spine-runtime"` 而 `target: "spine_runtime"` 这种搭配。写错是链接期 undefined symbol `cc_load_plugin_xxx`。
- 库必须 `target_compile_definitions(<t> PUBLIC CC_PLUGIN_STATIC)`，否则 `CC_PLUGIN_ENTRY` 走 DLL 导出分支（`Plugins.h:36`）。
- **不要在静态初始化期调 `se::ScriptEngine::getInstance()`**，那时它是 nullptr。入口函数里只能 `addRegisterCallback(fn)` 排队，真正碰 `se::Object` 的代码在回调里跑。

### 8.3 wasm 在四种宿主下到底去哪找文件

**用 `cc.wasm`，不要自己写 loader** —— 自己写就是绕弯路的根源。自定义引擎 `exports/webassembly.ts` 把 `pal/wasm` 以 `cc.wasm = {instantiateWasm, fetchBuffer, fetchUrl, ensureWasmModuleReady}` 公开。

传进去的是**裸文件名**（`'foo.wasm'`），**不要带目录** —— 各平台自己加前缀：

| 宿主 | `pal/wasm` 实际去哪找 | 传什么 |
|---|---|---|
| BUILD（web） | `new URL(name, import.meta.url)` → 产物 `cocos-js/` 旁边 | `'foo.wasm'` |
| 小游戏 | `cocos-js/<name>`（小米 `src/cocos-js/`、淘宝分包 `__ccWasmAssetSubpkg__/`）交给 `CCWebAssembly.instantiate` | `'foo.wasm'` |
| EDITOR | node `fs` 读 `<engine>/native/external/<name>` | `'external:foo.wasm'` |
| PREVIEW | `fetch('/engine_external/?url=external:foo.wasm')` | `'external:foo.wasm'` |

（出处 `pal/wasm/wasm-web.ts` 的 `fetchBuffer`、`pal/wasm/wasm-minigame.ts` 的 `getPlatformBinaryUrl`。）

- **小游戏的 `CCWebAssembly.instantiate` 只吃路径、不吃字节** —— 这是 base64 内嵌 / `SINGLE_FILE` 方案在微信上跑不起来的根因，也是必须留一个独立 `.wasm` + `instantiateWasm` hook 的原因。
- 微信多线程 Worker（旧版约束 + 新版 V2 灰度的 OffscreenCanvas / SharedArrayBuffer / worker 内 wx API / 多 worker）的完整参考，拆到了独立的 **wechat-minigame-worker** skill。对 wasm 交付影响最大的几条：**worker 目录只能放 JS**（`.wasm` 必须放外面）、postMessage 是拷贝、旧版及未灰度环境**同时最多 1 个 worker**——「解码器搬进 worker」的设想先过那边的约束表。
- 于是交付要**两条**：`main.js` 的 `load()` 拷到 `(await Editor.Message.request('engine','query-engine-info')).native.path + '/external/'`（管 EDITOR + PREVIEW），`contributions.builder` 的 `onAfterBuild` 拷进产物 `cocos-js/`（管 web + 小游戏）。**编辑器场景视图不走预览 HTTP 服务器**，所以只做 HTTP 那条覆盖不到场景视图。
- ⚠️ **最值钱的一条：`pal/wasm` 的 `fetchBuffer` 失败路径是 `// noop`，既不 resolve 也不 reject。** 路径错了拿到的是一个**永不 settle 的 Promise** —— 没有任何报错，加载悄悄卡住，`await` 永远不返回。小游戏那边同样是 `.catch((e) => {})`。所以「wasm 加载没反应」要先按上面那张表查文件在不在，**不要去找报错，没有报错**。
- `cc.wasm` **没有 `.d.ts`**，TS 侧必须 `(cc as any).wasm`，否则目标工程 typecheck 挂。可执行的能力判据见 §6。
- Emscripten 侧的接入点是 module option `instantiateWasm(imports, successCallback)`（老式单回调形态）。**hook 里 throw / reject 传不出去**（glue 不接），失败必须 reject 一个自己持有的外层 Promise —— 引擎自己的 spine / physx adapter 也有同样的注意事项。
- ⚠️ **`WebAssembly.instantiate` 两种入参的 resolve 形态不同**（worker 侧给 glue 的 `instantiateWasm` 回调喂 instance 时踩过）：传**已编译 Module**（Module 可结构化克隆、**严禁进 transfer 列表**）resolve 的直接是 **instance**；传 **bytes** 重载 resolve 的是 `{module, instance}` 二元组。两态混用时拿到 undefined 字段、报错点却在几行之外。
- ⚠️ **wasm 内部 C++ assert 的死法是 `RuntimeError: Aborted(Assertion failed: …)`**（emscripten abort 而非 JS 异常；2026-09 坏 JSON 喂 spine-cpp Json.cpp:120 实证）：worker **不死**，后续消息照常处理；错误对象跨 realm `e instanceof Error` 为 **false**（好在 `message` 字段还在，兜底用 `String(e)`）。错误路径测试按这个形态断言。

### 8.4 `contributions.builder` 的形态

- 两个文件：`builder.js` 只有一行 `exports.configs = { '*': { hooks: './hooks' } }`；`hooks.js` 导出 `onAfterBuild(options, result)`。
- 产物目录要**同时试** `result.dest` 和 `result.paths.output`（两种形态都出现过）。
- 原生平台要跳过（走 JSB，不需要 `.wasm`）。
- **钩子里 throw 会弄挂整个构建** → 全包 try/catch，只 log。为一个可选 codec 的载荷弄挂整个构建是完全不成比例的后果。
- 钩子跑在 **builder 的 worker 进程**里，那里**拿不到 `Editor.*`**。要读编辑期配置（§3.9）就派生一份纯 JSON 到扩展目录，钩子用 `fs` 读；文件缺失 / 解析失败一律当「全开」，这样没打开过面板的用户拿到的就是原本的行为。

### 8.5 Emscripten 链接选项里几个不显然但必需的

- **`-mno-reference-types -mno-bulk-memory`**：微信的 wasm 校验器只认 MVP 的 `call_indirect` 编码，带上新特性直接拒。
- **`-s DYNAMIC_EXECUTION=0`**：小游戏禁 `eval` / `new Function`。用 embind 时要配 **`EMBIND_AOT=1`**；接口只有几个函数的话别用 embind，纯 C ABI + `cwrap` 更省。
- `MODULARIZE=1 EXPORT_ES6=1 EXPORT_NAME=X`、`FILESYSTEM=0 NO_EXIT_RUNTIME=1 ALLOW_MEMORY_GROWTH=1`。
- glue 是 ESM，但**挂载进工程的 `.js` 被当 CJS** → 必须转换：干掉 `import.meta.url`、`new URL('x.wasm', import.meta.url).href` → `"x.wasm"`、中和环境探测守卫（改成 `if (false && …)`，小游戏 / 编辑器 scene worker 的环境和浏览器不同）、中和 `throw new Error('environment detection error')`、`export default` → `module.exports`。
- 提交 `.wasm` 进仓库时在 `.gitattributes` 里写明 `*.wasm binary`。git 靠 `\0asm` 魔数开头的 NUL 也能推断出二进制，但**别依赖内容嗅探**：`.wasm` 被 CRLF 转换弄坏的表现是上面那条「永不 settle 的 Promise」，即最难查的那种。

### 新增一个原生绑定函数的检查清单

1. embind 侧：写函数 + `function("name", &fn)`。
2. JSB 侧：写 `js_xxx_name(se::State&)` + `SE_BIND_FUNC` + `ns->defineFunction("name", _SE(js_xxx_name))`。
3. TS 侧的 binding 接口加同名签名——**两个后端必须暴露完全一致的调用形状**，调用方不按平台分支。**漏掉一边就会出现「某平台可用某平台不可用」的隐蔽 bug。**
4. 二进制 / `Uint8Array` 参数：embind 用 `emscripten::vecFromJSArray<uint8_t>(val)`；JSB 用 `se::Object::getTypedArrayData(&ptr, &len)`（零拷贝，不用手写内存搬运）。**不要**假设 embind 的 `std::string` 能安全传二进制——JS 字符串是 UTF-16，字节数组会被破坏。
5. 改完 wasm 侧必须重新编译并**同时替换 `.js` 和 `.wasm` 两个文件**（配套生成，不能只换一个）：
   ```bash
   source <emsdk>/emsdk_env.sh
   emcmake cmake -S native/wasm -B native/wasm/build && cmake --build native/wasm/build
   node native/wasm/scripts/gen-glue-file.js native/wasm/build   # 注意传 build 目录
   cp native/wasm/build/spine-runtime.{js,wasm} native/wasm/prebuilt/
   ```
   glue 生成脚本**默认从 `prebuilt/` 读输入**，不会自动用刚编译出的 `build/`。实际踩过的坑：编译成功但忘了带参数，生成的 glue 还是旧的。
6. JSB 侧不需要额外步骤（随正常原生工程构建）。但本地没有 emsdk 时，wasm 那侧就是「源码改完但产物没更新」的半成品，**必须明确告知用户**。
7. 验证 embind 绑定最快的方式：写个 Node 脚本 `require()` 编译出的 CJS glue + `fs.readFileSync` 对应 `.wasm`，手动 `instantiateWasm` 后直接调新函数。比在编辑器里反复试错快得多。**同一个 harness 也是验证「喂给插件的数据」的最好工具**：生成的 spine JSON / atlas 之类格式数据，错了的症状是静默丢 region 或干脆不渲染；在 Node 里把 解析 → 建实例 → 出渲染几何 全链路跑一遍，一次把数据格式和绑定正确性都盖上（样板：`D:\projects\performance\tools\verify-spine-data.mjs`）。

### 原生侧的其他坑

- 双 spine / 双同名 C++ 库共存要做命名空间隔离（`COMPILE_DEFINITIONS "spine=spine43"` 这类手法），否则**链接期静默选错符号**。再加一种场合：`native/external/` 里引擎预编译的第三方库，**各平台 `include/` 内容不一样**（实例：webp 在 android/ohos 只有 `decode.h`，win64/openharmony 才有 `demux.h`）。vendor 源码补功能时，自己那份公共头**写成一行转发到引擎按平台提供的那份**，让 ABI 常量和结构体布局始终跟实际链接的库一致而不是跟你 vendor 的版本一致；再用 `-include` 前缀头把 vendored 的符号整族重命名，这样在「本来就带这个模块」的平台上也不撞符号 —— 比按平台写 CMake 条件分支省事。
- JSB staging buffer 要显式复制，别持有引擎内部指针。
- 顶点格式要对齐，错位的症状是画面撕裂而不是崩。
- 小游戏平台有动态执行限制，`eval` 路线的绑定方案在那里跑不起来（具体链接选项见 §8.5）。
- **Native Simulator 与真机不同构，插件能不能带取决于引擎**（2026-09 实测定案，取代早先「未验证」的推断）：**stock 引擎的模拟器仍不带插件**（configure 从不调插件发现链，`globalThis.<binding>` 不存在）；**自定义引擎可加 `-DSIMULATOR_PLUGIN_SCAN_DIRS` opt-in**（引擎侧约 20 行，见 §9.5）让模拟器走与真机完全相同的发现链，装上即 `cc_load_all_plugins()` 注册 JSB（在模拟器 V8 里实测可见 `globalThis.spineruntime`）。原 `-DCMAKE_PROJECT_INCLUDE=<target>_simulator.cmake` 路线作废——那是把插件逻辑塞进模拟器构建的 workaround。配套不变：TS 侧 `if (NATIVE) { if (!binding) throw }`，**不要在原生上回退 wasm** —— 那会把「原生没接上」掩盖成「能跑但慢」，这种问题没人会注意到。模拟器预览本身的 staging 管线与黑屏排查见 §9。
- 自定义 UIMesh 类组件的生命周期方法**必须调 `super.onLoad()`**，漏了表现为不渲染。
- 两个后端（JSB / wasm）共用同一份 C 内核时，**逐帧输出必须逐字节相同**，拿一份公开语料的 md5 清单当 golden 去比 —— 「两边都能跑」和「两边结果一样」是两件事。

## 9. 模拟器预览（原生 exe）：黑屏排查与自动化（2026-09 实测）

模拟器预览不是浏览器预览换个壳——它是编辑器**staging 一份 JS 进预编译 exe** 里跑。自定义引擎下有两个结构性缺口，症状都是「窗口纯黑/纯白、零报错」，且**都不是插件的错**（插件 JSB 在黑屏实例里就已注册，可用 §9.4 直连验证）。

### 9.1 staging 管线：每次点预览都重做一遍（builtin preview 扩展 `browser/simulator.ts`）

1. 按工程勾选的模块生成 ~6KB `cc.js` 桩：`System.register(["2d", …, "webassembly", …])` —— **桩的 deps = 工程选定的 feature units，也是「模拟器必须能加载什么」的权威清单**（排查时直接数它）。
2. `<引擎>/bin/native-preview/` → `Release/src/cocos-js/`。**目录不存在时 `pathExistsSync` 静默跳过**——cocos-js 里只剩那个 6KB 桩就是这条路断了。
3. **`Release/src/import-map.json` 被编辑器安装目录 `resources/app.asar.unpacked/builtin/preview/static/simulator/import-map.json` 无条件覆盖**——你手工修 staged 那份只能活到下一次点击。
4. spawn `SimulatorApp-Win32.exe -workdir <Release> -writable-path <Release>`。

运行期 import-map 三层解析（staged `main.js`）：本地 `./src/import-map.json`（`cc-fu/*` → `./cocos-js/*.js`）→ 工程服务器 `http://<ip>:<port>/scripting/x/import-map.json`（其 `scopes` 把 chunk 的 `__unresolved_N` 映射到**裸 cc-fu id**）→ 基础 `{'cc':'cce:/internal/x/cc'}`。裸 id 本地 map 没有 → 直接挂。

### 9.2 自定义引擎黑屏的两个根因（按此顺序判）

1. **`bin/native-preview` 没人构建**：编辑器从不自动编译模拟器引擎 TS，唯一产出方是 开发者→编译模拟器引擎（只编译TS代码）菜单。症状：`Module no-schema:/src/cocos-js/2d.js did not instantiate (SystemJS Error#2)`。
2. **编辑器安装目录的 import-map 是 stock 引擎的 feature 表**：缺自定义引擎新增的 unit（实例：`cc-fu/webassembly`），工程 chunk 的 scope 引用裸 id → `Unable to instantiate cce:/internal/x/cc-fu/webassembly from …/chunks/…`。判法：比对 cc.js 桩 deps vs import-map keys vs cocos-js 里实际存在的 `<dep>.js`。

### 9.3 自动化杠杆：消息路由没有权限门槛

路由实现在 `@editor/creator/dist/message/protected/browser/index.jsg`（gzip，解压可读）：`request()` 只查 `messageMap[pkg][msg]` **存在性，无任何 public/权限检查** → 任何扩展可以 `Editor.Message.request('preview', 'build-simulator-engine-ts')` 调编辑器自己的构建器（与开发者菜单同一条 dispatch）。其非打包分支用 `query-engine-info` 的引擎路径：**同时重建 `bin/native-preview` 并重写安装目录 import-map（含该引擎全部 feature units）**——一次调用根治两个缺口，且零逻辑复刻、不手写编辑器目录。缺口检测全只读：native-preview 空？桩每个 dep 有文件、map 有键？样板：spine-runtime 扩展 `editor/simulator.js` 的 `ensureSimulatorEngineTS()`。

- 不想走消息时的头less 复刻（诊断用）：require **引擎自己的** `@cocos/ccbuild`（`{buildEngine, StatsQuery}`），options 逐项抄 `build-simulator-engine-ts.ts`：`moduleFormat:'system', compress:false, targets:'chrome 80', split:true, nativeCodeBundleMode:'wasm', platform:'WINDOWS', mode:'PREVIEW', flags:{DEBUG:true,SERVER_MODE:false}`，features = 全部 − gfx-webgpu − vendor-google − spine-* 再单选一个 spine 版本。
- ⚠️ 构建器第一步 `emptyDirSync(native-preview)`：后台构建期间用户点预览会 stage 到半空目录（照样黑屏）。编辑器也没有「预览前」钩子，接受「构建窗口内点击失败、完成后自愈」并在控制台说明。

### 9.4 直连模拟器 V8（CDP，零改源码的黑屏诊断法）

`%LOCALAPPDATA%\SimulatorApp-Win32\debugruntime\config.json` 记 `debugPort`（5086 起），**实例占用时端口顺延**——以 netstat 按 exe pid 查到的为准，别信配置里的数。`http://127.0.0.1:<port>/json/list` 拿 `webSocketDebuggerUrl`，node ≥22 自带 `WebSocket`。`Runtime.enable` 后用 `Runtime.evaluate {returnByValue, awaitPromise}` + `consoleAPICalled`/`exceptionThrown` 事件：`typeof cc`、场景树 walk（`c.__classname__`）、`cc.find('Canvas/…')`、XHR 探针、`typeof globalThis.<插件binding>`——「加载挂在哪一层」十分钟内定位，比截图猜快一个量级。

### 9.5 模拟器原生插件（引擎侧 opt-in，与真机同构）

引擎侧约 20 行（cocos4 `feat/plugin_support` 实测）：`templates/cmake/common.cmake` 的 `cc_gen_plugin_cmake_hook()` 扫描根 `CC_PLUGIN_PATH` 允许调用方预置；模拟器 CMakeLists 加 `SIMULATOR_PLUGIN_SCAN_DIRS` CACHE 变量（进 CMakeCache，后续 `cmake --build` 不必重传）走同一条 parser → `Pre-AutoLoadPlulgins.cmake` → `find_package` → `plugin_registry` 链；`Game.cpp` 补 `cc_load_all_plugins()`（模拟器的 Game 直接继承 CocosApplication，绕过了 `BaseGame::init()` 里的调用点——这就是插件链断的运行期原因）。**引擎 diff 里零插件名**：插件集永远是扫描结果这份数据。扩展侧用 stamp + 扫描根内容哈希守着，变了才后台 cmake（样板同 `editor/simulator.js`）。两个 CMake 坑：`target_link_libraries` 同一 target **混用 plain/keyword 签名是硬错误**（追加保持 plain）；parser 的不变量是「输出文件目录 == `CC_PROJECT_DIR`」（指到构建树内目录即可满足）。

### 9.6 Windows / PowerShell / git 工具链坑（通用，非模拟器专属）

- PS 5.1 `Set-Content -Encoding utf8` 写 **BOM** → V8 `JSON.parse` 报 `SyntaxError: Unexpected token '﻿'`（报错里那个不可见字符就是 BOM）。写任何会被 JS 读的 JSON 用 `[IO.File]::WriteAllText($f, $json, (New-Object System.Text.UTF8Encoding($false)))`。
- PowerShell 里 `stash@{0}` 是哈希表语法，要引号 `'stash@{0}'`；`git commit --fixup=<sha>` **不会自动 stage**，先 `git add`。
- 想给历史深处的提交补改动又不能用交互式 rebase 时：`git commit --fixup=<sha>` + `GIT_SEQUENCE_EDITOR=<脚本>` 的 `rebase -i --autosquash`，脚本（node 单文件）直接重写 todo 再退出，全程无真编辑器。

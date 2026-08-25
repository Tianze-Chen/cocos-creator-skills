---
name: cocos-creator-plugin
description: Use when scaffolding, extending, or debugging a Cocos Creator (3.8.x) editor extension/plugin under a project's extensions/ folder — especially one that needs a custom asset type + asset-db importer, generated engine assets (AnimationClip / SpriteAtlas / Spine), drag-and-drop into the scene or Inspector, a custom editor panel, extension-to-extension messages, or a native C++ runtime shared between JSB (native) and Emscripten/embind (wasm). Triggers on requests like "make a Cocos Creator plugin", "custom asset importer for Creator", "editor extension for Cocos", "generate an AnimationClip from code", "add drag-and-drop support to my plugin", "native/wasm binding for a Cocos Creator component", "why doesn't my asset-handler register", "why is my set-property not taking effect", or any work touching contributions.asset-db / contributions.scene / contributions.inspector / contributions.messages in a Creator package.json.
---

# Cocos Creator 插件开发

这份 skill **自包含**：所有骨架代码可以直接抄，所有结论都注明了来源。凡是没验证过的地方都明确写着「未验证」——那种地方要按「猜错也不会碎」的方式写代码，别当成事实。

结论基于 **Creator 3.8.8** 实测。换大版本前先按 §0 把版本确认一遍。

引擎侧的结论额外对着 **Cocos 4.0.0-alpha.31** 源码核过一遍，标了「3.8 / 4.0 一致」的那些可以跨这两个大版本用；没标的只保证 3.8.8。

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
   **schema 只管顶层字段和 `panels`；`contributions` 的 schema 是故意空的**，所以贡献点形态只能照内置扩展抄。
3. **编辑器的类型声明**（在 `app.asar` 里，但 `.d.ts` 是明文）：
   ```bash
   npx @electron/asar list  /Applications/Cocos/Creator/<版本>/CocosCreator.app/Contents/Resources/app.asar | grep '\.d\.ts'
   npx @electron/asar extract-file <app.asar 路径> 'builtin/asset-db/@types/protected/asset-handler.d.ts'
   ```
   `node_modules/@cocos/creator-types/editor/**` 下面是全套编辑器 API 声明。
   ⚠️ **实现是 `.ccc`，加密的**（`asset-handler-manager.ccc`、`libs/importer.ccc`），解不开。所以导入器的内部行为只能靠观测，没有静态替代路径。
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
| 把 TS 运行时代码交付给游戏（组件 / 资源类） | `asset-db.mount`（只读挂载） |
| 资源面板右键菜单 | `assets.menu` |
| 参数弹窗 / 工具窗口 | `panels` |
| 拖资源到已有节点的 Inspector 自动加组件 | `inspector.drop` + `messages` + 主进程转发 |
| 拖资源进层级面板自动建节点 | `scene.script` |
| 组件 / 资源自定义 Inspector 面板 | `inspector.section` |
| 被别的扩展或批处理脚本调用 | `messages` 里 `"public": true` |
| 监听资源变动 | `messages` 里订阅 `asset-db:asset-add` / `asset-db:asset-change` |
| 原生 C++ 运行时 | `native.plugins` + `native/cc_plugin.json`（见 §8） |
| Web / 小游戏也要同一套原生逻辑 | 独立 wasm（Emscripten + embind）+ `builder` 贡献复制产物（见 §8） |

**决策规则**：
- 只要涉及「自定义资源类型」，`asset-db.script` + `asset-handler` 是最小闭环。
- **只是要生成引擎已认识的资源（`.anim` / `.plist` / spine 三件套）就不要注册 asset-handler。** 输入后缀（`.png` / `.mp4`）早已被内置导入器认领，抢过来只会制造冲突；参数存在源资源的 `.meta` 里根本不需要自己的 importer。
- 不需要原生运行时就别碰 §8。

## 3. 骨架代码

### 3.1 package.json

**先说依据，因为这一节以前是靠记忆写的、错了三处。** 3.8.8 的 asar 里带着两份明文权威材料：

```
modules/editor-extensions/extensions/extension/static/extension-template/
    template/@types/schema/package/index.json        ← package.json 顶层的 JSON schema
    template/@types/schema/package/base/panels.json  ← panels 的 JSON schema
    html-panel/package.json                          ← 官方带面板模板
    blank/ vue2.x-panel/ vue3.x-panel/               ← 另三个官方模板
```

**但 schema 只覆盖顶层字段和 `panels`**——`contributions/index.json` 是**故意留空的**（`"properties": {}`），因为每个宿主包自己定义自己的 key。所以贡献点的形态没有 schema，唯一的事实来源是**内置扩展自己的 `package.json`**（`builtin/*/package.json`，明文，19 个）。下面每一条都注明了照的是哪一个。

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

贡献点的**声明**形态是确定的（照 `builtin/animator/package.json`，见 §3.1）：`{ node: [{ type: '<资源 ccclass 名>', message }] }`。

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

### 4.4 手写 Cocos 序列化格式（`.anim` 为例）

`.anim` / `.prefab` / `.scene` 都是同一种格式：**一个数组，元素之间用 `{"__id__": n}` 互相引用**，0 号是入口对象。

内联规则：**被 `@uniquelyReferenced` 标记的类内联**成 `{"__type__": ..., ...}`，其余 CCClass 实例在顶层数组里各占一格。反序列化器两种形态都吃，但照着序列化器的选择写，才不会「打开再保存就产生 diff」。

一条最小的逐帧 SpriteFrame 动画：

```json
[
  { "__type__": "cc.AnimationClip", "_name": "run", "_objFlags": 0, "_native": "",
    "sample": 12, "speed": 1, "wrapMode": 2, "enableTrsBlending": false,
    "_duration": 1, "_hash": 0, "_tracks": [{ "__id__": 1 }],
    "_events": [], "_exoticAnimation": null },
  { "__type__": "cc.animation.ObjectTrack",
    "_binding": { "__type__": "cc.animation.TrackBinding", "path": { "__id__": 2 } },
    "_channel": { "__id__": 4 } },
  { "__type__": "cc.animation.TrackPath", "_paths": [{ "__id__": 3 }, "spriteFrame"] },
  { "__type__": "cc.animation.ComponentPath", "component": "cc.Sprite" },
  { "__type__": "cc.animation.Channel", "_curve": { "__id__": 5 } },
  { "__type__": "cc.ObjectCurve",
    "_times": [0, 0.0833, 0.1667],
    "_values": [
      { "__uuid__": "<frame0>@ab12c", "__expectedType__": "cc.SpriteFrame" },
      { "__uuid__": "<frame1>@ab12d", "__expectedType__": "cc.SpriteFrame" },
      { "__uuid__": "<frame2>@ab12e", "__expectedType__": "cc.SpriteFrame" }
    ] }
]
```

逐字段核对的位置（引擎源码 `cocos/animation/` 下）：`animation-clip.ts`、`tracks/object-track.ts`、`tracks/track.ts`（`TrackBinding` 是那个 `@uniquelyReferenced` 的）、`target-path.ts`（`HierarchyPath.path` / `ComponentPath.component`）。

两个具体的坑：

- **`wrapMode` 要循环必须写 2。** `WrapModeMask`：`Default=0, Normal=1<<0=1, Loop=1<<1=2`（`cocos/core/geometry/curve.ts`）。Creator 自带的迁移 fixture 里写的是 `1`（Normal，只播一遍），照抄就会得到「播一遍就停」。
- **`_paths` 里空的 hierarchyPath 不要写成 `HierarchyPath('')`。** 动画组件所在节点自身就是不加 `HierarchyPath`；`HierarchyPath('')` 会走 `getChildByPath('')`，warn 一句然后返回 null。

`_hash` 写 0（运行时第一次取的时候自己算）。**拿不准的 `@serializable` 字段就整个省略**——有构造默认值的字段省略后是默认值，而猜错字段名比不写更糟。

**3.8 / 4.0 一致**：上面这些类名、`@uniquelyReferenced` 的位置、`@serializable` 字段名（`_paths` / `path` / `proxy` / `_binding` / `_curve` / `_channel` / `_times` / `_values`）、`WrapModeMask.Loop = 2`，在 3.8.8 和 4.0.0-alpha.31 里逐个核对过，一个没变——`track.ts` 里连行号都一样（60 / 310 / 324 / 453 / 514 / 542 / 561）。`CLASS_NAME_PREFIX_ANIM` 两版都是 `'cc.animation.'`（`animation/define.ts:25`）。`_times` / `_values` 不带 `@serializable` 装饰器，是 `CCClass.fastDefine` 注册的，源码里有注释专门说这事（`core/curves/keyframe-curve.ts:286-295`）。

### 4.5 生成引擎能直接播的资源，路只有这么几条

| 想要 | 唯一 / 首选路径 | 注意 |
|---|---|---|
| `cc.SpriteAtlas` + 逐帧 SpriteFrame 子资源 | 写 TexturePacker **`.plist`** | 3.8.8 的 55 个 asset-handler 里只有两个能产出 SpriteAtlas，另一个是 `auto-atlas` 的 `.pac`，它在**构建期重新打包**，不给你稳定的子资源 |
| `cc.AnimationClip` | 手写 `.anim`（见 §4.4） | 关键帧引用 UUID，必须在编辑器侧组装 |
| Spine 动画 | `skeleton.json` + `.atlas` + `.png` | 三个普通文件，**无 UUID 概念**，纯 CLI 就能产出，拖进工程即可 |

`.plist` 两代键名（format 2 `frame`/`offset`/`sourceSize`，format 3 `textureRect`/`spriteOffset`/`spriteSize`/`spriteSourceSize`/`textureRotated`）**同时写出**最稳，代价只有几百字节。

### 4.6 Spine：两个版本、两种互不兼容的 atlas 格式

3.8.8 **内置双版本 spine 运行时**，默认 3.8，可切 4.2。哪个进包看工程配置：

```bash
grep -o '"spine-[0-9.]*"' <工程>/settings/v2/packages/engine.json | sort -u
# includeModules 里出现的那个才是构建进包的
```

`.atlas` 格式必须和运行时版本对上：3.8 世代是 `rotate/xy/size/orig/offset/index`，4.x 是 `bounds/offsets`。**对不上的症状是 region 静默丢失或错位，没有报错。**

判断内置的是哪一代，看引擎里 `cocos/spine/lib/spine-core.d.ts`：有 `setFrame(frameIndex, time, …)` / `rotateMix` / `AtlasRegion{rotate,offsetX,offsetY}` 就是 3.8 世代。**4.0.0-alpha.31 里这几个签名还在**，所以 4.x 默认仍是 3.8 世代的旧 atlas 格式。

引擎侧真正的判定依据是 `cc.config.json` 里那个 module 的 `intrinsicFlags`（`SPINE_3_8` / `SPINE_4_2`），工程侧是 `settings/v2/packages/engine.json` 里 `spine._option`。4.x 还多了一套 `spine-version-dynamic.ts` / `spine-instantiate-dynamic.ts`（运行时可选版本，靠 `moduleOverrides` 换文件）——**如果引擎是这种构建，光看 `SPINE_VERSION` 常量会得到错的答案**，那个文件里的 `export const SPINE_VERSION = '3.8'` 只是给固定版本构建用的占位。

顺带两条：spine 的 `.json` 要和 dragonbones 抢导入器（见 §4.2）；`premultipliedAlpha` 勾错了表现为半透明边缘发暗。

### 4.7 其他零散事实（3.8.8）

- **`SpriteType.MESH` 不可用**：`cocos/2d/components/sprite.ts` 里 `// MESH: 4` 和 `/* || this._type === SpriteType.MESH */` 都是注释掉的。想做紧贴轮廓的 mesh 裁剪得自己写渲染组件。**3.8 / 4.0 一致**，连行号都没动（83 / 385）——别指望升 4.x 会恢复。
- **SpriteFrame 的 trim 字段齐全**：`_rect` / `_offset` / `_originalSize` / `_rotated`（`cocos/2d/assets/sprite-frame.ts`），序列化出来是 `{__type__, content:{rect, offset, originalSize, rotated, capInsets, vertices, …}}`。逐帧裁剪 + 还原位置用这套原生机制就够，不用自己算顶点。**3.8 / 4.0 一致**。
- **`SpriteFrame.offset` 是 y 向上的**（裁剪区中心 − 原始帧中心），而 `rect` 是图像坐标系（原点左上）。**同一个文件里混着两种 y 方向**，换算集中到一个函数里，别在多处重算。符号约定看 `_calcTrimmedBorder()`，**3.8 / 4.0 逐行相同**。验它最省事的办法是拿一段**上下移动**的素材烘一遍，看上半的帧 `offset.y` 是不是正的——纯读代码很容易把符号读反。
- **`cc.SpriteAtlas` 的序列化形状会让人误判**：字段是 `spriteFrames`（**无下划线**）、类型是字典，但它有自定义 `_serialize()` 把字典**摊平成交错数组** `[key0, uuid0, key1, uuid1, …]`。所以在 `library/*.json` 里看到 `content.spriteFrames` 是数组是正常的，不是版本差异（`2d/assets/sprite-atlas.ts:54,113-149`，**3.8 / 4.0 逐行相同**）。
- **不要凭猜写 `compressSettings`**：它在 Cocos 自己的类型里是 `Record<string, any>` 且带 `TODO`，预设 id 还是项目本地的，猜着写会静默失效。重新生成资源时**完整保留**上一份 `userData` 和每个 `subMetas[*].userData`，第一次导入时提示用户自己设。

## 5. 调试与验证

- **静默失败优先怀疑五个地方**：`set-property` 写了不存在的属性（§3.7）、`.meta` 里 importer 被复用（§4.2）、面板主文件用了 `export default`（§3.3）、`package.json` 缺 `package_version`（§3.1，症状是扩展管理器里看不见但其实已加载）、**`panels` 被写进了 `contributions` 里**（§3.1，症状是扩展一切正常只有开面板报 `Panel(...) is not defined`）。这五样都不报错，或者只报一行指向错方向的 warn。
- **`package.json` 结构错位这一类问题，症状永远指向错的地方**：扩展照常加载、消息照常收发、菜单照常出，于是你会去查脚本、查 `module.exports`、查路径，而错的地方在 JSON 的缩进层级里。所以贡献点**照着 §3.1 的 schema 和内置扩展抄，不要凭记忆写**——我凭记忆写的那份骨架，`panels` 位置、`assets.menu` 形态、`inspector.drop` 形态三处全错。
- **Cocos 报错的位置和真正的错误位置几乎从不重合**，上面那条只是其中一例。`Panel(...) is not defined` 指向面板脚本而错在 `package.json`；`parameter error` 指向 asset-db 内部而错在你某个循环的边界条件（§4.3.1）；`.effect` 下载失败 / 渲染管线为 null / 材质属性非法这一整串场景报错，错在**引擎目录的 `package.json`** 被换掉了（§0）。而栈都在 `.ccc` 里，看不了。**所以顺序是「先给每个跨进程调用加上带实参的标签，再开始分析」**——加标签的收益是确定的，分析的收益不是。
- **不是所有失败都安静**：也有反过来的——**扩展在后台无限重复干活**。如果它既写资源又监听资源变动，先怀疑自激循环（§4.3.2），判据是日志里同样几行按固定周期重复。这个不看日志基本发现不了，因为 UI 上什么都不弹。
- **「扩展装了没生效」先去看 `<工程>/temp/logs/project.log`**，别在扩展管理器里瞪着看。加载成功的话你自己 `load()` 里打的日志会出现在里面，`[Package] <name>: …` 和 `Panel(...) is not defined` 那几行 warn 也在，一眼就能分清「没加载」「加载了但没显示」「加载了但某个贡献点没注册」。
- `<工程>/temp/asset-db/log/*.log` 是导入问题的第一现场。
- 引擎侧的行为（资源能不能播、属性名对不对）**没法靠 mock 单测**——mock 出来的断言等于把实现抄一遍。把纯计算部分抽成零依赖模块去单测，剩下的消息往返写一份**带症状→成因对照表**的人工验证清单，比假装有覆盖率诚实得多。
- 格式类输出（`.atlas` / `.plist` / `.anim` / `skeleton.json`）用 **golden 文件逐字节比对**。这几种格式写错的症状都是「能导入但不显示」，没有 golden 守着基本查不出来。

## 6. 集成到工程 & 发布前检查

- 插件文件夹（含 `package.json` 的那一层）整个放进目标工程 `extensions/` 即可被识别；`asset-db.mount` 的 `runtime/` 自动只读挂载为项目脚本。
- 扩展目录里的 `node_modules` 要**在目标工程里装**（`cd extensions/<name> && npm install`），别指望能借到工程或编辑器的依赖。
- 编译产物尽量**自包含**：让扩展的 `tsconfig.json` 把共享的源码目录一起编译进 `dist`（`rootDir` 指到共同父目录），比引 workspace 软链靠得住。
- TS 运行时代码 `import from 'cc'` **不需要本地 tsconfig**——类型由挂载它的工程环境提供。编辑器侧脚本是 CommonJS（`require`、全局 `Editor` / `cce`）。**两条线不要混。**
- 涉及原生：目标引擎必须导出插件用到的引擎能力（例如 `cc.UIMesh` + 公开的 `cc.wasm`）——集成前先确认目标是不是这样的自定义引擎，不是的话原生功能装了也用不了。

## 7. 参数暴露：一条设计原则

把重活挪到编辑期，换掉的是**运行时成本，不是存储成本**。小游戏 500KB 首包之类的约束一点没变。

所以生成资源的插件**必须在界面上摆一个实时的体积 / 容量估算**，否则用户会稳定产出几 MB 的资源，而且要到上线前才发现。

配套的一条：**校验函数只放硬错误。** 决定「烘焙按钮能不能点」的那个函数里混进「能跑但不划算」的建议，会把用户拦在一个其实完全合法的参数上。建议另开一条通道显示、不禁用按钮。

## 8. 原生 C++ / wasm（只在真要跨语言共享同一份逻辑时读）

这一节的复杂度（命名空间隔离、AOT embind、JSB 手动绑定）只在**真的要让 native 和 web 跑同一份 C++ 逻辑**时才值得。纯 TS 组件 + 自定义资源导入，做到 §1–6 就完整可用。

核心结构：**一份 C++ facade 头文件**，JSB（`se::Object` / `se::Class`，`SE_BIND_FUNC`）和 embind（`EMSCRIPTEN_BINDINGS`）各写一层薄绑定，TS 侧按 `NATIVE` 宏**严格二选一**（不是「没有 JSB 就 fallback 到 wasm」）。

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
7. 验证 embind 绑定最快的方式：写个 Node 脚本 `require()` 编译出的 CJS glue + `fs.readFileSync` 对应 `.wasm`，手动 `instantiateWasm` 后直接调新函数。比在编辑器里反复试错快得多。

### 原生侧的其他坑

- 双 spine / 双同名 C++ 库共存要做命名空间隔离（`COMPILE_DEFINITIONS "spine=spine43"` 这类手法），否则**链接期静默选错符号**。
- JSB staging buffer 要显式复制，别持有引擎内部指针。
- 顶点格式要对齐，错位的症状是画面撕裂而不是崩。
- 小游戏平台有动态执行限制，`eval` 路线的绑定方案在那里跑不起来。
- Native Simulator 不是普通原生工程，别拿它的结果推断真机。
- 自定义 UIMesh 类组件的生命周期方法**必须调 `super.onLoad()`**，漏了表现为不渲染。

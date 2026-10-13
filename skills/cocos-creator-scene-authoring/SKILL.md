---
name: cocos-creator-scene-authoring
description: Use when creating or editing Cocos Creator (3.8.x) scenes, nodes, or components without the editor GUI — generating .scene and .meta files by hand or script, mounting custom script components into a scene, serializing static UI (Sprite/Label/Button/ScrollView) directly into the scene file so layout is visible in editor, wiring button clicks via serialized cc.ClickEvent (target component + handler method), building UI at runtime with zero image assets, or positioning/moving nodes and UI elements (lpos vs UITransform vs Widget, Canvas design-resolution coordinates). Triggers on requests like "create a scene programmatically", "hand-write scene JSON", "what's the scene meta format", "compressed uuid in __type__", "attach a script component to a scene file", "serialize nodes into the scene asset", "bind button click in the scene file", "create nodes by code", "add a button without assets", "move a node from code", or any task that must batch-create scenes/nodes/components in a Creator project from CLI or scripts.
---

# Cocos Creator 场景 / 节点 / 组件的代码化创建

这份 skill 教你**不开编辑器**完成:建场景 → 挂脚本 → 静态 UI 序列化进场景 → 按钮点击绑定 → (运行时)建动态节点 → 摆放位置。
所有结论均在 **Creator 3.8.8** 上实证过,样板工程:`D:\projects\performance`(多场景性能测试工程,全部场景/静态 UI/点击绑定由脚本生成)。
标注「未验证」的地方不要当事实用。

三个信息源,遇到分歧时以下面的优先级为准:

| 优先级 | 来源 | 本机路径 |
|---|---|---|
| 1 | 官方模板场景(编辑器自带,3.8.8 原始字节) | `C:\ProgramData\cocos\editors\Creator\3.8.8\resources\resources\3d\engine\editor\assets\default_file_content\scene\scene-2d.scene` |
| 2 | 引擎源码(和 3.8.8 编辑器内置版本一致) | `D:\code\cocos-engine` |
| 3 | 实证工程(脚本生成的场景 + 静态 UI 序列化 + 运行时 UI 工厂) | `D:\projects\performance` |

## 1. 心智模型

- **场景文件是纯 JSON 数组**(`assets/xxx/yyy.scene`),没有二进制、没有魔法。数组元素通过 `{"__id__": N}` 互相引用,N 是数组下标。这套容器格式的总论(`@uniquelyReferenced` 内联规则、省字段的安全性等)见 cocos-asset-formats skill。
- **资源的身份在 `.meta` 文件里**(uuid),不在文件本身。场景引用脚本、其他资源引用场景,全部靠 uuid。**meta 一旦提交,uuid 就不能再变**,否则所有引用断链(变成 Missing Script / 丢失资源)。
- **场景里挂自定义脚本组件,`__type__` 写的是脚本 meta uuid 的「压缩形式」**(23 字符,见 §4)。编辑器生成的工程全部这么写,手写场景照做即可。
- **静态/动态分界**:布局固定、不随数据变的 UI(背景/标题/按钮/滚动区骨架)直接序列化进场景,编辑器打开即可见;数量不定、每帧变化的内容运行时再建,代码按**节点名**取回挂载点(`getChildByName`)。这个混合模式是本工程的标准做法。
- 先写脚本 + 脚本 meta(uuid 固定),再写场景 —— 顺序反了会导致场景引用落空。

## 2. 创建一个场景(最小 2D 模板)

一个带 Canvas + Camera + 一个脚本组件的 2D 场景,完整 JSON 见下(对照官方模板逐字段核对过,可直接抄):
`D:\projects\performance\tools\gen-scenes.mjs` 里的 `createSceneBuilder()` 是这份模板的参数化实现(16 个基础对象 + 静态 UI 追加器),批量建场景直接复用它。

数组结构(前 16 个对象,下标即 `__id__`):

| __id__ | __type__ | 说明 |
|---|---|---|
| 0 | `cc.SceneAsset` | 资产壳,`scene: {"__id__": 1}` |
| 1 | `cc.Scene` | 场景根(Node 子类),`_children` 指向 Canvas,`_globals` 指向 id 8,**`_id` = 场景自己的 uuid** |
| 2 | `cc.Node` "Canvas" | `_components`: `[5(UITransform), 6(cc.Canvas), 7(cc.Widget), 15(你的脚本)]`,**`_layer`: 33554432(UI_2D)**,`_lpos`: 设计分辨率半宽半高(如 960x640 → (480, 320)) |
| 3 | `cc.Node` "Camera" | Canvas 的子节点,`_layer`: 1073741824(照抄模板) |
| 4 | `cc.Camera` | UI 相机,关键字段见下表 |
| 5 | `cc.UITransform` | Canvas 的尺寸 = 设计分辨率 |
| 6 | `cc.Canvas` | `_cameraComponent: {"__id__": 4}` **必须指**,否则 Canvas 没相机 |
| 7 | `cc.Widget` | 4 边对齐屏幕(`_alignFlags: 45`) |
| 8-14 | `cc.SceneGlobals` + Ambient/Shadows/Skybox/Fog/Octree/Skin Info | **整段从官方模板 verbatim 复制**,别手改 |
| 15 | 你的脚本组件 | `__type__` = 压缩 uuid(§4),`node: {"__id__": 2}` |

Camera(id 4)关键字段照抄模板:`_projection: 0`(正交)、`_clearFlags: 7`、`_orthoHeight: 10`(UI 相机不用真算,2D 渲染自己处理)、`_visibility: 1108344832`。

节点通用字段:`_name/_objFlags/_parent/_children/_active/_components/_prefab/_lpos/_lrot/_lscale/_layer/_euler/_id`。
**省字段是安全的**:反序列化只覆盖 JSON 里出现的键,其余用类默认值 —— 但 `_parent/_children/_components/_id/_layer` 这些结构字段建议全写。
渲染顺序 = 同级 `_children` 数组顺序,后写的在上层。

同时必须建 meta(与场景文件同目录同名加 `.meta`):

```json
{
  "ver": "1.1.50",
  "importer": "scene",
  "imported": true,
  "uuid": "<随机 uuid v4>",
  "files": [".json"],
  "subMetas": {},
  "userData": {}
}
```

新建目录也要 meta(`importer: "directory"`,ver `1.1.0`,userData 里带空的 `compressionType`/`isRemoteBundle`);不建的话编辑器会自动补,无害。

## 3. 脚本与脚本 meta

- 脚本放 `assets/scripts/`(或任意 assets 子目录),每份 `.ts` 配一份 `.ts.meta`:

```json
{
  "ver": "4.0.23",
  "importer": "typescript",
  "imported": true,
  "uuid": "<随机 uuid v4>",
  "files": [],
  "subMetas": {},
  "userData": {}
}
```

- uuid 用随机 v4,全局唯一即可;脚本内容改了不影响 uuid。
- 挂进场景的类必须有 `@ccclass('唯一名')`,继承 `Component`;抽象基类挂不进场景,只有具体子类出现在场景 JSON 里。
- 组件的可序列化字段(`@property`)在场景 JSON 的组件对象里**按字段名**直接写值;没有可序列化字段的组件条目只写结构字段即可。
- 引擎模块开关在工程 `settings/v2/packages/engine.json` 的 `includeModules`:2d/ui/tween/graphics/mask/physics-2d-box2d 等按需确认,没开的模块对应 API 在预览构建里不存在。

## 4. uuid 压缩算法(`__type__` 的 23 字符)

场景/预制体里引用自定义类时,`__type__` = uuid 去掉横线(32 位 hex)后:

- **前 5 位 hex 原样保留**;
- 其余 **27 位 hex,每 3 位(12 bit)编成 2 个 base64 字符**(标准字母表 `A-Za-z0-9+/`),共 9 组 18 字符;
- 总长 5 + 18 = **23 字符**。

```js
const B64 = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/';
function compressUuid (uuid) {
  const hex = uuid.replace(/-/g, '');
  let out = hex.slice(0, 5);
  for (let i = 5; i < 32; i += 3) {
    const v = parseInt(hex.slice(i, i + 3), 16);
    out += B64[v >> 6] + B64[v & 63];
  }
  return out;
}
```

实证:`0571b046-1cda-44ef-b6cd-dc27b6caa0de` ↔ `0571bBGHNpE77bN3Ce2yqDe`(与编辑器生成工程逐字符一致)。

注意区分:资源引用里的 22 字符压缩 uuid(`decodeUuid()` 那套,前 2 位保留)是**另一种**格式,别混用。组件 `__type__` 只用本节 23 字符版。

## 5. 静态 UI 直接序列化进场景文件

目的:布局在编辑器/预览里**不开运行**就能看到。节点/组件对象按 §2 的规则追加到数组尾部,并挂到父节点的 `_children` 里。完整参数化实现见 `D:\projects\performance\tools\gen-scenes.mjs`(`createSceneBuilder` 的 `node/sprite/label/button` 方法)。

每个 UI 节点 = 1 个 `cc.Node` + 1 个 `cc.UITransform`(`_contentSize`/`_anchorPoint`)+ N 个渲染/逻辑组件。组件对象通用结构字段:`{"__type__", "_name":"", "_objFlags":0, "node":{"__id__"}, "_enabled":true, "__prefab":null, ...业务字段..., "_id":"<随机>"}`。

### 5.1 cc.Sprite(引用图片子资产)

```json
{
  "__type__": "cc.Sprite", "node": {"__id__": N}, "_enabled": true, "__prefab": null,
  "_color": {"__type__": "cc.Color", "r": 24, "g": 28, "b": 38, "a": 255},
  "_spriteFrame": {"__uuid__": "<图片主uuid>@f9941", "__expectedType__": "cc.SpriteFrame"},
  "_type": 0, "_sizeMode": 0
}
```

- `_sizeMode: 0`(CUSTOM)= 跟随节点 UITransform 尺寸,纯色块/拉伸背景必须用这个。
- **每张图片的 sprite-frame 子资产 id 固定是 `f9941`**,texture 子资产固定是 `6c48a`(引擎图片导入器约定)。场景里引用精灵图一律 `uuid@f9941`。
- 混合因子(`_srcBlendFactor` 等)可不写,走类默认。

### 5.2 cc.Label(免字体文件)

```json
{
  "__type__": "cc.Label", "node": {"__id__": N}, "_enabled": true, "__prefab": null,
  "_color": {"__type__": "cc.Color", "r": 232, "g": 236, "b": 244, "a": 255},
  "_string": "标题文本", "_horizontalAlign": 1, "_verticalAlign": 1,
  "_fontSize": 24, "_lineHeight": 32, "_overflow": 0, "_enableWrapText": false,
  "_isSystemFontUsed": true, "_isBold": false, "_isItalic": false
}
```

`_horizontalAlign`: 0=LEFT 1=CENTER 2=RIGHT;`_overflow: 0`=NONE。系统字体免字体资产。锚点左对齐文本 = 节点 UITransform `_anchorPoint: {0, 0.5}`。

### 5.3 cc.Button —— 有一个必踩的颜色坑

```json
{
  "__type__": "cc.Button", "node": {"__id__": N}, "_enabled": true, "__prefab": null,
  "clickEvents": [], "_interactable": true,
  "_transition": 1,
  "_normalColor": {"__type__": "cc.Color", "r": 58, "g": 118, "b": 210, "a": 255},
  "_hoverColor":   { ...同底色... },
  "_pressedColor": { ...底色×1.4... },
  "_disabledColor": {"__type": "cc.Color", "r": 124, "g": 124, "b": 124, "a": 255},
  "_duration": 0.1, "_zoomScale": 1.2, "_target": {"__id__": N}
}
```

- **坑(引擎 `cocos/ui/button.ts`)**:Button 启用时 `_resetState()` 会执行 `renderComp.color = this._normalColor` —— **直接覆盖同节点 Sprite 的 `_color`**。`_normalColor` 类默认是纯白。所以背景色**必须**写进 `_normalColor`/`_hoverColor`(按下色用亮 1.4 倍),只给 Sprite 上色会被冲成白色。运行时同理:`btn.normalColor = bg` 而不是 `sprite.color = bg`。
- `_transition: 1` = COLOR 过渡;`_target` 指按钮自己的节点。

### 5.4 ScrollView / Mask / Layout(长列表骨架)

序列化字段名与运行时属性名**不一致的要特别注意**(反序列化按序列化名来):

| 组件 | 序列化字段 | 值(垂直列表) |
|---|---|---|
| `cc.ScrollView` | `_content`(指向 content 节点)、`_horizontal`、`_vertical` | `false` / `true` |
| `cc.Mask` | `_type`(默认 0 = GRAPHICS_RECT,矩形裁剪可整个省略) | 省 |
| `cc.Layout` | **`_layoutType`**(不是 `_type`)、`_resizeMode`、`_paddingTop`、`_spacingY` | `2`(VERTICAL)、`1`(CONTAINER)、`2`、`2` |

节点链:`scrollview → view(挂 Mask) → content(挂 Layout,锚点 (0.5,1) 顶部对齐向下生长)`。

### 5.5 白图资产:1×1 PNG + image meta(序列化 Sprite 的依赖)

场景 Sprite 必须引用一个真实图片资产。免美术资源的标准做法:生成 1×1 白 PNG,场景里染色用。完整生成代码见 gen-scenes.mjs 的 `makeWhitePng()/whitePngMeta()`。

- PNG 用 node 标准库拼:`zlib.deflateSync(滤波字节0+RGBA像素)` 做 IDAT,`zlib.crc32(type+data)` 做 CRC。**chunk 布局 = len(4) + type(4) + data + crc(4)**,Buffer 长度算错会静默产出坏文件(解码器按长度字段走,错位后 IDAT 长度读成 0)。
- image meta(ver `1.0.27`,importer `image`):`subMetas` 固定两把 —— `6c48a`(texture,ver 1.0.22)和 `f9941`(sprite-frame,ver 1.0.12),子 uuid = `主uuid@id`。sprite-frame 的 `userData` 需要 `width/height/rawWidth/rawHeight/offsetX/offsetY/trimX/trimY/vertices{rawPosition,indexes,uv,nuv,minPos,maxPos}/trimType:"auto"/packable/pivot*`;顶层 `userData.redirect` 指 `uuid@6c48a`。顶点数据编辑器导入时会重算,抄 1×1 的模板值即可。
- 场景里 `packable: false`(不进动态图集),运行时染色行为可控。

## 6. 按钮点击绑定:场景里的 clickEvents

除了代码 `node.on(Button.EventType.CLICK, ...)`,还**可以在场景资产里声明**"点这个按钮 → 调那个组件的那个方法" —— 即编辑器 Inspector 里 Button 的 Click Events。手写场景照下面格式,和编辑器产出完全等价,Inspector 里可见可改:

```json
"clickEvents": [
  {
    "__type__": "cc.ClickEvent",
    "target": {"__id__": 2},
    "component": "MainScene",
    "handler": "onSelectTest",
    "customEventData": "physics2d-test"
  }
]
```

规则(源码 `cocos/scene-graph/component-event-handler.ts`,3.8.8 逐行核对):

- `target` = **挂目标组件的节点**(通常 Canvas,`{"__id__": 2}`);`component` = **@ccclass 名字符串**(不是路径);`handler` = 方法名,**必须 public**,基类继承来的方法也算数。
- `customEventData` = 任意字符串,一个 handler 靠它区分多个按钮(如主菜单 6 个按钮全指 `onSelectTest`,data 各自带场景名)。
- 回调签名:`handler(event, customEventData?)` —— customEventData 为空串时**不会**传第二个参数,所以带 data 的方法写成 `onFoo (_event: Event, data: string)`,不带 data 的写成零参。
- `_componentId` 字段可省略:emit 时 `_genCompIdIfNeeded()` 会从 `component` 名自动推导(编辑器保存时可能会把它物化成压缩 uuid 并清空 `component`,两种形态等价)。
- 组件不存在 / 方法不存在时**静默 return,不报错** —— 手写场景务必跑一遍 handler 存在性校验(gen-scenes.mjs 同目录的 `verify-handlers.mjs` 干这个)。

推荐分工:**静态按钮全用 clickEvents**(绑定关系进资产、编辑器可见、代码零胶水);运行时动态创建的按钮才用 `node.on`。需要改按钮文字时用 `getButton(root, name)` 按名取回(UIFactory.ts)。

## 7. 运行时创建节点 / 组件(零资源依赖 UI 工厂)

完整实现见 `D:\projects\performance\assets\scripts\core\UIFactory.ts`,要点:

```ts
import { Layers, Node, UITransform, Sprite, SpriteFrame, Texture2D, Rect,
         Label, Button, Color } from 'cc';

// 1) 建节点:layer 必须显式设为 UI_2D,否则 UI 相机不渲染(new Node 默认值别依赖)
const n = new Node('name');
n.layer = Layers.Enum.UI_2D;
parent.addChild(n);

// 2) UI 节点先加 UITransform 定尺寸锚点(Sprite/Label 会自动依赖它)
const ut = n.addComponent(UITransform);
ut.setContentSize(100, 40);
ut.setAnchorPoint(0.5, 0.5);

// 3) 纯色贴图免图片资产:reset + uploadData(注意:3.8 没有 initWithData!)
const tex = new Texture2D();
tex.reset({ width: 1, height: 1, format: Texture2D.PixelFormat.RGBA8888 });
tex.uploadData(new Uint8Array([r, g, b, 255]));
const sf = new SpriteFrame();
sf.texture = tex;
sf.packable = false;              // 不进动态图集,合批行为可控
sf.rect = new Rect(0, 0, 1, 1);   // 默认 rect 是 0x0,不设会渲染不出来

// 4) Sprite 用 CUSTOM 尺寸模式,否则会被 1x1 纹理重置为 1 像素大
const sp = n.addComponent(Sprite);
sp.sizeMode = Sprite.SizeMode.CUSTOM;
sp.spriteFrame = sf;
sp.color = someColor;             // 白纹理 × color = 纯色块

// 5) Button:COLOR 过渡 + target 指自己;背景色必须写 normalColor(§5.3 的坑)
const btn = n.addComponent(Button);
btn.transition = Button.Transition.COLOR;
btn.target = n;
btn.normalColor = bg.clone();     // 只设 sp.color 会被 _resetState 冲掉!
btn.node.on(Button.EventType.CLICK, () => {});
```

### 7.1 packable 必须 false:动态图集会把原始字节贴图喂炸 WebGL(实证 2026-09)

`sf.packable = false` 不是可选项,是**防爆开关**。完整因果链(在 3.8.8 web 预览上复现,异常每帧 ~50 条直到页面不可用):

1. `macro.CLEANUP_IMAGE_CACHE` **web 默认 false** → `DynamicAtlasManager.init()` 执行 `enabled = !CLEANUP_IMAGE_CACHE` = **true**,动态图集默认开着(原生/小游戏才强制 true→图集关)。
2. Sprite 渲染时 `Simple.updateRenderData` → `packToDynamicAtlas(frame)`,只要 `frame.packable && !frame.original && 贴图宽高>0` 就往图集塞。
3. `Atlas.insertSpriteFrame` → `drawTextureAt` 把 **`ImageAsset.data` 原样传给** `gl.texSubImage2D(target, level, x, y, format, type, source)`(引擎 `webgl2-commands.ts` 的 7 参 DOM-source 形态)。源是 DOM 图(HTMLImageElement/Canvas/ImageBitmap)没问题;**源是 `Uint8Array`(reset+uploadData / builtin 白图的原始字节)时 WebGL2 没有任何重载匹配** → `TypeError: Overload resolution failed`,未捕获,而且塞不进去导致下一帧重试 → 永动异常风暴。
4. 引擎侧的洞(均为上游原版,不是本地改坏的):`builtin-res-mgr.ts` 的 `default-spriteframe` 是 `new SpriteFrame()`(packable 默认 true)包着原始字节 ImageAsset;`sprite-frame.ts _checkPackable` 只会关不会开,且它的 canvas 判断(`texture.image instanceof HTMLCanvasElement`,image 是 ImageAsset,永 false)是死代码 → packable 永远不会被自动关掉。

**推论:永远不要把 `builtinResMgr.get('default-spriteframe')` 直接挂到会被渲染的 Sprite 上。** 纯色块要走 §7 的「自有贴图 + 自有 SpriteFrame(packable=false)」;想复用 builtin 白贴图就包一层:

```ts
const frame = new SpriteFrame();
frame.texture = builtinResMgr.get<Texture2D>('default-texture'); // 借 2x2 白贴图
frame.packable = false;                                          // 关键
```

样板:`extensions/animated-image/runtime/AnimatedImageDemo.ts` 的 `getWhiteFrame()`。每帧更新内容的贴图(如动图播放器)更必须 false —— 就算能塞进图集,塞进去的也是旧帧快照。

**排查这类「页面直接 crash」的工具**:`node tools/repro-demo-crash.mjs <场景hash> [秒数] [--shim-white]`(D:\projects\performance)—— headless Chrome + CDP(Node 22 原生 WebSocket,零依赖),`Page.addScriptToEvaluateOnNewDocument` 在页面脚本前挂钩子(texSubImage2D 实参身份 / dynamicAtlasManager.enabled 赋值栈 / builtinResMgr 热替换),配 `Runtime.evaluate` 场景树遍历 + 截图。比人肉点页面快得多,且能拿到「到底哪个 Sprite 的哪个 frame 出事」。

ScrollView / Layout / Mask(长列表)与 RigidBody2D/BoxCollider2D(2D 物理)的运行时搭建,见 `D:\projects\performance\assets\scripts\tests\UIListTest.ts` 和 `Physics2DTest.ts` —— 都是 `addComponent` 顺序组装,无需预制体。字段名差异同 §5.4(`layout.type` 运行时属性 vs `_layoutType` 序列化名)。

## 8. 位置与移动

坐标系:Canvas 中心为原点,设计分辨率(本工程 960x640)决定边界;**y 向上为正**;子节点坐标相对父节点。

```ts
n.setPosition(x, y, 0);        // 写
const p = n.position;          // 读(Vec3,只读视图,别缓存)
n.getPosition();               // 同上
n.translate?                    // —— 不存在这个 API,移动就是 getPosition+setPosition
```

- **位置(`_lpos`/`position`)管变换,`UITransform` 管布局**:锚点决定"位置点"在自身的哪里;改尺寸不影响 position,但视觉中心由锚点决定。左对齐文本: `setAnchorPoint(0, 0.5)` 后 x 即左边缘。
- 场景 JSON 里节点位置写 `_lpos: {"__type__": "cc.Vec3", "x":.., "y":.., "z":0}`(Canvas 用设计分辨率半宽高)。
- **响应式对齐用 Widget**:`addWidget.isAlignTop = true; w.top = 0;`(或代码 `n.addComponent(Widget)` 后逐属性设),别手写屏幕尺寸换算。
- 同级节点**按 children 数组顺序渲染**,后加的在上层;**没有事件监听的节点不拦截点击**(Sprite 盖在 Button 上不挡点击),需要挡再给上层节点挂 BlockInputEvents。
- `destroy()` 延迟到帧末生效,当帧仍可访问;批量删除子节点用 `node.destroyAllChildren()`。
- 每帧驱动逻辑:组件里 `update(dt)` 或 `this.schedule(fn, 0)`(interval 0 = 每帧,合法)。

## 9. 3.8 的 API 命名坑(全在 3.8.8 上踩实)

| 错误写法(旧版本/凭印象) | 3.8.8 正确写法 |
|---|---|
| `Label.HorizontalAlignments.LEFT` | `Label.HorizontalAlign.LEFT` |
| `Layout.SizeMode.CONTAINER` | `Layout.ResizeMode.CONTAINER`(序列化名是 `_resizeMode`) |
| 序列化 Layout 写 `_type` | 序列化字段名是 **`_layoutType`**(`_type` 会被静默忽略) |
| `Layout.HorizontalDirection.CENTER` | 枚举只有 `LEFT_TO_RIGHT / RIGHT_TO_LEFT`,居中靠子节点锚点 |
| `ERigidBody2DType.STATIC` | `ERigidBody2DType.Static`(成员是 PascalCase) |
| `Texture2D.initWithData(...)` | **不存在**;用 `reset({width,height,format})` + `uploadData(data)` |
| `new SpriteFrame()` 直接用 | 默认 `rect` 是 (0,0,0,0),必须显式 `sf.rect = new Rect(...)` |
| Sprite 直接设 spriteFrame 期望随节点大小 | `sizeMode` 默认会取纹理尺寸,固定大小要 `Sprite.SizeMode.CUSTOM` |
| Button 背景色设 `sprite.color` | Button 启用时会用 `_normalColor` 覆盖渲染组件 color(§5.3),背景色必须写进 `normalColor/hoverColor` |
| `new BufferAsset(); ba.buffer = bytes` | 没有 `buffer` setter;塞字节是 `ba._nativeAsset = arrayBuffer`(deprecated 但公开,2025.9 实测 3.8.8)。给走 LOCAL 的动图/音频类插件喂数据用 |

## 10. 自检清单(交给编辑器之前)

1. 场景 JSON `JSON.parse` 通过,所有 `__id__` 落在数组下标范围内(gen-scenes.mjs 末尾有现成校验代码)。
2. 场景里每个自定义 `__type__`(23 字符)都能反查到一个 `.ts.meta` 的 uuid。
3. 场景里每个 `__uuid__` 资源引用(如 `uuid@f9941`)都能反查到对应 meta 的主 uuid/子资产 id。
4. 每个 `cc.ClickEvent` 的 `(component, handler)` 都能在目标脚本或其基类源码里搜到方法定义(`verify-handlers.mjs`);component 名与 `@ccclass` 名一致。
5. 二进制资产(PNG)要用解码器验一遍(chunk CRC + IDAT inflate),拼字节错位是静默损坏。**合成的格式数据(APNG/Spine JSON+atlas 等)优先用「消费方的真实解码器」端到端验证**——插件带 wasm 的就在 Node 里 require glue + `WebAssembly.instantiate` 喂 instantiateWasm hook,解析→建实例→出渲染数据全链路跑一遍(参考 `tools/verify-spine-data.mjs`),比静态查格式快且更真。
6. 脚本类型检查:参考 `D:\projects\performance\tools\tsconfig.check.json` —— `files` 里带引擎 cc.d.ts + 全部业务脚本,`skipLibCheck: true`(引擎声明自身有历史遗留错误,必须跳过)、`experimentalDecorators: true`,然后 `tsc -p` 零错误再开编辑器。**cc.d.ts 别引用工程的 `temp/declarations/cc.d.ts`**(那是编辑器再生成的一行 reference 桩,路径随引擎配置变),直接指向实体文件:自定义引擎用 `<引擎>/bin/.declarations/cc.d.ts`(没有就 `npm run build:declaration` 生成),stock 编辑器用 `C:/ProgramData/cocos/editors/Creator/<版本>/resources/resources/3d/engine/bin/.declarations/cc.d.ts`。声明生成器有缺口(如 `_decorator.editable/override` 漏掉),在管线里加一个 `declare module 'cc'` 的模块增强 d.ts 补齐,别改引擎。
7. 打开编辑器后若场景脚本报 Missing Script,九成是 meta uuid 和场景 `__type__` 对不上,用 §4 的算法反查。
8. 正式构建(web/原生)时,所有要 `director.loadScene` 的场景必须加进构建面板的场景列表;编辑器内预览不受此限制。
9. 工程里新装了 extensions/ 插件(或插件更新),场景脚本要消费其挂载运行时前**必须重启编辑器**让扩展加载、asset-db 挂载生效;`temp/logs/project.log` 里能看到扩展加载日志。

## 11. 测试场景消费编辑器插件的挂载运行时

给「带运行时组件的插件」(extensions/<pkg>/runtime 经 asset-db.mount 只读挂载)写测试场景的完整套路(2025.9 在 3.8.8 + animated-image / spine-runtime 两个插件上实测):

1. **导入路径 = `db://<package.json 的 name>/<runtime 内相对路径去后缀>`**,如 `import { AnimatedImage } from 'db://animated-image/AnimatedImage'`、`import { Spine, loadSpineRuntime } from 'db://spine-runtime/index'`。db 名取包名,不是扩展目录名(两者一致是惯例不是保证)。
2. **tsc 校验管线里用 paths 映射兜住这些导入**:`"db://<pkg>/*": ["../extensions/<pkg>/runtime/*"]`。注意映射进来的是 .ts 源码,会被 tsc 完整检查——插件代码 import 的引擎符号(如自定义引擎的 `UIMesh`)必须在你选的 cc.d.ts 里存在(§10.6)。
3. **异步初始化的组件要排队**:插件运行时若先要加载 wasm(`loadSpineRuntime()`),而组件加载 API(`loadFromJson`)是同步的,内部 `spine()` 未就绪会直接 throw。用一条 Promise 链把所有 spawn 排到 ready 之后:`this.readyQueue = this.readyQueue.then(() => loadSpineRuntime()).then(...)`。
4. **自包含测试数据,零外部资源依赖**:
   - APNG:程序合成(单色全幅帧,acTL/fcTL/IDAT/fdAT 布局见 `tools/gen-scenes.mjs` 的 makeApng),base64 内嵌进生成的 .ts;运行时 `decodeBase64 → new BufferAsset() → _nativeAsset`(§9 表)喂给插件。
   - Spine:程序合成最小合法骨骼(spine-cpp 4.3 解析器逐字段对齐,详见 cocos-asset-formats skill)+ libgdx 文本 atlas;纹理运行时 `Texture2D.reset + uploadData` 按区块颜色画出来。
   - **大体量真实样本(几百 KB 以上)别 base64 内嵌 .ts**:源文件缓存在 tools/ 下(README 记录来源 URL 和重下命令),生成器校验后复制为 `assets/resources/<子目录>/*.bin`,meta 用 buffer 导入器(`ver "1.0.2" / importer "buffer" / files [".bin",".json"]`),运行时 `resources.load(path, BufferAsset)` 拿到的就是可直接喂组件 `clip` 的字节资产,和手塞 `_nativeAsset` 等价。场景/脚本不引用其 uuid,按路径加载,所以 uuid 变动无风险。实测:13 个动图样本(最大 8.3MB)首次即通。
   - 生成器统一放在 gen-scenes.mjs,数据文件头标「勿手改」;格式真实性用 §10.5 的真实解码器验证。APNG 的生成期校验配方:逐 chunk 核 CRC + 按 fcTL 边界把 IDAT/fdAT 连续段各自拼成完整 zlib 流逐段 inflate(fdAT 每段先去头 4 字节序号)+ acTL 帧数/IHDR 尺寸对预期表 —— 坏字节在生成期就炸,不用等运行时(实现在 gen-scenes.mjs 的 validateAnimSample)。


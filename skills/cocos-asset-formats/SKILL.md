---
name: cocos-asset-formats
description: Use when hand-generating, parsing, or debugging the on-disk DATA FORMATS of Cocos Creator (3.8.x) assets without the editor — the serialized container shared by .anim/.prefab/.scene ({"__id__"} array, @uniquelyReferenced inlining) and hand-written AnimationClip JSON (wrapMode Loop=2), Spine skeleton JSON and libgdx .atlas text (dual 3.8/4.2 runtimes, incompatible atlas key sets; vendored spine-cpp 4.3 parser rules), SpriteFrame trim geometry (y-up offset), SpriteAtlas flattened _serialize, plist two-generation keys, SpriteType.MESH availability, compressSettings preservation. Triggers on requests like "hand-write an .anim file", "generate spine skeleton.json", "write a .atlas file", "which atlas format does spine 3.8 use", "skeleton.spine version rejected", "SpriteFrame offset y direction", "why is library spriteFrames an array", "SpriteType.MESH doesn't render", "regenerate image meta safely", or any task that must synthesize or verify asset data files for a Creator project from CLI or scripts.
---

# Cocos 资源数据格式（序列化容器 / Spine / SpriteFrame）

从 cocos-creator-plugin skill 拆出的「引擎侧资源格式」知识。用途：插件烘焙、CLI 合成测试数据、解析 `library/` 产物时，**唯一权威是格式本身**。结论基于 **Creator 3.8.8** 实测；标「3.8 / 4.0 一致」的条目对照过 4.0.0-alpha.31 源码。

分界线（详见 cocos-creator-plugin skill §0）：这里全是**引擎侧**结论，换自定义引擎就可能失效。编辑器侧的事（导入器争抢、`.meta` 的 importer 复用、UUID 只能由 asset-db 分配）看 cocos-creator-plugin skill 的 §4。

## 1. Cocos 序列化容器格式与手写 `.anim`

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

## 2. Spine：两个版本、两种互不兼容的 atlas 格式

3.8.8 **内置双版本 spine 运行时**，默认 3.8，可切 4.2。哪个进包看工程配置：

```bash
grep -o '"spine-[0-9.]*"' <工程>/settings/v2/packages/engine.json | sort -u
# includeModules 里出现的那个才是构建进包的
```

`.atlas` 格式必须和运行时版本对上：3.8 世代是 `rotate/xy/size/orig/offset/index`，4.x 是 `bounds/offsets`。**对不上的症状是 region 静默丢失或错位，没有报错。**

判断内置的是哪一代，看引擎里 `cocos/spine/lib/spine-core.d.ts`：有 `setFrame(frameIndex, time, …)` / `rotateMix` / `AtlasRegion{rotate,offsetX,offsetY}` 就是 3.8 世代。**4.0.0-alpha.31 里这几个签名还在**，所以 4.x 默认仍是 3.8 世代的旧 atlas 格式。

引擎侧真正的判定依据是 `cc.config.json` 里那个 module 的 `intrinsicFlags`（`SPINE_3_8` / `SPINE_4_2`），工程侧是 `settings/v2/packages/engine.json` 里 `spine._option`。4.x 还多了一套 `spine-version-dynamic.ts` / `spine-instantiate-dynamic.ts`（运行时可选版本，靠 `moduleOverrides` 换文件）——**如果引擎是这种构建，光看 `SPINE_VERSION` 常量会得到错的答案**，那个文件里的 `export const SPINE_VERSION = '3.8'` 只是给固定版本构建用的占位。

顺带两条：spine 的 `.json` 要和 dragonbones 抢导入器（编辑器侧问题，见 cocos-creator-plugin skill §4.2）；`premultipliedAlpha` 勾错了表现为半透明边缘发暗。

## 3. 插件自带 spine-cpp 4.3 时，格式的唯一权威是 vendored 解析器

不走引擎内置 spine、自己 vendor 一份 spine-cpp 4.3 编进 wasm/JSB 的插件（spine-runtime 就是这样），JSON / atlas 格式**别拿引擎内置版本的直觉套**，逐字段对着插件里那份解析器源码（`native/third_party/spine-cpp/src/spine/SkeletonJson.cpp` / `Atlas.cpp`）核。2025.9 合成测试数据时核出来的硬事实：

- **版本是硬闸门**：`skeleton.spine` 字段必须以 `"4.3"` 开头（`SPINE_VERSION_STRING = "4.3"`），写 `"4.3.43"` 可以，`"4.2"` / `"3.8"` 直接报版本错。
- **skins 是数组**：`skins: [{name, attachments}]`，attachments 是**两层按名映射的嵌套对象** `{slotName: {entryName: attachment}}`。
- **animations 里 bones / slots 也是按名映射的对象**（不是数组）：`{animName: {bones: {boneName: {rotate: [{time, angle}], translate: [{time, x, y}]}}}}`。
- region attachment 最小字段：`type/name/path/x/y/width/height/rotation`（`path` 是 atlas region 的查找键，可以和 `name` 不同），`scaleX/scaleY/color` 可省。
- **atlas 两种格式都吃**：4.3 的 `Atlas.cpp` 同时认经典键值（`rotate/xy/size/orig/offset/index`，`rotate` 收 `false`/`true`/度数）和 4.2 的元组（`bounds: x, y, w, h` / `offsets: x, y`）。合成数据用经典键值最稳。页头和第一个 region 之间**不需要空行**；**空行是翻页信号**，一页结束时才空一行。
- UV = `xy + size` 除以页 `size`；`orig` 缺省等于打包尺寸。

实证：按以上规则程序合成的最小骨骼（17 骨 / 16 槽 / 2 动画）+ 8 region 文本 atlas，第一次就通过插件真实 wasm 的全链路校验（解析 → 动画 → 渲染几何；Node 里 require glue + `WebAssembly.instantiate` 喂 instantiateWasm hook 的 harness，样板 `D:\projects\performance\tools\verify-spine-data.mjs`）。

## 4. SpriteFrame / SpriteAtlas / Sprite 零散事实（3.8.8）

- **`SpriteType.MESH` 不可用**：`cocos/2d/components/sprite.ts` 里 `// MESH: 4` 和 `/* || this._type === SpriteType.MESH */` 都是注释掉的。想做紧贴轮廓的 mesh 裁剪得自己写渲染组件。**3.8 / 4.0 一致**，连行号都没动（83 / 385）——别指望升 4.x 会恢复。
- **SpriteFrame 的 trim 字段齐全**：`_rect` / `_offset` / `_originalSize` / `_rotated`（`cocos/2d/assets/sprite-frame.ts`），序列化出来是 `{__type__, content:{rect, offset, originalSize, rotated, capInsets, vertices, …}}`。逐帧裁剪 + 还原位置用这套原生机制就够，不用自己算顶点。**3.8 / 4.0 一致**。
- **`SpriteFrame.offset` 是 y 向上的**（裁剪区中心 − 原始帧中心），而 `rect` 是图像坐标系（原点左上）。**同一个文件里混着两种 y 方向**，换算集中到一个函数里，别在多处重算。符号约定看 `_calcTrimmedBorder()`，**3.8 / 4.0 逐行相同**。验它最省事的办法是拿一段**上下移动**的素材烘一遍，看上半的帧 `offset.y` 是不是正的——纯读代码很容易把符号读反。
- **`cc.SpriteAtlas` 的序列化形状会让人误判**：字段是 `spriteFrames`（**无下划线**）、类型是字典，但它有自定义 `_serialize()` 把字典**摊平成交错数组** `[key0, uuid0, key1, uuid1, …]`。所以在 `library/*.json` 里看到 `content.spriteFrames` 是数组是正常的，不是版本差异（`2d/assets/sprite-atlas.ts:54,113-149`，**3.8 / 4.0 逐行相同**）。
- **TexturePacker `.plist` 图集两代键名同时写出最稳**：format 2 是 `frame`/`offset`/`sourceSize`，format 3 是 `textureRect`/`spriteOffset`/`spriteSize`/`spriteSourceSize`/`textureRotated`，两套都写代价只有几百字节（3.8.8 的 sprite-atlas 导入器都认）。
- **不要凭猜写 `compressSettings`**：它在 Cocos 自己的类型里是 `Record<string, any>` 且带 `TODO`，预设 id 还是项目本地的，猜着写会静默失效。重新生成资源时**完整保留**上一份 `userData` 和每个 `subMetas[*].userData`，第一次导入时提示用户自己设。

---
name: wechat-minigame-worker
description: Use when writing, debugging, or gating WeChat minigame (微信小游戏) multi-thread Worker code — wx.createWorker setup, game.json "workers" config and worker subpackages, worker.onMessage/postMessage copy semantics, old-version limits (1 worker max, no wx APIs, JS-only dir), and the gray-release Worker V2 with OffscreenCanvas transfer, SharedArrayBuffer detection and task-slot patterns, wx APIs via the worker object (innerAudio/webSocket/tcp/udp), and multi-worker support. Includes the real-device finding (2026-09) that SAB/Atomics globals existing on both sides does NOT mean the postMessage channel preserves SAB (bidirectional JSON husk — 全局在 ≠ 通道在). Triggers on requests like "wx.createWorker", "move decoding off the main thread", "OffscreenCanvas in a worker", "SharedArrayBuffer in WeChat", "isSupportStandardWorker", "新版 worker / worker V2", "worker 分包 preDownloadSubpackage", or any Cocos-minigame task that wants to offload computation, decoding, audio, or networking into a worker. Also covers the injected perf-driver matrix pipeline on real devices (§10): XMLHttpRequest arrives with the async web-adapter — probing before it lands throws and silently strands the driver; stale perfQuery zombie runs; preview QR ~25min expiry; Windows hotspot topology to reach a LAN collector without repacking. And the real-device wasm instantiation iron laws (§11): main context has NO WebAssembly global, WXWebAssembly.instantiate only accepts in-package paths (bytes and USER_DATA_PATH files all fail with the same misleading "only support file type" error), and the worker-side WebAssembly polyfill exposes its props as reporting getters that crash on a missing Reporter global — even `typeof WebAssembly.instantiate` throws; fix by injecting a recursive no-op Proxy Reporter stub.
---

# 微信小游戏多线程 Worker（旧版 + V2 灰度）

来源：官方文档 <https://developers.weixin.qq.com/minigame/dev/guide/base-ability/workers.html> 与 <https://developers.weixin.qq.com/minigame/dev/guide/base-ability/new-worker.html>（2026.9 读取）。V2 仍在灰度，细节以官方文档为准。

背景：小游戏每帧 16ms 极其宝贵，可异步处理的任务放 Worker 跑，结束后把结果传回主线程。Worker 跑在**单独的全局上下文与线程**里，不能直接调主线程方法，也不具备渲染能力（V2 的 OffscreenCanvas 例外，见 §2.2）。主线程与 Worker 之间用 `Worker.postMessage()` 发、`Worker.onMessage()` 收，**传输的数据是复制的，不是共享**（V2 的 SharedArrayBuffer 例外，见 §2.3）。

## 1. 旧版 Worker（稳定，所有环境可用）

### 1.1 配置 game.json

`game.json` 里配 Worker 代码目录，目录下所有 JS 会**打包成一个 JS 文件**：

```json
{ "workers": "workers" }
```

这样配置时 worker 代码进**首包**。小游戏首包上限 4M，为了不占首包体积，基础库 **v2.27.3** 起支持把 worker 代码打包为**分包**（需最新 nightly 开发者工具）：

```json
{
  "workers": {
    "path": "workers",
    "isSubpackage": true
  }
}
```

### 1.2 目录结构与代码

```
├── game.js
├── game.json
├── project.config.json
└── workers
    ├── request
    │   ├── index.js
    │   └── utils.js
    └── response
        └── index.js
```

worker 线程执行上下文**全局暴露一个 `worker` 对象**，直接 `worker.onMessage` / `worker.postMessage`：

```js
// workers/request/index.js
var utils = require('./utils')

worker.onMessage(function (res) {
  console.log(res)
})
```

### 1.3 主线程创建与通信

```js
// game.js（主线程）
var worker = wx.createWorker('workers/request/index.js') // 入口文件完整路径（相对小游戏根）
worker.postMessage({ msg: 'hello worker' })
```

入口文件由 `wx.createWorker()` 动态指定，可随时换。

### 1.4 worker 分包要先下载再创建

worker 配成了分包时，必须先 `wx.preDownloadSubpackage` 下载完再 `createWorker`，否则报错：

```js
var task = wx.preDownloadSubpackage({
   packageType: "workers",
   success(res) {
      var worker = wx.createWorker("workers/request/index.js") // 分包没下载完就调 createWorker 会报错
   },
   fail(res) { console.log("load worker fail", res) }
})

task.onProgressUpdate(res => {
  console.log(res.progress)                  // 下载进度
  console.log(res.totalBytesWritten)
  console.log(res.totalBytesExpectedToWrite)
})
```

### 1.5 旧版约束清单（官方 Tips）

1. **同时最多 1 个 worker**，创建下一个前先 `Worker.terminate()` 结束当前的。
2. worker 内只能 **require worker 目录内**的文件。
3. worker 内**不支持 `wx` 系列 API**。
4. Workers 之间**不支持互相发消息**。
5. **worker 目录内只能放 JS 文件**，其他静态文件放目录外。
6. 基础库 **v2.18.1** 起支持**插件内**使用 worker，需在 `plugin.json` 里配 workers 代码路径（相对插件代码包根目录）。

其中 2、5 以及「postMessage 是拷贝」在 V2 里同样成立。

## 2. 新版 Worker V2（灰度中）

### 2.1 环境门槛与探测

| 项 | 要求 |
|---|---|
| Android 客户端 | ≥ 8.0.66 |
| 基础库 | ≥ 3.16.1 |
| iOS | 能力支持中（未全量） |
| 开发者工具 | **暂不支持** |

灰度中，使用前还要确认灰度开通状态（官方文档「建联 / 灰度申请」）。运行时探测：

```js
wx.env.isSupportStandardWorker
```

基础使用步骤同旧版（§1）。新增能力：`OffscreenCanvas`、`SharedArrayBuffer`、worker 内调 innerAudio / webSocket / tcp / udp、**多 worker**。

### 2.2 OffscreenCanvas（渲染逻辑搬进 worker）

从主线程已有 Canvas 通过 `transferControlToOffscreen()` 转移，再 postMessage **transfer** 给 worker：

```js
// 主线程
const canvas = wx.createCanvas();
const offscreenCanvas = canvas.transferControlToOffscreen();
const worker = wx.createWorker('workers/render-worker/index.js');

worker.postMessage({
  type: 'init-canvas',
  canvas: offscreenCanvas,
}, [offscreenCanvas]);   // ← 第二参是 transfer list
```

```js
// workers/render-worker/index.js
let canvas;
let ctx;

let renderLoop = (time) => {
  const x = (Math.sin(time / 500) * 0.5 + 0.5) * canvas.width;
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.fillStyle = '#07c160';
  ctx.beginPath();
  ctx.arc(x, canvas.height / 2, 32, 0, Math.PI * 2);
  ctx.fill();
  requestAnimationFrame(renderLoop);
}

worker.onMessage((res) => {
  if (res.type === 'init-canvas') {
    canvas = res.canvas;
    ctx = canvas.getContext('2d');
    requestAnimationFrame(renderLoop);
  }
});
```

- **Canvas 控制权 transfer 给 worker 后，主线程不得再直接操作该 Canvas 的绘制上下文。**
- worker 里用自己的 `requestAnimationFrame` 循环驱动渲染。

### 2.3 SharedArrayBuffer（高频 / 大数据量通信）

主线程和 worker 共享同一段内存，免拷贝。**先验证通道再启用**——2026-09 真机实测（§5）：SAB 全局在主线程与 worker 两端都存在，但 postMessage 通道**双向 JSON 化**，SAB 过线即变无 `byteLength` 的空壳（连字节内容都丢）。全局存在 ≠ 通道可用。**判定 SAB 不要用 `instanceof`**（跨 realm 会失败）：

```js
function isSharedArrayBuffer(buffer) {
  return buffer?.constructor?.name === 'SharedArrayBuffer';
}
```

**「SAB 不可用」有两种形态，降级链要分开判**（2026-09 实证）：

- **微信真机**：两端全局都在，但通道 JSON 化把 SAB 序列化成无 `byteLength` 的空壳（连字节内容都丢）→ 靠形态校验（`byteLength` 是正数）+ 写读探针（token 写 header、worker 回读同一位置）发现，失败要响亮 warn 后降级 copy。
- **Chrome 152（2026-09）**：非跨域隔离页面 `typeof SharedArrayBuffer === 'undefined'` —— 构造器**整个缺失**，不是「有但不可用」。降级从第一次 open 起就是 copy，**没有探针失败、也就没有 warn**；判定「sab 档 ≡ copy 档」要靠传输字节计数（copy 档 >0），日志探针不响。

顺带：`SharedArrayBuffer` 是 **ES2017 语言内建**，Node `vm.createContext` 造的新 realm 自动带它 —— 沙箱里想模拟「无 SAB 宿主」必须显式 `sandbox.SharedArrayBuffer = undefined`，否则降级分支永远测不到。

部分接口只收普通 `ArrayBuffer`，传 SAB 前先拷贝：

```js
if (heap.buffer.constructor.name === 'SharedArrayBuffer') {
  return new Uint8Array(heap.subarray(offset, offset + length)).buffer; // sab 需要先拷贝
}
return heap.buffer.slice(offset, offset + length); // 非 sab 直接用 slice
```

官方最小**任务槽**范式——主线程写任务、worker 轮询消费、无需回传：

```js
// 主线程
const worker = wx.createWorker('workers/logic-worker/index.js');
const state = new SharedArrayBuffer(Int32Array.BYTES_PER_ELEMENT * 2);
const view = new Int32Array(state); // [hasTask, taskId]

worker.postMessage({ type: 'task-state', state });   // SAB 随消息共享，无需 transfer

function sendTask(taskId) {
  if (Atomics.load(view, 0) === 1) return; // 上一个任务还未被消费
  Atomics.store(view, 1, taskId);
  Atomics.store(view, 0, 1);
}

sendTask(1001);
```

```js
// workers/logic-worker/index.js
function consumeTask(taskId) {
  console.log('consume task:', taskId);
  // 在这里执行耗时逻辑
}

worker.onMessage((res) => {
  if (res.type !== 'task-state') return;

  const view = new Int32Array(res.state);
  setInterval(() => {
    if (Atomics.compareExchange(view, 0, 1, 0) !== 1) return;
    const taskId = Atomics.load(view, 1);
    consumeTask(taskId);
  }, 16);
});
```

### 2.4 worker 内调 wx 系列 API

V2 支持部分 wx API（innerAudio、webSocket、tcp、udp 等），但调用走**全局 `worker` 对象**，**不是 `wx`**：

```js
// 主线程
const worker = wx.createWorker('workers/audio-worker/index.js');
worker.postMessage({ type: 'play-audio', src: 'audio/click.mp3' });
```

```js
// workers/audio-worker/index.js
const audio = worker.createInnerAudioContext({
  useWebAudioImplement: true,
});

audio.onError((res) => {
  console.error('audio error:', res.errMsg);
});

worker.onMessage((res) => {
  if (res.type !== 'play-audio') return;
  audio.src = res.src;
  audio.stop();
  audio.play();
});
```

### 2.5 V2 仍然成立的限制（官方 Tips）

1. worker 内只能 require worker 目录内的文件。
2. 入口文件由 `wx.createWorker()` 动态指定。
3. **worker 目录内只支持放 JS 文件**，其他静态文件放目录外。

## 3. 与 Cocos 插件 / wasm 的关系

给 wasm 解码器（或任何 `.wasm` 载荷）做「搬进 worker」的设想，先过这张约束表：

| 约束 | 出处 | 对 wasm 的影响 |
|---|---|---|
| worker 目录只能放 JS | 新旧版 Tips 都有 | `.wasm` 文件必须放目录外，worker 里只能靠路径 / 消息拿字节 |
| `CCWebAssembly.instantiate` 只吃路径、不吃字节 | Cocos `pal/wasm`（见 cocos-creator-plugin skill §8.3） | base64 内嵌 / `SINGLE_FILE` 方案在微信跑不起来，独立 `.wasm` + `instantiateWasm` hook 是硬要求 |
| V2 灰度门槛（Android ≥8.0.66、基础库 ≥3.16.1、iOS 支持中、**开发者工具不支持**） | §2.1 | worker 化只能做渐进增强：先 `wx.env.isSupportStandardWorker` 探测，主线程路径必须完整保留 |
| 旧版（及未灰度到的环境）同时最多 1 个 worker | §1.5 | 「解码 worker + 音频 worker + 网络 worker」并存需要 V2，且要过灰度 |

Cocos 侧 wasm 的四宿主加载表（BUILD / 小游戏 / EDITOR / PREVIEW 各去哪找文件）在 **cocos-creator-plugin** skill §8.3。

## 4. devtools 模拟器实测（2026-09-14，devtools 2.01.2510290 / 基础库 3.17.2）

做 worker/解码 A/B 压测时模拟器环境的实证坑，全部踩过验证：

1. **模拟器 game 上下文自带 `location`，search 恒可能非空**（编译模式残留 query 就落在
   `location.search`，不是 `getLaunchOptionsSync().query`）。query 读取链若在
   location.search 命中时**提前 return**，后面的 storage / 全局对象注入源永远读不到——
   现象是"场景跑了、FPS 正常，但采样/ramp 全不启动"。正确写法：各源依次**合并且
   先到先得**（后源不覆盖已有键），别提前 return。
2. **automator 的 App 域在 compileType:game 下无响应者**：`Tool.getInfo` 秒回正常，
   `App.callFunction` 恒超时（含"游戏先跑、首次连接、首个 App 请求"的干净态）。
   外部注入路线不可用 → 改**游戏内自驱脚本**：注入进 build 产物入口（game.js 末尾），
   每轮 `wx.setStorageSync('perfQuery', {...})` 写参数 → 调场景暴露的
   `globalThis.__perfGotoScene` 切场景 → 轮询收集器 `GET /state`（看最近上报的
   variant+时间戳）判断本轮完成再推进。零协议依赖，模拟器重启后自动从头续跑。
3. **同场景 `director.loadScene` 是 no-op**（不重跑 onLoad）：自动跳转链
   `storage.scene → MainScene.onLoad 读 query 跳转`在已在菜单时会死在起点。
   两段式直跳：先 `go('main')`（no-op 无妨），1.5s 后 `go('目标场景')`——
   目标场景的 onLoad 每次加载都重新读参数，不依赖菜单中转。
4. **保存 build 目录内任意文件 → devtools 自动重编译 + 模拟器重载**（`Tool.compile`
   在该版本未实现的替代编译触发器；延迟秒级到 ~1 分钟）。注入代码改完存盘即生效。
   注意每次重载都会重启自驱矩阵，编辑要攒批做。
5. **渲染进程内存限 1024M**（设置-通用-内存限制），超限自动终止弹"是否重启"。
   每实例全帧缓存的负载（如 heavy APNG ~12MB/实例）很快顶爆，且**跨轮不释放会累积**
   ——表现为"每 ~3 轮周期性神秘重载"。对策：轮间 `wx.triggerGC()`（小游戏侧唯一
   显式 GC 入口）+ 实例数控在该机器限值内；重启后自驱脚本自动续跑，被腰斩的轮
   只是没有上报，不产生脏数据。
6. **模拟器 worker 传输开销 ≠ 真机/浏览器**：postMessage 回传随实例数急剧饱和
   （45 真实实例 postMsAvg ~6.7ms → 95 实例 ~70ms，x115 时 worker P95 65ms vs
   builtin 18ms）；而桌面 Chrome 同引擎同负载 worker 反超（builtin P95 -22.8%）。
   模拟器结论只能作管线验证 + 传输开销佐证，**不能外推真机**，报告中按环境分组呈现。
7. **压测协议两个正交约束**（详见 performance 仓库 tools/PERF.md）：a) 各 variant
   上报时实例数必须相同（异步资源载入延迟随路径不同，worker 开局拉 worker 会拖慢
   resources.load → ramp 批次右移 → 实例数不等，负载不可比）；b) 解码负载要落进
   采样窗（warmup>0 会把从不淘汰的帧缓存全部填满，窗口内变纯缓存回放，无 A/B 信号）。
   同时满足：warmup=0 + measure 拉长到盖住最慢 variant 的全部 ramp 批次。

## 5. 真机实测：SAB 通道双向 JSON 化（2026-09-14，Android 16 / 微信 8.0.77 / 基础库 3.17.3）

V2 灰度「半开」状态的决定性实证——**运行时给了，通道没给**：

1. **两端运行时全都在**：主线程与 worker 内 `SharedArrayBuffer` 均为 function、
   `Atomics` 均为 object，worker 内 `new SharedArrayBuffer(16)` 分配成功。
2. **postMessage 通道双向 JSON 化**：worker→main（生产协议方向）与 main→worker
   （协议倒置方向）都实测——SAB 过线即变成**无 `byteLength` 的空壳对象**（SAB 无
   可枚举属性，JSON 序列化结果就是 `{}`，字节内容全丢，连克隆拷贝都不是）。写读
   探针双向全不可见。**协议倒置救不了：通道对称地不支持**。
3. **官方文档措辞**：`Worker.postMessage` 页只说「基础库 2.20.2 之后支持传递*任意
   类型*的数据」——「传递」指序列化不报错，**不是结构化克隆/共享**；整个 Worker
   API 文档树（含 `onProcessKilled`、`getCameraFrameData` 等多进程 V2 API）无一
   处提到 SharedArrayBuffer 或 transfer list（OffscreenCanvas 的 transfer 除外）。
4. **SAB 可用性判定不能止于全局探测**：`typeof SharedArrayBuffer === 'function'`
   在该真机恒真但通道不可用。正确判定 = 过线后验形态（`typeof sab.byteLength ===
   'number'`）＋ 写读探针（token 写入、对端回读）两道兜底，任一失败即降级 copy。
   Cocos 扩展 `worker-decoder.ts` 的两道降级在真机被证实**必要且正确触发**。
5. **真机探针工程要点**：启动早期（<~2.5s）XHR POST 静默丢失，探针统一延迟 ≥3s
   发送，`console.log` 兜底（vConsole 日志面板可读；adb logcat 不镜像 JS console）；
   devtools 模拟器探不了通道问题——3.17.2 实测其 SAB 全局**不对称**（主线程
   `undefined`、worker 侧 function 且能分配），通道同样 JSON 化，worker-sab 自然
   回退 copy。**通道行为必须真机实测**。
6. **结论的适用范围**：官方 V2 文档明确「新版 Worker 仍在灰度中，使用前需确认
   基础库、客户端版本**和灰度开通状态**（建联/灰度申请）」。上述实测设备**未做
   灰度申请**，故现象应解读为「**未灰度环境的默认行为**」：SAB/Atomics 全局是
   运行时层面的部分放量，结构化克隆通道属于 V2 能力、整体在灰度。已开通灰度的
   环境通道是否保 SAB **未测**——不能下「微信全平台 SAB 不可用」的结论，只能下
   「默认环境（含最新 3.17.3/8.0.77）不可用」。运行时官方探测口：
   `wx.env.isSupportStandardWorker`。工程含义：SAB 路径必须做成「过线形态校验 +
   写读探针通过才启用，否则降级 copy」的渐进增强，灰度开了自动受益、没开自动兜底。

## 6. 真机实测：decode 合批（K=8）是负优化（2026-09-14，同机同窗 A/B）

动机：copy 通道的 postMs ≈ 消息条数 × 每消息固定开销 + 字节数 × 单位成本，
真机 postMsAvg 数十 ms、桌面 Chrome 仅 ~4ms。假设「每消息固定开销占大头，
把逐帧 decode 合成一条消息（K 帧一批）能把第一项摊薄 1/K」——协议、摊销记账
（每帧仍一条 emitSample，roundtrip=批往返÷K）、沙箱验证全链路做齐后真机实测
（65 个动图实例 × builtin/worker/worker-batch 各 5 轮，同预览包同时间窗）。

1. **结果 5/5 轮方向一致地变差**：帧时间 P50 不变（16ms）但 P95 +70%、
   P99 +150%（46/85ms vs 逐帧 27/34）、1% low FPS -59%、掉帧 +12%。摊销口径
   postMsAvg **不降反升**（32.0 → 35.6），解码采样数 -25%（播放被拖慢）。
2. **机制：多实例共享单 worker 时合批把响应粒度变粗**。一条批消息要串行解完
   K 帧才回，期间其余 64 个实例的请求全部排队——「省 N-1 条消息的固定开销」
   被排队时间吃掉还倒贴。批的收益模型（固定开销/K）只在**单消费者、请求稀疏**
   时成立；共享 worker 高并发下头阻塞反而主导。**合批前先画清排队模型：你的
   worker 有几个并发消费者？**
3. **摊销记账有陷阱**：postMsAvg 摊到每帧后数值「看起来可比」，但批模式下它
   不再对应任何真实单帧等待（真实等待是锯齿：每 K 帧一个长峰，帧预算 16.7ms
   下长峰必超 → P95/P99 崩而 P50 无感）。评估批处理**只看帧时间分布**，别信
   摊销均值。要直读真实消息条数，得把批字段透传进上报 schema，别靠行为差异反推。
4. **结论修正为**：copy 通道的两根杠杆（消息条数 / 字节数），「砍消息条数」
   这根在共享 worker 高并发下**真机证伪**；SAB（两根一起砍、等待归零）仍是
   唯一没被证伪的传输优化路径——SAB 解锁前提的论点反而被强化。
5. **同窗对照 vs 跨窗对照**：跨时间分离的两轮矩阵存在会话间漂移（同设备同
   负载，首轮 postMsAvg 54~88ms、当晚 30~33ms——机器状态本身在漂，首轮得出
   「worker P95 +33%」、同窗重测实为持平、收益集中在 1%low +17.6% p=0.004）。
   **A/B 结论只认同一时间窗内轮转的对照**。
6. **devtools CLI 出带参预览码的传参坑**（Windows）：`--compile-condition` 的
   JSON 从 PowerShell 5.1 传给 `cli.bat` 时内嵌双引号会被吞（devtools 报
   `Unexpected token q in JSON at position 1`）——写成 `'{\"query\":\"...\"}'`
   （单引号包裹、内层引号 `\"` 转义）可过；Git Bash 直调带空格的 cli.bat 路径
   会被 cmd 截断，别走。出码后立刻杀 devtools 进程（模拟器会自跑矩阵污染
   收集器），二维码用 `--qr-format image --qr-output` 落盘再给用户扫
   （preview 码几分钟即过期，复测须重新生成）。
7. **Node vm 沙箱验证 worker bundle 的两个还原点**（不开编辑器/浏览器跑通
   open→decode→decode-batch→close 全协议）：(a) **跨 realm 的 `instanceof`
   恒 false**——bundle 里 `value instanceof Uint8Array` 识别载荷时，沙箱外
   Node 传进去的数组永远不匹配（真实宿主无此问题：structured clone 在 worker
   自己的 realm 重建数组）。修法：把 Node 的 `Uint8Array`/`ArrayBuffer`/
   `SharedArrayBuffer` 构造器**注入沙箱全局**（同对象引用，顺带才能验 SAB
   「真共享」）；断言侧也同理，跨 realm 判类型用 `typeof`/长度，别用
   `instanceof`。(b) 载荷要按**宿主真实线格式**发：微信宿主发 b64 字符串
   （模拟 transfers=false 的 JSON 拷贝序列化），浏览器宿主直传 typed array。
   线格式模拟错 = 测的不是生产路径。

## 7. 真机实测：正式号 appid 的 V2 四项权限探针（2026-09-15，同机）

背景：§5 的实测跑在 Cocos 默认占位 appid 上；正式申请的小游戏 appid
（配置进 builder profile `packages.wechatgame.appid` + build 后的
`project.config.json`）用同一台真机（小米 2304FPN6DC / Android 16 /
微信 8.0.77 / 基础库 3.17.3 / benchmark 38）做了完整四项探针，
**三轮结果完全一致**（含手机开「开发调试」后冷启动的一轮——
**开发调试开关不改变任何判定**）。复用探针：
`tools/inject-v2-probe.mjs`（主线程块注入 `game.js` 末尾，幂等）+
`tools/v2-probe.js`（worker 侧，拷入 `workers/`），结果 POST collector。

1. **判定：未灰度**。`wx.env.isSupportStandardWorker === false`，且设备端
   门槛全达标（8.0.77 ≥ 8.0.66、3.17.3 ≥ 3.16.1）——挡的只有灰度名单，
   出路是官方「建联 / 灰度申请」。
2. **未灰度真机的四项能力形态**：
   - SAB：主线程与 worker 全局都在（function），通道**双向** JSON 空壳
     （与 §5 一致，「全局在 ≠ 通道在」再证）。
   - OffscreenCanvas：主线程 `canvas.transferControlToOffscreen` **存在**，
     但 `postMessage(msg, [transfer])` 抛
     `Error: transfer is not supported in this version`（与 devtools 同文案）
     ——API 在、通道不在。
   - 多 worker：第二个 `wx.createWorker` **返回 undefined 而不抛异常**
     （真机旧版限 1 的形态）。**devtools 返回的是对象**——又一处工具≠真机；
     代码要按「可能返回 undefined / 无方法对象」防御，`typeof w.onMessage`
     前先判 `w` 本身非空（对 undefined 取属性直接 TypeError）。
   - worker 沙箱：无 `OffscreenCanvas` 全局、无 `requestAnimationFrame`、
     无 `performance`（这三个是 V2 worker 才有；旧版 worker 内计时用
     `Date.now()`，别依赖 rAF）。
3. **接线形态要双形态兼容**：未灰度环境 `createWorker` 返回旧版对象
   （`onMessage(fn)` 注册 / `postMessage` 收裸对象）；灰度开通后可能返回
   标准 Worker 形态（`onmessage =` / `addEventListener('message')` /
   MessageEvent 包装载荷在 `.data`）。注册顺序：旧版 `onMessage(fn)` →
   `addEventListener('message')` → `onmessage =`；收包同时解
   `{cmd:...}` 裸对象与 `ev.data.cmd` 两种线格式——否则灰度真开了会被
   误判成「形态异常」。
4. **探针步骤机教训**：每个 `.onMessage` / `.postMessage` 调用点都要
   形状校验 + try/catch 响亮报错后**继续前进**，一处未兜异常 = 整台静默死、
   只剩全局超时（本次 devtools 上连环踩两次：w1 无方法对象、w2 undefined）。

## 8. 浏览器宿主实证：OffscreenCanvas overlay 把「省传输」变「消灭传输」（2026-09-15，Chrome 152 桌面）

微信 V2 未灰度（§7）期间，同一个 animated-image worker 解码器在**浏览器宿主**
（Cocos 预览 7456 + Chrome 非 headless）先落地了 OffscreenCanvas 终极形态：
worker 不只解码，还直接把帧画到一块「就是最终显示面」的 OffscreenCanvas——
一块 DOM canvas 盖在引擎画布正上方（`position:absolute` +
`pointer-events:none`，插 `game.canvas` 之后做兄弟节点），
`transferControlToOffscreen()` 后 transfer 过线给 worker。动图**不进引擎渲染**
（实例无 Sprite 无纹理无 update），主线程每帧成本归零。

### 8.1 架构要点（全部可在 D:\projects\performance 复查）

- **同一个 decoder worker 兼职合成器**（协议纯加法：`overlay-canvas/attach/resize/
  update/set-state/detach` + worker→main 的 `overlay-stats` 通知）；worker 自驱
  rAF 时钟（无 rAF 宿主落 setTimeout 1000/60 兜底）、按需解码、脏检测整帧重画。
- **worker 侧 compositor 零顶层副作用**——未收 `overlay-canvas` 前一行不执行，
  这是「新增合成器不砸现有 copy/SAB/batch 档」的核心保险（bundle 是同一个）。
- **主线程 OverlayManager 单例**：idle→init（Promise 去重；transferControlToOffscreen
  一次性，重复必抛）→ready/broken（会话级缓存永不重试）。能力门：
  `document` + `HTMLCanvasElement.prototype.transferControlToOffscreen` +
  worker 可用。投影：沿父链找 `cc.Canvas.cameraComponent`，对角两点
  `worldToScreen`（左下原点、单位=渲染窗口物理像素），`yTop = overlayH - max(y)`
  翻 Y；resize 经 `screen.on('window-resize')` 延后一帧读新值。
- **降级自愈**：宿主门不过 / 会话 broken / attach 失败 → 组件原地
  `addComponent(AnimatedImage)`（requireComponent 自动补 Sprite）同参重建后自毁。
  微信宿主（transfer 列表被拒，§7）必然整组走这条路 = 天然的「降级路径 =
  worker copy 回归」证据，不是 offscreen 证据。
- **单槽 onMessage 约束**：包装层 `worker.onmessage =` 已被 WorkerDecodeService
  占用时，新通知走 service 的 id 泵分流——id 是 number 进请求/响应表，
  无 id 且 `t` 以 `overlay-` 开头的转发给 `onNotification` 回调。

### 8.2 实测数字（x20 实例，real-chrome，warmup=0，每档 5-7 轮）

- **postMsAvg = 0.000 是设计值**（帧样本 roundtrip==compute、bytes=0，消灭了
  每帧传输本身）；实测 0~0.5ms 残差全部来自 **open 样本**（每实例一条，
  bytes=源文件体积，warmup=0 时落在测量窗内，20 实例合计 ≈0.03MiB）。
  worker 内 taskMs 2.1~2.6ms = 解码+ImageData→createImageBitmap+composite 摊销。
- **mainThread.logicMsAvg = 0.00ms**：主线程零成本的直接举证（frame 段不能归零
  举证——浏览器 DOM 合成器仍有节奏，frame ≈ 空引擎场景 + 合成器）。
- 帧指标 vs builtin：P95 持平（轻载全档被 vsync 钉在 16.7ms），P99 -3.9%
  （p=0.002）；1% low +2%。**跨档比较看 frame P95 / 1% low / mainThread.logic**，
  taskMs 两侧口径不同（copy 档仅解码）绝对值不可直比。
- **回退识别单门：`postMsAvg > 1.0`**（2026-09-15 两轮实证收敛）：轻载健康残差
  0.1~0.5ms、heavy 大图池 open 残差 ~0.3ms、整组回退 copy ≈3.3ms,阈值 1.0 干净
  分开。**postBytesTotal 不能当门**：open 字节 = 源文件体积 × 实例数，随池子缩放
  （heavy 池 80 实例 ≈280MiB 全健康，双门规则在重载下必误报）。
- **重载首屏期 taskMsAvg 是墙钟重叠口径**：compositor 的 ensureFrame 各条目并发
  解码、每条样本含自己的排队墙钟，ΣtaskMs 可超单线程容量 ~8×；位图全缓存填完后
  decode 归零只余 composite。跨档比较看 frame 段与稳态段，别拿首屏 taskMs 记账。
- 适用判据（复用价值）：解码+绘制占主线程大头 / 产出即像素 / 层可整体独立。
  spine（产出是顶点）与「整屏进 worker」（引擎锁屏 canvas）均不适用。
- **heavy 负载下的尾部形态（2026-09-15，x80 heavy×5 轮）**：P95 与 copy 档持平
  （16.9）但 P99 3/5 轮 ≈23ms（copy 17.1）—— 合成器与按需解码共享同一 worker
  线程，偶发长 tick；非回退（零降级警告、postMs 0.1ms）。汇报 offscreen 档别只
  报 P95，尾部要如实分开。

### 8.3 跑批基建教训（CDP 驱动 Chrome 性能矩阵）

- **同一 Chrome 进程连跑 ~11 个全量预览页后会 wedge**：后续页面在引擎 boot
  中途挂起（readyState=complete、资源全秒回、`Debugger.pause` 都不落地 =
  renderer native 层卡死，非 JS 死循环），~20 分钟自愈。**对策：逐轮新起
  Chrome 进程**（新临时 profile + 每轮 kill），+2s/轮，比挂起重跑快得多。
- kill 后**等进程 `exit` 事件再还端口**，别用 fetch 探测端口释放——负载下
  fetch 500ms 超时被误判「已释放」，下一轮 Chrome 绑不上调试端口直接退出。
- CDP `Runtime.evaluate` 取值要两层：`(await send(...)).result?.result?.value`
  （响应嵌套 `{id, result:{result:{value}}}`）；console 走
  `Runtime.consoleAPICalled` / `Runtime.exceptionThrown` 事件流。
- 起会话失败要**轮级容错**（重试一次、记 failure 继续），别让一次 spawn
  抖动炸掉整个 20 轮矩阵。
- 流程纪律：任何 `--runs N`(N>1) 大矩阵前，先 `--runs 1` 全变体确认端到端
  绿（数据可直接计入），再补余轮——矩阵挂在第 10 轮比挂在第 1 轮贵 10 倍。

## 9. 真机实测：灰度开通后的 SAB 形态与并发通道 bug（2026-09-16，同机正式号灰度）

§7 判定未灰度 → 建联灰度开通 → 本节为**开通后**的实测形态与规避。反馈文档
`performance 仓库 tools/wx-worker-v2-feedback.md`（拿去找微信的完整材料）。

1. **灰度开通后的 SAB 形态（判定规则）**：worker 侧**无** `SharedArrayBuffer`
   构造器、主线程有、通道真共享（主→worker 方向，写读互见）。即 SAB 只能
   「主线程 `new` 后 postMessage 回 worker 绑定」——协议：open(shared) 回
   `sharedRequestBytes`（建议字节数）→ 主线程 new → `attach-shared` 送回 →
   worker 形态校验（byteLength 严格比对）+ 主线程多点写读探针。两端
   `typeof SharedArrayBuffer` 答案相反，能力探测必须两端分别做。
2. **并发通道 bug（正确性级）**：大字符串消息（8~11MB b64 open）与 SAB 克隆
   消息**并发交织**时：①丢 open 消息——`worker 请求 open（handle=N）10000ms
   无响应`，完成率 3/38（轻载 7/40），且平台零错误零回调（请求方永久挂起）；
   ②概率性数据串台——SAB 像素区实际接着别的消息的 b64 字节，渲染出「文字
   纹理」，实例随后停帧。**证据链**：同规模 copy 档（x65，同样大 b64 并发、
   无 SAB 消息）全部通过 → 大消息本身不丢，SAB 克隆交织是触发因子；已 attach
   的 SAB 多点探针/每帧指纹零失败 → bug 在消息层，不在共享内存读写。
3. **规避（两层叠加，已实测恢复）**：
   - **open 协商全局串行**（默认开，`__sabSerialOpen=false` 可关回对照）：整个
     open→attach→probe 协商块排全局单队列，一次一条在途，交织窗口构造上消灭；
   - **源字节去重**：主线程 FNV-1a 内容指纹（`Math.imul`！普通 `*` 在 2^53
     以上丢精度）作 sourceKey，worker 侧缓存字节（FIFO 上限 64），同源 open 只
     发 key + 空串，未命中回 `sourceMiss` 主线程自动全量重发——x38 heavy 的
     大消息从 38 条降到 4 条，且免掉重复的 8MB b64 编码。
4. **请求-响应必须有超时**：`_request` 无超时 = 丢一条消息就静默永久挂起
   （本轮 35/38 挂起形态的根因面）。超时后补发一条 close 排在 FCFS 队尾，
   worker「先建条目再删」，不泄漏解码器。注意口径：**超时 ≠ 丢消息**——单
   worker 串行处理时纯排队也可能超时（38 条 × 数百 ms），「不设超时最终也
   没完成」才是丢消息的铁证。**超时要覆盖所有请求类型，不只协商类**：修复
   轮只给 open/attach/probe 加了 10s 超时，decode 漏了——满载运行数分钟后
   decode 消息也会丢，该实例 Promise 永不 settle、上层 `_pendingDecode`
   in-flight 守卫永久卡 true → 实例冻结且**零 warn**（「日志干净但画面卡住」
   就是它的签名）。decode 超时的善后与 open 不同：不回退主线程（多实例同时
   超时会雪崩），reject 让上层 catch 复位守卫、播放 tick 自动重发同帧
   （已合成帧 O(1)），自愈式恢复即可。
5. **丢消息不限于「大消息 × SAB 交织」——持续满载后小消息也丢**：open 串行 +
   源去重修完后上报窗口（前 ~45s）443 tasks/s 零丢失，但持续运行数分钟后通道
   劣化，decode 元数据消息（几十字节）也开始大量丢（自愈重发又丢，动图近乎静止）。
   对策与 open 串行同逻辑——**收缩并发交织窗口**：请求方加全局并发闸门（同时在途
   decode ≤8 条，吞吐 8×~21ms≈380/s 对满载需求仅轻微限流），并顺带消掉「同时
   超时→同时重发」的重发风暴。判定用逻辑判据很干净：**动图停 + 零超时 warn =
   主线程事件循环已死**（timer 挂主线程，活着必响）→ 排查渲染/GPU/内存层；
   动图停 + 有 warn = 通道还在丢。
6. **探针/指纹方法论**：单字节写读探针（header 1 字节）探不出大面积映射错乱
   ——attach 后至少 5 点位置相关 pattern 采样；每帧 decode 后做统计指纹
   （首/中/尾 + 非零字节计数 + 字节和 mod 65536）主线程同口径比对，串台
   发生的第一帧即暴露（透明背景样本采样点常全零，统计量才是主力）。
7. **判定陷阱**：`normalizeBytes('')` 返回空 `Uint8Array`（**truthy**！），
   引用 open 未命中必须按 `byteLength === 0` 判空，否则走到解码器报
   missing IHDR 而不是回 sourceMiss。
8. **CLI 构建 appid 双层来源坑**：`profiles/v2/packages/wechatgame.json` 的
   `taskOptionsMap.<taskId>.appid`（平台 profile）**优先于** builder.json
   taskMap options 里的 `packages.wechatgame.appid`——CLI 构建产物
   project.config.json 的 appid 会被占位号覆盖，**每次构建后必须复查产物**
   （node 改回正式号）。SAB/V2 灰度白名单绑 appid，用错号 = 整个环境判定
   作废。出码日志看 `Using AppID:` 一行确认。
9. **构建产物验证的 mangle 坑**：Cocos 构建会 mangle 局部变量名——搜源码
   局部标识符（knownSources/sabSerialChain）恒 0 命中，要搜**属性名 / 全局
   flag / 数字常量**（sourceMiss、`__sabSerialOpen`、`2166136261`=0x811C9DC5）；
   自家 worker bundle（rollup 未压缩）不受影响。中文 warn 文本在 PowerShell
   `Get-Content` 下乱码（编码错位），文件本身没问题——搜中文先换
   `[System.IO.File]::ReadAllText`。改产物 JSON（如 appid 回写）禁用
   `Out-File -Encoding utf8`——PowerShell 5.1 的 utf8 带 BOM，devtools 的
   project.config.json 解析器直接 `SyntaxError: Unexpected token ﻿ at
   position 0` 拒编译；用 `[System.IO.File]::WriteAllText($f, $json,
   (New-Object System.Text.UTF8Encoding($false)))` 无 BOM 写回。**更狠的坑
   （2026-09-16 实证）：PS 5.1 `Get-Content -Raw` 不带 `-Encoding` 读含中文的
   UTF-8 JSON 按 ANSI 误读 → `ConvertFrom-Json` 解析失败 → `$j` 为 null，下游
   赋值/写回照跑，`WriteAllText` 写出空文件——一次就把产物 project.config.json
   清零。产物 JSON 的读写一律用 node（`fs.readFileSync(p,'utf8')` /
   `writeFileSync` + `JSON.stringify`），别走 PowerShell 的 JSON 往返**。
10. **持续满载的终局形态：通道整体楔死（相位跃迁），不是丢包率上升**。双侧
   逐类型收发计数 + 每 10s 双向对账打点定案：断流前最后一次对账**完全健康**
   （双向差=在途数 2~3、累计重传 8 次全部恢复、判死 0、worker 0.0s 前有收包）
   → 日志**行号相邻**处双向整体死亡：312 条 decode（1s×3 重传）零成功、9 次
   诊断请求（每次 30s 耐心）全灭、请求速率 557/s 塌缩到 0.8/s、≥6 分钟不自愈、
   主线程 JS/渲染全程存活（fps 59.6，死的只有通道）。**微信原生层同窗口自报**
   `[基础库开发同学注意] main_message_sock wait 超过 5 秒内未 notify, val: 0,
   13452`——楔死在原生消息管道，不在两侧 JS 事件循环。触发量级高度可复现：
   两轮独立 SAB 满载分别在累计 **~10.3k / ~10.6k 条**主→工请求处断流
   （20-40s 满载）；copy 档因主线程全量缓存首轮后消息数天然封顶（~7.6k）
   全部通过——无法区分「累计消息数阈值」与「SAB 满载特有」。**ARQ 重传穿
   不过楔死态**（重发同样不可达）：应用层能做的只有把稳态消息量压到阈值下
   （主线程帧缓存副本，循环第二轮起零 decode 往返）或检测楔死后整体重建
   worker（代价：解码状态全丢、源字节重传）。
11. **通道诊断三件套（可复用模式）**：①双侧逐类型收发计数对差——gap≈在途数
   = 健康，gap 持续涨 = 该方向丢；②worker 最近收包时刻（快照经 channel-stats
   回传）停涨 = worker 收包侧断流；③微信基础库原生警告时间戳
   （`main_message_sock ... 未 notify`）= 原生管道楔死的「官方证人」。盲区注意：
   channel-stats 本身要往返，**通道死后查不到 worker 侧**——但「诊断请求 ×N 次
   全部无响应」本身就是整体断流级证据（个别丢消息会重传恢复）。
12. **arena 单块 SAB（工程模式 + 否定性结论）**：检验「多块 SAB 是否为丢消息
   触发因子」——全进程只建一块大 SAB（?sabArenaMB= 缺省 64MB）按 slot 分区
   （每 slot = header + w*h*4 像素区〔header 当时 4B，自驱变体扩到 24B，见第
   13 条〕，8 字节对齐 bump 分配不回收），首次
   attach-shared 带 arena 引用 + offset/length（worker 缓存引用），后续只带
   offset——SAB 对象过线从 N 次降为 1 次。真机生效（专有日志行 + 传输字节
   与逐块版逐字节相同）但**没治丢消息**：多 SAB 假设否定。arena 视图必须带
   显式长度（无长度视图延伸到 arena 末尾，串进别的 slot）；worker 侧收 offset
   前必须已有 arena 引用（缺失要拒绝，别拿 undefined 建视图）。
13. **worker-auto 自驱范式（方案 C，每帧消息 → O(1) 的应用层根治）**：官方
   任务槽范式（§2.2 / §8 overlay compositor）的反向应用——不要 OffscreenCanvas，
   播放时钟搬进 worker：worker 自驱解码直接写 SAB slot 像素区，经 header 帧序号
   （Atomics add 最后发布）通知；主线程每引擎帧轮询序号、变了才建视图
   `texture.uploadData`。稳态**每帧零消息**，通道只剩 open/attach/probe/
   auto-start/auto-seek 等 O(1) 控制消息——第 10 条楔死（~10k 累计请求阈值，
   ARQ 穿不透）的应用层根治极限形态：稳态消息量构造上归零，而非缓解。
   工程要点（全部在 performance 仓库 extensions/animated-image 可复查）：
   - **header 信箱布局**（24B / 6×Int32，SAB slot 头）：word0 byte0 探针 token
     （复用既有 attach 探针约定）、word1 frameIndex、word2 frameSeq、word3
     durationMs、word4 非零字节数、word5 字节和（<2^31 int32 无损）——主线程
     轮询侧对自家视图同口径算指纹比对（每帧数据正确性探针的自驱等价物）。
   - **发布序 = seqlock 单写者**：像素 → 元数据 store → `Atomics.add(word2, 1)`
     最后；轮询侧先读 seq、变了再读数据，指纹不符时**复读 seq 相同才判罪**
     （否则可能是读到发布中途的半新半旧，不能报串台）。
   - **autoPending 单飞守卫（踩过的坑）**：时钟每拍自排下一拍（rAF 优先、
     setTimeout ~60Hz 兜底、全暂停停钟、auto-start 重启；rAF 首拍必须过看门狗
     验证，见第 14 条）。连续 auto-start/
     resume 若各排一条 rAF，每拍翻倍、时钟越跑越快——排拍前置 pending、回调
     头清零，rAF 队列恒 1 条（沙箱断言 `rafQueue.length === 1` 防回归）。
   - **stale 写防护**：解码回调里 `currentFrame !== index` 则不写内存不推序号
     （防解码期间时钟已跨帧、过时帧被轮询侧上屏）；条目已死（close 单向通知
     丢失）autoAdvance 自动 delete 防僵尸条目吊着空转时钟；连续 8 次解码失败
     停摆该条目。
   - **auto-state 响应不入 ARQ 响应缓存**：重传 = 重执行幂等取新状态（seek/
     start 幂等），回旧快照会让主线程拿到过时播放态——与 channel-stats-reply
     同款理由。
   - **降级与口径**：无 SAB 环境协商自动落回拉模型 worker（识别单门
     postMsAvg > 0.1，健康档只有 open 摊销残差）；播放时钟在 worker ⇒ 主线程
     `duration` 恒 0、taskMs 恒 0（解码耗时不再经 decode 往返上报），跨档比较
     看 frame P95 / 1% low / mainThread.logic（口径详见 performance 仓库
     tools/PERF.md worker-auto 节）。
14. **「rAF 存在」≠「rAF 会回调」——worker 内定时器要看第四种组合（2026-09-16
   真机卡死定位）**：worker-auto 首次真机冒烟「一启动就卡死」，vConsole 判读 =
   主线程全程健康（fpsAvg 61.4）而 **postCount 只有 open+首帧 seek 各 20 条** ——
   自驱时钟一拍都没跑。根因：wx V2 灰度 worker **暴露 `requestAnimationFrame`
   却从不回调**（无渲染面；web 语义里 worker rAF 本就只在有已 bind 的
   OffscreenCanvas 时排帧）。ticker 的组合空间有四种：rAF 在且会调（浏览器）、
   rAF 不在（旧版 wx worker → setTimeout 兜底）、setTimeout 也不在（停钟）、
   **rAF 在但永不调（wx V2 真机）**——前三种沙箱都能复现，第四种只能靠防御。
   - **首拍看门狗模式（自愈且宿主无关）**：首次排 rAF 时同时挂一个 setTimeout
     看门狗（250ms）：rAF 回调了 → 会话级信任（proven），此后直排零开销；没回调
     → 会话级判死（dead），`autoPending` 清零、链交给 setTimeout（判死后到达的
     僵尸 rAF 回调检查 dead 标志直接忽略，不碰 pending）。
   - **worker 侧状态必须经消息回传诊断**：真机 worker 的 console **不透传
     vConsole**（整个 run 零 worker 侧日志），时钟死在哪只能靠把引擎健康快照
     （ticker/判死/拍数/条目/播放数/错误数）塞进 channel-stats-reply，主线程
     [channel] 对账行尾直接打出——下次真机一行日志读出死因，不用再猜。
   - **判读技巧**：postCount 只有「open+seek 条数」= 时钟零拍；主线程帧率健康
     + 动图全静止 = worker 时钟死（不是主线程死、不是通道死）。
   - **复测坐实（同日）**：看门狗版真机复测打出 `auto timeout(rAF判死已切timeout)
     拍2875 条目20/播20/拍挂起`，~57 拍/s 稳定推进、fpsAvg 60.8——与预测形态
     逐字一致。判读补两条：① **timeout 链下「拍挂起」是常态**（时钟链式自排、
     下一拍总在途），判时钟死只看「拍数不涨」，不看挂起标志；② auto 档回退识别
     门 postMsAvg 从 0.1 放宽到 1.0——真机健康 open 残差实测 0.166（20 次 open
     × ~11ms 往返摊进 ~1300 个轮询样本），0.1 会误报健康档（拉模型回退是数十 ms；
     门终值 5.0，见第 15 条）。
   - **顺带实证**：arena（主线程分配单块 SAB）× auto 组合在真机是通的（首帧
     seek 经微任务发布、主线程轮询命中并 upload）——坏的只有 ticker，该组合补
     了沙箱用例（driveAutoArenaCase / driveAutoDeadRafCase）。
15. **矩阵跑批三收获：分轮冷启协议 / auto 免疫 / 冻结轮识别（2026-09-16 真机
   x65 heavy 矩阵，正式号灰度环境）**：
   - **同冷启动连跑多轮矩阵会污染 copy 档**：r1 的 worker-auto（SAB arena attach）
     进会话之后，copy 档从 r2 起通道劣化崩盘——fps 55.2→38.6、1191 条消息→9272 条
     （7× 重传风暴）、P95 22→62；与第 10 条「~10k 累计请求楔死」同向但 SAB 在场时
     阈值更低（本轮累计 ~1.4k 即崩）。对策：**每次冷启动只跑一轮**（perf-driver
     `rounds=1`：三变体紧邻同窗、通道全新），N 次冷启动 = N 组干净同窗对照——
     真机 5/5 三连全健康。「真冷启动」判据：矩阵能重跑本身即证明 JS context 重建
     （driver IIFE 只在冷启动执行）；重开停在菜单 = 进程还热，退出等 10s 再进。
   - **worker-auto 对通道劣化免疫**：copy 崩掉的同一会话里，auto 连续 4-5 轮满帧
     （P95 19-21、fps 59.5+）——稳态零每帧消息，劣化通道无感。方案 C 的最硬论据
     之一：把消息量构造性归零，比任何重传/限流都彻底。
   - **auto 回退门终值 `postMsAvg > 5.0`**：残差 = Σopen 往返 ÷ 全部样本数
     （decode 样本三值全 0，分子只来自 open），**分子分母各自独立地随负载涨落**
     ——x20 残差 0.166、x65 heavy 0.605~2.824（Σopen≈11s〔串行 open × 大 b64〕
     摊进 7.4k~19k 个轮询样本），而拉模型真身 30~93ms。0.1→1.0→5.0 三次校准的
     教训：负载加重（实例多/源大/open 慢）或上屏帧变少都推高残差，**门要沿负载
     加重方向留余量**，别按轻载数值定门。
   - **动画冻结轮识别（worker 系变体通用）**：引擎 fps 不反映动图是否在动——
     冻结轮（taskCount≈纯 open 数、上屏帧≈0）帧面虚好（实测 fps 59.7/掉帧 13
     @x65 heavy 全虚，还把 open 残差顶到 15.3 误触当时的 1.0 门）。识别规则：
     **worker 样本数 < 2×实例数（load 标签 xN）⇒ 该 run 帧面无效**，perf-report
     逐 run 报警点名；处置 = 移出结果目录归档，别进对比组。

## 10. 真机自驱矩阵管线：探测时序 / 残留清理 / 联调拓扑（2026-09-16 晚，driver v3.2）

背景：driver（`tools/perf-driver.js`，注入 `build/wechatgame/game.js` 尾部的自驱矩阵脚本）v3 加了「真机探测自启」——无 perfMatrix 时探 collector `GET /state`，可达即固化配置并自动开跑全矩阵（普通预览码扫一次 = 15-20 分钟全自动跑完，零人工介入）；固化过的地址死了有陈旧守护（改写为默认地址续跑）。当晚真机表现「collector 全不可达」待机、**全晚零网络包**，换 3 种网络拓扑全无效——最后靠 vConsole 行序定位到探测时序 bug。修复版 v3.2 已部署（waitForAdapter），教训清单：

1. **真机没有原生 `XMLHttpRequest`，web-adapter 随引擎异步注入**。driver 挂 game.js
   尾部，求值早于 adapter → `new XMLHttpRequest()` 当场抛异常 → try/catch 吞掉 →
   探测毫秒级全灭、一个包都没发，假象「collector 全不可达」。模拟器（Chromium
   原生 XHR）不复现；同机傍晚成功的是无探测路径的老 driver——「同机刚才能通
   现在不通」先查代码路径差异再查环境。**诊断铁证：vConsole 里 driver 的待机行
   印在引擎初始化日志（如 `[rccEngine]`）之前** = driver 求值早于 adapter。修复：
   任何网络探测前 `waitForAdapter`（轮询 `typeof XMLHttpRequest === 'function'`，
   500ms 间隔、30s 超时放弃）。这是「全局在 ≠ 现在在」的时序版（对照 §5 的
   「全局在 ≠ 通道在」）。
2. **适配器 XHR 可能不支持 `.timeout` 属性**：对死地址的探测会挂到 wx.request
   默认超时（分钟级）。JS 侧看门狗兜底：`setTimeout(3500)` + once() 单发分流，
   回调只认第一次到达（onload/onerror/看门狗竞速）。
3. **server 的 /state 探测路由必须打日志**（`[probe] 时:分:秒 来源IP`）——静默
   响应让「探测包到没到」无从判断，整晚排障靠它收口。判读纪律：核对来源 IP，
   **排除人为浏览器手动访问**（实测误判过一次「通了」，其实是用户手点的内网
   链接）；tail 监控时把 `[static]` 资源拉取也放进过滤——游戏只要加载就有痕迹。
4. **storage 残留 `perfQuery` = 僵尸轮**：中断会话遗留的 query.scene 让 MainScene
   onLoad 自动跳进压测场景，拿已死 collector 跑废轮、报告发死地址（vConsole
   实证）。对策：boot 时**同步** `storageDel('perfQuery')`——必须赶在 MainScene
   读 query 之前，driver 每次 run 前都会重写，清掉不影响正常流程。
5. **CLI preview（不带任何参数）也会拉起 IDE，模拟器自动开跑注入的 driver**
   （platform=devtools 无条件跑内置矩阵）污染收集器——§6 第 6 条的杀 devtools
   纪律适用于一切出码。sim 跑完首个 run 需 ~3 分钟，**出码后 2 分钟内杀掉即零
   污染**；杀完确认 `remaining: 0`，WeChatAppEx 是用户微信本体绝不能碰。污染
   报告按时间窗隔离出结果目录。
6. **Windows 移动热点联调拓扑（免改包）**：开发机开「移动热点」（共享自 WLAN）
   后热点接口 192.168.137.1、手机在 137.x 网段——**包内写死的 192.168.9.x
   collector 地址无需改**：热点客户端发往 9.x 的包经网关（=本机）本地送达，
   server 监听 0.0.0.0 照收。适合「手机所在网段与包内固化地址不一致」的现场
   救急（双向 ping 验证后再跑）。
7. **预览码有效期 ~25 分钟**：过期后扫码拉包直接失败（微信历史列表里的旧入口
   同样失效）。用户离开再回来须重出码：preview → base64 解码成 jpg → 杀
   devtools → 交付，管线 ~1 分钟——别预出码等人，人到位再出。
8. **同名多 SSID 可能是同一只 AP 的多个 VAP**：SUD / SUD-005 是同盒子（同 MAC
   相邻尾号）的不同 VAP、同桥接网段，怎么连都互通；Windows
   `Get-NetConnectionProfile` 的 Name（如 "SUD 4"）只是配置文件数字后缀，真实
   SSID 看 `netsh wlan show interfaces`。
9. **工具链细节**：perf-server.mjs 启动必须带 `--out=`（缺省写
   tools/perf-results 会串目录、报告混入 Chrome 基线）；/state 的 lastReport 是
   内存态、重启即 null（driver 完成判定用时间戳比较，无碍）；inject 脚本的
   日志版本文案是写死的旧标签（v2 字样），判断注入成败看 game.js 字节数 +
   `node --check`。

## 11. 真机 wasm 实例化铁律与 reporting-getter 陷阱（2026-09-17，Android 16 / 微信 8.0.78 / 基础库 3.17.3，spine wasm worker 化五轮修复全记录）

背景：spine 解码器 wasm（emscripten MODULARIZE 胶水 + `instantiateWasm` hook）要同时
在主上下文（builtin 档）与 worker（worker/SAB 档）实例化。devtools 模拟器一轮绿，
真机五轮红——模拟器主上下文/worker 都有原生 `WebAssembly`、wasm 按本地磁盘路径读，
**真机限制一个都复现不了；模拟器跑绿零信息量，wx 产物问题直接真机验证**。

### 11.1 三条路径铁律（主上下文与 worker 同源）

1. **真机主上下文没有 `WebAssembly` 全局**（worker 里"有"，但见 11.2 的 getter 陷阱）。
   引擎 pal（wasm-minigame）路线本来就走 `WXWebAssembly`，别绕。
2. **`WXWebAssembly.instantiate` 只认「代码包内路径」**：字节形态（ArrayBuffer/
   Uint8Array）与 `USER_DATA_PATH` 本地文件（写盘成功也不行）统一报同一句**误导性**
   错误 `only support file type .wasm or .wasm.br`——错误文本与真实原因无关，按路径
   分类而不是按文案排障。唯一出路 = wasm 随包（拷进 `build/wechatgame/cocos-js/`，
   pal 的 getPlatformBinaryUrl 恒加 `cocos-js/` 前缀，正是它找的路径）。
3. **包体积账要算净重**：spine 包 3,333,347 + wasm 774,231 = 4,107,578 < 4MiB
   （余 86KB）。当年"必超限"是 assets/resources 与 cocos-js **两份 wasm 重叠**；
   resources 整目录 packOptions.ignore 后回包可行。preview 输出的 TOTAL 表是最终判据。

### 11.2 worker 侧 reporting-getter 陷阱（本轮最大新知）

worker 沙箱里 `WebAssembly` 存在，但基础库把它的属性做成**上报后返回真身**的
reporting getter。真机栈实证（六轮才拿到，见 11.4 诊断法）：

```
ReferenceError: Reporter is not defined
  at s (WAWorkerSubContext.js:1:139872)
  at WebAssembly.get [as instantiate] (WAWorkerSubContext.js:1:141389)
```

- getter 体内调用 `Reporter` 上报 API 用量，而 **worker 沙箱根本没注入 Reporter
  全局**（wx 基础库 bug，3.17.3）——`typeof WebAssembly.instantiate` 都会触发 getter
  当场 ReferenceError，`instanceof WebAssembly.Module` 同理。属性访问=踩雷，与拿它
  做什么无关。
- 修复 = worker 顶层注入**递归 no-op Proxy 桩**（缺场才注入，不覆盖真 Reporter）：

```ts
if (typeof (globalThis as { Reporter?: unknown }).Reporter === 'undefined') {
    const makeNoop = (): unknown => new Proxy(function noop () {} as unknown as object, {
        get: (t, p) => (p === Symbol.toPrimitive ? () => '' : makeNoop()),
        apply: () => makeNoop(),
        construct: () => ({}) as object,
    });
    (globalThis as { Reporter?: unknown }).Reporter = makeNoop();
}
```

  递归形态吃掉任意属性/调用/嵌套/new 链，上报静默，getter 返回真身。注入后
  `WebAssembly.instantiate(bytes)` 拒（铁律 2）→ 包路径实例化成功——worker wasm
  五轮失败一朝转绿。
- Module 鸭子判定也别用 `instanceof`（属性访问）：`Object.prototype.toString.call(m)
  === '[object WebAssembly.Module]'` 完全不碰 polyfill 属性。

### 11.3 emscripten 胶水的同步抛出口（读源码定罪，别猜）

胶水 `createWasm`（MODULARIZE + `instantiateWasm` hook 在场时）：**`instantiateAsync`
/ `instantiateStreaming` 全是死代码**——hook 分支 `return new Promise(executor)` 提前
返回，executor 里同步调 hook、hook 同步抛则 rethrow → 工厂 Promise reject。推论：
**同步阶段的 WebAssembly 属性访问只可能发生在你自己的 hook 代码里**（胶水其余路径
未执行）。另两个坑：

- **polyfill 参数校验是同步抛**（不是 promise 拒绝）：`.then(onReject)` 永远等不到，
  兜底路线从未被尝试——每条路线都要独立 try/catch，catch 里才好落下一候选。
- hook 的异步失败传不出去（胶水只认 successCallback）：给 hook 一个 abort 出口
  （`hookAbort.fn` 在 factory() **之前**赋值——hook 在工厂同步段就被调，时序敏感），
  失败立即带错误回主线程，别等 60s 超时护栏。

### 11.4 诊断基建（把"下一轮真机"变成"一次定位"）

1. **合并错误链**：每条实例化路线（module 重载/bytes/各路径候选）的 try/catch 都
   `errors.push('路线名[参数]: ' + msg)`，终态合并 join(' | ') 上报——错误文本自带
   路线画像。
2. **栈头随错误回传**：`e.stack.split('\n').slice(0,3).join(' <= ')` 拼进错误消息。
   轮6 靠它一行指认 `WebAssembly.get [as instantiate]`——worker 没有可导出的
   vConsole 时，栈是唯一现场。
3. **worker 错误直报 collector**：主上下文 service 的 catch 里把错误 POST 到
   `<collector>/driver`（地址从 `__spineWasmUrl` 截尾推导）——免导 vConsole、免
   人工转发，错误自动进监控 tail。
4. **产物取证用字节切割**：`https://usr/game.js` 的真机栈 ≠ 构建产物 game.js
   （微信合并编译单元）；产物内定位用 `grep -ob` 拿**字节**偏移后
   `tail -c +N | head -c M` 切片——awk substr 按**字符**数，中文注释场景两者错位。
5. **Monitor 换 filter 保持完整**：监控 grep 交替项时漏 `[static]` 会把"资源拉取
   痕迹"整段变盲区，"零拉取"的推论全作废。

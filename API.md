# mallNav 集成调用说明（API v1.2.0）

单文件版室内导航（`index.html`）把全部交互动作提取成了一个全局对象 **`window.mallNav`**。
页面内嵌、iframe 嵌入、上层业务系统（工单 / 招商 / 客流）都可以直接调用，不需要改一行渲染代码。

- 无依赖、无网络请求，`<script>` 执行完即可用。
- 坐标一律是**模型坐标**（单位：1 单位 = 14 cm，`data.mpp = 0.14`），楼层用 `floor: 0|1|2`（对应 1F/2F/3F）。
- 位置参数 `point` 支持 4 种写法，任选其一：

```js
'优衣库'                              // 店名 / POI 名，模糊匹配后自动吸附到可走点
{name:'星巴克甄选'}                    // 同上
{x:120, y:170, floor:0}               // 模型坐标（floor 缺省 = 当前层）
[120, 170, 0]                         // 数组形式
```

- 所有方法**同步返回**，失败返回 `null` / `false` 并在界面上给出提示，不抛异常，可安全链式调用。

---

## 1. 就绪与事件

| 方法 | 参数 | 返回 | 说明 |
|---|---|---|---|
| `mallNav.info()` | — | `{version, since, built, repo, file, deps, model, expected, ok, zones, floors}` | 版本与完整性自检：`model` 是几何/铺位数据的指纹，`ok:false` 说明这份副本被截断或改坏 |
| `mallNav.ready(cb)` | `cb(state)` | `mallNav` | 就绪回调。若页面已启动完成，会在下一个 tick 立即触发，因此任何时候调用都安全 |
| `mallNav.on(type, cb)` | `type` 见下表，`cb(detail, type)` | **取消监听的函数** | 注册事件 |
| `mallNav.off(type, cb)` | 同 `on` | — | 手动移除 |
| `mallNav.events` | — | `string[]` | 全部事件名 |

| 事件 | detail 结构 | 触发时机 |
|---|---|---|
| `ready` | `{floor, mode, …}`（同 `state()`） | 启动完成 |
| `floor` | `{floor, index, label}` | 切换楼层（含手势/动画自动切层） |
| `mode` | `{mode:'2'\|'3'}` | 2D / 3D 切换 |
| `camera` | `{mode, yaw, pitch, scale}` | 视角、缩放、居中变化 |
| `point` | `{role:'A'\|'B'\|'locate', point:{x,y,fi,name}}` | 选点 / 定位 |
| `route` | 成功：`{ok:true, meters, minutes, transfers, vertical, points, from, to, endFloor}`；失败：`{ok:false}` | 路径生成或清除 |
| `walk` | `{state:'start'\|'progress'\|'pause'\|'resume'\|'arrive'\|'stop', progress, floor, x, y, name}` | 模拟导航过程（progress 每 ~10% 触发一次，避免刷屏） |
| `layer` | `{layer, visible}` | 图层开关 |
| `blocked` | `{reason, x, y}` | 用户点到了不可通行区域 |
| `message` | `{text, ok}` | 界面提示文案变化 |
| `search` | `{query, results[]}` | 调用 `find()` |

---

## 2. 视图控制

| 方法 | 参数 | 返回 |
|---|---|---|
| `setFloor(i, keepView?)` | `i`:0/1/2；`keepView`:true 保持当前相机不重新取景 | `state()` |
| `nextFloor()` | — | `state()`（循环 1F→2F→3F→1F） |
| `setViewMode(m)` | `'2'`/`'3'`（也接受数字 2/3） | `state()` |
| `rotate(o)` | `{yaw?, pitch?, dYaw?, dPitch?}` 弧度；`pitch` 自动夹在 0~1.45（0=正俯视，1.45≈平视）。**仅 3D 生效**，2D 下返回 `{ignored:'…'}` | `{yaw, pitch}` |
| `zoom(k, anchorX?, anchorY?)` | `k`:目标倍数 0.12~3.6；可选屏幕坐标作缩放锚点 | 实际 `scale` |
| `fit()` | — 自动取景（3D 含三层，2D 仅当前层） | `state()` |
| `resetView()` | — 回到默认机位 | `state()` |
| `toScreen(x, y, fi)` | 模型坐标 → 画布像素 | `{sx, sy, depth}` |
| `toWorld(sx, sy, fi)` | 画布像素 → 模型坐标（视角过平时返回 `null`） | `{x, y}` |

## 3. 选点与路径

| 方法 | 参数 | 返回 |
|---|---|---|
| `setStart(p)` | `p` = point | `{x,y,fi,name}` 或 `null` |
| `setEnd(p)` | `p` = point，调用后立即算路 | `{x,y,fi,name}` 或 `null` |
| `route(a, b)` | 起终点，一次出结果 | 路线摘要 或 `null` |
| `goto(q, from?)` | `q` 终点；`from` 省略时用已选起点 | 路线摘要 或 `null` |
| `reroute()` | — 用当前 A/B 重算 | 路线摘要 |
| `clear()` | — 清空起终点、路线、小人 | `true` |
| `find(q)` | 关键字（店名 / 卫生间 / 扶梯 / 客梯 / 机房…） | `[{name, floor, floorLabel, x, y, type, walkable}]`，可走的排前面 |
| `locate(q, opts?)` | `opts:{floor?, zoom?, mode?}`；也可直接传 `find()` 的结果对象 | 命中的地点 或 `null` |
| `placeAt(x, y, fi)` | — | 该点可读名称，如 `'2F 无印良品 MUJI'` |
| `isWalkable(x, y)` | 模型坐标（三层轮廓一致，与楼层无关） | `true` / `false` |

**路线摘要**（`route()` / `state().route`）：

```js
{ ok:true, distance:1182.4,   // 模型单位累计长度
  meters:165.5, minutes:2.3,  // 按 1 单位 = 0.14 m、步行 72 m/min 估算
  transfers:2,                // 跨层次数
  vertical:'esc',             // 主要竖向交通方式：'esc' 电动扶梯 / 'lift' 客梯
  points:24,                  // 折点数
  from:{x,y,fi,name}, to:{x,y,fi,name}, endFloor:2 }
```

## 4. 模拟导航（小人）

| 方法 | 参数 | 返回 |
|---|---|---|
| `start()` | — 沿当前路线行走 | `true` 开始 / `false` 无路线 |
| `pause()` / `resume()` | — | `true` / `false`（`resume()` 在无进度时会重新 `start()`） |
| `stop()` | — 小人消失，进度条收起 | `true` |
| `seek(t)` | `t`:0~1，直接把小人摆到路径该进度处并**暂停**（做讲解/回放定位很有用） | 实际进度 或 `null` |
| `setSpeed(k)` | `k`:倍速，1 = 110 单位/秒（≈15 m/min 视觉速度） | 当前倍速 |

## 5. 图层开关

`toggleLayer(key, hide?)` / `setLayers({key:true,...})`

| key | 内容 |
|---|---|
| `shop` | 商铺色块 + 墙体线（关掉后只剩通道） |
| `nogo` | 不可通行区（中庭、扶梯井、电梯厅、疏散楼梯、机房、后勤仓储） |
| `esc` | 电动扶梯 |
| `lift` | 客梯井道与轿厢 |
| `poi` | 卫生间 / 服务台 / 客梯 图标 |
| `names` | 商铺名称文字 |

```js
mallNav.toggleLayer('names', true);          // 隐藏店名（hide=true）
mallNav.setLayers({shop:true, names:true, poi:false});   // true=显示
```

## 6. 状态、版本与数据

集成方在加载后可先校验副本完整性，再决定是否需要重新拉取：

```js
const bi = mallNav.info();
if (!bi.ok) console.error('导航文件不完整', bi.model, '≠', bi.expected);
else console.log(`v${bi.version} · ${bi.built} · ${bi.floors} 层 / ${bi.zones} 铺位 · ${bi.deps}`);
```
页面顶栏右侧的 `v1.2.0 · 09-28` 角标即同一份元数据，悬停可看构建时间、提交号、指纹与仓库地址。


```js
mallNav.state();
// {floor:0, floorLabel:'1F', mode:'3', yaw:-0.55, pitch:0.95, scale:1.02,
//  start:{...}|null, end:{...}|null, route:{...}|null,
//  walking:false, paused:false, progress:0, speed:1, walker:null,
//  layers:{shop:true,nogo:true,esc:true,lift:true,poi:true,names:true}}

mallNav.data;
// {mpp:0.14, floorHeight:62, units:'模型单位（1 单位 = 14 cm）',
//  floors:[{index:0,label:'1F',zones:['优衣库 UNIQLO',…],pois:['卫生间',…]}, …]}
```

## 7. iframe 集成（postMessage）

嵌入方无需拿到 JS 对象，直接发消息即可，返回结果带同一个 `id`：

```js
frame.contentWindow.postMessage(
  {type:'mallNav:call', id:1, method:'route', args:['优衣库','星巴克甄选']}, '*');

addEventListener('message', e => {
  if (e.data?.type === 'mallNav:result' && e.data.id === 1) console.log(e.data.data);
});
```

`method` = 上表任意方法名，`args` = 参数数组。异常时 `ok:false`，`data` 为错误信息。

---

## 8. 联动调用示例

见 [`INTEGRATION.md`](INTEGRATION.md)：搜索框联动、楼层导览、客流热力回放、导览剧本、iframe 嵌入、埋点上报 6 个完整场景。

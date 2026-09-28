# mallNav 联动调用示例

配合 [`API.md`](API.md) 使用。下面 6 个场景都是可直接粘贴运行的完整代码，
前提是页面里已经加载了 `index.html`（或把它放进 iframe，见示例 5）。

---

## 示例 1 · 搜索框联动：输入店名 → 定位 → 一键导航

最常见的用法：一个搜索框 + 一个「去这里」按钮。

```html
<input id="q" placeholder="搜索店铺 / 卫生间 / 扶梯" list="dl">
<datalist id="dl"></datalist>
<button id="go">到这里去</button>
```

```js
const N = window.mallNav, q = document.getElementById('q'), dl = document.getElementById('dl');

// ① 输入即联想：find() 返回按“可通行优先”排序的结果
q.addEventListener('input', () => {
  const hits = N.find(q.value).slice(0, 8);
  dl.innerHTML = hits.map(h => `<option value="${h.name}">${h.floorLabel} · ${h.type}</option>`).join('');
});

// ② 选中后定位（切层 + 居中 + 放大），再取当前起点算路
document.getElementById('go').addEventListener('click', () => {
  const hit = N.find(q.value)[0];
  if (!hit) return alert('没找到，换个关键词');
  N.locate(hit, { zoom: 1.6, mode: '3' });          // 相机先飞到目标
  const start = N.state().start || { x: 60, y: 166, floor: 0 };   // 缺省用北出入口
  const r = N.route(start, hit);                   // 生成 A→B 连线
  if (r) console.log(`${r.meters} 米 / 约 ${Math.ceil(r.minutes)} 分钟 / 换乘 ${r.transfers} 次`);
});
```

调用顺序：**`find()` → `locate()` → `route()` → `start()`**，四步就是完整的“搜索—定位—算路—带路”。

---

## 示例 2 · 楼层导览面板：点楼层按钮 + 只看该层

招商/运营后台里挂一个楼层切换条，同时把无关图层收起来。

```js
const N = window.mallNav;

function showFloor(i) {
  N.setViewMode('2');            // 平面模式看铺位最直观
  N.setLayers({ shop:true, names:true, nogo:true, esc:true, lift:false, poi:false });
  N.setFloor(i);                 // 会触发 floor 事件
  N.fit();                       // 重新取景，保证整层完整可见
  // 2D 下 rotate() 不生效（无姿态角），需要倾斜视角就先 setViewMode('3')
}

document.querySelectorAll('[data-f]').forEach(b =>
  b.addEventListener('click', () => showFloor(+b.dataset.f)));
```

```html
<button data-f="0">1F</button><button data-f="1">2F</button><button data-f="2">3F</button>
```

想圈出一类店，只要一行：`N.find('美食广场').forEach(h => console.log(h.floorLabel, h.name))`；
或者关掉其它店名：`N.toggleLayer('names', true); N.locate('美食广场');`

---

## 示例 3 · 客流回放 / 讲解模式：`seek()` 按进度跳点

`seek(t)` 会把小人直接摆到路径的 t 处并暂停，配合时间轴或语音讲解非常好用。

```js
const N = window.mallNav;

// 剧本：进度点 → 讲解词
const script = [
  [0.00, '从北出入口进入，左手边是优衣库'],
  [0.35, '前方右转到电动扶梯，1F→2F 约 30 秒'],
  [0.62, '上到 2F 中庭，右手边是星巴克'],
  [1.00, '到达 3F 终点，全程约 165 米'],
];

N.route('优衣库 UNIQLO', '外婆家');       // 先出路线
let i = 0;
N.on('walk', d => {                        // 小人行进时自动播报
  if (d.state !== 'progress') return;
  while (i < script.length && d.progress >= script[i][0]) {
    speak(script[i++][1]);                 // 换成你们的 TTS / 字幕
  }
});
N.setSpeed(1.5); N.start();

// 时间轴拖动 = 直接跳点
document.getElementById('timeline').addEventListener('input', e => {
  N.seek(e.target.value / 100);
  N.setViewMode('3');
});
```

---

## 示例 4 · 事件驱动看板：把导航过程同步给自己的 UI

不侵入渲染，只订阅事件，适合埋点、大屏、工单系统。

```js
const N = window.mallNav;

N.ready(state => console.log('导航组件就绪', state.floorLabel, state.mode));

N.on('point',   d  => track('nav_pick',    { name:d.point.name, floor:d.point.fi }));
N.on('blocked', d  => track('nav_blocked', { reason:d.reason }));  // 用户点了中庭/扶梯井/机房
N.on('route',   d  => { if (!d.ok) return;
  document.getElementById('panel').innerHTML =
    `${d.from.name} → ${d.to.name}<br>${d.meters} m · ${d.transfers} 次跨层 · ${d.vertical === 'esc' ? '扶梯' : '电梯'}`;
});
N.on('walk', d => {
  document.getElementById('bar').style.width = (d.progress * 100) + '%';
  if (d.state === 'arrive') track('nav_arrive', { name:d.name });
});
N.on('camera', d => localStorage.setItem('mall.cam', JSON.stringify(d)));  // 记住视角
```

刷新后恢复视角：

```js
const cam = JSON.parse(localStorage.getItem('mall.cam') || 'null');
if (cam) { mallNav.setViewMode(cam.mode); mallNav.rotate({ yaw:cam.yaw, pitch:cam.pitch }); mallNav.zoom(cam.scale); }
```

---

## 示例 5 · iframe 嵌入：跨文档调用

```html
<iframe id="m" src="index.html" style="width:100%;height:70vh;border:0"></iframe>
```

```js
const frame = document.getElementById('m');
let seq = 0; const pending = new Map();

function call(method, ...args) {                       // 把 postMessage 包成 Promise
  return new Promise(res => {
    const id = ++seq; pending.set(id, res);
    frame.contentWindow.postMessage({ type:'mallNav:call', id, method, args }, '*');
  });
}
addEventListener('message', e => {
  const d = e.data;
  if (d?.type === 'mallNav:result' && pending.has(d.id)) pending.get(d.id)(d);
});

// 联动：外部按钮 → 内部导航
document.getElementById('to-food').onclick = async () => {
  await call('setViewMode', '3');
  const r = await call('route', '北出入口', '美食广场 B 区');
  if (r.ok) await call('start');
  else console.warn('算路失败', r.data);
};
```

> 生产环境请把 `'*'` 换成导航页的实际 origin，并校验 `e.origin`。

---

## 示例 6 · 巡更剧本：多段路线串成一条动线

把一串点位依次导航，走完自动接下一段（安保巡更、带看动线、开业导览）。

```js
const N = window.mallNav;
const tour = ['优衣库 UNIQLO', '服务台', '电动扶梯 E1', '星巴克咖啡', 'Apple 授权店', '外婆家'];
let leg = 0, from = tour[0];

function nextLeg() {
  if (leg >= tour.length - 1) { N.clear(); console.log('巡更结束'); return; }
  const to = tour[++leg];
  const r = N.route(from, to);
  if (!r) { from = to; return nextLeg(); }             // 该段不可达就跳过
  document.getElementById('tip').textContent =
    `第 ${leg} 段 · ${r.meters} 米 · ${Math.ceil(r.minutes)} 分钟`;
  N.start();
  from = to;
}
N.on('walk', d => { if (d.state === 'arrive') setTimeout(nextLeg, 800); });
nextLeg();
```

想中途接管：`N.pause()` 停下讲解，`N.seek(0.8)` 跳到末段，`N.stop()` 全部结束。

---

## 速查：三条最常用的调用链

```js
// ① 搜索带路
mallNav.locate('外婆家', {zoom:1.6}); mallNav.route('北出入口', '外婆家'); mallNav.start();

// ② 大屏展示（缓慢环绕的 3D 总览）
mallNav.setViewMode('3'); mallNav.setLayers({lift:true, poi:true});
setInterval(() => mallNav.rotate({ dYaw: 0.004 }), 30);

// ③ 复位到干净状态
mallNav.stop(); mallNav.clear(); mallNav.resetView(); mallNav.setFloor(0);
```

---
name: browser-evaluate-human-actions
x-provider: dsh-browser-js-auto
x-version: 1
description: Use when writing a JS snippet to run inside browser_evaluate (page-context JS) that must reproduce real human interaction patterns — clicking a search box, typing text character by character, choosing a dropdown suggestion, selecting an option, pressing Enter — instead of raw one-shot automation. Also use when a synthetic click or input "ran but nothing happened" on an addEventListener-based page. 触发场景:需要用 browser_evaluate 模拟真人操作网页(点搜索框/输入/选联想词/点按钮/下拉选择)、做拟人 UI 操作演示、或合成点击没反应时排查页面与脚本问题。
---

# 用 browser_evaluate 模拟人类操作网页

## 核心原则

页面里直接运行的 JS 只能派发**合成事件**(`isTrusted=false`)。要"像人",不能只调一次 `el.click()` 或直接改 `value`,而要按人类的因果顺序派发**一组带随机节奏的标准 DOM 事件**。总原则:**状态必须通过事件通知页面,而不是偷偷改状态。** 同时认清边界:合成事件不执行浏览器默认动作(聚焦/提交/跳转),也驱动不了 CSS `:hover`。

## 何时用 / 何时不用

**用**:普通 `addEventListener` 页面里的 UI 自动化、操作演示、教学 demo、需要在 `browser_evaluate` 里直接跑的脚本(React 受控组件见「常见坑」的额外要求)。
**不用**(改用 Playwright 受信输入,如 `browser_run_code_unsafe`):站点判断 `e.isTrusted`、拖拽文件上传、原生 `<select>` 浮层、需要真实 `:hover`/聚焦/激活(gesture)的交互、验证码等。

## 快速参考:人类动作 → 事件序列

| 人类动作 | 事件序列(除 `mouseenter`/`mouseleave` 外均 `bubbles:true`) |
|---|---|
| 悬停 | `mousemove` → `mouseover` → `mouseenter`,坐标落在元素内部随机点(仅触发 JS 监听,**不**触发 CSS `:hover`) |
| 点击 | `pointerdown` → `mousedown` →(按住 50~140ms)→ `mouseup` → `pointerup` → `click`;默认动作需另显式调用(如 `el.focus()`) |
| 逐字输入 | 每字符:`keydown` → 改 `value`+`setSelectionRange` → `InputEvent('input',{inputType:'insertText',data:ch})` → `keyup`,间隔 40~110ms |
| 回车/方向键 | 目标元素上 `keydown` → `keyup`(`key` 对应值) |
| 原生 select / checkbox | 设 `value`/`checked` + 派发 `change`(浮层无法用 JS 展开) |
| 等异步渲染 | `waitFor` 轮询 `querySelector`,超时报错;**不要固定 sleep** |

## 精简 Helper(可直接抄)

```js
async () => {   // 整个文件即一个 async 函数,可整个粘贴进 browser_evaluate
  const wait = ms => new Promise(r => setTimeout(r, ms));
  const rand = (a, b) => Math.floor(a + Math.random() * (b - a));
  const $ = s => { const e = document.querySelector(s); if (!e) throw new Error('未找到: ' + s); return e; };
  const pt = el => { const r = el.getBoundingClientRect(); return { x: r.left + rand(3, Math.max(4, r.width - 3)), y: r.top + rand(3, Math.max(4, r.height - 3)) }; };
  const hover = async el => {
    const p = pt(el), o = { bubbles: true, cancelable: true, view: window, clientX: p.x, clientY: p.y };
    for (const t of ['mousemove', 'mouseover', 'mouseenter']) el.dispatchEvent(new MouseEvent(t, o));
    await wait(rand(80, 220));          // 悬停迟疑(仅触发 JS 监听,不触发 CSS :hover)
  };
  const click = async el => {           // 按下→停顿→抬起→click(合成 click 不做默认动作)
    await hover(el); const p = pt(el);
    const d = { bubbles: true, cancelable: true, view: window, clientX: p.x, clientY: p.y, button: 0, buttons: 1 };
    el.dispatchEvent(new PointerEvent('pointerdown', { ...d, pointerId: 1, pointerType: 'mouse' }));
    el.dispatchEvent(new MouseEvent('mousedown', d));
    await wait(rand(50, 140));
    const u = { ...d, buttons: 0 };
    el.dispatchEvent(new MouseEvent('mouseup', u));
    el.dispatchEvent(new PointerEvent('pointerup', { ...u, pointerId: 1, pointerType: 'mouse' }));
    await wait(rand(20, 60));
    el.dispatchEvent(new MouseEvent('click', u));
    await wait(rand(150, 350));         // 点完"缓一缓"
  };
  const type = async (el, text) => {    // 逐字符 keydown/input/keyup,带节奏
    el.focus(); await wait(200);        // 合成 click 不会聚焦,输入前显式 focus
    for (const ch of text) {
      const pos = el.selectionStart ?? el.value.length;
      const k = { key: ch, bubbles: true, code: /^[a-zA-Z]$/.test(ch) ? 'Key' + ch.toUpperCase() : '' };
      el.dispatchEvent(new KeyboardEvent('keydown', k));
      el.value = el.value.slice(0, pos) + ch + el.value.slice(el.selectionEnd ?? pos);
      el.setSelectionRange(pos + 1, pos + 1);
      el.dispatchEvent(new InputEvent('input', { bubbles: true, inputType: 'insertText', data: ch }));
      el.dispatchEvent(new KeyboardEvent('keyup', k));
      await wait(rand(45, 110));
    }
  };
  const waitFor = async (s, t = 5000) => { const t0 = Date.now(); while (Date.now() - t0 < t) { if (document.querySelector(s)) return; await wait(60); } throw new Error('等待超时: ' + s); };
  // —— 编排:固定动作序列(无决策),每步先留痕再执行 ——
  const steps = [
    ['点击搜索框',      () => click($('#search-input'))],
    ['输入「深度」',    () => type($('#search-input'), '深度')],
    ['等联想词出现',    () => waitFor('#suggest li')],
    ['点击联想词',      () => click($('#suggest li:first-child'))],
    ['点击搜索按钮',    () => click($('#search-btn'))],
    ['等待结果渲染',    () => waitFor('#results .card')],
  ];
  for (const [label, fn] of steps) { console.log('▶', label); await fn(); }
  return { done: steps.map(s => s[0]) };   // 返回值会带回调用方,可核对
}
```

**使用方式**:本地演示可用 `fetch + eval` 免粘贴反复执行:
`const src = await (await fetch('/human-actions-demo.js')).text(); return await (0, eval)('(' + src + '\n)')();`
**分步演示技巧**:把 helper 挂到 `window.__H`(一次注入),之后每次 `browser_evaluate` 只调一个动作(如 `__H.click(__H.$('#search-btn'))`),便于逐步截图、逐步核对返回状态;要一次跑完就整段粘贴。

## 常见坑

| 症状 | 原因与修法 |
|---|---|
| 脚本跑完但页面没反应 | 先验证目标元素**真的绑了对应监听器**、状态真的没变(读回 `value`/结果数),再怀疑脚本本身;页面缺监听器时先修页面 |
| 点击后没聚焦/没提交/没跳转 | 合成 `click` 不执行默认动作(聚焦、表单 submit、`<a>` 跳转、打开 select);显式 `el.focus()`/`form.requestSubmit()`,需要默认动作时用 Playwright 受信输入 |
| 输入框 value 变了但界面不刷新 | 只赋了 `value` 没派发 `input`;React 受控输入有 value tracker,直接赋值会被忽略 —— 先经原生 setter 写入(`Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set.call(el,v)`)再派发 `input`;普通页面只需 `InputEvent('input',…)` |
| 只派发一次 `click` 无效 | 部分框架/组件监听 pointer/mousedown 前缀;按上表完整序列派发,事件要发到具体元素且 `bubbles:true` |
| 悬停/事件后旧引用失效 | 事件可能触发重渲染、元素被替换;每次操作前重新 `querySelector`,别跨事件持旧节点引用 |
| `isTrusted=false` 被网页拦截 | 合成事件的天花板;换 Playwright 受信输入,别硬绕 |
| 用固定 `setTimeout` 等渲染,时灵时不灵 | 改 `waitFor` 轮询 + 超时报错;每步把关键状态 `return` 出来供调用方核对 |
| 想要 CSS `:hover` 效果 | 合成事件**无法**触发 `:hover`(由浏览器真实指针命中决定),`mouseenter`/`mouseover` 只触发 JS 监听;要真 hover 用 Playwright 移动真实光标,或改用 class 切换 |
| 原生 `<select>` 点不开选项 | 浮层无法用 JS 展开;等价做法是设 `value` + `change` |

## 验证清单

- 每跑完一段就读回状态(输入值/结果条数/标题/高亮项)确认,不要只看"没报错";
- 每步 `console.log` 留痕,便于对照活动日志;
- 错误信息带选择器与超时时间(如 `等待超时: #results .card`);
- 确定性验证:同一脚本跑两遍,动作序列应完全一致(只随机节奏、不随机决策);若需要"每步观察后再决策",那是 Agent 形态(规则/LLM 每步决策),不在本技能范围。

## 真实运行示例

完整可复现 demo(演示页 + 一次性可跑脚本 + 分步截图)为本技能开发时的配套产物,脚本与上文 helper 同源;可按需以 `examples/` 形式补充进本仓库。核心代码段在本页已完整给出,可直接照抄到普通 `addEventListener` 型页面验证。

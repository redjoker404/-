// ==UserScript==
// @name         智慧树·学习资源 视频自动连播
// @namespace    zhs.wenda.autonext
// @version      1.4.0
// @description  视频自然播完后自动切到下一个资源（支持跨栏目/跨章节，自动跳过已看完的）；遇到"文件未发布或已删除"的空节自动跳过；纯目录页自动进入第一个没看完的视频；自动找不着时点一次"示教"，之后让它按你的习惯点。不跳进度、不改倍速、不伪造学习时长、不做任何反检测。
// @author       DSH
// @match        https://hike.zhihuishu.com/*
// @match        https://wenda.zhihuishu.com/*
// @match        https://*.zhihuishu.com/*
// @run-at       document-idle
// @grant        GM_registerMenuCommand
// @grant        GM_getValue
// @grant        GM_setValue
// @noframes
// ==/UserScript==

/*
 * ================================= 它是怎么工作的 =================================
 *
 * 一、判断"这个视频真的播完了"
 *     - 主判据：<video> 的 ended 事件
 *     - 兜底：轮询 currentTime 距 duration ≤ 0.5 秒
 *     - 只认"自然播放到结尾"。绝不 seek、绝不改倍速、绝不静音偷跑。
 *
 * 二、"下一个"从哪来（逐级兜底，全自动失败时靠"示教"）
 *     ① 你示教过的容器里，"正在播放的那一项"的下一个兄弟
 *     ② 页面上写着"下一个 / 下一节 / 下一讲 / 继续学习"的按钮
 *     ③ 通用启发式：找到带 active/current/playing 样式的列表项，点它后面的兄弟
 *     ④ 视频在 iframe/新窗口里时，通知列表页去点
 *     ⑤ 全都不行 → 面板上亮红字，提示你点【示教】
 *
 * 三、遇到平台弹题 / 安全验证时
 *     脚本会主动停下来，不会替你答题、不会去关弹窗，等你手动处理完再继续。
 *     这是有意的：那类弹窗本来就是给人看的。
 *
 * 四、排查
 *     控制台输入 __ZHS状态() 看完整状态；面板上【诊断】按钮会把同样的内容复制到剪贴板。
 *     如果"自动下一个"点错了地方，点【清掉示教】重新来一次。
 *
 * 五、版本记录
 *     v1.3.2  修：章节节点是 id="folder_16364939"、视频节点是 id="file_35009241"。
 *             原来"下一项"会爬到章节标题上，点的是展开箭头 icon-arrow → 永远换不了视频
 *             （日志表现为"所有候选都点了，页面还是没变，期间请求数 0"）。
 *             现在按所有 file_ 节点的文档顺序取下一个视频，并显式排除 folder_ 节点。
 *     v1.3.1  修：取 fileId 时优先认目录项的 id="file_35009241"，不再被 data-id 带偏
 *             （v1.3.0 会拼出 16364939 这种不存在的 fileId，跳到"文件未发布或已删除"的空节）；
 *             空节跳过改为先按目录 DOM 顺序取下一项，再退回按编号猜。
 */

(function () {
  'use strict';

  /* ============================ 0. 常量与状态 ============================ */

  var VERSION = '1.4.0';
  var LOG = '%c[连播]';
  var LOG_CSS = 'color:#409eff;font-weight:bold';
  var IS_TOP = (function () { try { return window.top === window.self; } catch (e) { return false; } })();

  // 每种课单独记配置，换个课程不会互相干扰
  var CID = (function () {
    try {
      var u = new URL(location.href);
      return u.searchParams.get('courseId') || u.searchParams.get('recruitAndCourseId') || 'default';
    } catch (e) { return 'default'; }
  })();
  var PREFIX = 'zhsWenda.' + CID + '.';

  // 当前在看的那个资源的 id（智慧树 hike 页是 ?courseId=..&fileId=..，切文件只换 fileId）
  var FILE_ID = (function () {
    try {
      var u = new URL(location.href);
      return u.searchParams.get('fileId') || u.searchParams.get('file_id') ||
        u.searchParams.get('videoId') || u.searchParams.get('resourceId') || '';
    } catch (e) { return ''; }
  })();

  function kvGet(k, def) {
    try {
      if (typeof GM_getValue === 'function') {
        var v = GM_getValue(PREFIX + k, undefined);
        if (v !== undefined && v !== null && v !== '') return v;
      }
    } catch (e) { }
    try {
      var s = localStorage.getItem(PREFIX + k);
      return s === null ? def : s;
    } catch (e) { return def; }
  }
  function kvSet(k, v) {
    try { if (typeof GM_setValue === 'function') { GM_setValue(PREFIX + k, v); return; } } catch (e) { }
    try { localStorage.setItem(PREFIX + k, String(v)); } catch (e) { }
  }

  var S = {
    enabled: String(kvGet('en', '1')) !== '0',
    delay: Math.max(0, Math.min(30, Number(kvGet('delay', '2')) || 0)),
    autoPlay: String(kvGet('autoPlay', '1')) !== '0',
    teach: null,          // { containerSel, itemSig, index, text, href }
    teaching: false,
    fired: false,         // 当前这个视频是否已经触发过"下一个"
    pending: 0,           // 倒计时 setInterval 的 id
    blocked: false,       // 是否有弹题/验证挡住
    blockedText: '',
    video: null,
    logs: [],
    lastHref: location.href,
    lastMsg: '',
    sentPlay: 0,
    switchAt: 0,          // 上次成功切换的时间（自动开播的宽限期用）
    verifyId: 0,          // 点击后"到底有没有切成功"的核查定时器
    tries: [],            // 这一轮已经试过哪些可点元素
    listJumped: false,    // 目录页是否已经自动进过一次（防止在列表页反复跳）
    listAt: 0             // 目录页加载完成的时刻，用来等它渲染稳
  };

  try {
    var t = kvGet('teach', '');
    if (t) S.teach = JSON.parse(t);
  } catch (e) { S.teach = null; }

  function log() {
    var a = Array.prototype.slice.call(arguments);
    try { console.log.apply(console, [LOG, LOG_CSS].concat(a)); } catch (e) { }
    var line = a.map(function (x) { try { return typeof x === 'object' ? JSON.stringify(x) : String(x); } catch (e) { return '?'; } }).join(' ');
    S.logs.push(new Date().toLocaleTimeString() + ' ' + line);
    if (S.logs.length > 80) S.logs.shift();
  }

  /* ============================ 1. 页面上的小提示条 ============================ */

  var toastEl = null, toastTimer = 0;
  function toast(msg, keep) {
    try {
      S.lastMsg = msg;
      if (!document.body) return;
      if (!toastEl) {
        toastEl = document.createElement('div');
        toastEl.id = 'zhs-toast';
        toastEl.style.cssText = 'position:fixed;left:50%;top:18px;transform:translateX(-50%);z-index:2147483646;' +
          'max-width:76vw;background:rgba(18,18,20,.94);color:#e8f4ff;font:13px/1.6 system-ui,"Microsoft YaHei";' +
          'padding:9px 16px;border-radius:10px;box-shadow:0 8px 26px rgba(0,0,0,.45);pointer-events:none;' +
          'border:1px solid rgba(90,170,255,.35);text-align:center';
        document.body.appendChild(toastEl);
      }
      toastEl.textContent = msg;
      toastEl.style.display = 'block';
      clearTimeout(toastTimer);
      if (!keep) toastTimer = setTimeout(function () { try { toastEl.style.display = 'none'; } catch (e) { } }, 3200);
    } catch (e) { }
  }

  /* ============================ 2. 通用 DOM 工具 ============================ */

  function docs() {
    var out = [document];
    (function walk(doc) {
      var fs;
      try { fs = doc.querySelectorAll('iframe,frame'); } catch (e) { return; }
      for (var i = 0; i < fs.length; i++) {
        try {
          var d = fs[i].contentDocument;
          if (d && out.indexOf(d) < 0) { out.push(d); walk(d); }
        } catch (e) { /* 跨域，跳过 */ }
      }
    })(document);
    return out;
  }

  function visible(el) {
    try {
      if (!el || !el.getBoundingClientRect) return false;
      var r = el.getBoundingClientRect();
      if (r.width < 2 || r.height < 2) return false;
      var win = null;
      try { win = el.ownerDocument && el.ownerDocument.defaultView; } catch (e) { win = null; }
      if (!win) { try { win = window; } catch (e) { win = null; } }
      var st = (win && win.getComputedStyle) ? win.getComputedStyle(el) : null;
      if (st) {
        if (st.display === 'none' || st.visibility === 'hidden' || Number(st.opacity) === 0) return false;
      }
      return true;
    } catch (e) { return false; }
  }

  function txt(el) {
    try { return String(el.textContent || '').replace(/\s+/g, ' ').trim(); } catch (e) { return ''; }
  }

  var ESC_OK = (function () { try { return typeof CSS !== 'undefined' && typeof CSS.escape === 'function'; } catch (e) { return false; } })();
  function esc(s) {
    s = String(s);
    if (ESC_OK) { try { return CSS.escape(s); } catch (e) { } }
    return s.replace(/[^\w-]/g, function (c) { return '\\' + c; });
  }

  // 给元素算一条尽量短又能唯一定位的 CSS 路径
  function cssPath(el) {
    if (!el || el.nodeType !== 1) return '';
    if (el.id) return '#' + esc(el.id);
    var parts = [];
    var cur = el;
    while (cur && cur.nodeType === 1 && parts.length < 6) {
      if (cur.id) { parts.unshift('#' + esc(cur.id)); break; }
      var sel = cur.tagName.toLowerCase();
      var cls = '';
      try { cls = typeof cur.className === 'string' ? cur.className : ''; } catch (e) { cls = ''; }
      var cl = cls.trim().split(/\s+/).filter(Boolean).slice(0, 2);
      if (cl.length) sel += cl.map(function (c) { return '.' + esc(c); }).join('');
      var p = cur.parentElement;
      if (p) {
        var same = [], k;
        for (k = 0; k < p.children.length; k++) if (p.children[k].tagName === cur.tagName) same.push(p.children[k]);
        if (same.length > 1) sel += ':nth-of-type(' + (same.indexOf(cur) + 1) + ')';
      }
      parts.unshift(sel);
      cur = cur.parentElement;
    }
    return parts.join(' > ');
  }

  // 在元素内部找"最像可点"的那个后代：a/button/role=button/onclick/cursor:pointer
  // 智慧树的目录项外层是 li，真正带点击事件的是里面的 span —— 点 li 是没用的
  function pointerTarget(el) {
    if (!el || el.nodeType !== 1) return null;
    var best = null, bestD = -1, seen = 0;
    var queue = [{ n: el, d: 0 }];
    while (queue.length && seen < 90) {
      var cur = queue.shift(), n = cur.n, d = cur.d;
      seen++;
      if (!n || n.nodeType !== 1) continue;
      // 章节标题上的展开箭头（folder_xxx 里的 i.icon-arrow）不算可点目标：点它只会展开/收起
      if (/icon-arrow|folder-box|icon-expand|icon-collapse/i.test(String(n.className || ''))) continue;
      var hit = false;
      try {
        var tag = (n.tagName || '').toLowerCase();
        if (tag === 'a' || tag === 'button') hit = true;
        else if (n.getAttribute && n.getAttribute('role') === 'button') hit = true;
        else if (n.onclick) hit = true;
        if (!hit) {
          var win = (n.ownerDocument && n.ownerDocument.defaultView) || window;
          if (win && win.getComputedStyle) {
            var st = win.getComputedStyle(n);
            if (st && st.cursor === 'pointer') hit = true;
          }
        }
      } catch (e) { }
      if (hit && d > bestD) { best = n; bestD = d; }
      if (d < 5) {
        for (var i = 0; i < n.children.length; i++) queue.push({ n: n.children[i], d: d + 1 });
      }
    }
    return best;
  }

  // 从被点的元素里挑出"真正该点"的那个
  function clickable(el) {
    if (!el) return null;
    var pt = pointerTarget(el);
    if (pt) return pt;
    var t = el;
    for (var i = 0; i < 4 && t; i++) {
      var tag = (t.tagName || '').toLowerCase();
      if (tag === 'a' || tag === 'button' || t.getAttribute && t.getAttribute('role') === 'button') return t;
      if (t.onclick) return t;
      t = t.parentElement;
    }
    var inner = null;
    try { inner = el.querySelector('a,button,[role="button"]'); } catch (e) { }
    return inner || el;
  }

  function fire(el) {
    try {
      if (!el) return false;
      var r = el.getBoundingClientRect ? el.getBoundingClientRect() : { left: 1, top: 1 };
      var opt = { bubbles: true, cancelable: true, view: window, clientX: r.left + 3, clientY: r.top + 3 };
      try { el.dispatchEvent(new MouseEvent('mouseover', opt)); } catch (e) { }
      try { el.dispatchEvent(new MouseEvent('mousedown', opt)); } catch (e) { }
      try { el.dispatchEvent(new MouseEvent('mouseup', opt)); } catch (e) { }
      try { el.dispatchEvent(new MouseEvent('click', opt)); } catch (e) { }
      // 有些前端只监听 pointer 系列事件
      try {
        if (typeof PointerEvent === 'function') {
          el.dispatchEvent(new PointerEvent('pointerdown', opt));
          el.dispatchEvent(new PointerEvent('pointerup', opt));
        }
      } catch (e) { }
      try { if (typeof el.click === 'function' && !(el instanceof SVGElement)) el.click(); } catch (e) { }
      return true;
    } catch (e) {
      try { el.click(); return true; } catch (e2) { return false; }
    }
  }

  /* ---------------- 让视频动起来 ---------------- */

  // 找播放器上那个"播放"按钮（阿里/自定义播放器都常见 prism-play-btn 这类类名）
  function findPlayBtn() {
    var ds = docs();
    var re = /(^|[-_])(prism-)?(play|start)([-_]?(btn|button|icon|bar|mask))?$/i;
    for (var i = 0; i < ds.length; i++) {
      var ns;
      try { ns = ds[i].querySelectorAll('button,a,[role="button"],div,span,i'); } catch (e) { continue; }
      for (var j = 0; j < ns.length && j < 4000; j++) {
        var el = ns[j];
        if (isOurs(el)) continue;
        var toks = clsTokens(el), k, okBtn = false;
        for (k = 0; k < toks.length; k++) if (re.test(toks[k])) { okBtn = true; break; }
        if (!okBtn) continue;
        if (!visible(el)) continue;
        return el;
      }
    }
    return null;
  }

  function tryPlayNow(reason) {
    var v = S.video || findVideo();
    if (!v) return false;
    var ok = false;
    try {
      var p = v.play();
      ok = true;
      if (p && p.catch) p.catch(function (e) {
        log('浏览器拒绝了自动播放（' + (e && e.name) + '），等你点一下播放器');
      });
    } catch (e) { }
    if (!ok || v.paused) {
      var b = findPlayBtn();
      if (b) { log('点播放器上的播放按钮'); fire(b); if (reason) toast('已尝试点播放按钮'); }
    }
    // 兜底：直接点视频画面（很多播放器是"点画面播放/暂停"）
    if (v.paused) { try { fire(v); } catch (e) { } }
    return true;
  }

  /* ============================ 3. 视频探测 ============================ */

  function findVideo() {
    var best = null, bestScore = -Infinity;
    var ds = docs();
    for (var i = 0; i < ds.length; i++) {
      var vs;
      try { vs = ds[i].querySelectorAll('video'); } catch (e) { continue; }
      for (var j = 0; j < vs.length; j++) {
        var v = vs[j];
        var score = 0;
        try {
          var r = v.getBoundingClientRect();
          score = Math.max(1, r.width) * Math.max(1, r.height);   // 面积越大越可能是正片
          if (v.duration > 0) score += 1e7;                       // 有时长的是真视频
          if (!v.paused) score += 5e7;                            // 正在播的优先
          if (v.readyState < 1) score -= 1e6;                     // 还没元数据的靠后
        } catch (e) { }
        if (score > bestScore) { bestScore = score; best = v; }
      }
    }
    return best;
  }

  var bound = (function () { try { return new WeakSet(); } catch (e) { return null; } })();
  function isBound(v) {
    if (bound) return bound.has(v);
    return !!v.__zhsBound;
  }
  function markBound(v) {
    if (bound) bound.add(v); else v.__zhsBound = true;
  }

  function bindVideo(v) {
    if (!v || isBound(v)) return;
    markBound(v);
    log('接管视频：时长', Math.round(v.duration || 0), '秒');
    var onEnd = function () { onVideoEnded(v, 'ended 事件'); };
    try { v.addEventListener('ended', onEnd); } catch (e) { }
    try {
      v.addEventListener('timeupdate', function () {
        try {
          if (!v.duration || v.paused) return;
          if (v.currentTime >= v.duration - 0.5 && !S.fired) onVideoEnded(v, '接近结尾');
        } catch (e) { }
      });
    } catch (e) { }
    try {
      v.addEventListener('play', function () { S.fired = false; setStatus('播放中'); });
    } catch (e) { }
  }

  function onVideoEnded(v, why) {
    if (!S.enabled) return;
    if (S.fired) return;
    S.fired = true;
    log('视频播完（' + why + '）');
    setStatus('已播完，准备切下一个');
    scheduleNext(why);
  }

  /* ============================ 4. 弹题 / 验证检测 ============================ */

  var BLOCK_TEXT = /请(回答|选择|完成|作答)|题目|答题|验证码|安全验证|拖动滑块|人脸|拍照|点击验证|确认是本人/;
  var MODAL_SEL = '.el-dialog__wrapper,.el-message-box__wrapper,.ivu-modal-wrap,.ant-modal-wrap,' +
    '[class*="dialog"],[class*="modal"],[class*="popup"],[class*="verif"],' +
    '[class*="questions"],[class*="question-"],[class*="topic"]';

  function detectBlock() {
    var ds = docs();
    for (var i = 0; i < ds.length; i++) {
      var ns;
      try { ns = ds[i].querySelectorAll(MODAL_SEL); } catch (e) { continue; }
      for (var j = 0; j < ns.length; j++) {
        var el = ns[j];
        if (el.id === 'zhs-panel' || (el.closest && el.closest('#zhs-panel'))) continue;
        if (!visible(el)) continue;
        var t = txt(el);
        if (t.length > 900) continue;
        if (BLOCK_TEXT.test(t)) {
          S.blocked = true; S.blockedText = t.slice(0, 120);
          return true;
        }
      }
    }
    if (S.blocked) { S.blocked = false; S.blockedText = ''; }
    return false;
  }

  /* ============================ 5. 找"下一个" ============================ */

  var ACTIVE_CLS = /(^|[-_])(active|current|playing|selected|chosen|focus)([-_]|$)/i;
  function clsTokens(el) {
    try {
      var c = typeof el.className === 'string' ? el.className : '';
      return c.trim().split(/\s+/).filter(Boolean);
    } catch (e) { return []; }
  }
  function looksActive(el) {
    try {
      var t = clsTokens(el), i;
      for (i = 0; i < t.length; i++) if (ACTIVE_CLS.test(t[i])) return true;
      var inner = el.querySelectorAll('*');
      for (i = 0; i < inner.length && i < 30; i++) {
        var it = clsTokens(inner[i]), k;
        for (k = 0; k < it.length; k++) if (ACTIVE_CLS.test(it[k])) return true;
      }
    } catch (e) { }
    return false;
  }

  function nextSiblingOf(list, idx) {
    if (!list || idx < 0 || idx >= list.length - 1) return null;
    for (var k = idx + 1; k < list.length; k++) {
      if (visible(list[k])) return list[k];
    }
    return null;
  }

  // ① 用示教记下的容器 + 当前项
  function fromTeach() {
    if (!S.teach) return null;
    var ds = docs();
    for (var i = 0; i < ds.length; i++) {
      var c = null;
      try { c = ds[i].querySelector(S.teach.containerSel); } catch (e) { c = null; }
      if (!c || c.children.length < 2) continue;
      var kids = Array.prototype.slice.call(c.children);
      var idx = -1, k;
      for (k = 0; k < kids.length; k++) { if (looksActive(kids[k])) { idx = k; break; } }
      if (idx < 0 && typeof S.teach.index === 'number' && S.teach.index < kids.length) idx = S.teach.index;
      if (idx < 0) continue;
      var nx = nextSiblingOf(kids, idx);
      if (nx) return { el: clickable(nx) || nx, how: '示教容器第 ' + (idx + 2) + ' 项' };
    }
    return null;
  }

  // ② 用 URL 里的 fileId 找到"当前这一项"，再点它后面的兄弟
  //    —— 智慧树 hike 的目录列表基本就是 <li><a href="?fileId=xxx">…</a></li>，这条最稳
  var NEXT_TXT = /^(下一个|下一节|下一讲|下一课|下一集|下一章|下一条|继续学习|继续观看|下一个视频|下一个资源|播放下一)/;

  function currentIds() {
    var out = FILE_ID ? [FILE_ID] : [];
    try {
      var u = new URL(location.href);
      ['fileId', 'file_id', 'videoId', 'video_id', 'resourceId', 'sectionId', 'chapterId'].forEach(function (k) {
        var v = u.searchParams.get(k);
        if (v && out.indexOf(v) < 0) out.push(v);
      });
    } catch (e) { }
    return out.filter(Boolean);
  }

  function fromCurrentId() {
    var ids = currentIds();
    if (!ids.length) return null;
    var ds = docs();
    for (var i = 0; i < ds.length; i++) {
      var d = ds[i], anchor = null, w, s;
      for (w = 0; w < ids.length && !anchor; w++) {
        var id = ids[w];
        var sels = [
          // 智慧树目录项的真实标记就是 id="file_35009241"，优先认它
          '[id="file_' + id + '"]', '[id$="' + id + '"]', '[id*="' + id + '"]',
          '[data-file-id="' + id + '"]', '[data-fileid="' + id + '"]',
          '[data-id="' + id + '"]', '[data-resource-id="' + id + '"]',
          '[href*="' + id + '"]', '[onclick*="' + id + '"]'
        ];
        for (s = 0; s < sels.length && !anchor; s++) {
          try { anchor = d.querySelector(sels[s]); } catch (e) { anchor = null; }
          if (anchor && isOurs(anchor)) anchor = null;
        }
      }
      if (!anchor) continue;

      // 从"当前项"往上找列表容器，再拿它的下一个兄弟
      var node = anchor;
      for (var up = 0; up < 6 && node && node.parentElement; up++) {
        var p = node.parentElement;
        var kids = Array.prototype.slice.call(p.children);
        if (kids.length < 2) { node = p; continue; }
        var sameTag = kids.filter(function (x) { return x.tagName === node.tagName; });
        var idx = sameTag.indexOf(node);
        if (sameTag.length < 2 || sameTag.length > 400 || idx < 0) { node = p; continue; }
        var nx = nextSiblingOf(sameTag, idx);
        if (!nx || isOurs(nx) || isFolderish(nx)) { node = p; continue; }
        var t = txt(nx);
        if (!t || t.length > 200) { node = p; continue; }
        return { el: clickable(nx) || nx, how: '按 fileId=' + (FILE_ID || '?') + ' 定位到下一项"' + t.slice(0, 20) + '"' };
      }
    }
    return null;
  }

  // ③ 页面上明写着"下一个"的按钮
  function fromButton() {
    var ds = docs();
    for (var i = 0; i < ds.length; i++) {
      var ns;
      try { ns = ds[i].querySelectorAll('button,a,[role="button"],div,span,li'); } catch (e) { continue; }
      for (var j = 0; j < ns.length; j++) {
        var el = ns[j];
        if (el.id === 'zhs-panel' || (el.closest && el.closest('#zhs-panel'))) continue;
        var t = txt(el);
        if (!t || t.length > 14) continue;
        if (!NEXT_TXT.test(t)) continue;
        if (!visible(el)) continue;
        return { el: clickable(el) || el, how: '按钮"' + t + '"' };
      }
    }
    return null;
  }

  // ④ 通用启发式：找"自己带 active 样式"的列表项，点它后面的兄弟
  function selfActive(el) {
    var t = clsTokens(el);
    for (var i = 0; i < t.length; i++) if (ACTIVE_CLS.test(t[i])) return true;
    return false;
  }

  function isOurs(el) {
    try {
      if (!el) return true;
      if (el.id === 'zhs-panel' || el.id === 'zhs-toast') return true;
      if (el.closest && el.closest('#zhs-panel')) return true;
      if (el.querySelector && el.querySelector('#zhs-panel')) return true;
    } catch (e) { }
    return false;
  }

  function fromHeuristic() {
    var ds = docs();
    for (var i = 0; i < ds.length; i++) {
      var all;
      try { all = ds[i].querySelectorAll('*'); } catch (e) { continue; }
      for (var j = 0; j < all.length && j < 5000; j++) {
        var el = all[j];
        if (isOurs(el)) continue;
        if (!selfActive(el)) continue;

        // 从 active 元素往上找一层"像个列表"的父容器
        var node = el;
        for (var up = 0; up < 3 && node && node.parentElement; up++) {
          var p = node.parentElement;
          var kids = Array.prototype.slice.call(p.children);
          if (kids.length < 2) { node = p; continue; }
          var sameTag = kids.filter(function (x) { return x.tagName === node.tagName; });
          if (sameTag.length < 2 || sameTag.length > 300) { node = p; continue; }
          var idx = sameTag.indexOf(node);
          var nx = nextSiblingOf(sameTag, idx);
          if (!nx || isOurs(nx) || isFolderish(nx)) { node = p; continue; }
          var t = txt(nx);
          if (!t || t.length > 200) { node = p; continue; }
          return { el: clickable(nx) || nx, how: '列表下一项"' + t.slice(0, 24) + '"' };
        }
      }
    }
    return null;
  }

  // 章节/文件夹节点：智慧树标的是 id="folder_16364939"，里面第一个按钮是展开箭头 icon-arrow。
  // 一定要排掉，否则"下一项"会指向章节标题 → 点箭头只会展开/收起，绝不会换视频（就是"点了没反应"）
  function isFolderish(el) {
    try {
      if (!el) return false;
      if (/^folder[-_]/i.test(String(el.id || ''))) return true;
      if (el.closest) {
        if (el.closest('[id^="folder_"]') && !el.closest('[id^="file_"]')) return true;
      }
    } catch (e) { }
    return false;
  }

  // "这一行视频看完了没有"——右侧那个绿勾 vs 空心圆
  // 绿勾常见的类名：icon-finish / icon-complete / icon-success / checked / done / hasLearn…
  // 空心圆常见的类名：icon-circle / unfinish / notstart / undo …
  function isDoneRow(el) {
    var s = '', i, ns;
    try {
      if (typeof el.className === 'string') s += ' ' + el.className;
      ns = el.querySelectorAll('i,span,em,b,svg,div');
      for (i = 0; i < ns.length && i < 60; i++) s += ' ' + String(ns[i].className || '');
    } catch (e) { }
    if (/(unfinish|unfinished|uncompleted|undo|notstart|no-start|notlearn|nostudy|circle|empty|no-finish)/i.test(s)) return false;
    return /(finish|finished|completed|complete|checked|check-|success|passed|done|studied|learned|haslearn|has-learn)/i.test(s);
  }

  // 页面上"所有视频行"的文档顺序 —— 跨栏目、跨章节。
  // 智慧树的视频节点是 id="file_35009241"，章节节点是 id="folder_16364939"（必须排除）。
  // 即便某个栏目是折叠的、节点还没渲染出来，只要它在 DOM 里就一样能排上。
  function videoRows() {
    var out = [], ds = docs(), d, i, j, ns;
    for (d = 0; d < ds.length; d++) {
      try { ns = ds[d].querySelectorAll('[id^="file_"]'); } catch (e) { continue; }
      for (i = 0; i < ns.length && out.length < 600; i++) {
        var el = ns[i];
        if (isOurs(el)) continue;
        var m = String(el.id || '').match(/^file[-_]?(\d{5,})$/i);
        if (!m) continue;
        if (!txt(el)) continue;
        out.push({ el: el, fid: m[1], done: isDoneRow(el) });
      }
    }
    if (out.length) return out;

    // 退路：这个页面没有 file_ 标记 —— 退而求其次，把"最内层含 .mp4 的小块"当视频行
    for (d = 0; d < ds.length; d++) {
      var cands;
      try { cands = ds[d].querySelectorAll('li,div,tr,a,span,p'); } catch (e) { continue; }
      for (j = 0; j < cands.length && out.length < 400; j++) {
        var e2 = cands[j];
        if (isOurs(e2)) continue;
        var t = txt(e2);
        if (!t || t.length > 220 || t.indexOf('.mp4') < 0) continue;
        var hasInner = false;
        try {
          for (var k = 0; k < e2.children.length; k++) if (txt(e2.children[k]).indexOf('.mp4') >= 0) { hasInner = true; break; }
        } catch (e) { }
        if (hasInner) continue;                       // 只取最内层，免得把整页当成一行
        out.push({ el: e2, fid: fidOf(e2), done: isDoneRow(e2) });
      }
    }
    return out;
  }

  // 播放页的标准地址（列表页没有 fileId 参数时用它跳过去）
  function playerUrl(fid) {
    return 'https://hike.zhihuishu.com/aidedteaching/sourceLearning/sourceLearning?courseId=' +
      encodeURIComponent(CID) + '&fileId=' + encodeURIComponent(fid);
  }

  function fromFileOrder() {
    var list = videoRows();
    if (list.length < 2) return null;
    var i = -1, k;
    if (FILE_ID) {
      for (k = 0; k < list.length; k++) if (list[k].fid === FILE_ID) { i = k; break; }
    }
    // 当前这一节不在列表里（比如刚进播放页、目录还没渲染完）→ 退化成"第一个没看完的"
    var pick = -1;
    for (k = i + 1; k < list.length; k++) if (!list[k].done) { pick = k; break; }
    if (pick < 0 && i >= 0) pick = i + 1;                       // 后面都看完了，就取紧挨着的下一个
    if (pick < 0) for (k = 0; k < list.length; k++) if (!list[k].done) { pick = k; break; }
    if (pick < 0 || pick >= list.length) return null;
    if (i >= 0 && pick <= i) return null;                       // 别往回走

    var it = list[pick], t = txt(it.el);
    var missed = 0;
    if (i >= 0 && pick > i + 1) missed = pick - i - 1;          // 中间跳过了几个已经看完的
    return {
      el: clickable(it.el) || it.el,
      fid: it.fid,
      how: '目录第 ' + (pick + 1) + '/' + list.length + ' 个' +
        (missed > 0 ? '（跳过 ' + missed + ' 个已完成）' : '') +
        '「' + t.slice(0, 14) + '」'
    };
  }

  function findNext() {
    return fromTeach() || fromFileOrder() || fromCurrentId() || fromButton() || fromHeuristic();
  }

  /* ---------------- 失效资源（"文件未发布或已删除"）的处理 ---------------- */

  var DEAD_TXT = /文件未发布或已删除|文件不存在|视频不存在|资源不存在|已被删除|文件已删除|暂无文件|资源已下线/;

  function deadText() {
    try {
      var b = document.body;
      if (!b) return false;
      var t = b.innerText || b.textContent || '';
      try { if (panel && panel.innerText) t = t.split(panel.innerText).join(''); } catch (e) { }
      return DEAD_TXT.test(t);
    } catch (e) { return false; }
  }

  // 按 DOM 顺序收集页面上所有"资源 id"。跳过失效资源、以及没法靠点击时的直接跳转全靠它
  function collectIds() {
    var out = [], seen = {};
    function add(v) {
      v = String(v == null ? '' : v).replace(/\D/g, '');
      if (v.length < 5 || seen[v]) return;
      seen[v] = 1;
      out.push(v);
    }
    var ds = docs();
    for (var i = 0; i < ds.length; i++) {
      var d = ds[i], ns;
      try {
        ns = d.querySelectorAll('[data-id],[data-file-id],[data-fileid],[data-resource-id],[href*="fileId"],[id^="file_"]');
      } catch (e) { continue; }
      for (var j = 0; j < ns.length && out.length < 800; j++) {
        var el = ns[j];
        if (isOurs(el)) continue;
        // 目录项的 id 形如 file_35009241 —— 这是最靠得住的 fileId 来源
        var idm = String(el.id || '').match(/^file[-_]?(\d{5,})$/i);
        if (idm) add(idm[1]);
        add(el.getAttribute('data-file-id') ||
          el.getAttribute('data-fileid') || el.getAttribute('data-resource-id'));
        var m = String(el.getAttribute('href') || '').match(/fileId=(\d+)/i);
        if (m) add(m[1]);
        // data-id 最后再说（它可能是视频 id，不是 fileId）
        var d2 = el.getAttribute('data-id');
        if (d2 && sameSpace(d2)) add(d2);
      }
      // 兜底：DOM 上没带属性时，从整页 HTML 里抠 fileId
      if (out.length < 3) {
        var html = '';
        try { html = d.documentElement ? d.documentElement.outerHTML : ''; } catch (e) { html = ''; }
        var re = /fileId["']?\s*[:=]\s*["']?(\d{6,})/gi, mm, guard = 0;
        while ((mm = re.exec(html)) && guard++ < 800) add(mm[1]);
      }
    }
    return out;
  }

  // 记住"当前这一节在目录里的第几项"：万一下一节是失效文件，靠它才能接着往后跳
  function saveIdx() {
    try {
      var ids = collectIds();
      var i = FILE_ID ? ids.indexOf(FILE_ID) : -1;
      if (i >= 0) kvSet('zhsIdx', String(i));
    } catch (e) { }
  }

  function handleDead() {
    if (!deadText()) return;
    var now = Date.now();
    if (now - (S.deadAt || 0) < 2500) return;      // 别连着跳
    S.deadAt = now;

    var ids = collectIds();

    // 记下"已经确认失效的"，避免在原地来回跳
    var seenRaw = String(kvGet('zhsDeadSeen', '') || '');
    var seen = seenRaw ? seenRaw.split(',') : [];
    if (FILE_ID && seen.indexOf(FILE_ID) < 0) {
      seen.push(FILE_ID);
      kvSet('zhsDeadSeen', seen.slice(-40).join(','));
    }

    // ① 最可靠：直接按 DOM 顺序找"当前这一项后面的那一项"（不依赖任何 id 编号规律）
    var hit = null;
    try { hit = findNext(); } catch (e) { hit = null; }
    if (hit && hit.el) {
      var fidNext = '';
      try { fidNext = hit.fid || fidOf(hit.el); } catch (e) { fidNext = ''; }
      var u1 = '';
      try { u1 = directUrlFor(hit.el, fidNext); } catch (e) { u1 = ''; }
      if (u1 && u1 !== location.href && /[?&]fileId=\d+/i.test(u1)) {
        S.lastResult = '跳过失效文件 → fileId=' + fidNext;
        log('这一节失效（页面提示"文件未发布或已删除"），按目录顺序跳到 fileId=' + (fidNext || '?'));
        setStatus('这一节失效，跳到下一个…');
        toast('这一节"文件未发布或已删除"，自动跳到下一个');
        try { location.href = u1; } catch (e) { }
        return;
      }
    }

    if (!ids.length) {
      setStatus('这一节已失效，但页面上找不到目录 id');
      toast('这一节"文件未发布或已删除"，而且没找到别的视频。点【诊断】发我', true);
      return;
    }

    var idx = FILE_ID ? ids.indexOf(FILE_ID) : -1;
    if (idx < 0) idx = Number(kvGet('zhsIdx', '-1'));
    var k = idx + 1;
    while (k > 0 && k < ids.length && seen.indexOf(ids[k]) >= 0) k++;

    if (idx < 0 || k >= ids.length) {
      setStatus('这一节已失效，后面没有可跳的了');
      toast('这一节"文件未发布或已删除"，而且后面没有别的了。手动点下一节吧', true);
      return;
    }
    var skip = Number(kvGet('zhsDeadSkip', '0')) + 1;
    if (skip > 10) {
      setStatus('连续跳过 ' + (skip - 1) + ' 个失效文件，先停下');
      toast('连续好几个文件都失效，先停下，免得乱跳', true);
      return;
    }
    kvSet('zhsDeadSkip', String(skip));
    kvSet('zhsIdx', String(k));

    var next = ids[k];
    var u = '';
    try { u = location.href.replace(/([?&]fileId=)\d+/i, '$1' + next); } catch (e) { u = ''; }
    if (!urlFid() || u === location.href) {
      setStatus('这一节失效，但地址栏没有 fileId，没法自动跳过');
      toast('这一节失效。地址栏里没有 fileId，只能手动点下一节了', true);
      return;
    }
    S.lastResult = '跳过失效文件 → fileId=' + next;
    log('这一节失效（页面提示"文件未发布或已删除"），跳到第 ' + (k + 1) + ' 项 fileId=' + next);
    setStatus('这一节失效，跳到下一个…');
    toast('这一节"文件未发布或已删除"，自动跳到下一个');
    try { location.href = u; } catch (e) { }
  }

  /* ---------------- 目录页（没有播放器）→ 自动进入第一个没看完的视频 ---------------- */

  // 有些课程页只有目录、没有播放器（或者目录和播放器是两个页面）。
  // 这种情况下：等目录渲染稳 → 找到"已经看完的最后一个"之后的第一个没看完的 → 直接跳进去。
  function handleListPage() {
    if (S.listJumped || S.video) return;
    if (!/courseId=\d+/i.test(String(location.search || ''))) return;
    var rows = videoRows();
    if (rows.length < 2) return;
    if (!S.listAt) { S.listAt = Date.now(); return; }
    if (Date.now() - S.listAt < 3000) return;            // 等目录渲染稳，别抢
    try { if (document.hidden) return; } catch (e) { }    // 后台标签页别乱跳

    var cur = -1, i, fid = urlFid();
    if (fid) for (i = 0; i < rows.length; i++) if (rows[i].fid === fid) { cur = i; break; }
    var pick = -1;
    for (i = cur + 1; i < rows.length; i++) if (!rows[i].done) { pick = i; break; }
    if (pick < 0) for (i = 0; i < rows.length; i++) if (!rows[i].done) { pick = i; break; }
    if (pick < 0) { S.listJumped = true; setStatus('目录里 ' + rows.length + ' 个视频都已完成'); return; }

    var row = rows[pick], t = txt(row.el);
    S.listJumped = true;
    var u = '';
    try { u = directUrl(row.fid) || (row.fid ? playerUrl(row.fid) : ''); } catch (e) { u = ''; }
    if (u) {
      S.lastResult = '目录页自动进入第 ' + (pick + 1) + ' 个（fileId=' + row.fid + '）';
      log('目录页：' + rows.length + ' 个视频，自动进入第 ' + (pick + 1) + ' 个 fileId=' + row.fid);
      setStatus('目录页：自动进入第 ' + (pick + 1) + '/' + rows.length + ' 个');
      toast('目录页：自动进入第 ' + (pick + 1) + ' 个「' + t.slice(0, 12) + '」');
      try { location.href = u; } catch (e) { }
      return;
    }
    // 读不到 id → 只能试着点它
    var list = clickCandidates(row.el);
    if (list && list.length) {
      S.lastResult = '目录页：点第 ' + (pick + 1) + ' 个';
      setStatus('目录页：点第 ' + (pick + 1) + ' 个「' + t.slice(0, 12) + '」');
      log('目录页：没有 id 可读，改为点击 ' + describe(list[0]));
      fire(list[0]);
    }
  }

  /* ============================ 6. 切下一个的动作 ============================ */

  function setStatus(s) {
    S.lastMsg = s;
    render();
  }

  function scheduleNext(why) {
    if (S.pending) return;
    if (S.delay <= 0) { doNext(); return; }
    var left = S.delay;
    toast('播完（' + why + '），' + left + ' 秒后切下一个…', true);
    setStatus('倒计时 ' + left + ' 秒');
    S.pending = setInterval(function () {
      left--;
      if (left > 0) {
        toast('播完（' + why + '），' + left + ' 秒后切下一个…', true);
        setStatus('倒计时 ' + left + ' 秒');
        return;
      }
      clearInterval(S.pending); S.pending = 0;
      doNext();
    }, 1000);
  }

  // 页面状态的"指纹"：地址 + 视频源 + 时长。变了就说明真的切过去了
  function snap() {
    var v = S.video;
    var src = '';
    try { src = v ? String(v.currentSrc || v.src || '') : ''; } catch (e) { }
    return [location.href, src, v ? Math.round(v.duration || 0) : 0].join('|');
  }

  // 用"真实鼠标落点"找该点的元素：比猜类名/标签可靠得多
  function hitAt(el) {
    try {
      if (!el || !el.getBoundingClientRect) return null;
      var r = el.getBoundingClientRect();
      if (!r || !(r.width > 1 && r.height > 1)) return null;
      var d = el.ownerDocument || document;
      if (!d.elementFromPoint) return null;
      // 取行内偏左的位置：右侧通常是"已完成"对勾，点它会点错
      var x = r.left + Math.min(Math.max(r.width / 2, 6), 60);
      var y = r.top + r.height / 2;
      var n = d.elementFromPoint(x, y);
      if (!n) return null;
      var t = n, i = 0;
      while (t && i++ < 10) {
        if (t === el) return n;                 // 落点在目标行内部 → 点它（事件会冒泡到行上）
        t = t.parentElement;
      }
      return null;                              // 落点被别的元素挡住了
    } catch (e) { return null; }
  }

  // 一次点击可能不够：只试"最像可点"的那几个，绝不乱点外层容器
  // （点容器很危险：有些列表按 event.target 判断点了哪一项，点容器可能跳到第一项）
  function clickCandidates(el) {
    var out = [];
    function push(n) { if (n && n.nodeType === 1 && out.indexOf(n) < 0) out.push(n); }
    push(hitAt(el));
    try { push(pointerTarget(el)); } catch (e) { }
    push(el);
    return out.filter(Boolean).slice(0, 3);
  }

  // 直接改地址栏跳到目标（比点更硬：不依赖 ev.isTrusted）
  function directUrlFor(el, fid) {
    var u = directUrl(fid);
    if (u) return u;
    try {
      var t = el, i = 0;
      while (t && i++ < 4) {
        var h = t.getAttribute && t.getAttribute('href');
        if (h && /fileId=/i.test(h) && typeof URL === 'function') {
          var abs = new URL(h, location.href).href;
          if (abs !== location.href && abs.indexOf('zhihuishu.com') >= 0) return abs;
        }
        t = t.parentElement;
      }
    } catch (e) { }
    return '';
  }

  // 从元素身上挖出它的 fileId
  // 教训：智慧树目录项的 id 就是 "file_35009241"，而 data-id 指的是别的东西（视频 id），
  //       之前优先读 data-id，结果拼出了 16364939 这种不存在的 fileId → 跳到"文件未发布或已删除"。
  function sameSpace(v) {
    v = String(v || '');
    var cur = String(FILE_ID || '');
    if (!/^\d{5,}$/.test(v) || !/^\d{5,}$/.test(cur)) return false;
    return v.length === cur.length && v.slice(0, 4) === cur.slice(0, 4);
  }

  function fidOf(el) {
    try {
      var t = el, i = 0;
      while (t && i++ < 5) {
        // ① 元素自己的 id：file_35009241 / fileid-35009241 之类
        var idm = String(t.id || '').match(/(?:file|fileid|fileId)[-_]?(\d{5,})/i) ||
          String(t.id || '').match(/^(\d{6,})$/);
        if (idm) return idm[1];

        // ② href 里的 fileId=（最权威）
        var h = String((t.getAttribute && t.getAttribute('href')) || '');
        var hm = h.match(/fileId=(\d+)/i);
        if (hm) return hm[1];

        // ③ 明确的 file 属性
        if (t.getAttribute) {
          var v = t.getAttribute('data-file-id') || t.getAttribute('data-fileid') || t.getAttribute('fileid');
          if (v && /^\d{5,}$/.test(String(v))) return String(v);
        }
        t = t.parentElement;
      }

      // ④ 最后才考虑 data-id，而且必须和当前地址栏的 fileId 是同一批号
      t = el;
      var j = 0;
      while (t && j++ < 5) {
        if (t.getAttribute) {
          var d = t.getAttribute('data-id') || t.getAttribute('data-resource-id');
          if (d && sameSpace(d)) return String(d);
        }
        t = t.parentElement;
      }
    } catch (e) { }
    return '';
  }

  function urlFid() {
    var m = String(location.search || '').match(/[?&]fileId=(\d+)/i);
    return m ? m[1] : '';
  }

  // 只有地址栏本来就用 fileId 传参时才敢直接跳
  function directUrl(fid) {
    if (!fid || !urlFid()) return '';
    var u = location.href.replace(/([?&]fileId=)\d+/i, '$1' + fid);
    return u !== location.href ? u : '';
  }

  function navGuardOk(fid) {
    try {
      var raw = String(kvGet('navGuard', '') || '');
      if (!raw) return true;
      var p = raw.split('|');
      if (p[0] !== fid) return true;
      return (Date.now() - Number(p[1] || 0)) > 60000;   // 同一个目标最多 60 秒跳一次
    } catch (e) { return true; }
  }

  // 记录页面自己发出的请求：用来判断"点了到底有没有被平台感知到"
  function noteNet(u) {
    try {
      if (!S.nets) S.nets = [];
      S.nets.push(new Date().toLocaleTimeString() + ' ' + String(u).slice(0, 130));
      if (S.nets.length > 40) S.nets.shift();
    } catch (e) { }
  }

  function installNetProbe() {
    try {
      // 必须挂在"页面真正的 window"上：篡改猴给脚本的是代理对象，改它看不到页面的请求
      var W = (typeof unsafeWindow !== 'undefined' && unsafeWindow) ? unsafeWindow : window;
      if (!W || W.__zhsNetHooked) return;
      W.__zhsNetHooked = 1;
      var of = W.fetch;
      if (typeof of === 'function') {
        W.fetch = function (a) {
          try { noteNet((a && a.url) || a); } catch (e) { }
          return of.apply(this, arguments);
        };
      }
      var XO = W.XMLHttpRequest && W.XMLHttpRequest.prototype;
      if (XO && XO.open) {
        var oo = XO.open;
        XO.open = function (m, u) { try { noteNet(u); } catch (e) { } return oo.apply(this, arguments); };
      }
    } catch (e) { }
  }

  function armVerify(before, list, i) {
    clearTimeout(S.verifyId);
    S.verifyId = setTimeout(function () {
      var now = snap();
      if (now !== before) {
        log('切换成功（' + (i === 0 ? '第一次点击' : '第 ' + (i + 1) + ' 次点击') + '），等新视频加载');
        setStatus('已切过去，等待新视频加载…');
        S.lastResult = '成功（点了 ' + (i + 1) + ' 次）';
        try { kvSet('zhsClickMode', 'ok'); } catch (e) { }
        S.switchAt = Date.now();
        S.sentPlay = 0;
        S.lastHref = location.href;
        S.tries = [];
        setTimeout(function () { tryPlayNow(); }, 500);
        setTimeout(function () { tryPlayNow(); }, 2000);
        return;
      }
      // 没变化 → 换一个候选再点
      i++;
      if (i < list.length) {
        log('点了没反应，换目标再试（第 ' + (i + 1) + ' 次）：' + describe(list[i]));
        setStatus('点了没反应，换目标再试（第 ' + (i + 1) + ' 次）');
        fire(list[i]);
        armVerify(before, list, i);
        return;
      }
      // 点遍了都没反应 → 直接改地址栏（最粗暴但一定有效）
      var fid = S.lastFid || '';
      var url = directUrlFor(S.lastEl, fid);
      if (url && navGuardOk(fid || url)) {
        kvSet('navGuard', (fid || url) + '|' + Date.now());
        try { kvSet('zhsClickMode', 'bad'); } catch (e) { }
        S.lastResult = '点击无效 → 直接跳转';
        log('点击无效，改为直接跳转：' + url);
        setStatus('点击无效，直接跳到下一个…');
        toast('点了没反应，改成直接跳转');
        S.fired = false;
        try { location.href = url; } catch (e) { }
        return;
      }
      var netN = (S.nets || []).length - (S.netAt || 0);
      S.lastResult = '失败：点了没反应（期间页面发了 ' + netN + ' 个请求）';
      log('所有候选都点了，页面还是没变（期间请求数 ' + netN + '）');
      setStatus('点了没反应：目录结构特殊，请用【示教】');
      toast('点了没反应。点一下【示教】，然后点列表里你正在看的那一项', true);
      flashTeach();
    }, 1600);
  }

  function describe(n) {
    try {
      return (n.tagName || '').toLowerCase() + '.' + String(n.className || '').split(/\s+/).slice(0, 2).join('.') +
        (txt(n) ? '("' + txt(n).slice(0, 14) + '")' : '');
    } catch (e) { return '?'; }
  }

  function doNext() {
    if (!S.enabled) return;
    if (detectBlock()) {
      setStatus('检测到弹题/验证，已暂停自动切换');
      toast('页面上有弹题或安全验证，先手动处理，处理完我会继续', true);
      S.fired = false;   // 让你处理完后还有机会再触发
      return;
    }

    var hit = findNext();
    if (hit) {
      // 之前已经证明"这个平台的点击不认" → 别再点了，直接改地址栏
      var preFid = hit.fid || fidOf(hit.el);
      var preUrl = directUrlFor(hit.el, preFid);
      if (preUrl && kvGet('zhsClickMode', '') === 'bad' && navGuardOk(preFid || preUrl)) {
        kvSet('navGuard', (preFid || preUrl) + '|' + Date.now());
        S.lastFid = preFid;
        S.lastEl = hit.el;
        S.lastResult = '直接跳转（点击已被证实无效）';
        log('点击无效（本会话已确认），直接跳转：' + preUrl);
        setStatus('直接跳到下一节…');
        toast('直接跳到下一节');
        S.fired = false;
        try { location.href = preUrl; } catch (e) { }
        return;
      }
      var before = snap();
      var list = clickCandidates(hit.el);
      S.lastFid = hit.fid || fidOf(hit.el);
      S.lastEl = hit.el;
      S.netAt = (S.nets || []).length;
      S.lastResult = '已点 ' + describe(list[0]);
      log('切下一个：' + hit.how + ' → 先点 ' + describe(list[0]) +
        (S.lastFid ? '（目标 fileId=' + S.lastFid + '）' : ''));
      setStatus('正在切：' + hit.how);
      fire(list[0]);
      toast('已点击下一个（' + hit.how + '）');
      S.fired = false;
      armVerify(before, list, 0);
      return;
    }

    // 本地找不到 → 视频可能在 iframe / 独立窗口里，请列表页去点
    var broadcast = false;
    try {
      if (!IS_TOP && window.parent && window.parent !== window) {
        window.parent.postMessage({ __zhs: 'next', from: 'frame' }, '*'); broadcast = true;
      }
    } catch (e) { }
    try {
      if (window.opener && !window.opener.closed) {
        window.opener.postMessage({ __zhs: 'next', from: 'window' }, '*'); broadcast = true;
      }
    } catch (e) { }

    if (broadcast) {
      setStatus('已通知列表页去点下一个');
      toast('已通知列表页切下一个');
      return;
    }

    // 真的没辙了
    setStatus('没找到"下一个"');
    toast('没找到下一个视频：点面板上的【示教】，然后点一下列表里你正在看的那一项，我记住它', true);
    log('没找到下一个。试试示教。');
    flashTeach();
  }

  window.addEventListener('message', function (ev) {
    var d = ev.data;
    if (!d || d.__zhs !== 'next') return;
    log('收到 ' + (d.from || '其他窗口') + ' 的切换请求');
    S.fired = true;
    scheduleNext('来自其他窗口');
  });

  /* ============================ 7. 示教 ============================ */

  function startTeach() {
    if (S.teaching) { stopTeach(); return; }
    S.teaching = true;
    toast('示教：请点一下列表里【你正在看的那一项】（别点下一项），我记住这个列表', true);
    setStatus('示教中，等你点一下');
    render();
    var t0 = Date.now();
    setTimeout(function () {
      document.addEventListener('click', onTeachClick, true);
    }, 350);
  }

  function stopTeach() {
    S.teaching = false;
    document.removeEventListener('click', onTeachClick, true);
    render();
  }

  function onTeachClick(ev) {
    if (!S.teaching) return;
    var t = ev.target;
    if (!t || !t.closest) return;
    if (t.closest('#zhs-panel')) return;

    var el = clickable(t) || t;
    var parent = el.parentElement;
    if (!parent) { stopTeach(); return; }

    var kids = Array.prototype.slice.call(parent.children);
    var idx = kids.indexOf(el);
    if (idx < 0) idx = kids.indexOf(el.parentElement);

    var rec = {
      containerSel: cssPath(parent),
      itemSig: (el.tagName || '').toLowerCase() + '|' + (typeof el.className === 'string' ? el.className : ''),
      index: idx < 0 ? null : idx,
      text: txt(el).slice(0, 60),
      href: (el.getAttribute && el.getAttribute('href')) || '',
      at: new Date().toISOString()
    };
    S.teach = rec;
    kvSet('teach', JSON.stringify(rec));
    S.teaching = false;
    document.removeEventListener('click', onTeachClick, true);
    log('示教已保存', rec);
    toast('已记住："' + (rec.text || rec.containerSel).slice(0, 26) + '"');
    setStatus('示教已保存');
  }

  var teachFlash = 0;
  function flashTeach() {
    var p = document.getElementById('zhs-panel');
    if (!p) return;
    var b = p.querySelector('[data-act="teach"]');
    if (!b) return;
    var n = 0;
    clearInterval(teachFlash);
    teachFlash = setInterval(function () {
      n++;
      b.style.background = n % 2 ? '#c0392b' : '#e67e22';
      if (n > 9) { clearInterval(teachFlash); b.style.background = ''; }
    }, 400);
  }

  /* ============================ 8. 面板 ============================ */

  var panel = null;

  function btn(label, act, tip) {
    var b = document.createElement('button');
    b.textContent = label;
    b.setAttribute('data-act', act);
    if (tip) b.title = tip;
    b.style.cssText = 'all:unset;box-sizing:border-box;cursor:pointer;padding:3px 9px;border-radius:6px;' +
      'background:#2b3a4d;color:#cfe3ff;font:12px system-ui,"Microsoft YaHei";text-align:center;user-select:none';
    b.addEventListener('mouseenter', function () { b.style.background = '#3a4f68'; });
    b.addEventListener('mouseleave', function () { b.style.background = b.getAttribute('data-hot') === '1' ? '#c0392b' : '#2b3a4d'; });
    b.addEventListener('click', function (e) {
      try { if (e && e.stopPropagation) e.stopPropagation(); } catch (x) { }
      handleAct(act);
    });
    return b;
  }

  function handleAct(act) {
    if (act === 'toggle') { S.enabled = !S.enabled; kvSet('en', S.enabled ? '1' : '0'); render(); return; }
    if (act === 'next') { S.fired = false; S.pending && (clearInterval(S.pending), S.pending = 0); doNext(); return; }
    if (act === 'teach') { startTeach(); return; }
    if (act === 'clearTeach') { S.teach = null; kvSet('teach', ''); toast('已清掉示教记录'); render(); return; }
    if (act === 'copy') { copyText(diagText()); return; }
    if (act === 'play') { S.sentPlay = 0; S.switchAt = Date.now(); tryPlayNow(true); setStatus('已尝试播放'); return; }
    if (act === 'close') { if (panel) panel.style.display = 'none'; return; }
  }

  function ensurePanel() {
    if (panel && document.body && document.body.contains(panel)) return panel;
    if (!document.body) return null;
    panel = document.createElement('div');
    panel.id = 'zhs-panel';
    var pos = null;
    try { pos = JSON.parse(kvGet('panelPos', '')); } catch (e) { pos = null; }
    panel.style.cssText = 'position:fixed;z-index:2147483645;width:224px;' +
      'background:linear-gradient(180deg,rgba(22,28,38,.97),rgba(16,20,28,.97));' +
      'color:#dfe9f7;border:1px solid rgba(96,160,255,.3);border-radius:11px;' +
      'box-shadow:0 12px 34px rgba(0,0,0,.5);font:12px/1.5 system-ui,"Microsoft YaHei";' +
      'padding:0 0 8px;overflow:hidden';
    panel.style.right = (pos && pos.right != null ? pos.right : 14) + 'px';
    panel.style.top = (pos && pos.top != null ? pos.top : 80) + 'px';

    var head = document.createElement('div');
    head.setAttribute('data-head', '1');
    head.style.cssText = 'display:flex;align-items:center;gap:6px;padding:7px 10px;cursor:move;' +
      'background:rgba(58,110,190,.22);border-bottom:1px solid rgba(96,160,255,.18);user-select:none';
    var title = document.createElement('span');
    title.textContent = '连播 v' + VERSION;
    title.style.cssText = 'font-weight:700;color:#8fc2ff;flex:1';
    var x = document.createElement('span');
    x.textContent = '✕';
    x.style.cssText = 'cursor:pointer;color:#7f93a8;padding:0 2px;font-size:13px';
    x.addEventListener('click', function () { panel.style.display = 'none'; });
    head.appendChild(title); head.appendChild(x);

    var body = document.createElement('div');
    body.id = 'zhs-body';
    body.style.cssText = 'padding:8px 10px 0';

    panel.appendChild(head);
    panel.appendChild(body);
    document.body.appendChild(panel);

    // 拖动
    var drag = null;
    head.addEventListener('mousedown', function (e) {
      drag = { x: e.clientX, y: e.clientY, r: parseInt(panel.style.right, 10) || 14, t: parseInt(panel.style.top, 10) || 80 };
      e.preventDefault();
    });
    document.addEventListener('mousemove', function (e) {
      if (!drag) return;
      var nr = Math.max(0, drag.r - (e.clientX - drag.x));
      var nt = Math.max(0, drag.t + (e.clientY - drag.y));
      panel.style.right = nr + 'px';
      panel.style.top = nt + 'px';
    });
    document.addEventListener('mouseup', function () {
      if (!drag) return;
      drag = null;
      kvSet('panelPos', JSON.stringify({ right: parseInt(panel.style.right, 10), top: parseInt(panel.style.top, 10) }));
    });

    buildBody();
    return panel;
  }

  function buildBody() {
    var b = document.getElementById('zhs-body');
    if (!b) return;
    b.textContent = '';

    var line1 = document.createElement('div');
    line1.style.cssText = 'display:flex;gap:6px;margin-bottom:6px';
    var sw = btn('', 'toggle');
    sw.setAttribute('data-sw', '1');
    line1.appendChild(sw);
    line1.appendChild(btn('马上下一个 ▶', 'next', '不等倒计时，立刻切'));
    b.appendChild(line1);

    var line2 = document.createElement('div');
    line2.style.cssText = 'display:flex;gap:6px;margin-bottom:6px';
    line2.appendChild(btn('示教', 'teach', '手动点一次"下一个"，我记住它'));
    line2.appendChild(btn('清示教', 'clearTeach'));
    line2.appendChild(btn('试播', 'play', '让视频开始播放（play + 点播放按钮 + 点画面）'));
    line2.appendChild(btn('诊断', 'copy', '复制诊断信息'));
    b.appendChild(line2);

    var drow = document.createElement('div');
    drow.style.cssText = 'display:flex;align-items:center;gap:6px;margin-bottom:6px;color:#9fb4ca';
    var lab = document.createElement('span');
    lab.textContent = '延迟';
    var inp = document.createElement('input');
    inp.type = 'number'; inp.min = '0'; inp.max = '30'; inp.value = String(S.delay);
    inp.style.cssText = 'width:44px;background:#1b2432;border:1px solid #33506e;color:#dfe9f7;' +
      'border-radius:5px;padding:2px 4px;font:12px system-ui;text-align:center';
    inp.addEventListener('change', function () {
      var v = Math.max(0, Math.min(30, Number(inp.value) || 0));
      S.delay = v; inp.value = String(v); kvSet('delay', v);
    });
    var sec = document.createElement('span');
    sec.textContent = '秒';
    var sw2 = document.createElement('label');
    sw2.style.cssText = 'margin-left:auto;display:flex;align-items:center;gap:4px;cursor:pointer;color:#9fb4ca';
    var cb = document.createElement('input');
    cb.type = 'checkbox'; cb.checked = S.autoPlay;
    cb.style.cssText = 'accent-color:#4a9eff;cursor:pointer';
    cb.addEventListener('change', function () { S.autoPlay = cb.checked; kvSet('autoPlay', S.autoPlay ? '1' : '0'); });
    sw2.appendChild(cb);
    sw2.appendChild(document.createTextNode('自动开播'));
    drow.appendChild(lab); drow.appendChild(inp); drow.appendChild(sec); drow.appendChild(sw2);
    b.appendChild(drow);

    var st = document.createElement('div');
    st.id = 'zhs-status';
    st.style.cssText = 'color:#9fb4ca;min-height:32px;border-top:1px solid rgba(96,160,255,.15);' +
      'padding-top:6px;word-break:break-all;line-height:1.45';
    b.appendChild(st);
  }

  function render() {
    if (!panel) return;
    var sw = panel.querySelector('[data-sw]');
    if (sw) {
      sw.textContent = S.enabled ? '● 已开启' : '○ 已关闭';
      sw.style.background = S.enabled ? '#1f6b3f' : '#43303a';
      sw.style.color = S.enabled ? '#b7f0cc' : '#e6b0b0';
    }
    var th = panel.querySelector('[data-act="teach"]');
    if (th) th.textContent = S.teaching ? '点页面上的目标…' : (S.teach ? '重示教' : '示教');

    var st = document.getElementById('zhs-status');
    if (!st) return;
    var v = S.video;
    var lines = [];
    if (v) {
      var d = isFinite(v.duration) ? Math.round(v.duration) : 0;
      var c = Math.round(v.currentTime || 0);
      lines.push('视频 ' + mmss(c) + ' / ' + mmss(d) + (v.paused ? ' · 暂停' : ' · 播放中'));
    } else {
      lines.push('页面上还没发现 <video>');
    }
    if (S.teach) lines.push('示教：' + (S.teach.text || S.teach.containerSel).slice(0, 22));
    else lines.push('示教：未设置');
    var msg = S.blocked ? '⚠ ' + S.blockedText.slice(0, 30) : S.lastMsg;
    if (msg) lines.push(msg);
    st.textContent = lines.join('\n');
    st.style.whiteSpace = 'pre-line';
  }

  function mmss(s) {
    s = Math.max(0, Math.round(s));
    var m = Math.floor(s / 60), q = s % 60;
    return m + ':' + (q < 10 ? '0' : '') + q;
  }

  /* ============================ 9. 诊断 ============================ */

  function diagText() {
    var v = S.video;
    var out = [
      '智慧树·自动连播 v' + VERSION,
      '时间：' + new Date().toLocaleString(),
      '地址：' + location.href,
      '站点：' + location.hostname,
      '课程号：' + CID + ' / fileId：' + (FILE_ID || '无'),
      '帧：' + (IS_TOP ? '顶层' : '子帧'),
      '开关：' + (S.enabled ? '开' : '关') + ' / 延迟 ' + S.delay + 's / 自动开播 ' + (S.autoPlay ? '是' : '否'),
      '视频：' + (v ? (mmss(v.currentTime) + '/' + mmss(v.duration) + (v.paused ? ' 暂停' : ' 播放中')) : '未发现'),
      '示教：' + (S.teach ? JSON.stringify(S.teach) : '未设置'),
      '弹窗阻塞：' + (S.blocked ? S.blockedText : '无'),
      'iframe 数：' + (docs().length - 1),
      '目录：' + (function () {
        var r = [], f = [], d = docs()[0], i;
        try {
          var rs = videoRows();
          r.push('视频行 ' + rs.length + '（已完成 ' + (function () { var n = 0, k; for (k = 0; k < rs.length; k++) if (rs[k].done) n++; return n; })() + '）');
          var fs = d.querySelectorAll('[id^="folder_"]');
          for (i = 0; i < fs.length; i++) f.push(txt(fs[i]).split('\n')[0].slice(0, 12));
          r.push('栏目 ' + fs.length + ' 个' + (f.length ? '：' + f.join(' / ') : ''));
          var idx = -1, k2;
          if (FILE_ID) for (k2 = 0; k2 < rs.length; k2++) if (rs[k2].fid === FILE_ID) { idx = k2; break; }
          r.push(idx >= 0 ? ('当前在目录第 ' + (idx + 1) + '/' + rs.length + ' 个') : '当前这一节不在目录里');
        } catch (e) { r.push('读取失败：' + (e && e.message)); }
        return r.join('；');
      })(),
      '上次切换：' + (S.lastResult || '还没试过') + (S.lastFid ? '（目标 fileId=' + S.lastFid + '）' : ''),
      '直接跳转可用：' + (urlFid() ? '是' : '否（地址栏没 fileId 参数）'),
      '最近请求：' + ((S.nets && S.nets.length) ? ('\n  ' + S.nets.slice(-6).join('\n  ')) : '无')
    ];
    var hit = null;
    try { hit = findNext(); } catch (e) { }
    out.push('当前能定位到的下一个：' + (hit ? hit.how + ' → ' + cssPath(hit.el).slice(0, 90) : '没有'));
    out.push('--- 日志 ---');
    out.push.apply(out, S.logs.slice(-25));
    return out.join('\n');
  }

  function copyText(s) {
    try {
      if (navigator.clipboard && navigator.clipboard.writeText) {
        navigator.clipboard.writeText(s).then(function () { toast('诊断信息已复制，粘贴给我就行'); }, function () { fallback(s); });
        return;
      }
    } catch (e) { }
    fallback(s);
  }
  function fallback(s) {
    try {
      var ta = document.createElement('textarea');
      ta.value = s;
      ta.style.cssText = 'position:fixed;left:-9999px;top:0';
      document.body.appendChild(ta);
      ta.select();
      document.execCommand('copy');
      ta.remove();
      toast('诊断信息已复制');
    } catch (e) {
      try { console.log(s); } catch (e2) { }
      toast('复制失败，请看控制台');
    }
  }

  /* ============================ 10. 主循环 ============================ */

  // 让视频真的动起来：play() → 播放按钮 → 点画面，三种都试
  function ensureVideoRunning(v) {
    if (!v || !S.autoPlay || !S.enabled || S.blocked) return;
    if (!v.paused && !v.ended) { S.sentPlay = 0; return; }
    if (v.ended) return;                       // 播完了，交给"切下一个"那条路
    var t = v.currentTime || 0;
    if (t > 2) return;                         // 中途的暂停是用户自己按的，别抢
    if (v.readyState < 1) return;              // 还没开始加载，等下次
    var recent = (Date.now() - S.switchAt) < 25000;
    var cap = recent ? 14 : 6;
    if (S.sentPlay >= cap) return;
    S.sentPlay++;
    try {
      var p = v.play();
      if (p && p.catch) p.catch(function (e) {
        log('浏览器拦了自动播放（' + (e && e.name) + '），需要你点一下播放器');
      });
    } catch (e) { }
    // 隔几次点一下播放器按钮 / 画面，很多播放器不吃 play()
    if (S.sentPlay === 2 || S.sentPlay === 5 || S.sentPlay === 9) {
      var b = findPlayBtn();
      if (b) { log('点播放器按钮：' + describe(b)); fire(b); }
      else if (S.sentPlay === 5) { try { fire(v); } catch (e) { } }
    }
  }

  var tickId = 0, tickN = 0;
  function tick() {
    tickN++;
    try {
      // 地址变了 → 重置触发状态
      if (location.href !== S.lastHref) {
        S.lastHref = location.href;
        S.fired = false;
        S.blocked = false;
        S.listJumped = false;
        S.listAt = 0;
        log('地址变化，状态已重置');
      }

      var v = findVideo();
      S.video = v;
      if (v) {
        bindVideo(v);
        // 视频正常出现了 → 说明这一节是好的，把"跳过失效文件"的计数清零
        if (kvGet('zhsDeadSkip', '0') !== '0') kvSet('zhsDeadSkip', '0');
        if (!S.idxSaved) { S.idxSaved = true; saveIdx(); }
      } else if (S.enabled && IS_TOP) {
        // 没有 video：可能是"文件未发布或已删除"，自动跳过
        handleDead();
        // 也可能这就是一张纯目录页 → 自动进入第一个没看完的
        if (!deadText()) handleListPage();
      }

      detectBlock();
      ensureVideoRunning(v);

      // 面板：只在可能有视频的页面出现
      if (!panel && document.body && shouldShowPanel()) ensurePanel();
      if (!panel && document.body && deadText()) ensurePanel();
      if (tickN % 2 === 0) render();
    } catch (e) {
      log('循环里出错（已忽略）：' + (e && e.message));
    }
  }

  function shouldShowPanel() {
    if (!IS_TOP) return false;
    try {
      if (location.hostname.indexOf('zhihuishu.com') < 0) return false;
      if (findVideo()) return true;
      // 资源失效页（"文件未发布或已删除"）没有 video，但面板必须还在，否则没法诊断/跳过
      if (deadText()) return true;
      return /studyResource|courseInfo|study|video|player|resource|sourceLearning|aidedteaching/i
        .test(location.pathname + location.search);
    } catch (e) { return false; }
  }

  /* ============================ 11. 启动 ============================ */

  function boot() {
    log('v' + VERSION + ' 已启动，课程号 ' + CID);
    installNetProbe();
    try { console.log('%c[连播] v' + VERSION + ' 已加载 ' + new Date().toLocaleTimeString(), 'color:#0f0;background:#000;padding:2px 6px'); } catch (e) { }

    // 如果刚刚是脚本自己"直接跳转"过来的，说明已经到位，把闸门解开
    try {
      var gRaw = String(kvGet('navGuard', '') || '');
      var gFid = gRaw.split('|')[0];
      if (gFid && gFid === urlFid()) kvSet('navGuard', '');
    } catch (e) { }

    // 页面上留个肉眼可见的痕迹，省得判断"到底跑没跑"
    try {
      var m = document.createElement('div');
      m.textContent = '连播脚本 v' + VERSION + ' 已加载';
      m.style.cssText = 'position:fixed;left:50%;bottom:22px;transform:translateX(-50%);z-index:2147483647;' +
        'background:rgba(20,24,30,.94);color:#7fe0a7;font:12px system-ui;padding:6px 14px;border-radius:8px;' +
        'box-shadow:0 6px 18px rgba(0,0,0,.45);pointer-events:none';
      (document.body || document.documentElement).appendChild(m);
      setTimeout(function () { try { m.remove(); } catch (e) { } }, 4000);
    } catch (e) { }

    if (shouldShowPanel()) { ensurePanel(); render(); }
    tickId = setInterval(tick, 900);
    tick();

    // 页面内容可能是后渲染的：等一会儿再补一次面板
    var guard = 0;
    var g = setInterval(function () {
      guard++;
      if (guard > 120) { clearInterval(g); return; }
      if (shouldShowPanel() && ensurePanel()) { render(); clearInterval(g); }
    }, 500);

    try {
      if (typeof GM_registerMenuCommand === 'function') {
        GM_registerMenuCommand('切换连播开关', function () { handleAct('toggle'); });
        GM_registerMenuCommand('复制诊断信息', function () { copyText(diagText()); });
        GM_registerMenuCommand('立刻切下一个', function () { handleAct('next'); });
        GM_registerMenuCommand('清掉示教记录', function () { handleAct('clearTeach'); });
      }
    } catch (e) { }

    // 掉出视野就提醒一句（不改变行为，只是提示）
    document.addEventListener('visibilitychange', function () {
      if (document.visibilityState === 'hidden') log('页面切到后台了，部分平台的时长判定会和前台不一样');
    });
  }

  // 控制台自查入口
  try {
    var W = (typeof unsafeWindow !== 'undefined') ? unsafeWindow : window;
    W.__ZHS状态 = function () {
      var t = diagText();
      try { console.log(t); } catch (e) { }
      return t;
    };
    W.__ZHS_诊断 = W.__ZHS状态;
  } catch (e) { }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', boot);
  } else {
    boot();
  }
})();

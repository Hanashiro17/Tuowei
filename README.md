<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <title>鼠标/手指拖尾光晕 · 可换色</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }

    html, body {
      height: 100%;
      overflow: hidden;
      /* iOS 关键：禁止触摸滚动/回弹/长按菜单 */
      touch-action: none;
      -webkit-user-select: none;
      user-select: none;
      -webkit-touch-callout: none;
      -webkit-tap-highlight-color: transparent;
    }

    body {
      background: #ffffff;
      cursor: crosshair;
      transition: background 0.3s;
    }

    #trailLayer {
      position: fixed;
      inset: 0;
      pointer-events: none;
      z-index: 1;
    }

    .trail {
      position: absolute;
      top: 0;
      left: 0;
      border-radius: 50%;
      pointer-events: none;
      transform: translate(-50%, -50%);
      will-change: transform, opacity;
    }

    .hint {
      position: fixed;
      bottom: calc(24px + env(safe-area-inset-bottom, 0px));
      left: 50%;
      transform: translateX(-50%);
      color: #bbb;
      font-family: -apple-system, "PingFang SC", sans-serif;
      font-size: 14px;
      pointer-events: none;
      z-index: 2;
      white-space: nowrap;
    }

    .panel {
      position: fixed;
      top: calc(20px + env(safe-area-inset-top, 0px));
      right: calc(20px + env(safe-area-inset-right, 0px));
      background: rgba(255, 255, 255, 0.9);
      -webkit-backdrop-filter: blur(10px);
      backdrop-filter: blur(10px);
      border: 1px solid rgba(0, 0, 0, 0.08);
      border-radius: 14px;
      padding: 16px 18px;
      box-shadow: 0 8px 30px rgba(0, 0, 0, 0.08);
      font-family: -apple-system, "PingFang SC", sans-serif;
      font-size: 13px;
      color: #444;
      display: flex;
      flex-direction: column;
      gap: 12px;
      z-index: 10;
      min-width: 190px;
      transition: background 0.3s, color 0.3s, border-color 0.3s;
    }

    .panel label {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 10px;
      cursor: pointer;
    }

    .panel .row-title {
      font-weight: 600;
      color: #333;
      transition: color 0.3s;
    }

    input[type="color"] {
      -webkit-appearance: none;
      appearance: none;
      width: 36px;
      height: 28px;
      border: none;
      border-radius: 8px;
      background: none;
      cursor: pointer;
      padding: 0;
    }
    input[type="color"]::-webkit-color-swatch-wrapper { padding: 0; }
    input[type="color"]::-webkit-color-swatch {
      border: 2px solid #fff;
      border-radius: 8px;
      box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    }

    .swatches { display: flex; gap: 8px; flex-wrap: wrap; }
    .swatch {
      width: 24px;
      height: 24px;
      border-radius: 50%;
      cursor: pointer;
      border: 2px solid #fff;
      box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
      transition: transform 0.15s;
    }
    .swatch:active { transform: scale(1.15); }

    .bg-toggle { display: flex; gap: 8px; }
    .bg-btn {
      flex: 1;
      padding: 8px 0;
      border: 1px solid rgba(0, 0, 0, 0.12);
      border-radius: 8px;
      background: #fff;
      cursor: pointer;
      font-size: 12px;
      color: #444;
      transition: all 0.2s;
    }
    .bg-btn.active { background: #333; color: #fff; border-color: #333; }

    body.dark { background: #0f0f14; }
    body.dark .hint { color: #666; }
    body.dark .panel {
      background: rgba(40, 40, 52, 0.9);
      color: #ddd;
      border-color: rgba(255, 255, 255, 0.08);
    }
    body.dark .panel .row-title { color: #fff; }
    body.dark .bg-btn {
      background: #1e1e28;
      color: #ddd;
      border-color: rgba(255, 255, 255, 0.12);
    }
    body.dark .bg-btn.active {
      background: #fff;
      color: #222;
      border-color: #fff;
    }
  </style>
</head>
<body>

  <div id="trailLayer"></div>

  <div class="panel">
    <div class="row-title">🎨 拖尾颜色</div>
    <label>
      自定义
      <input type="color" id="colorPicker" value="#8a2be2">
    </label>
    <div class="swatches" id="swatches"></div>

    <div class="row-title" style="margin-top:4px;">🌓 背景</div>
    <div class="bg-toggle">
      <button class="bg-btn active" data-theme="light">白色</button>
      <button class="bg-btn" data-theme="dark">深色</button>
    </div>
  </div>

  <div class="hint">移动鼠标 / 滑动手指试试 ✨</div>

  <script>
    // ===== 可调参数 =====
    const TRAIL_COUNT = 25;
    const MAX_SIZE = 120;
    const MIN_SIZE = 10;
    const FOLLOW_SPEED = 0.35;

    const PRESETS = ['#8a2be2', '#ff3c78', '#00c2ff', '#00e676', '#ff9800', '#ffd600'];

    let currentRGB = hexToRgb('#8a2be2');
    let lastColorKey = '';

    const layer = document.getElementById('trailLayer');
    const colorPicker = document.getElementById('colorPicker');

    const dots = [];
    for (let i = 0; i < TRAIL_COUNT; i++) {
      const el = document.createElement('div');
      el.className = 'trail';
      layer.appendChild(el);
      dots.push({ el, x: window.innerWidth / 2, y: window.innerHeight / 2 });
    }

    let mouseX = window.innerWidth / 2;
    let mouseY = window.innerHeight / 2;

    // ===== 关键：pointer 事件同时覆盖鼠标 + 触摸（iOS 13+）=====
    // 再用 touchmove 兜底老 iOS
    function updatePoint(x, y) { mouseX = x; mouseY = y; }

    if (window.PointerEvent) {
      window.addEventListener('pointermove', (e) => {
        updatePoint(e.clientX, e.clientY);
      }, { passive: true });
    } else {
      // 老设备兜底
      window.addEventListener('mousemove', (e) => updatePoint(e.clientX, e.clientY), { passive: true });
      window.addEventListener('touchmove', (e) => {
        const t = e.touches[0];
        if (t) updatePoint(t.clientX, t.clientY);
        e.preventDefault(); // 阻止页面滚动
      }, { passive: false });
    }

    // ===== 工具 =====
    function hexToRgb(hex) {
      const h = hex.replace('#', '');
      return {
        r: parseInt(h.slice(0, 2), 16),
        g: parseInt(h.slice(2, 4), 16),
        b: parseInt(h.slice(4, 6), 16)
      };
    }

    function makeGradient({ r, g, b }, t) {
      const k = 1 - t * 0.95;
      const a1 = (0.85 * k).toFixed(3);
      const a2 = (0.5  * k).toFixed(3);
      const a3 = (0.15 * k).toFixed(3);
      return `radial-gradient(circle,
        rgba(${r},${g},${b},${a1}) 0%,
        rgba(${r},${g},${b},${a2}) 30%,
        rgba(${r},${g},${b},${a3}) 60%,
        rgba(${r},${g},${b},0) 100%)`;
    }

    // ===== 动画 =====
    function animate() {
      const colorKey = `${currentRGB.r},${currentRGB.g},${currentRGB.b}`;
      const colorChanged = colorKey !== lastColorKey;
      if (colorChanged) lastColorKey = colorKey;

      const gradientCache = colorChanged
        ? dots.map((_, i) => makeGradient(currentRGB, i / (dots.length - 1)))
        : null;

      let prevX = mouseX;
      let prevY = mouseY;

      for (let i = 0; i < dots.length; i++) {
        const dot = dots[i];

        dot.x += (prevX - dot.x) * FOLLOW_SPEED;
        dot.y += (prevY - dot.y) * FOLLOW_SPEED;

        const t = i / (dots.length - 1);
        const size = MAX_SIZE - (MAX_SIZE - MIN_SIZE) * t;
        const opacity = 1 - t * 0.95;

        dot.el.style.width = size + 'px';
        dot.el.style.height = size + 'px';
        dot.el.style.opacity = opacity;
        if (gradientCache) dot.el.style.background = gradientCache[i];
        dot.el.style.transform =
          `translate(${dot.x}px, ${dot.y}px) translate(-50%, -50%)`;

        prevX = dot.x;
        prevY = dot.y;
      }

      requestAnimationFrame(animate);
    }
    animate();

    // ===== 颜色控制 =====
    colorPicker.addEventListener('input', (e) => {
      currentRGB = hexToRgb(e.target.value);
    });

    const swatchBox = document.getElementById('swatches');
    PRESETS.forEach(hex => {
      const s = document.createElement('div');
      s.className = 'swatch';
      s.style.background = hex;
      s.title = hex;
      s.addEventListener('click', () => {
        colorPicker.value = hex;
        currentRGB = hexToRgb(hex);
      });
      swatchBox.appendChild(s);
    });

    // ===== 背景切换 =====
    document.querySelectorAll('.bg-btn').forEach(btn => {
      btn.addEventListener('click', () => {
        document.querySelectorAll('.bg-btn').forEach(b => b.classList.remove('active'));
        btn.classList.add('active');
        document.body.classList.toggle('dark', btn.dataset.theme === 'dark');
      });
    });
  </script>
</body>
</html>
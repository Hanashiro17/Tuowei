<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>鼠标拖尾光晕</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body {
      height: 100%;
      overflow: hidden;
      touch-action: none;
      -webkit-user-select: none;
      user-select: none;
      background: #ffffff;
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
    }
    .panel {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(255, 255, 255, 0.9);
      backdrop-filter: blur(10px);
      -webkit-backdrop-filter: blur(10px);
      border: 1px solid rgba(0, 0, 0, 0.08);
      border-radius: 14px;
      padding: 16px;
      box-shadow: 0 8px 30px rgba(0, 0, 0, 0.08);
      font-family: -apple-system, sans-serif;
      font-size: 13px;
      color: #444;
      display: flex;
      flex-direction: column;
      gap: 12px;
      z-index: 10;
    }
    .swatches { display: flex; gap: 8px; }
    .swatch {
      width: 24px;
      height: 24px;
      border-radius: 50%;
      border: 2px solid #fff;
      box-shadow: 0 2px 8px rgba(0,0,0,0.15);
    }
  </style>
</head>
<body>
  <div id="trailLayer"></div>

  <!-- 换色面板 -->
  <div class="panel">
    <div style="font-weight:600;">🎨 拖尾颜色</div>
    <div class="swatches" id="swatches"></div>
  </div>

  <script>
    const TRAIL_COUNT = 25;
    const MAX_SIZE = 120;
    const MIN_SIZE = 10;
    const FOLLOW_SPEED = 0.35;
    
    // 预设颜色
    const PRESETS = ['#8a2be2', '#ff3c78', '#00c2ff', '#00e676', '#ff9800'];
    let currentRGB = hexToRgb('#8a2be2');
    let lastColorKey = '';

    const layer = document.getElementById('trailLayer');
    const dots = [];
    
    for (let i = 0; i < TRAIL_COUNT; i++) {
      const el = document.createElement('div');
      el.className = 'trail';
      layer.appendChild(el);
      dots.push({ el, x: window.innerWidth / 2, y: window.innerHeight / 2 });
    }

    let mouseX = window.innerWidth / 2;
    let mouseY = window.innerHeight / 2;

    // 触摸和鼠标事件兼容
    window.addEventListener('pointermove', (e) => {
      mouseX = e.clientX;
      mouseY = e.clientY;
    }, { passive: true });
    
    window.addEventListener('touchmove', (e) => {
      if(e.touches[0]) {
        mouseX = e.touches[0].clientX;
        mouseY = e.touches[0].clientY;
      }
    }, { passive: true });

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
      return `radial-gradient(circle, rgba(${r},${g},${b},${(0.85*k).toFixed(2)}) 0%, rgba(${r},${g},${b},${(0.5*k).toFixed(2)}) 30%, rgba(${r},${g},${b},0) 100%)`;
    }

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
        
        dot.el.style.width = size + 'px';
        dot.el.style.height = size + 'px';
        dot.el.style.opacity = 1 - t * 0.95;
        if (gradientCache) dot.el.style.background = gradientCache[i];
        dot.el.style.transform = `translate(${dot.x}px, ${dot.y}px) translate(-50%, -50%)`;

        prevX = dot.x;
        prevY = dot.y;
      }
      requestAnimationFrame(animate);
    }
    animate();

    // 渲染色块
    const swatchBox = document.getElementById('swatches');
    PRESETS.forEach(hex => {
      const s = document.createElement('div');
      s.className = 'swatch';
      s.style.background = hex;
      s.addEventListener('click', () => {
        currentRGB = hexToRgb(hex);
      });
      swatchBox.appendChild(s);
    });
  </script>
</body>
</html>
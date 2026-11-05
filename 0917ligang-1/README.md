# 0917ligang# 0917ligang# 0917ligang<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>進化版小恐龍</title>
<style>
  * { box-sizing: border-box; }

  body {
    margin: 0;
    min-height: 100vh;
    display: grid;
    place-items: center;
    padding: 20px;
    background: #101c25;
    color: #ecfff4;
    font-family: system-ui, "Microsoft JhengHei", sans-serif;
  }

  main { width: min(960px, 100%); }
  h1 { margin-bottom: 8px; }
  p { color: #a8bdb5; line-height: 1.8; }

  .hud, .buttons {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    margin: 16px 0;
  }

  .hud span {
    background: #20352e;
    border-radius: 10px;
    padding: 10px 14px;
  }

  .stage {
    position: relative;
    overflow: hidden;
    border: 2px solid #426653;
    border-radius: 18px;
  }

  canvas {
    display: block;
    width: 100%;
    background: #193b32;
    touch-action: none;
  }

  .overlay {
    position: absolute;
    inset: 0;
    display: grid;
    place-content: center;
    text-align: center;
    padding: 16px;
    background: #081710d9;
  }

  .overlay h2 { margin: 0; }
  .hidden { display: none; }

  button {
    border: 0;
    border-radius: 10px;
    padding: 13px 20px;
    background: #bbf18e;
    color: #14261a;
    font: bold 16px system-ui;
    cursor: pointer;
    touch-action: manipulation;
    user-select: none;
    -webkit-user-select: none;
  }

  button:hover { background: #d5ffb0; }
  button:focus-visible { outline: 3px solid white; }

  .secondary {
    background: #30483d;
    color: white;
  }

  @media (max-width: 600px) {
    body { padding: 12px; }
    h1 { font-size: 26px; }
    .hud { gap: 6px; font-size: 12px; }
    .hud span { padding: 8px; }
    .overlay h2 { font-size: 21px; }
    .overlay p { margin: 8px 0; font-size: 13px; }
  }
</style>
</head>
<body>
<main>
  <h1>🦖 進化版小恐龍</h1>
  <p>二段跳、收集金幣、取得護盾，挑戰你的最高紀錄！</p>

  <div class="hud" aria-live="off">
    <span>分數：<b id="score">0</b></span>
    <span>金幣：<b id="coins">0</b></span>
    <span>護盾：<b id="shield">無</b></span>
    <span>進化：<b id="level">幼龍</b></span>
    <span>最高分：<b id="best">0</b></span>
  </div>

  <div class="stage">
    <canvas id="canvas" width="960" height="420"
      aria-label="恐龍跑酷遊戲，使用下方按鈕或鍵盤操作"></canvas>

    <div id="overlay" class="overlay">
      <h2 id="title">準備出發！</h2>
      <p id="message">空白鍵跳躍，可在空中再跳一次</p>
      <button id="start">開始遊戲</button>
    </div>
  </div>

  <div class="buttons">
    <button id="jump">↑ 跳躍 / 二段跳</button>
    <button id="duck" class="secondary">↓ 按住蹲下</button>
    <button id="pause" class="secondary">暫停</button>
  </div>

  <p>
    空白鍵 / ↑：跳躍　↓：蹲下　P：暫停<br>
    金幣 +25 分；藍色能量球提供一次護盾。每 500 公尺進化，
    速度逐漸增加。手機可點擊畫面跳躍。
  </p>
</main>

<script>
(() => {
  "use strict";

  const $ = id => document.getElementById(id);
  const canvas = $("canvas");
  const ctx = canvas.getContext("2d");
  const W = canvas.width;
  const H = canvas.height;
  const GROUND = 335;
  const STORAGE_KEY = "evolution-dino-best";

  let best = 0;

  try {
    const saved = Number(localStorage.getItem(STORAGE_KEY));
    best = Number.isFinite(saved) ? Math.max(0, saved) : 0;
  } catch {}

  $("best").textContent = best;

  let state = "ready";
  let distance = 0;
  let coinCount = 0;
  let hasShield = false;
  let crouching = false;
  let invincible = 0;
  let spawnTimer = 1;
  let elapsed = 0;
  let lastTime = 0;
  let objects = [];
  let particles = [];

  const dino = {
    x: 120,
    y: GROUND,
    velocityY: 0,
    jumps: 0
  };

  const random = (min, max) => min + Math.random() * (max - min);
  const getScore = () => Math.floor(distance) + coinCount * 25;
  const getLevel = () => Math.min(2, Math.floor(distance / 500));

  function showOverlay(title, message, buttonText) {
    $("title").textContent = title;
    $("message").textContent = message;
    $("start").textContent = buttonText;
    $("overlay").classList.remove("hidden");
  }

  function updateHUD() {
    $("score").textContent = getScore();
    $("coins").textContent = coinCount;
    $("shield").textContent = hasShield ? "有" : "無";
    $("level").textContent = ["幼龍", "迅猛龍", "極光龍"][getLevel()];
  }

  function startGame() {
    state = "playing";
    distance = 0;
    coinCount = 0;
    hasShield = false;
    crouching = false;
    invincible = 0;
    spawnTimer = 1.2;
    elapsed = 0;
    objects = [];
    particles = [];

    dino.y = GROUND;
    dino.velocityY = 0;
    dino.jumps = 0;

    $("overlay").classList.add("hidden");
    $("pause").textContent = "暫停";
    updateHUD();
  }

  function jump() {
    if (state === "ready" || state === "over") {
      startGame();
      return;
    }

    if (state !== "playing" || dino.jumps >= 2) return;

    crouching = false;
    dino.velocityY = dino.jumps === 0 ? -650 : -550;
    dino.jumps++;
    createParticles(dino.x + 20, dino.y, "#c5f59e", 8);
  }

  function togglePause() {
    if (state === "playing") {
      state = "paused";
      crouching = false;
      $("pause").textContent = "繼續";
      showOverlay("休息一下", "按 P 或點擊按鈕繼續", "繼續遊戲");
    } else if (state === "paused") {
      state = "playing";
      $("pause").textContent = "暫停";
      $("overlay").classList.add("hidden");
    }
  }

  function gameOver() {
    state = "over";
    const score = getScore();

    if (score > best) {
      best = score;
      $("best").textContent = best;

      try {
        localStorage.setItem(STORAGE_KEY, String(best));
      } catch {}
    }

    showOverlay(
      "冒險結束！",
      `距離 ${Math.floor(distance)} 公尺 · 金幣 ${coinCount} · 分數 ${score}`,
      "再玩一次"
    );
  }

  function createParticles(x, y, color, count) {
    for (let i = 0; i < count; i++) {
      particles.push({
        x,
        y,
        vx: random(-160, 160),
        vy: random(-220, 40),
        life: random(0.3, 0.7),
        color
      });
    }
  }

  function playerBounds() {
    const low = crouching && dino.y >= GROUND;

    return {
      x: dino.x + 6,
      y: dino.y - (low ? 28 : 58),
      w: low ? 56 : 39,
      h: low ? 26 : 56
    };
  }

  function overlaps(a, b) {
    return (
      a.x < b.x + b.w &&
      a.x + a.w > b.x &&
      a.y < b.y + b.h &&
      a.y + a.h > b.y
    );
  }

  function spawnObjects() {
    const isBird = distance > 180 && Math.random() < 0.3;

    objects.push({
      type: isBird ? "bird" : "cactus",
      x: W + 30,
      y: isBird ? GROUND - 73 : GROUND - 48,
      w: isBird ? 46 : 32,
      h: isBird ? 27 : 48,
      removed: false
    });

    if (Math.random() < 0.8) {
      objects.push({
        type: Math.random() < 0.18 ? "shield" : "coin",
        x: W + 115,
        y: GROUND - random(100, 170),
        w: 24,
        h: 24,
        removed: false
      });
    }
  }

  function update(dt) {
    elapsed += dt;

    const speed = Math.min(680, 310 + distance * 0.075);
    distance += speed * dt / 12;
    invincible = Math.max(0, invincible - dt);

    dino.velocityY += 1750 * dt;
    dino.y += dino.velocityY * dt;

    if (dino.y >= GROUND) {
      dino.y = GROUND;
      dino.velocityY = 0;
      dino.jumps = 0;
    }

    spawnTimer -= dt;

    if (spawnTimer <= 0) {
      spawnObjects();
      spawnTimer = random(1.2, 1.85) * Math.max(0.77, 310 / speed);
    }

    const player = playerBounds();

    for (const object of objects) {
      object.x -= speed * dt;

      if (object.removed || !overlaps(player, object)) continue;

      if (object.type === "coin") {
        coinCount++;
        object.removed = true;
        createParticles(object.x + 12, object.y + 12, "#ffda70", 12);
      } else if (object.type === "shield") {
        hasShield = true;
        object.removed = true;
        createParticles(object.x + 12, object.y + 12, "#80e7ff", 14);
      } else if (invincible <= 0) {
        if (hasShield) {
          hasShield = false;
          invincible = 1.4;
          object.removed = true;
          createParticles(dino.x + 25, dino.y - 30, "#80e7ff", 24);
        } else {
          gameOver();
          break;
        }
      }
    }

    objects = objects.filter(object => object.x > -80 && !object.removed);

    for (const particle of particles) {
      particle.life -= dt;
      particle.x += particle.vx * dt;
      particle.y += particle.vy * dt;
      particle.vy += 450 * dt;
    }

    particles = particles.filter(particle => particle.life > 0);
    updateHUD();
  }

  function rect(x, y, width, height, color) {
    ctx.fillStyle = color;
    ctx.fillRect(x, y, width, height);
  }

  function drawBackground() {
    const night = Math.floor(distance / 500) % 2 === 1;

    rect(0, 0, W, H, night ? "#142239" : "#193b32");

    ctx.fillStyle = night ? "#e1eafa" : "#f0dfa0";
    ctx.beginPath();
    ctx.arc(800, 85, 30, 0, Math.PI * 2);
    ctx.fill();

    if (night) {
      ctx.fillStyle = "#adbedb";

      for (let i = 0; i < 28; i++) {
        ctx.fillRect((i * 137 + 31) % W, (i * 61) % 165 + 20, 2, 2);
      }
    }

    for (let layer = 0; layer < 2; layer++) {
      ctx.fillStyle = layer === 0 ? "#23483f" : "#2b5543";

      for (let i = -1; i < 5; i++) {
        const x = i * 280 - (distance * (layer + 1) * 3) % 280;

        ctx.beginPath();
        ctx.moveTo(x, GROUND);
        ctx.lineTo(x + 140, 150 + layer * 65);
        ctx.lineTo(x + 300, GROUND);
        ctx.closePath();
        ctx.fill();
      }
    }

    rect(0, GROUND, W, H - GROUND, "#10291f");
    rect(0, GROUND, W, 3, "#85a86b");

    for (let i = 0; i < 22; i++) {
      const x = i * 51 - (distance * 12) % 51;
      rect(x, GROUND + 20 + (i % 3) * 18, 15, 2, "#35513c");
    }
  }

  function drawDino() {
    const x = dino.x;
    const y = dino.y;
    const low = crouching && y >= GROUND;
    const color = ["#bbf18e", "#ffce83", "#96e8ff"][getLevel()];

    ctx.save();

    if (invincible > 0 && Math.floor(elapsed * 15) % 2 === 0) {
      ctx.globalAlpha = 0.35;
    }

    if (low) {
      rect(x + 5, y - 25, 43, 18, color);
      rect(x + 39, y - 31, 25, 17, color);
      rect(x, y - 30, 10, 17, color);
      rect(x + 55, y - 27, 4, 4, "#183325");
    } else {
      rect(x + 7, y - 38, 28, 30, color);
      rect(x + 22, y - 60, 29, 27, color);
      rect(x + 32, y - 36, 13, 7, color);
      rect(x, y - 43, 12, 23, color);
      rect(x + 42, y - 54, 4, 5, "#183325");
      rect(x + 31, y - 22, 14, 6, color);
    }

    const step = y < GROUND ? 0 : Math.sin(elapsed * 20) * 4;
    rect(x + 10, y - 10, 8, 10 + step, color);
    rect(x + 27, y - 10, 8, 10 - step, color);

    ctx.restore();

    if (hasShield) {
      ctx.strokeStyle = "#80e7ff";
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.ellipse(x + 27, y - 29, 47, 49, 0, 0, Math.PI * 2);
      ctx.stroke();
    }
  }

  function drawObjects() {
    for (const object of objects) {
      const { x, y, type } = object;

      if (type === "coin" || type === "shield") {
        ctx.fillStyle = type === "coin" ? "#ffda70" : "#80e7ff";
        ctx.beginPath();
        ctx.arc(x + 12, y + 12, 11, 0, Math.PI * 2);
        ctx.fill();

        ctx.strokeStyle = type === "coin" ? "#b88727" : "#e2fbff";
        ctx.lineWidth = 2;
        ctx.stroke();

        rect(
          x + 10, y + 6, 4, 12,
          type === "coin" ? "#b88727" : "#e2fbff"
        );
      } else if (type === "cactus") {
        const green = "#8db875";
        rect(x + 10, y, 12, 48, green);
        rect(x, y + 15, 12, 8, green);
        rect(x, y + 7, 6, 16, green);
        rect(x + 20, y + 25, 12, 8, green);
        rect(x + 26, y + 14, 6, 19, green);
      } else {
        rect(x, y + 9, 46, 12, "#e6b68f");
        rect(x + 4, y + 2, 13, 17, "#e6b68f");
        rect(x + 7, y + 5, 4, 4, "#22352a");

        const wingY = Math.sin(elapsed * 18) > 0 ? -8 : 17;
        rect(x + 23, y + wingY, 14, 17, "#bc876d");
      }
    }
  }

  function draw() {
    drawBackground();
    drawObjects();
    drawDino();

    for (const particle of particles) {
      ctx.globalAlpha = Math.min(1, particle.life * 2);
      rect(particle.x, particle.y, 5, 5, particle.color);
    }

    ctx.globalAlpha = 1;
  }

  function frame(now) {
    const dt = Math.min((now - lastTime) / 1000 || 0, 0.033);
    lastTime = now;

    if (state === "playing") update(dt);

    draw();
    requestAnimationFrame(frame);
  }

  $("start").addEventListener("click", () => {
    if (state === "paused") togglePause();
    else startGame();
  });

  $("pause").addEventListener("click", togglePause);

  $("jump").addEventListener("pointerdown", event => {
    event.preventDefault();
    jump();
  });

  // 支援以鍵盤啟用跳躍按鈕。
  $("jump").addEventListener("click", event => {
    if (event.detail === 0) jump();
  });

  canvas.addEventListener("pointerdown", event => {
    event.preventDefault();
    jump();
  });

  $("duck").addEventListener("pointerdown", event => {
    event.preventDefault();
    crouching = true;
    $("duck").setPointerCapture(event.pointerId);
  });

  for (const eventName of [
    "pointerup",
    "pointercancel",
    "lostpointercapture"
  ]) {
    $("duck").addEventListener(eventName, () => {
      crouching = false;
    });
  }

  window.addEventListener("keydown", event => {
    const supported = ["Space", "ArrowUp", "ArrowDown", "KeyP"];
    if (!supported.includes(event.code)) return;

    // 保留按鈕原生的空白鍵操作。
    if (event.code === "Space" && event.target.tagName === "BUTTON") {
      return;
    }

    event.preventDefault();

    if (event.code === "ArrowDown") {
      crouching = true;
    } else if (!event.repeat) {
      if (event.code === "KeyP") togglePause();
      else jump();
    }
  });

  window.addEventListener("keyup", event => {
    if (event.code === "ArrowDown") crouching = false;
  });

  window.addEventListener("blur", () => {
    crouching = false;
    if (state === "playing") togglePause();
  });

  document.addEventListener("visibilitychange", () => {
    if (document.hidden && state === "playing") togglePause();
  });

  requestAnimationFrame(frame);
})();
</script>
</body>
</html>
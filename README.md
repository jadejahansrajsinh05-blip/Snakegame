

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Snake Game — Simple Mobile Friendly</title>
  <style>
    :root{
      --bg:#0f1724;
      --panel:#071226;
      --accent:#22c55e;
      --muted:#94a3b8;
      --danger:#ef4444;
    }
    html,body{height:100%;margin:0;font-family:Inter,system-ui,Segoe UI,Roboto,'Helvetica Neue',Arial;}
    body{display:flex;align-items:center;justify-content:center;background:linear-gradient(180deg,#071428 0%, #001021 100%);color:#e6eef8;padding:20px;}
    .container{width:100%;max-width:700px;background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border-radius:12px;padding:16px;box-shadow:0 8px 30px rgba(2,6,23,0.7);}
    header{display:flex;justify-content:space-between;align-items:center;margin-bottom:10px}
    h1{font-size:18px;margin:0}
    .meta{font-size:13px;color:var(--muted)}
    .panel{display:flex;gap:12px;align-items:center}
    .btn{background:transparent;border:1px solid rgba(255,255,255,0.06);padding:8px 12px;border-radius:8px;color:inherit;cursor:pointer}
    .btn.primary{background:var(--accent);color:#062016;border:none}
    .scoreboard{display:flex;gap:12px;align-items:center;font-size:14px}
    #gameArea{display:block;margin:10px auto;border-radius:8px;background:linear-gradient(180deg,#081826,#041426);touch-action:none}
    .info{margin-top:10px;font-size:13px;color:var(--muted)}
    .footer{display:flex;justify-content:space-between;align-items:center;margin-top:12px}
    .small{font-size:12px;color:var(--muted)}
    .controls{display:flex;gap:8px}
    .hint{color:var(--muted);font-size:13px}
    /* responsive canvas */
    @media (max-width:420px){
      .container{padding:12px}
      h1{font-size:16px}
    }
  </style>
</head>
<body>
  <div class="container" role="application" aria-label="Snake Game">
    <header>
      <div>
        <h1>Snake Game</h1>
        <div class="meta">Classic snake — keyboard + touch support</div>
      </div>
      <div class="panel">
        <div class="scoreboard">
          <div>Score: <strong id="score">0</strong></div>
          <div style="color:var(--muted)">High: <strong id="highscore">0</strong></div>
        </div>
      </div>
    </header>

    <canvas id="gameArea" width="600" height="600" tabindex="0"></canvas>

    <div class="footer">
      <div class="controls">
        <button class="btn" id="btnPause">Pause</button>
        <button class="btn" id="btnRestart">Restart</button>
        <button class="btn primary" id="btnStart">Start</button>
      </div>
      <div class="small">Use arrow keys / WASD or swipe to play</div>
    </div>

    <p class="info">
      <strong>How to play:</strong> Eat the green food to grow the snake. Don't hit walls or your own body.
    </p>

    <p class="hint">Tip: On mobile, swipe in the direction you want the snake to travel.</p>
  </div>

<script>
/*
  Simple Snake Game
  - Grid-based
  - Mobile swipe support
  - Local highscore (localStorage)
*/

// Config
const config = {
  canvasSize: 600,
  rows: 20,               // grid rows
  cols: 20,               // grid cols
  fps: 10,                // initial ticks per second (speed)
  snakeColor: '#60a5fa',
  foodColor: '#22c55e',
  bgColor: '#021627',
  cellGap: 2
};

const canvas = document.getElementById('gameArea');
const ctx = canvas.getContext('2d');
canvas.style.background = config.bgColor;
canvas.setAttribute('aria-label','Snake game canvas');

let cellSize = Math.floor(config.canvasSize / config.rows);
let gameLoopId = null;

// State
let state = {
  snake: [],          // array of {r,c}
  dir: {r:0,c:1},     // moving right initially
  nextDir: null,
  food: null,
  running: false,
  paused: false,
  score: 0,
  highscore: 0,
  speed: config.fps
};

// Init highscore
state.highscore = Number(localStorage.getItem('snake_high') || 0);
document.getElementById('highscore').innerText = state.highscore;

// Helpers
function posToPixel(r,c){
  return { x: c*cellSize + config.cellGap, y: r*cellSize + config.cellGap, w: cellSize - config.cellGap*2, h: cellSize - config.cellGap*2 };
}

// Initialize / reset game
function resetGame(){
  state.snake = [ {r: Math.floor(config.rows/2), c: Math.floor(config.cols/2) } ];
  state.dir = {r:0,c:1};
  state.nextDir = null;
  state.score = 0;
  state.running = false;
  state.paused = false;
  state.speed = config.fps;
  spawnFood();
  document.getElementById('score').innerText = state.score;
}
resetGame();

// Spawn food at random empty cell
function spawnFood(){
  let empty = [];
  for(let r=0;r<config.rows;r++){
    for(let c=0;c<config.cols;c++){
      if(!state.snake.some(s => s.r===r && s.c===c)){
        empty.push({r,c});
      }
    }
  }
  state.food = empty[Math.floor(Math.random()*empty.length)];
}

// Draw grid + snake + food
function draw(){
  // clear
  ctx.fillStyle = '#021627';
  ctx.fillRect(0,0,canvas.width,canvas.height);

  // draw cells (optional grid lines)
  // draw food
  if(state.food){
    const p = posToPixel(state.food.r, state.food.c);
    ctx.fillStyle = config.foodColor;
    roundRect(ctx, p.x, p.y, p.w, p.h, 6, true, false);
  }

  // draw snake
  state.snake.forEach((seg, idx) => {
    const p = posToPixel(seg.r, seg.c);
    ctx.fillStyle = idx===0 ? '#1e3a8a' : config.snakeColor;
    const radius = Math.max(4, (p.w)/6);
    roundRect(ctx, p.x, p.y, p.w, p.h, radius, true, false);
  });
}

// Rounded rect helper
function roundRect(ctx, x, y, w, h, r, fill, stroke){
  if (typeof r === 'undefined') r = 5;
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.arcTo(x + w, y, x + w, y + h, r);
  ctx.arcTo(x + w, y + h, x, y + h, r);
  ctx.arcTo(x, y + h, x, y, r);
  ctx.arcTo(x, y, x + w, y, r);
  ctx.closePath();
  if(fill){ ctx.fill(); }
  if(stroke){ ctx.stroke(); }
}

// Game update (move snake)
function update(){
  if(!state.running || state.paused) return;

  // apply queued direction if valid (no immediate 180 turn)
  if(state.nextDir){
    if(!(state.nextDir.r === -state.dir.r && state.nextDir.c === -state.dir.c)){
      state.dir = state.nextDir;
    }
    state.nextDir = null;
  }

  const head = {...state.snake[0]};
  head.r += state.dir.r;
  head.c += state.dir.c;

  // wall collision -> game over
  if(head.r < 0 || head.r >= config.rows || head.c < 0 || head.c >= config.cols){
    return gameOver();
  }

  // self collision -> game over
  if(state.snake.some(seg => seg.r === head.r && seg.c === head.c)){
    return gameOver();
  }

  // move
  state.snake.unshift(head);

  // check eat food
  if(state.food && head.r === state.food.r && head.c === state.food.c){
    state.score += 10;
    document.getElementById('score').innerText = state.score;
    // speed up slightly every 50 points
    if(state.score % 50 === 0) state.speed = Math.min(20, state.speed + 1);
    spawnFood();
  } else {
    state.snake.pop(); // remove tail
  }
}

// Game Over
function gameOver(){
  state.running = false;
  state.paused = false;
  // update highscore
  if(state.score > state.highscore){
    state.highscore = state.score;
    localStorage.setItem('snake_high', state.highscore);
    document.getElementById('highscore').innerText = state.highscore;
  }
  // show simple alert (could be improved)
  setTimeout(()=> {
    const playAgain = confirm('Game Over! Your score: ' + state.score + '. Play again?');
    if(playAgain) startGame();
  }, 10);
}

// Game loop using requestAnimationFrame but limiting by speed
let lastTick = 0;
function loop(ts){
  if(!lastTick) lastTick = ts;
  const interval = 1000 / state.speed;
  if(ts - lastTick >= interval){
    update();
    draw();
    lastTick = ts;
  }
  gameLoopId = requestAnimationFrame(loop);
}

// Controls: Keyboard
window.addEventListener('keydown', (e) => {
  const key = e.key;
  if(['ArrowUp','ArrowDown','ArrowLeft','ArrowRight','w','a','s','d','W','A','S','D'].includes(key)){
    e.preventDefault();
    if(!state.running) startGame();
    switch(key){
      case 'ArrowUp': case 'w': case 'W': state.nextDir = {r:-1,c:0}; break;
      case 'ArrowDown': case 's': case 'S': state.nextDir = {r:1,c:0}; break;
      case 'ArrowLeft': case 'a': case 'A': state.nextDir = {r:0,c:-1}; break;
      case 'ArrowRight': case 'd': case 'D': state.nextDir = {r:0,c:1}; break;
    }
  } else if(key === ' '){ // space to pause/unpause
    togglePause();
  }
});

// Touch (swipe) support for mobile
let touchStartX = 0, touchStartY = 0;
canvas.addEventListener('touchstart', (e) => {
  if(e.touches.length === 1){
    const t = e.touches[0];
    touchStartX = t.clientX;
    touchStartY = t.clientY;
  }
});

canvas.addEventListener('touchend', (e) => {
  if(e.changedTouches.length === 1){
    const t = e.changedTouches[0];
    const dx = t.clientX - touchStartX;
    const dy = t.clientY - touchStartY;
    const absX = Math.abs(dx), absY = Math.abs(dy);
    if(Math.max(absX, absY) < 20) return; // ignore tiny taps
    if(absX > absY){
      // horizontal swipe
      if(dx > 0) state.nextDir = {r:0,c:1};
      else state.nextDir = {r:0,c:-1};
    } else {
      // vertical swipe
      if(dy > 0) state.nextDir = {r:1,c:0};
      else state.nextDir = {r:-1,c:0};
    }
    if(!state.running) startGame();
  }
});

// Buttons
document.getElementById('btnStart').addEventListener('click', ()=> {
  startGame();
});
document.getElementById('btnPause').addEventListener('click', ()=> {
  togglePause();
});
document.getElementById('btnRestart').addEventListener('click', ()=> {
  restart();
});

function startGame(){
  if(!state.running){
    state.running = true;
    state.paused = false;
  }
  // ensure loop running
  if(!gameLoopId) gameLoopId = requestAnimationFrame(loop);
}

function togglePause(){
  state.paused = !state.paused;
  document.getElementById('btnPause').innerText = state.paused ? 'Resume' : 'Pause';
  if(state.paused === false && !gameLoopId) gameLoopId = requestAnimationFrame(loop);
}

function restart(){
  cancelAnimationFrame(gameLoopId);
  gameLoopId = null;
  resetGame();
  draw();
}

// initial draw
draw();

// make canvas focusable for keyboard on mobile (if external keyboard)
canvas.addEventListener('click', ()=> canvas.focus());

// Resize handling (keep square canvas, fit to container)
function adaptSize(){
  const containerWidth = Math.min(window.innerWidth - 40, 700);
  const size = Math.min(containerWidth, window.innerHeight - 120);
  canvas.style.width = size + 'px';
  canvas.style.height = size + 'px';
}
window.addEventListener('resize', adaptSize);
adaptSize();

</script>
</body>
</html>

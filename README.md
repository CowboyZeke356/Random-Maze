# Random-Maze
a maze game that after it is solve you can click the button and randomly create another maze
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Random Maze — HTML5</title>
  <style>
    : root{
      --bg:#0f172a; /* slate-900 */
      --panel:#111827; /* gray-900 */
      --ink:#e5e7eb; /* gray-200 */
      --accent:#38bdf8; /* sky-400 */
      --good:#34d399; /* emerald-400 */
    }
    html,body{
      margin:0; height:100%; background: var(--bg); color: var(--ink); font-family:ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji","Segoe UI Emoji";}
    .wrap{display:flex; flex-direction:column; min-height:100%;}
    header{padding:14px 16px; display:flex; gap:12px; align-items:center; flex-wrap:wrap; background:linear-gradient(180deg, rgba(56,189,248,.1), rgba(17,24,39,0));}
    h1{font-size:18px; margin:0; font-weight:700; letter-spacing:.4px;}
    .controls{margin-left:auto; display:flex; gap:10px; align-items:center; flex-wrap:wrap;}
    label{font-size:14px; opacity:.9}
    select, button, input[type="number"]{
      background:var(--panel); color:var(--ink); border:1px solid #1f2937; border-radius:10px; padding:8px 10px; font-size:14px; outline:none;
    }
    button{cursor:pointer; transition:transform .06s ease;}
    button:hover{transform:translateY(-1px)}
    #newMazeBtn{border-color:#0ea5e9}
    #mazeCanvas{display:block; width:100%; height:100%; image-rendering: crisp-edges;
      /* Mobile swipe/selection tweaks */
      touch-action: none; -webkit-user-select: none; user-select: none; -webkit-tap-highlight-color: transparent;
    }
    .stage{flex:1; min-height:0; display:grid; grid-template-rows: 1fr 28px;}
    .canvasWrap{position:relative;}
    .toast{position:absolute; left:50%; top:10px; transform:translateX(-50%); background:#10b98120; color:#a7f3d0; border:1px solid #10b98155; padding:8px 12px; border-radius:10px; font-size:14px; display:none}
    .status{padding:6px 12px; font-size:13px; opacity:.85;}
    .overlay{position:absolute; inset:0; display:none; place-items:center; background:rgba(2,6,23,.5); backdrop-filter: blur(2px);} 
    .card{background:var(--panel); border:1px solid #1f2937; padding:18px; border-radius:14px; text-align:center;}
    .card h2{margin:0 0 8px; font-size:18px}
    .card p{margin:0 0 12px; font-size:14px; opacity:.9}
    .pill{display:inline-block; padding:6px 10px; border-radius:999px; border:1px dashed #065f46; background:#064e3b; color:#d1fae5; font-size:12px}
    .kbd{font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono","Courier New", monospace; background:#0b1220; border:1px solid #1f2937; padding:2px 6px; border-radius:6px}
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Random Maze</h1>
      <div class="controls">
        <label>Difficulty
          <select id="difficulty">
            <option value="10x10">Easy (10×10)</option>
            <option value="16x16" selected>Medium (16×16)</option>
            <option value="24x24">Hard (24×24)</option>
            <option value="32x20">Wide (32×20)</option>
          </select>
        </label>
        <label>Seed <input id="seed" type="number" placeholder="random" step="1" /></label>
        <button id="newMazeBtn" title="Generate a new maze">New Maze</button>
        <button id="runTestsBtn" title="Run self-tests">Run Tests</button>
      </div>
    </header>

    <div class="stage">
      <div class="canvasWrap">
        <canvas id="mazeCanvas" aria-label="Maze game" role="img"></canvas>
        <div id="toast" class="toast">Solved! Click <b>New Maze</b> for another.</div>
        <div id="overlay" class="overlay" aria-modal="true" role="dialog">
          <div class="card">
            <h2>Nice! You escaped the maze 🎉</h2>
            <p>Press <span class="kbd">Enter</span> or tap <b>New Maze</b> to play again.</p>
            <button id="nextBtn">New Maze</button>
          </div>
        </div>
      </div>
      <div class="status" id="status">Use arrow keys, WASD, or swipe.</div>
    </div>
  </div>

<script>
(function(){
  'use strict';
  const canvas = document.getElementById('mazeCanvas');
  const ctx = canvas.getContext('2d');
  const difficultyEl = document.getElementById('difficulty');
  const seedEl = document.getElementById('seed');
  const newBtn = document.getElementById('newMazeBtn');
  const runTestsBtn = document.getElementById('runTestsBtn');
  const statusEl = document.getElementById('status');
  const overlay = document.getElementById('overlay');
  const nextBtn = document.getElementById('nextBtn');
  const toast = document.getElementById('toast');

  // ---------- Utilities ----------
  function seededRandom(seed){
    // Mulberry32 PRNG — small & fast
    if (seed == null || seed === '' || Number.isNaN(Number(seed))) {
      return Math.random; // fallback to native
    }
    let t = Number(seed) >>> 0;
    return function(){
      t += 0x6D2B79F5;
      let r = Math.imul(t ^ t >>> 15, 1 | t);
      r ^= r + Math.imul(r ^ r >>> 7, 61 | r);
      return ((r ^ r >>> 14) >>> 0) / 4294967296;
    };
  }

  const DIRS = [
    {dx:0, dy:-1, wall:'top',    opp:'bottom'}, // up
    {dx:1, dy:0,  wall:'right',  opp:'left'},   // right
    {dx:0, dy:1,  wall:'bottom', opp:'top'},    // down
    {dx:-1,dy:0,  wall:'left',   opp:'right'},  // left
  ];

  class Cell{
    constructor(x,y){
      this.x=x; this.y=y; this.vis=false;
      this.walls={top:true,right:true,bottom:true,left:true};
    }
  }

  class Maze{
    constructor(cols, rows, rand){
      this.cols = cols; this.rows = rows; this.rand = rand || Math.random;
      this.grid = Array.from({length:rows}, (_,y)=>Array.from({length:cols}, (_,x)=>new Cell(x,y)));
      this.generate();
    }
    inBounds(x,y){ return x>=0 && y>=0 && x<this.cols && y<this.rows; }
    neighbors(cell){
      return DIRS.map(d=>({d, nx:cell.x+d.dx, ny:cell.y+d.dy}))
        .filter(n=>this.inBounds(n.nx,n.ny))
        .map(n=>({cell:this.grid[n.ny][n.nx], dir:n.d}));
    }
    generate(){
      // Iterative DFS (recursive backtracker)
      const stack=[];
      const start=this.grid[0][0];
      start.vis=true; stack.push(start);
      while(stack.length){
        const current = stack[stack.length-1];
        const unvis = this.neighbors(current).filter(n=>!n.cell.vis);
        if(unvis.length){
          const {cell:next, dir} = unvis[(this.rand()*unvis.length)|0];
          current.walls[dir.wall]=false;
          next.walls[dir.opp]=false;
          next.vis=true; stack.push(next);
        }else{
          stack.pop();
        }
      }
      // Reset vis flags for potential future use (e.g., solving)
      for(const row of this.grid){ for(const c of row){ c.vis=false; } }
    }
  }

  // ---------- Game State ----------
  let maze, player, goal, rand, cellSize, offsetX, offsetY;

  function newMaze(){
    const [cw,ch] = getDims();
    rand = seededRandom(seedEl.value);
    maze = new Maze(cw, ch, rand);
    player = {x:0, y:0};
    goal = {x:maze.cols-1, y:maze.rows-1};
    hideOverlay(); hideToast();
    resizeCanvas();
    draw();
    announce(`New maze: ${maze.cols}×${maze.rows}.`);
  }

  function getDims(){
    const val = difficultyEl.value;
    if(val.includes('x')){
      const [a,b] = val.split('x').map(Number);
      return [a,b];
    }
    return [16,16];
  }

  // ---------- Rendering ----------
  function resizeCanvas(){
    const wrap = canvas.parentElement.getBoundingClientRect();
    const pad = 20; // visual breathing room
    const availW = Math.max(200, wrap.width - pad);
    const availH = Math.max(200, wrap.height - pad);
    const cs = Math.floor(Math.min(availW/maze.cols, availH/maze.rows));
    cellSize = Math.max(8, cs);
    canvas.width = maze.cols * cellSize + 1; // +1 to close last wall stroke
    canvas.height = maze.rows * cellSize + 1;
    // center within wrapper
    const cw = canvas.width, ch = canvas.height;
    offsetX = Math.max(0, (wrap.width - cw)/2); 
    offsetY = Math.max(0, (wrap.height - ch)/2);
    canvas.style.marginLeft = `${offsetX}px`;
    canvas.style.marginTop = `${Math.max(0, offsetY-10)}px`;
  }

  function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);
    ctx.lineWidth = 2; ctx.strokeStyle = getComputedStyle(document.documentElement).getPropertyValue('--ink');
    // draw walls
    for(let y=0; y<maze.rows; y++){
      for(let x=0; x<maze.cols; x++){
        const c = maze.grid[y][x];
        const px = x*cellSize; const py = y*cellSize;
        if(c.walls.top){ line(px,py, px+cellSize, py); }
        if(c.walls.right){ line(px+cellSize,py, px+cellSize, py+cellSize); }
        if(c.walls.bottom){ line(px,py+cellSize, px+cellSize, py+cellSize); }
        if(c.walls.left){ line(px,py, px, py+cellSize); }
      }
    }
    // start + goal
    fillCell(0,0, 'rgba(56,189,248,.15)'); // start highlight
    fillCell(goal.x, goal.y, 'rgba(52,211,153,.18)');

    // player
    drawPlayer();
  }

  function line(x1,y1,x2,y2){ ctx.beginPath(); ctx.moveTo(x1+.5,y1+.5); ctx.lineTo(x2+.5,y2+.5); ctx.stroke(); }
  function fillCell(x,y, color){ ctx.fillStyle=color; ctx.fillRect(x*cellSize+2,y*cellSize+2, cellSize-3, cellSize-3); }
  function drawPlayer(){
    const cx = player.x*cellSize + cellSize/2;
    const cy = player.y*cellSize + cellSize/2;
    const r = Math.max(3, cellSize*0.28);
    ctx.beginPath(); ctx.arc(cx,cy,r,0,Math.PI*2);
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--accent');
    ctx.fill();
  }

  // ---------- Movement ----------
  function canMove(dx,dy){
    const x = player.x, y = player.y;
    const nx = x+dx, ny = y+dy;
    if(nx<0||ny<0||nx>=maze.cols||ny>=maze.rows) return false;
    const here = maze.grid[y][x];
    if(dx===1 && here.walls.right) return false;
    if(dx===-1 && here.walls.left) return false;
    if(dy===1 && here.walls.bottom) return false;
    if(dy===-1 && here.walls.top) return false;
    return true;
  }

  function move(dx,dy){
    if(!canMove(dx,dy)) return;
    player.x += dx; player.y += dy; draw();
    if(player.x===goal.x && player.y===goal.y){
      solved();
    }
  }

  function solved(){
    showToast(); showOverlay();
    announce('Solved!');
  }

  // ---------- UX helpers ----------
  function announce(msg){ statusEl.textContent = msg; }
  function showOverlay(){ overlay.style.display='grid'; }
  function hideOverlay(){ overlay.style.display='none'; }
  function showToast(){ toast.style.display='block'; clearTimeout(showToast._t); showToast._t = setTimeout(()=>toast.style.display='none', 2500); }
  function hideToast(){ toast.style.display='none'; }

  // ---------- Self Tests ----------
  function runTests(){
    const results = [];
    const assert = (name, condition) => results.push({name, pass: !!condition});

    // Test 1: PRNG determinism
    const r1 = seededRandom(123), r2 = seededRandom(123);
    let det = true; for(let i=0;i<10;i++){ if(r1() !== r2()){ det = false; break; } }
    assert('PRNG returns same sequence for same seed', det);

    // Test 2: Maze wall reciprocity
    const m = new Maze(10,10, seededRandom(7));
    let reciprocal = true;
    for(let y=0;y<m.rows;y++){
      for(let x=0;x<m.cols;x++){
        const c = m.grid[y][x];
        const nbrs = m.neighbors(c);
        for(const {cell:nc, dir} of nbrs){
          if(c.walls[dir.wall] !== nc.walls[dir.opp]){ reciprocal = false; break; }
        }
      }
    }
    assert('Adjacent walls are reciprocal', reciprocal);

    // Test 3: Path exists from start to goal
    function reachable(mm){
      const seen = new Set(['0,0']);
      const q=[[0,0]];
      while(q.length){
        const [x,y]=q.shift();
        if(x===mm.cols-1 && y===mm.rows-1) return true;
        const c = mm.grid[y][x];
        if(!c.walls.top && y>0){ const k=`${x},${y-1}`; if(!seen.has(k)){ seen.add(k); q.push([x,y-1]); } }
        if(!c.walls.right && x<mm.cols-1){ const k=`${x+1},${y}`; if(!seen.has(k)){ seen.add(k); q.push([x+1,y]); } }
        if(!c.walls.bottom && y<mm.rows-1){ const k=`${x},${y+1}`; if(!seen.has(k)){ seen.add(k); q.push([x,y+1]); } }
        if(!c.walls.left && x>0){ const k=`${x-1},${y}`; if(!seen.has(k)){ seen.add(k); q.push([x-1,y]); } }
      }
      return false;
    }
    assert('Start to goal is reachable', reachable(m));

    // Test 4: Same seed -> identical mazes
    const mA = new Maze(8,8, seededRandom(42));
    const mB = new Maze(8,8, seededRandom(42));
    let identical = true;
    for(let y=0;y<8;y++){
      for(let x=0;x<8;x++){
        const a=mA.grid[y][x].walls, b=mB.grid[y][x].walls;
        if(a.top!==b.top||a.right!==b.right||a.bottom!==b.bottom||a.left!==b.left){ identical=false; break; }
      }
    }
    assert('Mazes with same seed are identical', identical);

    // Test 5: Cell size lower bound respected after resize
    resizeCanvas();
    assert('Cell size >= 8px', cellSize >= 8);

    // Report
    const passed = results.filter(r=>r.pass).length;
    console.group('%cMaze Self-Tests','color:#10b981');
    results.forEach(r=> console[r.pass?'log':'error'](`${r.pass?'✔':'✖'} ${r.name}`));
    console.groupEnd();
    showToastMsg(`${passed}/${results.length} tests passed. See console for details.`);
  }

  // Allow showing custom toast messages
  function showToastMsg(msg){
    toast.innerHTML = msg; showToast();
    // restore default after fade
    clearTimeout(showToastMsg._r);
    showToastMsg._r = setTimeout(()=>{ toast.innerHTML = 'Solved! Click <b>New Maze</b> for another.'; }, 3000);
  }

  // ---------- Events ----------
  window.addEventListener('resize', ()=>{ resizeCanvas(); draw(); });
  newBtn.addEventListener('click', newMaze);
  nextBtn.addEventListener('click', newMaze);
  difficultyEl.addEventListener('change', newMaze);
  runTestsBtn.addEventListener('click', runTests);

  window.addEventListener('keydown', (e)=>{
    const k = e.key.toLowerCase();
    if(k==='arrowup' || k==='w') move(0,-1);
    else if(k==='arrowright' || k==='d') move(1,0);
    else if(k==='arrowdown' || k==='s') move(0,1);
    else if(k==='arrowleft' || k==='a') move(-1,0);
    else if(k==='enter' && overlay.style.display==='grid') newMaze();
  });

  // Touch swipe
  let tStart=null;
  canvas.addEventListener('touchstart', (e)=>{ const t=e.changedTouches[0]; tStart={x:t.clientX,y:t.clientY}; });
  canvas.addEventListener('touchend', (e)=>{
    if(!tStart) return; const t=e.changedTouches[0];
    const dx=t.clientX - tStart.x, dy=t.clientY - tStart.y; tStart=null;
    if(Math.hypot(dx,dy) < 12) return;
    if(Math.abs(dx) > Math.abs(dy)) move(dx>0?1:-1,0); else move(0, dy>0?1:-1);
  }, {passive:true});

  // Boot
  newMaze();
})();
</script>
</body>
</html>

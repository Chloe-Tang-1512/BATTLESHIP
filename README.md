# BATTLESHIP
A digital version of the board game "Battleship"

<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Battleship — Single File (Player vs AI)</title>
<style>
  :root{
    --size: 36px;
    --gap: 8px;
    --bg: #0b1740;
    --panel: #0f254f;
    --accent: #3ec5ff;
    --hit: #ff5959;
    --miss: #9fb4d8;
    --ship: #2b9a6a;
    color-scheme: dark;
  }
  *{box-sizing:border-box;font-family:Inter,system-ui,Segoe UI,Roboto,"Helvetica Neue",Arial;}
  body{margin:18px;background:linear-gradient(180deg,#07112b 0%, #071a3b 100%);color:#e6f0ff;display:flex;flex-direction:column;align-items:center;gap:18px}
  h1{margin:0;font-size:20px}
  .top{display:flex;gap:16px;align-items:center}
  .board-wrap{display:flex;gap:20px;flex-wrap:wrap;justify-content:center}
  .panel{background:rgba(255,255,255,0.04);padding:14px;border-radius:10px;box-shadow:0 6px 18px rgba(2,6,23,0.6);min-width:calc(var(--size)*10 + var(--gap)*2 + 40px)}
  .grid{display:grid;grid-template-columns:repeat(10, var(--size));grid-template-rows:repeat(10,var(--size));gap:4px;padding:10px;background:linear-gradient(180deg,rgba(255,255,255,0.02),transparent);border-radius:8px}
  .cell{width:var(--size);height:var(--size);display:flex;align-items:center;justify-content:center;background:linear-gradient(180deg,#0d2a4a,#08203a);border-radius:6px;cursor:pointer;user-select:none;position:relative}
  .cell.disabled{cursor:not-allowed;opacity:0.85}
  .cell .dot{width:70%;height:70%;border-radius:50%;opacity:0.95;transform:scale(0.9)}
  .miss .dot{background:var(--miss)}
  .hit .dot{background:var(--hit);box-shadow:0 0 10px rgba(255,89,89,0.7)}
  .ship .dot{background:var(--ship);opacity:0.6}
  .legend{display:flex;gap:8px;align-items:center;margin-top:8px}
  .legend .item{display:flex;gap:6px;align-items:center;font-size:13px}
  .small{font-size:13px;color:#bcd7ff;opacity:0.9}
  button{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;color:#012238;font-weight:600;cursor:pointer}
  button.ghost{background:transparent;border:1px solid rgba(255,255,255,0.06);color:var(--accent)}
  .log{min-width:300px;max-width:420px;min-height:96px;padding:10px;background:rgba(0,0,0,0.18);border-radius:8px;overflow:auto;font-size:13px}
  .status{font-weight:700}
  .controls{display:flex;gap:8px;align-items:center;flex-wrap:wrap}
  .footer{font-size:12px;color:#9fb4d8;opacity:0.9}
  @media (max-width:800px){
    .board-wrap{flex-direction:column;align-items:center}
  }
</style>
</head>
<body>
  <div class="top">
    <div>
      <h1>Battleship — Player vs AI</h1>
      <div class="small">Click enemy grid to fire. AI uses hunt/target strategy.</div>
    </div>
    <div style="margin-left:12px" class="controls">
      <button id="randomPlaceBtn">Random place</button>
      <button id="startBtn">Start Game</button>
      <button id="restartBtn" class="ghost">Restart</button>
    </div>
  </div>

  <div class="board-wrap">
    <div class="panel">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div><strong>Your board</strong><div class="small">(shows your ships)</div></div>
        <div class="status" id="turnIndicator">Waiting to start</div>
      </div>
      <div id="playerBoard" class="grid" aria-label="Your board"></div>
      <div class="legend">
        <div class="item"><div style="width:14px;height:14px;border-radius:3px;background:var(--ship)"></div> Your Ship</div>
        <div class="item"><div style="width:14px;height:14px;border-radius:50%;background:var(--hit)"></div> Hit</div>
        <div class="item"><div style="width:14px;height:14px;border-radius:50%;background:var(--miss)"></div> Miss</div>
      </div>
    </div>

    <div class="panel">
      <strong>Enemy board</strong>
      <div class="small">(click to fire — fog of war)</div>
      <div id="enemyBoard" class="grid" aria-label="Enemy board"></div>
      <div style="margin-top:8px">
        <div class="log" id="log">Game log</div>
      </div>
    </div>
  </div>

  <div class="footer">Small, fast, and delightfully brutal. — tip: Random place then Start</div>

<script>
/* ======= Configuration ======= */
const GRID_SIZE = 10;
const SHIP_TYPES = [
  { name: "Carrier", size: 5 },
  { name: "Battleship", size: 4 },
  { name: "Cruiser", size: 3 },
  { name: "Submarine", size: 3 },
  { name: "Destroyer", size: 2 },
];

/* ======= State ======= */
let player = null;
let ai = null;
let gameStarted = false;
let currentTurn = null; // 'player' or 'ai'
const logEl = document.getElementById('log');
const turnIndicator = document.getElementById('turnIndicator');

/* ======= Utilities ======= */
function coordsToKey(r,c){ return `${r},${c}`; }
function randInt(max){ return Math.floor(Math.random()*max); }
function shuffle(arr){ for(let i=arr.length-1;i>0;i--){const j = Math.floor(Math.random()*(i+1));[arr[i],arr[j]]=[arr[j],arr[i]];} return arr; }
function log(msg){ const d = document.createElement('div'); d.textContent = msg; logEl.prepend(d); }

/* ======= Board / Ship helpers ======= */
function emptyBoard(){
  const b = new Array(GRID_SIZE).fill(null).map(()=> new Array(GRID_SIZE).fill(null));
  return b;
}

function makePlayer(name){
  return {
    name,
    board: emptyBoard(),
    ships: {}, // id -> { type, coords: [ [r,c], ... ], hits: 0, sunk: false }
    shotsFired: new Set(), // keys of coords we've fired at (for the shooter)
  };
}

/* Place ships randomly; ensures no overlap */
function placeShipsRandomly(target){
  // reset board & ships
  target.board = emptyBoard();
  target.ships = {};
  let idCounter=1;
  for(const type of SHIP_TYPES){
    let placed=false, tries=0;
    while(!placed && tries++ < 500){
      const horiz = Math.random() < 0.5;
      const r = randInt(GRID_SIZE - (horiz ? 0 : type.size -1));
      const c = randInt(GRID_SIZE - (horiz ? type.size -1 : 0));
      const coords = [];
      let blocked=false;
      for(let i=0;i<type.size;i++){
        const rr = r + (horiz?0:i);
        const cc = c + (horiz?i:0);
        if(target.board[rr][cc] !== null){ blocked=true; break; }
        coords.push([rr,cc]);
      }
      if(blocked) continue;
      const id = 'S'+(idCounter++);
      target.ships[id] = { id, type: type.name, size: type.size, coords, hits: 0, sunk: false };
      for(const [rr,cc] of coords) target.board[rr][cc] = id;
      placed=true;
    }
    if(!placed) throw new Error('Failed to place ships (rare)');
  }
}

/* Check shot at target by shooter */
function resolveShot(target, r, c){
  const cell = target.board[r][c];
  if(cell === null){
    target.board[r][c] = 'miss';
    return { result: 'miss' };
  }
  if(cell === 'miss' || cell === 'hit') return { result: 'already' };
  // it's a ship id
  const ship = target.ships[cell];
  ship.hits++;
  target.board[r][c] = 'hit';
  if(ship.hits >= ship.size){ ship.sunk = true; return { result: 'sunk', shipName: ship.type }; }
  return { result: 'hit', shipName: ship.type };
}

function allSunk(target){
  return Object.values(target.ships).every(s=>s.sunk);
}

/* ======= UI Rendering ======= */
const playerBoardEl = document.getElementById('playerBoard');
const enemyBoardEl  = document.getElementById('enemyBoard');

function buildGridEl(boardEl, onCellClick, revealShips=false, owner='player'){
  boardEl.innerHTML = '';
  for(let r=0;r<GRID_SIZE;r++){
    for(let c=0;c<GRID_SIZE;c++){
      const cell = document.createElement('div');
      cell.className = 'cell';
      cell.dataset.r = r; cell.dataset.c = c;
      const dot = document.createElement('div'); dot.className = 'dot'; cell.appendChild(dot);
      // initial visuals
      const val = (owner==='player' ? player.board[r][c] : null);
      if(revealShips && owner==='player' && val && val !== 'hit' && val !== 'miss'){
        cell.classList.add('ship');
      }
      boardEl.appendChild(cell);
      if(onCellClick) cell.addEventListener('click', onCellClick);
    }
  }
}

function refreshBoards(revealPlayerShips=false){
  // player board
  const playerCells = playerBoardEl.querySelectorAll('.cell');
  playerCells.forEach(div=>{
    const r = +div.dataset.r, c = +div.dataset.c;
    div.classList.remove('hit','miss','ship','disabled');
    const val = player.board[r][c];
    if(val === 'hit') div.classList.add('hit');
    else if(val === 'miss') div.classList.add('miss');
    else if(val && val !== 'hit' && val !== 'miss' && revealPlayerShips) div.classList.add('ship');
  });
  // enemy board (fog)
  const enemyCells = enemyBoardEl.querySelectorAll('.cell');
  enemyCells.forEach(div=>{
    const r = +div.dataset.r, c = +div.dataset.c;
    div.classList.remove('hit','miss','disabled','ship');
    const val = ai.board[r][c];
    // show only hits and misses
    if(val === 'hit') div.classList.add('hit');
    else if(val === 'miss') div.classList.add('miss');
  });
  // disable clicks if not player's turn or game not started
  const enemyClickable = gameStarted && currentTurn === 'player';
  enemyCells.forEach(div=>{
    if(!enemyClickable) div.classList.add('disabled');
    else div.classList.remove('disabled');
  });
  turnIndicator.textContent = gameStarted ? (currentTurn==='player' ? "Your turn" : "AI thinking...") : "Waiting to start";
}

/* ======= Game flow ======= */
function startNewGame(){
  player = makePlayer('You');
  ai = makePlayer('AI');
  placeShipsRandomly(player);
  placeShipsRandomly(ai);
  gameStarted = false;
  currentTurn = null;
  logEl.innerHTML = '';
  log('Ships placed. Press Start Game.');
  buildGridEl(playerBoardEl, null, true, 'player');
  buildGridEl(enemyBoardEl, onEnemyCellClick, false, 'ai');
  refreshBoards(true);
}

function beginPlay(){
  if(!player || !ai) { log('No setup yet.'); return; }
  gameStarted = true;
  // random who starts
  currentTurn = (Math.random() < 0.5) ? 'player' : 'ai';
  log(`Game start — ${currentTurn === 'player' ? 'You' : 'AI'} goes first.`);
  refreshBoards(true);
  if(currentTurn === 'ai') setTimeout(aiTakeTurn, 600);
}

/* ======= Player Actions ======= */
function onEnemyCellClick(e){
  if(!gameStarted || currentTurn !== 'player') return;
  const r = +thisOrTarget(e).dataset.r;
  const c = +thisOrTarget(e).dataset.c;
  const key = coordsToKey(r,c);
  if(player.shotsFired.has(key)) { log('You already fired there.'); return; }
  player.shotsFired.add(key);
  const res = resolveShot(ai, r, c);
  if(res.result === 'already'){ log('Already shot that cell (weird).'); return; }
  if(res.result === 'miss'){
    log(`You fired at ${r},${c} — Miss.`);
  } else if(res.result === 'hit'){
    log(`You fired at ${r},${c} — Hit ${res.shipName}!`);
  } else if(res.result === 'sunk'){
    log(`You fired at ${r},${c} — Sunk enemy ${res.shipName}!`);
  }
  refreshBoards(false);
  if(allSunk(ai)){
    log('🎉 You win! All enemy ships sunk.');
    endGame('player');
    return;
  }
  // change turn
  currentTurn = 'ai';
  refreshBoards(false);
  setTimeout(aiTakeTurn, 700);
}

/* small helper to allow clicks on inner dot */
function thisOrTarget(e){
  return e.target.classList.contains('cell') ? e.target : e.target.closest('.cell');
}

/* ======= AI: Hunt/Target strategy ======= */
const aiBrain = {
  mode: 'hunt', // or 'target'
  huntedTargets: [], // cells AI will try in hunt mode (checkerboard)
  targetStack: [], // candidate cells to resolve a found ship (LIFO)
  hitHistory: [], // coords of recent contiguous hits on current ship
  init(){
    this.mode = 'hunt';
    this.huntedTargets = [];
    this.targetStack = [];
    this.hitHistory = [];
    // create checkerboard pattern list to be efficient in hunt
    for(let r=0;r<GRID_SIZE;r++){
      for(let c=0;c<GRID_SIZE;c++){
        if((r+c)%2===0) this.huntedTargets.push([r,c]);
      }
    }
    // fill remaining cells too for late-game completeness
    for(let r=0;r<GRID_SIZE;r++) for(let c=0;c<GRID_SIZE;c++) if((r+c)%2!==0) this.huntedTargets.push([r,c]);
    shuffle(this.huntedTargets);
  },
  nextHunt(){
    while(this.huntedTargets.length){
      const [r,c] = this.huntedTargets.pop();
      if(ai.shotsFired.has(coordsToKey(r,c))) continue;
      return [r,c];
    }
    // fallback: find first unshot cell
    for(let r=0;r<GRID_SIZE;r++) for(let c=0;c<GRID_SIZE;c++) if(!ai.shotsFired.has(coordsToKey(r,c))) return [r,c];
    return null;
  },
  pushNeighbors(r,c){
    const deltas = [[1,0],[-1,0],[0,1],[0,-1]];
    for(const [dr,dc] of deltas){
      const nr=r+dr, nc=c+dc;
      if(nr<0||nc<0||nr>=GRID_SIZE||nc>=GRID_SIZE) continue;
      const k = coordsToKey(nr,nc);
      if(ai.shotsFired.has(k)) continue;
      // avoid duplicates in stack
      const exists = this.targetStack.some(([a,b]) => a===nr && b===nc);
      if(!exists) this.targetStack.push([nr,nc]);
    }
  },
  popTarget(){
    while(this.targetStack.length){
      const [r,c] = this.targetStack.pop();
      if(ai.shotsFired.has(coordsToKey(r,c))) continue;
      return [r,c];
    }
    return null;
  }
};

function aiTakeTurn(){
  if(!gameStarted || currentTurn !== 'ai') return;
  // AI chooses a cell according to brain
  if(!aiBrain.huntedTargets.length) aiBrain.init();
  let choice = null;
  if(aiBrain.mode === 'hunt'){
    choice = aiBrain.nextHunt();
  } else {
    choice = aiBrain.popTarget() || aiBrain.nextHunt();
  }
  if(!choice){
    log('AI: no moves left (weird).');
    endGame('draw');
    return;
  }
  const [r,c] = choice;
  ai.shotsFired.add(coordsToKey(r,c));
  const res = resolveShot(player, r, c);
  if(res.result === 'miss'){
    log(`AI fired at ${r},${c} — Miss.`);
    // if in target mode and no target stack left, revert to hunt
    if(aiBrain.mode === 'target' && aiBrain.targetStack.length === 0) {
      aiBrain.mode = 'hunt';
      aiBrain.hitHistory = [];
    }
  } else if(res.result === 'hit'){
    log(`AI fired at ${r},${c} — Hit your ${res.shipName}!`);
    aiBrain.mode = 'target';
    aiBrain.hitHistory.push([r,c]);
    aiBrain.pushNeighbors(r,c);
  } else if(res.result === 'sunk'){
    log(`AI fired at ${r},${c} — Sunk your ${res.shipName}!`);
    // wiped current ship tracking
    aiBrain.mode = 'hunt';
    aiBrain.targetStack = [];
    aiBrain.hitHistory = [];
  }
  refreshBoards(true);
  if(allSunk(player)){
    log('💀 AI wins. Better luck next fleet.');
    endGame('ai');
    return;
  }
  // back to player
  currentTurn = 'player';
  refreshBoards(false);
}

/* ======= End / Restart ======= */
function endGame(winner){
  gameStarted = false;
  currentTurn = null;
  turnIndicator.textContent = winner === 'player' ? 'You win!' : (winner === 'ai' ? 'AI wins' : 'Game ended');
  // reveal ships
  refreshBoards(true);
  // disable enemy clicks
  const enemyCells = enemyBoardEl.querySelectorAll('.cell');
  enemyCells.forEach(div => div.classList.add('disabled'));
}

/* ======= Buttons ======= */
document.getElementById('randomPlaceBtn').addEventListener('click', ()=>{
  if(!player) startNewGame();
  placeShipsRandomly(player);
  buildGridEl(playerBoardEl, null, true, 'player');
  refreshBoards(true);
  log('Player ships randomized.');
});

document.getElementById('startBtn').addEventListener('click', ()=>{
  if(!player) startNewGame();
  aiBrain.init();
  beginPlay();
});
document.getElementById('restartBtn').addEventListener('click', ()=> startNewGame());

/* ======= Init ======= */
startNewGame();
</script>
</body>
</html>

**<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Two-Stage Drag Game (Pages)</title>
<style>
  :root{
    --bg:#f3f6fb; --card:#ffffff; --accent:#ff6b6b; --muted:#556;
  }
  *{box-sizing:border-box}
  body{
    margin:0; font-family:Inter, Arial, Helvetica, sans-serif;
    background:linear-gradient(180deg,#e9f0fb,#f6fbff);
    color:#123;
    min-height:100vh;
    display:flex; align-items:center; justify-content:center;
    padding:20px;
  }

  .page{
    width:100%; max-width:760px; background:var(--card);
    border-radius:12px; box-shadow:0 14px 40px rgba(10,20,40,0.08);
    padding:28px; text-align:center;
  }
  .hidden{display:none}

  /* Start page */
  #startPage h1{font-size:38px;margin:22px 0 8px}
  #startPage p{color:var(--muted);margin-bottom:18px}
  .btn{
    display:inline-block;padding:12px 22px;border-radius:10px;border:0;
    background:linear-gradient(90deg,var(--accent),#ff8a8a); color:#051018;
    font-weight:700;font-size:16px;cursor:pointer;
  }

  /* Stage layout */
  #timer{font-weight:700;margin:6px 0 18px}
  .row{display:flex;gap:18px;align-items:flex-start;justify-content:center;flex-wrap:wrap}
  .itemsContainer{width:320px; min-height:180px; display:flex;flex-wrap:wrap; gap:10px; align-content:flex-start; justify-content:center}
  .draggable{
    width:80px;height:80px;border-radius:8px;border:2px solid rgba(0,0,0,0.06);
    background:#fff;display:flex;align-items:center;justify-content:center;
    cursor:grab; overflow:hidden;
  }
  .draggable img{width:100%;height:100%;object-fit:cover; display:block}

  .bigDrop{
    width:320px;height:320px;border-radius:10px;border:3px dashed rgba(0,0,0,0.12);
    background:linear-gradient(180deg,#ffffff,#f7fbff); display:flex;align-items:center;justify-content:center;
    color:var(--muted); font-weight:700;
  }

  .lineSlots{width:100%; display:flex; gap:12px; justify-content:center; margin-top:18px}
  .slot{
    width:90px;height:90px;border-radius:8px;border:2px dashed rgba(0,0,0,0.12);
    display:flex;align-items:center;justify-content:center;background:#fff;
  }

  /* Result */
  #resultVideo{width:100%;max-width:640px;margin-top:8px;border-radius:8px}

  footer{font-size:13px;color:var(--muted);margin-top:14px}
</style>
</head>
<body>

<div class="page" id="startPage">
  <h1>DRAG & DROP CHALLENGE</h1>
  <p>Two stages. Complete both within time. Laugh a little if you fail.</p>
  <button class="btn" id="startBtn">Start Game</button>
  <footer>Replace images in <code>images/</code> and add <code>success.mp4</code>/<code>fail.mp4</code> in same folder.</footer>
</div>

<!-- STAGE 1 PAGE -->
<div class="page hidden" id="stage1Page">
  <h2>Stage 1 — Put ALL items into the box</h2>
  <div id="timer1" class="timer" aria-live="polite">Time left: <span id="t1">20</span>s</div>

  <div class="row" style="margin-top:12px;">
    <div class="itemsContainer" id="stage1Items"></div>

    <div id="stage1Drop" class="bigDrop" aria-dropeffect="move">
      DROP HERE
    </div>
  </div>
</div>

<!-- STAGE 2 PAGE -->
<div class="page hidden" id="stage2Page">
  <h2>Stage 2 — Arrange the items in the line</h2>
  <div id="timer2" class="timer" aria-live="polite">Time left: <span id="t2">20</span>s</div>

  <div style="margin-top:12px">
    <!-- same big drop (start location) -->
    <div id="stage2Start" class="bigDrop">START ZONE (items begin here)</div>
  </div>

  <div class="lineSlots" id="lineSlots" style="margin-top:18px"></div>
</div>

<!-- RESULT PAGE -->
<div class="page hidden" id="resultPage">
  <h2 id="resultTitle">Result</h2>
  <video id="resultVideo" controls></video>
  <div style="margin-top:14px">
    <button class="btn" id="retryBtn">Play Again</button>
  </div>
</div>

<script>
/*
  Flow:
   - startPage -> on start -> stage1Page (20s)
   - Stage1: drag all 5 items into single drop (stage1Drop). If done before timer -> go to stage2Page
   - Stage2: same 5 items start inside stage2Start (the same location). Drag them to five separate line slots. If all slots filled before timer -> success video, else fail video
   - resultPage shows appropriate video
*/

const ITEM_COUNT = 5;
const STAGE_TIME = 20; // seconds per stage (adjust for demo)
const images = [
  'images/obj1.png',
  'images/obj2.png',
  'images/obj3.png',
  'images/obj4.png',
  'images/obj5.png'
];
// fallback placeholders if images missing
const placeholder = i => `https://via.placeholder.com/100?text=Obj${i+1}`;

let items = [];            // <img> elements
let timerId = null;
let remaining = STAGE_TIME;

// pages
const startPage = document.getElementById('startPage');
const stage1Page = document.getElementById('stage1Page');
const stage2Page = document.getElementById('stage2Page');
const resultPage = document.getElementById('resultPage');
const startBtn = document.getElementById('startBtn');
const retryBtn = document.getElementById('retryBtn');

const stage1Items = document.getElementById('stage1Items');
const stage1Drop = document.getElementById('stage1Drop');
const timer1Span = document.getElementById('t1');

const stage2Start = document.getElementById('stage2Start');
const timer2Span = document.getElementById('t2');
const lineSlots = document.getElementById('lineSlots');

const resultTitle = document.getElementById('resultTitle');
const resultVideo = document.getElementById('resultVideo');

/* Utility: show only one page */
function showPage(page){
  [startPage, stage1Page, stage2Page, resultPage].forEach(p => p.classList.add('hidden'));
  page.classList.remove('hidden');
}

/* Create draggable items once */
function createItems(){
  items = [];
  for(let i=0;i<ITEM_COUNT;i++){
    const img = document.createElement('img');
    img.id = 'item-'+i;
    img.className = 'draggable';
    // use provided image if present or fallback placeholder
    img.src = (typeof images[i] === 'string') ? images[i] : placeholder(i);
    img.alt = 'Obj ' + (i+1);
    img.draggable = true;
    img.addEventListener('dragstart', onDragStart);
    // prevent default image drag ghost in some browsers
    img.addEventListener('mousedown', ()=> img.style.cursor = 'grabbing');
    img.addEventListener('mouseup', ()=> img.style.cursor = 'grab');
    items.push(img);
  }
}

/* Drag handlers */
function onDragStart(e){
  // store the id
  e.dataTransfer.setData('text/plain', e.target.id);
}

/* Stage timer */
function startStageTimer(stageNum){
  clearInterval(timerId);
  remaining = STAGE_TIME;
  if(stageNum === 1) timer1Span.textContent = remaining;
  else timer2Span.textContent = remaining;

  timerId = setInterval(()=>{
    remaining--;
    if(stageNum === 1) timer1Span.textContent = remaining;
    else timer2Span.textContent = remaining;

    if(remaining <= 0){
      clearInterval(timerId);
      // timeout -> fail
      gotoResult(false);
    }
  }, 1000);
}

/* --- STAGE 1 setup & logic --- */
function setupStage1(){
  showPage(stage1Page);
  // clear containers
  stage1Items.innerHTML = '';
  stage1Drop.innerHTML = 'DROP HERE';

  // create and show items on left area
  // create items if not already
  if(items.length === 0) createItems();

  items.forEach((img, idx) => {
    // append a wrapper to maintain spacing
    const wrap = document.createElement('div');
    wrap.style.display = 'inline-block';
    wrap.appendChild(img);
    stage1Items.appendChild(wrap);
  });

  // Prepare drop zone listeners
  stage1Drop.addEventListener('dragover', e => e.preventDefault());
  stage1Drop.addEventListener('drop', stage1DropHandler);

  // start timer
  startStageTimer(1);
}

function stage1DropHandler(e){
  e.preventDefault();
  const id = e.dataTransfer.getData('text/plain');
  const el = document.getElementById(id);
  if(!el) return;
  // append the item into drop (allow duplicates only once)
  // move element (it will be removed from previous parent automatically)
  e.currentTarget.appendChild(el);

  // check if all items are inside stage1Drop
  if(stage1Drop.querySelectorAll('img').length === ITEM_COUNT){
    // success stage1 -> proceed to stage2
    clearInterval(timerId);
    // small delay so UI shows full box, then start next
    setTimeout(()=> setupStage2(), 400);
  }
}

/* --- STAGE 2 setup & logic --- */
function setupStage2(){
  showPage(stage2Page);
  // clear previous
  lineSlots.innerHTML = '';
  stage2Start.innerHTML = 'START ZONE (items start here)';

  // create 5 line slots
  for(let i=0;i<ITEM_COUNT;i++){
    const slot = document.createElement('div');
    slot.className = 'slot';
    slot.dataset.index = i;
    slot.addEventListener('dragover', e => e.preventDefault());
    slot.addEventListener('drop', onSlotDrop);
    lineSlots.appendChild(slot);
  }

  // Put all items into stage2Start (same place)
  items.forEach(img => {
    stage2Start.appendChild(img);
  });

  // start timer
  startStageTimer(2);
}

/* When slot receives an item */
function onSlotDrop(e){
  e.preventDefault();
  const id = e.dataTransfer.getData('text/plain');
  const el = document.getElementById(id);
  if(!el) return;
  // Only accept if slot empty
  if(e.currentTarget.children.length === 0){
    e.currentTarget.appendChild(el);
  }

  // check success: every slot has one child
  const filled = Array.from(lineSlots.children).every(slot => slot.children.length === 1);
  if(filled){
    clearInterval(timerId);
    // success -> show success video
    gotoResult(true);
  }
}

/* Show result page with proper video */
function gotoResult(success){
  showPage(resultPage);
  resultTitle.textContent = success ? "Success!" : "Time's up - Fail!";
  // set video source - replace with your files
  resultVideo.src = success ? 'success.mp4' : 'fail.mp4';
  resultVideo.style.display = 'block';
  // try to autoplay (may be blocked by browser policies unless user interacted)
  resultVideo.play().catch(()=>{/* autoplay blocked; user can press play */});
}

/* Restart for retry */
function resetAndStart(){
  // move items array back to empty
  // recreate items fresh (to avoid odd parents)
  items = [];
  createItems();
  // show start page again
  showPage(startPage);
}

/* Hook buttons */
startBtn.addEventListener('click', ()=> {
  // ensure items created
  if(items.length === 0) createItems();
  // start stage1
  setupStage1();
});

retryBtn.addEventListener('click', ()=> {
  resetAndStart();
});

/* Allow dragging back from drop zones - add dragstart listener is already on images */
document.addEventListener('DOMContentLoaded', ()=> {
  showPage(startPage);
  // create items but don't attach anywhere yet
  createItems();
});
</script>
</body>
</html>

**

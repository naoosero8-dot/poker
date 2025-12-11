<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>オセロニアポーカー — ローカル & GitHub 同期版</title>
<style>
  :root{--bg:#f6f6f8;--card:#fff;--accent:#2b6df6}
  body{font-family:system-ui,-apple-system,"Hiragino Kaku Gothic ProN","メイリオ",sans-serif;margin:0;background:var(--bg);color:#111}
  header{background:#111;color:#fff;padding:12px 16px}
  .wrap{max-width:1100px;margin:18px auto;padding:14px;background:#fff;border-radius:8px;box-shadow:0 6px 24px rgba(0,0,0,.06)}
  h1{margin:0 0 12px 0;font-size:1.15rem}
  .row{display:flex;gap:12px;align-items:center}
  .col{display:flex;flex-direction:column;gap:8px}
  input,select,textarea,button{padding:8px;border-radius:6px;border:1px solid #ccc}
  button{cursor:pointer;background:#fafafa}
  .btn-primary{background:var(--accent);color:#fff;border:none}
  .hidden{display:none}
  .card-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(90px,1fr));gap:8px}
  .card{background:var(--card);border:1px solid #ddd;border-radius:6px;padding:6px;text-align:center;font-size:12px}
  .small{font-size:12px;color:#666}
  .status{padding:8px;background:#f2f8ff;border-radius:6px;border:1px solid #dfefff}
  .notice{padding:8px;background:#fff7e6;border-radius:6px;border:1px solid #ffe6a8;color:#664400}
  .log{height:160px;overflow:auto;border:1px solid #eee;padding:8px;background:#fafafa}
  textarea{width:100%;min-height:64px}
  label.block{display:block;margin-bottom:6px}
  .controls{display:flex;gap:8px;flex-wrap:wrap}
  footer{font-size:12px;color:#666;margin-top:12px}
  .selected{outline:3px solid rgba(43,109,246,.18)}
</style>
</head>
<body>
<header><div class="wrap"><h1>オセロニアポーカー — ローカル & GitHub 同期版</h1></div></header>
<div class="wrap">

  <!-- GitHub 設定（任意） -->
  <section style="margin-bottom:12px">
    <h3>GitHub 同期（任意）</h3>
    <div class="small">複数端末で共同編集したい場合に使います。Public リポジトリなら読み取りはトークン不要ですが、書き込みには Personal Access Token が必要です。</div>
    <div style="margin-top:8px" class="row">
      <input id="ghOwner" placeholder="Owner (例: naoosero8-dot)" style="width:160px">
      <input id="ghRepo" placeholder="Repo (例: othelonia-poker)" style="width:200px">
      <input id="ghBranch" placeholder="Branch" value="main" style="width:90px">
      <input id="cardsPath" placeholder="cards.json path" value="cards.json" style="width:160px">
      <input id="roomsPath" placeholder="rooms.json path" value="rooms.json" style="width:160px">
    </div>
    <div style="margin-top:8px" class="row">
      <input id="ghToken" placeholder="GitHub Personal Access Token (repo scope)" style="width:560px">
      <button id="btnConnect" class="btn-primary">接続して読み込む</button>
      <button id="btnDisconnect">切断</button>
    </div>
    <div style="margin-top:8px" class="notice small">注意: トークンは sessionStorage にのみ保存します。公開端末での入力は避けてください。</div>
    <div id="ghMsg" class="small" style="margin-top:6px"></div>
  </section>

  <!-- Start / Join -->
  <div id="startSection" style="margin-bottom:12px">
    <div class="row">
      <div class="col" style="flex:1">
        <label class="block">合言葉（4桁・新規作成）<input id="newCode" maxlength="4" placeholder="例: 1234"></label>
        <label class="block">人数上限<select id="maxPlayers"><option>4</option><option>5</option><option>6</option></select></label>
        <label class="block">あなたのニックネーム<input id="creatorName" value="Player"></label>
        <div class="controls"><button id="btnCreate" class="btn-primary">部屋を作る</button>
        <button id="btnJoinView">合言葉で入室</button>
        <button id="btnAdminLogin">管理者ログイン</button></div>
        <div id="startMsg" class="small" style="margin-top:6px"></div>
      </div>
      <div style="width:340px">
        <div class="status"><strong>説明</strong>
          <div class="small">このファイルはサーバ不要で動く単一ファイルです。ローカルでの動作は localStorage に保存されます。GitHub 同期は任意です。</div>
        </div>
      </div>
    </div>
  </div>

  <!-- Join View -->
  <div id="joinView" class="hidden" style="margin-bottom:12px">
    <label class="block">合言葉（4桁）<input id="joinCode" maxlength="4"></label>
    <label class="block">ニックネーム<input id="joinName" value="Player"></label>
    <div class="controls"><button id="btnJoin" class="btn-primary">入室</button> <button id="btnBackStart">戻る</button></div>
    <div id="joinMsg" class="small"></div>
  </div>

  <!-- Lobby / Game -->
  <div id="lobbyView" class="hidden">
    <div class="row">
      <div style="flex:1">
        <div><strong>ルーム</strong> <span id="roomCodeDisplay"></span> <span class="small">部屋主: <span id="roomOwner"></span></span></div>
        <div style="margin-top:8px">
          <label>Act as:
            <select id="actAsSelect"></select>
          </label>
          <label style="margin-left:8px">自分の表示名<input id="myDisplay" style="width:160px"></label>
        </div>
        <div style="margin-top:8px" class="controls">
          <button id="btnReady">準備</button>
          <button id="btnStartGame" class="btn-primary hidden">ゲーム開始（部屋主のみ）</button>
          <button id="btnLeave">退出</button>
          <button id="btnDissolve" class="hidden">部屋解散</button>
        </div>

        <div style="margin-top:10px"><strong>プレイヤー一覧</strong></div>
        <div id="playersList" class="card-grid" style="margin-top:8px"></div>

        <div id="gameArea" class="hidden" style="margin-top:12px">
          <div><strong>状態:</strong> <span id="phaseLabel" class="small">—</span></div>
          <h3>あなたの手札（Act as に合わせて表示）</h3>
          <div id="handArea" class="card-grid"></div>

          <div id="exchangePanel" class="hidden" style="margin-top:12px">
            <h4>交換フェーズ</h4>
            <div>ラウンド: <span id="exchangeRoundDisplay"></span> / <span id="exchangeIndexDisplay"></span></div>
            <div class="controls"><button id="btnExchangeDo">選択したカードを交換</button><button id="btnExchangeOK">交換なし</button></div>
          </div>

          <div id="submitPanel" class="hidden" style="margin-top:12px">
            <h4>提出フェーズ</h4>
            <div id="submitHand" class="card-grid" style="margin-top:8px"></div>
            <div style="margin-top:8px">役名: <input id="roleInput" style="width:40%"><button id="btnSubmit" class="btn-primary">提出</button></div>
          </div>

          <div id="votePanel" class="hidden" style="margin-top:12px">
            <h4>投票フェーズ</h4>
            <div id="submissionsArea"></div>
          </div>

          <div id="resultPanel" class="hidden" style="margin-top:12px">
            <h4>結果</h4>
            <div id="resultDisplay"></div>
            <div style="margin-top:8px"><button id="btnNextGame">次のゲーム（リセット）</button></div>
          </div>
        </div>

      </div>
      <div style="width:340px">
        <div class="status"><strong>カード</strong>
          <div class="small">管理者はカード編集ができます（Local または GitHub に保存）。</div>
        </div>
        <div style="margin-top:12px">
          <button id="btnOpenAdmin">管理画面を開く</button>
        </div>
      </div>
    </div>
  </div>

  <!-- Admin -->
  <div id="adminPanel" class="hidden" style="margin-top:12px">
    <h3>管理画面 — カード編集</h3>
    <div class="small">カードは localStorage (op_cards) に保存されます。画像は DataURL として埋め込みます。大きな画像は注意。</div>
    <div style="margin-top:8px"><label class="block">検索<input id="adminSearch" placeholder="ID or 名前"></label></div>
    <div id="adminCards" class="card-grid" style="margin-top:8px;height:420px;overflow:auto"></div>
    <div style="margin-top:8px" class="controls">
      <button id="btnExport">カードをダウンロード</button>
      <button id="btnImport">カードをインポート</button>
      <input type="file" id="importFile" accept="application/json" class="hidden">
      <button id="btnSaveToGH">（任意）GitHub に保存</button>
    </div>
  </div>

  <div style="margin-top:12px">
    <strong>操作ログ</strong>
    <div id="logArea" class="log"></div>
  </div>

  <footer style="margin-top:12px" class="small">※ このファイルはローカル動作が基本です。GitHub 同期は任意の補助機能です。</footer>
</div>

<script>
/* Single-file app: localStorage core + optional GitHub sync (polling)
   - Local keys: op_cards, op_rooms
   - Optional GitHub: Owner/Repo/Branch/cardsPath and roomsPath + token (stored in sessionStorage)
   - Cross-tab: storage event
*/

// ---------- Config ----------
const CARD_TOTAL = 90;
const GH_TOKEN_KEY = 'GH_TOKEN_SESSION';
const POLL_MS = 4000; // polling interval for GitHub

// ---------- Helpers ----------
function el(id){ return document.getElementById(id); }
function log(s){ const a = el('logArea'); if(!a) return; a.innerHTML = `<div class="small">[${new Date().toLocaleTimeString()}] ${escapeHtml(s)}</div>` + a.innerHTML; }
function escapeHtml(s){ return (s+'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
function uid(p='p'){ return p + Math.random().toString(36).slice(2,9); }
function b64(s){ return btoa(unescape(encodeURIComponent(s))); }
function ub64(s){ return decodeURIComponent(escape(atob(s))); }
function sleep(ms){ return new Promise(res=>setTimeout(res, ms)); }

// ---------- Storage (local) ----------
function defaultCards(){
  const arr=[];
  for(let i=1;i<=CARD_TOTAL;i++){ const id=String(i).padStart(3,'0'); arr.push({id,name:`Card ${id}`,img:null}); }
  return arr;
}
function loadCardsLocal(){ try{ const raw = localStorage.getItem('op_cards'); if(!raw) return defaultCards(); return JSON.parse(raw); }catch(e){ return defaultCards(); } }
function saveCardsLocal(arr){ localStorage.setItem('op_cards', JSON.stringify(arr)); dispatchLocalChange('op_cards'); }
function loadRoomsLocal(){ try{ const raw = localStorage.getItem('op_rooms'); if(!raw) return {}; return JSON.parse(raw); }catch(e){ return {}; } }
function saveRoomsLocal(obj){ localStorage.setItem('op_rooms', JSON.stringify(obj)); dispatchLocalChange('op_rooms'); }
function dispatchLocalChange(key){ // notify other tabs
  try{ localStorage.setItem('op_sync_flag', Date.now().toString()); }catch(e){}
}

// ---------- Global state ----------
let cards = loadCardsLocal();
let rooms = loadRoomsLocal();
let currentRoom = null;
let actingPlayerId = null;
let pollTimer = null;
let ghPolling = false;
let lastRemoteCardsText = null;
let lastRemoteRoomsText = null;

// ---------- GitHub helpers ----------
function ghConfig(){
  return {
    owner: el('ghOwner').value.trim(),
    repo: el('ghRepo').value.trim(),
    branch: el('ghBranch').value.trim() || 'main',
    cardsPath: el('cardsPath').value.trim() || 'cards.json',
    roomsPath: el('roomsPath').value.trim() || 'rooms.json',
    token: sessionStorage.getItem(GH_TOKEN_KEY) || el('ghToken').value.trim()
  };
}
function setTokenToSession(token){
  if(token) sessionStorage.setItem(GH_TOKEN_KEY, token);
  else sessionStorage.removeItem(GH_TOKEN_KEY);
  el('ghToken').value = token || '';
}
function apiHeaders(token){
  const h = {'Accept':'application/vnd.github.v3+json'};
  if(token) h['Authorization'] = 'token ' + token;
  return h;
}
async function rawGet(owner, repo, branch, path){
  const url = `https://raw.githubusercontent.com/${owner}/${repo}/${branch}/${path}`;
  const res = await fetch(url);
  if(res.status===404) return null;
  if(!res.ok) throw new Error(`rawGet failed ${res.status}`);
  return await res.text();
}
async function ghGetFile(owner, repo, path, branch, token){
  const url = `https://api.github.com/repos/${owner}/${repo}/contents/${encodeURIComponent(path)}?ref=${encodeURIComponent(branch)}`;
  const res = await fetch(url,{headers:apiHeaders(token)});
  if(res.status===404) return null;
  if(!res.ok) throw new Error(`ghGetFile failed ${res.status} ${await res.text()}`);
  return await res.json();
}
async function ghPutFile(owner, repo, path, branch, token, contentString, message, sha){
  if(!token) throw new Error('書き込みにはトークンが必要です');
  const url = `https://api.github.com/repos/${owner}/${repo}/contents/${encodeURIComponent(path)}`;
  const body = { message, content: b64(contentString), branch };
  if(sha) body.sha = sha;
  const res = await fetch(url, { method: 'PUT', headers: apiHeaders(token), body: JSON.stringify(body) });
  if(!res.ok) throw new Error(`ghPutFile failed ${res.status} ${await res.text()}`);
  return await res.json();
}

// ---------- Polling ----------
async function pollOnce(){
  const cfg = ghConfig();
  if(!cfg.owner || !cfg.repo) return;
  try{
    // cards
    let remoteCardsText = null;
    try{ remoteCardsText = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.cardsPath); }catch(e){ /* ignore */ }
    if(remoteCardsText !== null){
      if(remoteCardsText !== lastRemoteCardsText){
        lastRemoteCardsText = remoteCardsText;
        try{ const parsed = JSON.parse(remoteCardsText); cards = parsed; saveCardsLocal(parsed); renderAdminCards(); log('リモート cards.json を取得し反映しました'); }catch(e){ log('cards.json parse error'); }
      }
    } else if(cfg.token){
      // try API
      const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token);
      if(meta){
        const txt = ub64(meta.content);
        if(txt !== lastRemoteCardsText){
          lastRemoteCardsText = txt; cards = JSON.parse(txt); saveCardsLocal(cards); renderAdminCards(); log('リモート (API) cards を反映'); 
        }
      }
    }
    // rooms
    let remoteRoomsText = null;
    try{ remoteRoomsText = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.roomsPath); }catch(e){}
    if(remoteRoomsText !== null){
      if(remoteRoomsText !== lastRemoteRoomsText){
        lastRemoteRoomsText = remoteRoomsText;
        try{ const parsed = JSON.parse(remoteRoomsText); rooms = parsed; saveRoomsLocal(parsed); log('リモート rooms.json を取得し反映しました'); refreshLobby(); refreshGameUI(); }catch(e){ log('rooms.json parse error'); }
      }
    } else if(cfg.token){
      const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token);
      if(meta){
        const txt = ub64(meta.content);
        if(txt !== lastRemoteRoomsText){
          lastRemoteRoomsText = txt; rooms = JSON.parse(txt); saveRoomsLocal(rooms); log('リモート (API) rooms を反映'); refreshLobby(); refreshGameUI();
        }
      }
    }
  }catch(err){ console.warn('poll error', err); }
}
function startPolling(){ if(pollTimer) clearInterval(pollTimer); pollTimer = setInterval(pollOnce, POLL_MS); ghPolling=true; log('リモートポーリング開始'); }
function stopPolling(){ if(pollTimer) clearInterval(pollTimer); pollTimer=null; ghPolling=false; log('リモートポーリング停止'); }

// ---------- Load/Save Remote Helpers ----------
async function loadFromGitHub(){
  const cfg = ghConfig();
  if(!cfg.owner || !cfg.repo) { el('ghMsg').innerText='Owner/Repo を入力してください'; return; }
  try{
    el('ghMsg').innerText = 'リモート読み込み中...';
    // cards
    let rawCards = null;
    try{ rawCards = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.cardsPath); }catch(e){}
    if(rawCards !== null){ cards = JSON.parse(rawCards); saveCardsLocal(cards); lastRemoteCardsText = rawCards; log('cards.json を取得しました (raw)'); }
    else if(cfg.token){
      const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token);
      if(meta){ const txt = ub64(meta.content); cards = JSON.parse(txt); saveCardsLocal(cards); lastRemoteCardsText = txt; log('cards.json を API で取得しました'); }
    }
    // rooms
    let rawRooms = null;
    try{ rawRooms = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.roomsPath); }catch(e){}
    if(rawRooms !== null){ rooms = JSON.parse(rawRooms); saveRoomsLocal(rooms); lastRemoteRoomsText = rawRooms; log('rooms.json を取得しました (raw)'); }
    else if(cfg.token){
      const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token);
      if(meta){ const txt = ub64(meta.content); rooms = JSON.parse(txt); saveRoomsLocal(rooms); lastRemoteRoomsText = txt; log('rooms.json を API で取得しました'); }
    }
    el('ghMsg').innerText = '読み込み完了';
    refreshLobby();
    renderAdminCards();
  }catch(e){ el('ghMsg').innerText = '読み込み失敗: '+e.message; log('GH load error: '+e.message); }
}

async function saveCardsToGitHub(){
  const cfg = ghConfig();
  if(!cfg.token){ alert('保存にはトークンが必要です'); return; }
  try{
    el('ghMsg').innerText = '保存中...';
    const content = JSON.stringify(cards, null, 2);
    const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token);
    const sha = meta ? meta.sha : undefined;
    await ghPutFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token, content, 'Update cards.json', sha);
    el('ghMsg').innerText = 'cards.json を保存しました';
    lastRemoteCardsText = content;
    log('cards.json を GitHub に保存しました');
  }catch(e){ el('ghMsg').innerText = '保存失敗: '+e.message; log('GH save cards error: '+e.message); }
}

async function saveRoomsToGitHub(){
  const cfg = ghConfig();
  if(!cfg.token){ alert('保存にはトークンが必要です'); return; }
  try{
    el('ghMsg').innerText = '保存中...';
    const content = JSON.stringify(rooms, null, 2);
    const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token);
    const sha = meta ? meta.sha : undefined;
    await ghPutFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token, content, 'Update rooms.json', sha);
    el('ghMsg').innerText = 'rooms.json を保存しました';
    lastRemoteRoomsText = content;
    log('rooms.json を GitHub に保存しました');
  }catch(e){ el('ghMsg').innerText = '保存失敗: '+e.message; log('GH save rooms error: '+e.message); }
}

// ---------- UI & App Logic (local-first) ----------
function renderAdminCards(){
  const wrap = el('adminCards'); wrap.innerHTML = '';
  const q = (el('adminSearch')||{value:''}).value.toLowerCase();
  for(const c of cards){
    if(q && !(c.id.includes(q) || c.name.toLowerCase().includes(q))) continue;
    const box = document.createElement('div'); box.className='card';
    const img = document.createElement('img'); img.src = c.img || placeholderForId(c.id); img.style.height='90px';
    const name = document.createElement('input'); name.value = c.name; name.onchange = ()=>{ c.name = name.value; saveCardsLocal(cards); };
    const file = document.createElement('input'); file.type='file'; file.accept='image/*';
    file.onchange = (ev)=>{ const f=ev.target.files[0]; if(!f) return; const r=new FileReader(); r.onload = e=>{ c.img = e.target.result; saveCardsLocal(cards); renderAdminCards(); }; r.readAsDataURL(f); };
    box.appendChild(img); box.appendChild(name); box.appendChild(file);
    wrap.appendChild(box);
  }
}

function placeholderForId(id){
  const svg = `<svg xmlns='http://www.w3.org/2000/svg' width='200' height='300'><rect width='100%' height='100%' fill='#eee'/><text x='50%' y='50%' dominant-baseline='middle' text-anchor='middle' fill='#666' font-size='18'>${id}</text></svg>`;
  return 'data:image/svg+xml;base64,' + btoa(svg);
}

// Lobby rendering
function refreshLobby(){
  const room = rooms[currentRoom]; if(!room) return;
  el('roomCodeDisplay').innerText = room.code;
  el('roomOwner').innerText = (room.owner ? (room.players.find(p=>p.id===room.owner)||{}).nick : '');
  const wrap = el('playersList'); wrap.innerHTML = '';
  room.players.forEach(p=>{
    const d = document.createElement('div'); d.className='card';
    d.innerHTML = `<div style="font-weight:700">${escapeHtml(p.nick)}</div><div class="small">ready: ${p.ready? '✅':'—'}</div>`;
    if(room.owner===actingPlayerId && p.id!==room.owner){
      const b = document.createElement('button'); b.textContent='Kick'; b.onclick = ()=>{ kickPlayer(p.id); };
      d.appendChild(b);
    }
    wrap.appendChild(d);
  });
  const sel = el('actAsSelect'); sel.innerHTML = '';
  room.players.forEach(p=>{ const o = document.createElement('option'); o.value = p.id; o.text = p.nick; sel.appendChild(o); });
  sel.value = actingPlayerId || '';
  sel.onchange = ()=>{ actingPlayerId = sel.value; renderHandArea(); };
  el('btnStartGame').classList.toggle('hidden', !(room.players.length >= 4 && room.players.every(p=>p.ready) && actingPlayerId===room.owner));
  el('btnDissolve').classList.toggle('hidden', actingPlayerId===room.owner?false:true);
  el('gameArea').classList.toggle('hidden', room.phase === 'waiting' || !room.phase);
}

// Hand rendering
function renderHandArea(){
  const wrap = el('handArea'); if(!wrap) return; wrap.innerHTML = '';
  const room = rooms[currentRoom]; if(!room) return;
  const player = room.players.find(p=>p.id===actingPlayerId);
  if(!player) return;
  player.hand.forEach((c,idx)=>{
    const meta = cards.find(x=>x.id===c.id);
    const div = document.createElement('div'); div.className='card'; div.dataset.index = idx;
    const img = document.createElement('img'); img.src = c.img || (meta && meta.img) || placeholderForId(c.id);
    const nm = document.createElement('div'); nm.textContent = c.name || (meta && meta.name) || c.id;
    div.appendChild(img); div.appendChild(nm);
    div.onclick = ()=> div.classList.toggle('selected');
    wrap.appendChild(div);
  });
  renderSubmitHand();
}

function renderSubmitHand(){
  const container = el('submitHand'); if(!container) return; container.innerHTML = '';
  const room = rooms[currentRoom]; if(!room) return;
  const player = room.players.find(p=>p.id===actingPlayerId); if(!player) return;
  player.hand.forEach(c=>{
    const meta = cards.find(x=>x.id===c.id);
    const div = document.createElement('div'); div.className='card'; div.dataset.id = c.id;
    const img = document.createElement('img'); img.src = c.img || (meta && meta.img) || placeholderForId(c.id);
    const nm = document.createElement('div'); nm.textContent = c.name || (meta && meta.name) || c.id;
    div.appendChild(img); div.appendChild(nm);
    container.appendChild(div);
  });
  makeSortable(container);
}

function renderSubmissions(){
  const container = el('submissionsArea'); if(!container) return; container.innerHTML = '';
  const room = rooms[currentRoom]; if(!room) return;
  (room.turnOrder||[]).forEach(pid=>{
    const sub = (room.submitted && room.submitted[pid]) || {order: (room.players.find(p=>p.id===pid)||{}).hand.map(c=>c.id), roleName:''};
    const box = document.createElement('div'); box.className='card';
    box.innerHTML = `<div style="font-weight:700">${escapeHtml((room.players.find(p=>p.id===pid)||{}).nick)}</div><div class="small">${escapeHtml(sub.roleName)}</div>`;
    const row = document.createElement('div'); row.style.display='flex'; row.style.gap='6px'; row.style.marginTop='6px';
    (sub.order||[]).forEach(cid=>{
      const cc = cards.find(x=>x.id===cid) || {id:cid,name:cid,img:null};
      const t = document.createElement('img'); t.src = cc.img || placeholderForId(cid); t.style.width='46px'; t.style.height='68px'; t.style.objectFit='cover';
      row.appendChild(t);
    });
    box.appendChild(row);
    if(actingPlayerId && actingPlayerId !== pid && !(room.votes && room.votes[actingPlayerId])){
      const voteBtn = document.createElement('button'); voteBtn.textContent='投票'; voteBtn.onclick = async ()=> { await castVote(pid); renderSubmissions(); };
      box.appendChild(voteBtn);
    } else {
      const note = document.createElement('div'); note.className='small'; note.innerText = actingPlayerId===pid ? 'あなたの提出' : (room.votes && room.votes[actingPlayerId] ? '投票済' : '');
      box.appendChild(note);
    }
    container.appendChild(box);
  });
}

// sortable
function makeSortable(container){
  let drag = null;
  Array.from(container.children).forEach(ch=>{
    ch.draggable = true;
    ch.ondragstart = e=>{ drag = ch; e.dataTransfer.setData('text/plain',''); ch.style.opacity = .4; };
    ch.ondragend = ()=>{ if(drag) drag.style.opacity=1; drag=null; };
    ch.ondragover = e=> e.preventDefault();
    ch.ondrop = e=>{ e.preventDefault(); if(!drag||drag===ch) return; container.insertBefore(drag, ch.nextSibling); };
  });
}

// ---------- Room operations (local-first) ----------
function createRoom(){
  const code = (el('newCode').value||'').trim();
  const max = parseInt(el('maxPlayers').value||4);
  const creator = (el('creatorName').value||'Owner').trim();
  if(!/^\d{4}$/.test(code)){ alert('合言葉は4桁の数字にしてください'); return; }
  rooms = loadRoomsLocal();
  if(rooms[code]){ alert('その合言葉は既に使われています'); return; }
  rooms[code] = { code, owner:null, maxPlayers: Math.min(6, Math.max(4,max)), players:[], state:'waiting', deck:[], discard:[], turnOrder:[], exchangeRound:1, exchangeIndex:0, submitted:{}, votes:{}, voteCounts:{}, phase:'waiting' };
  saveRoomsLocal(rooms);
  joinRoomWithName(code, creator, true);
  el('startMsg').innerText = `部屋 ${code} を作成しました。`;
  log('部屋作成: '+code);
}

function joinRoom(){
  rooms = loadRoomsLocal();
  const code = (el('joinCode').value||'').trim();
  const name = (el('joinName').value||('P'+Math.floor(Math.random()*1000))).trim();
  if(!/^\d{4}$/.test(code)){ el('joinMsg').innerText='合言葉は4桁'; return; }
  if(!rooms[code]){ el('joinMsg').innerText='その合言葉の部屋はありません'; return; }
  joinRoomWithName(code, name, false);
}

function joinRoomWithName(code,name,isCreator){
  rooms = loadRoomsLocal();
  const room = rooms[code];
  if(!room){ alert('その合言葉の部屋はありません'); return; }
  if(room.players.length >= room.maxPlayers){ alert('満員です'); return; }
  const pid = uid('p');
  const player = {id:pid, nick:name, ready:false, hand:[], score:0, connected:true};
  room.players.push(player);
  if(isCreator) room.owner = pid;
  saveRoomsLocal(rooms);
  currentRoom = code;
  actingPlayerId = pid;
  el('myDisplay').value = player.nick;
  refreshLobby();
  showView('lobbyView');
  log(`${player.nick} が部屋 ${code} に参加`);
}

function leaveRoom(){
  const room = rooms[currentRoom]; if(!room) return;
  const idx = room.players.findIndex(p=>p.id===actingPlayerId);
  if(idx!==-1){ const name=room.players[idx].nick; room.players.splice(idx,1); log(`${name} が退出`); }
  if(room.owner===actingPlayerId){ if(room.players.length>0) room.owner = room.players[0].id; else delete rooms[room.code]; }
  saveRoomsLocal(rooms);
  currentRoom=null; actingPlayerId=null; showView('startView');
}

function dissolveRoom(){
  const room = rooms[currentRoom]; if(!room) return;
  if(room.owner !== actingPlayerId){ alert('権限なし'); return; }
  if(!confirm('本当に解散しますか？')) return;
  delete rooms[room.code];
  saveRoomsLocal(rooms);
  currentRoom=null; actingPlayerId=null; showView('startView'); log('部屋を解散しました');
}

function kickPlayer(targetId){
  const room = rooms[currentRoom]; if(!room) return;
  if(room.owner !== actingPlayerId){ alert('権限なし'); return; }
  const idx = room.players.findIndex(p=>p.id===targetId); if(idx===-1) return;
  const name = room.players[idx].nick; room.players.splice(idx,1); saveRoomsLocal(rooms); refreshLobby(); log(name+' をキックしました');
}

function toggleReady(){
  const room = rooms[currentRoom]; if(!room) return;
  const player = room.players.find(p=>p.id===actingPlayerId); if(!player) return;
  player.ready = !player.ready; saveRoomsLocal(rooms); refreshLobby(); log(player.nick+' の準備: '+player.ready);
}

// ---------- Game operations (owner-driven simplified) ----------
function startGameAsOwner(){
  const room = rooms[currentRoom]; if(!room) return;
  if(room.owner !== actingPlayerId){ alert('部屋主のみ開始可能'); return; }
  if(room.players.length < 2){ alert('プレイヤー不足'); return; }
  // deck from cards
  const deck = loadCardsLocal().map(c=>({id:c.id,name:c.name,img:c.img}));
  for(let i=deck.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [deck[i],deck[j]]=[deck[j],deck[i]]; }
  const order = room.players.map(p=>p.id); for(let i=order.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [order[i],order[j]]=[order[j],order[i]]; }
  room.deck = deck; room.discard=[]; room.state='playing'; room.turnOrder=order; room.exchangeRound=1; room.exchangeIndex=0;
  room.players.forEach(p=>{ p.hand = room.deck.splice(0,5); p.score=0; });
  room.submitted = {}; room.votes={}; room.voteCounts={}; room.phase='exchange';
  saveRoomsLocal(rooms); refreshGameUI(); log('ゲーム開始');
}

async function doExchange(indices){
  const room = rooms[currentRoom]; if(!room) return;
  const pid = room.turnOrder[room.exchangeIndex];
  if(pid !== actingPlayerId){ alert('今はこのプレイヤーの番ではありません'); return; }
  const player = room.players.find(p=>p.id===pid);
  const idxs = indices.slice().sort((a,b)=>b-a);
  const removed = [];
  for(const idx of idxs){ if(idx>=0 && idx<player.hand.length) removed.push(player.hand.splice(idx,1)[0]); }
  room.deck.push(...removed);
  const newcards = room.deck.splice(0, removed.length);
  player.hand.push(...newcards);
  room.exchangeIndex++;
  saveRoomsLocal(rooms);
  renderHandArea(); log(player.nick+' が '+removed.length+' 枚交換しました');
}

function exchangeOK(){
  const room = rooms[currentRoom]; if(!room) return;
  if(room.turnOrder[room.exchangeIndex] !== actingPlayerId){ alert('今はこのプレイヤーの番ではありません'); return; }
  room.exchangeIndex++;
  saveRoomsLocal(rooms); log('交換なしでOK');
}

function startSubmitPhase(){
  const room = rooms[currentRoom]; if(!room) return;
  room.phase='submit';
  room.submitted = room.submitted || {};
  room.turnOrder.forEach(pid => { if(!room.submitted[pid]){ const p = room.players.find(x=>x.id===pid); room.submitted[pid] = { order: p.hand.map(c=>c.id), roleName:'', submitted:false }; } });
  saveRoomsLocal(rooms); refreshGameUI(); log('提出フェーズ開始');
}

function submitWork(){
  const room = rooms[currentRoom]; if(!room) return;
  const nodes = Array.from(el('submitHand').children);
  if(nodes.length !== 5){ alert('5枚を並べてください'); return; }
  const order = nodes.map(n=>n.dataset.id);
  const roleName = (el('roleInput').value||'').trim();
  room.submitted[actingPlayerId] = { order, roleName, submitted:true };
  saveRoomsLocal(rooms); log('提出: '+(roleName||'(無題)'));
}

function startVotePhase(){
  const room = rooms[currentRoom]; if(!room) return;
  room.phase='vote'; room.votes={}; room.voteCounts={}; saveRoomsLocal(rooms); refreshGameUI(); log('投票フェーズ開始');
}

async function castVote(targetId){
  const room = rooms[currentRoom]; if(!room) return;
  room.votes = room.votes || {}; room.voteCounts = room.voteCounts || {};
  if(room.votes[actingPlayerId]){ alert('投票済み'); return; }
  room.votes[actingPlayerId] = targetId; room.voteCounts[targetId] = (room.voteCounts[targetId]||0)+1;
  saveRoomsLocal(rooms); renderSubmissions(); log('投票しました');
}

function finishVoteAndShow(){
  const room = rooms[currentRoom]; if(!room) return;
  room.voteCounts = room.voteCounts || {};
  let max=-1; (room.turnOrder||[]).forEach(pid=>{ room.voteCounts[pid] = room.voteCounts[pid]||0; if(room.voteCounts[pid]>max) max=room.voteCounts[pid]; });
  const winners = (room.turnOrder||[]).filter(pid=>room.voteCounts[pid]===max);
  room.phase='result'; room.result = { winners, counts: room.voteCounts };
  saveRoomsLocal(rooms); refreshGameUI(); log('投票終了');
}

function resetAfterGame(){
  const room = rooms[currentRoom]; if(!room) return;
  room.players.forEach(p=>{ p.hand=[]; p.ready=false; p.score=0; });
  room.deck=[]; room.discard=[]; room.turnOrder=[]; room.exchangeRound=1; room.exchangeIndex=0; room.submitted={}; room.votes={}; room.voteCounts={}; room.phase='waiting';
  saveRoomsLocal(rooms); showView('lobbyView'); refreshLobby(); log('ゲームリセット');
}

// ---------- GitHub connect UI ----------
el('btnConnect').onclick = async ()=> {
  const token = el('ghToken').value.trim();
  setTokenToSession(token);
  await loadFromGitHub();
  startPolling();
};
el('btnDisconnect').onclick = ()=> { setTokenToSession(''); stopPolling(); el('ghMsg').innerText='切断しました'; log('GitHub 切断'); };

// ---------- Other UI wiring ----------
el('btnCreate').onclick = createRoom;
el('btnJoinView').onclick = ()=> showView('joinView');
el('btnBackStart').onclick = ()=> showView('startView');
el('btnJoin').onclick = joinRoom;
el('btnReady').onclick = toggleReady;
el('btnLeave').onclick = leaveRoom;
el('btnDissolve').onclick = dissolveRoom;
el('btnOpenAdmin').onclick = ()=> { showView('adminPanel'); renderAdminCards(); };
el('btnAdminLogin').onclick = ()=> { const p = prompt('管理者パスワード'); if(p==='nanamiya333'){ showView('adminPanel'); renderAdminCards(); } else alert('違います'); };
el('btnExport').onclick = ()=> { const data = JSON.stringify(cards, null, 2); const blob = new Blob([data], {type:'application/json'}); const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download='cards.json'; a.click(); };
el('btnImport').onclick = ()=> el('importFile').click();
el('importFile').onchange = e=> { const f = e.target.files[0]; if(!f) return; const r = new FileReader(); r.onload = ev=> { try{ const parsed = JSON.parse(ev.target.result); if(Array.isArray(parsed)){ cards = parsed; saveCardsLocal(cards); renderAdminCards(); alert('インポート完了'); } else alert('不正なファイル'); }catch(err){ alert('読み込み失敗'); } }; r.readAsText(f); };
el('btnSaveToGH').onclick = saveCardsToGitHub;
el('btnStartGame').onclick = startGameAsOwner;
el('btnExchangeDo').onclick = ()=> {
  const sel = Array.from(el('handArea').querySelectorAll('.selected')).map(elm=>parseInt(elm.dataset.index));
  if(sel.length===0){ alert('選択してください'); return; }
  doExchange(sel);
};
el('btnExchangeOK').onclick = exchangeOK;
el('btnSubmit').onclick = submitWork;
el('btnNextGame').onclick = resetAfterGame;
el('btnOpenAdmin').onclick = ()=> { showView('adminPanel'); renderAdminCards(); };

// Cast vote button rendered in renderSubmissions

// ---------- Storage event for cross-tab sync ----------
window.addEventListener('storage', (e)=>{
  if(e.key === 'op_cards'){ cards = loadCardsLocal(); renderAdminCards(); renderHandArea(); log('別タブでカードが更新されました'); }
  if(e.key === 'op_rooms'){ rooms = loadRoomsLocal(); refreshLobby(); refreshGameUI(); log('別タブでルームが更新されました'); }
});

// ---------- Utility showView / init ----------
function showView(id){
  ['startSection','joinView','lobbyView','adminPanel'].forEach(v=>{ const elv=document.getElementById(v); if(elv) elv.classList.add('hidden'); });
  const t = document.getElementById(id); if(t) t.classList.remove('hidden');
}

(function init(){
  // ensure defaults exist
  if(!localStorage.getItem('op_cards')) saveCardsLocal(cards);
  if(!localStorage.getItem('op_rooms')) saveRoomsLocal(rooms);
  showView('startSection');
  log('アプリ初期化（localStorage ベース）');
})();

</script>
</body>
</html>

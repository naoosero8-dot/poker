<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>オセロニアポーカー — GitHub リモート同期版</title>
<style>
  :root{--bg:#f6f6f8;--card:#fff;--accent:#2b6df6}
  body{font-family:system-ui,-apple-system,"Hiragino Kaku Gothic ProN","メイリオ",sans-serif;margin:0;background:var(--bg);color:#111}
  header{background:#111;color:#fff;padding:10px 16px}
  .wrap{max-width:1100px;margin:18px auto;padding:12px;background:#fff;border-radius:8px;box-shadow:0 4px 18px rgba(0,0,0,.06)}
  h1{margin:0 0 12px 0;font-size:1.2rem}
  .row{display:flex;gap:12px;align-items:center}
  .col{display:flex;flex-direction:column;gap:8px}
  button{padding:8px 10px;border-radius:6px;border:1px solid #ddd;background:#fafafa;cursor:pointer}
  input,select,textarea{padding:8px;border-radius:6px;border:1px solid #ccc}
  .hidden{display:none}
  .card-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(90px,1fr));gap:8px}
  .card{background:var(--card);border:1px solid #ddd;border-radius:6px;padding:6px;text-align:center;font-size:12px}
  .card img{width:100%;height:96px;object-fit:cover;border-radius:4px}
  .small{font-size:12px;color:#666}
  .controls{display:flex;gap:8px;flex-wrap:wrap}
  .timer{font-weight:700;color:#c33}
  .status{padding:8px;background:#f2f8ff;border-radius:6px;border:1px solid #dfefff}
  .selected{outline:3px solid rgba(43,109,246,.18)}
  .hit{background:#eaf5ff}
  footer{font-size:12px;color:#666;margin-top:12px}
  .btn-primary{background:var(--accent);color:#fff;border:none}
  label.block{display:block;margin-bottom:6px}
  .config{display:flex;gap:8px;align-items:center}
  .notice{padding:8px;background:#fff7e6;border-radius:6px;border:1px solid #ffe6a8;color:#664400}
  pre.small{font-size:11px;color:#666;white-space:pre-wrap}
</style>
</head>
<body>
<header><h1 style="display:inline">オセロニアポーカー — GitHub 同期版</h1></header>
<div class="wrap">

  <section id="configPanel" style="margin-bottom:12px">
    <h3>GitHub 設定（必須）</h3>
    <div class="small">このページは GitHub のリポジトリ内の JSON ファイル(cards.json, rooms.json) を読み書きして、複数端末でリアルタイム（ポーリング）同期を行います。書き込みには Personal Access Token (repo 権限) が必要です。トークンはセッションにのみ保存され、サーバへ送信されません。</div>
    <div style="margin-top:8px" class="config">
      <label class="block">Owner: <input id="ghOwner" placeholder="owner (例: naoosero8-dot)"></label>
      <label class="block">Repo: <input id="ghRepo" placeholder="repo (例: my-repo)"></label>
      <label class="block">Branch: <input id="ghBranch" value="main" style="width:80px"></label>
      <label class="block">cards.json path: <input id="cardsPath" value="cards.json" style="width:160px"></label>
      <label class="block">rooms.json path: <input id="roomsPath" value="rooms.json" style="width:160px"></label>
    </div>
    <div style="margin-top:8px" class="config">
      <input id="ghToken" placeholder="GitHub Personal Access Token (repo scope)" style="width:420px">
      <button id="btnConnect" class="btn-primary">接続して読み込む</button>
      <button id="btnClearToken">トークン削除</button>
    </div>
    <div id="connectMsg" class="small" style="margin-top:8px"></div>
    <div class="notice" style="margin-top:8px">
      注意: トークンはブラウザの sessionStorage にのみ保存します。公開リポジトリなら読み取りは可能ですが、書き込みにはトークンが必要です。複数端末で共同編集する場合は各端末でトークンを入力してください。
    </div>
  </section>

  <!-- Start / Join -->
  <div id="startView">
    <div class="row">
      <div class="col" style="flex:1">
        <label class="block">合言葉（4桁・新規作成）<input id="newCode" maxlength="4" placeholder="例: 1234"></label>
        <label class="block">人数上限<select id="maxPlayers"><option>4</option><option>5</option><option>6</option></select></label>
        <label class="block">あなたのニックネーム<input id="creatorName" value="Player"></label>
        <div class="controls"><button id="btnCreate" class="btn-primary">部屋を作る</button>
        <button id="btnJoinExisting">合言葉で入室</button>
        <button id="btnAdminLogin">管理者ログイン</button></div>
        <div id="startMsg" class="small"></div>
      </div>
      <div style="width:320px">
        <div class="status"><strong>使い方</strong>
          <div class="small">まず GitHub 設定を入力して「接続して読み込む」を押してください。読み込み後に部屋を作成 / 入室できます。管理者はカード編集が可能です。</div>
        </div>
      </div>
    </div>
  </div>

  <!-- Join -->
  <div id="joinView" class="hidden">
    <div class="row">
      <div class="col" style="flex:1">
        <label class="block">合言葉（4桁）<input id="joinCode" maxlength="4"></label>
        <label class="block">ニックネーム<input id="joinName" value="Player"></label>
        <div class="controls">
          <button id="btnJoin" class="btn-primary">入室</button>
          <button id="btnBackStart">戻る</button>
        </div>
        <div id="joinMsg" class="small"></div>
      </div>
    </div>
  </div>

  <!-- Lobby / Game -->
  <div id="lobbyView" class="hidden">
    <div class="row">
      <div style="flex:1">
        <div><strong>ルーム</strong> <span id="roomCodeDisplay"></span></div>
        <div class="small">部屋主: <span id="roomOwner"></span></div>
        <div style="margin-top:8px">
          <label>Act as:
            <select id="actAsSelect"></select>
          </label>
          <label style="margin-left:8px">自分の表示名<input id="myDisplay" style="width:160px"></label>
        </div>
        <div style="margin-top:8px">
          <button id="btnReady">準備OK</button>
          <button id="btnStartGame" class="hidden btn-primary">ゲーム開始（部屋主のみ）</button>
          <button id="btnLeave">退出</button>
          <button id="btnDissolve" class="hidden">部屋解散（部屋主）</button>
        </div>
        <div style="margin-top:8px"><strong>プレイヤー一覧</strong></div>
        <div id="playersList" class="card-grid" style="margin-top:8px"></div>
      </div>
      <div style="width:340px">
        <div class="status">
          <div><strong>ルーム操作</strong></div>
          <div class="small">部屋データは GitHub 上の rooms.json に保存されます。複数端末で同時編集できます（ポーリングで同期）。</div>
        </div>
        <div style="margin-top:12px">
          <strong>管理者</strong>
          <div class="small">管理者ログインでカード編集が可能（cards.json を書き込みます）。</div>
          <button id="btnOpenAdmin">管理画面を開く</button>
        </div>
      </div>
    </div>
  </div>

  <!-- Admin panel -->
  <div id="adminPanel" class="hidden" style="margin-top:12px">
    <h3>管理画面 — カード編集</h3>
    <div class="small">カードデータは GitHub の cards.json に保存されます。画像は DataURL として埋め込み可（サイズに注意）。</div>
    <div style="margin-top:8px">
      <label>カード検索（ID or 名前）<input id="adminSearch" placeholder="例: 001"></label>
    </div>
    <div id="adminCards" class="card-grid" style="margin-top:8px;height:420px;overflow:auto"></div>
    <div style="margin-top:8px">
      <button id="btnExport">カードデータをダウンロード（JSON）</button>
      <button id="btnImport">カードデータをインポート（JSON）</button>
      <input type="file" id="importFile" accept="application/json" class="hidden">
    </div>
  </div>

  <footer style="margin-top:12px" class="small">※このクライアントは GitHub をデータストアとして使っています。書き込みには各自の Personal Access Token（repo スコープ）が必要です。トークンは sessionStorage にのみ保存されます。</footer>
</div>

<script>
/* -----------------------------------------
   GitHub-backed single-file app
   - Stores cards.json and rooms.json in specified repository.
   - Uses GitHub REST API to PUT/GET files (requires token for writes).
   - Polls repository for changes every POLL_MS to achieve near-real-time sync across devices.
   ----------------------------------------- */

const POLL_MS = 4000;

// --- utility
function el(id){ return document.getElementById(id); }
function uid(prefix='p'){ return prefix + Math.random().toString(36).slice(2,9); }
function b64(s){ return btoa(unescape(encodeURIComponent(s))); }
function ub64(s){ return decodeURIComponent(escape(atob(s))); }
function escapeHtml(s){ return (s+'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
function log(msg){ const a=el('logArea'); if(a) a.innerHTML = `<div class="small">[${(new Date()).toLocaleTimeString()}] ${escapeHtml(msg)}</div>` + a.innerHTML; }

// --- GitHub configuration (from UI)
function ghConfig(){
  return {
    owner: el('ghOwner').value.trim(),
    repo: el('ghRepo').value.trim(),
    branch: el('ghBranch').value.trim() || 'main',
    cardsPath: el('cardsPath').value.trim() || 'cards.json',
    roomsPath: el('roomsPath').value.trim() || 'rooms.json',
    token: sessionStorage.getItem('GH_TOKEN') || (el('ghToken') && el('ghToken').value.trim()) || ''
  };
}

function setTokenInSession(token){
  if(!token) sessionStorage.removeItem('GH_TOKEN');
  else sessionStorage.setItem('GH_TOKEN', token);
  el('ghToken').value = token || '';
}

// --- GitHub API helpers
function apiHeaders(token){
  const h = {'Accept':'application/vnd.github.v3+json'};
  if(token) h['Authorization'] = 'token ' + token;
  return h;
}

// Get file metadata (sha) and base64 content via REST API
async function ghGetFile(owner, repo, path, branch, token){
  const url = `https://api.github.com/repos/${owner}/${repo}/contents/${encodeURIComponent(path)}?ref=${encodeURIComponent(branch)}`;
  const res = await fetch(url, {headers: apiHeaders(token)});
  if(res.status === 404) return null;
  if(!res.ok) throw new Error(`GitHub GET failed: ${res.status} ${await res.text()}`);
  return await res.json(); // content (base64), sha, etc.
}

// Put (create/update) file
async function ghPutFile(owner, repo, path, branch, token, contentString, message, sha){
  if(!token) throw new Error('書き込みには GitHub トークンが必要です');
  const url = `https://api.github.com/repos/${owner}/${repo}/contents/${encodeURIComponent(path)}`;
  const body = {
    message,
    content: b64(contentString),
    branch
  };
  if(sha) body.sha = sha;
  const res = await fetch(url, {method:'PUT', headers: apiHeaders(token), body: JSON.stringify(body)});
  if(!res.ok) throw new Error(`GitHub PUT failed: ${res.status} ${await res.text()}`);
  return await res.json();
}

// Raw file fetch (public read) using raw.githubusercontent (fast)
async function rawGet(owner, repo, branch, path){
  const url = `https://raw.githubusercontent.com/${owner}/${repo}/${branch}/${path}`;
  const res = await fetch(url);
  if(res.status === 404) return null;
  if(!res.ok) throw new Error(`Raw GET failed: ${res.status}`);
  return await res.text();
}

/* -------------------------
   Application state
   ------------------------- */
let cards = []; // array {id,name,img}
let rooms = {}; // map code -> room object
let cardsFileSha = null;
let roomsFileSha = null;
let pollTimer = null;

/* -------------------------
   Load / Save from GitHub
   ------------------------- */
async function loadCardsFromRepo(){
  const cfg = ghConfig();
  // try raw first (public)
  try {
    const raw = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.cardsPath);
    if(raw !== null){
      cards = JSON.parse(raw);
      // also fetch sha if token available
      if(cfg.token){
        const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token);
        cardsFileSha = meta ? meta.sha : null;
      } else cardsFileSha = null;
      return cards;
    }
  } catch(e){
    console.warn('rawGet cards failed', e);
  }
  // fallback to API
  try {
    const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token);
    if(meta){
      const txt = ub64(meta.content);
      cards = JSON.parse(txt);
      cardsFileSha = meta.sha;
      return cards;
    }
  } catch(e){
    console.warn('ghGetFile cards failed', e);
  }
  // if not exist -> init default
  cards = [];
  for(let i=1;i<=90;i++){ const id=String(i).padStart(3,'0'); cards.push({id, name:`Card ${id}`, img:null}); }
  cardsFileSha = null;
  return cards;
}

async function saveCardsToRepo(message){
  const cfg = ghConfig();
  if(!cfg.token) throw new Error('保存するにはトークンが必要です');
  const content = JSON.stringify(cards, null, 2);
  const resp = await ghPutFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token, content, message || 'Update cards.json', cardsFileSha);
  cardsFileSha = resp.content.sha;
  return resp;
}

async function loadRoomsFromRepo(){
  const cfg = ghConfig();
  try {
    const raw = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.roomsPath);
    if(raw !== null){
      rooms = JSON.parse(raw);
      if(cfg.token){
        const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token);
        roomsFileSha = meta ? meta.sha : null;
      } else roomsFileSha = null;
      return rooms;
    }
  } catch(e){ console.warn('rawGet rooms failed', e); }
  try {
    const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token);
    if(meta){
      rooms = JSON.parse(ub64(meta.content));
      roomsFileSha = meta.sha;
      return rooms;
    }
  } catch(e){ console.warn('ghGetFile rooms failed', e); }
  // not exist -> init empty
  rooms = {};
  roomsFileSha = null;
  return rooms;
}

async function saveRoomsToRepo(message){
  const cfg = ghConfig();
  if(!cfg.token) throw new Error('保存するにはトークンが必要です');
  const content = JSON.stringify(rooms, null, 2);
  const resp = await ghPutFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token, content, message || 'Update rooms.json', roomsFileSha);
  roomsFileSha = resp.content.sha;
  return resp;
}

/* -------------------------
   Polling for updates across devices
   ------------------------- */
async function pollOnce(){
  try{
    const cfg = ghConfig();
    // load remote versions (cards & rooms) and compare textual content
    // We use raw fetch to minimize rate-limit for public repos
    if(cfg.owner && cfg.repo){
      // cards
      let remoteCardsText = null;
      try { remoteCardsText = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.cardsPath); } catch(e){}
      if(remoteCardsText !== null){
        const parsed = JSON.parse(remoteCardsText);
        // simple deep-compare by JSON string
        const localText = JSON.stringify(cards);
        if(JSON.stringify(parsed) !== localText){
          cards = parsed;
          el('connectMsg').innerText = 'cards.json が更新されました（リモート）';
          renderAdminCards(); renderSubmitHand(); renderHandArea(); renderSubmissions();
        }
      } else {
        // maybe file removed or private; try API if token
        if(cfg.token){
          const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.cardsPath, cfg.branch, cfg.token);
          if(meta){
            const txt = ub64(meta.content);
            if(txt !== JSON.stringify(cards)){
              cards = JSON.parse(txt);
              renderAdminCards(); renderSubmitHand(); renderHandArea(); renderSubmissions();
            }
          }
        }
      }

      // rooms
      let remoteRoomsText = null;
      try { remoteRoomsText = await rawGet(cfg.owner, cfg.repo, cfg.branch, cfg.roomsPath); } catch(e){}
      if(remoteRoomsText !== null){
        const parsedRooms = JSON.parse(remoteRoomsText);
        if(JSON.stringify(parsedRooms) !== JSON.stringify(rooms)){
          rooms = parsedRooms;
          // if current room updated -> refresh
          if(document.getElementById('lobbyView') && !document.getElementById('lobbyView').classList.contains('hidden')) refreshLobby();
          if(document.getElementById('gameView') && !document.getElementById('gameView').classList.contains('hidden')) refreshGameUI();
          el('connectMsg').innerText = 'rooms.json が更新されました（リモート）';
        }
      } else {
        if(cfg.token){
          const meta = await ghGetFile(cfg.owner, cfg.repo, cfg.roomsPath, cfg.branch, cfg.token);
          if(meta){
            const txt = ub64(meta.content);
            if(txt !== JSON.stringify(rooms)){
              rooms = JSON.parse(txt);
              if(document.getElementById('lobbyView') && !document.getElementById('lobbyView').classList.contains('hidden')) refreshLobby();
              if(document.getElementById('gameView') && !document.getElementById('gameView').classList.contains('hidden')) refreshGameUI();
            }
          }
        }
      }
    }
  }catch(err){ console.warn('poll error', err); }
}

/* start/stop poll */
function startPolling(){
  if(pollTimer) clearInterval(pollTimer);
  pollTimer = setInterval(()=>{ pollOnce(); }, POLL_MS);
}
function stopPolling(){ if(pollTimer) clearInterval(pollTimer); pollTimer=null; }

/* -------------------------
   UI rendering (hand/submit/admin)
   ------------------------- */

function placeholderForId(id){
  const svg = `<svg xmlns='http://www.w3.org/2000/svg' width='200' height='300'><rect width='100%' height='100%' fill='#eee'/><text x='50%' y='50%' dominant-baseline='middle' text-anchor='middle' fill='#666' font-size='18'>${id}</text></svg>`;
  return 'data:image/svg+xml;base64,' + btoa(svg);
}

/* render admin cards */
function renderAdminCards(){
  const wrap = el('adminCards'); if(!wrap) return; wrap.innerHTML = '';
  const search = (el('adminSearch')||{value:''}).value.toLowerCase();
  for(const c of cards){
    if(search && !(c.id.includes(search) || c.name.toLowerCase().includes(search))) continue;
    const box = document.createElement('div'); box.className='card';
    const img = document.createElement('img'); img.src = c.img || placeholderForId(c.id); img.style.height='90px';
    const name = document.createElement('input'); name.value = c.name; name.onchange = async ()=> {
      c.name = name.value;
      try { await saveCardsToRepo('Update card name ' + c.id); el('connectMsg').innerText = 'カード名を保存しました'; } catch(e){ alert('保存失敗: '+e.message); }
    };
    const file = document.createElement('input'); file.type='file'; file.accept='image/*';
    file.onchange = (ev)=> {
      const f = ev.target.files[0];
      const r = new FileReader();
      r.onload = async (e)=> { c.img = e.target.result; try { await saveCardsToRepo('Update card img ' + c.id); renderAdminCards(); el('connectMsg').innerText = 'カード画像を保存しました'; } catch(e){ alert('保存失敗: '+e.message); } };
      if(f) r.readAsDataURL(f);
    };
    box.appendChild(img); box.appendChild(name); box.appendChild(file);
    wrap.appendChild(box);
  }
}

/* render submit hand */
function renderSubmitHand(){
  const container = el('submitHand'); if(!container) return;
  container.innerHTML = '';
  const room = currentRoomObj(); if(!room) return;
  const player = room.players.find(p=>p.id===actingPlayerId);
  if(!player) return;
  player.hand.forEach((c)=>{
    const meta = cards.find(x=>x.id===c.id);
    const div = document.createElement('div'); div.className='card'; div.dataset.id = c.id;
    const img = document.createElement('img'); img.src = (c.img || (meta && meta.img) || placeholderForId(c.id));
    const nm = document.createElement('div'); nm.textContent = (c.name || (meta && meta.name) || c.id);
    div.appendChild(img); div.appendChild(nm);
    container.appendChild(div);
  });
  makeSortable(container);
}

/* render hand area */
function renderHandArea(){
  const room = currentRoomObj(); if(!room) return;
  const player = room.players.find(p=>p.id===actingPlayerId);
  const wrap = el('handArea'); if(!wrap) return; wrap.innerHTML = '';
  if(!player) return;
  player.hand.forEach((c,idx)=>{
    const meta = cards.find(x=>x.id===c.id);
    const div = document.createElement('div'); div.className='card'; div.dataset.index = idx;
    const img = document.createElement('img'); img.src = (c.img || (meta && meta.img) || placeholderForId(c.id));
    const nm = document.createElement('div'); nm.textContent = (c.name || (meta && meta.name) || c.id);
    div.appendChild(img); div.appendChild(nm);
    div.onclick = ()=> { div.classList.toggle('selected'); };
    wrap.appendChild(div);
  });
  renderSubmitHand();
}

/* make sortable (drag/drop) */
function makeSortable(container){
  let dragEl = null;
  Array.from(container.children).forEach(ch=>{
    ch.draggable = true;
    ch.ondragstart = (e)=>{ dragEl = ch; e.dataTransfer.setData('text/plain',''); ch.style.opacity = .4; };
    ch.ondragend = ()=>{ if(dragEl) dragEl.style.opacity = 1; dragEl = null; };
    ch.ondragover = (e)=>{ e.preventDefault(); };
    ch.ondrop = (e)=>{ e.preventDefault(); if(!dragEl || dragEl===ch) return; container.insertBefore(dragEl, ch.nextSibling); };
  });
}

/* render submissions for vote */
function renderSubmissions(){
  const room = currentRoomObj(); if(!room) return;
  const container = el('submissionsArea'); if(!container) return; container.innerHTML = '';
  for(const pid of room.turnOrder){
    const sub = room.submitted[pid] || {order: room.players.find(p=>p.id===pid).hand.map(c=>c.id), roleName:''};
    const box = document.createElement('div'); box.className='card';
    box.innerHTML = `<div style="font-weight:700">${escapeHtml((room.players.find(p=>p.id===pid)||{}).nick)}</div><div class="small">${escapeHtml(sub.roleName)}</div>`;
    const row = document.createElement('div'); row.style.display='flex'; row.style.gap='6px'; row.style.marginTop='6px';
    sub.order.forEach(cid=>{
      const cc = cards.find(x=>x.id===cid) || {id:cid,name:cid,img:null};
      const thumb = document.createElement('img'); thumb.src = cc.img || placeholderForId(cc.id); thumb.style.width='46px'; thumb.style.height='68px'; thumb.style.objectFit='cover';
      row.appendChild(thumb);
    });
    box.appendChild(row);
    if(actingPlayerId && actingPlayerId !== pid && !(room.votes && room.votes[actingPlayerId])){
      const voteBtn = document.createElement('button'); voteBtn.textContent='投票'; voteBtn.onclick = async ()=>{
        room.votes = room.votes || {}; room.voteCounts = room.voteCounts || {};
        room.votes[actingPlayerId] = pid;
        room.voteCounts[pid] = (room.voteCounts[pid]||0)+1;
        try { await saveRoomsToRepo('Vote update'); renderSubmissions(); el('connectMsg').innerText = '投票を保存しました'; } catch(e){ alert('保存失敗: '+e.message); }
      };
      box.appendChild(voteBtn);
    } else {
      const note = document.createElement('div'); note.className='small'; note.innerText = actingPlayerId===pid ? 'あなたの提出' : ((room.votes && room.votes[actingPlayerId]) ? '投票済' : '');
      box.appendChild(note);
    }
    container.appendChild(box);
  }
}

/* refreshLobby */
function refreshLobby(){
  const room = currentRoomObj(); if(!room) return;
  el('roomCodeDisplay').innerText = room.code;
  el('roomOwner').innerText = (room.owner? (room.players.find(p=>p.id===room.owner)||{}).nick : '');
  const wrap = el('playersList'); wrap.innerHTML = '';
  room.players.forEach(p=>{
    const div = document.createElement('div'); div.className='card';
    div.innerHTML = `<div style="font-weight:700">${escapeHtml(p.nick)}</div><div class="small">ready: ${p.ready? '✅': '—'}</div>`;
    if(room.owner === actingPlayerId && p.id !== room.owner){
      const b = document.createElement('button'); b.textContent='Kick'; b.onclick = async ()=> {
        // update server-side
        try{
          const updated = room.players.filter(x=>x.id!==p.id);
          await ghUpdateRoomPlayers(room.code, updated);
          el('connectMsg').innerText = 'キックしました';
        }catch(e){ alert('キック失敗: '+e.message); }
      };
      div.appendChild(b);
    }
    wrap.appendChild(div);
  });
  const sel = el('actAsSelect'); sel.innerHTML = '';
  room.players.forEach(p => { const o = document.createElement('option'); o.value = p.id; o.text = p.nick; sel.appendChild(o); });
  sel.value = actingPlayerId || (room.players[0] && room.players[0].id) || '';
  sel.onchange = ()=> { actingPlayerId = sel.value; renderHandArea(); };
  const allReady = room.players.length >= 4 && room.players.every(p=>p.ready);
  el('btnStartGame').classList.toggle('hidden', !(allReady && actingPlayerId===room.owner));
  el('btnDissolve').classList.toggle('hidden', actingPlayerId===room.owner? false : true);
}

/* helper current room */
function currentRoomObj(){ return currentRoom ? rooms[currentRoom] : null; }

/* -------------------------
   Helper: update players array in room doc (via rooms.json)
   ------------------------- */
async function ghUpdateRoomPlayers(code, players, message){
  if(!rooms[code]) throw new Error('room not found');
  rooms[code].players = players;
  if(rooms[code].owner && !players.find(p=>p.id===rooms[code].owner)){
    rooms[code].owner = players.length>0 ? players[0].id : null;
  }
  await saveRoomsToRepo(message || `Update players of ${code}`);
}

/* -------------------------
   Room actions (create/join/leave etc.) stored in rooms object & saved to GitHub
   ------------------------- */

async function createRoom(){
  const cfg = ghConfig();
  if(!cfg.owner || !cfg.repo){ alert('まず GitHub 設定を入力して接続してください'); return; }
  const code = (el('newCode').value||'').trim();
  const max = parseInt(el('maxPlayers').value||4);
  const creator = (el('creatorName').value||'Owner').trim();
  if(!/^\d{4}$/.test(code)){ alert('合言葉は4桁の数字にしてください'); return; }
  reloadRoomsFromRepo(); // ensure latest
  if(rooms[code]){ alert('その合言葉は既に使われています'); return; }
  const room = {
    code, owner:null, maxPlayers: Math.min(6, Math.max(4,max)),
    players:[], state:'waiting', deck:[], discard:[], turnOrder:[], exchangeRound:1, exchangeIndex:0,
    submitted:{}, votes:{}, voteCounts:{}, phase:'waiting'
  };
  rooms[code] = room;
  try {
    await saveRoomsToRepo(`Create room ${code}`);
    // now join as creator
    await joinRoomWithName(code, creator, true);
    el('startMsg').innerText = `部屋 ${code} を作成しました。`;
  } catch(e){ alert('作成失敗: '+e.message); delete rooms[code]; }
}

async function joinRoom(){
  const code = (el('joinCode').value||'').trim();
  const name = (el('joinName').value||('P'+Math.floor(Math.random()*1000))).trim();
  if(!/^\d{4}$/.test(code)){ el('joinMsg').innerText = '合言葉は4桁'; return; }
  await reloadRoomsFromRepo();
  if(!rooms[code]){ el('joinMsg').innerText = 'その合言葉の部屋はありません'; return; }
  await joinRoomWithName(code, name, false);
}

async function joinRoomWithName(code, name, isCreator){
  await reloadRoomsFromRepo();
  const room = rooms[code];
  if(!room){ alert('その合言葉の部屋はありません'); return; }
  if(room.players.length >= room.maxPlayers){ alert('満員です'); return; }
  const pid = uid('p');
  const player = {id:pid, nick: name, ready:false, hand:[], score:0, connected:true};
  room.players.push(player);
  if(isCreator) room.owner = pid;
  try {
    await saveRoomsToRepo(`Join ${name} to ${code}`);
    currentRoom = code; actingPlayerId = pid;
    el('myDisplay').value = player.nick;
    refreshLobby();
    showView('lobbyView');
    log(`${player.nick} が部屋 ${code} に参加しました`);
  } catch(e){
    alert('参加失敗: '+e.message);
    // revert local change by reloading
    await reloadRoomsFromRepo();
  }
}

async function leaveRoom(){
  if(!currentRoom || !actingPlayerId) return;
  const room = rooms[currentRoom];
  const idx = room.players.findIndex(p=>p.id===actingPlayerId);
  if(idx!==-1){
    const name = room.players[idx].nick;
    room.players.splice(idx,1);
    if(room.owner === actingPlayerId){
      if(room.players.length>0) room.owner = room.players[0].id; else delete rooms[room.code];
    }
    try { await saveRoomsToRepo(`${name} left ${room.code}`); } catch(e){ alert('退出保存失敗: '+e.message); }
  }
  currentRoom = null; actingPlayerId = null;
  showView('startView');
}

/* dissolve */
async function dissolveRoom(){
  if(!currentRoom || !actingPlayerId) return;
  const room = rooms[currentRoom];
  if(room.owner !== actingPlayerId){ alert('権限なし'); return; }
  if(!confirm('本当に部屋を解散しますか？')) return;
  delete rooms[room.code];
  try { await saveRoomsToRepo(`Dissolve room ${room.code}`); } catch(e){ alert('解散保存失敗: '+e.message); }
  currentRoom = null; actingPlayerId = null;
  showView('startView');
}

/* toggleReady */
async function toggleReadyUI(){
  if(!currentRoom || !actingPlayerId) return;
  const room = rooms[currentRoom];
  const p = room.players.find(x=>x.id===actingPlayerId);
  if(!p) return;
  p.ready = !p.ready;
  try { await saveRoomsToRepo(`${p.nick} ready=${p.ready}`); } catch(e){ alert('保存失敗: '+e.message); }
}

/* -------------------------
   Reload helpers
   ------------------------- */
async function reloadCardsFromRepo(){
  try{ await loadCardsFromRepo(); renderAdminCards(); renderSubmitHand(); renderHandArea(); renderSubmissions(); } catch(e){ console.warn('reloadCards failed', e); }
}
async function reloadRoomsFromRepo(){
  try{ await loadRoomsFromRepo(); } catch(e){ console.warn('reloadRooms failed', e); }
}

/* -------------------------
   Initialization & UI wiring
   ------------------------- */
el('btnConnect').onclick = async ()=>{
  const token = el('ghToken').value.trim();
  setTokenInSession(token);
  const cfg = ghConfig();
  if(!cfg.owner || !cfg.repo){ el('connectMsg').innerText = 'owner/repo を入力してください'; return; }
  try {
    el('connectMsg').innerText = '読み込み中...';
    await reloadCardsFromRepo();
    await reloadRoomsFromRepo();
    el('connectMsg').innerText = '読み込み完了';
    startPolling();
  } catch(e){
    el('connectMsg').innerText = '読み込み失敗: ' + e.message;
  }
};
el('btnClearToken').onclick = ()=> { setTokenInSession(''); el('connectMsg').innerText = 'トークンを削除しました'; };

el('btnCreate').onclick = ()=> createRoom();
el('btnJoinExisting').onclick = ()=> { showView('joinView'); };
el('btnBackStart').onclick = ()=> showView('startView');
el('btnJoin').onclick = ()=> joinRoom();
el('btnReady').onclick = ()=> toggleReadyUI();
el('btnLeave').onclick = ()=> leaveRoom();
el('btnDissolve').onclick = ()=> dissolveRoom();
el('btnOpenAdmin').onclick = ()=> { showView('adminPanel'); renderAdminCards(); };
el('btnExport').onclick = ()=> {
  const data = JSON.stringify(cards, null, 2);
  const blob = new Blob([data], {type:'application/json'});
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'cards.json'; a.click();
};
el('btnImport').onclick = ()=> el('importFile').click();
el('importFile').onchange = async (e)=> {
  const f=e.target.files[0]; if(!f) return;
  const r=new FileReader(); r.onload = async ev => {
    try{
      const parsed = JSON.parse(ev.target.result);
      if(Array.isArray(parsed)){ cards = parsed; await saveCardsToRepo('Import cards'); renderAdminCards(); alert('インポート完了'); }
      else alert('不正なファイル');
    }catch(err){ alert('読み込み失敗'); }
  }; r.readAsText(f);
};

/* showView */
function showView(id){
  const views = ['startView','joinView','lobbyView','adminPanel'];
  views.forEach(v=>{ const elv = document.getElementById(v); if(elv) elv.classList.add('hidden'); });
  const t=document.getElementById(id); if(t) t.classList.remove('hidden');
}

/* initial simple init */
(function boot(){
  // try to prefill owner/repo from URL or sessionStorage
  const storedOwner = sessionStorage.getItem('GH_OWNER'); if(storedOwner) el('ghOwner').value = storedOwner;
  const storedRepo = sessionStorage.getItem('GH_REPO'); if(storedRepo) el('ghRepo').value = storedRepo;
  const token = sessionStorage.getItem('GH_TOKEN');
  if(token) el('ghToken').value = token;
  showView('startView');
})();

/* save settings to session when changed */
['ghOwner','ghRepo','ghBranch','cardsPath','roomsPath'].forEach(id=>{
  const input=el(id); if(!input) return;
  input.addEventListener('change', ()=> {
    sessionStorage.setItem(id.toUpperCase(), input.value);
  });
});

/* expose some functions for debugging */
window._app = {
  loadCardsFromRepo, loadRoomsFromRepo, saveCardsToRepo, saveRoomsToRepo, pollOnce, ghConfig
};

</script>
</body>
</html>

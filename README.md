<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>オセロニアポーカー — リアルタイム完全版 (Firestore)</title>
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
  textarea{width:100%;min-height:64px}
  .selected{outline:3px solid rgba(43,109,246,.18)}
  .hit{background:#eaf5ff}
  footer{font-size:12px;color:#666;margin-top:12px}
  .btn-primary{background:var(--accent);color:#fff;border:none}
  .notice{padding:8px;background:#fff7e6;border-radius:6px;border:1px solid #ffe6a8;color:#664400}
  .muted{color:#888;font-size:12px}
</style>
</head>
<body>
<header><h1 style="display:inline">オセロニアポーカー — リアルタイム版（Firestore）</h1></header>
<div class="wrap">

  <!-- Firebase config / quick start -->
  <div id="setupPanel" class="status">
    <strong>セットアップ（最初に必要）</strong>
    <div class="small">
      1) Firebase コンソールでプロジェクトを作成。Firestore を有効化し、Authentication → Sign-in method で「匿名」を有効にしてください。<br>
      2) プロジェクト設定 → アプリを追加（Web）で firebaseConfig を取得し、下のフォームに貼り付けて「接続して開始」ボタンを押してください。<br>
      （セキュリティルールは開発用に緩くしてください：request.auth != null を使う想定です）
    </div>
    <div style="margin-top:8px">
      <textarea id="firebaseConfig" placeholder="Paste firebaseConfig JSON here (e.g. { apiKey: '...', projectId: '...'} )" style="width:100%;min-height:68px"></textarea>
      <div style="margin-top:8px" class="controls"><button id="btnInit" class="btn-primary">接続して開始</button> <button id="btnClear">設定をクリア</button></div>
      <div id="setupMsg" class="small"></div>
    </div>
  </div>

  <!-- Start / create room -->
  <div id="startView" class="hidden" style="margin-top:12px">
    <div class="row">
      <div class="col" style="flex:1">
        <label>合言葉（4桁・新規作成）<input id="newCode" maxlength="4" placeholder="例: 1234"></label>
        <label>人数上限<select id="maxPlayers"><option>4</option><option>5</option><option>6</option></select></label>
        <label>あなたのニックネーム<input id="creatorName" value="Player"></label>
        <div class="controls"><button id="btnCreate" class="btn-primary">部屋を作る</button>
        <button id="btnJoinExisting">合言葉で入室</button>
        <button id="btnAdminLogin">管理者ログイン</button></div>
        <div id="startMsg" class="small"></div>
      </div>
      <div style="width:340px">
        <div class="status"><strong>動作説明</strong>
          <div class="small">Firestore をバックエンドにして複数端末でリアルタイム同期します。管理者はカード編集が可能です。</div>
        </div>
      </div>
    </div>
  </div>

  <!-- Join -->
  <div id="joinView" class="hidden" style="margin-top:12px">
    <div class="row">
      <div class="col" style="flex:1">
        <label>合言葉（4桁）<input id="joinCode" maxlength="4"></label>
        <label>ニックネーム<input id="joinName" value="Player"></label>
        <div class="controls">
          <button id="btnJoin" class="btn-primary">入室</button>
          <button id="btnBackStart">戻る</button>
        </div>
        <div id="joinMsg" class="small"></div>
      </div>
    </div>
  </div>

  <!-- Lobby / Game -->
  <div id="lobbyView" class="hidden" style="margin-top:12px">
    <div class="row">
      <div style="flex:1">
        <div class="row" style="align-items:center;gap:8px">
          <div><strong>ルーム</strong> <span id="roomCodeDisplay"></span></div>
          <div class="small">部屋主: <span id="roomOwner"></span></div>
          <div style="margin-left:auto"><button id="btnBackToStart">スタートへ</button></div>
        </div>

        <div style="margin-top:8px">
          <label>Act as:
            <select id="actAsSelect"></select>
          </label>
          <label style="margin-left:8px">自分の表示名<input id="myDisplay" style="width:160px"></label>
        </div>

        <div style="margin-top:8px" class="controls">
          <button id="btnReady">準備OK</button>
          <button id="btnStartGame" class="hidden btn-primary">ゲーム開始（部屋主のみ）</button>
          <button id="btnLeave">退出</button>
          <button id="btnDissolve" class="hidden">部屋解散（部屋主）</button>
        </div>

        <div style="margin-top:12px"><strong>プレイヤー一覧</strong></div>
        <div id="playersList" class="card-grid" style="margin-top:8px"></div>

        <!-- Game panels -->
        <div id="gameArea" style="margin-top:12px" class="hidden">
          <div><strong>状態:</strong> <span id="phaseLabel" class="small hit">—</span></div>
          <div style="margin-top:8px">
            <h3>あなたの手札（Act as に合わせて表示）</h3>
            <div id="handArea" class="card-grid"></div>
          </div>

          <div id="exchangePanel" class="hidden" style="margin-top:12px">
            <h4>交換フェーズ</h4>
            <div>ラウンド: <span id="exchangeRoundDisplay"></span> / <span id="exchangeIndexDisplay"></span></div>
            <div class="controls"><button id="btnExchangeDo">選択したカードを交換</button> <button id="btnExchangeOK">交換なし</button></div>
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

          <div style="margin-top:12px" id="ownerControls" class="hidden">
            <div class="small">部屋主コントロール（手動で進行）</div>
            <div class="controls" style="margin-top:8px">
              <button id="btnOwnerAdvanceExchange" class="btn-primary">次の交換ターンへ</button>
              <button id="btnOwnerStartSubmit" class="btn-primary">提出フェーズへ</button>
              <button id="btnOwnerStartVote" class="btn-primary">投票フェーズへ</button>
              <button id="btnOwnerFinishVote" class="btn-primary">投票終了・結果表示</button>
            </div>
          </div>
        </div>

      </div>

      <div style="width:340px">
        <div class="status">
          <div><strong>カード / 山札</strong></div>
          <div class="small">カードデータは Firestore の meta/cards に保存されます。管理者ログインで編集可能。</div>
        </div>
        <div style="margin-top:12px">
          <strong>管理</strong>
          <div class="small">管理者はカード編集ができます（匿名認証を使っています）。</div>
          <div style="margin-top:8px"><button id="btnOpenAdmin">管理画面を開く</button></div>
        </div>
      </div>
    </div>
  </div>

  <!-- Admin panel -->
  <div id="adminPanel" class="hidden" style="margin-top:12px">
    <h3>管理画面 — カード編集</h3>
    <div class="small">カードは Firestore の meta/cards で一括管理。画像は DataURL として保存されます（サイズ注意）。</div>
    <div style="margin-top:8px">
      <label>カード検索（ID or 名前）<input id="adminSearch" placeholder="例: 001"></label>
    </div>
    <div id="adminCards" class="card-grid" style="margin-top:8px;height:420px;overflow:auto"></div>
    <div style="margin-top:8px">
      <button id="btnExport">カードデータをエクスポート（JSON）</button>
      <button id="btnImport">カードデータをインポート（JSON）</button>
      <input type="file" id="importFile" accept="application/json" class="hidden">
    </div>
  </div>

  <div style="margin-top:12px">
    <strong>操作ログ</strong>
    <div id="logArea" style="height:180px;overflow:auto;border:1px solid #eee;padding:8px;background:#fafafa"></div>
  </div>

  <footer style="margin-top:12px" class="small">※このページは Firestore を利用します。firebaseConfig を設定し、Firestore と匿名認証を有効にしてください。</footer>
</div>

<!-- Firebase compat SDKs for simpler migration with this single-file -->
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-firestore-compat.js"></script>

<script>
/*
  Complete single-file app using Firestore realtime listeners.
  - Replace firebaseConfig JSON in the textarea and press "接続して開始".
  - Enable Firestore and Anonymous Auth in your Firebase project.
  - Security rules should require request.auth != null for writes (recommended).
  - This implementation uses owner-driven phase progression to keep transitions robust.
*/

/* ---------- Utilities ---------- */
function el(id){ return document.getElementById(id); }
function uid(prefix='p'){ return prefix + Math.random().toString(36).slice(2,9); }
function escapeHtml(s){ return (s+'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
function log(msg){ el('logArea').innerHTML = `<div class="small">[${(new Date()).toLocaleTimeString()}] ${escapeHtml(msg)}</div>` + el('logArea').innerHTML; }

/* ---------- Firebase state ---------- */
let firebaseApp = null;
let auth = null;
let db = null;
let currentUser = null;

/* Cached realtime listeners */
let cardsUnsub = null;
let roomUnsub = null;

/* Local cache */
let cards = []; // array of {id,name,img}
let currentRoom = null; // room code string
let localRoomsCache = {}; // not always used; main state stored in Firestore
let actingPlayerId = null;

/* ---------- Init / Connect ---------- */
el('btnInit').onclick = async () => {
  const txt = el('firebaseConfig').value.trim();
  if(!txt){ el('setupMsg').innerText = 'firebaseConfig を貼り付けてください'; return; }
  let cfg = null;
  try { cfg = JSON.parse(txt); } catch(e){ el('setupMsg').innerText = 'firebaseConfig は JSON 形式です'; return; }
  try {
    if(firebase.apps && firebase.apps.length){ firebase.app().delete().catch(()=>{}); } // cleanup
  } catch(e){}
  try {
    firebaseApp = firebase.initializeApp(cfg);
    auth = firebase.auth();
    db = firebase.firestore();
    el('setupMsg').innerText = 'Firebase 初期化中...';
    // anonymous sign-in
    auth.onAuthStateChanged(u => {
      if(u){ currentUser = u; el('setupMsg').innerText = '認証済み: UID=' + u.uid; afterAuthInit(); }
      else { auth.signInAnonymously().catch(err => el('setupMsg').innerText = '匿名サインイン失敗: '+err.message); }
    });
  } catch(e){
    el('setupMsg').innerText = 'Firebase 初期化エラー: ' + e.message;
  }
};

function afterAuthInit(){
  // Load cards & listen realtime
  attachCardsListener();
  showView('startView');
  el('startView').classList.remove('hidden');
  log('アプリ準備完了（Firebase 接続済み）');
}

/* ---------- Firestore helpers ---------- */
function cardsDocRef(){ return db.doc('meta/cards'); }
function roomDocRef(code){ return db.collection('rooms').doc(code); }

/* Attach realtime listener to cards doc */
function attachCardsListener(){
  if(cardsUnsub) cardsUnsub();
  cardsUnsub = cardsDocRef().onSnapshot(doc => {
    if(!doc.exists){
      // initialize default cards
      const arr=[];
      for(let i=1;i<=90;i++){ const id = String(i).padStart(3,'0'); arr.push({id,name:`Card ${id}`,img:null}); }
      cardsDocRef().set({list: arr}).then(()=> log('cards doc 初期化')).catch(e=>log('cards init failed: '+e.message));
      return;
    }
    const d = doc.data();
    cards = d.list || [];
    renderAdminCards();
    renderSubmitHand();
    renderHandArea();
    renderSubmissions();
    log('カードデータが更新されました');
  }, err => { log('cards listener error: ' + err.message); });
}

/* Listen to a specific room in realtime (when joined) */
function attachRoomListener(code){
  if(roomUnsub) roomUnsub();
  roomUnsub = roomDocRef(code).onSnapshot(doc => {
    if(!doc.exists){
      log('ルームが削除されました: ' + code);
      // if we were in that room, leave
      if(currentRoom === code){ currentRoom = null; showView('startView'); }
      return;
    }
    const room = doc.data();
    localRoomsCache[code] = room;
    // if this is our current room, update UI
    if(currentRoom === code){
      refreshLobby(); refreshGameUI();
    }
    log('ルーム更新: ' + code);
  }, err => { log('room listener error: ' + err.message); });
}

/* ---------- Room management (Firestore-backed) ---------- */

async function createRoom(){
  const code = (el('newCode').value||'').trim();
  const max = parseInt(el('maxPlayers').value||4);
  const creator = (el('creatorName').value||'Owner').trim();
  if(!/^\d{4}$/.test(code)){ alert('合言葉は4桁の数字で入力してください'); return; }
  // check exists
  const ref = roomDocRef(code);
  const doc = await ref.get();
  if(doc.exists){ alert('その合言葉は既に使われています'); return; }
  // create room
  const room = {
    code,
    owner: null,
    maxPlayers: Math.min(6, Math.max(4,max)),
    players: [],
    state: 'waiting',
    deck: [],
    discard: [],
    turnOrder: [],
    exchangeRound: 1,
    exchangeIndex: 0,
    submitted: {},
    votes: {},
    voteCounts: {},
    phase: 'waiting',
    createdAt: firebase.firestore.FieldValue.serverTimestamp()
  };
  await ref.set(room);
  // join as creator
  await joinRoomWithName(code, creator, true);
  log('部屋を作成しました: ' + code);
}

/* Join by code: uses transaction to avoid races */
async function joinRoom(){
  const code = (el('joinCode').value||'').trim();
  const name = (el('joinName').value||('P'+Math.floor(Math.random()*1000))).trim();
  if(!/^\d{4}$/.test(code)){ el('joinMsg').innerText = '合言葉は4桁'; return; }
  try{
    await joinRoomWithName(code, name, false);
    el('joinMsg').innerText = '';
  } catch(e){
    el('joinMsg').innerText = e.message || e;
  }
}

async function joinRoomWithName(code, name, isCreator){
  const ref = roomDocRef(code);
  const res = await db.runTransaction(async tx => {
    const doc = await tx.get(ref);
    if(!doc.exists) throw new Error('その合言葉の部屋はありません');
    const room = doc.data();
    if(room.players.length >= room.maxPlayers) throw new Error('満員です');
    const pid = uid('p');
    const player = {id: pid, nick: name, ready:false, hand:[], score:0, connected:true};
    const newPlayers = room.players.slice();
    newPlayers.push(player);
    const update = {players: newPlayers};
    if(isCreator) update.owner = pid;
    tx.update(ref, update);
    return {room: {...room, players: newPlayers, owner: update.owner || room.owner}, player};
  });
  // local state
  localRoomsCache[code] = res.room;
  currentRoom = code;
  actingPlayerId = res.player.id;
  el('myDisplay').value = res.player.nick;
  attachRoomListener(code);
  refreshLobby();
  showView('lobbyView');
  log(`${res.player.nick} が部屋 ${code} に参加しました`);
}

/* Leave room */
async function leaveRoom(){
  if(!currentRoom || !actingPlayerId) return;
  const code = currentRoom;
  const ref = roomDocRef(code);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref);
    if(!doc.exists) return;
    const room = doc.data();
    const players = room.players.filter(p=>p.id !== actingPlayerId);
    let owner = room.owner;
    if(room.owner === actingPlayerId) owner = players.length>0 ? players[0].id : null;
    if(players.length === 0) tx.delete(ref);
    else tx.update(ref, {players, owner});
  });
  currentRoom = null; actingPlayerId = null;
  showView('startView');
  log('退出しました');
}

/* Dissolve */
async function dissolveRoom(){
  if(!currentRoom || !actingPlayerId) return;
  const code = currentRoom;
  const ref = roomDocRef(code);
  const doc = await ref.get();
  if(!doc.exists) return;
  const room = doc.data();
  if(room.owner !== actingPlayerId){ alert('権限がありません'); return; }
  if(!confirm('本当に部屋を解散しますか？')) return;
  await ref.delete();
  currentRoom = null; actingPlayerId = null;
  showView('startView');
  log('部屋を解散しました');
}

/* Toggle ready */
async function toggleReady(){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref); if(!doc.exists) throw new Error('room gone');
    const room = doc.data();
    const players = room.players.map(p => p.id===actingPlayerId ? {...p, ready: !p.ready} : p);
    tx.update(ref, {players});
  });
  log('準備状態を切替');
}

/* Kick player (owner only) */
async function kickPlayer(targetId){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref); if(!doc.exists) throw new Error('room gone');
    const room = doc.data();
    if(room.owner !== actingPlayerId) throw new Error('権限なし');
    const players = room.players.filter(p=>p.id!==targetId);
    tx.update(ref, {players});
  });
  log('プレイヤーをキックしました');
}

/* ---------- Game flow (owner-driven, to ensure robust transitions) ---------- */

/* Start game (owner) - distribute deck */
async function startGameAsOwner(){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref); if(!doc.exists) throw new Error('room gone');
    const room = doc.data();
    if(room.owner !== actingPlayerId) throw new Error('部屋主のみ開始可能');
    if(room.players.length < 2) throw new Error('プレイヤー不足'); // adjust as needed
    // build deck from current cards doc
    const cardsDoc = await tx.get(cardsDocRef());
    if(!cardsDoc.exists) throw new Error('cards not initialized');
    const cardList = cardsDoc.data().list || [];
    const deck = cardList.map(c=>({id:c.id, name:c.name, img:c.img}));
    // shuffle
    for(let i=deck.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [deck[i],deck[j]]=[deck[j],deck[i]]; }
    // distribute 5 each
    const order = room.players.map(p=>p.id);
    // shuffle order
    for(let i=order.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [order[i],order[j]]=[order[j],order[i]]; }
    const newPlayers = room.players.map(p => ({...p, hand:[], score:0}));
    for(const pid of order){
      const idx = newPlayers.findIndex(x=>x.id===pid);
      if(idx!==-1){
        newPlayers[idx].hand = deck.splice(0,5);
      }
    }
    tx.update(ref, {
      deck,
      discard: [],
      state: 'playing',
      turnOrder: order,
      exchangeRound: 1,
      exchangeIndex: 0,
      players: newPlayers,
      submitted: {},
      votes: {},
      voteCounts: {},
      phase: 'exchange'
    });
  });
  log('ゲーム開始しました');
}

/* Owner advances exchange turn (manual) */
async function ownerAdvanceExchange(){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref); if(!doc.exists) throw new Error('room gone');
    const room = doc.data();
    if(room.owner !== actingPlayerId) throw new Error('権限なし');
    let idx = (room.exchangeIndex||0) + 1;
    let rnd = room.exchangeRound||1;
    if(idx >= (room.turnOrder||[]).length){
      // next round
      rnd = (room.exchangeRound||1) + 1;
      idx = 0;
    }
    const updates = { exchangeIndex: idx, exchangeRound: rnd };
    // if rounds finished (>2) move to submit
    if(rnd > 2){
      updates.phase = 'submit';
    }
    tx.update(ref, updates);
  });
  log('交換ターンを進めました');
}

/* Player performs exchange (must be current exchanger) */
async function doExchange(selectedIndices){
  // selectedIndices is array of indices in player's hand
  if(!currentRoom || !actingPlayerId) { alert('Act as を設定してください'); return; }
  const ref = roomDocRef(currentRoom);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref); if(!doc.exists) throw new Error('room gone');
    const room = doc.data();
    if(room.phase !== 'exchange') throw new Error('交換フェーズではありません');
    const currentPid = room.turnOrder[room.exchangeIndex||0];
    if(currentPid !== actingPlayerId) throw new Error('今はあなたの番ではありません');
    const players = room.players.map(p => ({...p}));
    const meIdx = players.findIndex(p=>p.id===actingPlayerId);
    if(meIdx===-1) throw new Error('player not found');
    const player = players[meIdx];
    // remove selected cards from hand (by index, descending)
    const removed = [];
    selectedIndices.sort((a,b)=>b-a).forEach(i => {
      if(i>=0 && i<player.hand.length) removed.push(player.hand.splice(i,1)[0]);
    });
    // push removed to deck bottom
    const deck = (room.deck || []).slice();
    deck.push(...removed);
    // draw same number
    const draw = deck.splice(0, removed.length);
    player.hand.push(...draw);
    // write back
    players[meIdx] = player;
    tx.update(ref, { players, deck, exchangeIndex: (room.exchangeIndex||0) + 1 });
  });
  log('交換を実行しました');
}

/* Player no-exchange */
async function exchangeOK(){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await ref.update({ exchangeIndex: firebase.firestore.FieldValue.increment(1) });
  log('交換なしでOKしました');
}

/* Owner manually start submit phase */
async function ownerStartSubmit(){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await ref.update({ phase: 'submit' });
  log('提出フェーズに移行しました');
}

/* Player submit */
async function submitWork(){
  if(!currentRoom || !actingPlayerId) return;
  const role = (el('roleInput').value||'').trim();
  // collect player's current hand order as ids
  const roomSnap = await roomDocRef(currentRoom).get();
  if(!roomSnap.exists) return;
  const room = roomSnap.data();
  const players = (room.players||[]).map(p=>({...p}));
  const me = players.find(p=>p.id===actingPlayerId);
  if(!me) return;
  const order = me.hand.map(c=>c.id);
  // update submitted map
  const submitted = room.submitted || {};
  submitted[actingPlayerId] = { order, roleName: role, submitted: true };
  await roomDocRef(currentRoom).update({ submitted });
  log('提出しました: ' + (role||'(無題)'));
}

/* Owner start vote phase */
async function ownerStartVote(){
  if(!currentRoom || !actingPlayerId) return;
  await roomDocRef(currentRoom).update({ phase: 'vote', votes: {}, voteCounts: {} });
  log('投票フェーズに移行しました');
}

/* Cast vote */
async function castVote(targetId){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref); if(!doc.exists) throw new Error('room gone');
    const room = doc.data();
    const votes = room.votes || {};
    const voteCounts = room.voteCounts || {};
    if(votes[actingPlayerId]) throw new Error('既に投票済みです');
    votes[actingPlayerId] = targetId;
    voteCounts[targetId] = (voteCounts[targetId]||0) + 1;
    tx.update(ref, { votes, voteCounts });
  });
  log('投票しました');
}

/* Owner finish vote and compute result */
async function ownerFinishVote(){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await db.runTransaction(async tx=>{
    const doc = await tx.get(ref); if(!doc.exists) throw new Error('room gone');
    const room = doc.data();
    const counts = room.voteCounts || {};
    // ensure all present have count entries
    const turnOrder = room.turnOrder || [];
    let max = -1;
    turnOrder.forEach(pid => { counts[pid] = counts[pid] || 0; if(counts[pid] > max) max = counts[pid]; });
    const winners = turnOrder.filter(pid => counts[pid] === max);
    tx.update(ref, { phase: 'result', result: { winners, counts } });
  });
  log('投票結果を確定しました');
}

/* Next game reset (owner) */
async function resetAfterGame(){
  if(!currentRoom || !actingPlayerId) return;
  const ref = roomDocRef(currentRoom);
  await ref.update({
    players: firebase.firestore.FieldValue.delete() // we'll rebuild via client leaving/joining for simplicity
  });
  // simpler: just set phase to waiting and clear game fields (actual removal kept minimal)
  await roomDocRef(currentRoom).update({
    state: 'waiting', deck: [], discard: [], turnOrder: [], exchangeRound:1, exchangeIndex:0,
    submitted: {}, votes: {}, voteCounts: {}, phase: 'waiting'
  });
  log('ゲームをリセットしました');
}

/* ---------- Cards admin (Firestore) ---------- */
function renderAdminCards(){
  const wrap = el('adminCards'); if(!wrap) return; wrap.innerHTML = '';
  const q = (el('adminSearch')||{value:''}).value.toLowerCase();
  for(const c of cards){
    if(q && !(c.id.includes(q) || c.name.toLowerCase().includes(q))) continue;
    const box = document.createElement('div'); box.className='card';
    const img = document.createElement('img'); img.src = c.img || placeholderForId(c.id); img.style.height='90px';
    const name = document.createElement('input'); name.value = c.name; name.onchange = async ()=> {
      c.name = name.value;
      await saveCardsToServer();
    };
    const file = document.createElement('input'); file.type='file'; file.accept='image/*'; file.onchange = (ev)=> {
      const f = ev.target.files[0];
      const r = new FileReader();
      r.onload = async (e)=> { c.img = e.target.result; await saveCardsToServer(); renderAdminCards(); };
      if(f) r.readAsDataURL(f);
    };
    box.appendChild(img); box.appendChild(name); box.appendChild(file);
    wrap.appendChild(box);
  }
}

function placeholderForId(id){
  const svg = `<svg xmlns='http://www.w3.org/2000/svg' width='200' height='300'><rect width='100%' height='100%' fill='#eee'/><text x='50%' y='50%' dominant-baseline='middle' text-anchor='middle' fill='#666' font-size='18'>${id}</text></svg>`;
  return 'data:image/svg+xml;base64,' + btoa(svg);
}

/* Fetch cards once (used on init) */
async function fetchCardsOnce(){
  const doc = await cardsDocRef().get();
  if(!doc.exists){
    const arr=[];
    for(let i=1;i<=90;i++){ const id = String(i).padStart(3,'0'); arr.push({id,name:`Card ${id}`,img:null}); }
    await cardsDocRef().set({list: arr});
    cards = arr;
  } else {
    cards = (doc.data().list || []);
  }
  renderAdminCards();
}

/* Save cards to server (firestore) */
async function saveCardsToServer(){
  await cardsDocRef().set({list: cards});
  log('カードを保存しました');
}

/* Export / Import */
el('btnExport').onclick = async ()=> {
  const doc = await cardsDocRef().get();
  const data = JSON.stringify(doc.exists ? doc.data().list : cards, null, 2);
  const blob = new Blob([data], {type:'application/json'});
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download='cards.json'; a.click();
};
el('btnImport').onclick = ()=> el('importFile').click();
el('importFile').onchange = (e)=> {
  const f = e.target.files[0]; if(!f) return;
  const r = new FileReader();
  r.onload = async ev => {
    try{
      const parsed = JSON.parse(ev.target.result);
      if(Array.isArray(parsed)){
        cards = parsed;
        await saveCardsToServer();
        renderAdminCards();
        alert('インポート完了');
      } else alert('不正なファイル');
    } catch(err){ alert('読み込み失敗'); }
  };
  r.readAsText(f);
};

/* ---------- Render UI helpers (hand/submit/vote) ---------- */

function renderSubmitHand(){
  const container = el('submitHand'); if(!container) return;
  container.innerHTML = '';
  if(!currentRoom) return;
  const room = localRoomsCache[currentRoom] || {};
  const player = (room.players||[]).find(p=>p.id===actingPlayerId);
  if(!player) return;
  (player.hand||[]).forEach(c => {
    const meta = cards.find(x=>x.id===c.id);
    const div = document.createElement('div'); div.className='card'; div.dataset.id = c.id;
    const img = document.createElement('img'); img.src = c.img || (meta && meta.img) || placeholderForId(c.id);
    const nm = document.createElement('div'); nm.textContent = c.name || (meta && meta.name) || c.id;
    div.appendChild(img); div.appendChild(nm); container.appendChild(div);
  });
  makeSortable(container);
}

function renderHandArea(){
  const wrap = el('handArea'); if(!wrap) return; wrap.innerHTML = '';
  if(!currentRoom) return;
  const room = localRoomsCache[currentRoom] || {};
  const player = (room.players||[]).find(p=>p.id===actingPlayerId);
  if(!player) return;
  (player.hand||[]).forEach((c,idx) => {
    const meta = cards.find(x=>x.id===c.id);
    const div = document.createElement('div'); div.className='card'; div.dataset.index = idx;
    const img = document.createElement('img'); img.src = c.img || (meta && meta.img) || placeholderForId(c.id);
    const nm = document.createElement('div'); nm.textContent = c.name || (meta && meta.name) || c.id;
    div.appendChild(img); div.appendChild(nm); div.onclick = ()=> div.classList.toggle('selected');
    wrap.appendChild(div);
  });
  renderSubmitHand();
}

function renderSubmissions(){
  const container = el('submissionsArea'); if(!container) return; container.innerHTML = '';
  if(!currentRoom) return;
  const room = localRoomsCache[currentRoom] || {};
  const turnOrder = room.turnOrder || [];
  for(const pid of turnOrder){
    const sub = (room.submitted && room.submitted[pid]) || {order: (room.players||[]).find(p=>p.id===pid).hand.map(c=>c.id), roleName: ''};
    const box = document.createElement('div'); box.className='card';
    box.innerHTML = `<div style="font-weight:700">${escapeHtml(((room.players||[]).find(p=>p.id===pid)||{}).nick || pid)}</div><div class="small">${escapeHtml(sub.roleName)}</div>`;
    const row = document.createElement('div'); row.style.display='flex'; row.style.gap='6px'; row.style.marginTop='6px';
    (sub.order||[]).forEach(cid => {
      const meta = cards.find(x=>x.id===cid) || {id:cid,name:cid,img:null};
      const thumb = document.createElement('img'); thumb.src = meta.img || placeholderForId(cid); thumb.style.width='46px'; thumb.style.height='68px'; thumb.style.objectFit='cover';
      row.appendChild(thumb);
    });
    box.appendChild(row);
    if(actingPlayerId && actingPlayerId !== pid && !(room.votes && room.votes[actingPlayerId])){
      const voteBtn = document.createElement('button'); voteBtn.textContent='投票'; voteBtn.onclick = ()=> { castVote(pid).catch(e=>alert('投票失敗:'+e.message)); };
      box.appendChild(voteBtn);
    } else {
      const note = document.createElement('div'); note.className='small';
      note.innerText = actingPlayerId === pid ? 'あなたの提出' : (room.votes && room.votes[actingPlayerId] ? '投票済' : '');
      box.appendChild(note);
    }
    container.appendChild(box);
  }
}

/* sortable for submit hand */
function makeSortable(container){
  let dragEl = null;
  Array.from(container.children).forEach(ch=>{
    ch.draggable = true;
    ch.ondragstart = (e)=>{ dragEl = ch; e.dataTransfer.setData('text/plain', ''); ch.style.opacity = .4; };
    ch.ondragend = ()=>{ if(dragEl) dragEl.style.opacity = 1; dragEl = null; };
    ch.ondragover = (e)=> e.preventDefault();
    ch.ondrop = (e)=>{ e.preventDefault(); if(!dragEl || dragEl===ch) return; container.insertBefore(dragEl, ch.nextSibling); };
  });
}

/* ---------- UI refresh functions ---------- */

const localRoomsCache = {}; // keep last snapshot for quick reads

// refresh lobby UI from cached room
function refreshLobby(){
  const room = localRoomsCache[currentRoom];
  if(!room) return;
  el('roomCodeDisplay').innerText = room.code;
  el('roomOwner').innerText = (room.owner ? (room.players.find(p=>p.id===room.owner)||{}).nick : '');
  const wrap = el('playersList'); wrap.innerHTML = '';
  (room.players||[]).forEach(p=>{
    const div = document.createElement('div'); div.className='card';
    div.innerHTML = `<div style="font-weight:700">${escapeHtml(p.nick)}</div><div class="small">ready: ${p.ready? '✅':'—'}</div>`;
    if(room.owner === actingPlayerId && p.id !== room.owner){
      const b = document.createElement('button'); b.textContent='Kick'; b.onclick = ()=> kickPlayer(p.id);
      div.appendChild(b);
    }
    wrap.appendChild(div);
  });
  const sel = el('actAsSelect'); sel.innerHTML='';
  (room.players||[]).forEach(p=>{ const o=document.createElement('option'); o.value=p.id; o.text=p.nick; sel.appendChild(o); });
  sel.value = actingPlayerId || '';
  sel.onchange = ()=> { actingPlayerId = sel.value; renderHandArea(); };
  el('btnStartGame').classList.toggle('hidden', !(room.players.length >= 4 && (room.players||[]).every(p=>p.ready) && actingPlayerId===room.owner));
  el('btnDissolve').classList.toggle('hidden', actingPlayerId===room.owner?false:true);
  // show/hide game area based on phase
  el('gameArea').classList.toggle('hidden', room.phase === 'waiting' || !room.phase);
  // owner controls visible to owner
  el('ownerControls').classList.toggle('hidden', !(actingPlayerId===room.owner));
}

/* refresh game UI */
function refreshGameUI(){
  const room = localRoomsCache[currentRoom]; if(!room) return;
  el('phaseLabel').innerText = room.phase || '—';
  el('exchangeRoundDisplay').innerText = room.exchangeRound || '-';
  el('exchangeIndexDisplay').innerText = ((room.exchangeIndex||0)+1) + '/' + ((room.turnOrder||[]).length||0);
  // toggle panels
  el('exchangePanel').classList.toggle('hidden', room.phase !== 'exchange');
  el('submitPanel').classList.toggle('hidden', room.phase !== 'submit');
  el('votePanel').classList.toggle('hidden', room.phase !== 'vote');
  el('resultPanel').classList.toggle('hidden', room.phase !== 'result');
  renderHandArea();
  renderSubmissions();
  if(room.phase === 'result'){
    const res = room.result || {};
    let html = '<div class="small">得票数</div>';
    const counts = res.counts || room.voteCounts || {};
    (room.turnOrder||[]).forEach(pid => { html += `<div>${escapeHtml((room.players.find(p=>p.id===pid)||{}).nick||pid)}: ${counts[pid]||0}</div>`; });
    html += `<div style="margin-top:8px"><strong>勝者: ${((res.winners||[]).map(w=>escapeHtml((room.players.find(p=>p.id===w)||{}).nick||w))).join(', ')}</strong></div>`;
    el('resultDisplay').innerHTML = html;
  }
}

/* ---------- Listeners wiring (buttons) ---------- */
el('btnCreate').onclick = createRoom;
el('btnJoinExisting').onclick = ()=> showView('joinView');
el('btnBackStart').onclick = ()=> { showView('startView'); };
el('btnJoin').onclick = joinRoom;
el('btnBackToStart').onclick = ()=> { currentRoom = null; actingPlayerId = null; showView('startView'); };
el('btnReady').onclick = toggleReady;
el('btnLeave').onclick = leaveRoom;
el('btnDissolve').onclick = dissolveRoom;
el('btnOpenAdmin').onclick = ()=> { showView('adminPanel'); renderAdminCards(); };
el('btnAdminLogin').onclick = async ()=> { // simple admin check via prompt
  const pass = prompt('管理者パスワードを入力してください (ローカルテスト用)'); if(pass === 'nanamiya333'){ showView('adminPanel'); renderAdminCards(); } else alert('パスワードが違います'); };
el('btnExchangeDo').onclick = async ()=> {
  // gather selected indices
  const nodes = Array.from(el('handArea').children).filter(n=>n.classList.contains('selected'));
  if(nodes.length===0){ alert('交換するカードを選択してください'); return; }
  const indices = nodes.map(n=>parseInt(n.dataset.index));
  try{ await doExchange(indices); } catch(e){ alert('交換失敗: '+e.message); }
};
el('btnExchangeOK').onclick = exchangeOK;
el('btnSubmit').onclick = submitWork;
el('btnOwnerAdvanceExchange').onclick = ownerAdvanceExchange;
el('btnOwnerStartSubmit').onclick = ownerStartSubmit;
el('btnOwnerStartVote').onclick = ownerStartVote;
el('btnOwnerFinishVote').onclick = ownerFinishVote;
el('btnNextGame').onclick = resetAfterGame;

/* Attach cards search input */
el('adminSearch').addEventListener('input', renderAdminCards);

/* ---------- Firestore snapshot caching ---------- */
/* Keep localRoomsCache updated when room snapshots arrive via attachRoomListener callback */
roomUnsub = null; // defined earlier

/* Attach room listener wrapper to update local cache and UI */
function attachRoomListenerWithCache(code){
  if(roomUnsub) roomUnsub();
  const ref = roomDocRef(code);
  roomUnsub = ref.onSnapshot(doc=> {
    if(!doc.exists){ delete localRoomsCache[code]; if(currentRoom===code){ showView('startView'); currentRoom=null; actingPlayerId=null; } return; }
    const room = doc.data();
    localRoomsCache[code] = room;
    // update UI if this is our current room
    if(currentRoom === code){
      refreshLobby(); refreshGameUI();
    }
  }, err => log('room snapshot error: '+err.message));
}

/* Overwrite attachRoomListener to use cache version */
attachRoomListener = attachRoomListenerWithCache;

/* ---------- Init after auth changes ---------- */
/* When signed in we fetch cards and start listening; handled in afterAuthInit() */

/* Provide cardsDocRef function used above */
function cardsDocRef(){ return db.doc('meta/cards'); }

/* ---------- Export of helper for debugging ---------- */
window._debug = {
  db, fetchCardsOnce, attachCardsListener, attachRoomListenerWithCache
};
</script>
</body>
</html>

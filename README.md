<!-- frontend.html (GitHub Pages). Single-page app: auth (email), register+verify, upload, support chat.
     Set API_URL to your backend (e.g., http://127.0.0.1:8000 or https://your-vps) -->
<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>ShareBox</title>
<style>
:root{
  --bg:#0b0f0f; --text:#e8f7ea; --muted:#9fb7a7; --red:#ff4d4d;
  --accent:#2aff77; --accent2:#0bd466;
  --auracolor:#0f351f; --glow:.5; --aura:.35;
  --radius:16px; --pblur:6; --pshadow:.5;
}
*{box-sizing:border-box} html,body{height:100%}
body{margin:0;background:var(--bg);color:var(--text);font:16px/1.45 Inter,Segoe UI,Roboto,Arial,sans-serif;overflow-x:hidden}

.wrap{max-width:940px;margin:0 auto;padding:24px}
.top{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}
.brand{font-weight:800}
.btn, .iconbtn{appearance:none;border:none;border-radius:10px;padding:10px 14px;cursor:pointer;transition:.15s}
.btn{background:linear-gradient(180deg,var(--accent),var(--accent2));color:#04240f;font-weight:700;box-shadow:0 8px 30px #00ff8055}
.iconbtn{background:#101714;border:1px solid #173b29;color:#bfe6cd}
.iconbtn:hover{filter:brightness(1.1)} .btn:hover{filter:brightness(1.05)}
.card{background:linear-gradient(180deg,#0d1412,#0a0f0e);border:1px solid #123326;border-radius:var(--radius);padding:18px;backdrop-filter:blur(calc(var(--pblur)*1px));filter:drop-shadow(0 30px calc(40px*var(--pshadow)) rgba(0,0,0,.6));margin-bottom:14px}

.grid{display:grid;grid-template-columns:1fr;gap:12px}
.row{display:flex;gap:12px;flex-wrap:wrap}
.col{flex:1 1 320px;min-width:280px}

.input{width:100%;background:#0d1413;border:1px solid #173b29;color:#e8f7ea;border-radius:10px;padding:10px}
.input-group{position:relative} .eye{position:absolute;right:8px;top:50%;transform:translateY(-50%);background:transparent;border:none;color:#bfe6cd;cursor:pointer}
.eye svg{width:20px;height:20px}

.link{color:var(--accent);text-decoration:none} .link:hover{text-decoration:underline}
.err{color:#ff6d6d;margin-top:8px;min-height:20px}
.muted{color:var(--muted)}
.hidden{display:none}

.drop{border:2px dashed #1b3b2e;border-radius:14px;padding:18px;text-align:center;background:#0e1513}
#file{display:none}
.filebtn{display:inline-block;border-radius:12px;padding:10px 14px;background:linear-gradient(180deg,var(--accent),var(--accent2));color:#04240f;font-weight:700;cursor:pointer;box-shadow:0 8px 30px #00ff8055}
.name{color:#9fb7a7;margin-top:8px;min-height:20px}
.progress{height:10px;background:#0c1412;border:1px solid #173b29;border-radius:999px;overflow:hidden;margin-top:10px;display:none}
.progress>div{height:100%;width:0%;background:linear-gradient(90deg,#0bd466,#2aff77);transition:width .15s}

.chat{display:flex;gap:12px}
.ticketlist{flex:0 0 240px;border-right:1px dashed #173b29;padding-right:10px}
.msgs{flex:1;border:1px solid #173b29;border-radius:12px;padding:10px;min-height:220px;background:#0f1614;overflow:auto}
.msg{margin:6px 0}.msg .from{font-size:12px;color:#86c7a0}.msg .bubble{display:inline-block;padding:8px 10px;background:#0c1412;border:1px solid #173b29;border-radius:10px;margin-top:2px}

.sets{display:flex;gap:12px;flex-wrap:wrap}
.sliderrow{display:flex;align-items:center;gap:10px}
.sliderrow input[type=range]{flex:1}
</style>
</head>
<body>
<div class="wrap">
  <div class="top">
    <div class="brand">ShareBox</div>
    <div>
      <button id="settingsBtn" class="iconbtn">Настройки</button>
      <button id="logoutBtn" class="iconbtn hidden">Выйти</button>
    </div>
  </div>

  <!-- AUTH -->
  <div id="authCard" class="card">
    <div class="row">
      <div class="col">
        <h3>Войти</h3>
        <div class="input-group" style="margin-top:6px">
          <input id="loginEmail" class="input" placeholder="Почта">
        </div>
        <div class="input-group" style="margin-top:6px">
          <input id="loginPass" class="input" type="password" placeholder="Пароль">
          <button class="eye" data-for="loginPass" aria-label="Показать/скрыть">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor"><path d="M3 3l18 18"/><path d="M10.58 10.58A2 2 0 0 0 12 14a2 2 0 0 0 1.42-3.42"/><path d="M21 12s-3.5-6-9-6-9 6-9 6 3.5 6 9 6 9-6 9-6z"/></svg>
          </button>
        </div>
        <button id="loginBtn" class="btn" style="margin-top:8px">Войти</button>
        <div class="err" id="loginErr"></div>
        <div class="muted" style="margin-top:8px">Нет аккаунта? <a id="toReg" class="link" href="#">Создай</a></div>
      </div>
      <div class="col" id="regCol">
        <h3>Регистрация</h3>
        <input id="regNick" class="input" placeholder="Никнейм" style="margin-top:6px">
        <input id="regEmail" class="input" placeholder="Почта" style="margin-top:6px">
        <div class="input-group" style="margin-top:6px">
          <input id="regPass" class="input" type="password" placeholder="Пароль">
          <button class="eye" data-for="regPass" aria-label="Показать/скрыть">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor"><path d="M3 3l18 18"/><path d="M10.58 10.58A2 2 0 0 0 12 14a2 2 0 0 0 1.42-3.42"/><path d="M21 12s-3.5-6-9-6-9 6-9 6 3.5 6 9 6 9-6 9-6z"/></svg>
          </button>
        </div>
        <div class="input-group" style="margin-top:6px">
          <input id="regPass2" class="input" type="password" placeholder="Подтверждение пароля">
          <button class="eye" data-for="regPass2" aria-label="Показать/скрыть">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor"><path d="M3 3l18 18"/><path d="M10.58 10.58A2 2 0 0 0 12 14a2 2 0 0 0 1.42-3.42"/><path d="M21 12s-3.5-6-9-6-9 6-9 6 3.5 6 9 6 9-6 9-6z"/></svg>
          </button>
        </div>
        <button id="regBtn" class="btn" style="margin-top:8px">Зарегистрироваться</button>
        <div class="err" id="regErr"></div>
        <div id="verifyBox" class="hidden" style="margin-top:10px">
          <div class="muted">Мы отправили код на почту. Введите:</div>
          <input id="vEmail" class="input" placeholder="Почта" style="margin-top:6px">
          <input id="vCode" class="input" placeholder="Код" style="margin-top:6px">
          <div class="row" style="margin-top:6px">
            <button id="verifyBtn" class="iconbtn">Подтвердить</button>
            <button id="resendBtn" class="iconbtn">Отправить код ещё раз</button>
          </div>
          <div class="err" id="verifyErr"></div>
        </div>
      </div>
    </div>
  </div>

  <!-- APP -->
  <div id="appCard" class="card hidden">
    <div class="row">
      <div class="col">
        <h3>Загрузка файла</h3>
        <div class="drop">
          <label class="filebtn"><input id="file" type="file">Выбрать файл</label>
          <div id="fname" class="name"></div>
          <div style="margin-top:8px">Срок (дней): <span id="daysVal">3</span></div>
          <input id="days" type="range" min="1" max="365" value="3"/>
          <div class="row" style="margin-top:8px"><button id="uploadBtn" class="btn">Загрузить</button><button id="copyBtn" class="iconbtn hidden">Скопировать ссылку</button></div>
          <div class="progress" id="progress"><div></div></div>
          <div class="err" id="upErr"></div>
          <div id="result" class="muted hidden" style="margin-top:6px">Ссылка: <a id="link" class="link" target="_blank" rel="noopener"></a> <span id="meta"></span></div>
        </div>
      </div>
      <div class="col">
        <h3>Тех. поддержка</h3>
        <div class="chat">
          <div class="ticketlist">
            <button id="newTicket" class="iconbtn">Новый тикет</button>
            <div id="tickets" style="margin-top:8px" class="muted"></div>
          </div>
          <div style="flex:1;display:flex;flex-direction:column">
            <div id="msgs" class="msgs"></div>
            <div class="row" style="margin-top:8px">
              <input id="msgText" class="input" placeholder="Сообщение...">
              <button id="sendMsg" class="iconbtn">Отправить</button>
            </div>
          </div>
        </div>
        <div class="err" id="chatErr"></div>
      </div>
    </div>
  </div>

  <!-- SETTINGS -->
  <div id="setCard" class="card hidden">
    <h3>Настройки</h3>
    <div class="sets">
      <div class="col">
        <div class="muted">Акцент</div>
        <input id="accentPick" type="color" class="input" value="#2aff77">
      </div>
      <div class="col">
        <div class="muted">Цвет ауры</div>
        <input id="auraPick" type="color" class="input" value="#0f351f">
      </div>
      <div class="col">
        <div class="sliderrow"><label style="min-width:130px">Интенсивность ауры</label><input id="aura" type="range" min="0" max="1" step="0.01" value=".35"></div>
        <div class="sliderrow" style="margin-top:6px"><label style="min-width:130px">Свет (glow)</label><input id="glow" type="range" min="0" max="1" step="0.01" value=".5"></div>
      </div>
      <div class="col">
        <div class="sliderrow"><label style="min-width:130px">Скругление</label><input id="rad" type="range" min="8" max="30" step="1" value="16"></div>
        <div class="sliderrow" style="margin-top:6px"><label style="min-width:130px">Размытие</label><input id="pblur" type="range" min="0" max="20" step="1" value="6"></div>
        <div class="sliderrow" style="margin-top:6px"><label style="min-width:130px">Тень</label><input id="pshadow" type="range" min="0" max="1" step="0.01" value=".5"></div>
      </div>
    </div>
  </div>
</div>

<script>
const API_URL = localStorage.getItem('api_url') || 'http://127.0.0.1:8000';
let TOKEN=null, CURRENT_TICKET=null, ES=null;

// UI helpers
const $ = (id)=> document.getElementById(id);
function show(el){ el.classList.remove('hidden') }
function hide(el){ el.classList.add('hidden') }

// Auth
$('toReg').onclick = (e)=>{ e.preventDefault(); $('regCol').scrollIntoView({behavior:'smooth'}) };
document.querySelectorAll('.eye').forEach(btn=>{
  btn.addEventListener('click', ()=>{
    const id = btn.getAttribute('data-for');
    const inp = $(id);
    const isPass = inp.getAttribute('type') === 'password';
    inp.setAttribute('type', isPass? 'text':'password');
    // toggle icon slash
    btn.innerHTML = isPass
      ? '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor"><path d="M1 12s4-7 11-7 11 7 11 7-4 7-11 7-11-7-11-7z"/><circle cx="12" cy="12" r="3"/></svg>'
      : '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor"><path d="M3 3l18 18"/><path d="M10.58 10.58A2 2 0 0 0 12 14a2 2 0 0 0 1.42-3.42"/><path d="M21 12s-3.5-6-9-6-9 6-9 6 3.5 6 9 6 9-6 9-6z"/></svg>';
  })
});
$('loginBtn').onclick = async ()=>{
  $('loginErr').textContent='';
  const email = $('loginEmail').value.trim(), password = $('loginPass').value;
  try{
    const r = await fetch(API_URL+'/api/auth/login',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({email,password})});
    const j = await r.json();
    if(!j.ok){ $('loginErr').textContent = j.error || 'Ошибка входа'; return; }
    TOKEN = j.token; localStorage.setItem('token', TOKEN);
    onLoggedIn();
  }catch(e){ $('loginErr').textContent='Сеть недоступна'; }
};
$('regBtn').onclick = async ()=>{
  $('regErr').textContent='';
  const nick=$('regNick').value.trim(), email=$('regEmail').value.trim(), password=$('regPass').value, confirm=$('regPass2').value;
  try{
    const r = await fetch(API_URL+'/api/auth/register',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({nick,email,password,confirm})});
    const j = await r.json();
    if(!j.ok){ $('regErr').textContent=j.error||'Ошибка регистрации'; return; }
    $('vEmail').value = email; show($('verifyBox'));
  }catch(e){ $('regErr').textContent='Сеть недоступна'; }
};
$('verifyBtn').onclick = async ()=>{
  $('verifyErr').textContent='';
  const email=$('vEmail').value.trim(), code=$('vCode').value.trim();
  try{
    const r = await fetch(API_URL+'/api/auth/verify',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({email,code})});
    const j = await r.json();
    if(!j.ok){ $('verifyErr').textContent=j.error||'Код неверен'; return; }
    alert('Почта подтверждена! Теперь войдите.');
  }catch(e){ $('verifyErr').textContent='Сеть недоступна'; }
};
$('resendBtn').onclick = async ()=>{
  const email=$('vEmail').value.trim();
  const r = await fetch(API_URL+'/api/auth/resend',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({email})});
  const j = await r.json();
  if(!j.ok) alert('Не удалось отправить код: '+(j.error||''));
};
$('logoutBtn').onclick = ()=>{ TOKEN=null; localStorage.removeItem('token'); onLoggedOut(); };

function onLoggedIn(){
  hide($('authCard')); show($('appCard')); show($('logoutBtn'));
  loadTickets();
}
function onLoggedOut(){
  show($('authCard')); hide($('appCard')); hide($('logoutBtn'));
  if(ES){ ES.close(); ES=null; }
}
window.addEventListener('load', async ()=>{
  TOKEN = localStorage.getItem('token');
  if(TOKEN){
    try{
      const r = await fetch(API_URL+'/api/user/me',{headers:{'Authorization':'Bearer '+TOKEN}});
      const j = await r.json(); if(j.ok){ onLoggedIn(); return; }
    }catch(e){}
    TOKEN=null; localStorage.removeItem('token');
  }
  onLoggedOut();
});

// Upload
const fileInput=$('file'), fname=$('fname'), days=$('days'), daysVal=$('daysVal'), progress=$('progress'), bar=progress.querySelector('div'), upErr=$('upErr'), result=$('result'), link=$('link'), meta=$('meta');
let THE_FILE=null; $('file').addEventListener('change', e=>{ const f=e.target.files[0]; THE_FILE=f; fname.textContent=f? `${f.name} (${(f.size/1024/1024).toFixed(2)} MB)` : '' });
days.addEventListener('input', ()=> daysVal.textContent=days.value);
$('uploadBtn').onclick = ()=>{
  upErr.textContent=''; result.classList.add('hidden');
  if(!THE_FILE){ upErr.textContent='Выбери файл'; return; }
  const form=new FormData(); form.append('file', THE_FILE); form.append('days', days.value);
  const xhr=new XMLHttpRequest(); xhr.open('POST', API_URL+'/api/upload', true); xhr.setRequestHeader('Authorization','Bearer '+TOKEN);
  xhr.upload.onprogress=(e)=>{ if(e.lengthComputable){ progress.style.display='block'; bar.style.width=Math.round(e.loaded/e.total*100)+'%'; } };
  xhr.onreadystatechange=()=>{ if(xhr.readyState===4){ progress.style.display='none'; bar.style.width='0%';
    try{ const j=JSON.parse(xhr.responseText); if(xhr.status===200 && j.ok){ link.textContent=j.link; link.href=j.link; meta.textContent=` | ${j.provider} | ${j.ttl}`; result.classList.remove('hidden'); $('copyBtn').classList.remove('hidden'); } else { upErr.textContent = j.error || ('Ошибка '+xhr.status) } }
    catch(e){ upErr.textContent='Ошибка сервера' }
  }};
  xhr.send(form);
};
$('copyBtn').onclick = ()=>{ if(link.href){ navigator.clipboard.writeText(link.href) } };

// Chat
async function loadTickets(){
  const r=await fetch(API_URL+'/api/support/tickets',{headers:{'Authorization':'Bearer '+TOKEN}});
  const j=await r.json(); const list=$('tickets'); list.innerHTML='';
  if(!j.ok){ list.textContent='Ошибка'; return; }
  j.items.forEach(it=>{ const d=document.createElement('div'); d.textContent=`${it.id} — ${it.created_at}`; d.style.cursor='pointer'; d.onclick=()=>openTicket(it.id); list.appendChild(d) });
}
$('newTicket').onclick = async ()=>{
  const r=await fetch(API_URL+'/api/support/ticket',{method:'POST',headers:{'Authorization':'Bearer '+TOKEN}});
  const j=await r.json(); if(j.ok){ await loadTickets(); openTicket(j.ticket_id); }
};
function openTicket(id){
  CURRENT_TICKET=id; $('msgs').innerHTML='';
  if(ES){ ES.close(); ES=null; }
  ES = new EventSource(`${API_URL}/api/support/stream?ticket_id=${encodeURIComponent(id)}`, { withCredentials:false });
  ES.onmessage = ev=>{ try{ const m=JSON.parse(ev.data); appendMsg(m.from,m.text) }catch(_){ } };
}
function appendMsg(from,text){ const d=document.createElement('div'); d.className='msg'; const f=document.createElement('div'); f.className='from'; f.textContent=from; const b=document.createElement('div'); b.className='bubble'; b.textContent=text; d.appendChild(f); d.appendChild(b); $('msgs').appendChild(d); $('msgs').scrollTop=$('msgs').scrollHeight }
$('sendMsg').onclick = async ()=>{
  const t=$('msgText').value.trim(); if(!t||!CURRENT_TICKET) return; $('msgText').value='';
  const r=await fetch(API_URL+'/api/support/send',{method:'POST',headers:{'Content-Type':'application/json','Authorization':'Bearer '+TOKEN},body:JSON.stringify({ticket_id:CURRENT_TICKET,text:t})});
  const j=await r.json(); if(!j.ok) alert('Не отправлено');
};
$('msgText').addEventListener('keydown', e=>{ if(e.key==='Enter' && !e.shiftKey){ e.preventDefault(); $('sendMsg').click(); } });

// Settings
const accentPick=$('accentPick'), auraPick=$('auraPick'), auraIn=$('aura'), glowIn=$('glow'), rad=$('rad'), pblur=$('pblur'), pshadow=$('pshadow');
$('settingsBtn').onclick = ()=>{ $('setCard').classList.toggle('hidden') }
function applyAccent(hex){
  document.documentElement.style.setProperty('--accent', hex);
  function darken(h,p){const c=parseInt(h.slice(1),16),r=(c>>16)&255,g=(c>>8)&255,b=c&255;const n=x=>Math.max(0,Math.min(255,Math.round(x*(1-p))));return '#'+(n(r)<<16|n(g)<<8|n(b)).toString(16).padStart(6,'0')}
  document.documentElement.style.setProperty('--accent2', darken(hex,.3));
}
function applyAura(){
  const rgb = (h=>{const n=parseInt(h.slice(1),16);return[(n>>16)&255,(n>>8)&255,n&255]})(auraPick.value);
  const a = (0.5*parseFloat(auraIn.value||'.35')).toFixed(3);
  document.body.style.background = `radial-gradient(1400px 900px at -15% -15%, rgba(${rgb[0]},${rgb[1]},${rgb[2]},${a}), transparent 70%), radial-gradient(1400px 900px at 115% 115%, rgba(${rgb[0]},${rgb[1]},${rgb[2]},${a}), transparent 70%), var(--bg)`;
}
function applyPanel(){ document.documentElement.style.setProperty('--radius', rad.value+'px'); document.documentElement.style.setProperty('--pblur', pblur.value); document.documentElement.style.setProperty('--pshadow', pshadow.value) }
function saveSets(){ localStorage.setItem('ui_settings', JSON.stringify({acc:accentPick.value,aura:auraPick.value,a: auraIn.value,g:glowIn.value,r:rad.value,b:pblur.value,s:pshadow.value})) }
function loadSets(){ try{ const s=JSON.parse(localStorage.getItem('ui_settings')||'{}'); if(s.acc){accentPick.value=s.acc;auraPick.value=s.aura;auraIn.value=s.a;glowIn.value=s.g;rad.value=s.r;pblur.value=s.b;pshadow.value=s.s} }catch(e){} applyAccent(accentPick.value); applyAura(); document.documentElement.style.setProperty('--glow',glowIn.value); applyPanel() }
[accentPick,auraPick,auraIn,glowIn,rad,pblur,pshadow].forEach(x=> x.addEventListener('input', ()=>{ applyAccent(accentPick.value); applyAura(); document.documentElement.style.setProperty('--glow',glowIn.value); applyPanel(); saveSets() }))
loadSets();
</script>
</body>
</html>

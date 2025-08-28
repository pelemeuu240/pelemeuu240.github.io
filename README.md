# app.py
# ShareBox: пользовательский сайт (127.0.0.1:8000) + сайт поддержки (192.168.0.106:8000)
# Требуется вход через Google перед использованием. Файлы загружаются на внешние хостинги (24/7 ссылки).
import os, time, json, uuid, threading, tempfile, queue
from datetime import datetime
from urllib.parse import quote

from flask import Flask, request, jsonify, Response, stream_with_context, redirect, session
import requests
from dotenv import load_dotenv
from requests_oauthlib import OAuth2Session

load_dotenv()
os.environ.setdefault("OAUTHLIB_INSECURE_TRANSPORT", "1")  # разрешаем http для локалки

# Хосты/порт
USER_HOST = "127.0.0.1"        # пользовательский сайт
SUPPORT_HOST = "192.168.0.106" # сайт техподдержки
PORT = 8000
MAX_DAYS = 365

# Google OAuth2
CLIENT_ID = os.getenv("GOOGLE_CLIENT_ID", "").strip()
CLIENT_SECRET = os.getenv("GOOGLE_CLIENT_SECRET", "").strip()
AUTHORIZATION_BASE_URL = "https://accounts.google.com/o/oauth2/v2/auth"
TOKEN_URL = "https://oauth2.googleapis.com/token"
SCOPES = ["openid", "email", "profile"]

# Flask session
SECRET = os.getenv("FLASK_SECRET", "")
if not SECRET:
    SECRET = uuid.uuid4().hex + uuid.uuid4().hex
app = Flask(__name__)
app.secret_key = SECRET

# Ограничение доступа в панель поддержки (по email). Если пусто — пускаем всех авторизованных.
SUPPORT_EMAILS = {e.strip().lower() for e in os.getenv("SUPPORT_EMAILS", "").split(",") if e.strip()}

# ----- Авторизация -----
def is_logged():
    return bool(session.get("user"))

def user_email():
    u = session.get("user") or {}
    return (u.get("email") or "").lower()

def require_support_access():
    if not is_logged():
        return False
    if not SUPPORT_EMAILS:
        return True
    return user_email() in SUPPORT_EMAILS

def oauth_client(redirect_uri):
    return OAuth2Session(CLIENT_ID, scope=SCOPES, redirect_uri=redirect_uri)

@app.route("/login")
def login():
    host = (request.host or "").split("#")[0]
    redirect_uri = f"http://{host}/oauth2/callback"
    google = oauth_client(redirect_uri)
    auth_url, state = google.authorization_url(
        AUTHORIZATION_BASE_URL,
        access_type="offline",
        prompt="consent"
    )
    session["oauth_state"] = state
    return redirect(auth_url)

@app.route("/oauth2/callback")
def oauth2_callback():
    host = (request.host or "").split("#")[0]
    redirect_uri = f"http://{host}/oauth2/callback"
    google = oauth_client(redirect_uri)
    token = google.fetch_token(
        TOKEN_URL,
        client_secret=CLIENT_SECRET,
        authorization_response=request.url
    )
    session["oauth_token"] = token
    r = google.get("https://www.googleapis.com/oauth2/v1/userinfo?alt=json")
    profile = r.json()
    session["user"] = {
        "email": profile.get("email"),
        "name": profile.get("name") or profile.get("email"),
        "picture": profile.get("picture")
    }
    return redirect("/")

@app.route("/logout")
def logout():
    session.clear()
    return redirect("/")

# ----- Внешние хостинги -----
def upload_litterbox(path, filename, days):
    # TTL: 1h / 12h / 24h / 72h (макс 72h)
    if days <= 0:  ttl = "1h"
    elif days <= 1: ttl = "24h"
    elif days <= 3: ttl = "72h"
    else:           ttl = "72h"
    url = "https://litterbox.catbox.moe/resources/internals/api.php"
    with open(path, "rb") as f:
        files = {"fileToUpload": (filename, f)}
        data = {"reqtype": "fileupload", "time": ttl}
        r = requests.post(url, files=files, data=data, timeout=None)
    r.raise_for_status()
    link = r.text.strip()
    if not link.startswith("http"):
        raise RuntimeError(f"litterbox error: {r.text[:200]}")
    return {"provider": f"litterbox ({ttl})", "link": link, "page": link, "ttl_note": f"до {ttl}"}

def upload_transfersh(path, filename):
    # 14 дней хранения
    url = f"https://transfer.sh/{quote(filename)}"
    with open(path, "rb") as f:
        r = requests.put(url, data=f, timeout=None, headers={"Content-Type":"application/octet-stream"})
    r.raise_for_status()
    link = r.text.strip()
    return {"provider": "transfer.sh (14d)", "link": link, "page": link, "ttl_note": "14 дней"}

def upload_gofile(path, filename):
    s = requests.Session()
    gs = s.get("https://api.gofile.io/getServer", timeout=30).json()
    if gs.get("status") != "ok":
        raise RuntimeError(f"getServer: {gs}")
    server = gs["data"]["server"]
    with open(path, "rb") as f:
        up = s.post(f"https://{server}.gofile.io/uploadFile", files={"file": (filename, f)}, data={}, timeout=None)
    j = up.json()
    if j.get("status") != "ok":
        raise RuntimeError(f"gofile upload: {j}")
    d = j["data"]
    code = d["code"]
    fname = d.get("fileName") or filename
    direct = f"https://{server}.gofile.io/download/{code}/{quote(fname)}"
    page = d["downloadPage"]
    return {"provider": "gofile", "link": direct, "page": page, "ttl_note": "без авто-истечения"}

def upload_pixeldrain(path, filename):
    with open(path, "rb") as f:
        r = requests.post("https://pixeldrain.com/api/file", files={"file": (filename, f)}, timeout=None)
    r.raise_for_status()
    j = r.json()
    fid = j.get("id")
    if not fid:
        raise RuntimeError(f"pixeldrain: {j}")
    page = f"https://pixeldrain.com/u/{fid}"
    direct = f"https://pixeldrain.com/api/file/{fid}?download"
    return {"provider": "pixeldrain", "link": direct, "page": page, "ttl_note": "без авто-истечения"}

def choose_and_upload(file_storage, days):
    name = file_storage.filename or "file"
    tmp_path = tempfile.NamedTemporaryFile(delete=False).name
    try:
        file_storage.save(tmp_path)
        if days <= 1:
            strategies = [upload_litterbox, upload_transfersh, upload_gofile, upload_pixeldrain]
        elif days <= 3:
            strategies = [upload_litterbox, upload_transfersh, upload_gofile, upload_pixeldrain]
        elif days <= 14:
            strategies = [upload_transfersh, upload_gofile, upload_pixeldrain]
        else:
            strategies = [upload_gofile, upload_pixeldrain, upload_transfersh]
        last_err = None
        for fn in strategies:
            try:
                return fn(tmp_path, name) if fn != upload_litterbox else fn(tmp_path, name, days)
            except Exception as e:
                last_err = e
                continue
        raise last_err or RuntimeError("all providers failed")
    finally:
        try: os.remove(tmp_path)
        except Exception: pass

# ----- Чат (без Telegram) -----
TICKETS = {}   # tid -> { owner_nick, created_at, subs:set(Queue), history:list }
TICK_LOCK = threading.Lock()
SESSIONS = {}  # токены для пользовательского чата (после ввода ника)
RESERVED_NICKS = {n.lower() for n in ["support","поддержка","admin","админ","moderator","модератор","system","система","pelemeuu"]}

def ticket_get(tid):
    with TICK_LOCK:
        return TICKETS.get(tid)

def ticket_create(owner_nick):
    tid = uuid.uuid4().hex[:8]
    created = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with TICK_LOCK:
        TICKETS[tid] = {"owner_nick": owner_nick, "created_at": created, "subs": set(), "history": []}
    return tid, created

def subscribe(tid):
    q = queue.Queue()
    with TICK_LOCK:
        meta = TICKETS.get(tid)
        if not meta: return None, None
        meta["subs"].add(q)
        history = list(meta["history"])
    return q, history

def unsubscribe(tid, q):
    with TICK_LOCK:
        meta = TICKETS.get(tid)
        if meta and q in meta["subs"]:
            meta["subs"].discard(q)

def broadcast(tid, msg_dict):
    with TICK_LOCK:
        meta = TICKETS.get(tid)
        if not meta: return
        meta["history"].append(msg_dict)
        if len(meta["history"]) > 200:
            meta["history"] = meta["history"][-200:]
        subs = list(meta["subs"])
    for q in subs:
        try: q.put_nowait(msg_dict)
        except Exception: pass

def sse_stream(ticket_id):
    sub, history = subscribe(ticket_id)
    if not sub:
        yield "event: error\ndata: no_ticket\n\n"; return
    for m in history:
        yield f"data: {json.dumps(m, ensure_ascii=False)}\n\n"
    yield "event: hello\ndata: ok\n\n"
    try:
        while True:
            try:
                msg = sub.get(timeout=25)
                yield f"data: {json.dumps(msg, ensure_ascii=False)}\n\n"
            except Exception:
                yield "event: ping\ndata: keep\n\n"
    finally:
        unsubscribe(ticket_id, sub)

# ----- API: поддержка -----
@app.route("/api/support/auth", methods=["POST"])
def sup_auth():
    if not is_logged():
        return jsonify(ok=False, error="auth_required"), 401
    data = request.get_json(force=True, silent=True) or {}
    nick = (data.get("nick") or "").strip()
    if not nick:
        return jsonify(ok=False, error="empty_nick"), 400
    if nick.lower() in RESERVED_NICKS:
        return jsonify(ok=False, error="reserved_nick"), 403
    if len(nick) > 24:
        return jsonify(ok=False, error="nick_too_long"), 400
    token = uuid.uuid4().hex
    SESSIONS[token] = nick
    return jsonify(ok=True, token=token, nick=nick)

@app.route("/api/support/tickets", methods=["GET"])
def sup_list():
    if not is_logged():
        return jsonify(ok=False, error="auth_required"), 401
    token = request.args.get("token","")
    if token not in SESSIONS:
        return jsonify(ok=False, error="unauthorized"), 401
    nick = SESSIONS[token]
    items = []
    with TICK_LOCK:
        for tid, meta in TICKETS.items():
            if meta.get("owner_nick") == nick:
                items.append({"id": tid, "created_at": meta["created_at"]})
    items.sort(key=lambda x:x["created_at"])
    return jsonify(ok=True, items=items)

@app.route("/api/support/ticket", methods=["POST"])
def sup_new():
    if not is_logged():
        return jsonify(ok=False, error="auth_required"), 401
    data = request.get_json(force=True, silent=True) or {}
    token = data.get("token","")
    if token not in SESSIONS:
        return jsonify(ok=False, error="unauthorized"), 401
    nick = SESSIONS[token]
    tid, created = ticket_create(nick)
    return jsonify(ok=True, ticket_id=tid, created_at=created)

@app.route("/api/support/send", methods=["POST"])
def sup_send():
    if not is_logged():
        return jsonify(ok=False, error="auth_required"), 401
    data = request.get_json(force=True, silent=True) or {}
    token = data.get("token","")
    tid   = data.get("ticket_id","")
    text  = (data.get("text") or "").strip()
    if token not in SESSIONS: return jsonify(ok=False, error="unauthorized"), 401
    if not tid or not ticket_get(tid): return jsonify(ok=False, error="no_ticket"), 400
    if not text: return jsonify(ok=False, error="empty"), 400
    nick = SESSIONS[token]
    broadcast(tid, {"from": nick, "text": text, "ts": int(time.time())})
    return jsonify(ok=True)

@app.route("/api/support/stream")
def sup_stream():
    if not is_logged():
        return "auth_required", 401
    token = request.args.get("token","")
    tid   = request.args.get("ticket_id","")
    if token not in SESSIONS: return "unauthorized", 401
    if not tid or not ticket_get(tid): return "no ticket", 400
    return Response(stream_with_context(sse_stream(tid)), mimetype="text/event-stream")

# --- Панель оператора ---
@app.route("/api/support/admin/tickets", methods=["GET"])
def admin_tickets():
    if not is_logged() or not require_support_access():
        return jsonify(ok=False, error="forbidden"), 403
    items = []
    with TICK_LOCK:
        for tid, meta in TICKETS.items():
            items.append({"id": tid, "created_at": meta["created_at"], "owner_nick": meta["owner_nick"]})
    items.sort(key=lambda x:x["created_at"])
    return jsonify(ok=True, items=items)

@app.route("/api/support/admin/stream")
def admin_stream():
    if not is_logged() or not require_support_access():
        return "forbidden", 403
    tid = request.args.get("ticket_id","")
    if not tid or not ticket_get(tid): return "no ticket", 400
    return Response(stream_with_context(sse_stream(tid)), mimetype="text/event-stream")

@app.route("/api/support/admin/send", methods=["POST"])
def admin_send():
    if not is_logged() or not require_support_access():
        return jsonify(ok=False, error="forbidden"), 403
    data = request.get_json(force=True, silent=True) or {}
    tid  = data.get("ticket_id","")
    text = (data.get("text") or "").strip()
    if not tid or not ticket_get(tid): return jsonify(ok=False, error="no_ticket"), 400
    if not text: return jsonify(ok=False, error="empty"), 400
    broadcast(tid, {"from": "Support", "text": text, "ts": int(time.time())})
    return jsonify(ok=True)

# ----- API: загрузка файла -----
@app.route("/api/upload", methods=["POST"])
def api_upload():
    if not is_logged():
        return jsonify(ok=False, error="auth_required"), 401
    if "file" not in request.files:
        return jsonify(ok=False, error="no_file"), 400
    f = request.files["file"]
    if not f or not f.filename:
        return jsonify(ok=False, error="empty_filename"), 400
    try:
        days = int(request.form.get("days", "3"))
    except:
        days = 3
    days = max(1, min(days, MAX_DAYS))
    try:
        res = choose_and_upload(f, days)
        return jsonify(ok=True, link=res["link"], page=res["page"], provider=res["provider"], ttl=res["ttl_note"], days=days)
    except Exception as e:
        return jsonify(ok=False, error=str(e)), 500

# ----- Рендер страниц по Host -----
@app.route("/", methods=["GET"])
def entry():
    host = (request.host or "").split(":")[0]
    if not is_logged() or not (CLIENT_ID and CLIENT_SECRET):
        return Response(LOGIN_HTML.replace("{{NEEDCFG}}", "" if (CLIENT_ID and CLIENT_SECRET) else "true"), mimetype="text/html")
    if host == SUPPORT_HOST:
        if not require_support_access():
            return Response(SUPPORT_DENIED_HTML, mimetype="text/html", status=403)
        return Response(ADMIN_HTML, mimetype="text/html")
    return Response(USER_HTML, mimetype="text/html")

# ----- HTML: пользовательский -----
USER_HTML = r"""<!doctype html>
<html lang="ru"><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>ShareBox — загрузка</title>
<style>
:root{
  --bg:#0b0f0f; --text:#e8f7ea; --muted:#9fb7a7; --red:#ff4d4d;
  --accent:#2aff77; --accent2:#0bd466;
  --auracolor:#0f351f; --glow:0.5; --aura:0.35;
  --radius:16px; --pblur:6; --pshadow:0.5;
}
*{box-sizing:border-box} html,body{height:100%}
body{margin:0;background:var(--bg);color:var(--text);font:16px/1.45 Inter,Segoe UI,Roboto,Arial,sans-serif;overflow-x:hidden}
.topbar{ position:fixed; right:14px; top:10px; display:flex; gap:10px; z-index:20 }
.iconbtn{ background:#101714; border:1px solid #173b29; color:#bfe6cd; border-radius:10px; padding:8px 12px; cursor:pointer; transition:.15s } .iconbtn:hover{ filter:brightness(1.1) } .iconbtn:active{ transform:translateY(1px) }
.bg{ position:fixed; inset:0; z-index:-1; overflow:hidden }
.bg-circles{ position:absolute; inset:0; }
.bg-circles .circle{ position:absolute; border-radius:50%; filter:blur(16px) drop-shadow(0 0 calc(40px*var(--glow)) rgba(42,255,119,0.35)); opacity:.25; background:radial-gradient(circle at 30% 30%, var(--accent), #0a5a2f) }
.bg-circles.animated .circle{ animation-duration:18s; animation-iteration-count:infinite; animation-timing-function:ease-in-out }
.bg-circles .c1{ width:380px; height:380px; left:5%; top:10% }
.bg-circles .c2{ width:260px; height:260px; right:10%; top:20%; animation-delay:-3s }
.bg-circles .c3{ width:420px; height:420px; left:15%; bottom:-10%; animation-delay:-7s }
@keyframes float{ 0%{transform:translate(0,0) scale(1)} 50%{transform:translate(25px,-30px) scale(1.05)} 100%{transform:translate(0,0) scale(1)} }
@keyframes drift{ 0%{transform:translate(-15px,0)} 50%{transform:translate(15px,-20px)} 100%{transform:translate(-15px,0)} }
@keyframes pulse{ 0%{transform:scale(1)} 50%{transform:scale(1.08)} 100%{transform:scale(1)} }
@keyframes orbit{ 0%{transform:translate(0,0) rotate(0)} 100%{transform:translate(0,0) rotate(360deg)} }
@keyframes swirl{ 0%{transform:rotate(0) scale(1)} 50%{transform:rotate(180deg) scale(1.03)} 100%{transform:rotate(360deg) scale(1)} }
.bg-circles.float  .circle{ animation-name:float }
.bg-circles.drift  .circle{ animation-name:drift }
.bg-circles.pulse  .circle{ animation-name:pulse }
.bg-circles.orbit  .circle{ animation-name:orbit; transform-origin:center }
.bg-circles.swirl  .circle{ animation-name:swirl }

.fx-canvas{ position:fixed; inset:0; z-index:-2; pointer-events:none }

.container{ max-width:920px; margin:0 auto; padding:70px 20px 46px; transition:.2s opacity,.2s transform }
.card{ background:linear-gradient(180deg,#0d1412,#0a0f0e); border:1px solid #123326; border-radius:var(--radius); padding:28px; backdrop-filter: blur(calc(var(--pblur)*1px)); filter: drop-shadow(0 30px calc(40px*var(--pshadow)) rgba(0,0,0,.6)) }
h1{margin:0 0 10px} .sub{color:var(--muted); margin-bottom:22px}

.row{ display:flex; gap:16px; flex-wrap:wrap } .box{ flex:1 1 360px; min-width:300px }
.drop{ border:2px dashed #1b3b2e; border-radius:14px; padding:28px; text-align:center; background:#0e1513; transition:.2s border-color,.2s transform,.2s background; min-height:210px; display:flex; flex-direction:column; justify-content:center }
.drop.drag{ border-color:var(--accent); transform:scale(1.01); background:#0e1715 }
#file{ display:none }
label.filebtn{ display:inline-block; border-radius:12px; padding:12px 16px; background:linear-gradient(180deg,var(--accent),var(--accent2)); color:#04240f; font-weight:700; cursor:pointer; box-shadow:0 8px 30px #00ff8055 }
.name{ color:var(--muted); margin-top:10px; word-break:break-word; min-height:22px }

.range{ background:#0e1513; border:1px solid #163626; border-radius:14px; padding:18px }
.range label{ display:block; margin-bottom:10px; color:#cfeedd } .range input[type=range]{ width:100% }
.days-val{ font-weight:700; color:var(--accent) }

.btn{ appearance:none; border:none; border-radius:12px; padding:12px 16px; background:linear-gradient(180deg,var(--accent),var(--accent2)); color:#04240f; font-weight:700; transition:transform .1s, filter .2s; cursor:pointer; box-shadow:0 8px 30px #00ff8055; position:relative; overflow:hidden }
.btn:hover{ filter:brightness(1.05) } .btn:active{ transform:translateY(1px) scale(.99) }
.btn.secondary{ background:transparent; color:#fff; border:1px solid #1e3c2c; box-shadow:none }

.progress{ height:10px; background:#0c1412; border:1px solid #173b29; border-radius:999px; overflow:hidden; margin-top:14px; display:none }
.progress>div{ height:100%; width:0%; background:linear-gradient(90deg,#0bd466,#2aff77); transition:width .15s }

.result{ margin-top:18px; padding:14px; background:#0d1413; border:1px solid #193a2b; border-radius:12px; display:none }
a.link{ color:var(--accent); text-decoration:none; font-weight:600; display:inline-block } a.link:hover{text-decoration:underline}
.meta{ color:#9cc3a9; margin-top:6px; font-size:14px } .err{ color:#ff4d4d; margin-top:10px; min-height:22px }

.brand{ position:fixed; right:14px; bottom:10px; color:#78a68c; font-size:13px; opacity:.85; user-select:none }

.overlay{ position:fixed; inset:0; display:none; align-items:center; justify-content:center; z-index:30 }
.overlay.show{ display:flex } .overlay .back{ position:absolute; inset:0; background:#0007; opacity:0; animation:fadeIn .2s forwards }
.panel{ width:min(920px,calc(100% - 40px)); background:linear-gradient(180deg,#0d1412,#0a0f0e); border:1px solid #123326; border-radius:var(--radius); padding:22px; backdrop-filter:blur(calc(var(--pblur)*1px)); filter: drop-shadow(0 30px calc(40px*var(--pshadow)) rgba(0,0,0,.6)); transform:translateY(8px) scale(.98); opacity:0; animation:popIn .22s forwards }
@keyframes fadeIn{ to{opacity:1} } @keyframes popIn{ to{transform:none; opacity:1} }
.hide .back{ animation:fadeOut .18s forwards } .hide .panel{ animation:popOut .18s forwards }
@keyframes fadeOut{ to{opacity:0} } @keyframes popOut{ to{ transform:translateY(8px) scale(.98); opacity:0 } }

.set-row{ display:flex; gap:16px; flex-wrap:wrap } .set-box{ flex:1 1 260px; min-width:240px; background:#0e1513; border:1px solid #163626; border-radius:14px; padding:16px }
.set-title{ font-weight:700; margin-bottom:10px } .input{ width:100%; background:#0d1413; border:1px solid #173b29; color:#e8f7ea; border-radius:10px; padding:8px 10px }
.switch{ display:flex; align-items:center; gap:8px } .radio{ display:flex; gap:8px; flex-wrap:wrap }
.radio label{ padding:6px 10px; border:1px solid #173b29; border-radius:999px; cursor:pointer } .radio input{ display:none }
.radio input:checked + span{ color:#04240f; background:linear-gradient(180deg,var(--accent),var(--accent2)); border-color:transparent }
.sliderrow{ display:flex; align-items:center; gap:10px } .sliderrow input[type=range]{ flex:1 }

.chat-box{ height:420px; display:flex; gap:12px }
.chat-left{ flex:0 0 220px; border-right:1px dashed #173b29; padding-right:12px }
.chat-right{ flex:1; display:flex; flex-direction:column }
.chat-list{ font-size:14px; color:#9fb7a7 }
.chat-msgs{ flex:1; border:1px solid #173b29; border-radius:12px; padding:10px; overflow:auto; background:#0f1614 }
.chat-input{ display:flex; gap:8px; margin-top:10px } .chat-input input{ flex:1 }
.msg{ margin:6px 0 } .msg .from{ color:#86c7a0; font-size:12px } .msg .bubble{ display:inline-block; padding:8px 10px; background:#0c1412; border:1px solid #173b29; border-radius:10px; margin-top:2px }
</style>
</head>
<body>
<div class="topbar">
  <a class="iconbtn" href="/logout">Выйти</a>
  <button id="supportBtn" class="iconbtn">Тех. поддержка</button>
  <button id="settingsBtn" class="iconbtn">Настройки</button>
</div>

<div class="bg" id="bg">
  <div id="bgCircles" class="bg-circles animated float">
    <div class="circle c1"></div><div class="circle c2"></div><div class="circle c3"></div>
  </div>
  <canvas id="fxCanvas" class="fx-canvas" style="display:none"></canvas>
</div>

<div class="container" id="mainView">
  <div class="card">
    <h1>ShareBox</h1>
    <div class="sub">Загрузи файл и выбери срок жизни ссылки. Ссылки живут на внешнем хостинге 24/7.</div>
    <div class="row">
      <div class="box">
        <div id="drop" class="drop">
          <div style="font-size:18px; font-weight:700">Перетащи файл сюда</div>
          <div class="muted">или</div>
          <label class="filebtn"><input id="file" type="file">Выбрать файл</label>
          <div id="fname" class="name"></div>
          <div class="progress" id="progress"><div></div></div>
          <div class="err" id="err"></div>
        </div>
      </div>
      <div class="box">
        <div class="range">
          <label>Срок действия: <span class="days-val" id="daysVal">3</span> дней</label>
          <input type="range" id="days" min="1" max="365" step="1" value="3">
          <div style="display:flex; gap:10px; margin-top:14px">
            <button class="btn" id="upload">Загрузить</button>
            <button class="btn secondary" id="copy" style="display:none">Скопировать ссылку</button>
          </div>
          <div class="result" id="result">
            <div>Ссылка:</div>
            <a id="link" class="link" href="#" target="_blank" rel="noopener"></a>
            <div class="meta" id="meta"></div>
          </div>
        </div>
      </div>
    </div>
    <div class="sub">1–3 дня — Litterbox, 14 дней — transfer.sh, 15+ дней — gofile.</div>
  </div>
</div>

<div class="brand">by PelemeuU</div>

<!-- Настройки -->
<div id="settingsOverlay" class="overlay">
  <div class="back"></div>
  <div class="panel">
    <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:10px">
      <div style="font-weight:700; font-size:18px">Настройки</div>
      <button id="closeSettings" class="iconbtn">Закрыть</button>
    </div>
    <div class="set-row">
      <div class="set-box">
        <div class="set-title">Акцентный цвет</div>
        <input id="accentPick" type="color" class="input" value="#2aff77">
      </div>
      <div class="set-box">
        <div class="set-title">Цвет ауры (фон)</div>
        <input id="auraPick" type="color" class="input" value="#0f351f">
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:90px">Интенсивность ауры</label><input id="aura" type="range" min="0" max="1" step="0.01" value="0.35"></div>
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:90px">Свет</label><input id="glow" type="range" min="0" max="1" step="0.01" value="0.5"></div>
      </div>
      <div class="set-box">
        <div class="set-title">Фон</div>
        <label class="switch"><input id="animToggle" type="checkbox" checked> Анимация</label>
        <div class="radio" style="margin-top:8px">
          <label><input type="radio" name="bgtype" value="circles" checked><span>Круги</span></label>
          <label><input type="radio" name="bgtype" value="snow"><span>Снег</span></label>
          <label><input type="radio" name="bgtype" value="leaves"><span>Листья</span></label>
        </div>
        <div class="set-title" style="margin-top:10px">Вид анимации кругов</div>
        <div class="radio" id="circleStyles">
          <label><input type="radio" name="cstyle" value="float" checked><span>Float</span></label>
          <label><input type="radio" name="cstyle" value="drift"><span>Drift</span></label>
          <label><input type="radio" name="cstyle" value="pulse"><span>Pulse</span></label>
          <label><input type="radio" name="cstyle" value="orbit"><span>Orbit</span></label>
          <label><input type="radio" name="cstyle" value="swirl"><span>Swirl</span></label>
        </div>
      </div>
      <div class="set-box">
        <div class="set-title">Панель</div>
        <div class="sliderrow"><label style="min-width:140px">Скругление</label><input id="rad" type="range" min="8" max="30" step="1" value="16"></div>
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:140px">Размытие (px)</label><input id="pblur" type="range" min="0" max="20" step="1" value="6"></div>
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:140px">Тень</label><input id="pshadow" type="range" min="0" max="1" step="0.01" value="0.5"></div>
      </div>
    </div>
  </div>
</div>

<!-- Техподдержка -->
<div id="supportOverlay" class="overlay">
  <div class="back"></div>
  <div class="panel">
    <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:10px">
      <div style="font-weight:700; font-size:18px">Тех. поддержка</div>
      <button id="closeSupport" class="iconbtn">Закрыть</button>
    </div>
    <div id="loginBox">
      <div class="set-row">
        <div class="set-box">
          <div class="set-title">Ваш ник</div>
          <input id="nick" class="input" placeholder="Введите ник">
          <button id="loginBtn" class="btn" style="margin-top:10px">Продолжить</button>
          <div id="loginErr" class="err"></div>
          <div class="sub" style="margin-top:8px">Запрещено: Support, Поддержка, Admin, Админ, Moderator, Модератор, System, Система, PelemeuU.</div>
        </div>
      </div>
    </div>
    <div id="chatBox" style="display:none">
      <div class="chat-box">
        <div class="chat-left">
          <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:8px">
            <div style="font-weight:700">Тикеты</div>
            <button id="newTicket" class="iconbtn">Новый</button>
          </div>
          <div id="ticketList" class="chat-list"></div>
        </div>
        <div class="chat-right">
          <div id="msgs" class="chat-msgs"></div>
          <div class="chat-input">
            <input id="msgText" class="input" placeholder="Написать сообщение...">
            <button id="sendMsg" class="btn">Отправить</button>
          </div>
          <div id="chatHint" class="sub" style="margin-top:6px"></div>
        </div>
      </div>
    </div>
  </div>
</div>

<script>
const bg = document.getElementById('bg'), bgCircles = document.getElementById('bgCircles'), fxCanvas = document.getElementById('fxCanvas');
const settingsBtn=document.getElementById('settingsBtn'), settingsOverlay=document.getElementById('settingsOverlay'), closeSettings=document.getElementById('closeSettings');
const supportBtn=document.getElementById('supportBtn'), supportOverlay=document.getElementById('supportOverlay'), closeSupport=document.getElementById('closeSupport');
const container=document.getElementById('mainView');

function showOverlay(el){ el.classList.remove('hide'); el.classList.add('show'); el.style.display='flex'; container.style.opacity=.2; container.style.transform='scale(.995)' }
function hideOverlay(el){ el.classList.add('hide'); setTimeout(()=>{ el.classList.remove('show'); el.classList.remove('hide'); el.style.display='none'; container.style.opacity=''; container.style.transform=''; },180) }
settingsBtn.onclick=()=>showOverlay(settingsOverlay); closeSettings.onclick=()=>hideOverlay(settingsOverlay);
supportBtn.onclick=()=>showOverlay(supportOverlay); closeSupport.onclick=()=>hideOverlay(supportOverlay);

// Загрузка
const fileInput=document.getElementById('file'), drop=document.getElementById('drop'), fname=document.getElementById('fname'), days=document.getElementById('days'), daysVal=document.getElementById('daysVal');
const uploadBtn=document.getElementById('upload'), progress=document.getElementById('progress'), bar=progress.querySelector('div'), result=document.getElementById('result'), link=document.getElementById('link'), copyBtn=document.getElementById('copy'), err=document.getElementById('err');
let theFile=null; function setFile(f){ theFile=f; fname.textContent=f? f.name+' ('+(f.size/1024/1024).toFixed(2)+' MB)':'' }
document.querySelector('label.filebtn').onclick=()=> fileInput.click();
fileInput.addEventListener('change', e=> setFile(e.target.files[0]));
['dragenter','dragover'].forEach(ev=> drop.addEventListener(ev, e=>{e.preventDefault();e.stopPropagation();drop.classList.add('drag')}))
;['dragleave','drop'].forEach(ev=> drop.addEventListener(ev, e=>{e.preventDefault();e.stopPropagation();drop.classList.remove('drag')}))
drop.addEventListener('drop', e=>{ const f=e.dataTransfer.files[0]; if(f) setFile(f) });
days.addEventListener('input', ()=> daysVal.textContent=days.value);
uploadBtn.addEventListener('click', ()=>{
  err.textContent=''; result.style.display='none'; copyBtn.style.display='none';
  if(!theFile){ err.textContent='Выбери файл'; return; }
  const form=new FormData(); form.append('file', theFile); form.append('days', days.value);
  const xhr=new XMLHttpRequest(); xhr.open('POST','/api/upload',true);
  xhr.upload.onprogress=(e)=>{ if(e.lengthComputable){ progress.style.display='block'; bar.style.width=Math.round(e.loaded/e.total*100)+'%' } };
  xhr.onreadystatechange=()=>{ if(xhr.readyState===4){ progress.style.display='none'; bar.style.width='0%';
    if(xhr.status===200){ const j=JSON.parse(xhr.responseText); if(j.ok){ link.textContent=j.link; link.href=j.link; document.getElementById('meta').textContent=`Хост: ${j.provider}, срок: ${j.ttl}`; result.style.display='block'; copyBtn.style.display='inline-block'; navigator.clipboard && navigator.clipboard.writeText(j.link).catch(()=>{}) } else { err.textContent='Ошибка: '+(j.error||'unknown') } }
    else { try{ const j=JSON.parse(xhr.responseText); err.textContent='Ошибка ('+xhr.status+'): '+(j.error||'server') } catch(e){ err.textContent='Ошибка сервера ('+xhr.status+')' } } } };
  xhr.send(form);
});
copyBtn.addEventListener('click', ()=>{ if(link.href){ navigator.clipboard.writeText(link.href).then(()=>{ copyBtn.textContent='Скопировано!'; setTimeout(()=>copyBtn.textContent='Скопировать ссылку',1200) }) } });

// Настройки
const accentPick=document.getElementById('accentPick'), auraPick=document.getElementById('auraPick'), glowIn=document.getElementById('glow'), auraIn=document.getElementById('aura');
const animToggle=document.getElementById('animToggle'); const radios=[...document.querySelectorAll('input[name=bgtype]')]; const cstyleRadios=[...document.querySelectorAll('input[name=cstyle]')];
const rad=document.getElementById('rad'), pblur=document.getElementById('pblur'), pshadow=document.getElementById('pshadow');
let FX={ type:'circles', anim:true, accent:'#2aff77', auracolor:'#0f351f', glow:0.5, aura:0.35, cstyle:'float', radius:16, panel_blur:6, panel_shadow:0.5 };

function applyAccent(hex){
  document.documentElement.style.setProperty('--accent', hex);
  function darken(h,p){const c=parseInt(h.slice(1),16),r=(c>>16)&255,g=(c>>8)&255,b=c&255;const n=x=>Math.max(0,Math.min(255,Math.round(x*(1-p))));return '#'+(n(r)<<16|n(g)<<8|n(b)).toString(16).padStart(6,'0')}
  document.documentElement.style.setProperty('--accent2', darken(hex,.3));
}
function applyAuraBg(){
  document.documentElement.style.setProperty('--auracolor', FX.auracolor);
  const rgb = (h=>{const n=parseInt(h.slice(1),16);return[(n>>16)&255,(n>>8)&255,n&255]})(FX.auracolor);
  const a = (0.5*FX.aura).toFixed(3);
  bg.style.background = `radial-gradient(1400px 900px at -15% -15%, rgba(${rgb[0]},${rgb[1]},${rgb[2]},${a}), transparent 70%), radial-gradient(1400px 900px at 115% 115%, rgba(${rgb[0]},${rgb[1]},${rgb[2]},${a}), transparent 70%)`;
}
function saveSettings(){ localStorage.setItem('sharebox_settings', JSON.stringify(FX)) }
function loadSettings(){
  try{ Object.assign(FX, JSON.parse(localStorage.getItem('sharebox_settings')||'{}')); }catch(e){}
  accentPick.value=FX.accent; auraPick.value=FX.auracolor; glowIn.value=FX.glow; auraIn.value=FX.aura; animToggle.checked=FX.anim;
  (radios.find(x=>x.value===FX.type)||radios[0]).checked=true; (cstyleRadios.find(x=>x.value===FX.cstyle)||cstyleRadios[0]).checked=true;
  rad.value=FX.radius; pblur.value=FX.panel_blur; pshadow.value=FX.panel_shadow;
  applyAccent(FX.accent); applyAuraBg();
  document.documentElement.style.setProperty('--glow', FX.glow); document.documentElement.style.setProperty('--aura', FX.aura);
  document.documentElement.style.setProperty('--radius', FX.radius+'px'); document.documentElement.style.setProperty('--pblur', FX.panel_blur); document.documentElement.style.setProperty('--pshadow', FX.panel_shadow);
  switchBG(FX.type, FX.anim, FX.cstyle);
}
accentPick.oninput=()=>{ FX.accent=accentPick.value; applyAccent(FX.accent); saveSettings() }
auraPick.oninput=()=>{ FX.auracolor=auraPick.value; applyAuraBg(); saveSettings() }
glowIn.oninput=()=>{ FX.glow=parseFloat(glowIn.value); document.documentElement.style.setProperty('--glow', FX.glow); saveSettings() }
auraIn.oninput=()=>{ FX.aura=parseFloat(auraIn.value); document.documentElement.style.setProperty('--aura', FX.aura); applyAuraBg(); saveSettings() }
animToggle.onchange=()=>{ FX.anim=animToggle.checked; switchBG(FX.type, FX.anim, FX.cstyle); saveSettings() }
radios.forEach(r=> r.onchange=()=>{ FX.type=r.value; switchBG(FX.type, FX.anim, FX.cstyle); saveSettings() })
cstyleRadios.forEach(r=> r.onchange=()=>{ FX.cstyle=r.value; switchBG(FX.type, FX.anim, FX.cstyle); saveSettings() })
rad.oninput=()=>{ FX.radius=parseInt(rad.value); document.documentElement.style.setProperty('--radius', FX.radius+'px'); saveSettings() }
pblur.oninput=()=>{ FX.panel_blur=parseInt(pblur.value); document.documentElement.style.setProperty('--pblur', FX.panel_blur); saveSettings() }
pshadow.oninput=()=>{ FX.panel_shadow=parseFloat(pshadow.value); document.documentElement.style.setProperty('--pshadow', FX.panel_shadow); saveSettings() }

let FX_LOOP=null;
function stopFX(){ if(FX_LOOP){ cancelAnimationFrame(FX_LOOP); FX_LOOP=null } }
function switchBG(type,anim,cstyle){
  const c=fxCanvas, ctx=c.getContext('2d');
  function resize(){ c.width=innerWidth; c.height=innerHeight }
  bgCircles.style.display=(type==='circles')?'':'none'; fxCanvas.style.display=(type==='circles')?'none':'';
  bgCircles.classList.toggle('animated', anim); ['float','drift','pulse','orbit','swirl'].forEach(s=> bgCircles.classList.remove(s)); bgCircles.classList.add(cstyle||'float');
  if(type==='circles'){ stopFX(); return; }
  resize(); onresize=resize;
  const N=(type==='snow')?120:70;
  const acc=getComputedStyle(document.documentElement).getPropertyValue('--accent').trim()||'#2aff77';
  const [lr,lg,lb]=((h)=>{const n=parseInt(h.slice(1),16);return[(n>>16)&255,(n>>8)&255,n&255]})(acc);
  const flakes=[], leaves=[];
  for(let i=0;i<N;i++){
    if(type==='snow') flakes.push({x:Math.random()*c.width,y:Math.random()*c.height,r:1+Math.random()*2.5,s:0.5+Math.random()*1.5,a:Math.random()*Math.PI*2});
    else leaves.push({x:Math.random()*c.width,y:Math.random()*c.height,s:8+Math.random()*14,v:0.4+Math.random()*0.9,a:Math.random()*Math.PI*2,av:(Math.random()*0.01)+0.005});
  }
  function drawLeaf(x,y,s,rot){ ctx.save();ctx.translate(x,y);ctx.rotate(rot); const al=0.18+0.35*parseFloat(getComputedStyle(document.documentElement).getPropertyValue('--glow')||'0.5'); ctx.fillStyle=`rgba(${lr},${lg},${lb},${al})`; ctx.beginPath(); ctx.ellipse(0,0,s*0.6,s,0,0,Math.PI*2); ctx.fill(); ctx.strokeStyle=`rgba(${lr},${lg},${lb},${Math.min(1,al*1.6)})`; ctx.lineWidth=1; ctx.beginPath(); ctx.moveTo(-s*0.5,0); ctx.lineTo(s*0.5,0); ctx.stroke(); ctx.restore(); }
  function tick(){ ctx.clearRect(0,0,c.width,c.height);
    if(type==='snow'){ const w=0.7+0.8*parseFloat(getComputedStyle(document.documentElement).getPropertyValue('--glow')||'0.5'); ctx.fillStyle=`rgba(255,255,255,${w})`; flakes.forEach(f=>{ if(anim){ f.y+=f.s; f.x+=Math.sin(f.a+=0.01)*0.3; if(f.y>c.height+5){ f.y=-5; f.x=Math.random()*c.width; } } ctx.beginPath(); ctx.arc(f.x,f.y,f.r,0,Math.PI*2); ctx.fill(); }); }
    else { leaves.forEach(l=>{ if(anim){ l.y+=l.v; l.x+=Math.sin(l.a+=l.av)*0.6; if(l.y>c.height+10){ l.y=-10; l.x=Math.random()*c.width; } } drawLeaf(l.x,l.y,l.s,l.a); }); }
    if(anim){ FX_LOOP=requestAnimationFrame(tick) }
  }
  stopFX(); tick(); if(anim && !FX_LOOP) FX_LOOP=requestAnimationFrame(tick);
}
loadSettings();

// Поддержка (user)
const loginBox=document.getElementById('loginBox'), chatBox=document.getElementById('chatBox'), loginBtn=document.getElementById('loginBtn'), loginErr=document.getElementById('loginErr'), nickIn=document.getElementById('nick');
const ticketList=document.getElementById('ticketList'), newTicket=document.getElementById('newTicket'), msgs=document.getElementById('msgs'), msgText=document.getElementById('msgText'), sendMsg=document.getElementById('sendMsg'), chatHint=document.getElementById('chatHint');
let AUTH={ token:null, nick:null }, CUR={ ticket:null, es:null };

loginBtn.onclick=async()=>{ loginErr.textContent=''; const nick=(nickIn.value||'').trim(); if(!nick){ loginErr.textContent='Введите ник'; return; }
  try{ const r=await fetch('/api/support/auth',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({nick})}); const j=await r.json();
    if(!j.ok){ loginErr.textContent=(j.error==='reserved_nick')?'Этот ник нельзя использовать':'Ошибка входа'; return; }
    AUTH={ token:j.token, nick:j.nick }; loginBox.style.display='none'; chatBox.style.display=''; chatHint.textContent='Создайте тикет и начните переписку.'; reloadTickets();
  }catch(e){ loginErr.textContent='Ошибка сети' } };

async function reloadTickets(){ ticketList.innerHTML=''; const r=await fetch('/api/support/tickets?token='+encodeURIComponent(AUTH.token)); const j=await r.json(); if(!j.ok){ ticketList.textContent='Ошибка загрузки списка'; return; } j.items.forEach(it=>{ const d=document.createElement('div'); d.textContent=`${it.id} — ${it.created_at}`; d.style.cursor='pointer'; d.onclick=()=>openTicket(it.id); ticketList.appendChild(d); }); }

newTicket.onclick=async()=>{ const r=await fetch('/api/support/ticket',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({token:AUTH.token})}); const j=await r.json(); if(j.ok){ await reloadTickets(); openTicket(j.ticket_id); } };

function openTicket(id){ CUR.ticket=id; msgs.innerHTML=''; if(CUR.es){ CUR.es.close(); CUR.es=null } CUR.es=new EventSource(`/api/support/stream?token=${encodeURIComponent(AUTH.token)}&ticket_id=${encodeURIComponent(id)}`); CUR.es.onmessage=(ev)=>{ try{ const m=JSON.parse(ev.data); appendMsg(m.from,m.text) }catch(_){ } }; CUR.es.addEventListener('hello',()=>appendSys('Подключено к тикету '+id)) }
function appendMsg(from,text){ const d=document.createElement('div'); d.className='msg'; const f=document.createElement('div'); f.className='from'; f.textContent=from; const b=document.createElement('div'); b.className='bubble'; b.textContent=text; d.appendChild(f); d.appendChild(b); msgs.appendChild(d); msgs.scrollTop=msgs.scrollHeight }
function appendSys(text){ appendMsg('system', text) }
sendMsg.onclick=async()=>{ const t=msgText.value.trim(); if(!t||!CUR.ticket) return; msgText.value=''; const r=await fetch('/api/support/send',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({token:AUTH.token,ticket_id:CUR.ticket,text:t})}); const j=await r.json(); if(!j.ok){ appendSys('Не отправлено') } };
msgText.addEventListener('keydown',(e)=>{ if(e.key==='Enter' && !e.shiftKey){ e.preventDefault(); sendMsg.click() } });
</script>
</body></html>
"""

# ----- HTML: панель поддержки -----
ADMIN_HTML = r"""<!doctype html>
<html lang="ru"><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Support — ShareBox</title>
<style>
:root{ --bg:#0b0f0f; --text:#e8f7ea; --muted:#9fb7a7; --accent:#2aff77; --accent2:#0bd466; --auracolor:#0f351f; --glow:0.5; --aura:0.35; --radius:16px; --pblur:6; --pshadow:0.5 }
*{box-sizing:border-box} html,body{height:100%}
body{margin:0;background:var(--bg);color:var(--text);font:16px/1.45 Inter,Segoe UI,Roboto,Arial,sans-serif}
.topbar{ position:fixed; right:14px; top:10px; display:flex; gap:10px }
.iconbtn{ background:#101714; border:1px solid #173b29; color:#bfe6cd; border-radius:10px; padding:8px 12px; cursor:pointer; transition:.15s } .iconbtn:hover{ filter:brightness(1.1) } .iconbtn:active{ transform:translateY(1px) }
.wrap{max-width:1000px;margin:0 auto;padding:60px 20px 20px}
.card{background:linear-gradient(180deg,#0d1412,#0a0f0e);border:1px solid #123326;border-radius:var(--radius);padding:16px;backdrop-filter:blur(calc(var(--pblur)*1px));filter:drop-shadow(0 30px calc(40px*var(--pshadow)) rgba(0,0,0,.6))}
h1{margin:0 0 10px} .muted{color:var(--muted)}
.grid{display:grid;grid-template-columns:320px 1fr;gap:12px}
.list{border-right:1px dashed #173b29;padding-right:12px}
.item{padding:6px 8px;border:1px solid #173b29;border-radius:8px;margin:6px 0;cursor:pointer}
.item:hover{filter:brightness(1.1)}
.chat{display:flex;flex-direction:column;height:70vh}
.msgs{flex:1;border:1px solid #173b29;border-radius:12px;padding:10px;overflow:auto;background:#0f1614}
.msg{margin:6px 0} .msg .from{color:#86c7a0;font-size:12px} .msg .bubble{display:inline-block;padding:8px 10px;background:#0c1412;border:1px solid #173b29;border-radius:10px;margin-top:2px}
.input{display:flex;gap:8px;margin-top:10px} .input input{flex:1;background:#0d1413;border:1px solid #173b29;color:var(--text);border-radius:10px;padding:8px 10px}
.overlay{ position:fixed; inset:0; display:none; align-items:center; justify-content:center; z-index:30 } .overlay.show{ display:flex } .overlay .back{ position:absolute; inset:0; background:#0007; opacity:0; animation:fadeIn .2s forwards }
.panel{ width:min(920px,calc(100% - 40px)); background:linear-gradient(180deg,#0d1412,#0a0f0e); border:1px solid #123326; border-radius:var(--radius); padding:22px; backdrop-filter:blur(calc(var(--pblur)*1px)); filter: drop-shadow(0 30px calc(40px*var(--pshadow)) rgba(0,0,0,.6)); transform:translateY(8px) scale(.98); opacity:0; animation:popIn .22s forwards }
@keyframes fadeIn{ to{opacity:1} } @keyframes popIn{ to{transform:none; opacity:1} } .hide .back{ animation:fadeOut .18s forwards } .hide .panel{ animation:popOut .18s forwards } @keyframes fadeOut{ to{opacity:0} } @keyframes popOut{ to{ transform:translateY(8px) scale(.98); opacity:0 } }
.set-row{ display:flex; gap:16px; flex-wrap:wrap } .set-box{ flex:1 1 260px; min-width:240px; background:#0e1513; border:1px solid #163626; border-radius:14px; padding:16px }
.set-title{ font-weight:700; margin-bottom:10px } .input{ width:100%; background:#0d1413; border:1px solid #173b29; color:#e8f7ea; border-radius:10px; padding:8px 10px }
.sliderrow{ display:flex; align-items:center; gap:10px } .sliderrow input[type=range]{ flex:1 }
</style>
</head>
<body>
<div class="topbar">
  <a class="iconbtn" href="/logout">Выйти</a>
  <button id="settingsBtn" class="iconbtn">Настройки</button>
</div>

<div class="wrap">
  <div class="card">
    <h1>Панель оператора</h1>
    <div class="muted" style="margin-bottom:8px">Выберите тикет слева, отвечайте в чат справа. Enter — отправка.</div>
    <div class="grid">
      <div class="list">
        <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:6px">
          <div style="font-weight:700">Тикеты</div>
          <button class="iconbtn" onclick="reload()">Обновить</button>
        </div>
        <div id="list"></div>
      </div>
      <div class="chat">
        <div id="msgs" class="msgs"></div>
        <div class="input">
          <input id="text" placeholder="Сообщение...">
          <button class="iconbtn" onclick="send()">Отправить</button>
        </div>
      </div>
    </div>
  </div>
</div>

<!-- Настройки (как на основном сайте) -->
<div id="settingsOverlay" class="overlay">
  <div class="back"></div>
  <div class="panel">
    <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:10px">
      <div style="font-weight:700; font-size:18px">Настройки</div>
      <button id="closeSettings" class="iconbtn">Закрыть</button>
    </div>
    <div class="set-row">
      <div class="set-box">
        <div class="set-title">Акцентный цвет</div>
        <input id="accentPick" type="color" class="input" value="#2aff77">
      </div>
      <div class="set-box">
        <div class="set-title">Цвет ауры (фон)</div>
        <input id="auraPick" type="color" class="input" value="#0f351f">
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:90px">Интенсивность ауры</label><input id="aura" type="range" min="0" max="1" step="0.01" value="0.35"></div>
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:90px">Свет</label><input id="glow" type="range" min="0" max="1" step="0.01" value="0.5"></div>
      </div>
      <div class="set-box">
        <div class="set-title">Панель</div>
        <div class="sliderrow"><label style="min-width:140px">Скругление</label><input id="rad" type="range" min="8" max="30" step="1" value="16"></div>
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:140px">Размытие (px)</label><input id="pblur" type="range" min="0" max="20" step="1" value="6"></div>
        <div class="sliderrow" style="margin-top:8px"><label style="min-width:140px">Тень</label><input id="pshadow" type="range" min="0" max="1" step="0.01" value="0.5"></div>
      </div>
    </div>
  </div>
</div>

<script>
let CUR=null, ES=null;
function append(from,text){ const d=document.createElement('div'); d.className='msg'; const f=document.createElement('div'); f.className='from'; f.textContent=from; const b=document.createElement('div'); b.className='bubble'; b.textContent=text; d.appendChild(f); d.appendChild(b); msgs.appendChild(d); msgs.scrollTop=msgs.scrollHeight }
function openTicket(id){ CUR=id; msgs.innerHTML=''; if(ES){ ES.close(); ES=null } ES=new EventSource('/api/support/admin/stream?ticket_id='+encodeURIComponent(id)); ES.onmessage=ev=>{ try{ const m=JSON.parse(ev.data); append(m.from,m.text) }catch(_){ } } }
async function reload(){ const r=await fetch('/api/support/admin/tickets'); const j=await r.json(); const el=document.getElementById('list'); el.innerHTML=''; if(!j.ok){ el.textContent='Нет доступа или ошибка.'; return } j.items.forEach(it=>{ const d=document.createElement('div'); d.className='item'; d.textContent=`${it.id} — ${it.owner_nick} — ${it.created_at}`; d.onclick=()=>openTicket(it.id); el.appendChild(d); }) }
async function send(){ const t=document.getElementById('text').value.trim(); if(!t||!CUR) return; document.getElementById('text').value=''; await fetch('/api/support/admin/send',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({ticket_id:CUR,text:t})}) }
document.getElementById('text').addEventListener('keydown',(e)=>{ if(e.key==='Enter'&&!e.shiftKey){ e.preventDefault(); send() } });

const settingsBtn=document.getElementById('settingsBtn'), settingsOverlay=document.getElementById('settingsOverlay'), closeSettings=document.getElementById('closeSettings');
function showOverlay(el){ el.classList.remove('hide'); el.classList.add('show'); el.style.display='flex' }
function hideOverlay(el){ el.classList.add('hide'); setTimeout(()=>{ el.classList.remove('show'); el.classList.remove('hide'); el.style.display='none' },180) }
settingsBtn.onclick=()=>showOverlay(settingsOverlay); closeSettings.onclick=()=>hideOverlay(settingsOverlay);

// настройки панели/ауры/акцента
const accentPick=document.getElementById('accentPick'), auraPick=document.getElementById('auraPick'), glowIn=document.getElementById('glow'), auraIn=document.getElementById('aura');
const rad=document.getElementById('rad'), pblur=document.getElementById('pblur'), pshadow=document.getElementById('pshadow');

function applyAccent(hex){
  document.documentElement.style.setProperty('--accent', hex);
  function darken(h,p){const c=parseInt(h.slice(1),16),r=(c>>16)&255,g=(c>>8)&255,b=c&255;const n=x=>Math.max(0,Math.min(255,Math.round(x*(1-p))));return '#'+(n(r)<<16|n(g)<<8|n(b)).toString(16).padStart(6,'0')}
  document.documentElement.style.setProperty('--accent2', darken(hex,.3));
}
function applyAuraBg(){
  const rgb = (h=>{const n=parseInt(h.slice(1),16);return[(n>>16)&255,(n>>8)&255,n&255]})(auraPick.value);
  const a = (0.5*parseFloat(auraIn.value||'0.35')).toFixed(3);
  document.body.style.background = `radial-gradient(1400px 900px at -15% -15%, rgba(${rgb[0]},${rgb[1]},${rgb[2]},${a}), transparent 70%), radial-gradient(1400px 900px at 115% 115%, rgba(${rgb[0]},${rgb[1]},${rgb[2]},${a}), transparent 70%), var(--bg)`;
}
function save(){ localStorage.setItem('admin_panel', JSON.stringify({acc:accentPick.value,aura:auraPick.value,glow:glowIn.value,aur: auraIn.value,r:rad.value,b:pblur.value,s:pshadow.value})) }
function load(){ try{ const s=JSON.parse(localStorage.getItem('admin_panel')||'{}'); if(s.acc){accentPick.value=s.acc;auraPick.value=s.aura;glowIn.value=s.glow;auraIn.value=s.aur;rad.value=s.r;pblur.value=s.b;pshadow.value=s.s} }catch(e){} apply() }
function apply(){ applyAccent(accentPick.value); applyAuraBg(); document.documentElement.style.setProperty('--glow',glowIn.value); document.documentElement.style.setProperty('--aura',auraIn.value); document.documentElement.style.setProperty('--radius',rad.value+'px'); document.documentElement.style.setProperty('--pblur',pblur.value); document.documentElement.style.setProperty('--pshadow',pshadow.value); save() }
[accentPick,auraPick,glowIn,auraIn,rad,pblur,pshadow].forEach(x=> x.addEventListener('input', apply))
load(); reload();
</script>
</body></html>
"""

# ----- HTML: логин -----
LOGIN_HTML = r"""<!doctype html>
<html lang="ru"><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>ShareBox — вход</title>
<style>
body{margin:0;background:#0b0f0f;color:#e8f7ea;font:16px/1.45 Inter,Segoe UI,Roboto,Arial,sans-serif;display:flex;align-items:center;justify-content:center;height:100vh}
.card{background:linear-gradient(180deg,#0d1412,#0a0f0e);border:1px solid #123326;border-radius:16px;padding:24px 20px;max-width:520px;box-shadow:0 30px 60px #000a}
h1{margin:0 0 10px} .muted{color:#9fb7a7}
.btn{display:inline-block;margin-top:14px;padding:10px 14px;border-radius:10px;background:linear-gradient(180deg,#2aff77,#0bd466);color:#04240f;font-weight:700;text-decoration:none}
.warn{margin-top:12px;color:#ffb067;display:none}
</style></head>
<body>
<div class="card">
  <h1>ShareBox</h1>
  <div class="muted">Войдите через Google, чтобы использовать сайт.</div>
  <a class="btn" href="/login">Войти через Google</a>
  <div id="w" class="warn">Google OAuth не настроен. Укажи GOOGLE_CLIENT_ID/GOOGLE_CLIENT_SECRET в .env</div>
</div>
<script>if("{{NEEDCFG}}"==="true"){ document.getElementById('w').style.display='block' }</script>
</body></html>
"""

# ----- HTML: отказ в доступе к панели -----
SUPPORT_DENIED_HTML = r"""<!doctype html>
<html lang="ru"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Нет доступа</title><style>
body{margin:0;background:#0b0f0f;color:#e8f7ea;font:16px/1.45 Inter,Segoe UI,Roboto,Arial,sans-serif;display:flex;align-items:center;justify-content:center;height:100vh}
.card{background:#101615;border:1px solid #193a2b;border-radius:16px;padding:28px;max-width:560px;box-shadow:0 30px 60px #000a}
h1{margin:0 0 10px}.muted{color:#9fb7a7}
</style></head><body><div class="card"><h1>Нет доступа</h1><div class="muted">Ваш аккаунт не допущен в панель поддержки.</div></div></body></html>
"""

if __name__ == "__main__":
    print(f"User UI:    http://{USER_HOST}:{PORT}/")
    print(f"Support UI: http://{SUPPORT_HOST}:{PORT}/")
    app.run(host="0.0.0.0", port=PORT, debug=False, threaded=True)

# Karanel
Голосовой помощник 
from pathlib import Path
import zipfile, shutil

root=Path("/mnt/data/Karanell_Web_Deploy")
if root.exists(): shutil.rmtree(root)
(root/"static").mkdir(parents=True)

(root/"app.py").write_text('''import os
from flask import Flask, request, jsonify, send_from_directory
from openai import OpenAI

app = Flask(__name__, static_folder="static", static_url_path="/static")
key = os.getenv("OPENAI_API_KEY")
client = OpenAI(api_key=key) if key else None

SYSTEM = """Ты — Каранел, голосовой помощник. Отвечай по-русски естественно и кратко. Не выдумывай факты."""

@app.get("/")
def home():
    return send_from_directory("static", "index.html")

@app.get("/health")
def health():
    return jsonify({"ok": True, "api_configured": bool(key)})

@app.post("/api/chat")
def chat():
    if not client:
        return jsonify({"error": "OPENAI_API_KEY не настроен на сервере."}), 500
    data = request.get_json(silent=True) or {}
    message = (data.get("message") or "").strip()
    if not message:
        return jsonify({"error": "Пустое сообщение."}), 400
    try:
        r = client.responses.create(
            model=os.getenv("KARANELL_MODEL", "gpt-6-astra"),
            instructions=SYSTEM,
            tools=[{"type": "web_search"}],
            input=message
        )
        return jsonify({"reply": r.output_text.strip()})
    except Exception:
        return jsonify({"error": "Ошибка сервера."}), 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.getenv("PORT", "8080")))
''', encoding="utf-8")

(root/"static/index.html").write_text('''<!doctype html>
<html lang="ru"><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<meta name="theme-color" content="#0b1020"><title>Каранел</title>
<style>
body{margin:0;background:#080d1a;color:#fff;font-family:system-ui,-apple-system,Segoe UI,sans-serif}
main{width:min(620px,100%);min-height:100vh;margin:auto;padding:28px 20px;box-sizing:border-box}
h1{text-align:center;font-size:40px;margin:20px 0 4px}header p{text-align:center;color:#9ca8bc;margin:0}
.orb{width:190px;height:190px;margin:65px auto 35px;border-radius:50%;display:grid;place-items:center;font-size:58px;background:radial-gradient(circle at 35% 25%,#60a5fa,#4f46e5 48%,#111827 75%);box-shadow:0 0 70px #4f46e555}
.on{animation:pulse 1s infinite}@keyframes pulse{50%{transform:scale(1.05)}}
.status{text-align:center;color:#c7d2fe;min-height:28px}button{width:100%;border:0;border-radius:20px;padding:18px;font-size:19px;font-weight:700;background:#fff;color:#111827;margin-top:18px}
.log{margin-top:20px;border:1px solid #24304a;border-radius:18px;padding:14px;min-height:100px;max-height:260px;overflow:auto;background:#101827}
.msg{margin:9px 0;line-height:1.45}.me{color:#93c5fd}.bot{color:#c4b5fd}.hint{text-align:center;color:#68758d;font-size:12px;margin-top:14px}
</style></head><body><main>
<header><h1>Каранел</h1><p>Голосовой помощник</p></header>
<div id="orb" class="orb">🎙️</div><div id="status" class="status">Готов к работе</div>
<button id="talk">Нажми и говори</button>
<div id="log" class="log"><div class="msg bot"><b>Каранел:</b> Привет! Нажми кнопку и скажи что-нибудь.</div></div>
<div class="hint">Для микрофона через интернет нужен HTTPS.</div>
<script>
const talk=document.querySelector("#talk"),status=document.querySelector("#status"),orb=document.querySelector("#orb"),log=document.querySelector("#log");
const SR=window.SpeechRecognition||window.webkitSpeechRecognition;
function esc(s){return s.replace(/[&<>"']/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#039;"}[c]))}
function add(c,n,t){log.innerHTML+=`<div class="msg ${c}"><b>${n}:</b> ${esc(t)}</div>`;log.scrollTop=log.scrollHeight}
function speak(t){if(!speechSynthesis)return;speechSynthesis.cancel();let u=new SpeechSynthesisUtterance(t);u.lang="ru-RU";speechSynthesis.speak(u)}
async function ask(text){status.textContent="Каранел думает…";talk.disabled=true;try{let r=await fetch("/api/chat",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify({message:text})});let d=await r.json();if(!r.ok)throw Error();add("bot","Каранел",d.reply);speak(d.reply);status.textContent="Готов к работе"}catch(e){add("bot","Каранел","Сервер пока недоступен.");status.textContent="Ошибка соединения"}finally{talk.disabled=false}}
if(SR){let rec=new SR();rec.lang="ru-RU";rec.continuous=false;rec.interimResults=false;rec.onstart=()=>{orb.classList.add("on");status.textContent="Слушаю…";talk.textContent="Слушаю…"};rec.onend=()=>{orb.classList.remove("on");talk.textContent="Нажми и говори"};rec.onerror=()=>status.textContent="Не удалось услышать речь";rec.onresult=e=>{let t=e.results[0][0].transcript.trim();if(t){add("me","Ты",t);ask(t)}};talk.onclick=()=>rec.start()}else{talk.disabled=true;status.textContent="Открой в современном Chrome или Safari"}
</script></main></body></html>
''', encoding="utf-8")

(root/"requirements.txt").write_text("Flask>=3.0,<4\nopenai>=1.0\ngunicorn>=23,<24\n", encoding="utf-8")
(root/"Procfile").write_text("web: gunicorn app:app\n", encoding="utf-8")
(root/"render.yaml").write_text("""services:
  - type: web
    name: karanell
    runtime: python
    buildCommand: pip install -r requirements.txt
    startCommand: gunicorn app:app
    envVars:
      - key: OPENAI_API_KEY
        sync: false
      - key: KARANELL_MODEL
        value: gpt-6-astra
""", encoding="utf-8")
(root/".gitignore").write_text(".venv/\n__pycache__/\n.env\n*.pyc\n", encoding="utf-8")
(root/"README.md").write_text("""# Каранел — готовая веб-версия

Загрузка на Render:
1. Создай GitHub-репозиторий и загрузи файлы из ZIP.
2. В Render выбери New Web Service и этот репозиторий.
3. Добавь Environment Variable `OPENAI_API_KEY`.
4. Deploy.
5. Получишь HTTPS-адрес вида `https://karanell-xxxx.onrender.com`.

Не публикуй API-ключ в GitHub или в index.html.
""", encoding="utf-8")

zip_path=Path("/mnt/data/Karanell_Ready_For_Web_Deploy.zip")
with zipfile.ZipFile(zip_path,"w",zipfile.ZIP_DEFLATED) as z:
    for f in root.rglob("*"):
        if f.is_file(): z.write(f,f.relative_to(root))
print(zip_path)
# Guida completa a ruyijs

## 1. Cos'e ruyijs

ruyijs e un motore JavaScript headless basato sul vero motore Firefox 159
(SpiderMonkey) e sullo stack di rete Necko. Non ha finestra, renderer ne
layout engine: esegue JavaScript di siti web esattamente come farebbe Firefox,
ma in un processo leggero e isolato, controllabile da Python.

Usalo quando un client HTTP puro non basta (pagine renderizzate da JS,
challenge anti-bot, firme, fingerprint) ma un browser completo
(Playwright/CDP) e troppo pesante o troppo facilmente rilevabile.

Non usarlo per pagine statiche o quando esiste un'API/MCP ufficiale.

## 2. Installazione su questa macchina

- Source checkout: D:\Programmi\ff-ruyijs
- Package Python: D:\Programmi\ff-ruyijs\python
- Runtime nativo (SpiderMonkey + Necko): D:\Programmi\ff-ruyijs\python\ruyijs\_runtime
  (js.exe, mozglue.dll, runtime VC++)
- Installazione editable nel venv Hermes (Python 3.11)
- Installazione editable anche nel Python 3.14 usato da execute_code
  (tramite .pth nel venv Hermes)

Verifica rapida:

```python
import sys
sys.path.insert(0, r"D:\Programmi\ff-ruyijs\python")
from ruyijs import Context
with Context() as ctx:
    print(ctx.eval("navigator.userAgent"))
# Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:159.0) Gecko/20100101 Firefox/159.0
```

Se get_runtime_path() solleva FileNotFoundError, il runtime nativo manca:
installa la wheel di piattaforma (pip install ruyijs) o copia _runtime da una
wheel installata in python/ruyijs/_runtime, oppure imposta RUYIJS_SHELL su
un js.exe esistente.

## 3. Come lanciarlo

### 3.1 Da terminal (venv Hermes)

Il venv Hermes ha gia l'installazione editable:

```bash
C:\Users\emili\AppData\Local\hermes\hermes-agent\venv\Scripts\python.exe script.py
```

### 3.2 Da execute_code (Hermes)

execute_code usa un Python diverso dal venv Hermes. Aggiungi il source tree
al sys.path prima dell'import:

```python
import sys
sys.path.insert(0, r"D:\Programmi\ff-ruyijs\python")
from ruyijs import Context
```

### 3.3 Utilizzo diretto (Python di sistema)

```bash
python -c "import sys; sys.path.insert(0,'D:/Programmi/ff-ruyijs/python'); from ruyijs import Context; print(Context().eval('1+2'))"
```

## 4. Cosa si puo fare

- Eseguire JavaScript arbitrario in un motore Firefox reale e leggere il
  risultato in Python.
- Navigare URL reali (HTTP/HTTPS) con lo stack Necko.
- Caricare HTML catturato ed eseguirne gli script (replay offline).
- Servire risposte offline (put_resource) per riprodurre flussi catturati.
- Iniettare dati Python nel mondo JS (set) ed esporre funzioni (expose).
- Configurare fingerprint, ambiente, clock, random, replay di timer/messaggi/
  navigazioni/crypto.
- Usare proxy per sessioni isolate.
- Eseguire piu sessioni in un solo processo con Engine/use_engine.
- Raccogliere garbage, fare checkpoint, chiudere il contesto.

## 5. API principali

### Creazione contesto

```python
from ruyijs import Context

# contesto isolato (processo dedicato)
with Context() as ctx:
    ...

# con proxy / timeout / ambiente
ctx = Context(proxy="http://user:pass@host:port", timeout=30.0,
              environment="desktop", mode="default")
```

### Eval

```python
value = ctx.eval("1 + 2")                      # 3
ua = ctx.eval("navigator.userAgent")
raw = ctx.eval("JSON.stringify(window.__DATA__)")
```

Preferisci JSON.stringify(...) in eval quando restituisci oggetti, poi
json.loads in Python. Gli oggetti JS complessi non sempre si convertono
puliti.

### Navigazione

```python
ctx.navigate("https://example.com")
ctx.advance(5000)          # lascia fetchare, eseguire script, microtask
title = ctx.eval("document.title")
```

navigate() avvia la richiesta ma non blocca. Chiamare advance(ms) prima di
leggere il DOM.

### Caricamento HTML catturato

```python
ctx.load_page(url, html)   # esegue script parser-inserted, readyState complete
data = ctx.eval("JSON.stringify(window.__DATA__)")
```

### Iniezione dati / risorse offline

```python
ctx.set("payload", {"id": 42})
ctx.put_resource("https://example.com/api/data", body,
                 status=200, headers={"Content-Type": "application/json"})
```

### Configurazione (prima della navigazione)

```python
ctx.configure_fingerprint(fp)
ctx.configure_environment(env)
ctx.configure_clock("logical", epoch_ms=0)   # o "system"
ctx.configure_random(state0, state1)
ctx.configure_timer_replay(True)
ctx.configure_message_replay(True)
ctx.configure_navigation_replay(docs)
ctx.configure_crypto_replay(docs)
```

### Lifecycle / utility

```python
ctx.checkpoint()
ctx.collect_garbage()
ctx.close()   # ripetibile, non fa nulla se gia chiuso
```

### Più sessioni in un processo

```python
from ruyijs import Engine, use_engine

engine = Engine()
with use_engine(engine):
    with Context() as a:
        ...
    with Context() as b:
        ...
```

## 6. Firme esatte (verificate)

- Context(shell=None, timeout=10.0, environment=None, mode="default", proxy=None, ...)
- ctx.eval(code, filename="<eval>", line=-1, col=-1, to_py=...)
- ctx.navigate(url)
- ctx.advance(ms, step_ms=16.667, max_tasks=100000)
- ctx.load_page(url, html, *, replay_async_scripts=None)
- ctx.set(name, value)
- ctx.put_resource(url, body, status=200, headers=None)
- ctx.configure_fingerprint(fingerprint)
- ctx.configure_environment(environment)
- ctx.configure_clock(mode, epoch_ms=None)
- ctx.configure_random(state0, state1)
- ctx.configure_crypto_replay(documents)
- ctx.configure_timer_replay(enabled=True)
- ctx.configure_message_replay(enabled=True)
- ctx.configure_navigation_replay(documents)
- ctx.expose(function, name=None)
- ctx.checkpoint(), ctx.collect_garbage(), ctx.close()

## 7. Esempi d'uso

### 7.1 Eval base (verificato)

```python
import sys
sys.path.insert(0, r"D:\Programmi\ff-ruyijs\python")
from ruyijs import Context

with Context() as ctx:
    print(ctx.eval("navigator.userAgent"))
    print(ctx.eval("Math.sqrt(16)"))
```

### 7.2 Replay offline di HTML catturato (verificato)

```python
import sys, json
sys.path.insert(0, r"D:\Programmi\ff-ruyijs\python")
from ruyijs import Context

html = """<!doctype html>
<html><body>
<script>window.__DATA__ = {value: 42, title: "demo"};</script>
</body></html>"""

with Context() as ctx:
    ctx.load_page("https://example.test/page", html)
    raw = ctx.eval("JSON.stringify(window.__DATA__)")
    print(json.loads(raw))   # {'value': 42, 'title': 'demo'}
```

### 7.3 Navigazione reale

```python
import sys
sys.path.insert(0, r"D:\Programmi\ff-ruyijs\python")
from ruyijs import Context

with Context() as ctx:
    ctx.navigate("https://example.com")
    ctx.advance(5000)
    print(ctx.eval("document.title"))
```

### 7.4 Challenge / firma (pattern)

```python
import sys, json
sys.path.insert(0, r"D:\Programmi\ff-ruyijs\python")
from ruyijs import Context

# 1) cattura html+script dal browser reale
# 2) servi eventuali risorse offline
# 3) load_page + eval della firma
with Context() as ctx:
    ctx.put_resource("https://target.test/api/token",
                     '{"token":"abc"}', status=200,
                     headers={"Content-Type": "application/json"})
    ctx.load_page("https://target.test/", captured_html)
    sig = ctx.eval("window.computeSignature && window.computeSignature()")
    print(sig)
```

### 7.5 Proxy e sessione isolata

```python
with Context(proxy="http://user:pass@host:port") as ctx:
    ctx.navigate("https://example.com")
    ctx.advance(5000)
    print(ctx.eval("location.href"))
```

## 8. Pitfall e troubleshooting

- Runtime mancante: i checkout sorgente non includono js.exe. Installa la
  wheel o copia _runtime.
- Navigazione asincrona: leggere il DOM subito dopo navigate() da dati
  vecchi/vuoti. Chiama advance() prima.
- Processi orfani: crea i contesti in blocchi with; contesti non chiusi
  lasciano processi SpiderMonkey attivi.
- Valori di ritorno complessi: usa JSON.stringify in eval.
- Proxy/fingerprint vanno configurati prima della navigazione.
- Clock logico per replay deterministico; clock di sistema per comportamento
  reale.
- Mismatch ambienti Hermes: terminal e execute_code usano Python diversi;
  verifica l'import nell'ambiente dove gira il codice.
- Non condividere stato tra contesti non correlati.

## 9. Integrazione con Hermes

- Skill dedicata: ruyijs (caricabile con skill_view(name='ruyijs')).
- Tool browser CDP resta preferibile per sessioni reali con login.
- ruyijs e preferibile per scraping JS, challenge, firme, replay offline.
- Per eseguire codice, usa execute_code (con sys.path insert) o terminal
  (venv Hermes).

## 10. Riferimenti

- Repo: https://github.com/cutetailbud/ff-ruyijs
- Checkout locale: D:\Programmi\ff-ruyijs
- AGENTS.md, docs/engine.md, docs/tls-profile.md
- Esempi: examples/ (akamai, creepjs, DataDome, truepeople)
- Skill Hermes: ruyijs

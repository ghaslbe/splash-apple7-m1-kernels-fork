# Splash auf einem M1 Max: ein Nachmittags-Experiment

## Ausgangslage

Ziel war, den vom Nutzer geposteten Snippet zu prüfen und ggf. zu installieren:

```bash
git clone https://github.com/npanj/splash.git -b q8
cd splash
make -j4
./splash serve --model nitinpanj/Qwen3.8-27B-Splash-HQ
```

Splash ist eine Inferenz-Engine für Apple Silicon von [incoai](https://github.com/incoai/splash),
gebaut "around the model" statt als generisches Framework: Metal-Kernel pro Modell handgetuned,
spekulative Dekodierung (DFlash 2) als Standardpfad statt Option, und ein Speicherplan, der beim
Start exakt für die vorhandene GPU/RAM-Kombination berechnet wird.

## Schritt 1: Ist der Q8-Fork überhaupt seriös?

Vor dem Clonen erst geprüft, statt blind auszuführen:

- `npanj/splash` (Branch `q8`): Fork von `incoai/splash`, 6 Stars, 0 Forks, **frisch gepusht am
  selben Tag** — sehr dünne Historie für ein 30 GB-Modell-Release.
- Das referenzierte HuggingFace-Modell `nitinpanj/Qwen3.8-27B-Splash-HQ` existierte zwar mit
  echten Gewichtsdateien, war aber ebenfalls erst wenige Tage alt.
- Anforderung laut Q8-Fork-README: **Apple M3 oder neuer**, macOS 26.4+, 48 GB RAM (64 GB
  empfohlen) — für ein Q8-Modell mit 27 GB Rohgewicht.

Ergebnis: kein Malware-Fund, aber technisch für M1 **nicht vorgesehen** und für 32 GB RAM ohnehin
zu groß. Abgelehnt.

## Schritt 2: Der eigentliche M1-Fork

Der Nutzer brachte den entscheidenden Link:
[`paperniuk/splash`](https://github.com/paperniuk/splash), Branch `apple7-m1-kernels`.

- Legitimer Community-Fork (44 Stars, Owner seit 2021 aktiv), der **Metal-Kernel explizit für
  Apple7/8-GPUs (M1, M2)** nachrüstet — genau die Generation, die im offiziellen Repo
  ausgeschlossen ist.
- Nutzt das offizielle, kleinere 4-bit-Modell `incoai/Qwen3.8-27B-Splash` (17,4 GB statt 27 GB).
- Laut Quick-Start weiterhin 36 GB RAM-Minimum — bei 32 GB physischem RAM also knapp unter der
  Empfehlung, aber ein Versuch war es wert.

## Schritt 3: Build

- Xcode 26.6 vorhanden, aber `xcrun metal` schlug fehl: fehlendes Metal-Toolchain-Bundle.
  Nachinstalliert mit `xcodebuild -downloadComponent MetalToolchain` (~688 MB Download).
- Danach lief `make -j4` durch: eigener Build-Cache (`build_config.py record`), Metal-Kernel
  kompilieren zu `.air` → `.metallib`, C++/Objective-C++-Runtime zu `libsplash.a`, linken zu
  `build/splash` (Mach-O arm64). Keine Fehler.

## Schritt 4: Erster Start – Speicherplatz statt RAM war das erste Hindernis

Modell-Download (17,4 GB, 79 Dateien) brach bei 30 % ab:

```
error: could not download Splash runtime package ...: File reconstruction error:
IO Error: No space left on device (os error 28)
```

Ursache: von 926 GB Festplatte waren nur noch 4,9 GB frei — mehrere alte, für Splash nicht
kompatible Qwen3.8-27B-Downloads (GGUF, MLX-4bit, mxfp4 aus früheren Experimenten mit anderen
Tools) belegten zusammen über 30 GB im HuggingFace-Cache. Nutzer hat über den Papierkorb
aufgeräumt; nach kurzer Verzögerung (APFS-Snapshots/purgeable space brauchen einen Moment, bis sie
wirklich freigegeben werden) waren 112 GB frei, Download lief danach zügig durch.

## Schritt 5: Es läuft — und das Speicherbudget ist transparent

Nach Download und Ready-Meldung zeigte `/status` einen bemerkenswert detaillierten
Speicherplan: exakte Byte-Aufschlüsselung für Gewichte (Target 15,16 GB, Draft 1,27 GB, Vision
0,93 GB), KV-Page-Geometrie, `maximum_context_tokens` abhängig vom konfigurierten
`--max-memory`. Bei `--max-memory 27G` errechnete der Server selbst **184.313 Tokens** möglichen
Kontext (nativ unterstützt das Modell bis 262.144).

Erster echter Request (`"Was ist die Hauptstadt von Deutschland?"`) lief in ~1,1 s Time-to-first-
Token durch — funktionierte auf Anhieb korrekt.

## Schritt 6: Coding-Test unter Last — hier wurde es interessant

Der Nutzer startete einen eigenen CRUD-Codegenerierungs-Test (`mc.py`, "Personalversammlung")
gegen den lokalen Server. Dabei zwei unabhängige Probleme sichtbar:

**A) Client-Kompatibilität.** Das Testskript fragte Ollama-typische Endpunkte an
(`/api/tags`, `/api/v0/models`), die Splash nicht kennt (nur OpenAI-/Anthropic-kompatible
`/v1/...`-Routen). Führte zu einer Serie harmloser `not_found`-Fehler parallel zu echten,
erfolgreichen `/v1/chat/completions`-Aufrufen.

**B) Wiederkehrende Engine-Abstürze unter Speicherdruck.** Alle paar Minuten:

```
error: native transport stopped after an engine failure
```

gefolgt von automatischem Neustart des nativen Engine-Prozesses durch den Python-Supervisor.
Musterhaft korreliert mit System-RAM, das auf wenige hundert MB frei fiel.

### Root-Cause-Analyse im Code

Quellcode direkt durchsucht (`runtime/engine/NativeRuntime.cpp`, `runtime/metal/MetalBackend.mm`):

- Der eingebaute `MemoryGovernor` funktioniert wie vorgesehen — Log zeigt aktives Drosseln
  (`Memory: growth paused` / `growth available`) unter Druck.
- Aber: **jede** Ausnahme während eines Engine-Ticks markiert die komplette Engine als
  "unhealthy" und beendet die Verbindung (`NativeRuntime::executionFailed`, kein Retry, kein
  differenziertes Fehlschlagen nur des einen Requests).
- Wahrscheinlichster Auslöser: ein "sparse unmapping timed out"-Pfad in `MetalBackend.mm`
  (30 Sekunden Timeout) für das placement-sparse KV-Speicher-Management, das dieser M1-Fork
  extra nutzt. Unter starkem System-weiten Speicherdruck (Compressor/Swap) kann die
  GPU-Command-Queue so weit verzögern, dass dieser Timeout reißt.
- Zusätzlich beobachtet: `--max-memory` deckelt nur den *dynamischen* Anteil (KV-Cache). Die
  Fixkosten (Gewichte + Runtime-Overhead, ca. 17–20 GB) werden davon nicht proportional
  reduziert — bei 18 GB Limit passte nicht mal mehr ein aktiver State-Cell + eine KV-Extent
  (`kv_pool_does_not_fit`).

### Iteratives Tuning

| Versuch | `--max-context` | `--max-memory` | Ergebnis |
|---|---|---|---|
| 1 | 8K | 24G | lief, aber ungetestet unter Last |
| 2 | 128K | 27G | 2× Engine-Absturz unter Speicherdruck |
| 3 | 16K | 18G | Start-Fehler: `kv_pool_does_not_fit` |
| 4 | 8K | 20G | stabil, aber `context_length_exceeded` bei längeren Konversationen |
| 5 | 32K | 20G | laufender Kompromiss |

Kernerkenntnis: Bei einem 27B-Modell auf einer 32-GB-Maschine gibt es keinen Parameter, der das
strukturelle Problem löst — die Fixkosten von ~20 GB für Gewichte + Overhead lassen bei
gleichzeitig laufenden anderen Anwendungen (Browser, IDE, Coding-Agent) kaum Puffer für macOS
selbst. `iogpu.wired_limit_mb` hochzusetzen hätte das sogar verschlimmert, da es der GPU noch
*mehr* wired-Speicher erlaubt hätte statt dem System mehr Luft zu lassen.

## Messwerte (Auszug, 4-bit-Modell, M1 Max, 32 Kerne GPU)

- Time-to-first-Token (kalt, ~2.700 Input-Tokens): ~20,4–20,7 s
- Time-to-first-Token (mit Prompt-Cache-Hit, 2.720/3.207 Tokens gecacht): ~4,0 s
- Decode-Geschwindigkeit: ~31–33 Tokens/Sekunde
- Kein Absturz mehr beobachtet, seit `--max-context` auf 8K–32K und `--max-memory` auf 20G
  begrenzt wurde (Restrisiko bleibt bei parallelem Speicherdruck durch andere Prozesse)

## Nachtrag: Aufräumen und finaler Zustand

Nach der längeren Coding-Test-Session lief der Verdacht auf, dass nicht mehr sauber nachvollziehbar
war, was auf der Maschine alles noch aktiv ist. Ein voller Prozess-Check ergab:

- Neben Splash und den beiden Instanzen des `mc.py`-Testclients liefen noch über ein Dutzend
  **fremde, teils monatealte Python-Prozesse** im Hintergrund: ein `uvicorn`-Server auf Port 8005,
  mehrere `python -m http.server`-Instanzen (Ports 8123, 8044, 8087, 8042, 8099, 8741, teils seit
  Juli), ein OpenRouter-Debug-Proxy, ein `static_preview_server.py`. Alle auf Nutzerwunsch beendet
  (`pkill -f -i python`), nachdem klar war, dass sie nicht mehr gebraucht werden.
- Dabei fiel zusätzlich ein **hängender `node /tmp/smoke.mjs`-Prozess** auf, der seit dem 18. August
  ununterbrochen lief und **100 % CPU** verbrauchte — ein klassischer vergessener Debug-/Smoke-Test-
  Prozess. Ebenfalls beendet.
- Nicht angerührt: eine zweite, parallele Claude-Code-Session (`claude --resume ...`) mit ihrem
  eigenen React-Dev-Server für ein anderes Projekt — die gehörte erkennbar zu einer separaten,
  aktiven Arbeit des Nutzers.

Nach dem Aufräumen stand deutlich mehr Kopf-Spielraum zur Verfügung (kurzzeitig bis zu 6 GB frei
direkt nach dem Kill, siedelte sich danach bei ca. 1,5–1,7 GB frei ein). Splash wurde sauber neu
gestartet mit etwas mehr Kontext, da der rechnerische Spielraum (`memory_plan.maximum_context_tokens`)
unverändert bei ca. 184.000 Tokens lag, unabhängig vom kleineren `--max-context`-Flag.

**Finale Start-Zeile dieser Session:**

```bash
./splash serve --model incoai/Qwen3.8-27B-Splash --max-context 64K --max-memory 24G
```

Intern aufgelöst zu:

```
server/server.py ... --host 127.0.0.1 --port 8000 --max-memory 25769803776 --max-context 65536
build/splash serve-native .../target .../draft 65536 25769803776
```

Lehre daraus: Auf einer Maschine mit vielen parallel laufenden, lange offenen Terminal-Sessions
lohnt sich ein `ps aux | sort -rk3` (CPU) bzw. eine Sortierung nach RSS (Speicher) *vor* dem
Feintuning von `--max-memory` — ein einzelner vergessener 100-%-CPU-Prozess oder ein Dutzend alter
`http.server`-Instanzen verzerren sonst jede Beobachtung darüber, wie viel Spielraum Splash
tatsächlich hat.

## Nachtrag 2: Abgebrochene Tool-Calls und ein Patch in Splash

Später tauchten im Log wiederholt abgebrochene Requests auf, jeweils mit identischem, fast
vollständig gecachtem Input (8.738 Tokens) und Tool-Calls:

```
17:09:13 Cancelled · input 8,738 · cached 8,480 · output 810 · tools 12 · 0.9 tok/s
17:24:58 Cancelled · input 8,738 · cached 8,736 · output 865 · tools 12 · 0.9 tok/s
17:42:14 Cancelled · input 8,738 · cached 8,736 · output 3,816 · tools 12 · 3.8 tok/s
```

`Cancelled` heißt bei Splash: Der Client hat die Verbindung geschlossen. Die Ursache war ein
Zusammenspiel von Splash und dem Coding-Agenten mc.py:

- **Splash puffert bestimmte Tool-Argumente komplett** (`server/output.py`). Parameter mit
  einfachem String-Schema werden Token für Token gestreamt. Arrays und Objekte hält Splash
  zurück, bis sie fertig sind, weil es sie erst dann typisiert und validiert. In dieser Zeit
  gingen nur SSE-Kommentare (`: splash-keepalive`) über die Leitung.
- **mc.py bricht nach 150 s ohne echtes Datenereignis ab** (`STALL_TIMEOUT`) und ignoriert
  Kommentare absichtlich.
- Das mc.py-Tool `write_files` erwartet `files: [{path, content}, …]`, also ein Array von
  Objekten mit komplettem Dateiinhalt. Schreibt das Modell damit mehrere Dateien, kommen über
  Minuten keine Daten an → mc.py bricht ab → mc.py sendet denselben Request erneut.

**Patch in Splash** (`server/server.py`, Chat-Completions-Stream): Solange nichts sichtbar
gestreamt wird, sendet Splash jetzt alle 2 s einen leeren Delta-Chunk
(`data: {"choices":[{"delta":{},"finish_reason":null}]}`) statt eines Kommentars. Das ist gültiges
OpenAI-Streaming-Format und wird von Splash schon für `prompt_progress` genutzt. Ein Test, der
das alte Verhalten erwartete, wurde angepasst; alle 196 Server-Tests laufen grün.

### Der Patch im Detail

**Wo:** `server/server.py`, Methode `FrontendHandler._stream()`. Das ist der Streaming-Pfad für
`POST /v1/chat/completions` mit `"stream": true`, also die OpenAI-kompatible Schnittstelle, die
mc.py benutzt.

**Wie Splash vorher arbeitete:** Die Methode `_collect()` wartet auf Ereignisse der nativen
Engine und bekommt einen Callback für Leerlauf übergeben. Ist seit dem letzten Schreibvorgang auf
dem HTTP-Stream mehr als `SSE_KEEPALIVE_SECONDS` (2 s) vergangen, ruft sie diesen Callback auf.
Übergeben wurde bisher `self._sse_keepalive`, der eine SSE-Kommentarzeile schreibt:

```
: splash-keepalive
```

Kommentarzeilen halten die TCP-Verbindung am Leben, sind aber laut SSE-Spezifikation keine
Ereignisse. SDKs und Clients wie mc.py sehen sie nicht als Fortschritt.

**Was der Patch ändert:** Im Chat-Completions-Stream wird stattdessen ein eigener Callback
übergeben, der einen regulären, leeren Chunk im OpenAI-Format schreibt:

```
data: {"id":"chatcmpl-…","object":"chat.completion.chunk","created":…,"model":"incoai/Qwen3.8-27B-Splash","choices":[{"index":0,"delta":{},"finish_reason":null}]}
```

Ein leeres `delta` bedeutet für jeden OpenAI-kompatiblen Client „nichts Neues“. Es wird beim
Zusammensetzen der Antwort einfach übersprungen, zählt aber als Datenereignis. Splash nutzt genau
diese Form schon selbst für die `prompt_progress`-Meldungen während des Prefills.

Der vollständige Diff:

```diff
--- a/server/server.py
+++ b/server/server.py
@@ -1527,13 +1527,19 @@ class FrontendHandler(BaseHTTPRequestHandler):
                     )
                 )
 
+            def keepalive():
+                # Array/object tool arguments are buffered until complete, which
+                # can take minutes. Clients that time out on missing data events
+                # ignore SSE comments, so send an empty delta chunk instead.
+                self._sse(stream_chunk(self.app.model, public_id, created, {}))
+
             _, _, tool_calls, result, _ = self._collect(
                 job,
                 thinking,
                 has_tools,
                 put_text,
                 put_tool_delta,
-                self._sse_keepalive,
+                keepalive,
                 put_progress,
             )
```

**Was bewusst unverändert blieb:**
- *Vor dem Start der Antwort* (Warteschlange, Ressourcenwartezeit) schickt Splash weiterhin
  Kommentar-Keepalives. Zu diesem Zeitpunkt ist noch kein Chunk mit der Rolle `assistant`
  verschickt, und ein Daten-Chunk davor wäre formal falsch.
- *Anthropic-Messages-Stream* (`/v1/messages`) und *Responses-Stream* (`/v1/responses`): Beide
  senden in solchen Pausen schon echte Ereignisse (`ping` bzw. `response.in_progress`) und
  brauchen den Patch nicht.
- *Wenn Text oder Tool-Deltas fließen*, fällt kein zusätzlicher Chunk an: Der Leerlauf-Timer
  misst die Zeit seit dem letzten Schreiben auf den Stream, jeder echte Chunk setzt ihn zurück.

**Test-Anpassung:** In `dev/tests/test_server.py` prüfte
`test_stream_sends_keepalive_while_native_is_idle`, dass im Chat-Stream nach dem Rollen-Chunk ein
Kommentar-Keepalive folgt. Der Test erwartet jetzt einen leeren Delta-Chunk:

```diff
                 role = json.loads(self.next_sse_data(response))
                 self.assertEqual(role["choices"][0]["delta"]["role"], "assistant")
-                self.assertEqual(
-                    response.readline().decode().rstrip("\r\n"),
-                    ": splash-keepalive",
-                )
+                heartbeat = json.loads(self.next_sse_data(response))
+                self.assertEqual(heartbeat["choices"][0]["delta"], {})
+                self.assertIsNone(heartbeat["choices"][0]["finish_reason"])
```

Die Tests für gepufferte Tool-Generierung und für die anderen Stream-Typen übergeben ihren
Callback direkt und blieben unverändert. Gelaufen ist die komplette Server-Testsuite
(`python -m unittest dev.tests.test_server`) in einer separaten virtuellen Umgebung mit den
Dev-Abhängigkeiten: 196 Tests, alle grün.

**Einschränkungen:** Der Patch liegt nur lokal im Clone des `apple7-m1-kernels`-Forks und ist
nicht committet oder upstream eingereicht. Er beseitigt die Abbrüche durch Pufferung, macht die
Pufferung selbst aber nicht kürzer: Ein Client sieht während der Generierung von Array- oder
Objekt-Argumenten weiterhin keinen Inhalt, nur die leeren Chunks. Die gründlichere Lösung wäre,
auch Strings innerhalb von Arrays und Objekten live zu streamen (in `server/output.py`), was einen
Umbau des Tool-Call-Parsers bedeuten würde.

Live-Test mit einem `write_files`-artigen Tool (Array von Objekten):

| | vorher (unter Swap-Druck) | nach dem Patch |
|---|---|---|
| Dauer | 459 s, Engine-Absturz | 12,9 s, fertig |
| größte Lücke zwischen Datenereignissen | 442 s | 2,0 s |
| Speed | – | 26,2 tok/s |

Der erste Testlauf zeigte zugleich die Grenze des Patches: Wenn macOS swappt (1,27 GB Swap,
190 MB frei), wird auch der Splash-Prozess selbst ausgebremst. Dann kommen gar keine Events
mehr, bis die Engine abstürzt. Erst nach dem Schließen weiterer Apps (u. a. Citrix) lief der
Test sauber durch. Der Patch behebt die Abbrüche durch die Pufferung, aber nicht den
Speichermangel.

**Kontextlänge:** Der KV-Cache kostet bei diesem Modell ca. 33 KB pro Token
(1.064.960 Bytes pro 32-Token-Seite). 32K statt 24K kostet also höchstens ~270 MB mehr, und das
auch nur, wenn eine Konversation tatsächlich so lang wird. Deshalb der Endstand:

```bash
./splash serve --model incoai/Qwen3.8-27B-Splash --max-context 32K --max-memory 24G
```

## Nachtrag 3: Dauerbetrieb mit dem Coding-Agenten

Nach dem Patch lief Splash rund anderthalb Stunden unter echter Last: mc.py baute eine
CRUD-Webanwendung (Flask-Backend + React/Vite-Frontend) mit nativen Tool-Calls und 12 Tools. Eine
Hintergrund-Überwachung wertete dabei das Splash-Log und den `/status`-Endpunkt aus.

**Zahlen (18:25–19:56, Konfiguration `--max-context 32K --max-memory 24G`):**

| Kennzahl | Wert |
|---|---|
| fertige Requests | 22 |
| abgebrochene Requests | 3 (alle mit 0 Output, durch Neustarts von mc.py) |
| Engine-Abstürze | 1 (18:42) |
| Decode-Geschwindigkeit | min 18,9 · Ø 28,0 · max 48,0 tok/s |
| Zeit bis zum ersten Token | min 3,3 s · Ø 47,8 s · max 133,5 s |
| Anteil gecachter Prompt-Tokens | 60,6 % |
| Output gesamt | 30.704 Tokens, größte Einzelantwort 6.651 Tokens |
| Kontextgröße | bis ~20.000 Tokens |

**Beobachtungen:**

- **Der Streaming-Patch bewährt sich.** Tool-Antworten mit 5.056 und 6.651 Output-Tokens liefen
  3,5 bzw. gut 4 Minuten am Stück durch. Vorher hätte mc.py solche Antworten nach 150 s
  abgebrochen, weil während der Pufferung keine Datenereignisse ankamen. Seit dem Patch gab es
  keinen einzigen Abbruch mehr durch Pufferung.
- **Der Prompt-Cache entscheidet über das Tempo.** Mit hohem Cache-Anteil kam der erste Token nach
  3–10 s, ohne Cache erst nach 80–134 s. Zwei Ursachen für Cache-Fehlschläge:
  mc.py kürzte in einem früheren Lauf den Verlauf bei jedem Schritt (dadurch änderte sich der
  Prompt-Anfang), und der Wechsel zwischen Hauptverlauf und `explore`-Unterläufen verdrängt bei
  knappem Speicher den jeweils anderen Cache-Eintrag. Im späteren Lauf kürzte mc.py nur noch bei
  Kontextdruck, danach lag der Cache-Anteil oft bei über 90 %.
- **Speed-Spitzen bei kurzen Antworten.** Die höchsten Werte (41–48 tok/s) gab es bei kurzen
  Tool-Aufrufen. Dort trifft das Draft-Modell der spekulativen Dekodierung besonders oft.
- **Ein Engine-Absturz ohne Swap-Anstieg.** Um 18:42 hing ein Request gut 5 Minuten und die Engine
  starb (`native transport stopped after an engine failure`), obwohl der Swap diesmal stabil
  blieb. Splash startete die Engine automatisch neu, mc.py beendete sich aber. Das spricht dafür,
  dass der inoffizielle M1-Fork unter Dauerlast nicht völlig stabil ist, unabhängig vom knappen
  RAM.
- **Abgeschnittene Antworten.** Zweimal endete eine Antwort bei genau 4.000 Tokens am
  `max_tokens`-Limit von mc.py und musste fortgesetzt werden. Im späteren Lauf war das Limit höher.
- **Aufgeräumt hilft.** Nach dem Beenden von Citrix blieb der Speicherdruck meist „normal“ mit
  0,1–1,4 GB freiem RAM, und der Swap wuchs nicht mehr (konstant 1,15 GB).

## Nachtrag 4: Drei Fallen im Agenten und ein erfolgreicher Lauf

Nachdem Splash stabil lief, lagen die restlichen Probleme fast alle im Zusammenspiel mit dem
Agenten. Drei davon kosteten jeweils einen ganzen Lauf.

### Falle 1: Denken frisst das Token-Budget

Ein Schritt endete mehrfach bei genau 8.000 Output-Tokens, dem `max_tokens`-Limit von mc.py.
Qwen3.8 denkt standardmäßig mit, und diese Reasoning-Tokens zählen gegen das Limit. Denken plus ein
großer Tool-Call mit Dateiinhalt passten nicht hinein. Bei nativen Tool-Calls ist eine
abgeschnittene Antwort komplett verloren: mc.py meldete „Unvollständige native Antwort; keine
Aktion ausgeführt“ und stellte denselben Request neu, dreimal hintereinander, jeweils nach rund
6 Minuten Generierung.

**Lösung:** Agent mit `--no-think` starten (`reasoning_effort: "none"`) und `max_tokens` auf 12.000
anheben. Danach brauchten dieselben Schritte nur noch wenige hundert Tokens.

### Falle 2: Das Modell ahmt gekürzte Verläufe nach

Wird der Kontext voll, ersetzt mc.py alte Tool-Runden durch Text-Zusammenfassungen im Stil von:

```
[Fruehere Tool-Runde zusammengefasst; Inhalte bei Bedarf erneut lesen]
read_file {"path": ".../App.jsx"}
[Ergebnis von read_file]
Inhalt von .../App.jsx (3646 Zeichen, 123 Zeilen): …
```

Das Modell hat dieses Muster nachgeahmt. Statt echte Tools aufzurufen, schrieb es ein
**erfundenes Protokoll samt erfundener Dateiinhalte** als normalen Text. mc.py wertete die
Textantwort als Schlussantwort und beendete sich. Ohne Denkphase (`--no-think`) passierte das
noch leichter. Weil der Verlauf mit `--resume` gespeichert und fortgesetzt wurde, kippte danach
jeder neue Lauf schon beim ersten Request. mc.py übernahm die erfundenen Antworten sogar in den
Verlauf, was das Muster weiter verstärkte.

**Lösung:** Mit frischem Verlauf starten. Dauerhaft muss die Kürzung in mc.py die Struktur aus
Tool-Aufruf und Tool-Ergebnis erhalten und nur den Inhalt kürzen, und eine Textantwort, die wie
ein Tool-Protokoll aussieht, darf nicht als Ende zählen.

### Falle 3: Prompt plus `max_tokens` sprengt den Kontext

Ein Lauf endete mit drei `context_length_exceeded` hintereinander, obwohl der Prompt nur ~21.000
Tokens groß war. Splash rechnet bei `/v1/chat/completions` **Prompt + `max_tokens`** gegen das
Kontextfenster (`server/frontend.py`) und lehnt ab, wenn die Summe zu groß ist. Das entspricht dem
Verhalten der OpenAI-API. Nur die Anthropic-Schnittstelle kappt das Output-Budget stattdessen
automatisch. 21.000 + 12.000 lagen über den eingestellten 32.768.

Die Empfehlung für `max_tokens` 12.000 kam aus dieser Analyse und hatte das Wachstum des Prompts
nicht eingerechnet. mc.py hielt den Fehler außerdem für einen zu langen Prompt und kürzte hart,
aber diese harte Kürzung entfernte nichts: Dreimal wurden exakt dieselben 74.925 Zeichen gesendet.

**Lösung:** Splash auf 48K Kontext umgestellt. Der zusätzliche Speicher fällt nur an, wenn der
Kontext tatsächlich genutzt wird (ca. 33 KB pro Token). Dauerhaft sollte der Agent `max_tokens`
pro Request dynamisch begrenzen, also auf höchstens Kontextgröße minus Prompt minus Puffer.

Eine Nebenwirkung eines aufgeblähten, fortgesetzten Verlaufs zeigte sich ebenfalls: Lag er über
der Kürzungsschwelle, kürzte mc.py bei *jedem* Schritt. Der Prompt-Anfang änderte sich dadurch
ständig, der Cache griff nicht mehr, und jeder Schritt kostete über 4 Minuten Prefill.

### Der erfolgreiche Lauf

Mit frischem Verlauf, `--no-think`, `max_tokens` 12.000 und 48K Kontext lief der Agent durch und
schloss die Aufgabe mit `finish` ab. Die Anwendung war laut Agent real verifiziert:

- **Backend** (Flask + SQLite): alle vier REST-Endpunkte inkl. Fehlerfällen per curl geprüft
  (GET 200, POST 201, PUT 200/404, DELETE 204/404, viermal 400 bei ungültigen Eingaben)
- **Frontend** (React/Vite + `@material/web`): `vite build` fehlerfrei, `oxlint` exit 0,
  Dev-Server erreichbar
- **CORS** zwischen Frontend-Origin und Backend geprüft

| Kennzahl | Wert |
|---|---|
| Dauer | ~25 Minuten |
| Requests | 44, kein Fehler |
| Prompt-Tokens | 671.308, davon 84 % aus dem Cache |
| Output-Tokens | 9.324 |
| Decode-Geschwindigkeit | meist 30–50 tok/s |
| Nachahmung des Tool-Protokolls | keine |

**Endstand der Konfiguration:**

```bash
./splash serve --model incoai/Qwen3.8-27B-Splash --max-context 48K --max-memory 24G
```

Agent: `mc.py --tool-mode native --no-think --check --max-steps 60`, `MC_MAX_TOKENS=12000`,
frischer Verlauf.

## Fazit

Splash läuft auf einem M1 Max dank des inoffiziellen `apple7-m1-kernels`-Forks, und zwar
brauchbar: Im Dauerbetrieb mit einem Coding-Agenten lagen im Schnitt 28 tok/s an, mit Spitzen
bis 48 tok/s, und über 30.000 Output-Tokens liefen in anderthalb Stunden durch. Der
Speicherplan ist beeindruckend transparent.

Bei 32 GB Gesamt-RAM ist das 4-bit-27B-Modell aber am oberen Limit dessen, was die Maschine
hergibt. Es funktioniert nur, wenn nebenher wenig läuft; die sinnvolle Kontextlänge liegt bei
32–48K statt der nativen 256K, und unter Dauerlast gab es einen Engine-Absturz.

Am Ende hat ein Coding-Agent darauf eine vollständige, geprüfte CRUD-Anwendung gebaut, in
25 Minuten und ohne einen einzigen Fehler von Splash. Am meisten gebracht haben:
- ein kleiner Patch in Splash, der während gepufferter Tool-Argumente echte Datenereignisse
  schickt,
- ein Agent, der den Prompt-Anfang stabil hält, damit der Prompt-Cache greift (84 % im
  erfolgreichen Lauf),
- das Abschalten der Denkphase und ein zum Kontext passendes `max_tokens`,
- und ein frischer Verlauf, wenn ein gespeicherter Verlauf Muster enthält, die das Modell
  nachahmt.

Die meisten Stolpersteine lagen dabei nicht in der Inferenz-Engine, sondern im Agenten drumherum.
Für produktiven Dauerbetrieb auf dieser Hardware bleibt ein kleineres Modell oder mehr RAM
trotzdem der nachhaltigere Weg.

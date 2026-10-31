# Fix-Anleitung: hängende wrangler-PowerShell-Skripte (D1-Debug-Skripte)

**Zielgruppe:** Claude Code in einem anderen Projekt. Mit dieser Anleitung lassen sich dieselben Korrekturen in gleichen oder analogen PowerShell-Skripten umsetzen, die `wrangler` (Cloudflare D1) aufrufen.

**Herkunft:** TraderScope-Homepage-Repo, Ordner `_debug/`, Stand 2026-10-31. Korrigiert wurden:

| Skript | Zweck | Art |
|---|---|---|
| `_debug/change-user-trial-end.ps1` | Trial-Ende eines Users setzen (2x `UPDATE`, dann `SELECT`) | schreibend |
| `_debug/show-users.ps1` | Übersicht User/Lizenz/Subscription (`SELECT`) | nur lesend |
| `_debug/delete-user.ps1` | User komplett löschen (SQL-Datei via `--file`, dann 2x Verifikations-`SELECT`) | destruktiv |

---

## 1. Ausgangslage: gemeinsamer Aufbau der Skripte

Alle drei Skripte haben dasselbe Gerüst. Analoge Skripte im Zielprojekt sehen typischerweise so aus:

1. `param([switch]$Local, ...)`, `$ErrorActionPreference = "Stop"`
2. Manueller Konfig-Block: `$TargetEnv = "prod"|"dev"` plus Ziel-Parameter (E-Mail, Datum ...)
3. `$repoRoot = Split-Path -Parent $PSScriptRoot` (Skripte liegen in `<repo>/_debug/`)
4. `database_name` und `database_id` per Regex aus `wrangler.toml` bzw. `wrangler.dev.toml` lesen
5. `$mode = if ($Local) { "--local" } else { "--remote" }`
6. Bisherige `CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID` sichern, im `finally` wiederherstellen
7. Im `try`: Token und Account-ID aus `.deploy.env` laden (`CLOUDFLARE_API_TOKEN_{PROD,DEV}`, `CLOUDFLARE_ACCOUNT_ID_{PROD,DEV}`) und als Env-Vars setzen
8. **Pre-Flight:** `wrangler whoami` und prüfen, ob die erwartete Account-ID in der Ausgabe vorkommt. Abbruch vor jedem SQL, falls nicht.
9. `wrangler d1 execute <db> --remote --config <cfg> --command <sql>` (oder `--file <tmp.sql>`)

**Originalcode (problematisch):**

```powershell
$who = (npx wrangler whoami | Out-String)
...
npx wrangler d1 execute $dbName $mode --config $configPath --command $sql
npx wrangler d1 execute $dbName $mode --config $configPath --file $tempSqlFile   # nur delete-user
```

---

## 2. Symptome

- Das Skript bleibt nach `Pruefe aktiven Cloudflare-Account fuer 'prod'...` scheinbar endlos stehen, ohne jede Ausgabe.
- Bei `delete-user.ps1` erscheint `This process may take some time, during which your D1 database will be unavailable to serve queries. Ok to proceed? » (Y/n)`, und das Fenster nimmt dann zeitweise **keine Tastatureingaben** an.

---

## 3. Ursachen

### 3.1 `npx` kann unsichtbar an einer Install-Rückfrage hängen

Wird das Skript außerhalb des Repos gestartet (oder findet `npx` kein lokales `node_modules/.bin/wrangler`), fragt `npx`: `Need to install the following packages: wrangler ... Ok to proceed? (y)`. Durch `| Out-String` ist diese Frage **unsichtbar**, und das Skript wartet für immer.

**Fix:** Das lokal installierte wrangler direkt über `node` starten, also `node <repo>/node_modules/wrangler/bin/wrangler.js`. Im TraderScope-Repo macht `scripts/lib/wrangler.mjs` (`spawnWrangler`) das genauso. So läuft das Skript unabhängig vom Startverzeichnis und ohne Rückfrage.

### 3.2 `| Out-String` puffert die komplette Ausgabe

`$who = (npx wrangler whoami | Out-String)` zeigt nichts an, bis wrangler fertig ist. Ein langsamer API-Call (`GET /client/v4/user/tokens/verify`) sieht dadurch aus wie ein Absturz.

**Fix:** `Tee-Object -Variable` statt `Out-String`. Die Ausgabe erscheint live, und der Account-Abgleich bekommt trotzdem den Text.

### 3.3 wranglers interaktive `(Y/n)`-Abfrage bei `d1 execute --remote --file`

Bei `--file` nutzt wrangler den Import-Weg und fragt vorher interaktiv nach. Im Windows-PowerShell-Fenster nahm diese Abfrage teils keine Eingaben an.

**Fix:** Eine eigene Bestätigung per `Read-Host` (PowerShell-nativ, zuverlässig) und wrangler mit `--yes` aufrufen. Laut `wrangler d1 execute --help` (wrangler 4.100.0) gilt: `-y, --yes  Answer "yes" to any prompts  [boolean]`. Die Sicherheitsbremse vor dem Löschen bleibt so erhalten.

### 3.4 Nicht vom Skript verursacht: Netzwerk

Ein Teil der Hänger lag an der Netzwerkverbindung der Maschine, nicht am Skript. Belege: Auch `Invoke-WebRequest https://www.google.com` und rohe TCP-Connects auf Port 443 liefen in einen Timeout, und das wrangler-Log endete nach `START CF API REQUEST: GET .../user/tokens/verify` ohne Antwort. Der Browser funktionierte währenddessen, Programme über den Windows-Netzwerkstack nicht.

**Konsequenz für die Diagnose im Zielprojekt:** Hängt ein Skript nach den Fixes weiter bei `Getting User settings...` oder `Executing on remote database ...`, zuerst die Verbindung prüfen, bevor am Code gesucht wird (siehe Abschnitt 7).

---

## 4. Die Änderungen im Einzelnen

### Änderung A: `Invoke-Wrangler`-Helfer (alle 3 Skripte)

**Wo:** Nach der Definition von `$repoRoot` und `$mode`, vor dem ersten wrangler-Aufruf. In TraderScope steht er direkt hinter `$mode = ...` (bzw. in `delete-user.ps1` direkt hinter `$expectedDbId = $Matches[1]`, weil `$mode` dort weiter oben steht). Entscheidend ist nur: `$repoRoot` muss vorher gesetzt sein.

**Einfügen:**

```powershell
# Lokales wrangler direkt ueber node starten (wie scripts/lib/wrangler.mjs), NICHT
# ueber npx: npx haengt sonst unsichtbar an einer "Ok to proceed?"-Install-Abfrage,
# wenn das Skript ausserhalb des Repos gestartet wird.
$wranglerJs = Join-Path $repoRoot "node_modules\wrangler\bin\wrangler.js"
if (-not (Test-Path -LiteralPath $wranglerJs)) {
  throw "wrangler nicht gefunden: $wranglerJs (vorher 'npm install' im Repo-Root ausfuehren)"
}
function Invoke-Wrangler { & node $wranglerJs @args }
```

Hinweise:
- `@args` reicht alle Argumente unverändert durch, auch Strings mit Leerzeichen wie das SQL in `--command`.
- `$LASTEXITCODE` wird nach `Invoke-Wrangler ...` korrekt vom `node`-Prozess gesetzt. Bestehende Prüfungen `if ($LASTEXITCODE -ne 0) { throw ... }` funktionieren unverändert.
- Liegt `wrangler` im Zielprojekt woanders (Monorepo, anderes Package), den Pfad anpassen. Prüfen mit `Test-Path <repo>\node_modules\wrangler\bin\wrangler.js`.

### Änderung B: Pre-Flight `whoami` mit Live-Ausgabe (alle 3 Skripte)

**Vorher:**

```powershell
    $who = (npx wrangler whoami | Out-String)
```

**Nachher:**

```powershell
    # Tee statt Out-String: Ausgabe live sichtbar (vorher wirkte ein langsamer
    # API-Call wie ein Haenger, weil alles bis zum Ende gepuffert wurde).
    Invoke-Wrangler whoami | Tee-Object -Variable whoLines
    $who = $whoLines | Out-String
```

Die nachfolgenden Zeilen bleiben unverändert:

```powershell
    if ($LASTEXITCODE -ne 0) {
      throw "wrangler whoami fehlgeschlagen (exit $LASTEXITCODE) - Account nicht verifizierbar. Abbruch VOR SQL."
    }
    if ($who -notmatch [regex]::Escape($accountId)) {
      throw "Account-Check fehlgeschlagen: ..."
    }
```

**Wichtig, nicht tun:** Kein `2>&1` an den node/wrangler-Aufruf hängen. In Windows PowerShell 5.1 macht `2>&1` bei nativen Programmen aus jeder stderr-Zeile einen `ErrorRecord`. Zusammen mit `$ErrorActionPreference = "Stop"` bricht das Skript dann schon bei der ersten harmlosen wrangler-Warnung ab.

### Änderung C: alle `npx wrangler d1 execute` ersetzen (alle 3 Skripte)

Jedes `npx wrangler` wird zu `Invoke-Wrangler`, die Argumente bleiben gleich:

```powershell
# vorher
npx wrangler d1 execute $dbName $mode --config $configPath --command $sql
# nachher
Invoke-Wrangler d1 execute $dbName $mode --config $configPath --command $sql
```

Betroffene Stellen in TraderScope:

| Skript | Aufrufe |
|---|---|
| `change-user-trial-end.ps1` | `--command $updateSql`, `--command $verifySql` |
| `show-users.ps1` | `--command $showSql` |
| `delete-user.ps1` | `--file $tempSqlFile` (siehe Änderung D), `--command $targetCountsSql`, `--command $remainingUsersSql` |

**Kontrolle danach:** Im ganzen Skript darf kein `npx wrangler` mehr vorkommen.

### Änderung D: eigene Bestätigung plus `--yes` beim destruktiven `--file`-Aufruf (nur `delete-user.ps1`)

**Vorher:**

```powershell
  Write-Host "Deleting $email from D1..." -ForegroundColor Yellow
  npx wrangler d1 execute $dbName $mode --config $configPath --file $tempSqlFile
  if ($LASTEXITCODE -ne 0) { throw "Delete fehlgeschlagen (wrangler exit $LASTEXITCODE)." }
```

**Nachher:**

```powershell
  # Eigene Bestaetigung statt wranglers "Ok to proceed? (Y/n)": dessen Prompt nimmt
  # im Windows-Terminal teils keine Tastatureingaben an -> wrangler laeuft mit --yes.
  $answer = Read-Host "Wirklich $email aus env=$TargetEnv loeschen? 'ja' eingeben zum Fortfahren"
  if ($answer.Trim().ToLowerInvariant() -ne "ja") {
    throw "Abgebrochen - nichts geloescht."
  }
  Write-Host "Deleting $email from D1..." -ForegroundColor Yellow
  Invoke-Wrangler d1 execute $dbName $mode --config $configPath --file $tempSqlFile --yes
  if ($LASTEXITCODE -ne 0) { throw "Delete fehlgeschlagen (wrangler exit $LASTEXITCODE)." }
```

Regeln für die Übertragung:
- Die Bestätigung steht **nach** dem Account-Check und **nach** der Ausgabe `Ziel: env=... db=... mode=...`, damit der Nutzer vor dem `ja` sieht, was getroffen wird.
- Nur exakt `ja` (Groß-/Kleinschreibung egal) führt weiter. Alles andere bricht ab, bevor SQL zur DB geht.
- Das `throw` liegt im `try`, deshalb laufen die Aufräumarbeiten im `finally` trotzdem: temporäre SQL-Datei löschen, Credentials wiederherstellen.
- `--yes` **nur** an den Aufruf, dem die eigene Bestätigung vorausgeht. Nie pauschal in `Invoke-Wrangler` einbauen.
- Analog für andere destruktive Skripte im Zielprojekt verwenden (z. B. `cleanup-*`, `reset-*`, alles mit `DELETE`/`DROP` per `--file`).

### Änderung E (optional, im Ausgangsprojekt nachträglich ergänzt): Exit-Code der Verifikation prüfen

In `change-user-trial-end.ps1` wurde der Exit-Code der abschließenden Verifikations-Abfrage ergänzt:

```powershell
  Invoke-Wrangler d1 execute $dbName $mode --config $configPath --command $verifySql
  if ($LASTEXITCODE -ne 0) {
    throw "Verifikation fehlgeschlagen (wrangler exit $LASTEXITCODE)."
  }
```

Empfehlung: Nach jedem `Invoke-Wrangler d1 execute` den `$LASTEXITCODE` prüfen, auch nach rein lesenden Abfragen.

---

## 5. Vorgehen im Zielprojekt (Checkliste für Claude)

1. **Kandidaten finden:** In allen `*.ps1` nach `npx wrangler` suchen (Grep, Glob `**/*.ps1`). Auch `npx.cmd wrangler` und `& npx` berücksichtigen.
2. **Pfad prüfen:** Existiert `<repoRoot>\node_modules\wrangler\bin\wrangler.js`? Wie wird `$repoRoot` im Skript ermittelt? Fehlt die Variable, `Split-Path -Parent $PSScriptRoot` (Skript in Unterordner) bzw. `$PSScriptRoot` (Skript im Root) verwenden.
3. **Änderung A** einfügen (nach `$repoRoot`, vor dem ersten wrangler-Aufruf).
4. **Änderung B** an jedem `whoami ... | Out-String`. Allgemein: jedes `(<wrangler-Aufruf> | Out-String)` auf `Tee-Object -Variable` umstellen.
5. **Änderung C** an jedem übrigen `npx wrangler`.
6. **Änderung D** an jedem `d1 execute --remote --file` und an jedem anderen Aufruf, bei dem wrangler interaktiv nachfragt.
7. **Änderung E**: fehlende `$LASTEXITCODE`-Prüfungen ergänzen.
8. Bestehende Kommentare und Abläufe nicht umbauen. Konfig-Block, `.deploy.env`-Logik, `finally`-Restore und SQL bleiben unverändert.
9. **Testen** nach Abschnitt 6, **keine** destruktiven Läufe gegen Produktion.
10. **Nicht** committen oder pushen, solange der Nutzer das nicht ausdrücklich verlangt.

---

## 6. Testverfahren (ohne Risiko für Produktionsdaten)

### 6.1 Rein lesende Skripte (wie `show-users.ps1`)

Komplett ausführen, aus einem Verzeichnis **außerhalb** des Repos (damit ist der `npx`-Fall abgedeckt), mit Timeout in einem Hintergrund-Job. Personenbezogene Ausgaben nicht in den Chat holen, nur Status-Zeilen filtern:

```powershell
$j = Start-Job -ScriptBlock { param($d) Set-Location $d
  $o = & powershell -NoProfile -ExecutionPolicy Bypass -File "<repo>\_debug\show-users.ps1" | Out-String
  "EXIT=$LASTEXITCODE"
  ($o -split "`n" | Where-Object { $_ -match 'OK: Account|Ziel:|rows_read|ERROR|fehlgeschlagen' }) -join "`n"
} -ArgumentList "<scratchpad>"
if (Wait-Job $j -Timeout 120) { Receive-Job $j } else { "TIMEOUT"; Stop-Job $j }
Remove-Job $j -Force
```

Erwartet: `EXIT=0`, `OK: Account <id> (prod) aktiv.`, `Ziel: ...`, `"rows_read": <n>`.

### 6.2 Schreibende oder destruktive Skripte: nur Pre-Flight testen

Eine Kopie im Scratchpad anlegen, in der **alle** `d1 execute`-Aufrufe durch einen Stub ersetzt sind und `$PSScriptRoot` auf den echten Skriptordner zeigt. Sonst findet die Kopie z. B. `sql\delete-user.sql` nicht und bricht mit `SQL file not found` ab.

```powershell
$sp  = "<scratchpad>"
$src = Get-Content "<repo>\_debug\delete-user.ps1" -Raw
$src = $src.Replace('$PSScriptRoot', '"<repo>\_debug"')
$src = $src -replace 'Invoke-Wrangler d1 execute[^\r\n]*', 'Write-Host "SKIP d1"'
"d1-Aufrufe uebrig: " + ([regex]::Matches($src,'Invoke-Wrangler d1')).Count   # muss 0 sein
$src | Out-File "$sp\preflight-test.ps1" -Encoding utf8
$j = Start-Job -ScriptBlock { param($p,$d) Set-Location $d
  & powershell -NoProfile -ExecutionPolicy Bypass -File $p; "EXIT=$LASTEXITCODE" } -ArgumentList "$sp\preflight-test.ps1",$sp
if (Wait-Job $j -Timeout 90) { Receive-Job $j | Where-Object { $_ -match 'OK: Account|Ziel:|SKIP|EXIT|ERROR|fehl' } } else { "TIMEOUT"; Stop-Job $j }
Remove-Job $j -Force; Remove-Item "$sp\preflight-test.ps1"
```

Erwartet: `d1-Aufrufe uebrig: 0`, `OK: Account ...`, `Ziel: ...`, `SKIP d1` pro Aufruf, `EXIT=0`.

Hat das Skript eine `Read-Host`-Bestätigung (Änderung D), steht sie in der Kopie vor dem Stub und blockiert den Job. Für den Test die Bestätigungszeilen zusätzlich auskommentieren oder `"nein" |` voranstellen. Bei `"nein"` muss `Abgebrochen - nichts geloescht.` erscheinen.

**Den eigentlichen Lauf mit echtem SQL führt der Nutzer selbst aus.**

### 6.3 Statische Kontrolle

- Kein `npx wrangler` mehr in den geänderten Skripten.
- `wrangler d1 execute --help` muss `-y, --yes` auflisten, bevor `--yes` verwendet wird (ältere wrangler-Versionen prüfen).

---

## 7. Diagnose, wenn es nach den Fixes weiter hängt

wrangler schreibt Logs nach `%APPDATA%\xdg.config\.wrangler\logs\wrangler-<datum>.log`. Endet das neueste Log nach `-- START CF API REQUEST: GET https://api.cloudflare.com/client/v4/...` ohne Antwort oder mit `The request to Cloudflare's API timed out`, liegt es am Netz, nicht am Skript. Beim Auslesen keine Authorization- oder Bearer-Zeilen ausgeben.

Netzwerk-Schnelltest (unabhängig von wrangler):

```powershell
Test-NetConnection api.cloudflare.com -Port 443            # TcpTestSucceeded : True erwartet
Invoke-WebRequest https://www.google.com -UseBasicParsing -TimeoutSec 10   # Gegenprobe ohne Cloudflare
netsh winhttp show proxy                                   # System-Proxy?
```

Beobachtet in TraderScope: DNS funktionierte, TCP-Connects auf Port 443 (Cloudflare **und** Google) liefen ins Leere, der Browser ging trotzdem. Abhilfe: Netzwerk, VPN oder Proxy prüfen bzw. neu starten. Am Code ist dann nichts zu ändern.

Hinweis zu Hintergrund-Tests: Ein mit `Stop-Job` abgebrochener Test kann node-Kindprozesse hinterlassen. Prüfen und ggf. gezielt beenden:

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" | Select-Object ProcessId, CreationDate, CommandLine
```

---

## 8. Kurzfassung

| # | Problem | Fix | Skripte |
|---|---|---|---|
| A | `npx` hängt unsichtbar an einer Install-Rückfrage | `Invoke-Wrangler` = `& node <repo>\node_modules\wrangler\bin\wrangler.js @args` | alle |
| B | `\| Out-String` puffert, Hänger-Optik | `Invoke-Wrangler whoami \| Tee-Object -Variable whoLines; $who = $whoLines \| Out-String` | alle |
| C | restliche `npx wrangler`-Aufrufe | durch `Invoke-Wrangler` ersetzen | alle |
| D | wranglers `(Y/n)` bei `--file` nimmt keine Eingabe an | eigene `Read-Host`-Bestätigung (`ja`) plus `--yes` nur an diesem Aufruf | destruktive (`delete-user`) |
| E | Fehler in Verifikations-Abfragen bleiben unbemerkt | `$LASTEXITCODE` nach jedem `d1 execute` prüfen | optional, alle |
| - | API-Timeouts trotz Fix | Netzwerk prüfen (Abschnitt 7), kein Code-Problem | - |

# Übergabe: Stand, Regeln, Stolperfallen

Zusammenfassung der bisherigen Arbeit an diesem Repo, damit es mit einem anderen
Assistenten (Copilot) oder einer anderen Person nahtlos weitergeht. Die
**README.md** beschreibt Bedienung und Einrichtung im Detail. Diese Datei
erklärt, *warum* der Code so aussieht, welche Regeln gelten und was schon
schiefging.

Stand: November 2026, Branch `stable`.

---

## 1. Umgebung

| Was | Wert |
|---|---|
| Semaphore UI | Docker auf `mgt-ansible-01`, Compose unter `/opt/docker/semaphore` |
| Ansible im Image | `ansible` 13.5.0 / `ansible-core` 2.20, venv `/opt/semaphore/apps/ansible/13.5.0/venv` |
| Collections (aus dem Image) | `ansible.windows` 3.5.0, `community.general` 12.5.0, `ansible.posix` 2.1.0, `vmware.vmware` 2.8.0 (ungenutzt, `pyvmomi` fehlt) |
| Proxy | Sophos UTM 9, `http://10.98.98.222:8080` |
| `NO_PROXY` | `localhost,127.0.0.1,db,gutingia.local,.gutingia.local` |
| Domäne | `gutingia.local` |
| vCenter | `vcenter1.gutingia.local` (Standard), ab 8.0 U1 |
| Repo | `https://github.com/myLife-Admins/playbooks.git`, Branch `stable` |
| Lokaler Klon (Windows) | `C:\mylife\ansible-assets` |
| Playbook-Ordner in Semaphore | Host `/opt/docker/semaphore/playbooks` → Container `/playbooks` (Git-Klon, Besitzer `1001:0`, **ohne** `:ro`) |
| Container-Benutzer | uid 1001, gid 0 |
| Protokoll-Share (alt) | `\\fs5\IT-Protokolle\<Jahr>\<Monat>\Serverpatch\` |

### Wie Code in Semaphore landet

1. Lokal ändern, committen, nach `stable` pushen.
2. In Semaphore das Template **Pull** starten (`pull-playbooks.yml`). Es macht
   `git pull --ff-only` in `/playbooks`.
3. Erst danach den eigentlichen Test-Template starten.

Das Repository in Semaphore ist ein **lokaler Pfad** (`/playbooks`), kein
Git-Remote. Das ist bewusst so: Fällt das Internet aus, laufen interne Jobs mit
dem letzten Stand weiter. Ein Commit ändert also nie unbemerkt den nächsten
Lauf. Details: README → „Repository: gemounteter Ordner statt Git“.

### Test-Hosts

Inventory in Semaphore, alle als FQDN: `PATCH01` (2016), `PATCH02` (2022),
`PATCHTEST2016`, `PATCHTEST2022`, `PATCHTEST2025` (`*.gutingia.local`).
Template zum Testen der Windows-Updates: `[TEST]Updates - Windows Server`.
`infra04` ist ein Admin-Arbeitsplatz (für winget und Benutzer-Installationen).

Das Repo hat derzeit **kein** `inventory/` und keine `ansible.cfg`. Beides
wurde mit Commit `clear` (1092d36) entfernt, der alte Stand liegt in
`482ad68`. Das Inventory wird in Semaphore gepflegt.

---

## 2. Was gebaut wurde (Chronik)

| Commit(s) | Ergebnis |
|---|---|
| `74b8d30`, `ce39c07` | `update-apt.yml`: Linux-Test, `apt_dry_run` mit `\| bool` |
| `c2e7a78` … `c906699` | Semaphore hinter dem Proxy, Collections aus dem Image, Ordner statt Git, `pull-playbooks.yml` |
| `9b847b7` … `bd3bfd8` | `update-winget.yml`: als SYSTEM, pro Paket statt `--all`, Mailbericht, Nachkontrolle |
| `8da0746`, `c243fa0` | Offline-Hosts sind kein Fehler, Port-Check gegen den echten Host |
| `0759709`, `1b64b32` | **Pins**: `pin list`, `winget_pin_ids` und die `pinning.db` **aller** Profile |
| `8942240` | `update-windows.yml` mit Patch-Protokoll; Bausteine nach `tasks/`, `templates/`, `vars/` ausgelagert |
| `d2dfc91` | Snapshot über die vCenter-REST- und VI/JSON-API statt `vm_snapshot` (kein `pyvmomi` im Image) |
| `cdbeba8`, `40c2e6c` | höchstens `winupd_snapshot_parallel` Snapshots gleichzeitig; Snapshot-Play nur über Hosts, die einen brauchen |
| `c0be93d` | Snapshots mit Arbeitsspeicher (`snapshot_memory`, Standard true) |
| `44848bd` | Mail an mehrere Empfänger, `mail_cc` / `mail_bcc` |
| `a9d5948`, `5d7e9b0`, `b195081` | `report-user-installs.yml`: Programme mit `--scope user` finden, mit winget-ID, CSV und fertigen Befehlen |
| `1baade5` | Fehlalarm bei WiX-Burn-Installern (z.B. Python) ausgeblendet |

Getestet und bestätigt:
- winget-Bericht mit Pins
- Windows-Updates im Testlauf auf 5 Hosts
- vCenter-Anmeldung, VM-Suche und parallele Snapshots
- Mail an mehrere Empfänger

---

## 3. Playbooks

Alle Patch-Playbooks laufen standardmäßig **nur als Testlauf**. Jedes hat
`hosts: "{{ target_hosts | default('all') }}"` und schickt am Ende einen
Bericht per Mail.

### `pull-playbooks.yml`
Läuft auf `localhost` (= Task-Container) und macht
`git -c safe.directory=/playbooks -C /playbooks pull --ff-only` mit `LC_ALL=C`.
Vorher prüft es `.git` und die Schreibrechte; die Fehlermeldung nennt `:ro` und
`chown 1001:0`. Gezeigt werden der Stand vorher und nachher.

### `update-apt.yml`
Linux. `apt_dry_run: true` listet die anstehenden Updates per `dist-upgrade` im
check_mode. Scharf führt es das Upgrade aus und meldet „reboot-required“.
**Startet nie neu.**

### `update-winget.yml`
Windows-Programme über winget.
- **Ablauf:** `winget_dry_run: true` listet nur. Scharf wird jedes Paket
  einzeln mit `winget upgrade --id <id> --exact --silent` aktualisiert. Danach
  folgt eine Nachkontrolle (`winget_verify_seconds`), bis die Pakete aus der
  Liste verschwunden sind.
- **Läuft als SYSTEM** (`become_method: runas`, `become_user: SYSTEM`), mit
  vollem Pfad zu `winget.exe` unter `C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_*_x64__8wekyb3d8bbwe\`.
- **Pins:** winget speichert Pins **pro Benutzer** in
  `%LOCALAPPDATA%\Packages\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe\LocalState\pinning.db`
  (SQLite). SYSTEM sieht sie nicht. Zurückgehalten wird deshalb:
  - was erst mit `--include-pinned` auftaucht,
  - was `winget pin list` als SYSTEM zeigt,
  - die Pins aus der `pinning.db` **jedes** Profils,
  - `winget_pin_ids` (Liste exakter IDs aus Inventory oder Extra-Var).

  Gelesen wird die SQLite-Datei mit `winsqlite3.dll` per Add-Type/P-Invoke, an
  einer Kopie inklusive `-wal`/`-shm`. Im Bericht erscheint
  `[GEPINNT] … (Pin von <user>)`.
- Das Ziel: Jeder Entwickler setzt seine Pins selbst (`winget pin add`), und
  der Lauf respektiert sie.

### `update-windows.yml` (Windows-Server-Updates mit Protokoll)
Für Server 2016 bis 2025. Einstellungen stehen in `vars/windows-update.yml`.

Vier Plays, alle mit `vars_files: [vars/windows-update.yml]`:

1. **Vorbereitung** (alle Hosts parallel)
   - Erreichbarkeit prüfen.
   - `host-info` → `winupd_info_before`.
   - Updates suchen → `winupd_pending`, `winupd_needed`.
   - Mit `group_by` in `winupd_snapshot` bzw. `winupd_no_snapshot` einsortieren.
2. **Snapshots**
   - `hosts: winupd_snapshot`, `serial: winupd_snapshot_parallel` (Standard 2).
   - Snapshot `WSU_<JJJJ-MM-TT>` mit RAM.
3. **Installation** (alle Hosts parallel)
   - Steht ein Neustart aus, zuerst neu starten.
   - Danach Runden über `tasks/windows/update-round.yml`:
     - **Runde 0:** nur `winupd_first` (Servicing Stack / Wartungsstapel). Die
       müssen vor allem anderen drauf sein.
     - **Runde 1 und folgende:** alles aus `win_update_categories`, Neustart
       wenn nötig, erneut suchen. Höchstens `winupd_max_rounds` Runden, bis
       nichts mehr kommt. Manche Updates werden erst nach einem Neustart
       angeboten.
   - Patchende erfassen und Critical-/Warning-/Error-Ereignisse aus
     Application und System mit Meldung sammeln.
   - Dann Rescan, `host-info` → `winupd_info_after` und das Protokoll aus
     `templates/windows-update-protocol.j2`.
   - Nach erfolgreichem Patch die Snapshot-Loeschung im vCenter vormerken.
4. **Versand** (localhost): Protokolle und gegebenenfalls Ereignisse in
   getrennten Mails.

Bei einem echten Patchlauf wird zusaetzlich die Patchzeit in UTC festgehalten.
Danach zaehlt das Playbook Critical-, Warning- und Error-Ereignisse aus
Application und System fuer genau diesen Zeitraum, unter Ausschluss der
bekannten Event-IDs, und verschickt Zaehler sowie Ereignismeldungen in einer
zweiten Mail.
Testlaeufe, manuelle Hosts, Offline-Hosts und Hosts ohne Updates bekommen
keinen Ereignisblock; ohne gepatchte Hosts entfaellt die zweite Mail. Die sechs
manuell auszufuellenden Protokollzeilen bleiben unveraendert leer.

Nach erfolgreichem Patch wird die Loeschung des konkreten Snapshots als
einmalige vCenter-Aufgabe geplant. `winupd_snapshot_retention_hours` ist per
Extra-Var/Survey ueberschreibbar (Standard 24, `0` deaktiviert die Loeschung).
Bei Updatefehlern oder ausstehendem manuellem Neustart bleibt der Snapshot
erhalten. Das vCenter-Konto benoetigt `ScheduledTask.Create`,
`ScheduledTask.Edit`, `ScheduledTask.Delete`, `System.View` und
`VirtualMachine.State.RemoveSnapshot`. Eine bereits geplante Loeschung wird
direkt nach dem Ermitteln des Snapshots und vor den Patchschritten aufgehoben;
scheitert das, wird der Patch blockiert. Bei Patchfehlern bleibt der Snapshot
ohne automatische Loeschung erhalten.

Ohne Snapshot werden keine Updates installiert. Ein Fehler landet in
`winupd_error`; die folgenden Plays fassen den Host dann nicht mehr an, das
Protokoll entsteht trotzdem.

Pro Host im Inventory:
- `winupd_manual=true`: nur der leere Manuell-Block, sonst nichts.
- `winupd_reboot=false`: kein Neustart, im Protokoll steht „manueller Neustart
  noetig“.
- `vm_name=…`: wenn die VM im vCenter anders heißt als der Host.
- `snapshot_required=false`: Host ohne Snapshot ueberspringen und als Problem
  melden; ohne Snapshot wird nicht gepatcht.

#### Protokollformat
Das Format entspricht den bisherigen Patch-Protokollen auf `\\fs5`; Vorlage
waren ein echtes Protokoll und eine Beispieldatei mit allen Varianten.

Normaler Server:

```
patch02(Windows Server 2022 Standard):
 ToDo: Create Snapshot 'WSU_2026-11-16' = OK
 ToDo: Updates(3) Installation gestarted 11/16/2026 20:01:12 = OK
       - KB5066782 1GB 2026-11 Kumulatives Update …
       - KB5065432 12MB …
       - [Nachlauf 11/16/2026 20:40:02] KB… (ab Runde 2)
       - [FEHLER 0x80240022] KB…           (gescheitert und später nicht nachgeholt)

 ToDo: Neustart von 11/16/2026 20:31:00 bis 11/16/2026 20:35:44 = OK
 ToDo: Rescan intern (0) = OK
 ToDo: Festplattenspeicher 42GB = OK
 ToDo: Versionsupdate von 20348.4171 zu 20348.4294 = OK
 ToDo: WSUS Status prüfen = 
 ToDo: Logs > Anwendung = 
 ToDo: Logs > System = 
 ToDo: Abmelden = 
 ToDo: VMTools aktualisiert = 
 ToDo: Snapshot löschen = 
```

Regeln dazu:
- Die **letzten sechs Punkte bleiben leer** (`= ` mit Leerzeichen dahinter).
  Dort trägt das Team von Hand Datum und Uhrzeit ein.
- Alle übrigen Punkte bekommen `OK`, `FEHLER …`, `Testlauf`,
  `nicht noetig …`, `PRUEFEN …` usw.
- Festplatte unter `winupd_min_free_gb` (16 GB) → `FEHLER (unter 16GB)`.
- Wurde ein Update installiert, der Build ist aber unverändert →
  `PRUEFEN (Build unveraendert)`.
- Uhrzeiten im Format `MM/dd/yyyy HH:mm:ss`, so wie im Original.
- Bei `winupd_manual` erscheint `[MANUELL] Dieser Server muss manuell gepatcht werden.`
  und alle Punkte bleiben leer.
- In der Mail trennt eine Linie aus 74 Bindestrichen die Server.

### `report-user-installs.yml`
Liest nur, ändert nichts. Findet Programme, die nur für einen Benutzer
installiert sind (`winget install --scope user`). winget als SYSTEM sieht die
nicht, `update-winget.yml` aktualisiert sie also nie.
- Liest den Uninstall-Zweig in HKU jedes Profils. Bei abgemeldeten Benutzern
  wird `NTUSER.DAT` per `reg load` geladen und danach mit GC und `reg unload`
  wieder entladen.
- Die **winget-ID** kommt aus der `installed.db` des Profils (SQLite, gleiche
  Technik wie bei den Pins). Die Abfrage verknüpft
  `productcodes_map → productcodes → manifest → ids` und ordnet so den
  Registry-Schlüssel der ID zu.
- **WiX-Burn-Bundles** (z.B. Python) tragen sich immer in den HKCU dessen ein,
  der den Installer gestartet hat, auch bei `--scope machine`. Erkannt werden
  sie an `BundleCachePath` / `BundleProviderKey`. Gibt es in HKLM einen Eintrag
  mit gleicher Version und gleichem Namensanfang, wird das Bundle als
  „OK, ausgeblendet“ gemeldet.
- OneDrive und Teams sind per `user_install_ignore` ausgenommen.
- Ergebnis:
  - Mail mit Übersicht nach winget-ID und einer Liste pro Host.
  - CSV im Anhang (Semikolon, UTF-8 mit BOM, für Excel) mit fertigen Befehlen:
    `winget uninstall --id X --exact --scope user; winget install --id X --exact --scope machine`
    und einer Spalte „Erledigt“ zum Abhaken.
- Damit es nicht wieder passiert: winget-Einstellungen
  (`installBehavior.preferences.scope = machine`) per GPO verteilen, Quellen
  über `DesktopAppInstaller.admx` (siehe README).

---

## 4. Wiederverwendbare Bausteine (`tasks/`)

Eingebunden wird **immer mit vollem Pfad**:
`ansible.builtin.include_tasks: "{{ playbook_dir }}/tasks/<bereich>/<datei>.yml"`.
Jede Datei beschreibt im Kopf ihre Eingaben, Optionen und Ergebnisse.

| Datei | Eingabe | Ergebnis |
|---|---|---|
| `common/remember-run-hosts.yml` | – | `report_hosts` auf localhost (für das Abschluss-Play) |
| `common/check-reachable.yml` | optional `reach_timeout` (5) | `host_offline` (nur bei „Timeout when waiting“; Port 5985 für winrm/psrp, sonst 22) |
| `common/send-report-mail.yml` | `report_subject`, `report_body`, optional `report_attach` | Mail, sofern `mail_to` gesetzt |
| `common/fail-on-problems.yml` | `report_hosts`, `report_results[].problem` | Lauf wird **nach** der Mail rot |
| `windows/host-info.yml` | – | `win_host_info` {os, version `Release.UBR`, free_gb, pending_reboot, now} |
| `windows/host-time.yml` | – | `win_host_now` (Ortszeit), `win_host_now_utc` |
| `windows/collect-event-logs.yml` | `winupd_log_start_utc`, `winupd_log_end_utc` | Critical/Warning/Error-Zaehler und Ereignismeldungen |
| `windows/search-updates.yml` | `win_update_categories` | `win_update_search` {updates[{id,kb,title,size,categories}], now} |
| `windows/reboot-logged.yml` | optional `win_reboot_timeout` (3600) | hängt {from,to} an `win_reboots` |
| `windows/update-round.yml` | `round_idx`, `winupd_*` | eine Runde suchen, installieren, neu starten |
| `windows/user-installs.yml` | optional `user_install_ignore` | `win_user_installs` {installs, profiles, errors, bundles} |
| `vmware/create-snapshot.yml` | `snapshot_name`, `snapshot_description`, `vm_user`, `vm_password` | Snapshot oder Abbruch |
| `vmware/schedule-snapshot-cleanup.yml` | Snapshot-MOIDs, `snapshot_name`, `winupd_snapshot_retention_hours`, vCenter-Zugang | einmalige automatische Loeschung nach erfolgreichem Patch |

### vCenter-Snapshot ohne pyvmomi
Alles läuft über `ansible.builtin.uri` mit `delegate_to: localhost`:
1. `POST /api/session` (Basic Auth, Antwort 201, Token im JSON-Body)
2. `GET /api/vcenter/vm`; die VM wird über den Namen gesucht (Kurzname oder
   FQDN, Groß-/Kleinschreibung egal)
3. `DELETE /api/session`
4. `POST /sdk/vim25/8.0.1.0/SessionManager/SessionManager/Login` → Header
   `vmware_api_session_id`
5. `GET VirtualMachine/{moid}/snapshot`; gibt es den Namen schon, wird kein
   zweiter Snapshot angelegt
6. `POST VirtualMachine/{moid}/CreateSnapshot_Task` mit
   `{name, description, memory, quiesce:false}`
7. `GET Task/{id}/info` abfragen, bis `success` oder `error` (120 × 5 s)
8. Im `always`-Block: `Logout`

Die Anmelde-Tasks laufen mit `no_log: true`.

---

## 5. Wo welche Variable hingehört

Das ist die häufigste Fehlerquelle, deshalb hier ausführlich.

| Art | Wo | Beispiele |
|---|---|---|
| Pro Host/Gruppe | **Inventory** (Semaphore) | `winupd_manual`, `winupd_reboot`, `vm_name`, `snapshot_required`, `winget_pin_ids`, `user_install_ignore`, `ansible_connection=winrm` |
| Pro Lauf | **Survey** des Templates (Bool-Schalter als Enum, Aufbewahrung als Integer) | `winget_dry_run`, `apt_dry_run`, `winupd_dry_run`, `target_hosts`, `winupd_snapshot_parallel`, `winupd_snapshot_retention_hours` |
| Zugangsdaten / Mail / vCenter | **Variablengruppe** (Environment), Passwörter als **Secret** | `mail_host`, `mail_to`, `mail_from`, `mail_cc`, `mail_bcc`, `mail_port`, `mail_secure`, `mail_username`, `mail_password`, `vm_user`, `vm_password`, `vcenter_hostname`, `vcenter_validate_certs` |
| Feste Einstellungen eines Playbooks | `vars/<playbook>.yml` per `vars_files` | `winupd_min_free_gb`, `winupd_first`, `win_update_categories` |

Präzedenz, die man kennen muss:
- **Play-`vars` und `vars_files` schlagen Inventory-Variablen.** Deshalb nie
  eine Variable, die pro Host gesteuert wird, als Play-Variable oder in
  `vars_files` setzen. Der Standard kommt erst bei der Verwendung:
  `{{ winupd_reboot | default(true) | bool }}`.
- **Extra-Vars schlagen alles.** Survey-Werte und Variablengruppen kommen in
  Semaphore als Extra-Vars an.
- `serial` wird **einmal pro Play** ausgewertet und kennt keine Hostvariablen,
  nur Extra-Vars oder `vars_files`.
- Ein Template bindet genau **ein** Inventory und **eine** Variablengruppe.

---

## 6. Regeln und Konventionen

### Code
- **FQCN** für alle Module: `ansible.builtin.*`, `ansible.windows.*`,
  `community.general.*`.
- **Kommentare auf Deutsch.** In YAML-Dateien nur ASCII (`ae`, `oe`, `ue`,
  `ss`). In README, HANDOVER und Protokoll-Template sind Umlaute erlaubt, und
  das Protokoll **braucht** sie („prüfen“, „löschen“).
- **Testlauf ist immer der Standard** (`*_dry_run: true`). Scharf läuft nur,
  wer es bewusst auswählt.
- **Schalter immer mit `| bool` auswerten.** Semaphore liefert Survey-Werte
  als Text, und `"false"` ist für Jinja wahr.
- Jinja liefert nativ `None`, wenn ein Template leer rendert. Für Texte deshalb
  `default('', true) | trim`.
- Pro-Host-Variablen nie als Play-Variable setzen (siehe Abschnitt 5).
- Wiederkehrende Schritte gehören nach `tasks/<bereich>/` mit sprechendem
  Namen und Kopfkommentar (Eingabe / Optional / Ergebnis).
- Bei `delegate_to: localhost` zeigt `ansible_host` auf den Delegierten. Für
  den echten Host also `hostvars[inventory_hostname].ansible_host` nehmen.
- `include_tasks` mit `loop` wertet ein `when` am Include für alle Elemente
  **vorab** aus. Soll der Abbruch während der Schleife greifen
  (`winupd_done`), muss `when` auf einen Block **in** der eingebundenen Datei.
- Keywords an eingebundene Tasks weitergeben: `apply:` (z.B.
  `ignore_errors`, `ignore_unreachable`).
- Kein `run_once` in bedingten Blöcken. Es greift für alle Hosts, auch wenn
  der Block für den ersten Host übersprungen wird.
- In Jinja-Templates keine Listen mit `.append` verändern, sondern
  `namespace()` nehmen und Zeilen sammeln. Ansible-Templates laufen mit
  `trim_blocks`.
- Mehrzeilige Mailtexte als `|-`, nicht `>-`: `>-` faltet die
  Zeilenumbrüche weg.
- In `win_powershell` liefert eine Funktion Objekte zurück. Kein
  `$script:`-Scope, der ist dort unzuverlässig.
- Keine zusätzlichen Python-Abhängigkeiten und **keine `requirements.yml`**.
  Collections kommen aus dem Image, sonst zieht Semaphore vor jedem Lauf die
  neueste Fassung. Wenn doch einmal nötig: Versionen festnageln. Deshalb läuft
  vCenter per `uri` und SQLite per `winsqlite3.dll`.
- winget immer **als SYSTEM** und mit vollem Pfad aufrufen. Als SYSTEM gibt es
  kein `winget` im PATH.

### Fehler und Berichte
- **Offline ist kein Fehler.** `check-reachable.yml` setzt `host_offline`, und
  der Host erscheint im Bericht in der Offline-Liste.
- Probleme pro Host sammeln (`*_error`, `*_problem`). Erst schickt das
  Abschluss-Play die Mail, dann lässt `fail-on-problems.yml` den Lauf
  scheitern. Die Mail kommt also **immer**.
- Kritischer Pfad (Snapshot, Installation, Neustart) läuft in
  `block`/`rescue`, und der Fehler landet im Protokoll. Kein pauschales
  `ignore_errors` auf dem kritischen Pfad.
- **Kein Patch ohne Snapshot.**

### Netzwerk
- Hosts und vCenter **als FQDN** (`*.gutingia.local`). `NO_PROXY` vergleicht
  per Suffix. Ein Kurzname geht an den Proxy, und WinRM oder vCenter scheitern
  dann.
- Der Proxy wird nur per `SEMAPHORE_FORWARDED_ENV_VARS` an die Tasks
  durchgereicht (README → Proxy).

### Sicherheit
- Passwörter (`mail_password`, `vm_password`, Proxy-Login) nur als **Secret**
  in der Semaphore-Variablengruppe, nie im Repo oder im Klartext.
- Vault-Passwortdateien nie ins Repo; `.gitignore` deckt sie ab.
- Anmelde-Tasks mit `no_log: true`.

### Git und Dokumentation
- Commit-Nachricht auf Deutsch, Form `datei: was sich ändert`, z.B.
  `update-windows.yml: hoechstens winupd_snapshot_parallel Snapshots gleichzeitig`.
- Fertige Änderungen direkt committen und nach `stable` pushen. Branches
  kündigt der Verantwortliche selbst an.
- **README.md immer mitpflegen**: neue Playbooks, Variablen, Bausteine und
  den Abschnitt „Aufbau“.
- Shell-Befehle in der Doku: ein Befehl pro Codeblock.

---

## 7. Stolperfallen, die schon passiert sind

| Symptom | Ursache | Lösung |
|---|---|---|
| Gepinntes Programm wird trotzdem aktualisiert | Pins gelten pro Benutzer, SYSTEM sieht sie nicht | alle `pinning.db` lesen und `winget_pin_ids` beachten |
| `winget_problem` ist `None` statt leer | natives Jinja | `default('', true) \| trim` |
| Mailtext in einer Zeile | `>-` faltet Zeilen | `\|-` |
| `pyvmomi` fehlt (Task #80) | `vmware.vmware.vm_snapshot` braucht es | REST und VI/JSON per `uri` |
| Snapshots liefen nacheinander | `serial` bildet Gruppen in Inventory-Reihenfolge, auch mit Hosts ohne Snapshot | `group_by` → Play nur über `winupd_snapshot` |
| Inventory-Wert wird ignoriert | Play-Variable schlägt Inventory | Standard erst per `default()` bei der Verwendung |
| Schleife bricht nicht ab | `when` am `include_tasks`-Loop wird vorab ausgewertet | `when` auf einen Block in der Datei |
| Mehrere Empfänger brechen den Absender | `mail_from` fiel auf den ganzen `mail_to` zurück | `mail_rcpt \| first` |
| Python als Benutzer-Installation gemeldet | Burn-Bundle steht im HKCU des Installierenden | Bundle-Abgleich mit HKLM |
| Test-Path schreibt Fehler | Profil ohne Zugriff | `-ErrorVariable`, Eintrag in `errors` |
| Task kommt nicht ins Internet, `docker exec` schon | Semaphore reicht seine Umgebung nicht an Tasks weiter | `SEMAPHORE_FORWARDED_ENV_VARS` |
| `FileNotFoundError` in `unfrackpath` | Volume `/playbooks` fehlt | Volume eintragen, Container neu erzeugen |
| Pull: `Read-only file system` | Mount mit `:ro` | ohne `:ro` mounten, `chown -R 1001:0` |

---

## 8. Testen

Unter Windows gibt es lokal kein Ansible. Bisher wurde so geprüft:

- **YAML parsen:** Python mit PyYAML (`yaml.safe_load` auf jede Datei).
- **Jinja-Templates** (Protokoll, Mailtext, CSV): mit Jinja2 und Testdaten
  rendern und mit den Beispielprotokollen vergleichen. Ansible-Filter wie
  `bool` oder `regex_replace` dafür nachbilden.
- **PowerShell aus `win_powershell`:** den `script:`-Teil herausziehen und mit
  Windows PowerShell 5.1 parsen, ohne ihn auszuführen. Das ist dieselbe
  Version wie auf den Servern.
  ```powershell
  [System.Management.Automation.Language.Parser]::ParseFile($pfad, [ref]$null, [ref]$err); $err
  ```
  Lesende Teile (Registry, SQLite) lassen sich lokal als Admin auch ausführen.
- **In Semaphore:** pushen, das Pull-Template starten, dann das
  `[TEST]`-Template, zuerst mit `*_dry_run=true`. Für Snapshots und Neustarts
  die PATCH-Testserver nehmen.

---

## 9. Offen / Ideen

Noch nicht umgesetzt, nur auf Zuruf angehen:

- **Dienste stoppen und starten** in `update-windows.yml`, wie im alten
  `update-windows-server.yml`. Die alte Liste in `windows_patch_services`:
  Spooler, CogonCalculationService, CogonCommonService,
  CogonCommunicationService, CogonMonitoringProcess,
  CogonPathMonitoringService, CogonReportingService, CogonServiceManager,
  CogonShadowAccountingService, glpi-agent. Starten im `always`-Block, damit
  sie nach einem Abbruch nicht gestoppt bleiben.
- **Protokoll direkt auf den Share** `\\fs5\IT-Protokolle\<Jahr>\<Monat>\Serverpatch\`
  schreiben statt nur per Mail. Dafür braucht der Container den CIFS-Mount
  (README → Volumes).
- Inventory aus dem **AD** ableiten.
- `PATCH01` hat nur 15 GB frei, das Protokoll zeigt deshalb korrekt `FEHLER`.
- Prüfen, ob winget als SYSTEM ein Python erkennt und aktualisiert, dessen
  Bundle im HKCU eines Admins steht:
  ```powershell
  psexec -s -i powershell
  ```
  ```powershell
  $w = Get-ChildItem "$env:ProgramFiles\WindowsApps\Microsoft.DesktopAppInstaller_*_x64__8wekyb3d8bbwe\winget.exe" | Select-Object -Last 1; & $w list --id Python.Python.3.15 --accept-source-agreements
  ```
- Eigenes Linux-Dienstkonto `ansible` mit Key im Key Store (README →
  Linux-Gruppen).
- `infra6`: winget-Fehler `0xC0000135`, vermutlich fehlt die
  VC++-Laufzeit.
- winget-`settings.json` per GPO-Einstellungen verteilen
  (`scope: machine`); Quellen über `DesktopAppInstaller.admx`.

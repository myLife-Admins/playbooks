# ansible-assets

Patch-Playbooks für die monatlichen Serverupdates, betrieben über Semaphore.

## Stand dieses Branches

`stable` enthält derzeit `update-apt.yml`, `update-winget.yml`,
`update-windows.yml` und `pull-playbooks.yml`, dazu gemeinsame Bausteine unter
`tasks/` und `templates/`. Der Commit `clear` (1092d36) hat
den übrigen Bestand entfernt: `ansible.cfg`, `inventory/`, `requirements.yml`
und beide Produktions-Playbooks samt `tasks/`. Verloren ist davon nichts, der
Stand davor liegt in `482ad68` und lässt sich gezielt zurückholen:

```bash
git checkout 482ad68 -- ansible.cfg inventory playbooks
```

`requirements.yml` bleibt bewusst draußen, siehe [Collections](#collections).

Die Kapitel [Zugangsdaten](#zugangsdaten), [Fehlerstrategie](#fehlerstrategie)
und [Fortschritt sehen](#fortschritt-sehen) beschreiben `update-windows-server.yml`.
Sie gelten unverändert — nur liegen die Dateien dazu gerade nicht im Branch.

## Setup

Unter Semaphore ist nichts zu installieren: das Image bringt das volle
`ansible`-Paket mit, und damit alle Collections, die hier gebraucht werden —
siehe [Collections](#collections). Für einen lokalen Lauf mit bloßem
`ansible-core`:

```bash
ansible-galaxy collection install ansible.windows vmware.vmware ansible.posix
```

`ansible.cfg` zeigt auf `inventory/hosts`, die Playbooks sind also aus dem
Repo-Wurzelverzeichnis heraus ohne `-i` aufrufbar. `update-apt.yml` braucht
weder das eine noch das andere.

Dort, wo Ansible läuft, muss `{{ protokoll_basis }}` (Standard `/mnt/protokolle`)
als CIFS-Share `//fs5/it/Protokolle` gemountet sein — dort landen die
Patchprotokolle unter `<Jahr>/<Monat>/Serverpatch/`.

## Playbooks

### `pull-playbooks.yml` — Ordner aktualisieren

Eigenes Template, das `git pull --ff-only` im Playbook-Ordner ausführt. Damit
holt man neue Stände aus dem Git, ohne sich auf den Docker-Host einzuloggen.

Läuft gegen `localhost`, also **im Task-Container** — und das ist hier genau
richtig: dessen `/playbooks` ist per Volume derselbe Ordner wie
`/opt/docker/semaphore/playbooks` auf dem Host. Ein Inventory wird nicht
benutzt.

Drei Voraussetzungen, die das Playbook vorab prüft und sonst mit klarer Meldung
abbricht:

- `/playbooks` ist ein **Klon**, keine Kopie der Dateien. Einmalig auf dem Host:
  `git clone https://github.com/myLife-Admins/playbooks.git /opt/docker/semaphore/playbooks`
- Das Volume ist **ohne `:ro`** gemountet. Ein schreibgeschützter Mount ist
  sonst eine sinnvolle Absicherung, verträgt sich aber nicht mit dem Pull —
  git meldet dann `cannot open '.git/FETCH_HEAD': Read-only file system`.
- Der Ordner gehört dem Container-Benutzer (`uid=1001`, `gid=0`). Nach einem
  `sudo git clone` gehört alles root, und der Pull scheitert:
  `sudo chown -R 1001:0 /opt/docker/semaphore/playbooks`
- Die Proxy-Variablen hängen am Container, siehe [Proxy](#proxy).

`safe.directory` gibt das Playbook bei jedem Aufruf mit, statt es global zu
setzen — git verweigert sonst mit „dubious ownership", und eine Konfiguration
im Container überlebt `docker compose up -d` nicht.

`--ff-only` ist Absicht: gibt es lokale Änderungen im Ordner, bricht der Pull
ab, statt sie wegzuräumen oder einen Merge-Commit zu bauen.

Von Hand aktualisieren geht über den Container — dort existiert Benutzer 1001,
und die Proxy-Variablen sind gesetzt. Auf dem Host gibt es UID 1001 nicht,
`sudo -u '#1001'` scheitert dort mit „unknown user", und `sudo git` würde die
Dateien wieder root übereignen:

```bash
sudo docker exec semaphore git -C /playbooks pull
```

### `update-apt.yml` — Linux / APT

Liegt im Wurzelverzeichnis, weil das Semaphore-Template genau diesen Pfad
erwartet. Zielgruppe ist `{{ target_hosts | default('all') }}` — ohne gesetzte
Variable also alles, was die Semaphore-Inventory enthält.

**Standardmäßig ändert das Playbook nichts.** `apt_dry_run` steht auf `true`:
Paketlisten werden aktualisiert, das Upgrade läuft nur im `check_mode`, ein
nötiger Reboot wird gemeldet statt ausgeführt. Für den echten Lauf:

```bash
ansible-playbook update-apt.yml -e apt_dry_run=false
```

Im Playbook wird der Schalter überall mit `| bool` ausgewertet. Per `-e` oder
Survey kommt er als Text an, und der Text `"false"` ist für Ansible wahr —
ohne den Filter würde das Upgrade stillschweigend übersprungen.

Ablauf: Debian/Ubuntu prüfen → Paketlisten aktualisieren → anstehende Updates
per `check_mode` ermitteln → Upgrade (nur ohne dry run) →
`/var/run/reboot-required` prüfen → Zusammenfassung.

Rebootet wird nie, auch nicht mit `apt_dry_run=false` — das unterscheidet es
vom alten `playbooks/update-apt.yml`. Es benutzt ausschließlich
`ansible.builtin.*` und braucht deshalb keine Collection aus Galaxy, was es
hinter dem Proxy als Erstes lauffähig macht.

`become: true` setzt passwortloses sudo für den SSH-Benutzer voraus, sonst
zusätzlich ein Become-Passwort im Key Store.

Warum `all` und nicht `linux_server`: heißt die Gruppe in der Inventory
irgendwann anders, matcht ein fester Gruppenname nichts, und Ansible quittiert
das mit „skipping: no hosts matched" und Exit-Code 0 — ein Lauf, der nichts tut
und trotzdem grün aussieht.

### `update-winget.yml` — Windows / winget

Aktualisiert alle Programme, die winget kennt. Wie bei `update-apt.yml` wird
standardmäßig nur gelistet, was anstünde. Für den echten Lauf:

```bash
ansible-playbook update-winget.yml -e winget_dry_run=false
```

Rebootet wird nie. Quelle ist nur das Community-Repository `winget` — die
Store-Quelle braucht Store-Zugang und scheitert auf Servern.

**Bericht pro Paket.** Statt `upgrade --all` holt das Playbook erst die Liste
und aktualisiert dann jedes Paket einzeln mit `--id <id> --exact`. Das ergibt
dasselbe, aber mit einem Exit-Code pro Paket — und damit einem Status, der
nicht davon abhängt, in welcher Sprache winget antwortet (als SYSTEM auf einem
deutschen Server: deutsch). Im Log steht danach etwa:

```
aktualisiert: 1 | nicht bestaetigt: 0 | fehlgeschlagen: 1 | nicht anwendbar: 0 | uebersprungen: 0 | gepinnt: 1
[OK]         7zip.7zip                                26.05.00.0 -> 26.06
[FEHLER]     Mozilla.Firefox.de                       131.0 -> 132.0  0x8A150101  Programm laeuft gerade
Gepinnt - werden nur von Hand aktualisiert:
[GEPINNT]    angryziber.AngryIPScanner                3.9.3 -> 3.10.0
```

| Status | Bedeutung |
|---|---|
| `anstehend` | nur gelistet (dry run) |
| `aktualisiert` | Exit-Code 0, und die Nachkontrolle listet das Paket nicht mehr |
| `nicht bestaetigt` | Exit-Code 0, aber das Paket steht nach der Nachkontrolle noch in der Liste |
| `nicht anwendbar` | `0x8A15002B` — winget fand für dieses System kein passendes Update |
| `fehlgeschlagen` | jeder andere Code; die letzten fünf Zeilen der winget-Ausgabe stehen dabei |
| `uebersprungen` | ID in der Liste abgeschnitten (`…`), nicht eindeutig aufrufbar |

**Nachkontrolle.** Manche Installer kehren sofort zurück und installieren im
Hintergrund weiter — winget meldet dann Erfolg, bevor das Programm wirklich
neu ist. Nach den Updates listet das Playbook deshalb erneut, alle 20 s, bis
die aktualisierten Pakete verschwunden sind, höchstens `winget_verify_seconds`
lang (Standard 180). Erst danach geht die Mail raus. Was dann noch dasteht,
wird `nicht bestaetigt`: entweder noch nicht fertig, oder winget erkennt die
neue Version nicht — das gibt es bei einzelnen Paketen dauerhaft. Es ist eine
Warnung, kein Fehler; der Lauf wird davon nicht rot. Steht ein Paket bei jedem
Lauf so da, ist es Letzteres.

**Gepinnte Pakete.** Pins heißen „nur von Hand aktualisieren", und das
Playbook fasst sie nicht an. Damit sie trotzdem nicht aus dem Blick geraten,
stehen die mit verfügbarem Update in einem eigenen Abschnitt. Erkannt werden
sie auf vier Wegen: am Unterschied zwischen `winget upgrade` und
`winget upgrade --include-pinned`, an `winget pin list` (SYSTEM), an den Pins
**aller Benutzerprofile** am Host und an der Variable `winget_pin_ids`.

Hintergrund: winget speichert Pins pro Benutzer in
`%LOCALAPPDATA%\Packages\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe\LocalState\pinning.db`.
Das Playbook läuft als SYSTEM und liest diese Datei in jedem Profil (über die
`winsqlite3.dll`, die Windows ab 10 / Server 2016 mitbringt). Damit gilt:
Jeder Benutzer pinnt selbst, ganz normal mit `winget pin add --id <id>`, und
das Paket wird auf diesem Rechner nicht mehr automatisch aktualisiert. Im
Bericht steht, wer gepinnt hat (`(Pin von adminjs)`). Ist eine Pin-Datei nicht
lesbar, steht das als Hinweis im Bericht.

Feste Ausnahmen für ganze Gruppen gehen zusätzlich übers Inventory:

```ini
[windows_server:vars]
winget_pin_ids=['7zip.7zip', 'Mozilla.Firefox']
```

Exakte Ids wie in der Spalte „ID" von `winget upgrade`. Nicht als Play-Variable
im Playbook setzen — die würde das Inventory überstimmen.

**Offline-Hosts sind kein Fehler.** Vor winget prüft das Playbook kurz, ob der
WinRM-Port antwortet (`winget_reach_timeout`, Standard 5 s, direkt aus dem
Container). Wer nicht antwortet — ausgeschaltet, Notebook unterwegs, nicht mehr
im DNS —, gilt als offline: kein Verbindungsfehler im Log, kein Problem, der
Lauf wird davon nicht rot. In der Mail stehen diese Hosts gesammelt in einer
Zeile am Ende, im Betreff als „N offline". Antwortet der Port, aber WinRM
scheitert danach (falsches Passwort, Proxy statt Server), ist das weiterhin
ein Problem.

Probleme brechen einen Host nicht ab, sondern werden gesammelt: ein
fehlgeschlagenes Paket, ein WinRM-Fehler, fehlendes oder nicht startendes
winget, ein Nicht-Windows-Host im Inventory. Rot wird der Lauf erst ganz am Ende, nach dem
Mailversand — sonst käme die Mail ausgerechnet dann nicht, wenn etwas
schiefgeht. Alle Ergebnisse liegen zusätzlich strukturiert in
`winget_run.result.packages` (Name, ID, Versionen, Status, Code, Hinweis);
daraus wird später das Protokoll geschrieben.

**Bericht per Mail.** Ein zweites Play sammelt die Berichte aller Hosts und
schickt sie als eine Mail, sofern `mail_to` gesetzt ist. Am bequemsten als
Variablengruppe in Semaphore, dann für jedes Template wiederverwendbar:

```json
{
  "mail_host": "smtp.example.local",
  "mail_to": "it@example.de",
  "mail_from": "semaphore@example.de"
}
```

Optional `mail_port` (Standard 25), `mail_secure` (Standard `try`),
`mail_username` und `mail_password` — Letzteres als Secret der Variablengruppe,
nicht im Klartext. Ohne `mail_from` ist der Empfänger zugleich Absender.

Der Betreff fasst zusammen, der Text enthält die Berichte pro Host:

```
[winget] 2 Host(s): 2 aktualisiert, 1 fehlgeschlagen - 2 mit Problem
```

Gesendet wird aus dem Semaphore-Container. SMTP geht nicht über den Proxy,
der Container braucht also direkten Zugang zum Mailserver auf Port 25. Und der
Mailserver muss Mails von der IP des Docker-Hosts annehmen: Exchange-Relays
lassen anonymes Senden meist nur von freigegebenen Adressen zu. Scheitert
STARTTLS am Zertifikat des internen Servers, `mail_secure: never` setzen.

Die Tabelle von `winget upgrade` wird über die Positionen der Überschriften
gelesen, nicht über deren Text. Mit `--source` hat sie vier Spalten (Name, ID,
Version, Verfügbar), ohne fünf. Passt das Format nicht, liest das Playbook
lieber gar nichts als falsch und gibt die Rohausgabe aus. Pakete mit
unbekannter Version überspringt winget selbst; das steht als Hinweis im
Bericht.

**Warum als SYSTEM.** winget ist eine App-Paket-Anwendung (App Installer). In
einer WinRM-Sitzung fehlt der Aufruf-Alias `winget`, und den Ordner
`C:\Program Files\WindowsApps`, in dem die `winget.exe` liegt, darf nicht
einmal ein Administrator auflisten. SYSTEM darf das. Das Playbook wechselt
deshalb per `become_method: runas` zu SYSTEM — ohne Passwort, das braucht es
für SYSTEM nicht — und ruft die neueste installierte `winget.exe` über den
vollen Pfad auf.

Voraussetzungen am Zielhost:

- **winget vorhanden.** Windows 10/11 und Server 2025 ja, Server 2022 nur mit
  nachinstalliertem App Installer, Server 2019 gar nicht. Fehlt es, bricht
  das Playbook mit genau dieser Meldung ab.
- **Internetzugang.** Als SYSTEM gelten die Proxy-Einstellungen angemeldeter
  Benutzer nicht. Kommt winget nicht an seine Quelle, ist das der erste Verdacht.

Inventory — der Host als **FQDN**. WinRM geht über `requests` und damit über
den Proxy, sobald der Name nicht unter `NO_PROXY` fällt (siehe [Proxy](#proxy)).
Ein Kurzname wie `infra04` landet beim Proxy statt beim Server:

```
[windows]
infra04.gutingia.local

[windows:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_port=5985
ansible_winrm_server_cert_validation=ignore
ansible_winrm_operation_timeout_sec=1000
ansible_winrm_read_timeout_sec=1200
```

Die beiden Timeouts stammen aus dem alten Inventory, das damit die langen
Windows-Update-Läufe abgesichert hat. Ein winget-Lauf mit vielen Paketen und
Nachkontrolle dauert ebenfalls Minuten.

Zugangsdaten für einen Testhost am einfachsten als „Login with Password" im
Key Store, als User Credentials ans Inventory gehängt — Benutzer als
`GUTINGIA\name` oder `name@gutingia.local`. Sudo Credentials leer lassen. Für
mehrere Hosts mit unterschiedlichen Konten siehe [Zugangsdaten](#zugangsdaten).

Das winget-eigene Protokoll eines Laufs als SYSTEM liegt auf dem Zielhost unter
`C:\Windows\System32\config\systemprofile\AppData\Local\Packages\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe\LocalState\DiagOutputDir`.

Für die Codes `0x8A150014`, `0x8A15002B`, `0x8A150101`, `0x8A150109` und
`0xC0000135` (fehlende DLL, meist die VC++-Laufzeit) steht ein Hinweistext im
Bericht. Der Text ist nur Erläuterung — ob ein Paket als Erfolg zählt,
entscheidet allein der Code.

### `update-windows.yml` — Windows-Updates mit Patch-Protokoll

Installiert Windows-Updates auf Servern 2016 bis 2025 und schreibt pro Server
einen Block im Format der Patch-Protokolle unter `\\fs5\IT-Protokolle`.
Vorerst geht das Protokoll per Mail raus (Einstellungen wie bei winget).

**Standard ist ein Testlauf** (`winupd_dry_run: true`): suchen und listen,
kein Snapshot, keine Installation, kein Neustart. Scharf mit
`winupd_dry_run=false` (Survey-Variable, Typ Enum).

Ablauf pro Server:

1. Snapshot `WSU_<Datum>` im vCenter — nur wenn Updates anstehen. Ohne
   Snapshot keine Updates. Höchstens `winupd_snapshot_parallel` (Standard 2)
   Snapshots laufen gleichzeitig.
2. Steht schon ein Neustart aus, zuerst neu starten.
3. **Runde 0:** nur die Updates aus `winupd_first` (Servicing Stack). Die
   müssen vor allem anderen drauf sein, sonst scheitert der Rest.
4. **Runde 1:** alles Übrige, Neustart wenn nötig.
5. **Nachlauf:** suchen → installieren → neu starten, bis nichts mehr kommt
   (höchstens `winupd_max_rounds`). Manche Updates werden erst nach einem
   Neustart angeboten; sie stehen im Protokoll als `[Nachlauf <Zeit>]`.
6. Rescan, Version und freier Platz auf C:.

Im Protokoll füllt das Playbook Snapshot, Updates, Neustart, Rescan,
Festplattenspeicher (unter `winupd_min_free_gb` = 16 GB → FEHLER) und
Versionsupdate. WSUS-Status, Logs, Abmelden, VMTools und Snapshot löschen
bleiben leer — dort tragt ihr wie bisher Uhrzeit und Datum ein. Server mit
`winupd_manual=true` bekommen den Leerblock `[MANUELL]`. Offline-Server stehen
am Ende als Liste. Die Vorlage liegt in `templates/windows-update-protocol.j2`.

Zeiten (`Installation gestarted`, `Neustart von … bis …`) sind die Uhrzeit am
Server, nicht die des Semaphore-Containers.

Das Playbook besteht aus vier Plays: Vorbereitung (alle Server parallel),
Snapshots (`serial: winupd_snapshot_parallel`), Installation (wieder parallel)
und Versand. Ein `throttle` am Snapshot-Task reichte nicht: damit würden zwar
nur zwei Snapshots gleichzeitig *angestoßen*, aber alle, bevor auf den ersten
gewartet wird. Mit `serial` kommt die nächste Gruppe erst dran, wenn alle
Snapshots der aktuellen fertig sind.

Einstellungen für den ganzen Lauf stehen in `vars/windows-update.yml` und
werden per Extra-Var oder Survey überschrieben — nicht im Inventory, dafür
haben `vars_files` Vorrang:

| Variable | Standard | Wirkung |
|---|---|---|
| `winupd_dry_run` | `true` | Testlauf, nur suchen und listen |
| `winupd_snapshot_parallel` | `2` | höchstens so viele Snapshots gleichzeitig |
| `winupd_min_free_gb` | `16` | darunter FEHLER bei Festplattenspeicher |
| `winupd_max_rounds` | `4` | Nachlauf-Runden nach Runde 0 |
| `winupd_first` | Servicing Stack | Updates, die vor allen anderen kommen |

Variablen pro Host oder Gruppe im Inventory:

| Variable | Wirkung |
|---|---|
| `winupd_manual=true` | wird von Hand gepatcht, nur Leerblock |
| `winupd_reboot=false` | kein automatischer Neustart, Protokoll: „manueller Neustart noetig" |
| `vm_name=<Name>` | VM-Name im vCenter, falls weder Kurzname noch FQDN |
| `snapshot_required=false` | Host ohne VM (physisch) |

vCenter-Zugang wie im alten `update-windows-server.yml` als Variablengruppe:
`vm_user`, `vm_password` (Secret), optional `vcenter_hostname`
(Standard `vcenter1.gutingia.local`) und `vcenter_validate_certs` (Standard
`false`). Alles läuft über die HTTP-APIs des vCenters mit `ansible.builtin.uri`:
die VM wird per Automation-API (`/api`) über den Namen gefunden, der Snapshot
per VI/JSON-API (`/sdk/vim25`, ab vCenter 8.0 Update 1). Bewusst nicht über
`vmware.vmware.vm_snapshot`: das braucht die Python-Bibliothek `pyvmomi`, die
im Semaphore-Image fehlt — und nachinstalliert wäre sie beim nächsten
Image-Update wieder weg. Gibt es den Snapshot mit dem Namen schon (zweiter
Lauf am selben Tag), wird kein zweiter angelegt. `vcenter_hostname` muss als
FQDN unter `NO_PROXY` fallen.

Noch nicht drin, was das alte Playbook konnte: Dienste vor dem Patchen stoppen
und danach starten (`windows_patch_services`) und das Schreiben direkt auf den
Share.

### `update-windows-server.yml` — Windows (alt)

*Derzeit nicht im Branch, siehe [Stand](#stand-dieses-branches).*

Patcht die per `target_hosts` übergebene Gruppe. Zugangsdaten für Domäne und
vCenter kommen aus den verschlüsselten `vault.yml` (siehe [Zugangsdaten](#zugangsdaten)),
das Playbook prüft sie im ersten Task.

Ablauf pro Host: VM-Snapshot über vCenter → Dienste stoppen → Windows Updates
installieren → Reboot → Versions- und Plattenplatz-Check → Dienste starten.
Das Protokoll wird im `always`-Block in jedem Fall geschrieben.

```bash
ansible-playbook playbooks/update-windows-server.yml -e target_hosts=windows_upla
```

**Resume:** Nach dem Snapshot wird unter `{{ resume_state_dir }}` ein Marker
`patch_resume_<host>.txt` abgelegt. Läuft ein Durchgang auf einen Fehler,
kann er nach der Behebung erneut gestartet werden, ohne dass ein zweiter
Snapshot erzeugt wird. Bei Erfolg löscht sich die Datei selbst.

## Betrieb über Semaphore UI

Semaphore läuft als Container auf einer eigenen VM. Daraus folgen ein paar
Dinge, die in den Playbooks bereits berücksichtigt sind:

**`localhost` ist der Task-Container, nicht der Controller.** `update-apt.yml`
hatte früher einen zweiten Play, mit dem sich der Controller selbst patchte —
unter Semaphore hätte der versucht, den Container zu patchen und neu zu
starten. Der Play ist weg; der bisherige Controller gehört stattdessen unter
seinem echten Hostnamen nach `[linux_server]`.

**Kein `vars_prompt`.** Semaphore ruft `ansible-playbook` nicht-interaktiv auf.
Die Zugangsdaten kommen jetzt von außen, das Playbook prüft sie beim Start und
bricht mit klarer Meldung ab, wenn eine fehlt.

**`delegate_to: localhost`** — alle vCenter-Aufrufe und das Schreiben der
Protokolle laufen im Container. Der CIFS-Share muss dort also gemountet sein.

### Repository: gemounteter Ordner statt Git

Das Semaphore-Repository ist **kein** Git-Remote, sondern der lokale Pfad
`/playbooks`, den ein Volume in den Container reicht. Semaphore klont dann
nichts, es liest das Verzeichnis. Aktualisiert wird es über das Template
[`pull-playbooks.yml`](#pull-playbooksyml--ordner-aktualisieren).

Angefangen hat das als Notlösung, weil der Container nicht an GitHub kam. Seit
der Proxy durchgereicht wird, ginge auch Git direkt. Der Ordner bleibt
trotzdem, und zwar **als Puffer für den Fall, dass das Internet weg ist**:

- Bei einem Git-Repository zieht Semaphore vor **jedem** Task den aktuellen
  Stand. Ist GitHub nicht erreichbar, bricht der Task ab, bevor das Playbook
  startet — auch ein rein interner Job: ein Patchlauf gegen den WSUS, eine
  VM aus vCenter, das Einsammeln von Protokollen.
- Der Klon dafür liegt in `/tmp` im Container und ist nach jedem Neuerzeugen
  weg. Es gibt also nicht einmal einen alten Stand, auf den man zurückfallen
  könnte.
- Mit dem Ordner scheitert in dem Fall nur der Pull. Alles andere läuft mit
  dem zuletzt gezogenen Stand weiter.

Nebeneffekt: Änderungen landen erst, wenn jemand den Pull anstößt. Ein Commit
auf `stable` verändert also nicht stillschweigend den nächsten Lauf.

Erkennbar ist der Ordner-Betrieb im Task-Log: dort stehen Zeilen wie
`No /playbooks/requirements.yml file found`, und eine Zeile „Updating
repository" fehlt. Bei einem Git-Repository läge das Arbeitsverzeichnis unter
`/tmp/semaphore/repository_<id>_<template>`.

Fehlt das Volume, stirbt Ansible schon beim Import:

```
File "ansible/utils/path.py", line 51, in unfrackpath
    b_basedir = to_bytes(os.getcwd(), errors='surrogate_or_strict')
FileNotFoundError: [Errno 2] No such file or directory
```

Das ist keine Aussage über das Playbook — `os.getcwd()` scheitert, weil das
Arbeitsverzeichnis des Prozesses nicht existiert.

Einrichten — der Ordner ist ein Git-Klon und gehört dem Container-Benutzer:

```bash
sudo git clone https://github.com/myLife-Admins/playbooks.git /opt/docker/semaphore/playbooks
```

```bash
sudo chown -R 1001:0 /opt/docker/semaphore/playbooks
```

Volume eintragen (siehe [Volumes](#volumes)), dann den Container **neu
erzeugen** — `docker restart` übernimmt neue Volumes nicht:

```bash
docker compose up -d
```

```bash
docker exec semaphore ls -la /playbooks
```

Auf RHEL/Rocky/Alma braucht der Mount ein `:z`, sonst sieht der Container den
Ordner leer — auf Debian/Ubuntu nicht. Ein `:ro` dagegen darf nicht dran, sonst
scheitert der Pull mit `Read-only file system`.

### Collections

Es liegt **bewusst keine `requirements.yml`** im Repo. Das Semaphore-Image
enthält das vollständige `ansible`-Paket (13.5.0), darin stecken rund hundert
Collections — unter anderem die drei, um die es hier geht:

| Collection | im Image | wofür |
|---|---|---|
| `ansible.windows` | 3.5.0 | `win_updates`, `win_shell`, `win_powershell` |
| `vmware.vmware` | 2.8.0 | derzeit ungenutzt — braucht `pyvmomi`, das im Image fehlt; Snapshots laufen über `uri` |
| `ansible.posix` | 2.1.0 | `profile_tasks` / `timer` Callbacks |

Grund für das Weglassen: Semaphore ruft den Galaxy-Install **vor jedem Task**
auf. Steht dort eine `requirements.yml` ohne Versionsangabe, zieht er jedes Mal
die neueste Fassung nach `~/.ansible/collections`, und die hat Vorrang vor der
im Image. Der Patchlauf im Dezember liefe dann womöglich mit einer anderen
`ansible.windows` als der getestete im November — unangekündigt, mitten im
Wartungsfenster. Ohne die Datei ist der Stand an das Image gebunden und
wechselt nur, wenn ihr das Image wechselt.

Wird doch einmal eine Collection gebraucht, die nicht dabei ist: `requirements.yml`
anlegen **und Versionen festnageln**, sonst handelt man sich die Drift wieder ein.

### Proxy

Der Container kommt nur über den Proxy nach draußen — Git, Galaxy und alles
weitere. Die Variablen am Container zu setzen reicht dafür **nicht**:
Semaphore gibt seine eigene Umgebung nicht an Tasks weiter. Welche Variablen
durchgereicht werden, muss man ausdrücklich mit `SEMAPHORE_FORWARDED_ENV_VARS`
festlegen:

```yaml
services:
  semaphore:
    environment:
      HTTP_PROXY: http://10.98.98.222:8080
      HTTPS_PROXY: http://10.98.98.222:8080
      NO_PROXY: localhost,127.0.0.1,db,gutingia.local,.gutingia.local
      http_proxy: http://10.98.98.222:8080
      https_proxy: http://10.98.98.222:8080
      no_proxy: localhost,127.0.0.1,db,gutingia.local,.gutingia.local
      SEMAPHORE_FORWARDED_ENV_VARS: '["HTTP_PROXY","HTTPS_PROXY","NO_PROXY","http_proxy","https_proxy","no_proxy"]'
```

Die Falle dabei: ein Test per `docker exec semaphore …` sieht die Variablen,
weil er direkt in der Container-Umgebung läuft — ein Task sieht sie ohne
`SEMAPHORE_FORWARDED_ENV_VARS` nicht. Das Symptom im Task-Log ist ein direkter
Verbindungsversuch, den die Firewall nach gut zwei Minuten verwirft:

```
Failed to connect to github.com port 443 after 134101 ms: Could not connect to server
```

Der Proxy steht als IP drin, damit er nicht von der DNS-Konfiguration des
Containers abhängt.

Beide Schreibweisen sind kein Versehen: `ansible-galaxy` liest die Umgebung
über urllib, `pywinrm` und `vmware.vmware` über `requests` — die werten Groß-
und Kleinschreibung unterschiedlich aus.

**Achtung bei Windows und vCenter.** Mit gesetztem `HTTPS_PROXY` schickt
`requests` auch WinRM-Verbindungen und vCenter-Aufrufe durch den Proxy; SSH zu
den Linux-Hosts ist nicht betroffen, das kennt die Variablen nicht. `NO_PROXY`
hilft dabei nur bedingt: Hostnamen werden per **Suffix** verglichen, eine
CIDR-Angabe wie `10.0.0.0/8` greift ausschließlich bei nackten IP-Adressen. Auf
Kurznamen wie `glpi-01` oder `fs5` passt damit weder das eine noch das andere.
Entweder die Inventory auf FQDNs umstellen und `.gutingia.local` eintragen,
oder die Kurznamen einzeln listen — vCenter nicht vergessen.

Zum führenden Punkt: der ist weniger wichtig, als es aussieht, aber die beiden
Bibliotheken behandeln ihn unterschiedlich.

| | `urllib` (ansible-galaxy) | `requests` (WinRM, vCenter) |
|---|---|---|
| Vergleich | schneidet den Punkt ab, prüft `(.+\.)?domain$` | stumpfes `endswith` |
| `gutingia.local` | `dc-1.gutingia.local` ✓, Domäne selbst ✓ | beides ✓, dazu `xgutingia.local` ✓ |
| `.gutingia.local` | identisch zur Zeile darüber | Subdomains ✓, Domäne selbst ✗ |

Für `ansible-galaxy` ist es also egal. Nur `requests` sieht einen Unterschied,
und der betrifft ausschließlich einen Host, der exakt `gutingia.local` heißt.
Am saubersten beide Formen eintragen — `gutingia.local,.gutingia.local` — und
vor allem in `NO_PROXY` und `no_proxy` **denselben** Wert, sonst hängt das
Verhalten davon ab, welche Bibliothek gerade fragt.

Verlangt der Proxy Authentifizierung, gehört `http://user:pass@proxy:3128`
nicht im Klartext in die compose-Datei, sondern als Secret in eine
Semaphore-Environment (im Template das Feld „Variablengruppen"). Sonderzeichen
im Passwort URL-kodieren, `@` also als `%40`.

### Einrichtung

| Semaphore-Objekt | Inhalt |
|---|---|
| Repository | Lokaler Pfad `/playbooks` — gemountetes Volume, nicht Git |
| Inventory | Typ **File** mit Pfad `inventory/hosts`, oder statisch in Semaphore gepflegt |
| Key Store → SSH Key | Eigener ed25519-Key für das Dienstkonto `ansible`, an das Inventory hängen (Linux) |
| Key Store → Ansible Vault | Das Vault-Passwort, an das Template hängen — damit werden die `vault.yml` entschlüsselt |
| Environment („Variablengruppen") | Mail-Einstellungen, Proxy-Variablen sofern nicht container-weit gesetzt |
| Task Template → Survey Variable | Schalter pro Lauf, siehe [Schalter per Survey](#schalter-per-survey) |

Die Windows-Zugangsdaten hängen bewusst **nicht** als „Login with Password" am
Inventory — siehe [Zugangsdaten](#zugangsdaten). Semaphore bindet dort ein
einziges Konto ans ganze Inventory; die lokalen Konten bräuchten dann eigene
Templates.

### Schalter per Survey

Variablen, die sich von Lauf zu Lauf ändern, gehören nicht ins Playbook und
nicht in die Variablengruppe, sondern als **Survey-Variable** ins Template
(„Survey-Variablen" → „+ Variable hinzufügen"). Semaphore fragt sie dann bei
jedem Start ab, mit einem voreingestellten Wert.

| Variable | Typ | Werte | Standard | Template |
|---|---|---|---|---|
| `winget_dry_run` | Auswahl (enum) | `true`, `false` | `true` | `update-winget.yml` |
| `apt_dry_run` | Auswahl (enum) | `true`, `false` | `true` | `update-apt.yml` |
| `target_hosts` | Auswahl (enum) | Gruppennamen aus dem Inventory | — | alle |

Standard `true` heißt: Ein versehentlicher Klick listet nur, und „wirklich
installieren" ist eine bewusste Auswahl.

Semaphore übergibt Survey-Werte als **Text**. Für Ansible ist der Text
`"false"` wahr — die Playbooks werten die Schalter deshalb überall mit
`| bool` aus. Wer neue Schalter dieser Art einbaut, muss das genauso machen,
sonst wird aus „false" stillschweigend „true".

Für zeitgesteuerte Läufe, bei denen niemand etwas auswählt, besser ein eigenes
Template mit festem CLI-Argument, etwa `--extra-vars=winget_dry_run=false`.

### Linux-Gruppen im Inventory

```
[linux_server:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
```

**Zum Interpreter.** Ohne die Zeile rät Ansible bei jeder Verbindung, welches
Python auf dem Ziel zu nehmen ist, und warnt entsprechend. Das Raten ist nicht
stabil: kommt später eine weitere Version dazu, kann die Erkennung dorthin
springen — und dann fehlt dem `apt`-Modul `python3-apt`, das nur für den
System-Python der Distribution installiert ist. Der Fehler lautet dann
„Could not import python modules: apt, apt_pkg" und sieht nach einem Problem
am Playbook aus. `/usr/bin/python3` ist der Symlink der Distribution und zeigt
nach einem Upgrade von selbst auf die richtige Version; eine harte Angabe wie
`/usr/bin/python3.12` überlebt das nächste Release-Upgrade nicht.

Alternativ ließe sich die Warnung mit `interpreter_python = auto_silent` in der
`ansible.cfg` stummschalten — das behebt aber nur die Meldung, nicht die
Ursache.

**Zum Konto.** Eigenes Dienstkonto `ansible` auf den Zielhosts, eigener
Schlüssel, der ausschließlich in Semaphores Key Store liegt, und
passwortloses sudo:

```bash
echo 'ansible ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible && sudo chmod 440 /etc/sudoers.d/ansible && sudo visudo -c
```

`NOPASSWD` auf einzelne Befehle einzuschränken funktioniert mit Ansible nicht:
Module laufen nicht als `apt`, sondern als `/usr/bin/python3 /tmp/…/AnsiballZ_apt.py`,
und dieser Pfad ist bei jedem Lauf ein anderer. Die Eingrenzung muss deshalb
woanders herkommen — eigenes Konto, `from="<semaphore-ip>"` in der
`authorized_keys`, Schlüssel nur im Key Store.

### Volumes

Der Container braucht zwei Mounts: den Playbook-Ordner und den
Protokoll-Share. Ohne letzteren schlägt jedes Logging fehl und die
Resume-Marker landen im Nichts:

```yaml
services:
  semaphore:
    volumes:
      - /opt/docker/semaphore/playbooks:/playbooks
      - /mnt/protokolle:/mnt/protokolle
```

Auf dem Docker-Host also vorher den CIFS-Mount einrichten. Die Resume-Marker
liegen bewusst unter `{{ protokoll_basis }}/ansible-state` und nicht in `/tmp`:
ein flüchtiges `/tmp` würde dazu führen, dass bei jedem Lauf stillschweigend
ein neuer Snapshot gezogen wird.

`become: yes` in `update-apt.yml` braucht passwortloses sudo für `ladmin` —
sonst zusätzlich ein Become-Passwort im Key Store hinterlegen.

## Zugangsdaten

Die Mehrheit der Hosts läuft über den Domänen-Admin, ein paar über lokale
Konten. Das braucht **kein** zweites Task-Template: Ansible löst `ansible_user`
und `ansible_password` pro Host über die Variablenpräzedenz auf.

Die Kette, von schwach nach stark:

```
inventory/group_vars/windows/vault.yml            Domänen-Admin  (gilt für alle)
  └─ inventory/group_vars/windows_notdomain/…     lokales Konto  (schlägt das Elternteil)
       └─ inventory/host_vars/<host>.yml          Einzelfall     (schlägt alles)
```

`[windows_notdomain]` ist ein Kind von `[windows]`, und Kind-Gruppen
überschreiben Eltern-Gruppen. Für jede weitere Gruppe mit eigenem Konto
genügt ein Verzeichnis `inventory/group_vars/<gruppe>/` mit einer `vault.yml`.

### Anlegen

Neben jeder Stelle liegt eine `.example`-Datei mit der erwarteten Struktur.
Die echten Dateien legst du selbst an — verschlüsselt, mit deinen Werten:

```bash
ansible-vault create inventory/group_vars/windows/vault.yml
```

```bash
ansible-vault create inventory/group_vars/windows_notdomain/vault.yml
```

Bearbeiten mit `ansible-vault edit <datei>`, ansehen mit `ansible-vault view`.
Die verschlüsselten `vault.yml` gehören ins Git — sie sind verschlüsselt. Das
Vault-Passwort selbst gehört in Semaphores Key Store und **niemals** ins Repo;
die `.gitignore` deckt die üblichen Dateinamen dafür ab.

### Warum nicht Semaphores Key Store

Ein „Login with Password"-Eintrag hängt am Inventory und setzt damit *ein*
Konto für alle Hosts darin. Gemischte Konten hieße: pro Kontotyp ein eigenes
Inventory und ein eigenes Template — also genau die Aufteilung, die man
vermeiden will. Über `group_vars` bleibt es ein Template.

Der Key Store wird trotzdem gebraucht: für den SSH-Key von `ladmin` und für
das Vault-Passwort.

### Sonderfall myrootca1

Der Host steht in `[windows_ad]` **und** `[windows_notdomain]` — Geschwister
auf derselben Ebene. Bei gleichrangigen Gruppen entscheidet Ansible nicht nach
Bedeutung, sondern nach Alphabet; das ergibt hier zufällig das richtige
Ergebnis, ist aber nirgends so gemeint und kippt beim nächsten Umbenennen.

`inventory/host_vars/myrootca1.yml` legt es explizit fest, weil host_vars über
allen Gruppenvariablen stehen. Sauberer wäre, die Gruppenzugehörigkeit zu
bereinigen — entweder ist die Maschine domänengebunden oder nicht. Dann kann
die Datei entfallen.

## Fehlerstrategie

Leitgedanke: **es wird immer ein Protokoll geschrieben**, auch wenn der Lauf
scheitert. Das leistet der `always:`-Block — dafür braucht es kein
`ignore_errors`.

Früher stand `ignore_errors: yes` auf fast jedem Task. Dadurch erreichte nie
ein Fehler den `rescue:`-Block, `patch_error` blieb leer, und das Protokoll
meldete `= OK`, obwohl das Update fehlgeschlagen war. Die beiden Mechanismen
haben sich gegenseitig neutralisiert.

Jetzt gilt:

| Kategorie | Behandlung | Beispiel |
|---|---|---|
| Kritischer Pfad | kein `ignore_errors` → `rescue` fängt, Fehler steht im Protokoll | Snapshot, `win_updates`, Reboot |
| Diagnose | `failed_when: false`, Ergebnis wird „(nicht ermittelt)" | Versionsabfrage, Plattenplatz |
| Dienste | `ignore_errors` bleibt, Fehler werden gesammelt und protokolliert | Stoppen/Starten |

Die Dienste werden im `always:`-Block gestartet, nicht am Ende des `block:`.
Sonst blieben sie nach einem Abbruch dauerhaft gestoppt — `rescue` überspringt
den Rest des Blocks. Ihr `ignore_errors` muss bleiben: ein Fehler im
`always:`-Block würde diesen abbrechen und damit genau das Protokoll
verhindern, um das es geht.

### Kein Patch ohne Snapshot

Findet der Lauf die VM nicht in vCenter, bricht er für diesen Host ab.
Vorher wurde in dem Fall **ohne Rollback-Punkt weitergepatcht**, und das
Protokoll erwähnte es mit keinem Wort.

Für Hosts ohne VM in diesem vCenter (physische Maschinen, Fremd-vCenter) pro
Gruppe `snapshot_required: false` setzen, z.B. in
`inventory/group_vars/windows_notdomain.yml`.

Der Resume-Marker wird außerdem erst geschrieben, wenn der Snapshot wirklich
durch ist. Vorher wurde er unbedingt gesetzt — schlug der Snapshot fehl,
meldete der nächste Lauf „snapshot_done" und patchte ohne Rollback-Punkt.

## Fortschritt sehen

Standardmäßig ist Ansibles Ausgabe knapp. Die `ansible.cfg` stellt deshalb um:

| Einstellung | Effekt |
|---|---|
| `callback_result_format = yaml` | Task-Ergebnisse lesbar statt einzeiliges JSON |
| `display_skipped_hosts = False` | Übersprungene Tasks raus — sonst viel Rauschen |
| `ansible.posix.profile_tasks` | Startzeit und Dauer **pro Task**, am Ende die langsamsten |
| `ansible.posix.timer` | Gesamtlaufzeit |
| `forks = 10` | Default ist 5, kleiner als die meisten Patchgruppen |

`profile_tasks` ist dabei der eigentliche Gewinn: im Semaphore-Log steht damit
zu jedem Task, wann er begann und wie lange er schon läuft — man sieht also,
wo ein Lauf gerade steht und ob er hängt.

Den `yaml`-Callback aus `community.general` gibt es nicht mehr; ansible-core
kann das seit 2.13 über `callback_result_format` selbst.

### Ausgabe von mehreren Hosts

`update-windows-server.yml` nutzt `strategy: free`: jeder Host läuft unabhängig
durch. Das ist schnell, aber die Ausgaben mehrerer Hosts verschränken sich.
Nimmt man die Zeile heraus, gilt der Standard `linear` — Ansible arbeitet Task
für Task und gibt pro Task einen Block mit allen Hosts aus. Viel besser
nachvollziehbar, dafür wartet nach jedem Task der schnellste Host auf den
langsamsten. Eine Zeile, jederzeit umschaltbar.

### Optionale stdout-Callbacks

Davon ist immer nur **einer** aktiv (im Gegensatz zu `profile_tasks`/`timer`,
die als aggregate-Callbacks nebenherlaufen). Alle in `community.general`:

- `counter_enabled` — hängt Zähler an („Task 5 von 23")
- `unixy` — kompakte, prosanahe Ausgabe
- `timestamp` — Zeitstempel an jeden Kopf
- `dense` — minimale, sich überschreibende Fortschrittsanzeige

Nicht aktiviert, weil keiner davon gebraucht wird. `community.general` (12.5.0)
steckt im Image, nachzuinstallieren ist also nichts — es genügt
`stdout_callback = community.general.counter_enabled` in der `ansible.cfg`.

### Was Ansible nicht kann

`win_updates` ist **ein** Task. Während er 40 Minuten lang zwölf Updates
installiert, gibt Ansible nichts aus — erst am Ende das Gesamtergebnis. Die
Granularität „welches Update läuft gerade", die das PowerShell-Skript hatte,
gibt es hier nicht, weil die Schleife im Modul steckt und nicht im Playbook.

`profile_tasks` zeigt immerhin, dass der Task seit 40 Minuten läuft. Für mehr
Detail kennt `ansible.windows.win_updates` die Option `log_path`, die ein
ausführliches Protokoll **auf dem Windows-Host** mitschreibt — gut für die
Nachbetrachtung, aber nicht im Semaphore-Log sichtbar.

## Aufbau

Im Branch:

```
update-apt.yml                  Linux-Programme (apt), dry run als Standard
update-winget.yml               Windows-Programme (winget), dry run als Standard
update-windows.yml              Windows-Updates + Patch-Protokoll, dry run als Standard
pull-playbooks.yml              git pull im Playbook-Ordner
tasks/
  common/
    remember-run-hosts.yml      Hosts des Laufs fuer das Abschluss-Play merken
    check-reachable.yml         Port-Check vorab -> host_offline
    send-report-mail.yml        Bericht per Mail (report_subject/report_body)
    fail-on-problems.yml        Lauf am Ende rot, wenn es Probleme gab
  windows/
    host-info.yml               OS, Version (Release.UBR), freier Platz, Neustart aus?
    host-time.yml               Uhrzeit am Host im Protokollformat
    search-updates.yml          Updates suchen (WUA, mit Groesse)
    reboot-logged.yml           Neustart mit Beginn/Ende -> win_reboots
    update-round.yml            eine Runde suchen/installieren/neu starten
  vmware/
    create-snapshot.yml         Snapshot ueber vCenter, VM per Name
templates/
  windows-update-protocol.j2    Protokollblock eines Servers
vars/
  windows-update.yml            Einstellungen fuer update-windows.yml
README.md
.gitignore
```

Jeder Baustein beschreibt im Kopf, was er erwartet und was er hinterlässt.
Eingebunden wird mit `include_tasks: "{{ playbook_dir }}/tasks/…"` — der volle
Pfad, weil Ansible relative Pfade in verschachtelten Includes zuerst neben der
einbindenden Datei sucht.

Nach `git checkout 482ad68 -- …` (siehe [Stand](#stand-dieses-branches)) kommt
der volle Bestand zurück:

```
ansible.cfg                     Inventory-Pfad, Ausgabe-Callbacks, forks
inventory/
  hosts                         Hostgruppen + Verbindungsparameter
  group_vars/
    all.yml                     protokoll_basis, resume_state_dir
    windows/
      main.yml                  Dienstliste, vCenter, Schwellwerte
      vault.yml                 Domänen-Admin (verschlüsselt, selbst anlegen)
    windows_notdomain/
      vault.yml                 lokales Konto (verschlüsselt, selbst anlegen)
  host_vars/
    myrootca1.yml               Einzelfall, siehe Zugangsdaten
playbooks/
  update-apt.yml                ältere Fassung mit Reboot, siehe Playbooks
  update-windows-server.yml
  tasks/
    linux/    dist-upgrade.yml, write-log.yml
    windows/  get-services.yml, stop-services.yml,
              start-services.yml, write-patchlog.yml
```

Die Trennung ist bewusst: **Inventory** beantwortet „wie erreiche ich den Host",
**group_vars** beantwortet „was mache ich mit ihm".

Dass `update-apt.yml` im Wurzelverzeichnis liegt und die übrigen Playbooks
unter `playbooks/`, ist kein Konzept, sondern der Pfad, auf den das Template
„myPatchTest" zeigt. Beim Zusammenführen also entweder das Template
nachziehen oder die Datei verschieben.

## Inventory

Alle Windows-Gruppen hängen als `children` unter `[windows]`. Die
WinRM-Parameter stehen dadurch einmal in `[windows:vars]` statt pro Gruppe.

**Eine neue Windows-Gruppe muss unter `[windows:children]` eingetragen werden**,
sonst fehlen ihr sowohl die Verbindungsparameter als auch die Dienstliste aus
`group_vars/windows.yml`.

Die Dienstliste lässt sich pro Gruppe überschreiben — z.B. legt
`inventory/group_vars/windows_ad.yml` mit eigenem `windows_patch_services`
fest, was auf den DCs gestoppt wird.

### Bekannte Unsauberkeiten

- `[test]` sind die Patch-Testmaschinen, `[windows_test]` ist die Testumgebung.
- `myrootca1` steht in `windows_ad` **und** `windows_notdomain`. Nicht
  domänengebundene Hosts brauchen lokale Credentials, die das Playbook aktuell
  nicht getrennt behandelt.
- `fs2` steht in `windows_produktion` und `windows_fileserver` — beide Gruppen
  im selben Monat patchen heißt zwei Durchläufe inkl. zweitem Snapshot.
- Die Protokollnamen leiten sich aus `target_hosts.split('_')[1]` ab. Bei
  `[test]` (kein Unterstrich) fällt das auf den Hostnamen zurück und erzeugt
  Einzelprotokolle statt eines Gruppenprotokolls.

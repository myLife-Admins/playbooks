# ansible-assets

Patch-Playbooks für die monatlichen Serverupdates, betrieben über Semaphore.

## Stand dieses Branches

`stable` enthält derzeit nur `update-apt.yml`. Der Commit `clear` (1092d36) hat
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

### `update-windows-server.yml` — Windows

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

Wir liegen hinter einem Proxy, der Container kommt nicht an GitHub. Das
Semaphore-Repository ist deshalb **kein** Git-Remote, sondern der lokale Pfad
`/playbooks`, den ein Volume in den Container reicht. Semaphore klont dann
nichts, es liest das Verzeichnis.

Erkennbar ist das im Task-Log: dort stehen Zeilen wie
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
Arbeitsverzeichnis des Prozesses nicht existiert. Einrichten:

```bash
sudo mkdir -p /opt/semaphore/playbooks
```

Volume eintragen (siehe [Volumes](#volumes)), dann den Container **neu
erzeugen** — `docker restart` übernimmt neue Volumes nicht:

```bash
docker compose up -d
```

```bash
docker exec semaphore ls -la /playbooks
```

Zwei Stolpersteine: Der Prozess im Container ist nicht root, bei „Permission
denied" hilft `chmod -R a+rX` auf dem Host. Und auf RHEL/Rocky/Alma braucht der
Mount ein `:z`, sonst sieht der Container den Ordner leer — auf Debian/Ubuntu
nicht.

Der Ordner und GitHub wissen nichts voneinander, Änderungen müssen von Hand
hinein. Kommt der Docker-Host (anders als der Container) über den Proxy an
GitHub, ist ein `git clone` in dieses Verzeichnis bequemer; Semaphore liest
weiterhin nur den Ordner.

### Collections

Es liegt **bewusst keine `requirements.yml`** im Repo. Das Semaphore-Image
enthält das vollständige `ansible`-Paket (13.5.0), darin stecken rund hundert
Collections — unter anderem die drei, um die es hier geht:

| Collection | im Image | wofür |
|---|---|---|
| `ansible.windows` | 3.5.0 | `win_updates`, `win_shell`, `win_powershell` |
| `vmware.vmware` | 2.8.0 | Snapshots über vCenter |
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
weitere. Die Variablen gehören an den Container, damit sie sowohl einen
etwaigen Galaxy-Schritt als auch das Playbook erreichen:

```yaml
services:
  semaphore:
    environment:
      HTTP_PROXY: http://proxy:8080
      HTTPS_PROXY: http://proxy:8080
      NO_PROXY: localhost,127.0.0.1,gutingia.local,.gutingia.local
      http_proxy: http://proxy:8080
      https_proxy: http://proxy:8080
      no_proxy: localhost,127.0.0.1,gutingia.local,.gutingia.local
```

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
| Key Store → SSH Key | Key für `ladmin`, an das Inventory hängen (Linux) |
| Key Store → Ansible Vault | Das Vault-Passwort, an das Template hängen — damit werden die `vault.yml` entschlüsselt |
| Environment („Variablengruppen") | Proxy-Variablen, sofern nicht container-weit gesetzt |
| Task Template → Survey Variable | `target_hosts`, am besten als Dropdown mit den Gruppennamen |

Die Windows-Zugangsdaten hängen bewusst **nicht** als „Login with Password" am
Inventory — siehe [Zugangsdaten](#zugangsdaten). Semaphore bindet dort ein
einziges Konto ans ganze Inventory; die lokalen Konten bräuchten dann eigene
Templates.

### Volumes

Der Container braucht zwei Mounts: den Playbook-Ordner und den
Protokoll-Share. Ohne letzteren schlägt jedes Logging fehl und die
Resume-Marker landen im Nichts:

```yaml
services:
  semaphore:
    volumes:
      - /opt/semaphore/playbooks:/playbooks
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
update-apt.yml                  Linux-Patchlauf, Pfad des Semaphore-Templates
README.md
.gitignore
```

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

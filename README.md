# ansible-assets

Patch-Playbooks für die monatlichen Serverupdates. Zwei Einstiegspunkte, sonst nichts.

## Setup

```bash
ansible-galaxy collection install -r requirements.yml
```

`ansible.cfg` zeigt auf `inventory/hosts`, die Playbooks sind also aus dem
Repo-Wurzelverzeichnis heraus ohne `-i` aufrufbar.

Dort, wo Ansible läuft, muss `{{ protokoll_basis }}` (Standard `/mnt/protokolle`)
als CIFS-Share `//fs5/it/Protokolle` gemountet sein — dort landen die
Patchprotokolle unter `<Jahr>/<Monat>/Serverpatch/`.

## Playbooks

### `playbooks/update-apt.yml` — Linux / APT

Patcht die Gruppe `linux_server`. Der bisherige Controller gehört dort mit
rein — unter Semaphore patcht er sich nicht mehr selbst über `localhost`.

Ablauf pro Host: Log schreiben → `apt upgrade dist` → Reboot falls
`/var/run/reboot-required` existiert → `autoremove` / `autoclean`.

```bash
ansible-playbook playbooks/update-apt.yml
```

### `playbooks/update-windows-server.yml` — Windows

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

### Einrichtung

| Semaphore-Objekt | Inhalt |
|---|---|
| Repository | Dieses Repo als Git-Remote (Semaphore klont per Git) |
| Inventory | Typ **File**, Pfad `inventory/hosts` |
| Key Store → SSH Key | Key für `ladmin`, an das Inventory hängen (Linux) |
| Key Store → Ansible Vault | Das Vault-Passwort, an das Template hängen — damit werden die `vault.yml` entschlüsselt |
| Task Template → Survey Variable | `target_hosts`, am besten als Dropdown mit den Gruppennamen |

Die Windows-Zugangsdaten hängen bewusst **nicht** als „Login with Password" am
Inventory — siehe [Zugangsdaten](#zugangsdaten). Semaphore bindet dort ein
einziges Konto ans ganze Inventory; die lokalen Konten bräuchten dann eigene
Templates.

Collections aus `requirements.yml` installiert Semaphore selbst — prüfen, ob
deine Version den Pfad im Repo-Wurzelverzeichnis findet.

### Volumes

Der Container braucht den Protokoll-Share. Ohne ihn schlägt jedes Logging fehl
und die Resume-Marker landen im Nichts:

```yaml
services:
  semaphore:
    volumes:
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

Nicht aktiviert, weil `community.general` eine sehr große Collection ist, die
sonst nirgends gebraucht wird. Bei Bedarf in `requirements.yml` aufnehmen und
`stdout_callback = community.general.counter_enabled` in der `ansible.cfg`
setzen.

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

```
ansible.cfg                     Inventory-Pfad, Ausgabe-Callbacks, forks
requirements.yml                ansible.windows, vmware.vmware, ansible.posix
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
  update-apt.yml
  update-windows-server.yml
  tasks/
    linux/    dist-upgrade.yml, write-log.yml
    windows/  get-services.yml, stop-services.yml,
              start-services.yml, write-patchlog.yml
```

Die Trennung ist bewusst: **Inventory** beantwortet „wie erreiche ich den Host",
**group_vars** beantwortet „was mache ich mit ihm".

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

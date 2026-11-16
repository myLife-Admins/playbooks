# Copilot-Anweisungen für dieses Repo

Ansible-Patch-Playbooks (Linux apt, Windows winget, Windows-Server-Updates mit
vCenter-Snapshot), betrieben über **Semaphore UI** in Docker hinter einem
Proxy. Vor größeren Änderungen **HANDOVER.md** lesen (Hintergrund, Regeln,
bekannte Fallen); Bedienung und Einrichtung stehen in **README.md**.

Antworte und kommentiere auf Deutsch.

## Harte Regeln

- **Testlauf ist Standard**: jedes Patch-Playbook hat `*_dry_run: true`.
  Schalter immer mit `| bool` auswerten, weil Semaphore Survey-Werte als Text
  liefert (`"false"` wäre sonst wahr).
- **Keine Pro-Host-Variable als Play-`vars` oder in `vars_files`**, denn beide
  schlagen das Inventory. Den Standard bei der Verwendung setzen:
  `{{ x | default(...) }}`.
- **Keine `requirements.yml`, keine neuen Python-Abhängigkeiten.** Collections
  kommen aus dem Semaphore-Image (ansible 13.5.0): `ansible.windows`,
  `community.general`, `ansible.posix`. `pyvmomi` fehlt, deshalb läuft vCenter
  per `ansible.builtin.uri` (REST + VI/JSON).
- **FQCN** für alle Module.
- Wiederverwendbares nach `tasks/<common|windows|vmware>/…yml`, mit
  Kopfkommentar (Eingabe / Optional / Ergebnis). Einbinden mit vollem Pfad:
  `include_tasks: "{{ playbook_dir }}/tasks/…"`.
- Kommentare in YAML auf Deutsch und **nur ASCII** (ae/oe/ue/ss). Umlaute nur
  in Markdown und im Protokoll-Template.
- **Offline ist kein Fehler** (`tasks/common/check-reachable.yml`). Probleme
  sammeln, zuerst die Mail verschicken (`send-report-mail.yml`), dann den Lauf
  scheitern lassen (`fail-on-problems.yml`).
- Kritischer Pfad (Snapshot, Installation, Neustart) läuft in `block`/`rescue`,
  ohne pauschales `ignore_errors`. **Kein Patch ohne Snapshot.**
- winget immer **als SYSTEM** (`become_method: runas`) mit vollem Pfad zur
  `winget.exe` unter `WindowsApps`. Pins aller Benutzer respektieren.
- Hosts und vCenter als **FQDN** (`*.gutingia.local`), sonst greift `NO_PROXY`
  nicht.
- Passwörter nur als Secret in der Semaphore-Variablengruppe, Login-Tasks mit
  `no_log: true`. Keine Vault-Passwortdateien ins Repo.

## Jinja / Ansible-Eigenheiten

- Leere Templates werden zu `None`: `default('', true) | trim`.
- Mehrzeilige Texte als `|-`, nicht `>-`.
- In Templates `namespace()` statt `.append`.
- `when` an einem `include_tasks` mit `loop` wird vorab ausgewertet. Für einen
  Abbruch mitten in der Schleife das `when` auf einen Block in der Datei legen.
- Kein `run_once` in bedingten Blöcken.
- Mit `delegate_to: localhost` steht der echte Host in
  `hostvars[inventory_hostname]`.
- `serial` kennt nur Extra-Vars und `vars_files`.

## Protokoll (update-windows.yml)

Das Format in `templates/windows-update-protocol.j2` entspricht den
Patch-Protokollen des Teams und darf sich nicht ändern. Die letzten sechs
Punkte (WSUS Status prüfen, Logs > Anwendung, Logs > System, Abmelden,
VMTools aktualisiert, Snapshot löschen) bleiben **leer** (`= `), die füllt das
Team von Hand.

## Arbeitsweise

- Commits auf Deutsch: `datei: was sich aendert`. Branch `stable`, sofern
  nicht anders angesagt.
- README.md bei jeder Änderung mitpflegen, auch den Abschnitt „Aufbau“.
- Testen: YAML parsen, Templates mit Jinja2 und Testdaten rendern,
  PowerShell-Teile mit Windows PowerShell 5.1 parsen. Danach in Semaphore das
  Pull-Template und das `[TEST]`-Template mit Testlauf starten.

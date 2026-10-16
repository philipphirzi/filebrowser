# Externen Prüf-/Review-Zugang einrichten und entfernen

> Internes Verfahren (kein Teil der ursprünglichen File-Browser-Projektdokumentation).
> Anwendungsfall: Eine externe Person (Kooperationspartner, Auditor, Dienstleister)
> muss vorübergehend bestimmte Dokumente einsehen können, ohne Zugriff auf den
> restlichen Datenbestand zu erhalten.

## Prinzip

Kein anonymer Freigabe-Link (File Browsers eingebaute "Share"-Funktion greift seit
Einführung des Authelia-Vorschalt-Logins ohnehin nicht mehr, da jeder Zugriff auf
die Domain eine Anmeldung erfordert). Stattdessen: ein **eigener, namentlicher
Nur-Lese-Account**, beschränkt auf genau den Ordner, der eingesehen werden soll.

Das erfüllt gleichzeitig die ISO-27001-Erwartung an nachvollziehbaren,
minimalberechtigten Zugriff (u. a. A.5.18 Zugriffsrechte, A.8.3
Zugriffsbeschränkung auf Informationen, A.5.14 Informationsübertragung).

## Einrichten

Voraussetzung: SSH-Zugriff auf die VM (`ssh filebrowser-vm` bzw. Host
`<eure-VM>`), `sudo`-Rechte.

### 1. Dedizierten Ordner anlegen

```bash
sudo install -d -o phil -g phil -m 750 /mnt/BigData/<ORDNERNAME>
```

`/mnt/BigData` selbst gehört bewusst `root:root` (`755`). File Browser läuft als
`phil` und kann dort daher **keine** Ordner auf oberster Ebene anlegen — weder
über "Neuer Ordner" in der Oberfläche (→ 403) noch beim Anlegen eines Nutzers mit
automatischem Home-Verzeichnis (→ 500 `failed to create user home dir ...
permission denied`). Neue Ordner auf oberster Ebene deshalb immer wie oben per
SSH anlegen, **bevor** der Nutzer angelegt wird. Das gilt für alle Nutzer, nicht
nur für externe.

Nur die Dokumente, die die externe Person sehen soll, in diesen Ordner legen
(nicht auf einen bestehenden, umfassenderen Ordner freigeben).

### 2. File-Browser-Nutzer anlegen (Nur-Lese, auf den Ordner beschränkt)

```bash
sudo systemctl stop filebrowser
sudo -u phil /usr/local/bin/filebrowser users add <BENUTZERNAME> "<PLATZHALTER>" \
  --scope <ORDNERNAME> \
  --perm.admin=false --perm.create=false --perm.rename=false \
  --perm.modify=false --perm.delete=false --perm.share=false \
  --perm.download=true \
  --database /home/phil/FileBrowser/filebrowser.db
sudo systemctl start filebrowser
```

`<PLATZHALTER>` ist beliebig (z. B. `openssl rand -hex 24`) — das eigentliche
Passwort für den Login kommt aus Authelia (Proxy-Auth), dieses Feld wird nicht
verwendet.

Wird der Nutzer stattdessen in der File-Browser-Oberfläche angelegt: **Scope**
auf `/<ORDNERNAME>` setzen (nicht `/` — sonst sieht der Nutzer ganz
`/mnt/BigData`), und das dort eingegebene Passwort ist ebenfalls bedeutungslos.

### 3. Authelia-Account anlegen

**Ohne diesen Schritt ist kein Login möglich** — auch nicht für Nutzer, die in
der File-Browser-Oberfläche angelegt wurden. Authelia meldet dann nur
"Benutzername oder Passwort falsch" (im Log: `user not found`).

Die Datei ist mit **4 Leerzeichen** eingerückt; der neue Block muss genau dazu
passen, sonst ist die YAML-Datei kaputt und **niemand** kann sich mehr anmelden.
Daher erst eine Kopie bearbeiten, prüfen und dann übernehmen:

```bash
F=/etc/authelia/users_database.yml
sudo cp -p "$F" "$F.bak-$(date +%Y%m%d-%H%M%S)"
PW=$(openssl rand -base64 15 | tr -d '=+/' | cut -c1-16)
HASH=$(sudo authelia crypto hash generate argon2 --variant argon2id --password "$PW" | sed -n 's/^Digest: //p')
TMP=$(mktemp)
sudo cat "$F" > "$TMP"
cat >> "$TMP" << EOF
    <BENUTZERNAME>:
        disabled: false
        displayname: '<Klarname / Firma>'
        password: '$HASH'
        email: <E-Mail-Adresse>
        groups:
            - external
EOF
python3 -c 'import sys, yaml; print(sorted(yaml.safe_load(open(sys.argv[1]))["users"]))' "$TMP" \
  && sudo install -o authelia -g authelia -m 640 "$TMP" "$F"
rm -f "$TMP"
echo "<BENUTZERNAME>:$PW"
```

Der Benutzername muss exakt (inkl. Groß-/Kleinschreibung) dem File-Browser-Nutzer
entsprechen. Für Mitarbeiter statt `external` die Gruppe `users` eintragen.

Dank `watch: true` in `/etc/authelia/configuration.yml` wirkt die Änderung
sofort, kein Neustart von Authelia nötig. Ein einzelnes `permission denied` beim
Reload im Authelia-Log direkt beim Ersetzen der Datei ist harmlos, solange
danach `Reloaded successfully` folgt.

### 4. Testen, bevor die Zugangsdaten verschickt werden

```bash
curl -s -H "Remote-User: <BENUTZERNAME>" -X POST http://127.0.0.1:8282/files/api/login -o /tmp/tok.txt
TOKEN=$(cat /tmp/tok.txt)
curl -s -H "X-Auth: $TOKEN" http://127.0.0.1:8282/files/api/resources/   # sollte NUR den freigegebenen Ordner zeigen
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "X-Auth: $TOKEN" --data test http://127.0.0.1:8282/files/api/resources/test.txt   # sollte 403 sein
rm -f /tmp/tok.txt
```

### 5. Zugangsdaten übermitteln

Benutzername + temporäres Passwort + URL (`https://files.example.com/files/`)
über einen separaten, vertrauenswürdigen Kanal an die Person schicken (nicht im
selben Kanal wie ggf. mitgelesene Dokumente). Empfehlung: Person bittet, das
Passwort nach dem ersten Login zu ändern (siehe unten).

## Nach Abschluss der Prüfung: Zugang entfernen

**Nicht vergessen** — das ist der Teil, der für ISO 27001 tatsächlich zählt
(Zugriff endet, wenn der Zweck endet):

```bash
sudo systemctl stop filebrowser
sudo -u phil /usr/local/bin/filebrowser users rm <BENUTZERNAME> --database /home/phil/FileBrowser/filebrowser.db
sudo systemctl start filebrowser
```

Und den passenden Block aus `/etc/authelia/users_database.yml` löschen
(`sudo nano /etc/authelia/users_database.yml`, Block entfernen, speichern —
wirkt dank `watch: true` sofort).

Optional: den Ordner unter `/mnt/BigData/<ORDNERNAME>` archivieren oder löschen,
falls er nur für diesen Zweck angelegt wurde.

## Upload-Fehler 413 (nginx)

File Browser lädt Dateien per TUS in 10-MB-Stücken hoch. Der nginx-Standard
(`client_max_body_size 1m`) blockt das mit `413 Request Entity Too Large`. In
`/etc/nginx/sites-available/filebrowser-auth.conf` ist deshalb im `server`-Block
`client_max_body_size 64m;` gesetzt (gilt pro Stück, nicht für die Gesamtdatei).
Bei Änderungen an der nginx-Konfiguration nicht entfernen; danach immer
`sudo nginx -t && sudo systemctl reload nginx`.

## Passwort ändern

Siehe [`passwort-verwaltung.md`](passwort-verwaltung.md) für den vollständigen
Ablauf (manuell durch Admin oder per Reset-Link). Siehe auch
[`risikoakzeptanz-restrisiken.md`](risikoakzeptanz-restrisiken.md) für den
Gesamtkontext der Authelia-Einführung (FB-RISK-002).

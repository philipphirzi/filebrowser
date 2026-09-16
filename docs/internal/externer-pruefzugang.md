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
`ubuntu-file-share`), `sudo`-Rechte.

### 1. Dedizierten Ordner anlegen

```bash
sudo mkdir -p /mnt/BigData/<ORDNERNAME>
sudo chown phil:phil /mnt/BigData/<ORDNERNAME>
sudo chmod 750 /mnt/BigData/<ORDNERNAME>
```

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

### 3. Authelia-Account anlegen

```bash
PW=$(openssl rand -base64 15 | tr -d '=+/' | cut -c1-16)
HASH=$(sudo authelia crypto hash generate argon2 --variant argon2id --password "$PW" | sed -n 's/^Digest: //p')
sudo tee -a /etc/authelia/users_database.yml > /dev/null << EOF
  <BENUTZERNAME>:
    disabled: false
    displayname: '<Klarname / Firma>'
    password: '$HASH'
    email: <optional>
    groups:
      - external
EOF
echo "<BENUTZERNAME>:$PW"
```

Dank `watch: true` in `/etc/authelia/configuration.yml` wirkt die Änderung
sofort, kein Neustart von Authelia nötig.

### 4. Testen, bevor die Zugangsdaten verschickt werden

```bash
curl -s -H "Remote-User: <BENUTZERNAME>" -X POST http://127.0.0.1:8282/files/api/login -o /tmp/tok.txt
TOKEN=$(cat /tmp/tok.txt)
curl -s -H "X-Auth: $TOKEN" http://127.0.0.1:8282/files/api/resources/   # sollte NUR den freigegebenen Ordner zeigen
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "X-Auth: $TOKEN" --data test http://127.0.0.1:8282/files/api/resources/test.txt   # sollte 403 sein
rm -f /tmp/tok.txt
```

### 5. Zugangsdaten übermitteln

Benutzername + temporäres Passwort + URL (`https://files.nexgen-itsec.com/files/`)
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

## Passwort ändern

Siehe [`passwort-verwaltung.md`](passwort-verwaltung.md) für den vollständigen
Ablauf (manuell durch Admin oder per Reset-Link). Siehe auch
[`risikoakzeptanz-restrisiken.md`](risikoakzeptanz-restrisiken.md) für den
Gesamtkontext der Authelia-Einführung (FB-RISK-002).

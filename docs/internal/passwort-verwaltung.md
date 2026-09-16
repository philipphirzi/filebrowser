# Passwörter verwalten (Authelia)

> Internes Verfahren (kein Teil der ursprünglichen File-Browser-Projektdokumentation).
> Gilt für alle Nutzer-Accounts (Mitarbeiter, Steuerberater, externe Prüf-Accounts,
> siehe [`externer-pruefzugang.md`](externer-pruefzugang.md)).

## Hintergrund

Seit der Umstellung auf Authelia als vorgeschalteten Login (siehe
[`risikoakzeptanz-restrisiken.md`](risikoakzeptanz-restrisiken.md), FB-RISK-002)
liegt die eigentliche Anmeldung — und damit auch die Passwörter — bei Authelia,
nicht mehr bei File Browser selbst. Authelias Nutzerdatenbank ist die Datei
`/etc/authelia/users_database.yml` auf der VM.

Es gibt zwei Wege, ein Passwort zu ändern: **manuell durch einen Admin** (aktuell
der einzige praktikable Weg) oder über den **"Passwort zurücksetzen"-Link** im
Login-Portal (funktioniert technisch, aber ohne echten E-Mail-Versand nur mit
Umweg über die VM — siehe unten).

## Weg 1: Manuell durch einen Admin (empfohlen, aktueller Standardweg)

Voraussetzung: SSH-Zugriff auf die VM (`ssh filebrowser-vm`), `sudo`-Rechte.

```bash
# 1. Neuen Passwort-Hash erzeugen (fragt das Passwort versteckt ab,
#    landet nirgends im Klartext, auch nicht in der Shell-History)
sudo authelia crypto hash generate argon2
```

Ausgabe ist eine Zeile wie:
```
Digest: $argon2id$v=19$m=65536,t=3,p=4$....$....
```

```bash
# 2. Datei öffnen
sudo nano /etc/authelia/users_database.yml
```

Beim betreffenden Nutzer die Zeile `password: '...'` durch den neuen Digest
ersetzen (Anführungszeichen beibehalten). Speichern: in `nano` mit `Strg+O`,
`Enter`, `Strg+X`.

**Kein Neustart nötig** — die Konfiguration hat `watch: true` gesetzt
(`authentication_backend.file.watch` in `/etc/authelia/configuration.yml`),
Änderungen an der Datei werden automatisch erkannt und wirken sofort beim
nächsten Login-Versuch.

## Weg 2: "Passwort zurücksetzen"-Link (Self-Service, aktuell mit Umweg)

1. Auf der Login-Seite (`https://files.nexgen-itsec.com/authelia`) auf
   **"Passwort zurücksetzen?"** klicken, Benutzername eingeben.
2. Authelia erzeugt einen Reset-Link — aber es ist noch **kein echter
   E-Mail-Versand (SMTP)** eingerichtet, sondern nur der `filesystem`-Notifier.
   Das heißt: Der Link landet **nicht** im Postfach der Person, sondern wird in
   eine Datei auf der VM geschrieben:
   ```bash
   sudo cat /var/lib/authelia/notification.txt
   ```
3. Den Link aus der Datei herauskopieren und der Person auf einem sicheren Weg
   zukommen lassen (z. B. Chat, Telefon vorlesen — nicht unverschlüsselt per Mail
   im selben Thread wie andere Dokumente).
4. Die Person öffnet den Link und setzt ihr eigenes neues Passwort.

**Für externe Personen (z. B. Steuerberater) unpraktisch**, da sie keinen
VM-Zugriff haben — für sie ist aktuell **Weg 1** der richtige Weg, bis SMTP
eingerichtet ist.

## Ausblick: Echter E-Mail-Versand (SMTP)

Sobald ein SMTP-Zugang (z. B. bestehendes Geschäfts-Postfach) hinterlegt wird,
funktioniert Weg 2 vollautomatisch für alle Nutzer selbständig, ohne Admin-
Beteiligung. Dazu in `/etc/authelia/configuration.yml` den `notifier`-Abschnitt
von `filesystem` auf `smtp` umstellen (Host, Port, Absender, Zugangsdaten).
Bisher nicht umgesetzt, da noch kein SMTP-Zugang bereitgestellt wurde.

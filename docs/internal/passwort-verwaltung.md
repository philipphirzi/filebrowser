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

Seit Einrichtung des echten SMTP-Versands (IONOS, `office@example.com`)
ist der **Self-Service-Weg (Weg 1) der Standardweg** für alle Nutzer, die eine
funktionierende, hinterlegte E-Mail-Adresse haben — inklusive externer Personen
wie dem Steuerberater. Der manuelle Admin-Weg (Weg 2) bleibt als Fallback für
Sonderfälle (z. B. Account ohne hinterlegte/funktionierende E-Mail-Adresse,
oder schnelle Erstvergabe eines Accounts).

## Weg 1: Self-Service über "Passwort zurücksetzen" (Standardweg)

1. Auf der Login-Seite auf **"Passwort zurücksetzen?"** klicken, Benutzername
   eingeben.
2. Authelia verschickt automatisch eine E-Mail mit Bestätigungscode an die im
   Nutzer hinterlegte `email:`-Adresse (siehe `/etc/authelia/users_database.yml`).
3. Die Person trägt den Code ein und setzt ihr eigenes neues Passwort — ganz
   ohne Admin-Beteiligung.

**Alles auf einem Gerät im selben Browser erledigen.** Authelia merkt sich den
laufenden Reset im Session-Cookie: Wird der Reset z. B. am Rechner gestartet und
der Mail-Link auf dem Handy geöffnet, scheitert das Speichern des neuen Passworts
(im Log: `no identity verification process has been initiated`). Der Link ist
nur 5 Minuten gültig und nach dem Öffnen verbraucht — dann einfach neu anfordern.

Funktioniert nur, wenn beim Nutzer eine echte, erreichbare E-Mail-Adresse
hinterlegt ist. Beim Anlegen neuer Accounts also immer eine korrekte Adresse
eintragen (siehe [`externer-pruefzugang.md`](externer-pruefzugang.md)).

Derselbe E-Mail-Versand wird auch für die **Identitätsbestätigung beim
Einrichten neuer 2FA-Methoden** (TOTP) verwendet (siehe
[`audit-log-und-2fa.md`](audit-log-und-2fa.md)) — auch das läuft jetzt
automatisch per Mail statt über den manuellen VM-Umweg.

## Weg 2: Manuell durch einen Admin (Fallback)

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

## SMTP-Konfiguration (Referenz)

In `/etc/authelia/configuration.yml`, Abschnitt `notifier.smtp`:
- Server: `smtp.ionos.de:587` (STARTTLS)
- Absender/Login: `office@example.com`
- Passwort: nicht in `configuration.yml`, sondern in der Secret-Datei
  `/etc/authelia/secrets/notifier_smtp_password` (nicht in Git, nur lokal auf der VM)

Bei einem Passwortwechsel des Postfachs muss dieser Wert entsprechend
aktualisiert und Authelia neu gestartet werden (`sudo systemctl restart authelia`
— anders als bei `users_database.yml` gibt es hier kein automatisches Neuladen).

**Rechte des Secret-Verzeichnisses beachten.** Authelia läuft als Nutzer
`authelia` und braucht auf das Verzeichnis das Ausführungsrecht (`x`), um die
Datei darin öffnen zu können:

```bash
sudo chown -R authelia:authelia /etc/authelia/secrets
sudo chmod 750 /etc/authelia/secrets
sudo chmod 640 /etc/authelia/secrets/notifier_smtp_password
```

Fehlt das `x` (z. B. `drw-r-----`), startet Authelia nicht mehr
(`file permission error occurred: open /etc/authelia/secrets/notifier_smtp_password:
permission denied`, Portal liefert 502). `authelia validate-config` erkennt das
**nicht** — nach jeder Änderung an den Secrets daher Authelia tatsächlich neu
starten und mit `systemctl is-active authelia` prüfen, statt sich auf
`validate-config` zu verlassen. Genau das ist am 2026-10-16 passiert: Die
falschen Rechte bestanden seit der SMTP-Einrichtung am 2026-09-16 und fielen erst
beim nächsten Neustart auf.

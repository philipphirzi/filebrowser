# Zugriffsprotokollierung und Zwei-Faktor-Pflicht für sensible Daten

> Internes Verfahren (kein Teil der ursprünglichen File-Browser-Projektdokumentation).
> Ergänzt [`risikoakzeptanz-restrisiken.md`](risikoakzeptanz-restrisiken.md) (FB-RISK-002)
> um zwei zusätzliche Härtungsmaßnahmen für besonders vertrauliche Daten.

## 1. Zugriffsprotokollierung (Audit-Log)

**Warum:** Vorher protokollierte nginx nur die IP des internen Apache-Reverse-Proxys
und keinen Benutzernamen — es war nicht nachvollziehbar, *wer* wann *welche* Datei
angesehen oder heruntergeladen hat, und die echte Herkunfts-IP externer Zugriffe
ging verloren. Für ISO 27001 A.8.15 (Protokollierung) braucht es einen
nachvollziehbaren Zugriffs-Audit-Trail.

**Was protokolliert wird:** `/var/log/nginx/filebrowser-audit.log`, ein Eintrag
pro Request:

```
2026-09-16T15:45:38+02:00 real_client=203.0.113.42 proxy_hop=10.0.3.57 user=Andreas status=200 method=GET uri="/files/api/resources/Belege" bytes=8311 ref="-" ua="Mozilla/5.0 ..."
```

- `real_client` — echte Herkunfts-IP des Nutzers (aus `X-Forwarded-For`, von Apache weitergereicht)
- `proxy_hop` — IP des internen Apache-Reverse-Proxys (zur Nachvollziehbarkeit der Kette)
- `user` — Authelia-Benutzername, sofern zum Zeitpunkt des Requests bereits erfolgreich authentifiziert (bei nicht angemeldeten/abgelehnten Requests leer)
- `status`, `method`, `uri`, `bytes`, `ref`, `ua` — wie bei einem normalen Webserver-Log

**Konfiguration:**
- Format definiert in `/etc/nginx/nginx.conf` (`log_format audit ...`)
- Aktiviert im geschützten Bereich in `/etc/nginx/sites-available/filebrowser-auth.conf` (`access_log ... audit;`)
- Aufbewahrung: **400 Tage** statt der nginx-Standard-14-Tage, eigene Regel unter
  `/etc/logrotate.d/filebrowser-audit` (separat von den generischen nginx-Logs,
  damit die längere Aufbewahrung nicht unnötig Speicherplatz für uninteressante
  Logs verbraucht)

**Log durchsuchen (Beispiele):**

```bash
# Alle Zugriffe eines bestimmten Nutzers
sudo grep "user=Andreas" /var/log/nginx/filebrowser-audit.log

# Alle Zugriffe von einer bestimmten IP
sudo grep "real_client=203.0.113.42" /var/log/nginx/filebrowser-audit.log

# Auch in bereits rotierten (komprimierten) Logs suchen
sudo zgrep "user=Andreas" /var/log/nginx/filebrowser-audit.log.*.gz
```

## 2. Zwei-Faktor-Pflicht für die Gruppe `geheim`

**Warum:** Für besonders vertrauliche Dokumente reicht ein einfaches Passwort
nicht aus (ISO 27001 A.8.5, sichere Authentifizierung). Mitglieder der
Authelia-Gruppe `geheim` müssen sich zusätzlich per TOTP (Authenticator-App)
verifizieren.

**Wie es technisch funktioniert:**
- In `/etc/authelia/configuration.yml`, Abschnitt `access_control`, gibt es eine
  Regel, die *vor* der allgemeinen Regel steht und für `subject: 'group:geheim'`
  `policy: two_factor` verlangt. Alle anderen Nutzer bleiben bei `one_factor`.
- In `/etc/authelia/users_database.yml` ist die Gruppenzugehörigkeit einfach ein
  weiterer Eintrag unter `groups:` beim jeweiligen Nutzer.

**Wichtige Einschränkung:** Die 2FA-Pflicht gilt für den **gesamten**
File-Browser-Zugriff der Person, nicht nur für einen bestimmten Unterordner.
Grund: File Browser bildet pro Nutzer einen eigenen "Scope" (Stammverzeichnis)
ab — von außen (nginx/Authelia) ist anhand der URL nicht erkennbar, welchen
echten Unterordner eine Person gerade ansieht, da die URL für alle Nutzer gleich
aussieht (`/files/api/resources/...`), nur der tatsächliche Inhalt unterscheidet
sich je nach Scope. Eine ordnerspezifische 2FA-Pflicht ist mit File Browsers
Architektur daher nicht sauber abbildbar — nur eine personenbezogene.

Wer nur für einzelne, seltene Geheim-Zugriffe 2FA will, ohne den Alltags-Account
zu betreffen, sollte dafür einen **zweiten, dedizierten Account** anlegen (wie
im Beispiel [`externer-pruefzugang.md`](externer-pruefzugang.md) beschrieben),
der ausschließlich auf den Geheim-Ordner gescoped ist und in die Gruppe `geheim`
kommt.

### Weiteren Nutzer zur 2FA-Pflicht hinzufügen

Kein Authelia-Neustart nötig (nur die Nutzerdatei wird geändert, `watch: true`
greift automatisch):

```bash
sudo nano /etc/authelia/users_database.yml
```

Beim Nutzer unter `groups:` eine Zeile `- geheim` ergänzen, speichern. Beim
nächsten Login wird die Person automatisch zur TOTP-Einrichtung (QR-Code
scannen) aufgefordert.

### Neue Zwei-Faktor-Regel für eine andere Gruppe anlegen

Das ist eine Änderung an `access_control` in `/etc/authelia/configuration.yml`
selbst (nicht an der Nutzerdatei) und **erfordert einen Neustart**:

```bash
sudo -u authelia authelia validate-config --config /etc/authelia/configuration.yml
sudo systemctl restart authelia
```

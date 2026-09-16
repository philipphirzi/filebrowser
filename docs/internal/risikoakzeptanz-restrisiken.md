# Risikoakzeptanz – Restrisiken File Browser (interner Fork)

> **Hinweis:** Dies ist ein internes ISMS-Dokument des Unternehmens und kein Teil der
> ursprünglichen File-Browser-Projektdokumentation. Es dokumentiert die Risikobewertung
> und -akzeptanz für den Weiterbetrieb dieses Forks nach der Archivierung des
> Upstream-Projekts.

| | |
|---|---|
| **Dokumenttyp** | ISMS-Risikoakzeptanz gemäß ISO/IEC 27001:2022 |
| **Geltungsbereich** | Betrieb des internen File-Browser-Forks (`philipphirzi/filebrowser`) zur Dateifreigabe im Unternehmen sowie gegenüber dem Steuerberater |
| **Bezug** | Clause 6.1.3 (Risikobehandlung), Control A.8.8 (Management technischer Schwachstellen), A.8.9 (Konfigurationsmanagement), A.5.3 (Verantwortlichkeiten) |
| **Version** | 1.0 – Entwurf |
| **Datum** | 2026-09-16 |
| **Risikoeigner (Freigabe)** | _______________________ (Name, Funktion: stellv. Geschäftsführung) |

## 1. Hintergrund

Das Upstream-Projekt File Browser (`filebrowser/filebrowser`) wurde am 2026-09-01
archiviert; die ursprünglichen Maintainer stellen keine weiteren Sicherheitsupdates
mehr bereit. Das Unternehmen betreibt seither einen eigenen Fork
(`https://github.com/philipphirzi/filebrowser`) mit eigenem Dependency- und
Schwachstellenmanagement (siehe `.github/dependabot.yml` und
`.github/workflows/security.yml`).

Zwei vom Upstream-Projekt selbst dokumentierte, strukturelle Schwachstellenklassen
sind davon nicht abgedeckt, da es sich um Architekturentscheidungen im Code und
nicht um reine Abhängigkeitsprobleme handelt. Diese werden hier formal bewertet
und als Restrisiko dokumentiert.

## 2. Risiko FB-RISK-001: Command-Runner / Hook- und Shell-Ausführung

| Feld | Angabe |
|---|---|
| Betroffene Assets | Server, auf dem File Browser läuft; alle darüber zugänglichen Daten |
| Bedrohung | Remote Code Execution über Hook-Runner bzw. interaktive Shell |
| Schwachstelle | Feature ist laut Upstream "plagued with vulnerabilities", bräuchte kompletten Rewrite (siehe [`docs/command-execution.md`](../command-execution.md), Issue [#5199](https://github.com/filebrowser/filebrowser/issues/5199)) |
| Eintrittswahrscheinlichkeit (ohne Maßnahmen) | Mittel – nur ausnutzbar, wenn Feature aktiv genutzt wird |
| Auswirkung (ohne Maßnahmen) | Sehr hoch – vollständige Serverkompromittierung, äquivalent zu Shell-Zugriff |
| Bruttorisiko | Hoch |
| Kompensationsmaßnahmen | 1) Feature bleibt deaktiviert (`--disable-exec` Standardwert, `auth.method` ≠ `hook`, keine Commands je Nutzer hinterlegt). 2) Regelmäßige Konfigurationsprüfung als Teil des internen Audits (siehe Abschnitt 4). 3) Betrieb im unprivilegierten Container mit minimalem Dateisystemzugriff. |
| Restrisiko (mit Maßnahmen) | Niedrig |
| Risikoeigner | _______________________ (IT-/Security-Verantwortlicher) |
| Entscheidung | ☐ Akzeptiert, unter der Bedingung, dass die Kompensationsmaßnahmen dauerhaft bestehen bleiben und im Audit-Log nachgewiesen werden. |

## 3. Risiko FB-RISK-002: Session-/JWT-Handling ohne serverseitige Widerrufsmöglichkeit

| Feld | Angabe |
|---|---|
| Betroffene Assets | Nutzer-Sessions, darüber zugängliche Unternehmensdaten inkl. Steuerberater-Belege |
| Bedrohung | Fortgesetzter unautorisierter Zugriff mit gestohlenem/kompromittiertem Token trotz Logout, Passwortänderung oder Sperrung des Nutzerkontos |
| Schwachstelle | Sessions sind selbstenthaltende JWTs ohne serverseitige Widerrufsliste; Refresh-Token mehrfach einlösbar (Issue [#5216](https://github.com/filebrowser/filebrowser/issues/5216)) |
| Eintrittswahrscheinlichkeit (ohne Maßnahmen) | Mittel – erfordert vorherigen Token-Diebstahl (Phishing, Endgerät-Kompromittierung, Mitschnitt bei fehlendem TLS) |
| Auswirkung (ohne Maßnahmen) | Hoch – Zugriff auf sensible Unternehmens-/Mandantendaten bis zum Token-Ablauf |
| Bruttorisiko | Hoch |
| Kompensationsmaßnahmen | 1) File Browser nicht direkt exponiert, sondern hinter Auth-Reverse-Proxy ([Authelia](https://www.authelia.com/) oder [Authentik](https://goauthentik.io/), Open Source) mit eigener, sofort widerrufbarer Session-Verwaltung. 2) Kurze JWT-Lebensdauer in der File-Browser-Konfiguration. 3) Ausschließlich TLS-verschlüsselter Zugriff. 4) Kein direkter Internetzugriff; Zugriff nur aus Firmennetz/VPN bzw. für den Steuerberater über einen dedizierten, protokollierten Zugang. |
| Restrisiko (mit Maßnahmen) | Niedrig-Mittel |
| Risikoeigner | _______________________ (IT-/Security-Verantwortlicher) |
| Entscheidung | ☐ Akzeptiert, unter der Bedingung, dass der Auth-Reverse-Proxy produktiv eingerichtet ist, bevor der Steuerberater-Zugriff aktiviert wird, Zieldatum: _______________ |

## 4. Wiederkehrende Überprüfung

- **Turnus:** Vierteljährlich sowie anlassbezogen bei jedem größeren Update des Forks.
- **Prüfpunkte:**
  - Ist der Command-Runner weiterhin deaktiviert (Konfiguration, nicht nur Default)?
  - Ist der Auth-Reverse-Proxy aktiv und korrekt konfiguriert?
  - Sind offene Dependabot-/govulncheck-/OSV-Scanner-/Trivy-Befunde abgearbeitet oder bewusst zurückgestellt und begründet?
- **Verantwortlich:** _______________________ (OSS-Component-Owner)
- **Nächste Überprüfung:** _______________________

## 5. Freigabe

Dieses Restrisiko wurde bewertet und unter den oben genannten Bedingungen akzeptiert.

| Rolle | Name | Datum | Unterschrift |
|---|---|---|---|
| Stellv. Geschäftsführung | | | |
| IT-/Security-Verantwortlicher | | | |

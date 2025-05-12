# Government - HackMyVM (Medium) - Writeup

![Government Icon](Government.png)

Dieses Repository enthält einen zusammengefassten Bericht über die Kompromittierung der HackMyVM-Maschine "Government" (Schwierigkeitsgrad: Medium).

## Metadaten

*   **Maschine:** Government (HackMyVM - Medium)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Government](https://hackmyvm.eu/machines/machine.php?vm=Government)
*   **Autor des Writeups:** DarkSpirit
*   **Datum:** 11. Oktober 2022
*   **Original Writeup:** [https://alientec1908.github.io/Government_HackMyVM_Medium/](https://alientec1908.github.io/Government_HackMyVM_Medium/)

## Zusammenfassung des Angriffspfads

Die initiale Erkundung mit `nmap` offenbarte eine Vielzahl offener Dienste, darunter FTP (Port 21) mit erlaubtem anonymen Login, SSH (Port 22), HTTP (Port 80), NFS (Port 2049) und SMB (Ports 139/445). Der anonyme FTP-Zugriff wurde genutzt (`wget -m`), um Dateien herunterzuladen. Im Verzeichnis `files` wurden Textdateien (`.txt`) gefunden, die eine Liste von Benutzernamen (`emma`, `christian`, `luds`, `malic`, `susan`) und vier MD5-Passwort-Hashes enthielten.

Die weitere Web-Enumeration mit `ffuf` deckte ein `/phppgadmin`-Verzeichnis auf. Es gelang, sich bei der PostgreSQL-Datenbank über das phpPgAdmin-Webinterface mit Standard-Credentials (`postgres:admin`) anzumelden. Innerhalb von phpPgAdmin wurde die SQL-Funktion `COPY FROM PROGRAM` missbraucht, um mittels `nc` eine Reverse Shell als Benutzer `postgres` zu erhalten.

Als `postgres` wurde die versteckte Log-Datei `/var/log/.creds.log` entdeckt. Diese enthielt einen Klartext-String (`h4cK1sMyf4v0ri73G4m3`), der als Passwort für den Benutzer `erik` funktionierte (`su erik`).

Als `erik` wurde mittels `find` eine verdächtige SUID-Datei `/home/erik/backups/nuclear/remove` identifiziert. `strings` auf diese Datei zeigte, dass sie den Befehl `time` ohne vollständigen Pfad aufrief. Dies ermöglichte eine Path-Hijacking-Schwachstelle: Ein bösartiges Skript namens `time` (welches `/bin/bash -p` enthielt) wurde in `/tmp` erstellt, ausführbar gemacht und `/tmp` zum `PATH` hinzugefügt. Der anschließende Aufruf von `/home/erik/backups/nuclear/remove` führte das bösartige Skript mit Root-Rechten aus und gewährte eine Root-Shell (`euid=0`). Abschließend wurden die User- und Root-Flags ausgelesen.

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `lftp`
*   `wget`
*   `cat`
*   `ffuf`
*   `psql` (via phppgadmin)
*   `nc` (netcat)
*   `su`
*   `find`
*   `strings`
*   `echo`
*   `chmod`
*   `export`
*   (Implied: Hash Cracker für MD5 - nicht im Log, aber erwähnt)

## Angriffsschritte (Übersicht)

1.  **Reconnaissance:** Ziel-IP (`arp-scan`), Portscan (`nmap`) -> FTP (anon), SSH, HTTP, NFS, SMB.
2.  **Information Disclosure (FTP):** Anonym via `lftp`/`wget` verbinden. Dateien aus `/files` herunterladen -> `*.txt` enthalten Benutzernamen und MD5-Hashes.
3.  **Web Enumeration:** `ffuf` -> `/phppgadmin` gefunden.
4.  **Initial Access (phpPgAdmin RCE):** Login bei `phpPgAdmin` als `postgres:admin`. SQL-Befehl `COPY cmd_exec FROM PROGRAM 'nc...'` ausführen -> Reverse Shell als `postgres`.
5.  **Privilege Escalation (`postgres` -> `erik`):** Datei `/var/log/.creds.log` finden und lesen -> Passwort `h4cK1sMyf4v0ri73G4m3` extrahieren. Mit `su erik` und Passwort wechseln.
6.  **Privilege Escalation Vector (`erik` -> `root`):** SUID-Datei `/home/erik/backups/nuclear/remove` finden (`find`). Mit `strings` unsicheren `time`-Aufruf identifizieren.
7.  **Privilege Escalation Execution (`erik` -> `root`):** Path Hijacking: `/tmp/time` mit `/bin/bash -p` erstellen (`echo`, `chmod`), `/tmp` zu `PATH` hinzufügen (`export`). `/home/erik/backups/nuclear/remove` ausführen -> Root-Shell (`euid=0`).
8.  **Flags:** User-Flag (`cat /home/erik/user.txt`) und Root-Flag (`cat /root/root.txt`) lesen.

## Flags

*   **User Flag (`/home/erik/user.txt`):** `349efd4b1ccbeb4d3ca0108fa5cc5802`
*   **Root Flag (`/root/root.txt`):** `df2f7ef853a5b36c51509a6f6ae2e7b1`

---

## Disclaimer

Die hier dargestellten Informationen und Techniken dienen ausschließlich Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Methoden dürfen nur auf Systemen angewendet werden, für die eine ausdrückliche Genehmigung vorliegt (z.B. in CTF-Umgebungen wie HackMyVM, Penetrationstests mit schriftlicher Erlaubnis). Das unbefugte Eindringen in fremde Computersysteme ist illegal und strafbar. Die Autoren übernehmen keine Haftung für missbräuchliche Verwendung der bereitgestellten Informationen. Handeln Sie stets legal und ethisch verantwortlich.

---

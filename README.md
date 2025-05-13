# Random - HackMyVM (Hard)

![Random.png](Random.png)

## Übersicht

*   **VM:** Random
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Random)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 18. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Random_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Random" zu erlangen. Der initiale Zugriff erfolgte durch das Brute-Forcen von FTP-Credentials für den Benutzer `eleanor`, dessen Name durch eine Notiz auf der Webseite (`index.html`) bekannt wurde. Nach dem FTP-Login wurde festgestellt, dass der Benutzer `eleanor` Schreibrechte im Web-Root-Verzeichnis (`/html`) über SFTP hatte. Eine PHP-Reverse-Shell wurde hochgeladen und über HTTP ausgeführt, was zu einer Shell als `www-data` führte. Die finale Rechteausweitung zu Root gelang durch Ausnutzung eines SUID-Programms (`/home/alan/random`). Dieses Programm lud eine Shared Library (`/lib/librooter.so`) und rief deren Funktion `makemeroot` auf, wenn eine als Argument übergebene Zahl mit einer intern generierten Zufallszahl übereinstimmte. Da das Verzeichnis `/lib` für `www-data` beschreibbar war, konnte `librooter.so` durch eine bösartige Version ersetzt werden, die beim Aufruf von `makemeroot` eine Root-Shell startete.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `hydra`
*   `ftp`
*   `sftp`
*   `nc` (netcat)
*   Python3 (`http.server`, `pty` Modul)
*   `stty`
*   `find`
*   `ldd`
*   IDA Pro/Ghidra (impliziert für Binary-Analyse)
*   `nano` (impliziert)
*   `gcc`
*   `wget`
*   `./random` (benutzerdefiniertes SUID-Binary)
*   Standard Linux-Befehle (`cat`, `vi`, `chmod`, `ls`, `export`, `cd`, `id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Random" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web/FTP Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.109) mit `arp-scan` identifiziert.
    *   `nmap`-Scan offenbarte Port 21 (FTP, vsftpd 3.0.3, anonymer Login erlaubt), 22 (SSH, OpenSSH 7.9p1) und 80 (HTTP, Nginx 1.14.2).
    *   Die `index.html` auf Port 80 enthielt eine Nachricht mit den Benutzernamen `eleanor` und `alan`.
    *   `gobuster` auf Port 80 fand keine weiteren relevanten Dateien.

2.  **Initial Access (FTP Brute-Force & Webshell als `www-data`):**
    *   Mittels `hydra` wurde ein Brute-Force-Angriff auf den FTP-Dienst für den Benutzer `eleanor` mit `rockyou.txt` durchgeführt. Das Passwort `ladybug` wurde gefunden.
    *   FTP-Login als `eleanor:ladybug`. Ein erster Upload-Versuch einer PHP-Reverse-Shell (`rev.php`) in das Home-Verzeichnis (welches `/html` zu sein schien) schlug fehl.
    *   Mittels SFTP (als `eleanor:ladybug`) gelang der Upload von `rev.php` in das Verzeichnis `/html` (Web-Root).
    *   Durch Aufrufen von `http://192.168.2.109/rev.php` wurde die Webshell ausgeführt.
    *   Eine Reverse Shell als `www-data` wurde auf einem Netcat-Listener (Port 9001) empfangen und stabilisiert.

3.  **Privilege Escalation (von `www-data` zu `root` via Shared Library Injection):**
    *   Als `www-data` wurde bei der SUID-Suche (`find / -perm -u=s -type f`) das Programm `/home/alan/random` gefunden.
    *   `ldd /home/alan/random` zeigte, dass das Programm von der Shared Library `/lib/librooter.so` abhing und die Funktion `makemeroot` daraus aufrief, wenn eine als Argument übergebene Zahl (1-9) mit einer internen Zufallszahl übereinstimmte (ermittelt durch Reverse Engineering von `/home/alan/random`).
    *   Es wurde festgestellt, dass das Verzeichnis `/lib` für `www-data` beschreibbar war (kritische Fehlkonfiguration).
    *   Eine bösartige C-Datei (`rooter.c`) wurde erstellt, die eine Funktion `makemeroot()` definierte, welche `setuid(0); setgid(0); system("/bin/bash");` ausführt.
    *   Diese Datei wurde auf dem Angreifer-System mit `gcc -shared rooter.c -o librooter.so -fPIC` zu einer Shared Library kompiliert.
    *   Die kompilierte `librooter.so` wurde über einen Python-HTTP-Server bereitgestellt und mittels `wget` von der `www-data`-Shell nach `/lib/librooter.so` auf dem Zielsystem heruntergeladen, wodurch die Originaldatei überschrieben wurde.
    *   Durch wiederholtes Ausführen von `/home/alan/random [ZAHL]` (wobei ZAHL von 1 bis 9 variiert wurde) wurde die Bedingung erfüllt (mit der Zahl 6). Das SUID-Programm rief die manipulierte `makemeroot`-Funktion auf.
    *   Dies führte zu einer Root-Shell (`uid=0(root)`).
    *   Die User-Flag (`ihavethapowah`) wurde in `/home/eleanor/user.txt` gefunden.
    *   Die Root-Flag (`howiarrivedhere`) wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Information Disclosure:** Benutzernamen wurden auf der Webseite preisgegeben.
*   **Schwaches FTP-Passwort:** Das Passwort für `eleanor` (`ladybug`) konnte durch Brute-Force erraten werden.
*   **Unsichere Dateiberechtigungen (Web-Root & `/lib`):** Das Web-Root-Verzeichnis war für den FTP-Benutzer `eleanor` über SFTP beschreibbar. Das Systemverzeichnis `/lib` war für `www-data` beschreibbar.
*   **SUID-Programm mit Shared Library Injection:** Ein SUID-Root-Programm (`/home/alan/random`) lud eine Shared Library (`/lib/librooter.so`) aus einem beschreibbaren Verzeichnis, was das Einschleusen von bösartigem Code ermöglichte.
*   **Unsichere Zufallszahlengenerierung:** Das SUID-Programm verwendete `rand()` basierend auf `time()`, was die Bedingung zur Ausführung der kritischen Funktion zwar erschwerte, aber nicht unmöglich machte.

## Flags

*   **User Flag (`/home/eleanor/user.txt`):** `ihavethapowah`
*   **Root Flag (`/root/root.txt`):** `howiarrivedhere`

## Tags

`HackMyVM`, `Random`, `Hard`, `FTP Brute-Force`, `SFTP Upload`, `Webshell`, `SUID Exploit`, `Shared Library Injection`, `LD_PRELOAD` (ähnlicher Effekt), `Writable /lib`, `Linux`, `Web`, `Privilege Escalation`, `Nginx`, `vsftpd`

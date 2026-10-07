# ESP32-Projekte mit ESP-IDF, CMake und VS Code — Anleitung für Windows

Diese Anleitung beschreibt, wie ein ESP32-Projekt von Grund auf
aufgesetzt, gebaut, geflasht, debuggt und versioniert wird — in einem
Stil, der auf reproduzierbare, industriell nutzbare Firmware zielt.
Sie ist so geschrieben, dass sie ohne Vorwissen durchgearbeitet werden
kann, und behandelt alle gängigen Chip-Familien (ESP32, ESP32-S2/S3,
ESP32-C3/C6).

Grundprinzip in einem Satz: **ESP-IDF ist Framework UND Build-System;
`idf.py` ist das eine Werkzeug für bauen, flashen, konfigurieren und
überwachen; VS Code ist der Arbeitsplatz; Git/GitHub sind das
Gedächtnis.**

---

## Inhalt

- [Glossar: Unsere Werkzeuge und warum](#glossar-unsere-werkzeuge-und-warum)
- [VS Code in Kürze: Oberfläche und Tastenkürzel](#vs-code-in-kürze-oberfläche-und-tastenkürzel)
  - [Ausgabe, Debug-Konsole, Terminal — wer zeigt was?](#ausgabe-debug-konsole-terminal--wer-zeigt-was)
- [Teil 1: Rechner einrichten (einmalig pro Rechner)](#teil-1-rechner-einrichten-einmalig-pro-rechner)
  - [1.1 ESP-IDF installieren](#11-esp-idf-installieren)
  - [1.2 Umgebung aktivieren — der wichtigste Unterschied zu anderen Toolchains](#12-umgebung-aktivieren--der-wichtigste-unterschied-zu-anderen-toolchains)
  - [1.3 Zugriff auf die serielle Schnittstelle](#13-zugriff-auf-die-serielle-schnittstelle)
  - [1.4 VS Code und Erweiterungen](#14-vs-code-und-erweiterungen)
  - [1.5 Git und GitHub einmalig verbinden](#15-git-und-github-einmalig-verbinden)
- [Teil 2: Neues Projekt anlegen (pro Projekt)](#teil-2-neues-projekt-anlegen-pro-projekt)
  - [2.1 Projekt erzeugen und Ziel-Chip festlegen](#21-projekt-erzeugen-und-ziel-chip-festlegen)
  - [2.2 Erster Build, Flashen und Monitor](#22-erster-build-flashen-und-monitor)
  - [2.3 Konfigurieren: menuconfig und sdkconfig.defaults](#23-konfigurieren-menuconfig-und-sdkconfigdefaults)
  - [2.4 Debug- und Release-Profil einrichten](#24-debug--und-release-profil-einrichten)
  - [2.5 Ausgabedateien: .bin, Sammelabbild und .dfu](#25-ausgabedateien-bin-sammelabbild-und-dfu)
  - [2.6 Projekt-Hygiene ergänzen (unser Standard)](#26-projekt-hygiene-ergänzen-unser-standard)
  - [2.7 Git und GitHub — was wir tun und warum](#27-git-und-github--was-wir-tun-und-warum)
- [Teil 3: Der tägliche Arbeitszyklus](#teil-3-der-tägliche-arbeitszyklus)
- [Exkurs: Die Chip-Familien im Vergleich](#exkurs-die-chip-familien-im-vergleich)
- [Exkurs: Fehlersuche — Monitor, Absturzanalyse, JTAG](#exkurs-fehlersuche--monitor-absturzanalyse-jtag)
- [Exkurs: Komponenten und Bibliotheken einbinden](#exkurs-komponenten-und-bibliotheken-einbinden)
- [Exkurs: Projektstruktur — Komponenten statt alles in main.c](#exkurs-projektstruktur--komponenten-statt-alles-in-mainc)
- [Exkurs: OTA — Firmware-Updates im Feld über WLAN](#exkurs-ota--firmware-updates-im-feld-über-wlan)
- [Teil 4: Checkliste für jedes neue Projekt](#teil-4-checkliste-für-jedes-neue-projekt)
- [Teil 5: Troubleshooting — typische Fehler und ihre Lösung](#teil-5-troubleshooting--typische-fehler-und-ihre-lösung)

---


## Glossar: Unsere Werkzeuge und warum

| Werkzeug | Was es ist | Warum wir es benutzen |
|---|---|---|
| **ESP-IDF** | Espressifs offizielles Entwicklungs-Framework: Treiber, Netzwerk-Stack, FreeRTOS, Build-System und Konfiguration in einem | Offiziell und kostenlos; erscheint in versionierten Releases, die sich festnageln lassen → reproduzierbare Builds. Terminal-first, passt zu CMake, Git und VS Code. |
| **idf.py** | Kommandozeilen-Werkzeug von ESP-IDF | Ein Werkzeug für alles: konfigurieren, bauen, flashen, überwachen, debuggen — mit identischen Befehlen auf jedem Betriebssystem. |
| **CMake + Ninja** | Das Build-System unter der Haube (bringt ESP-IDF mit) | Der Build steckt in versionierten Textdateien statt in IDE-Einstellungen → gleiches Ergebnis auf jedem Rechner. |
| **xtensa-esp-elf-gcc / riscv32-esp-elf-gcc** | Die Compiler für Xtensa- bzw. RISC-V-ESP32 | GCC, kostenlos; die Installation holt automatisch exakt die Version, die zur gewählten ESP-IDF-Version gehört. |
| **menuconfig / sdkconfig** | Konfigurationsoberfläche und die daraus erzeugte Konfigurationsdatei | Software-Konfiguration (Logging, WLAN, Partitionen, Optimierung) — reproduzierbar über die versionierte `sdkconfig.defaults`. |
| **FreeRTOS** | Ist in ESP-IDF fest eingebaut und läuft immer | Kein Extra-Schritt und keine Entscheidung: `app_main()` ist bereits eine Task. |
| **esptool / esp-idf-monitor** | Flash-Werkzeug und serieller Monitor | Flashen über USB ohne Zusatzhardware; der Monitor übersetzt Absturz-Rückverfolgungen automatisch in Dateinamen und Zeilennummern. |
| **OpenOCD + GDB** | JTAG-Debugging (Breakpoints, Einzelschritt) | Bei ESP32-S3/C3/C6 ohne Zusatzhardware über dasselbe USB-Kabel. |
| **VS Code + ESP-IDF-Erweiterung** | Editor und Bedienoberfläche | Kostenlos, plattformübergreifend, gleiche Umgebung für alle im Team. |
| **Git** | Versionsverwaltung: protokolliert jede Änderung lokal | Jeder Stand ist wiederherstellbar, jede Änderung nachvollziehbar. |
| **GitHub** | Online-Ablage für Git-Projekte | Backup, Austausch im Team, zentrale "Wahrheit" — was nicht auf GitHub liegt, existiert offiziell nicht. |

Zwei Dinge, die ESP-IDF anders macht als klassische Mikrocontroller-
Umgebungen und die man früh wissen sollte:

1. **Es gibt keinen grafischen Pin-Planer.** Die Zuordnung von Signalen
   zu Pins passiert im Code (der ESP32 hat eine flexible GPIO-Matrix,
   fast jede Funktion kann auf fast jeden Pin). Konfiguriert wird per
   `menuconfig` — das betrifft Software-Einstellungen, nicht Pins.
2. **Die Umgebung wird pro Terminal aktiviert**, sie liegt nicht
   dauerhaft im Systempfad (siehe Abschnitt 1.2). Das ist die mit
   Abstand häufigste Stolperfalle für Einsteiger.

---

## VS Code in Kürze: Oberfläche und Tastenkürzel

Für alle, die neu in VS Code sind — vier Bereiche reichen zur
Orientierung: Die **Seitenleiste** links (Symbole für Dateien, Suche,
Git, Debug, Erweiterungen), der **Editor** in der Mitte, das **Panel**
unten (Terminal, Ausgabe, Probleme) und die **Statusleiste** ganz unten
(dort blendet die ESP-IDF-Erweiterung Ziel-Chip, Port und ihre Knöpfe
ein). Der wichtigste Griff überhaupt ist die **Befehlspalette**:
Strg+Umschalt+P öffnet ein Suchfeld, über das JEDER Befehl erreichbar
ist — wann immer diese Anleitung "ESP-IDF: ..." oder "Developer: Reload
Window" sagt, ist genau das gemeint: Palette öffnen, Befehl eintippen,
Enter.

| Kürzel | Wirkung |
|---|---|
| **Strg+Umschalt+P** | Befehlspalette — der Generalschlüssel zu allem |
| Strg+P | Datei im Projekt schnell öffnen (Namen tippen) |
| Strg+S | Speichern (vor jedem Build! ungespeichert = alter Stand) |
| Strg+Umschalt+E | Seitenleiste: Datei-Explorer |
| Strg+Umschalt+F | Seitenleiste: Suchen im ganzen Projekt |
| Strg+Umschalt+G | Seitenleiste: Quellcodeverwaltung (Git-Änderungen) |
| Strg+Umschalt+X | Seitenleiste: Erweiterungen |
| **Strg+Umschalt+D** | Seitenleiste: Ausführen & Debuggen (Dropdown + F5) |
| Strg+Ö (bzw. Strg+` auf US-Tastatur) | Panel: integriertes Terminal ein/aus |
| Strg+Umschalt+U | Panel: Ausgabe (Meldungen der Erweiterungen) |
| Strg+, | Einstellungen |
| **F5** | Debuggen: startet die im Dropdown gewählte Konfiguration |
| F10 / F11 | Im Debugger: Zeile überspringen / in Funktion hinein |
| Umschalt+F5 | Debug-Sitzung beenden |

Anders als bei manchen anderen Toolchains gibt es in unserem
ESP-IDF-Workflow **keinen eigenen Build-Tastendruck**: Gebaut wird mit
`idf.py build` im Terminal oder über die Knöpfe der ESP-IDF-Erweiterung
in der Statusleiste. F5 ist ausschließlich fürs JTAG-Debuggen zuständig.

### Ausgabe, Debug-Konsole, Terminal — wer zeigt was?

Im Panel unten liegen drei Reiter, die Einsteiger ständig verwechseln,
weil alle drei "Text ausgeben". Die Unterscheidung ist einfach, wenn man
fragt: WER redet hier?

- **Ausgabe (Strg+Umschalt+U)** — hier reden die **Erweiterungen**. Nur
  lesen, nichts eintippen; das Dropdown rechts oben wählt den Kanal.
  *Beispiel:* Die ESP-IDF-Erweiterung meldet dort, welchen Port und
  welches Ziel sie benutzt.
- **Debug-Konsole** — hier redet der **Debugger** während einer
  F5-Sitzung, und hier darf man mit ihm reden. *Beispiel:* Am
  Breakpoint den Namen einer Variablen eintippen und Enter — die
  Konsole zeigt ihren aktuellen Wert.
- **Terminal (Strg+Ö (bzw. Strg+` auf US-Tastatur))** — eine **echte Shell** im Projektordner.
  Hier tippst du selbst Befehle: `idf.py build`, `git status`. Wichtig:
  In diesem Terminal muss die ESP-IDF-Umgebung aktiviert sein (Abschnitt
  1.2). Auch der serielle Monitor (`idf.py monitor`) läuft hier — er ist
  im ESP32-Alltag das meistgenutzte Diagnosewerkzeug überhaupt.

Merkhilfe: **Ausgabe = Protokoll der Werkzeuge, Debug-Konsole = Dialog
mit dem Debugger, Terminal = Dialog mit dem System (und mit dem Board).**

---

## Teil 1: Rechner einrichten (einmalig pro Rechner)

### 1.1 ESP-IDF installieren

Espressif liefert für Windows einen Installer, der alles Nötige
mitbringt: ESP-IDF selbst, die Compiler für alle Chip-Familien, Python,
Git und die USB-Treiber.

Auf der Espressif-Downloadseite gibt es zwei Varianten:

- **Offline-Installer (empfohlen):** enthält alles, braucht während der
  Installation kein Netz — und liefert damit reproduzierbar exakt die
  Versionen, die im Paket stecken.
- **Online-Installer:** klein, lädt alles nach, erlaubt die Auswahl
  beliebiger ESP-IDF-Versionen.

Beim Durchklicken:

- **ESP-IDF-Version:** die aktuell stabile wählen (Stand dieser
  Anleitung: v6.0.x; v5.5 wird noch bis Anfang 2028 gepflegt). Welche
  Version aktuell stabil ist, steht auf der Releases-Seite von ESP-IDF.
- **Zielverzeichnis:** Vorgabe `C:\Espressif` übernehmen. Wichtig:
  **Der Pfad darf keine Leerzeichen und keine Umlaute enthalten** —
  das ist unter Windows die häufigste Ursache für rätselhafte
  Build-Fehler. Deshalb NICHT unter "Eigene Dateien" o. ä. installieren.
- **Treiber:** die angebotenen USB-Treiber (Silicon Labs CP210x, FTDI,
  WCH CH340) mitinstallieren.
- **VS-Code-Integration:** kann man mitnehmen, wir richten VS Code aber
  ohnehin in Schritt 1.4 selbst ein.

Alternativ gibt es inzwischen den **ESP-IDF Installation Manager (EIM)**
als grafisches Werkzeug, das mehrere Versionen nebeneinander verwaltet —
funktional gleichwertig, Geschmackssache.

Die installierte Version gehört in das README jedes Projekts. Mehrere
Versionen dürfen parallel unter `C:\Espressif\frameworks\` liegen.

### 1.2 Umgebung aktivieren — der wichtigste Unterschied zu anderen Toolchains

ESP-IDF liegt **nicht dauerhaft im PATH**. Der Installer legt dafür im
Startmenü Verknüpfungen an:

- **ESP-IDF PowerShell** (empfohlen)
- ESP-IDF Command Prompt (cmd.exe)

Diese Verknüpfung öffnet ein Terminal, in dem die Umgebung bereits
aktiviert ist — nur dort funktioniert `idf.py`. In einem normalen
PowerShell-Fenster aktiviert man sie von Hand:

```powershell
C:\Espressif\frameworks\esp-idf-v6.0.2\export.ps1
```

Kontrolle:

```powershell
idf.py --version
```

Merksatz für später: **Fast jedes "idf.py wird nicht erkannt" bedeutet,
dass die Umgebung in diesem Fenster nicht aktiviert ist** — auch im
Terminal von VS Code.

### 1.3 Zugriff auf die serielle Schnittstelle

Unter Windows erscheint das Board als COM-Port. Anstecken, dann im
**Geräte-Manager** unter *Anschlüsse (COM & LPT)* nachsehen, welche
Nummer vergeben wurde (z. B. COM5) — diese Nummer brauchst du gleich
beim Flashen.

- Taucht dort ein **"USB Serial Device"** auf, ist es das eingebaute
  USB-Serial-JTAG von ESP32-S3/C3/C6 — Windows 10/11 bringt den Treiber mit.
- Taucht ein Gerät mit Fragezeichen auf, fehlt der Treiber des
  USB-UART-Wandlers (CP210x, CH340, FTDI) — Installer erneut ausführen
  oder Treiber vom Chiphersteller nachinstallieren.

Für JTAG-Debugging muss der JTAG-Teil des Geräts dem WinUSB-Treiber
zugeordnet sein. Der Installer erledigt das in der Regel; falls
OpenOCD später mit "LIBUSB_ERROR_NOT_FOUND" abbricht, hilft das
Werkzeug **Zadig**: dort *Options → List All Devices*, in der Liste
"USB JTAG/serial debug unit (Interface 2)" auswählen und **WinUSB**
zuweisen. Wichtig: nur Interface 2 umstellen — der serielle Teil
(Interface 0) muss der COM-Port bleiben.


### 1.4 VS Code und Erweiterungen

```powershell
winget install Git.Git
winget install GitHub.cli
winget install Microsoft.VisualStudioCode
```

(Falls der ESP-IDF-Installer Git bereits mitgebracht hat, meldet winget
das und überspringt es.) Danach in einem NEUEN PowerShell-Fenster die
Zeilenenden-Strategie setzen, damit Windows- und Linux-Rechner dieselben
Repos ohne CRLF/LF-Rauschen teilen:

```powershell
git config --global core.autocrlf input
```

Erweiterungen (in VS Code: Strg+Umschalt+X, suchen, *Install*):

- **ESP-IDF** (Herausgeber: Espressif Systems) — Bedienoberfläche für
  bauen, flashen, monitoren, menuconfig und JTAG-Debugging
- **C/C++** (Microsoft) — Sprachunterstützung
- **CMake** (twxs) — Syntaxfarben für CMake-Dateien

Projekte, die eine `.vscode/extensions.json` mitbringen (siehe Abschnitt
2.6), schlagen diese Erweiterungen beim Öffnen automatisch vor. Kommt
keine Abfrage: Erweiterungs-Ansicht öffnen und `@recommended` ins
Suchfeld eingeben — die Rubrik "Workspace Recommendations" zeigt sie,
das Wolken-Symbol installiert alle fehlenden auf einmal.

Beim ersten Start fragt die ESP-IDF-Erweiterung nach dem
Installationspfad (Befehlspalette → **"ESP-IDF: Configure ESP-IDF
Extension"**). Dort "Use existing setup" wählen und auf die vorhandene
Installation zeigen — nicht noch einmal herunterladen lassen, sonst
liegen zwei ESP-IDF-Kopien auf dem Rechner und niemand weiß mehr, welche
gerade baut.

### 1.5 Git und GitHub einmalig verbinden

```powershell
git config --global user.name  "Vorname Nachname"
git config --global user.email "mail@zur-github-adresse.de"
gh auth login        # GitHub.com -> HTTPS -> "Login with a web browser"
```

Die E-Mail sollte die des GitHub-Kontos sein, damit Commits zugeordnet
werden. `gh auth login` erledigt die Anmeldung einmalig über den
Browser — danach funktioniert `git push` dauerhaft ohne Passwortabfrage.
(Das normale GitHub-Passwort funktioniert beim Push grundsätzlich NICHT
— GitHub verlangt Token, und genau die verwaltet `gh` für uns.)

---

## Teil 2: Neues Projekt anlegen (pro Projekt)

### 2.1 Projekt erzeugen und Ziel-Chip festlegen

Terminal öffnen, **Umgebung aktivieren** (Abschnitt 1.2), dann:

```powershell
cd C:\projekte                      # Pfad ohne Leerzeichen/Umlaute!
idf.py create-project Projekt123
cd Projekt123
idf.py set-target esp32s3           # esp32 | esp32s2 | esp32s3 | esp32c3 | esp32c6
```

`create-project` legt ein minimales, lauffähiges Projekt an;
`set-target` bestimmt den Chip und damit die verwendete Toolchain
(Xtensa oder RISC-V). Welcher Name zu welchem Chip gehört, steht im
Exkurs "Die Chip-Familien im Vergleich".

**Wichtig:** `set-target` erzeugt die Konfiguration neu und verwirft
dabei eine bestehende `sdkconfig` (die alte wird als `sdkconfig.old`
gesichert). Deshalb IMMER zuerst das Ziel setzen und Einstellungen
danach vornehmen — und dauerhafte Einstellungen ohnehin in
`sdkconfig.defaults` schreiben (Abschnitt 2.3).

Alternativ statt eines leeren Projekts ein offizielles Beispiel als
Startpunkt kopieren — oft der schnellere Weg:

```powershell
idf.py create-project-from-example "espressif/led_strip:led_strip_rmt_ws2812"
```

Namensregeln wie überall: kurz, ohne Leerzeichen und Umlaute — auch im
Pfad.

### 2.2 Erster Build, Flashen und Monitor

```powershell
idf.py build
idf.py -p COM5 flash
idf.py -p COM5 monitor
```

Oder alles in einem Rutsch — der Befehl, den man im Alltag am häufigsten
tippt:

```powershell
idf.py -p COM5 flash monitor
```

Erfolgskriterien: Der Build endet mit einer Größenübersicht und dem
Hinweis, wie geflasht wird; nach `flash` läuft `monitor` und zeigt den
Startvorgang des Chips. **Den Monitor beendet man mit Strg+], nicht mit
Strg+C** (Strg+C bricht mitten in der Ausgabe ab und lässt den Port
belegt zurück). Weitere nützliche Tastenfolgen im Monitor: Strg+T
gefolgt von Strg+R startet den Chip neu, Strg+T gefolgt von Strg+H zeigt
die Hilfe.

Speicherverbrauch anzeigen — die Zahlen gehören ins README:

```powershell
idf.py size                 # Gesamtübersicht
idf.py size-components      # aufgeschlüsselt nach Komponenten
```

In VS Code geht dasselbe über die Knöpfe der ESP-IDF-Erweiterung in der
Statusleiste (Ziel-Chip, Port, Build, Flash, Monitor) oder über die
Befehlspalette mit "ESP-IDF: Build your project" usw. Beides ruft
dieselben Werkzeuge auf — die Kommandozeile bleibt die Referenz.

### 2.3 Konfigurieren: menuconfig und sdkconfig.defaults

```powershell
idf.py menuconfig
```

öffnet die Konfigurationsoberfläche (Pfeiltasten navigieren, Leertaste
schaltet um, `/` sucht, `S` speichert, `Q` beendet). Hier stellt man
Logging, WLAN, Partitionstabelle, Optimierungsstufe, Stackgrößen und
Hunderte weiterer Software-Optionen ein.

Das Ergebnis landet in der Datei **`sdkconfig`** — und jetzt kommt der
wichtigste Punkt dieses Abschnitts:

> **`sdkconfig` gehört NICHT ins Repository, `sdkconfig.defaults` schon.**

Begründung: `sdkconfig` ist eine generierte Datei, die den kompletten
Zustand aller Optionen enthält (Tausende Zeilen), sich beim
Chip-Wechsel komplett ändert und bei jeder ESP-IDF-Aktualisierung neue
Einträge bekommt. Im Repository erzeugt sie endlose Konflikte. In
`sdkconfig.defaults` stehen dagegen nur die Optionen, die das Projekt
**bewusst** vom Standard abweichend setzt — kurz, lesbar, im `git diff`
sofort verständlich:

```
# sdkconfig.defaults - bewusst gesetzte Projekteinstellungen
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_FREERTOS_HZ=1000
CONFIG_ESP_MAIN_TASK_STACK_SIZE=4096
```

Wer eine Einstellung dauerhaft haben will, trägt sie also nach dem
Ausprobieren in `menuconfig` zusätzlich in `sdkconfig.defaults` ein.
Löscht jemand seine lokale `sdkconfig`, entsteht beim nächsten Build
automatisch wieder dieselbe Konfiguration.

### 2.4 Debug- und Release-Profil einrichten

ESP-IDF kennt keine fertigen "Presets" wie manch andere CMake-Umgebung,
bietet aber alles, um zwei saubere Profile zu bauen. Der Unterschied
steckt in der Optimierungsstufe und im Log-Umfang — und er ist real:
Ein Release-Build ist ein **anderes Maschinenprogramm** als der
Debug-Build.

Zwei zusätzliche Dateien anlegen:

`sdkconfig.defaults.debug`:

```
CONFIG_COMPILER_OPTIMIZATION_DEBUG=y
CONFIG_LOG_DEFAULT_LEVEL_DEBUG=y
CONFIG_COMPILER_OPTIMIZATION_ASSERTIONS_ENABLE=y
```

`sdkconfig.defaults.release`:

```
CONFIG_COMPILER_OPTIMIZATION_SIZE=y
CONFIG_LOG_DEFAULT_LEVEL_WARN=y
CONFIG_COMPILER_OPTIMIZATION_ASSERTIONS_SILENT=y
```

Gebaut wird dann in getrennte Ordner, damit sich die Profile nie
vermischen:

```powershell
# Debug
idf.py -B build-debug   -D SDKCONFIG=sdkconfig.debug \
       -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.debug" build

# Release
idf.py -B build-release -D SDKCONFIG=sdkconfig.release \
       -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.release" build
```

(`-B` wählt den Build-Ordner, `-D` setzt CMake-Variablen. Die Befehle
sind lang — deshalb gehören sie in ein kleines Skript oder in
`.vscode/tasks.json`, siehe Abschnitt 2.6.)

Daraus folgt die Regel, die für jedes ausgelieferte Gerät gilt:
**Entwickelt und gesucht wird im Debug — validiert und ausgeliefert wird
IMMER der Release.** Was nur im Debug getestet wurde, ist ungetestet.

Das Gegenstück zu bedingten Debug-Ausgaben ist bei ESP-IDF das
eingebaute Logging — es kennt Stufen und filtert im Release automatisch:

```c
#include "esp_log.h"

static const char *TAG = "melder";

ESP_LOGE(TAG, "Fehler: Sensor antwortet nicht");   // Error   - auch im Release
ESP_LOGW(TAG, "Messwert grenzwertig: %d", wert);   // Warning - auch im Release
ESP_LOGI(TAG, "Alarm ausgeloest");                 // Info    - nur im Debug-Profil
ESP_LOGD(TAG, "Flanken im Fenster: %u", flanken);  // Debug   - nur im Debug-Profil
```

Die Stufen unterhalb der eingestellten Grenze werden vom Compiler
**vollständig entfernt** — kein Flash-Verbrauch, keine Laufzeit, kein
versehentliches Geschwätz im Seriengerät. Genau deshalb braucht es keine
selbstgebauten `#ifdef`-Konstruktionen.

### 2.5 Ausgabedateien: .bin, Sammelabbild und .dfu

Nach dem Bauen liegen im Build-Ordner unter anderem:

```
build-debug/Projekt123.elf                     # mit Debug-Symbolen (fuer GDB und Monitor)
build-debug/Projekt123.bin                     # die Anwendung
build-debug/bootloader/bootloader.bin          # Bootlader
build-debug/partition_table/partition-table.bin
build-debug/flash_args                         # welche Datei an welche Adresse gehoert
```

Anders als bei vielen Mikrocontrollern gibt es also **nicht eine einzige
Flash-Datei**, sondern mehrere Abschnitte. Für die Entwicklung ist das
egal (`idf.py flash` weiß das alles), für die Fertigung nicht: Dort will
man **ein** Abbild, das ab Adresse 0 geschrieben wird. Das erzeugt man
so:

```powershell
cd build-release
esptool.py --chip esp32s3 merge_bin -o Projekt123-v1.0-merged.bin @flash_args
```

(In neueren ESP-IDF-Versionen gibt es dafür auch `idf.py merge-bin`.)
Dieses eine `-merged.bin` ist die Datei, die an einen Auftragsfertiger
geht oder in ein Produktionswerkzeug geladen wird.

**DFU (Firmware-Update über USB ohne Zusatzhardware)** unterstützt
ESP-IDF für **ESP32-S2 und ESP32-S3** — diese Chips haben eine
USB-OTG-Einheit. Für ESP32 (klassisch), C3 und C6 gibt es kein DFU;
dort geht der Weg über die serielle Schnittstelle oder über OTA:

```powershell
idf.py dfu              # erzeugt das DFU-Abbild
idf.py dfu-flash        # flasht es (Board vorher in den Download-Modus versetzen)
```

Für Updates an bereits ausgelieferten Geräten ist auf dem ESP32 aber
ohnehin **OTA über WLAN** der eigentliche Weg — siehe den OTA-Exkurs.

### 2.6 Projekt-Hygiene ergänzen (unser Standard)

Vier bis fünf Dateien gehören zusätzlich in jedes Projekt. Als
Beispielname dient überall `Projekt123`.

#### `.gitignore`

Sagt Git, was NICHT versioniert wird: alles Erzeugte und alle
maschinenspezifischen Dateien.

```gitignore
# ==== Build-Ausgaben ====
build/
build-debug/
build-release/

# ==== Generierte Konfiguration (Vorlagen liegen in sdkconfig.defaults*) ====
sdkconfig
sdkconfig.debug
sdkconfig.release
sdkconfig.old

# ==== Vom Component Manager heruntergeladene Fremdkomponenten ====
managed_components/

# ==== Editor- und Werkzeug-Caches ====
.cache/
.vscode/ipch

# ==== Betriebssystem-Artefakte ====
.DS_Store
Thumbs.db
desktop.ini
```

**Ausdrücklich NICHT ignorieren** (die gehören ins Repository):
`sdkconfig.defaults*`, `partitions.csv`, `dependencies.lock` (die
Sperrdatei des Component Managers — sie hält die exakten Versionen
fremder Komponenten fest, siehe Komponenten-Exkurs) und natürlich
`CMakeLists.txt`, `main/` und `components/`.

#### `.vscode/extensions.json`

VS Code liest seine Konfigurationsdateien als "JSON mit Kommentaren" —
`//`-Kommentare sind hier erlaubt.

```jsonc
{
    "recommendations": [
        "espressif.esp-idf-extension",   // bauen, flashen, monitoren, JTAG-Debugging
        "ms-vscode.cpptools",            // C/C++ Sprachunterstuetzung
        "twxs.cmake"                     // Syntaxfarben fuer CMake-Dateien

        // Optional, bewaehrt - zum Aktivieren einkommentieren
        // (und Komma am Vorgaenger ergaenzen!):
        // ,"eamodio.gitlens"            // Git-Historie im Editor: wer/wann/warum pro Zeile
        // ,"ms-vscode.hexeditor"        // .bin-Dateien als Hexdump ansehen
    ]
}
```

#### `.vscode/settings.json`

Projektbezogene Einstellungen — hier stehen Port und OpenOCD-Konfiguration,
damit jeder im Team dieselben Vorgaben hat.

```jsonc
{
    // Serieller Port des Boards - pro Arbeitsplatz ggf. anzupassen
    "idf.port": "COM5",

    // OpenOCD-Konfiguration passend zum Chip (siehe Chip-Exkurs):
    //   ESP32-S3 : board/esp32s3-builtin.cfg
    //   ESP32-C3 : board/esp32c3-builtin.cfg
    //   ESP32-C6 : board/esp32c6-builtin.cfg
    //   ESP32    : interface/ftdi/esp32_devkitj_v1.cfg + target/esp32.cfg (ESP-Prog noetig)
    "idf.openOcdConfigs": ["board/esp32s3-builtin.cfg"],

    // Editor-Anzeige an den echten Build koppeln:
    "C_Cpp.default.compileCommands": "${workspaceFolder}/build-debug/compile_commands.json"
}
```

Die Datei `compile_commands.json` erzeugt der Build automatisch. Sie
enthält für jede Quelldatei die exakten Compiler-Aufrufe — damit stimmt
die Fehleranzeige im Editor mit dem überein, was der Compiler
tatsächlich sieht.

#### `.vscode/launch.json`

Für JTAG-Debugging. Die ESP-IDF-Erweiterung bringt einen eigenen
Debug-Adapter mit (`"type": "espidf"`), der OpenOCD und GDB selbst
startet:

```jsonc
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "ESP-IDF Debug (JTAG)",
            "type": "espidf",              // Debug-Adapter der ESP-IDF-Erweiterung
            "request": "launch"
            // ,"initialBreakpoint": "app_main"   // gleich am Programmstart anhalten
        }
    ]
}
```

Voraussetzung ist ein Chip mit eingebautem JTAG (S3, C3, C6) oder eine
externe Sonde beim klassischen ESP32 — Details im Debug-Exkurs.
Wichtig: Debuggt wird der Stand, der auf dem Board LÄUFT; das ELF auf
der Platte muss dazu passen. Also erst flashen, dann debuggen.

#### `.vscode/tasks.json` (empfohlen)

Weil die Profil-Befehle aus Abschnitt 2.4 lang sind, hinterlegt man sie
einmal als Aufgaben und ruft sie danach über Strg+Umschalt+P →
"Tasks: Run Task" auf:

```jsonc
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build Debug",
            "type": "shell",
            "command": "idf.py -B build-debug -D SDKCONFIG=sdkconfig.debug -D SDKCONFIG_DEFAULTS=\"sdkconfig.defaults;sdkconfig.defaults.debug\" build",
            "group": { "kind": "build", "isDefault": true }
        },
        {
            "label": "Build Release",
            "type": "shell",
            "command": "idf.py -B build-release -D SDKCONFIG=sdkconfig.release -D SDKCONFIG_DEFAULTS=\"sdkconfig.defaults;sdkconfig.defaults.release\" build",
            "group": "build"
        }
    ]
}
```

Der Standard-Build lässt sich damit auch mit **Strg+Umschalt+B**
starten. Wichtig: Das Terminal, in dem die Aufgabe läuft, braucht die
aktivierte ESP-IDF-Umgebung.

#### `README.md`

Die Visitenkarte des Projekts — GitHub zeigt sie als Startseite. Regel:
Ein neuer Kollege muss allein mit dem README bauen, flashen und den
Stand einordnen können.

````markdown
# Projekt123 — <Einzeiler: was ist das Geraet/die Firmware?>

- Chip: <z. B. ESP32-S3-WROOM-1-N8R2, 8 MB Flash, 2 MB PSRAM>
- ESP-IDF: <VERSION EINTRAGEN, z. B. v6.0.2 — Pflichtangabe!>
- Besonderheiten: <WLAN-Provisionierung, OTA, Partitionslayout, ...>

## Voraussetzungen

ESP-IDF <Version> (Installation siehe Firmen-Anleitung), VS Code mit
den Erweiterungen aus .vscode/extensions.json.

## Bauen

```bash
idf.py -B build-debug -D SDKCONFIG=sdkconfig.debug \
       -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.debug" build
```

Referenzwerte (mit obiger ESP-IDF-Version):
- Debug:   App <...> Bytes
- Release: App <...> Bytes

## Flashen und Monitor

```bash
idf.py -p <PORT> flash monitor      # Monitor beenden: Strg+]
```

## Versionen

- v1.0 (<Datum>): <was ist drin> — gebaut mit ESP-IDF <Version>

## Offene Punkte

- <bekannte Baustellen, geplante Validierungen>
````

Warum die ESP-IDF-Version Pflicht ist: Sie legt Compiler, Bibliotheken
und Standardeinstellungen fest und macht damit jeden Build in Jahren
reproduzierbar. Warum Referenzgrößen: Wenn ein Build auf einem anderen
Rechner dieselben Zahlen liefert, ist das der schnellste Beweis, dass
die Umgebung korrekt eingerichtet ist.

### 2.7 Git und GitHub — was wir tun und warum

Kurz die Begriffe, dann die Befehle:

**Git** führt im versteckten Unterordner `.git` ein Protokoll über jeden
festgeschriebenen Stand des Projekts. Ein festgeschriebener Stand heißt
**Commit** — wie ein beschriftetes Foto des kompletten Projektordners zu
einem Zeitpunkt. **GitHub** ist der Server, auf den diese Commits
hochgeladen werden (**push**). `main` ist der Name der Hauptlinie
(**Branch**). Ein **Tag** ist ein Etikett an einem bestimmten Commit
("das hier ist v1.0").

Erst auf github.com ein LEERES Repository anlegen — ohne Häkchen bei
"Add README", sonst kollidiert es mit dem lokalen Stand. Dann im
Projektordner:

```powershell
git init                       # macht den Ordner zu einem Git-Projekt
git add .                      # alle Dateien in den "Warenkorb" legen
                               #   (alles, was .gitignore nicht ausschliesst)
git commit -m "Projektgeruest: idf.py create-project, Ziel esp32s3, ESP-IDF v6.0.2"
git branch -M main             # Hauptlinie "main" nennen (GitHub-Standard)
git remote add origin https://github.com/<benutzer>/<repo>.git
git push -u origin main        # hochladen; -u merkt sich die Verbindung
```

Den **frisch erzeugten Stand als ersten Commit** festhalten, bevor
eigener Code dazukommt — dann trennt die Historie für immer sauber
zwischen "hat das Werkzeug erzeugt" und "haben wir geändert".

Die drei Kontrollbefehle für den Alltag (kosten nichts, ändern nichts):

```powershell
git status          # Welche Dateien sind geaendert/neu? Wo stehe ich?
git diff            # Was GENAU wurde geaendert (Zeile fuer Zeile)? Beenden: q
git log --oneline   # Die Historie: ein Commit pro Zeile
```

Angewöhnen: **vor jedem Commit einmal `git status` und `git diff`** —
committen, was man gesehen und verstanden hat, nicht blind. In VS Code
geht dasselbe über die Quellcodeverwaltung (Strg+Umschalt+G) mit
Seite-an-Seite-Vergleich.

#### Serienstände veröffentlichen: GitHub-Releases

Binärdateien gehören nicht ins Repository — `build*/` steht ja in der
.gitignore. Trotzdem soll ein ausgelieferter Stand dauerhaft
griffbereit sein. Dafür gibt es **Releases**: Sie hängen Dateien an
einen Tag, also an exakt den Quellstand, aus dem sie gebaut wurden.

```powershell
git tag -a v1.0 -m "v1.0 - gebaut mit ESP-IDF v6.0.2"
git push origin v1.0
gh release create v1.0 build-release/Projekt123-v1.0-merged.bin \
   --title "v1.0" --notes "Erste Serienversion. Gebaut mit ESP-IDF v6.0.2."
```

Alternativ im Browser: Repo-Seite → **Releases** → *Draft a new release*
→ Tag wählen → Beschreibung → Datei per Drag & Drop anhängen →
*Publish release*.

#### Zusammenarbeit: Wie ein Kollege einsteigt

1. **Zugriff geben:** github.com → *Settings → Collaborators → Add
   people* (bei privaten Repos; öffentliche kann jeder lesen).
2. **Rechner einrichten** nach Teil 1 — mit **derselben ESP-IDF-Version,
   die im README steht**.
3. **Projekt holen und Umgebung testen:**

```powershell
git clone https://github.com/<benutzer>/<repo>.git
cd <repo>
idf.py -B build-debug -D SDKCONFIG=sdkconfig.debug \
       -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.debug" build
```

   Nutzt das Projekt Git-Submodule, gehört `--recurse-submodules` an das
   `clone`. Der erste Build ist zugleich der Umgebungstest: Die Größen
   müssen den Referenzwerten im README entsprechen.

**Der tägliche Rhythmus zu zweit — drei Regeln:** vor Arbeitsbeginn
`git pull`; klein und oft committen und pushen; nur gebaute Stände
pushen (`main` muss für alle jederzeit baubar sein).

**Wenn der Push abgelehnt wird** (`rejected — fetch first`): kein Fehler,
der Kollege war nur schneller. `git pull` holt und verschmilzt. Haben
beide dieselben Zeilen angefasst, entsteht ein **Merge-Konflikt**: Git
markiert die Stelle mit `<<<<<<<`, `=======`, `>>>>>>>`; VS Code bietet
dafür einen Merge-Editor mit Knöpfen. Vorgehen: Konflikt inhaltlich
entscheiden, Marker restlos entfernen, **bauen**, `git add`,
`git commit`, `git push`.

**Konflikte klein halten — zwei projektspezifische Absprachen:**

- Code in Komponenten aufteilen (siehe Struktur-Exkurs) — wer an
  verschiedenen Komponenten arbeitet, kollidiert gar nicht erst.
- **`sdkconfig.defaults` und `partitions.csv` ändert immer nur einer zur
  Zeit** und kündigt es an: Diese Dateien wirken sich auf das gesamte
  Projekt aus.

**Branches und Pull Requests** sind der nächste Schritt, sobald das Team
wächst oder eine riskante Änderung ansteht: `git switch -c feature/ota`,
arbeiten, `git push -u origin feature/ota`, auf GitHub "Compare & pull
request" — der Kollege sieht das komplette Diff, kommentiert an den
Zeilen, und erst nach Freigabe wird nach `main` gemergt. Eine Regel gilt
immer: **niemals `git push --force` auf `main`**.

---

## Teil 3: Der tägliche Arbeitszyklus

```
Code aendern -> speichern -> bauen -> flashen + Monitor -> git status/diff -> committen -> pushen
```

- Bauen und flashen:
  `idf.py -B build-debug ... build` bzw. `idf.py -p COM5 -B build-debug flash monitor`
  (oder die Aufgaben aus `tasks.json`, Strg+Umschalt+B).
- Der **Monitor ist das Hauptdiagnosewerkzeug**: `ESP_LOGI`-Ausgaben,
  und bei einem Absturz übersetzt er die Rückverfolgung automatisch in
  Dateinamen und Zeilennummern. JTAG-Debugging kommt erst dann ins
  Spiel, wenn Logging nicht mehr reicht.
- Committen: `git add -u` (geänderte, bereits versionierte Dateien),
  `git add <neue Datei>`, `git commit -m "..."`, `git push`.

#### Beispiel: kompletter Durchlauf vom Debug bis zur Auslieferung

Wichtigste Erkenntnis vorweg: **Für den Wechsel von Debug auf Release
wird KEINE Quelldatei geändert.** Genau dafür gibt es die zwei Profile —
derselbe Code, zwei Übersetzungen.

**Schritt 1 — Entwickeln, alles im Debug:**

```powershell
idf.py -B build-debug -D SDKCONFIG=sdkconfig.debug \
       -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.debug" build
idf.py -p COM5 -B build-debug flash monitor
```

**Schritt 2 — Release bauen (identischer Quellstand):**

```powershell
idf.py -B build-release -D SDKCONFIG=sdkconfig.release \
       -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.release" build
```

Ergebnis: kleineres Abbild, `ESP_LOGI`/`ESP_LOGD` sind verschwunden —
ohne dass eine Datei angefasst wurde.

**Schritt 3 — DIESEN Build flashen und auf der Hardware validieren:**

```powershell
idf.py -p COM5 -B build-release flash monitor
```

**Schritt 4 — Sammelabbild erzeugen, festschreiben, veröffentlichen:**

```powershell
cd build-release && esptool.py --chip esp32s3 merge_bin -o Projekt123-v1.0-merged.bin @flash_args && cd ..
git tag -a v1.0 -m "v1.0 - gebaut mit ESP-IDF v6.0.2"
git push origin v1.0
gh release create v1.0 build-release/Projekt123-v1.0-merged.bin --title "v1.0" --notes "Erste Serienversion."
```

#### Konfiguration ändern (menuconfig, Partitionen, Komponenten)

Das ESP32-Gegenstück zur "Hardware-Änderung": Alles, was das ganze
Projekt betrifft, wird bewusst und getrennt festgeschrieben.

**Schritt 0 — sauberer Ausgangspunkt:** `git status` muss leer sein,
sonst erst committen. Nur so zeigt das Diff ausschließlich die neue
Änderung.

**Schritt 1 — Ändern:** `idf.py menuconfig`, Option umstellen, speichern.

**Schritt 2 — Dauerhaft machen:** Die geänderte Option zusätzlich in
`sdkconfig.defaults` eintragen (sonst ist sie beim nächsten frischen
Klon weg). Den passenden Namen findet man in `menuconfig` mit `?` bei
markierter Option oder durch Vergleich:

```powershell
git diff sdkconfig.defaults
```

**Schritt 3 — Bauen und testen:**

```powershell
idf.py -B build-debug ... build      # muss durchlaufen
idf.py -p COM5 -B build-debug flash monitor
```

Niemals einen ungebauten Stand committen — sonst liegt ein kaputter
Zwischenstand in der Historie.

**Schritt 4 — Festschreiben:**

```powershell
git add -u
git commit -m "Konfiguration: FreeRTOS-Tick auf 1000 Hz"
git push
```

#### Und wenn etwas schiefging? Rückgängig machen in drei Stufen

*Stufe 1 — noch NICHT committet:*

```powershell
git restore main/main.c    # eine Datei auf den letzten Commit zuruecksetzen
git restore .              # ALLE offenen Aenderungen verwerfen (endgueltig!)
```

Klick-Weg: Quellcodeverwaltung → Pfeil-Symbol ("Änderungen verwerfen").

*Stufe 2 — schon committet:* nicht löschen, sondern umkehren.
`git revert` erzeugt einen neuen Commit, der den alten exakt rückgängig
macht; die Historie bleibt vollständig und ehrlich:

```powershell
git log --oneline          # Historie; vorne der Kurz-Hash (z. B. a1b2c3d)
git revert a1b2c3d         # diesen Commit umkehren
git push
```

*Stufe 3 — nur nachschauen oder eine Datei zurückholen:*

```powershell
git checkout a1b2c3d                        # alten Stand ansehen ("detached HEAD",
                                            #   nur Leseausflug - hier nichts committen)
git checkout main                           # zurueck in die Gegenwart
git restore --source a1b2c3d main/main.c    # nur EINE Datei aus einem alten Commit holen
```

Klick-Weg für die Datei-Historie: Datei im Explorer markieren → unten
die **Zeitachse (Timeline)** aufklappen.

Was es bewusst NICHT braucht: `git reset --hard` wirft committete
Historie unwiederbringlich weg — `revert` und `restore` decken alles ab
und bleiben nachvollziehbar.

---

## Exkurs: Die Chip-Familien im Vergleich

Alle hier genannten Chips werden mit demselben ESP-IDF und denselben
Befehlen entwickelt — der Unterschied liegt in Rechenkern, Toolchain und
vor allem darin, ob sie **JTAG und USB an Bord** haben.

| Chip | Kern(e) | Toolchain | `set-target` | USB-JTAG eingebaut | DFU |
|---|---|---|---|---|---|
| **ESP32** (klassisch) | 2× Xtensa LX6 | xtensa-esp-elf-gcc | `esp32` | nein — externe Sonde nötig | nein |
| **ESP32-S2** | 1× Xtensa LX7 | xtensa-esp-elf-gcc | `esp32s2` | ja | **ja** |
| **ESP32-S3** | 2× Xtensa LX7 | xtensa-esp-elf-gcc | `esp32s3` | ja | **ja** |
| **ESP32-C3** | 1× RISC-V | riscv32-esp-elf-gcc | `esp32c3` | ja | nein |
| **ESP32-C6** | 1× RISC-V | riscv32-esp-elf-gcc | `esp32c6` | ja | nein |

Was das praktisch bedeutet:

- **ESP32 (klassisch):** größte Modul- und Board-Auswahl, sehr
  erprobt, WLAN 4 + Bluetooth Classic/BLE. Zum Flashen sitzt auf den
  Boards ein USB-UART-Wandler (erscheint als `COM3`). Für echtes
  JTAG-Debugging braucht es eine externe Sonde (z. B. ESP-Prog, ~30 €) —
  in der Praxis arbeiten die meisten hier ausschließlich mit dem
  seriellen Monitor.
- **ESP32-S3:** zwei Kerne, viel RAM, PSRAM-Unterstützung, BLE, und —
  für die Entwicklung entscheidend — **USB-Serial-JTAG an Bord**:
  flashen, monitoren und debuggen über ein einziges USB-Kabel
  (`COM5`), ohne Zusatzhardware. Zusätzlich DFU-fähig. Wer die Wahl
  hat und Wert auf komfortables Debuggen und USB-Updates legt, ist hier
  richtig.
- **ESP32-C3:** günstig, sparsam, ein RISC-V-Kern, BLE, USB-JTAG an
  Bord. Ideal für einfache, stromsparende Geräte.
- **ESP32-C6:** wie C3, aber mit **WLAN 6 sowie Thread und Zigbee** —
  der Chip für moderne Gebäude- und Sensornetze (Matter).

Der Chip-Wechsel im Projekt ist ein einziger Befehl
(`idf.py set-target esp32c6`) — dabei wird aber die Konfiguration neu
erzeugt. Genau deshalb gehören alle bewussten Einstellungen in
`sdkconfig.defaults` und nicht nur in die generierte `sdkconfig`.

---

## Exkurs: Fehlersuche — Monitor, Absturzanalyse, JTAG

Auf dem ESP32 gibt es drei Stufen der Fehlersuche. Die meisten Probleme
löst Stufe 1 — deshalb in dieser Reihenfolge vorgehen.

**Stufe 1: Serieller Monitor und Logging.** `idf.py monitor` zeigt die
`ESP_LOG...`-Ausgaben. Das ist auf dem ESP32 kein Notbehelf, sondern das
übliche Vorgehen: Das System läuft mit FreeRTOS, WLAN-Stacks und
Interrupts weiter, während man zusieht — ein Breakpoint würde genau das
verhindern (und bei aktivem WLAN sogar Verbindungsabbrüche auslösen).

**Stufe 2: Absturzanalyse.** Stürzt die Firmware ab, gibt der Chip eine
Meldung wie `Guru Meditation Error: Core 0 panic'ed (LoadProhibited)`
samt Rückverfolgung aus. **Solange man `idf.py monitor` benutzt**,
übersetzt dieser die Adressen automatisch in Funktionsnamen, Dateien und
Zeilennummern — in einem beliebigen Terminalprogramm sieht man
stattdessen nur Zahlen. Das ist der wichtigste Grund, den mitgelieferten
Monitor zu verwenden. Für Abstürze im Feld lässt sich zusätzlich der
**Core-Dump** aktivieren (`menuconfig` → Core dump → Flash), der beim
nächsten Start ausgelesen werden kann.

**Stufe 3: JTAG-Debugging** (Breakpoints, Einzelschritt, Variablen). Bei
S3/C3/C6 ohne Zusatzhardware:

```powershell
idf.py -B build-debug openocd          # in einem Terminal laufen lassen
idf.py -B build-debug gdb              # in einem zweiten Terminal
```

Bequemer über VS Code: F5 mit der Konfiguration aus Abschnitt 2.6. Beim
klassischen ESP32 ist dafür eine externe Sonde (ESP-Prog) nötig, die
mit vier Leitungen (TMS, TCK, TDI, TDO) plus GND am Board hängt; die
passende OpenOCD-Konfiguration steht dann in `idf.openOcdConfigs`.

Zwei Punkte, die man beim JTAG-Debuggen auf dem ESP32 kennen muss:
Erstens muss der geflashte Stand zum ELF auf der Platte passen — also
erst flashen, dann debuggen. Zweitens vertragen sich lange Haltepunkte
schlecht mit aktiven Funkverbindungen und Watchdogs; für zeitkritische
Abläufe ist Logging oft aussagekräftiger als ein Breakpoint.

---

## Exkurs: Komponenten und Bibliotheken einbinden

Fremdcode kommt bei ESP-IDF auf drei Wegen ins Projekt. Der erste ist
der Normalfall.

**Weg 1: Component Manager (empfohlen).** ESP-IDF hat eine offizielle
Komponentenregistrierung; Abhängigkeiten werden deklariert und
automatisch geladen:

```powershell
idf.py add-dependency "espressif/led_strip^2.5.0"
```

Das trägt die Abhängigkeit in `main/idf_component.yml` ein, lädt die
Komponente beim nächsten Build nach `managed_components/` und schreibt
die exakt verwendeten Versionen in **`dependencies.lock`**. Wichtig für
die Reproduzierbarkeit: **`dependencies.lock` gehört ins Repository**,
`managed_components/` nicht (Letzteres wird jederzeit neu geladen).
Damit baut jeder Kollege garantiert mit denselben Fremdkomponenten.

**Weg 2: Git-Submodule.** Sinnvoll für eigenen, projektübergreifend
genutzten Firmencode oder für Bibliotheken, die nicht in der
Registrierung stehen:

```powershell
git submodule add https://github.com/<...>/<lib>.git components/<lib>
cd components/<lib> && git checkout v2.3.0 && cd ../..
git add . && git commit -m "Komponente <lib> v2.3.0 eingebunden"
```

Ein Submodule speichert im eigenen Repo nicht die fremden Dateien,
sondern nur einen **Zeiger** auf ein exaktes Commit/Tag — die Version
ist damit technisch garantiert, nicht bloß notiert. Updates sind ein
Mini-Commit ("v2.3.0 → v2.4.0") statt eines Riesen-Diffs aus
Ordner-löschen-und-neu-kopieren, bei dem lokal hineingeflickte
Änderungen unbemerkt überschrieben würden. Kollegen klonen solche
Projekte mit `git clone --recurse-submodules`.

**Weg 3: Einfach hineinkopieren ("Vendoring").** Ordner unter
`components/` legen, fertig. Ehrlich gesagt der Weg, der am häufigsten
Ärger macht: Die Version ist unsichtbar, Updates sind riskant. Er ist
dann richtig, wenn der Code klein ist, ohnehin angepasst wird und
dauerhaft zum Projekt gehört.

Faustregel: **Aus der Registrierung → Component Manager. Eigener oder
externer Git-Code → Submodule mit Versions-Tag. Nur was wirklich
Projekteigentum wird → Kopie.**

*Update-Ablauf im Vergleich:* Beim Component Manager ändert
`idf.py update-dependencies` (bzw. eine angepasste Version in
`idf_component.yml`) die Datei `dependencies.lock` — das `git diff` ist
kurz und zeigt exakt die Versionssprünge. Beim Submodule wechselt man im
Unterordner den Tag; das Diff im Hauptprojekt besteht aus **einer**
Zeile (`-Subproject commit …` / `+Subproject commit …`). Was sich
inhaltlich geändert hat, liest man in beiden Fällen im Änderungsprotokoll
des Fremdprojekts. Danach: bauen, auf der Hardware gegentesten, als
eigenen Commit festschreiben.

---

## Exkurs: Projektstruktur — Komponenten statt alles in main.c

Ein frisch erzeugtes Projekt hat nur `main/main.c`. Für alles jenseits
eines Testprogramms lohnt es sich, den Code in **Komponenten** zu
schneiden — das ESP-IDF-Gegenstück zu Modulen:

```
Projekt123/
├── CMakeLists.txt              # Projekt-Wurzel
├── sdkconfig.defaults          # bewusste Einstellungen (versioniert)
├── sdkconfig.defaults.debug
├── sdkconfig.defaults.release
├── partitions.csv              # eigene Partitionstabelle (optional)
├── dependencies.lock           # exakte Versionen der Fremdkomponenten
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml       # Abhaengigkeiten (Component Manager)
│   └── main.c                  # nur noch Ablauflogik
└── components/
    ├── sensor/
    │   ├── CMakeLists.txt
    │   ├── sensor.c
    │   └── include/sensor.h
    └── anzeige/
        ├── CMakeLists.txt
        ├── anzeige.c
        └── include/anzeige.h
```

Eine Komponente registriert sich in ihrer eigenen `CMakeLists.txt`:

```cmake
idf_component_register(SRCS "sensor.c"
                       INCLUDE_DIRS "include"
                       REQUIRES driver esp_timer)
```

`SRCS` sind die Quelldateien, `INCLUDE_DIRS` die Ordner mit den
öffentlichen Headern (dieser Pfad wird für alle sichtbar, die die
Komponente benutzen), `REQUIRES` nennt die benötigten anderen
Komponenten. Mehr braucht es nicht — ESP-IDF findet alles unter
`components/` automatisch, es muss **nichts** in der Wurzel-CMakeLists
eingetragen werden.

Vorteile: Auffindbarkeit ("wo ist der Sensor?" → `components/sensor`),
kleine und präzise `git diff`s, weniger Konflikte bei Teamarbeit,
klare Schnittstellen über die Header — und einzelne Komponenten lassen
sich später per Submodule in andere Projekte übernehmen. Nachteil: ein
paar Dateien mehr, was in der Praxis niemanden stört.

Empfehlung: Bei jedem neuen Projekt von Anfang an so schneiden. Bei
bestehenden Projekten lohnt der Umbau meist erst, wenn `main.c`
unübersichtlich wird — dann Komponente für Komponente, jeweils als
eigener Commit.

---

## Exkurs: OTA — Firmware-Updates im Feld über WLAN

Auf dem ESP32 ist der übliche Weg für Updates an ausgelieferten Geräten
nicht USB, sondern **OTA** (Over-the-Air): Das Gerät lädt selbst ein
neues Abbild und startet damit. Drei Bausteine gehören dazu:

**1. Partitionslayout.** Es braucht zwei App-Partitionen, damit das
neue Abbild geschrieben werden kann, während das alte noch läuft. Dafür
eine eigene `partitions.csv` anlegen (versioniert!):

```
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     ,        0x6000
otadata,  data, ota,     ,        0x2000
phy_init, data, phy,     ,        0x1000
ota_0,    app,  ota_0,   ,        1500K
ota_1,    app,  ota_1,   ,        1500K
```

und in `sdkconfig.defaults` aktivieren:

```
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
```

Wichtig: Die App muss in **eine** dieser Partitionen passen — die
Größen also großzügig, aber zum Flash-Bausteins des Moduls passend
wählen.

**2. Ablauf im Gerät.** ESP-IDF bringt die Bibliothek `esp_https_ota`
mit: Das Gerät prüft (z. B. per HTTPS und Versionsvergleich), ob ein
neues Abbild vorliegt, lädt es in die freie Partition, markiert sie als
Startpartition und startet neu. Schlägt der Start fehl oder meldet die
neue Firmware sich nicht als "gut", fällt der Bootlader auf die alte
Partition zurück (Rollback) — genau das macht OTA feldtauglich.

**3. Verteilung.** Die `.bin` der Anwendung (nicht das Sammelabbild!)
wird auf einen Server gelegt. Praktischerweise ist das exakt die Datei,
die auch am GitHub-Release hängt — Version und Abbild bleiben so
verknüpft.

Sicherheitshinweise, die zur Serienreife gehören: HTTPS mit geprüftem
Zertifikat verwenden, in der Firmware eine **Versionsprüfung** einbauen
(kein Downgrade), und für Produkte mit erhöhtem Schutzbedarf **Secure
Boot** und **Flash-Verschlüsselung** einschalten — beides erst nach
gründlicher Lektüre der Espressif-Dokumentation, da diese Optionen
teilweise unwiderruflich in den Chip gebrannt werden.

Wer USB-Updates bevorzugt: Bei ESP32-S2/S3 ist zusätzlich der DFU-Weg
möglich (Abschnitt 2.5), bei allen Chips das Flashen über die serielle
Schnittstelle.

---

## Teil 4: Checkliste für jedes neue Projekt

- [ ] ESP-IDF-Umgebung im Terminal aktiviert (`get_idf` / ESP-IDF-PowerShell)
- [ ] Projekt mit `idf.py create-project` angelegt, Name und Pfad ohne
      Leerzeichen/Umlaute
- [ ] `idf.py set-target <chip>` als ERSTES ausgeführt
- [ ] Erster Build läuft, Flashen und Monitor getestet
- [ ] Bewusste Einstellungen in `sdkconfig.defaults` (nicht nur in `sdkconfig`)
- [ ] Debug- und Release-Profil angelegt (`sdkconfig.defaults.debug/.release`)
- [ ] `.gitignore`, `.vscode/`, `README.md` ergänzt — inkl. **ESP-IDF-Version**
      und Referenz-Buildgrößen
- [ ] Erzeugter Stand = erster Commit, Repo auf GitHub, Push erfolgreich
- [ ] Code in Komponenten geschnitten statt alles in `main.c`
- [ ] `dependencies.lock` versioniert (falls Fremdkomponenten genutzt)
- [ ] Bei OTA-Produkten: eigene `partitions.csv` versioniert, Rollback getestet
- [ ] Release-Build gebaut, auf Hardware validiert, getaggt und als
      GitHub-Release veröffentlicht

---

## Teil 5: Troubleshooting — typische Fehler und ihre Lösung

**"idf.py: Kommando nicht gefunden" / "wird nicht erkannt"**
Die ESP-IDF-Umgebung ist in diesem Terminal nicht aktiviert — mit
Abstand der häufigste Fehler. Siehe Abschnitt 1.2. Gilt auch für das
Terminal in VS Code und für Aufgaben aus `tasks.json`.

**"Der Befehl idf.py wird nicht erkannt"**
Die ESP-IDF-Umgebung ist in diesem Fenster nicht aktiviert. Terminal
über die Startmenü-Verknüpfung **ESP-IDF PowerShell** öffnen — oder
`export.ps1` ausführen (Schritt 1.2). Gilt genauso für das Terminal
INNERHALB von VS Code.

**Build bricht mit rätselhaften Pfadfehlern ab**
Fast immer Leerzeichen, Umlaute oder ein zu langer Pfad. ESP-IDF und
Projekte gehören nach `C:\Espressif` bzw. `C:\projekte` — nicht unter
"Eigene Dateien"/OneDrive. Zusätzlich in Windows lange Pfade erlauben
(Gruppenrichtlinie bzw. Registry `LongPathsEnabled`).

**Kein COM-Port im Geräte-Manager / Gerät mit Fragezeichen**
Treiber des USB-UART-Wandlers fehlt (CP210x, CH340, FTDI) — den
ESP-IDF-Installer erneut ausführen und die Treiberoption mitnehmen. Bei
S3/C3/C6 mit eingebautem USB: anderes Kabel probieren (viele
USB-C-Kabel sind reine Ladekabel ohne Datenleitungen).

**OpenOCD: "LIBUSB_ERROR_NOT_FOUND"**
Dem JTAG-Interface fehlt der WinUSB-Treiber. Mit **Zadig** dem Eintrag
"USB JTAG/serial debug unit (Interface 2)" WinUSB zuweisen — Interface 0
(der COM-Port) bleibt unangetastet.

**Virenscanner/SmartScreen bremst oder blockiert**
Espressif-Installer sind signiert ("Weitere Informationen → Trotzdem
ausführen"). Bei Firmen-Virenscannern `C:\Espressif` und den
Projektordner als Ausnahme eintragen lassen — sonst wird jeder Build
quälend langsam.

**Flashen scheitert: "Failed to connect to ESP32: Timed out waiting for packet header"**
Der Chip ist nicht im Download-Modus. Abhilfe in dieser Reihenfolge:
BOOT-Taste gedrückt halten, kurz RESET/EN drücken, BOOT loslassen, dann
erneut flashen. Weitere Kandidaten: reines Ladekabel ohne Datenleitungen,
zu hohe Baudrate (`idf.py -b 115200 flash`), belegter Port (läuft noch
ein Monitor?), zu schwache Stromversorgung.

**Monitor zeigt nur wirre Zeichen**
Falsche Baudrate — Vorgabe ist 115200 (`CONFIG_ESP_CONSOLE_UART_BAUDRATE`).
Wirre Zeichen NUR in den ersten Zeilen sind dagegen normal: Das ist die
Ausgabe des ROM-Bootladers mit anderer Baudrate.

**"Guru Meditation Error" / das Gerät startet ständig neu**
Ein Absturz. Immer `idf.py monitor` verwenden — dieser übersetzt die
Rückverfolgung in Dateinamen und Zeilennummern. Häufige Ursachen:
Zugriff über einen Null-Zeiger, Stapelüberlauf einer Task (Stackgröße in
`menuconfig` erhöhen), oder eine zu kleine Stromversorgung, wenn der
Neustart genau beim Einschalten des Funkmoduls passiert
("Brownout detector was triggered" — anderes Kabel, anderer USB-Port
oder externes Netzteil).

**`set-target` hat meine Konfiguration gelöscht**
Das ist beabsichtigt: Die Konfiguration ist chip-spezifisch und wird neu
erzeugt (die alte liegt als `sdkconfig.old` daneben). Genau deshalb
gehören bewusste Einstellungen in `sdkconfig.defaults` — von dort werden
sie automatisch wieder übernommen.

**Nach einem ESP-IDF-Update baut das Projekt nicht mehr**
Zuerst gründlich aufräumen: `idf.py fullclean`, dann neu bauen. Danach
prüfen, ob die neue ESP-IDF-Version Änderungen verlangt (Espressif
veröffentlicht zu jedem größeren Release einen Migrationsleitfaden).
Grundsätzlich gilt: ESP-IDF-Updates sind ein eigener Commit und werden
auf der Hardware gegengetestet — nicht nebenbei mitgemacht.

**`idf.py build` meldet fehlende Komponenten oder Header**
Häufig fehlt der Eintrag in `REQUIRES` der eigenen Komponente (siehe
Struktur-Exkurs) oder eine Abhängigkeit im `idf_component.yml`. Nach
Änderungen an Komponenten hilft ebenfalls `idf.py fullclean`.

**VS Code: Warnung wegen doppelter IntelliSense-Motoren**
Wer zusätzlich zur C/C++-Erweiterung einen clangd-basierten Motor
installiert hat, bekommt doppelte Fehleranzeigen. Einen von beiden
deaktivieren — in den Einstellungen (Reiter "Workspace")
`C_Cpp: Intelli Sense Engine` auf `disabled`, wenn clangd gewinnen soll.
Auf Bauen und Flashen hat das keinen Einfluss.

**Editor unterkringelt korrekten Code / kennt ESP-IDF-Funktionen nicht**
Die Verknüpfung zur Build-Datenbank fehlt: `C_Cpp.default.compileCommands`
in `.vscode/settings.json` auf die `compile_commands.json` des
Build-Ordners zeigen lassen (Abschnitt 2.6) und einmal bauen, damit die
Datei existiert.

**ESP-IDF-Erweiterung findet die Installation nicht**
Befehlspalette → "ESP-IDF: Configure ESP-IDF Extension" → "Use existing
setup" und auf den Installationspfad zeigen. Niemals parallel eine
zweite Kopie herunterladen lassen.

**`git status`: "nichts zu committen" — obwohl etwas geändert wurde**
Fast immer: falscher Ordner, oder die Änderung ist nie angekommen
(z. B. Datei woanders entpackt). Aktuellen Pfad prüfen, dann `git diff`
— ist der leer, wurde real nichts geändert.

**`git push` → Authentifizierungsfehler**
GitHub akzeptiert keine Kontopasswörter. `gh auth login` ausführen
(Abschnitt 1.5), danach klappt der Push dauerhaft.

**Build ist plötzlich sehr langsam**
Häufig der Virenscanner (Build-Ordner und Toolchain als Ausnahme
eintragen) oder ein Projekt auf einem Netzlaufwerk bzw. in einem
Cloud-Sync-Ordner — Projekte gehören auf die lokale Platte.

## Lizenz

[MIT](LICENSE).

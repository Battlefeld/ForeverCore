# ForeverCore – Konzept (Neustart)

Stand: 13.11.2026 · Status: **Entwurf zur Abstimmung**, noch kein Code.

ForeverCore ist ein All-in-One-Addon für **WoW: Forever**: **Levelroute, Skillung, Rotation und Item-Vergleich** in
einem Addon mit einer gemeinsamen Oberfläche. Schwere Berechnungen (Kampf-Simulation, Talent-Optimierung,
Routenplanung) laufen in einem Begleitprogramm auf dem PC (`ForeverCore.exe`), das Addon zeigt die Ergebnisse an.

Der Code wird **komplett neu geschrieben**. Das alte Projekt [ForeverSim](https://github.com/Battlefeld/ForeverSim)
dient nur als Nachschlagewerk: Was dort über den Client, den Transport und die Engine herausgefunden wurde, fließt hier
als Erkenntnis ein, nicht als Code.

---

## 1. Lehren aus ForeverSim

| Problem damals | Entscheidung jetzt |
|---|---|
| Sehr viele Funktionen gleichzeitig, fast nichts im echten Client bestätigt | **Jeder Meilenstein endet mit einer Ingame-Abnahme.** Erst wenn sie bestanden ist, kommt das nächste Feature. |
| Drei Addons (ForeverCoreBridge, ForeverSimBridge, ForeverRoutes) plus 2.048 Slot-Addons | **Ein** Addon `ForeverCore` mit Modulen. Nur die Antwortslots bleiben technisch bedingt eigene Ordner (siehe 3.2). |
| Routenplanung über ein zweites Programm (`foreverroutes.exe`) und `Lua.exe` | Planung direkt in `ForeverCore.exe` (Go), keine weiteren Programme. |
| Viele Sonderfälle und Rückfallstufen, bevor die Grundfunktion bewiesen war | Erst der einfachste funktionierende Weg, Erweiterungen nur bei gemessenem Bedarf. |
| Unklare Rechte an Routendaten (RestedXP-Abschnitte unter CC BY-NC-SA) | Nur Quellen mit klarer Lizenz; eigene Routen werden aus Questdaten **berechnet**, nicht von Guides übernommen. |

## 2. Funktionsumfang (Ziel)

| Bereich | Im Spiel | Auf dem PC |
|---|---|---|
| **Levelroute** | Fenster mit den nächsten Schritten (Annehmen, Erledigen, Abgeben), Wegpunkt, Fortschritt aus dem Questlog | Route ab dem aktuellen Stand planen (Questdaten, Kampfzeit aus der Simulation) |
| **Skillung** | Empfohlene Talentverteilung für Stufe und Rolle, Vergleich mit der aktuellen | Talent-Optimierung per Simulation |
| **Rotation** | Prioritätenliste und optional ein schlankes Kampf-HUD (nächster Zauber) | Rotation per Simulation bewerten |
| **Item-Vergleich** | „+x % DPS“ im Tooltip (Beute, Quest-Belohnungen, Taschen, Auktionshaus) | Simulation mit und ohne Item, gebündelt |

Gemeinsam: ein Hauptfenster mit Reitern (`/fc`), Minimap-Knopf mit Verbindungsstatus, ein Cache, der nur bei echten
Änderungen (Gear, Talente, Level) ungültig wird.

**Nicht geplant:** automatisierte Spielaktionen, Lesen geschützter Daten, Kampfentscheidungen, die auf eine Antwort vom
PC warten müssen.

## 3. Architektur

```
WoW-Client                                       PC
──────────                                       ──
ForeverCore (Addon)                              ForeverCore.exe (Go)
 ├─ Core       Ereignisse, Zustand, Cache          ├─ Capture   liest den Pixelstreifen
 ├─ Transport  Pixelstreifen ──────────────────►   ├─ Jobs      Auftragswarteschlange
 │             Antwortslots  ◄──────────────────   ├─ Reply     schreibt Antwortslots
 ├─ Character  Gear, Talente, Zauber, Quests       ├─ Sim       WoWSims-Engine (eingebettet)
 ├─ Items      Tooltip-Vergleich                   ├─ Talents   Optimierung
 ├─ Talents    Anzeige + Vergleich                 ├─ Route     Routenplaner
 ├─ Rotation   Liste + HUD                         ├─ Data      Item-/Quest-/Zauberdaten
 ├─ Route      Schritte, Wegpunkt                  └─ Setup     Installation, Updates
 └─ UI         Hauptfenster, Minimap
```

### 3.1 Hinweg: Addon → PC (Pixelstreifen)

Addons können nicht mit anderen Programmen sprechen. Der bewährte Weg aus ForeverSim: Das Addon malt einen schmalen
Streifen aus hellen und dunklen Zellen oben links ins Fenster, die EXE liest ihn per **DXGI Desktop Duplication**.
(Das ältere `GetDC`/`BitBlt` hat auf einer AMD-Karte mit D3D12 das Bild gestört und kommt nicht wieder.)

- Ein Rahmen trägt rund 240 Byte Nutzdaten plus Kopf mit Prüfsumme.
- Der Streifen ist nur während einer Übertragung sichtbar, im Kampf wird nicht gesendet.
- WoW muss im Fenster- oder Fenster-Vollbildmodus laufen.

### 3.2 Rückweg: PC → Addon (Antwortslots)

Die EXE schreibt Antworten in vorab installierte **LoadOnDemand-Addons**, die das Addon per `LoadAddOn` nachlädt.
Jedes Slot-Addon lässt sich pro Sitzung nur einmal laden, deshalb gibt es einen Vorrat an Slots, und mehrere Ergebnisse
werden in einer Antwort gebündelt. WoW findet Addons nur als eigene Ordner direkt unter `Interface/AddOns`, daher bleiben
die Slots einzelne Ordner, die die EXE anlegt und verwaltet. Die Slot-Anzahl legen wir nach der Messung in M1 fest.

Große, selten wechselnde Daten (Route, empfohlene Skillung) können zusätzlich als Datei vor dem Spielstart oder vor einem
`/reload` geschrieben werden.

### 3.3 Simulation

Grundlage ist die **WoWSims-Forever-Engine** (MIT), fest auf eine Version gepinnt und in die EXE eingebunden. Eine eigene
Kampf-Simulation schreiben wir nicht. Für Stufen unter 60 braucht die Engine eine Levelschicht (Grundwerte,
Zauberränge, Gegnerwerte); die bauen wir neu und sauber getestet auf.

### 3.4 Technik

| Teil | Sprache / Werkzeug |
|---|---|
| Addon | Lua 5.1 (WoW-API des Forever-Clients) |
| PC-Programm | Go, eine einzelne `ForeverCore.exe` für Windows |
| Tests | Lua-Tests mit nachgebautem Client, Go-Tests; beides in GitHub Actions |
| Build und Release | GitHub Actions baut die EXE aus genau diesem Repo |

## 4. Meilensteine

Jeder Meilenstein ist erst fertig, wenn die automatischen Tests grün sind **und** die Ingame-Abnahme bestanden ist.

| # | Ergebnis | Ingame-Abnahme |
|---|---|---|
| **M0** | Repo-Grundgerüst: Addon-Gerüst, Go-Modul, CI, Lizenz | Addon lädt ohne Fehler, `/fc` öffnet ein leeres Fenster |
| **M1** | **Durchstich:** `/fc ping` geht über Pixelstreifen zur EXE und über einen Slot zurück | Ping kommt an; gemessen werden Latenz, Fehlerquote und FPS-Einfluss |
| **M2** | Charakter-Abbild (Klasse, Stufe, Gear, Talente) → Simulation → DPS im Fenster | Angezeigte DPS sind plausibel, Gear-Wechsel löst eine neue Rechnung aus |
| **M3** | Item-Vergleich im Tooltip mit Cache und Bündelung | Prozentwerte an Beute, Quests und Taschen; Buffs/Reiten lösen keine Rechnung aus |
| **M4** | Skillung und Rotation | Empfehlung erscheint, Übernahme der Prioritäten funktioniert |
| **M5** | Levelroute | Schritte und Wegpunkt stimmen mit dem Spiel überein (Questgeber am richtigen Ort) |
| **M6** | Installation und Updates aus der EXE, Release | Neuinstallation auf einem frischen PC ohne Handarbeit |

## 5. Offene Fragen

1. **Client:** Gilt weiterhin WoW: Forever 1.60.x als Ziel? Welche Interface-Nummer für die `.toc`?
2. **Lizenz:** MIT für den eigenen Code wie bisher? Wenn wir Daten unter GPL-3.0 einbinden (z. B. Questdaten), muss der
   betroffene Teil unter GPL-3.0 stehen.
3. **Routendaten:** Woher die Quest-Daten (Questgeber, Ziele, Koordinaten)? Kandidaten: QuestieDB, cmangos classic-db,
   ForeverDB. Was hat der Forever-Client an eigenen Quests oder Änderungen?
4. **ForeverRoutes:** Geht das alte Routen-Addon vollständig in ForeverCore auf und wird eingestellt?
5. **Name im Spiel:** Slash-Befehl `/fc` und Addon-Name `ForeverCore` passt?

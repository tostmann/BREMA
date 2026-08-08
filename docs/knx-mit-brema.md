# KNX mit BREMA orchestrieren — HowTo für LLM-Agenten, Beispiel Claude Code

Stand 2026-08-08. Grundlage ist eine echte Sitzung: ein Merten 649802 Jalousieaktor (Maske 0x0701, App `M-000C_A-5700-11-8E6A`) wurde **ohne ETS** repariert, verdrahtet, an Home Assistant übergeben und parametriert — von einem LLM-Agenten orchestriert. Alles Beschriebene ist so passiert; die Fehlschläge stehen mit drin, weil sie der lehrreichste Teil sind.

**Die Frage, die dieses Dokument beantwortet:** Du hast einen busware-KNX-Stick am Bus und einen LLM-Agenten. Wie kommst Du von dort zu einer Installation, die ein normaler Home-Assistant-Nutzer bedienen kann — und was musst Du dem Agenten geben, verbieten und abverlangen, damit dabei nichts kaputtgeht?

## 0. Das Bild: wer redet mit wem

```
   Claude Code  ──MCP──►  busware-knx-stick  ──MQTT──►  TUL32  ──TP1──►  KNX-Bus
       │                    (Python-Plugin)              (C6 + NCN5130)      │
       │                                                                     │
       └──MCP──►  ha-mcp  ──►  Home Assistant  ◄──MQTT (Discovery/State)─────┘
```

Zwei Dinge daran sind wichtiger, als sie aussehen.

**Der Agent redet nie direkt mit dem Bus.** Er ruft Verben auf einer Admin-Ebene; die Firmware entscheidet, was daraus auf dem Draht wird. Das ist die Stelle, an der Sicherheitsregeln durchsetzbar sind statt bloß erbeten — ein Agent kann eine Regel „bitte kein scharfer Write" ignorieren, ein `dry_run`-Default in C nicht.

**Home Assistant ist ein zweiter, gleichrangiger Konsument, kein Anhängsel.** Der Agent baut die Installation auf; danach bedient sie ein Mensch in HA, ohne je einen Verbnamen zu sehen. Wenn Du den Agenten als Dauer-Bediener einplanst, hast Du die Arbeitsteilung falsch geschnitten.

Optional, aber in dieser Sitzung entscheidend: **ein zweiter MCP-Server für HA** (`ha-mcp`). Damit kann der Agent prüfen, was HA aus seiner Arbeit gemacht hat — und das ist etwas anderes, als zu prüfen, was er selbst publiziert hat. Zwei der drei echten Fehler dieser Sitzung wurden nur dadurch gefunden.

## 1. Aufsetzen

**Schritt 0 — Firmware.** Der Web-Flasher https://install.busware.de/TUL/ führt „MCP for TUL (KNX rule engine + MCP server)" im Dropdown (ESP32-C6; Chrome/Edge/Opera). WLAN wird direkt danach in derselben USB-Sitzung per Improv gesetzt, ohne dass Zugangsdaten in die Firmware wandern. Der Stick zeigt seine eigene Seite unter `http://busware-knx-<id>.local/` — dort stehen mDNS-Name, beide Adressen, die Hardware-Kennung und das fertige Kommando. Beschreibung der Firmware: https://install.busware.de/TUL/mcp/

**Der Stick ist selbst der MCP-Server, es ist nichts zu installieren:** `claude mcp add --transport http busware-knx http://busware-knx-<id>.local/mcp` — JSON-RPC 2.0 über HTTP POST, die Tools werden aus der Verb-Registry der Firmware erzeugt und können deshalb nicht auseinanderlaufen mit dem, was das Gerät wirklich kann.

Dieselbe Verb-Fläche gibt es zusätzlich über MQTT (ein Python-Plugin, heute nicht veröffentlicht). Der Unterschied ist der Transportweg, nicht der Funktionsumfang — die hier beschriebene Sitzung lief darüber, weil es den On-Device-Server damals noch nicht gab. Alles Folgende gilt unverändert für beide Wege.

Zwei Startpunkte, die dem Agenten das Raten ersparen:

- **`stick_describe`** als allererster Aufruf. Er liefert `api_version`, alle Verben, ihre Gefahrenklassen und `auth.token_set` — also LAB oder SECURED. Ein Agent, der stattdessen aus dem Gedächtnis Verbnamen bildet, erfindet welche.
- **Die Skill-Resource** `skill://busware-knx-stick/SKILL.md`. Dort steht das Runbook. Ein Agent, der sie liest, muss nicht instruiert werden.

Der Server beschreibt sich außerdem selbst mit dem, was er **nicht** weiß. Diese Sätze sind absichtlich drin: sie halten den Agenten davon ab, Produktsemantik zu halluzinieren.

## 2. Was der Agent wissen kann — und was Du ihm geben musst

**Auf dem KNX-Bus steht nicht, was etwas bedeutet.** Ein Gruppentelegramm sagt nicht, ob die zwei Oktette eine Temperatur oder eine Windgeschwindigkeit sind. Eine ComObject-Nummer sagt nicht, welcher Kanal das ist. Eine Parameter-Adresse steht nirgends im Gerät. Das alles liegt ausschließlich in der Produktdatei des Herstellers.

Deshalb ist die **`.knxprod` Pflicht-Eingabe**, und zwar Deine: der MCP-Server liefert sie nicht mit (bewusste Entscheidung — Hersteller-Produktdateien bleiben Anwender-Eingabe). Praktisch heißt das: die Datei liegt lokal, und der Agent liest sie. Sie ist ein ZIP mit unverschlüsseltem XML; ein Agent mit Dateizugriff kommt allein damit klar.

**Die wichtigste Anweisung dazu überhaupt:** *destillieren, nicht behaupten.* Verlange, dass jede Zahl, die der Agent aus der Produktdatei nimmt, mit ihrer Herkunft kommt — Element, Attribut, berechnete Absolutadresse. Ein Agent, der „die Fahrzeit liegt bei etwa 18986" sagt, hat geraten. Ein Agent, der sagt „`<Memory CodeSegment=AS-4A00 BitOffset=0 Offset=42/>` ⇒ 18944 + 42 = 18986, 16 Bit, Default 1200" hat gearbeitet.

In derselben Sitzung sind drei Auswertefallen aufgetreten, die man dem Agenten vorher sagen kann:

- `BitOffset` zählt **von MSB**.
- Die Feldbreite steht im `ParameterType` (`SizeInBit`), nicht beim Parameter. `PT-10ms/100ms` ist **3 Bit** breit — mit 2 Bit gerechnet liest man 10 ms statt 100 ms und liegt Faktor 10 daneben.
- Regex-Fallen: `Offset="…"` matcht auch **in** `BitOffset="…"` (also `\bOffset`), und die `<Memory>`-Kinder haben die Attributreihenfolge `CodeSegment, BitOffset, Offset` — wer `Offset` vor `BitOffset` erwartet, findet gar nichts und hält die Parameter für adresslos.

Alle drei sind mir in dieser Sitzung passiert. Alle drei kosten eine falsche Zahl, die *plausibel aussieht*.

## 3. Die Arbeitsteilung

| Rolle | Aufgabe | Häufigkeit |
|---|---|---|
| **Du** | Produktdatei bereitstellen, Ziel formulieren, scharfe Schritte freigeben | pro Gerät |
| **Agent** | Produktdatei destillieren, Adresse + Tabellen, HA-Übergabe, Parameter deklarieren | **einmal** pro Gerät |
| **HA-Nutzer** | Rollladen fahren, Fahrzeit als Zahl in ein Feld tippen | täglich |
| **Firmware** | Load-Session, Verifikation, Grenzen erzwingen | bei jedem Schreiben |

Das ist der Kern der ganzen Übung: **die Komplexität wandert in die Inbetriebnahme, nicht in den Alltag.** Der Agent hantiert mit Load State Machines und Task-Records; der Endnutzer tippt „45" in ein Feld namens *Fahrzeit*. Wenn Dein Aufbau den Endnutzer mit Segmentlisten konfrontiert, ist die Orchestrierung nicht fertig.

## 4. Der Ablauf, als Orchestrierung

Die Beispiele unten sind im Stil der echten Sitzung gehalten und stark gekürzt. Auffällig daran ist, wie **wenig** Fachsprache nötig ist: Du sagst das Ziel, der Agent besorgt sich den Rest. Wo im Beispiel Fachbegriffe fallen, kommen sie vom Agenten, nicht von Dir.

### 4.1 Erkunden — bevor irgendetwas geschrieben wird

```
Du     Am Bus hängt ein Merten-Jalousieaktor. Verschaff Dir erst mal
       einen Überblick, schreib noch nichts.

Agent  stick_describe → api 0.14.0, 20 Verben, LAB-Modus (kein Token),
       eigene Adresse 1.1.250, TPUART bereit, auto_ack aus.
       knx_addr_scan → niemand im Programmiermodus.
       knx_device_describe 1.1.5 → antwortet, Maske 0x0701 (BimM112).
       knx_tables_read → 5 Gruppenadressen gebunden: 11/0/0 → ComObj 0,
       11/0/1 → ComObj 2, 11/0/8 → 133, 11/0/9 → 135, 11/0/10 → 16.
       own_address_matches: true.
       Die Load States lese ich noch: LSM 1–4 alle "Loaded".

       Sicher: die Tabelleninhalte oben, direkt vom Gerät gelesen.
       Abgeleitet: nichts.
       Vermutung: was die ComObject-Nummern bedeuten — dafür brauche
       ich die Produktdatei, das steht nicht auf dem Bus.
```

Die letzten drei Zeilen sind der Punkt. Frag danach, wenn sie nicht kommen: *„Was weißt Du sicher, was hast Du abgeleitet, was ist Vermutung?"* Bleibt die dritte Kategorie leer, stimmt etwas nicht.

Lass den Agenten `stick_describe`, `knx_addr_scan` und `knx_device_describe` fahren und **das Gerät selbst** nach seinem Zustand fragen (`knx_tables_read`, `knx_property_read` auf die Load States). Erst danach über Änderungen sprechen.

Nützliche Frage an den Agenten in dieser Phase: *„Was weißt Du sicher, was hast Du abgeleitet, und was ist Vermutung?"* Wenn die dritte Kategorie leer bleibt, stimmt etwas nicht.

### 4.2 Produktdatei destillieren

Konkreter Auftrag statt offener Frage — und Belege verlangen:

```
Du     Die .knxprod liegt in ~/knx-bench/. Sag mir, was ComObj 0, 2 und 16
       auf Kanal 1 wirklich sind — und woher Du es hast.

Agent  Entpackt, M-000C/M-000C_A-5700-11-8E6A.xml, ComObject- und
       ComObjectRef-Texte:
         0  = 1 Bit,  Fahren (manuell)        Write, kein Read
         2  = 1 Byte, Höhe absolut (manuell)  Write
        16  = 1 Byte, "Status of height"      Read + Transmit
       Kanalbreite 19 Objekte, ich prüfe damit die anderen zwei:
         133 = 7·19+0 → Kanal 8, Fahren
         135 = 7·19+2 → Kanal 8, Höhe absolut

       ⚠️ Das ist ein 2×-Gerät — Kanal 8 existiert nicht. 11/0/8 und
       11/0/9 hängen an Objekten, die nie etwas senden werden. Ich würde
       dort keine DPT zuweisen; sonst stehen in HA Entities, die für
       immer leer bleiben.
```

Der Warnhinweis am Ende ist genau das, wofür man den Agenten hat. In der echten Sitzung hatte ich diesen beiden Adressen vorher versehentlich `9.001` (Temperatur) zugewiesen — geraten, nicht destilliert. Die Auswertung hat den Fehler aufgedeckt.

Bei Parametern dasselbe Muster, nur strenger — hier entscheidet eine Zahl über einen Speicherschreibzugriff:

```
Du     Wo liegt die Fahrzeit von Kanal 1?

Agent  <Parameter Name="Faktor Laufzeit Höhe (10-64000)" ... Value="1200">
         <Memory CodeSegment="…AS-4A00" BitOffset="0" Offset="42"/>
       AS-4A00 beginnt bei 0x4A00 = 18944, also 18944+42 = 18986,
       16 Bit big-endian, Default 1200.
       Zeitbasis: gleiches Segment, Offset 41 → 18985, BitOffset 5.
       ⚠️ Deren ParameterType PT-10ms/100ms ist SizeInBit="3", nicht 2 —
       mit 2 Bit gerechnet läse man 10 ms statt 100 ms.
       Gerät gelesen: 18985 = 0x0A, Bits 5–7 = 010 = 2 = 100 ms;
       18986/87 = 0x04B0 = 1200 ⇒ 1200 × 100 ms = 120 s.
```

Und dann die Gegenprobe **gegen die Wirklichkeit**, nicht gegen die Datei: eine volle Fahrt messen. Am Referenzgerät ergab das 128 s zu 120 s konfiguriert — konsistent mit einem Endlagen-Überhang, der bei 90 s konfiguriert genauso auftrat (95 s gemessen). *Das* ist Bestätigung; eine bloß plausible Zahl ist keine.

### 4.3 Adresse und Tabellen

Reihenfolge: `knx_addr_scan` → `knx_addr_assign` → `knx_device_describe` → `knx_tables_build` (**pur**, beliebig oft) → `knx_tables_read` (Vergleich) → je Tabelle `knx_load_begin` → `knx_memory_write` → `knx_load_finish`.

Zwei Gates, die der Agent nicht umgehen kann und die Du kennen solltest: `knx_addr_assign` prüft Belegung und *genau einer im Programmiermodus* selbst, und **jeder** `knx_memory_write` verifiziert per Rücklesen. Wenn ein Agent behauptet, geschrieben zu haben, ohne dass `verified: true` zurückkam, hat er nicht geschrieben.

So klingt die Freigabe des scharfen Schritts — kurz, aber an einer Stelle, an der Du weißt, was passiert:

```
Du     Bau mir die Tabellen so, dass 11/0/10 auf das Status-Objekt geht.
       Erst trocken.

Agent  knx_tables_build → Adresstabelle 13 Oktette, Assoziationstabelle
       11 Oktette. Diff gegen das Gerät: TSAP 5 → 11/0/10 → ComObj 16
       fehlt bisher, alles andere identisch.
       load_begin LSM 2 (dry_run) → würde durchlaufen.
       Soll ich scharf schreiben? Das ändert die Verdrahtung im Gerät.

Du     leg los

Agent  load_begin → Loading. mem_write 16896 → verified: true.
       load_finish → Loaded. Rücklesen: TSAP 5 → 11/0/10 → ComObj 16. ✓
```

### 4.4 Übergabe an Home Assistant

Die HA-Sicht ist eine Projektion dessen, was der Stick gelernt hat — ausgedrückt in Attributen (`dpt:` `role:` `name:` `pos:` `setpos:` `stop:`), dann `knx_ha_announce`.

**Die eine Regel, die Du dem Agenten nicht abkaufen musst, weil die Firmware sie erzwingt:** ohne ausdrückliche `role` ist ein Datenpunkt **read-only**. Eine Gruppenadresse sagt nicht, ob man sie schreiben darf — das weiß nur das Projekt. Schreibbar wird ein Datenpunkt nur auf Ansage, und diese Ansage ist eine Entscheidung, die Du treffen solltest, nicht der Agent im Vorbeigehen.

Danach: **prüfen lassen, was HA daraus gemacht hat**, nicht was der Stick publiziert hat.

Das ist der lehrreichste Dialog der ganzen Sitzung, weil er zeigt, wie ein Symptom-Bericht mehr wert ist als eine Diagnose:

```
Du     Das Cover lässt sich in HA nicht schließen, und der Slider
       bewegt sich nicht.

Agent  Ich schaue nach, statt zu raten.
       Retained Discovery-Config gelesen — sieht korrekt aus.
       HA-Entity: current_position 24, zuletzt aktualisiert vor 3 min.
       Also kommt die Rückmeldung an.
       Ich reproduziere Deinen Klick und höre am Bus mit:
       CLOSE → busware/knx/cmd/5800/cover/set → 11/0/0 = 01, kompakt. ✓
       Die Kommandokette ist es nicht.
       … 90 s später: der Aktor meldet 16.86 %.

       Im Mitschnitt sehe ich außerdem Deine Slider-Bewegung:
       5801/value/set mit 69, 51, 71, 80, 95, 63, 47, 38, 29, 17 —
       nicht monoton. Das ist ein Finger, der hin und her fährt, weil
       der Slider zurückspringt.

       Ursache: dieser Aktor meldet die Höhe erst am FAHRTENDE. Mit
       optimistic:false springt HAs Slider bei jeder Berührung auf den
       letzten bestätigten Wert — bis zu zwei Minuten lang.
       Und ich habe gestern einen Folgefehler eingebaut: die Invertierung
       über position_open/closed vergiftet HAs optimistic-Pfad.
       Ich lese das in HAs cover.py nach, statt es anzunehmen … bestätigt,
       Zeile 598. Beides fixe ich.
```

Beachte, was der Agent hier **nicht** tut: er glaubt Deiner Diagnose nicht ungeprüft („lässt sich nicht schließen" war faktisch falsch — CLOSE ging raus), und er behauptet die Ursache nicht, sondern liest sie im installierten HA-Quellcode nach.

### 4.5 Parameter editierbar machen

Der Teil, den kein KNX2MQTT-Gateway kann. Zwei Deklarationen, danach tippt der Nutzer Zahlen.

1. **`knx_loadproc_set`** — die `<LoadProcedure>` des Produkts, einmal je Gerät. Sie ist nötig, weil Parameterspeicher einen *geänderten* Write nur innerhalb einer Load-Session annimmt (siehe §6).
2. **`knx_param_declare`** — ein Parameter: Adresse, Breite, Rohgrenzen, Skalierung, Einheit. `per_unit` ist der Trick, der aus Rohwerten Sekunden macht: Faktor zählt in 100 ms ⇒ `per_unit: 10` ⇒ HA zeigt 45 s für Rohwert 450.
3. **`knx_ha_announce`** — der Parameter erscheint als Zahlenfeld in HAs Konfigurationsbereich, mit dem vom Gerät gelesenen Istwert.

**Warum das eine Allowlist ist:** der cmd-Plane ist durch die Broker-ACL eingezäunt, nicht durch den Admin-Token — richtig für Aktorik, falsch für Speicherschreibzugriffe, wo eine falsche Adresse kein Licht schaltet, sondern ein Gerät zerlegt. Deshalb kommt **nie eine Adresse über MQTT**: nur Deklariertes ist adressierbar, und nur innerhalb der deklarierten Grenzen. Der Agent deklariert bewusst und begründet; er improvisiert nicht zur Laufzeit.

⚠️ **Byte-Alignment ist die Stelle, an der ein Agent still Schaden anrichtet.** Die Ebene macht kein Read-Modify-Write: eine 8-Bit-Deklaration schreibt das **ganze Byte**. Mehrere Parameter teilen sich oft ein Byte. Verlange, dass der Agent vor jeder Deklaration in der Produktdatei prüft und *belegt*, dass der Parameter Alleinbewohner seines Bytes ist:

```
Du     Schalt die Referenzfahrt frei.

Agent  Erst prüfe ich, was sonst in dem Byte liegt — die Ebene schreibt
       ganze Bytes.
       Alle Parameter mit AS-4A00 Offset 11: genau einer,
       "Referenzfahrt allgemein", Bit 7, 1 Bit. Kein Nachbar.
       ⇒ ein Byte-Write auf 18955 ist hier verlustfrei.
       Begleitparameter "Referenzfahrt nach" liegt separat auf 19003,
       Bits 5–6, gelesen 0x38 → Feld = 00 = "Anzahl Bewegungen" (Default).
       Lasse ich unverändert.
       Setze 18955 = 01 … Gerät gegengelesen: 01. ✓
```

Und der Zahlungseingang der ganzen Übung — so sieht es aus, wenn der Endnutzer dran ist:

```
Du     Wie stelle ich die Fahrzeit auf 45 Sekunden?

Agent  Du tippst 45 in das Feld "Jalousie K1 Fahrzeit" in HA. Das war's.
       Dahinter: 45 s × 10 = Rohwert 450 = 0x01C2, und der Stick fährt
       die Load-Session selbst (begin → write → finish → restart).
       Zwei Nebenwirkungen, die Du wissen solltest: jeder Parameter-Write
       startet den Aktor neu, und dadurch verliert er die Positions-
       referenz bis zur nächsten vollen Fahrt.
       Vorbehalt: der Behang hier braucht real ~120 s. Mit 45 s stoppt
       der Aktor jede "volle" Fahrt nach ~48 s und glaubt trotzdem, in
       der Endlage zu sein. Für einen Test richtig, fürs Bench-Setup nicht.
```

### 4.6 Steuern, nicht nur ausführen lassen

Die zwei wirksamsten Eingriffe in dieser Sitzung waren keine Aufträge, sondern **Kurskorrekturen**. Beide bestehen aus einem Satz.

**Der eine:** eine Lösung zurückweisen, die technisch funktioniert, aber die falsche Person belastet.

```
Agent  Fahrzeit auf 45 s setzen:
         cc -std=gnu11 -I…/knx_mgmt/include -o /tmp/knxparmwrite …
         /tmp/knxparmwrite 18986 01C2
       Danach einmal CLOSE bis Endlage, dann OPEN.

Du     Das soll ein HA-User so machen? Wir brauchen eine Variante mit
       positiver UX!

Agent  Berechtigt — die CLI ist Bench-Werkzeug, kein Produkt. Richtig
       ist: der HA-User tippt 45 ins Feld. Also gehört die Session in
       die Firmware und die Komplexität in die EINMALIGE Inbetriebnahme.
       Ich baue: knx/loadproc/set (Deklaration je Gerät, aus dem
       Produktfile) + parm_command fährt die Session selbst.
       … 40 min später: HA-Feld → 45 → Gerät 0A 01C2 gegengelesen.
```

Dass der Agent zuerst die CLI-Lösung anbot, war kein Zufall, sondern typisch: er löst die Aufgabe, die dasteht, nicht die, die dahinter steht. **Diese Korrektur muss von Dir kommen** — sie erfordert ein Urteil darüber, wer das Werkzeug am Ende benutzt, und das steht in keinem Datenblatt.

**Der andere:** Autonomie befristet und mit einem Abbruchkriterium freigeben.

```
Du     Setz autonom fort, bis Ping-Pong läuft — ich bin kurz nicht
       an der Bench.

Agent  [12 Schritte ohne Rückfrage: cmd-Plane getestet, knxprod
       gegengelesen, Cover-Attribute gesetzt, announce, HA-Entities
       geprüft, Encoder-Fix gebaut, geflasht, on-air verifiziert]
       … HA set_cover_position 30 → cmd/5801/value/set 70 → Bus →
       Aktor fährt 100 % → 70,20 % → HA current_position 30. ✓
```

„Bis X läuft" ist ein gutes Kriterium, weil es prüfbar ist. „Mach mal weiter" ist keines. Und beachte, was in dem Block **nicht** stand: eine Freigabe, den Aktor zu bewegen, war implizit — bei einem Rollladen in einer bewohnten Wohnung solltest Du das explizit machen.

## 5. Leitregeln fürs Prompten

**Verlange Belege, nicht Ergebnisse.** „Woher weißt Du das?" ist die produktivste Frage im ganzen Ablauf. Bei Produktdaten: Element und Attribut. Bei Gerätezuständen: der Rohwert, den das Gerät geantwortet hat. Bei Messungen: der Zeitstempel und die Bedingung, die erfüllt war. Der Unterschied in der Praxis:

```
schwach   "Die Fahrzeit liegt bei 18986, Default sind 120 Sekunden."

belegt    "<Memory CodeSegment=AS-4A00 BitOffset=0 Offset=42/> ⇒
           18944+42 = 18986, 16 Bit, Value=1200. Gerät gelesen: 0x04B0
           = 1200. Zeitbasis 18985 Bits 5–7 = 010 = 100 ms ⇒ 120 s.
           Gemessene Vollfahrt: 128 s."
```

Die erste Antwort ist nicht falsch — sie ist bloß ungeprüft, und man kann ihr nicht ansehen, ob sie aus der Datei oder aus dem Modellgedächtnis kommt.

**Dry-run ist keine Höflichkeit, sondern der Default.** Alle mutierenden Verben starten trocken; `dry_run: false` ist eine Entscheidung. Ein Dry-run führt **jede Leseoperation** aus und hält nur den Write zurück — sein Bericht ist also wahrheitsfähig darüber, ob das Scharfe funktionieren würde.

**Eine Schreibprobe muss den Wert ändern.** Einen Wert zu schreiben, der schon dort steht, kann „angenommen" und „still verworfen" nicht unterscheiden — die Verifikation gelingt in beiden Fällen. Diese Falle hat in dieser Sitzung *zweimal* zugeschlagen und einen falschen Befund einen Tag überleben lassen.

**Zwei Pfade sind besser als einer.** Der Stick kann sich selbst bestätigen, und das ist wenig wert. Ein unabhängiger Zeuge ist viel wert: hier ein zweiter Adapter am selben Bus mit knxd dahinter, gegen den dieselben `knx_mgmt`-Quellen gebaut wurden. Wenn Firmware-Pfad und Host-Pfad dasselbe sagen, liegt es am Gerät; wenn nicht, an der Firmware. Diese Unterscheidung hat in dieser Sitzung mehrere Stunden gespart.

**Messen lassen, nicht schätzen — und die Messung prüfbar machen.** Dazu §6, letzter Punkt.

**Physische Wirkung ankündigen.** Ein Rollladen, der zwei Minuten fährt, ist eine sichtbare Handlung in jemandes Wohnung. Der Agent sollte sagen, *was* er auslösen will und *wie lange* es dauert, bevor er es tut — und den Zustand hinterher zurückstellen.

## 6. Wo LLM-Orchestrierung schiefgeht — Fallstudie

Alle vier Fehler sind in dieser Sitzung wirklich passiert, von einem Agenten, der die Regeln oben kannte. Sie sind die eigentliche Warnliste.

**Erfundene Ursache.** Ein schwacher Messwert sollte mit „die billigen Klon-Boards haben eine schwache Antenne" erklärt werden — frei erfunden, und von den eigenen Messdaten sogar widerlegt. Ein LLM füllt Erklärungslücken bevorzugt mit Plausiblem. **Gegenmittel:** Ursachen offen lassen dürfen. „Ich weiß nicht, warum" ist eine akzeptable Antwort und muss ausdrücklich als solche autorisiert sein, sonst wird geraten. So sieht die gute Variante aus:

```
Agent  Gemessen 128 s gegen 120 s konfiguriert. Die beiden Zuschlags-
       parameter erklären davon 200 ms. Die restlichen ~8 s kann ich
       nicht erklären und rate nicht. Kandidaten wären Nachlauf an der
       Endlage oder ein verzögertes Statustelegramm — beides müsste
       man prüfen.
```

**No-op-Probe als Beweis gelesen.** Ein Parameter-Write schien außerhalb einer Load-Session zu funktionieren (`verified: true`). Tatsächlich stand der Wert schon dort — die Probe konnte nicht scheitern. Der falsche Befund „Parameter brauchen keine Session" überlebte einen Tag und stand in den Projektnotizen, bis eine *ändernde* Probe ihn widerlegte:

```
Agent  Ein Byte geht, zwei nicht — also kein Rechteproblem, sondern
       die Länge.
       [zwei Proben später]
       Korrektur: ich nehme das zurück. Ich hatte in beiden Fällen den
       Wert geschrieben, der schon dort stand. So eine Probe kann
       "angenommen" und "still verworfen" gar nicht unterscheiden — die
       Verifikation gelingt zwangsläufig. Sie war als Test untauglich.
```

**Gegenmittel:** jede Probe daraufhin prüfen, ob sie einen negativen Ausgang überhaupt haben *kann*.

**Warte-Prüfung, die auf Altzustand matchte — zweimal.** Erst wurde ein Poll-Timeout (110 s + ein 5-s-Sleep) als „Fahrzeit 115 s gemessen" berichtet. Dann meldete `until grep -q '"value":0.00'` „Endlage erreicht", weil `mosquitto_sub` retained Nachrichten **sofort beim Abonnieren** ausliefert — die Fahrt hatte nicht stattgefunden.

```
schlecht  until grep -q '"value":0.00' log; do sleep 5; done
          echo "Endlage erreicht"          # matcht die retained Zeile

gut       mosquitto_sub -R …               # -R = keine retained
          S=$SECONDS
          until [ -s log ] || [ $((SECONDS-S)) -gt 170 ]; do sleep 10; done
          if [ -s log ]; then echo TREFFER; else echo KEINE MELDUNG; fi
```

**Gegenmittel:** `-R` benutzen, Timeout und Treffer unterscheidbar ausgeben, nie eine Zahl drucken, die bei Versagen plausibel aussieht, und im Zweifel das **Gerät** fragen statt den Log — das ist die einzige Quelle, die nichts erinnert.

**Regex, der zu viel matcht.** `Offset="(\d+)"` matcht auch in `BitOffset="…"`; das Ergebnis war eine falsche Parameter-Adresse, die sich echt anfühlte. Verräterisch war, dass Offset und BitOffset in *jeder* Zeile gleich waren. **Gegenmittel:** bei Auswertungen nach einem Muster suchen, das *auffallen* würde, wenn es falsch ist.

Ein Muster über alle vier: **die Fehler sahen richtig aus.** Kein Absturz, keine Fehlermeldung — plausible Zahlen und bestätigende Rückmeldungen. Deshalb ist der einzige zuverlässige Schutz die Frage nach dem Beleg, nicht die Frage nach dem Ergebnis.

## 7. Claude-Code-Spezifika

**Permissions greifen mitten im Ablauf.** In dieser Sitzung blockte der Auto-Mode-Classifier gerätenahe Schreibzugriffe — teils MCP-Verben, teils das Kompilieren eines Host-Tools —, obwohl identische Aufrufe zuvor durchgingen. Die richtige Reaktion des Agenten ist anhalten und sagen, was er wollte:

```
Agent  Der Permission-Classifier hat mir die Schreibzugriffe geblockt
       (gestern gingen identische Calls durch). Da höre ich auf — um so
       eine Sperre baue ich nicht herum.
       Das Werkzeug liegt fertig da: ein kleines Host-Tool, generisch
       addr+hex, das die Session auch im Fehlerfall sauber schließt.
       Entweder Du stößt es selbst an, oder Du gibst die Permission frei.

Du     Ich gebe Dir volle Permission — bitte versuch es nochmals!

Agent  [kompiliert, fährt die Session, 95 s gemessen, restauriert]
```

Praktische Konsequenz: **plane die Freigabe ein**, bevor Du den Agenten allein arbeiten lässt. Und wenn er stattdessen kreativ eine Umgehung baut, ist das ein Grund einzugreifen, kein Fortschritt.

**Lange physische Vorgänge gehören in Hintergrundaufgaben.** Eine Rollladenfahrt dauert zwei Minuten; ein blockierendes `sleep` im Vordergrund wird abgelehnt. Warteschleifen mit klarer Abbruchbedingung im Hintergrund laufen lassen und sich melden lassen — und die Bedingung so bauen, dass sie den Altzustand nicht trifft (§6).

**Hardware exklusiv allokieren.** Ein zweiter Öffner auf einem seriellen Port killt die laufende Messung des ersten. Vor jedem Port-Zugriff prüfen und belegen, nicht danach eintragen — mit welchem Werkzeug auch immer. Der Grund dahinter ist der eigentliche Punkt: **Öffnen ist ein Reset**, es gibt kein risikofreies Nachschauen.

**Zwei MCP-Server sind besser als einer.** Stick *und* HA. Der Agent, der nur seinen eigenen Stick befragen kann, bestätigt sich selbst.

**Notizen sind Teil der Orchestrierung.** Befunde, die eine Sitzung nicht überleben, werden in der nächsten neu erraten. Ebenso wichtig: **falsche Notizen aktiv korrigieren.** Der No-op-Fehlbefund oben stand in den Projektnotizen und hätte von dort weiter Schaden angerichtet.

## 8. Technischer Anhang

### Topics

```
busware/knx/cmd/<GAHEX>/value/set    Wert schreiben, kodiert per zugewiesenem DPT
busware/knx/cmd/<GAHEX>/cover/set    OPEN / CLOSE / STOP
busware/knx/cmd/<GAHEX>/read/set     GroupValueRead auslösen
busware/knx/cmd/<name>/parm/set      Geräteparameter, Wert in Anzeige-Einheit
busware/knx/state/<GAHEX>/value      retained, dekodiert
busware/knx/state/<name>/parm        retained, Parameter-Istwert
busware/knx/rx/11/0/10               unretained Rohsicht
busware/knx/avail/<stick_id>         retained online/offline
```

Die Form `busware/<proto>/cmd/<addr>/<kind>/set` ist familienweit, damit **eine** Broker-ACL (`busware/+/cmd/+/+/set`) die Aktorik aller Backends einzäunt. `<addr>` ist hexadezimal, weil `11/0/10` Topic-Trenner enthält — und weil zwei- und dreistufig geschrieben so auf einem Schlüssel landen.

### Attribute

`dpt:<GAHEX>` (Pflicht) · `role:<GAHEX>` = sensor|binary_sensor|switch|number|cover · `name:<GAHEX>` · `pos:` `setpos:` `stop:<GAHEX>` (Cover) · `parm:<name>` = `<ia>,<addr>,<breite>,<min>,<max>,<pro_einheit>[,<einheit>]`

### Belegte Geräte-Fallen (Merten 649802)

- **Parameter-Writes brauchen die Load-Session.** Außerhalb verwirft das Gerät jeden *geänderten* Write stumm — autorisiert oder nicht, ein Byte oder zwei, vor und nach einem Neustart. Innerhalb verifiziert derselbe Write und übersteht den Restart. Bewiesen: Faktor 900 geschrieben → Vollfahrt 95 s gemessen (vorher 128 s bei 1200).
- **Jeder Parameter-Write startet das Gerät neu** und kostet die Positionsreferenz. Absolutpositionen sind dann wirkungslos, bis eine volle Fahrt in eine Endlage sie setzt. Gegenmittel: Parameter „Referenzfahrt allgemein" (18955) freigeben.
- **Kompaktform gehört der Datenpunkt-BREITE, nicht dem Zahlenwert.** DPT ≤ 6 Bit reist im APCI-Oktett, ab DPT 5 immer im eigenen. Ein Encoder, der „passt in 6 Bit" prüft, sendet 20 % (Rohwert 51) kompakt — falsch, und nur unter 64 sichtbar.
- **GA binden ≠ Funktion aktivieren.** Ein Statusobjekt ist doppelt deaktiviert: Parameter-Byte *und* Communication-Bit im GroupObjectTable-Deskriptor.
- **ComObject-Nummern sind kanalweise, und die App deckt mehr Kanäle ab als die Hardware hat.** Ein Kanal ist hier 19 Objekte breit; ComObj 133/135 gehören zu Kanal 8, den ein 2×-Gerät nicht hat. Kanal 1: 0 = Fahren (1 Bit), 1 = Stopp, 2 = **Höhe absolut (1 Byte)**, 16 = Status Höhe.
- **Segment AS-5100 existiert auf der 2×-Variante nicht** (Speicher endet 0x5100); ein Alloc darauf hängt die Sequenz. Die Load-Prozedur führt es trotzdem auf — sie gilt für die ganze Gerätefamilie.
- **Anlernen der Fahrzeit gibt es in dieser App nicht** (Suche über alle 2207 Parameter: null Treffer). Die Referenzfahrt referenziert Position, nicht Zeit.

### Belegte HA-Fallen

- **`set_position_template` ERSETZT die Skala, es ergänzt sie nicht.** HA reicht dem Template den bereits umgerechneten Wert als Eingabe, während die Variable `position` die rohe Prozentzahl bleibt. Und `position_open`/`position_closed` zu setzen vergiftet den optimistic-Pfad. Lösung: Skala auf Default lassen, in zwei symmetrischen Templates invertieren.
- **`optimistic: true` ist bei diesem Aktor Pflicht.** Er meldet die Höhe erst am Fahrtende, es gibt keine Meldung während der Fahrt. Ohne optimistic springt HAs Slider bei jeder Berührung auf den letzten bestätigten Wert zurück — bis zu zwei Minuten. HA markiert die Entity dann als `assumed_state`, die Vorhersage ist also gekennzeichnet, und das state-Topic trägt weiterhin nur Gemessenes.
- **Das `avail`-Topic braucht ein aktives „online"** — das LWT publiziert nur „offline", sonst stehen alle Entities ewig auf *unavailable*.

### Bekannte Schönheitsfehler

- Die Gruppen-Ebene publiziert **L2-Wiederholungen 4×** (Auto-ACK aus ⇒ der Partner sendet jede Antwort viermal; der Sequenzfilter sitzt nur im Management-Pfad). Für HA idempotent.
- `describe.dpt_assignments` zählt **alle** attr-Keys, nicht nur DPT-Zuweisungen — bei 3 DPT plus Rollen, Namen und Parametern meldet es 14. Nutze `knx_dpt_list` / `knx_param_list`.
- **Ungeklärt:** warum das Referenzgerät Parameter-Writes außerhalb der Session verwirft, während ETS partielle Downloads über `ApplicationRunControl` (Objekt 3, PID 6) fährt. Die Event-Codes dafür liegen hier in keiner belegbaren Quelle; die volle Load-Session ist der einzige bewiesene Weg.

### Merten-Referenz zum Abtippen

```
Verdrahtung (1.1.5):  11/0/0 → ComObj 0 · 11/0/1 → ComObj 2 · 11/0/10 → ComObj 16

dpt:5800  = 1.008    role:5800 = cover   name:5800   = Jalousie Merten K1
dpt:5801  = 5.001    role:5801 = number  pos:5800    = 580A
dpt:580A  = 5.001                        setpos:5800 = 5801

parm:travel1 = 1.1.5,18986,16,10,64000,10,s
parm:refrun  = 1.1.5,18955,8,0,1,1
```

Load-Prozedur — AS-5100 fehlt absichtlich:

```json
{"device": "1.1.5", "lsm": 3,
 "segments": [
   {"address": 1792,  "size": 384,  "access": 241, "mem_type": 2, "seg_flags": 0},
   {"address": 17408, "size": 1373, "access": 241, "mem_type": 3, "seg_flags": 128},
   {"address": 18944, "size": 1536, "access": 241, "mem_type": 3, "seg_flags": 128},
   {"address": 20480, "size": 256,  "access": 241, "mem_type": 3, "seg_flags": 0}],
 "task_segment": {"address": 18422},
 "task_ctrl1":   {"address": 18408, "count": 1}}
```

Parameter Kanal 1 (Segment `AS-4A00` ab 18944, **Kanalabstand 102 Byte**):

| Parameter | Adresse | Feld | am Gerät |
|---|---|---|---|
| Zeitbasis Laufzeit Höhe | 18985 | Bit 5–7, 3 Bit | `2` = 100 ms |
| Faktor Laufzeit Höhe (10–64000) | 18986–18987 | 16 Bit, big-endian | `1200` = 120 s |
| Faktor Laufzeitzuschlag aufwärts | 18988 | 8 Bit | `20` |
| Status Höhe | 19073 | Byte | `05` |
| Referenzfahrt allgemein | 18955 | Bit 7, 1 Bit | `01` = frei |
| Referenzfahrt nach | 19003 | Bit 5–6, 2 Bit | `0` = Anzahl Bewegungen |

Fahrzeit = Zeitbasis × Faktor; neuer Faktor = Sekunden × 1000 / Zeitbasis_ms, bei 100 ms also **Sekunden × 10**.

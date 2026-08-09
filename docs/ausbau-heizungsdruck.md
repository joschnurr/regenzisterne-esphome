# Ausbau (geplant): Heizungsdruck am Zisternen-ESP mitmessen

Erweiterung des bestehenden Zisternen-Knotens (`regenzisterne`) um einen **Drucksensor für den
Heizungs-Wasserkreislauf**. Weil Heizung und Zisternen-ESP **im selben Raum** stehen (Kabelweg
**max. 2 m**), wird der Sensor **mit an den vorhandenen NodeMCU** gehängt — kein zweiter Knoten
nötig. Status: **geplant, noch nicht umgesetzt.**

## Entscheidung in einem Satz
Digitaler **I2C-Drucksensor XIDIBEI XDB401** (G1/4", 0–5 bar relativ) an die **freien** Pins
D6/D7 des Zisternen-ESP, per nativem ESPHome-Treiber `xdb401` ausgelesen, in bar kalibriert gegen
das Kessel-Manometer.

## Warum genau dieser Sensor
- **I2C statt Analog:** umgeht den einzigen, verrauschten 10-bit-A0 des ESP8266 **und** das
  5-V-Problem (ESP8266 ist nicht 5-V-tolerant) komplett. Kein ADS1115, kein Spannungsteiler.
- **Nativer ESPHome-Support** (`platform: xdb401`) — liefert **Druck und Temperatur**.
- **G1/4"-Gewinde**, echter Flüssigkeitsanschluss, Edelstahl/Keramikzelle.
- **Bereich:** 0–5 bar relativ (Überdruck) ist der Sweetspot: Systemdruck 1–2 bar, Sicherheitsventil
  3 bar → genug Reserve, gute Auflösung. **Wichtig: „relativ"/Überdruck, nicht „absolut".**

## Anschluss am ESP (Verdrahtung)
Der Ultraschall nutzt **GPIO4 (D2) = TRIG** und **GPIO5 (D1) = ECHO** — das sind **Digital**-Pins.
I2C ist am ESP8266 in Software nachgebildet und darf **beliebige freie** Pins nutzen; nur **nicht**
D1/D2. Gewählt: **SDA = D6 (GPIO12), SCL = D7 (GPIO13)** (frei, keine Boot-Strapping-Pins).

| XDB401 | NodeMCU | Bemerkung |
|---|---|---|
| V+ | **3V3** | am 3,3-V-Pin betreiben (nicht 5 V nötig) |
| GND | **GND** | gemeinsame Masse |
| SDA | **D6 (GPIO12)** | **nicht** D1/D2 (Ultraschall!) |
| SCL | **D7 (GPIO13)** | " |

Der Ultraschall-Block bleibt **unverändert**; der XDB401 kommt nur zusätzlich dazu.

## I2C über 2 m — damit es stabil bleibt
2 m sind die obere Grenze für I2C, mit diesen Vorkehrungen unkritisch:
- **Kabel:** verdrillt/geschirmt — ideal ein Stück **CAT5/Netzwerkkabel**: ein Paar SDA+GND, ein
  Paar SCL+GND. GND immer mitführen.
- **Takt niedrig:** **100 kHz** (`frequency: 100kHz`), bei Bedarf 50 kHz. Kein 400 kHz.
- **Pull-ups:** auf 2 m etwas kräftiger (~**3,3–4,7 kΩ** nach 3V3). Hat die XDB401-Platine nur
  schwache 10 kΩ, je einen ~3,3-kΩ-Widerstand SDA→3V3 und SCL→3V3 **am ESP-Ende** ergänzen.
- **Verlegung:** nicht eng an der **Heizungspumpe**/Netzleitungen führen (Störeinstreuung).
- Falls es wider Erwarten zickt (fehlende Werte/NACKs): erst **50 kHz**, dann stärkere Pull-ups,
  zur Not I2C-Bus-Extender (bei 2 m praktisch nie nötig).

## ESPHome-Konfiguration (Ansatz)
Kommt **zusätzlich** in die bestehende `regenzisterne.yaml`. Der Ultraschall-Teil bleibt, wie er ist.

> **Vor dem Einspielen prüfen:** genaues Schema und I2C-Adresse des `xdb401`-Components gegen die
> aktuelle ESPHome-Doku abgleichen; `i2c: scan: true` zeigt die real erkannte Adresse im Log
> (oft `0x7F`). Die Kalibrier-Stützpunkte unten sind **Platzhalter** und werden real gegen das
> Manometer gemessen.

```yaml
i2c:
  sda: GPIO12          # D6 — frei; NICHT D1/D2 (dort haengt der Ultraschall)
  scl: GPIO13          # D7
  frequency: 100kHz    # auf 2 m Kabel bewusst langsam
  scan: true           # beim ersten Start die erkannte Adresse im Log kontrollieren

sensor:
  - platform: xdb401
    # Adresse/Details gegen die aktuelle ESPHome-xdb401-Doku pruefen (i2c-scan zeigt die reale Adresse)
    pressure:
      name: "Heizungsdruck"
      id: heizungsdruck
      unit_of_measurement: bar
      device_class: pressure
      state_class: measurement
      accuracy_decimals: 2
      filters:
        # Zweipunkt-Kalibrierung gegen das Kessel-Manometer:
        # linke Werte = ROH-Anzeige des Sensors bei zwei bekannten Manometer-Druecken (real ablesen!)
        - calibrate_linear:
            method: least_squares
            datapoints:
              - 0.0 -> 0.0     # Anlage drucklos -> 0 bar
              - 1.8 -> 1.8     # bei Betriebsdruck: Rohwert -> Manometer-bar
        - sliding_window_moving_average:
            window_size: 6
            send_every: 1
    temperature:
      name: "Heizung Sensortemperatur"
      unit_of_measurement: "°C"
      device_class: temperature
      state_class: measurement
      accuracy_decimals: 1
    update_interval: 30s
```

Einspielen per **OTA** (Dashboard → `regenzisterne` → Install → „Wirelessly").

## Home Assistant
- Neue Entitäten erscheinen per MQTT-Discovery **unter dem bestehenden Gerät „Regenzisterne"**:
  `sensor.regenzisterne_heizungsdruck` und `sensor.regenzisterne_heizung_sensortemperatur`.
  (Rein kosmetisch, dass der Heizungsdruck unter „Regenzisterne" hängt — funktioniert einwandfrei,
  lässt sich in HA beschriften. Nur wer strikte Trennung will, nähme einen eigenen Knoten.)
- **Sinnvolle Automatisierung:** Meldung, wenn `Heizungsdruck < 1,0 bar` (Nachfüllen fällig) oder
  `> 2,5 bar` (Richtung Sicherheitsventil) — analog zu den anderen HA-Benachrichtigungen.

## Mechanischer Einbau & Sicherheit
- **Abgriff:** am **KFE-Hahn** (Kessel-Füll-/Entleerhahn) oder per **T-Stück am Manometer** —
  vorhandenen Mess-/Füllpunkt nutzen, nicht neu in die Leitung schneiden.
- **Adapter:** Sensor ist G1/4" außen; per Reduziernippel/T-Stück an den vorhandenen Anschluss.
- **Thermischer Schutz:** Vorlauf bis ~90 °C, günstige Sensoren nur ~85 °C → ein
  **Wassersackrohr/Siphon** (kurzes gebogenes Standrohr) davor schützt die Membran ohne
  Genauigkeitsverlust (Wasser ist inkompressibel). Alternativ kühlere Stelle (Rücklauf/KFE) oder
  ein 150-°C-Sensor.
- **Abdichten:** G1/4" ist **zylindrisch** → dichtet an der **Planfläche** (Flach-/Profildichtring,
  O-Ring), **kein** Hanf/PTFE auf dem Gewinde.
- **Sicherheit:** reines Lesen, Kleinspannung, keine Netzspannung → ungefährlich. **Anlage vor dem
  Einbau drucklos machen und abkühlen** lassen; **Sicherheitsventil unangetastet** lassen (nie in
  dessen Leitung einbauen/absperren). Bei Unsicherheit den Stutzen vom Heizungsbauer setzen lassen.

## Teileliste (ca.)
| Teil | ca. |
|---|---|
| XIDIBEI XDB401 (I2C, G1/4, 0–5 bar relativ) | 12–20 € |
| CAT5-Kabelstück (2 m) | wenige € |
| ggf. 2× Pull-up 3,3 kΩ | <1 € |
| Wassersackrohr/Manometer-Siphon | 8–20 € |
| T-Stück / Reduziernippel G1/4 + Dichtring | wenige € |
| **Summe** | **~25–45 €** |

## Umsetzungsschritte (Reihenfolge)
1. XDB401 (I2C-Variante, 0–5 bar relativ) besorgen; Temperatur-Spezifikation prüfen (sonst Siphon einplanen).
2. Am ESP verdrahten (3V3, GND, SDA=D6, SCL=D7), CAT5, ggf. Pull-ups.
3. `i2c:`-Block mit `scan: true` ergänzen, per OTA aufspielen, im **Log die I2C-Adresse** kontrollieren.
4. `xdb401`-Sensor ergänzen (Schema/Adresse gegen ESPHome-Doku), OTA.
5. **Kalibrieren:** zwei Punkte gegen das Manometer eintragen (drucklos = 0 bar, Betriebsdruck = X bar), OTA.
6. Sensor **physisch** an der Heizung einbauen (drucklos/kalt, Siphon, KFE/T-Stück, dichten), befüllen, entlüften, Dichtheit prüfen.
7. HA: Entitäten beschriften, optional Alarm < 1,0 bar / > 2,5 bar.

## Offene Punkte / vor dem Kauf prüfen
- Genaues `xdb401`-Schema und I2C-Adresse in der aktuellen ESPHome-Doku (`i2c: scan`).
- XDB401-Variante: Bereich **5 bar** (Auflösung) vs. 10 bar (mehr Reserve); Medien-Temperatur der konkreten Variante.
- Ob der Heizungsdruck bewusst unter dem Gerät „Regenzisterne" bleiben soll oder ein eigener Knoten gewünscht ist.

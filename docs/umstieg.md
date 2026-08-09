# Umstieg von opencistern auf ESPHome

Reihenfolge ist wichtig: Home Assistant hätte sonst kurz zwei Sensoren mit derselben ID (die alte
YAML-Variante blockiert die ID, die Discovery-Variante bekäme ein `_2`-Suffix). Deshalb **erst die
alten Sensoren entfernen, dann flashen**.

## Schritt 1 — alte manuelle Sensoren in HA entfernen

In `ha-config/mqtt_sensor.yaml` die drei Blöcke **„Regenzisterne Liter/Abstand/Füllstand"** löschen
(die anderen MQTT-Sensoren bleiben). Danach in HA **MQTT neu laden** (Entwicklerwerkzeuge → YAML →
„MQTT-Einträge neu laden") oder HA neu starten. Kurz danach zeigen die drei Zisternen-Sensoren
„unavailable" — das ist erwartet, bis der neue ESP läuft.

*(Auf Zuruf erledigt Claude diesen Schritt in Sekunden.)*

## Schritt 2 — ESP per USB flashen

Der Wechsel von der Fremdfirmware geht **nur per USB** (OTA über opencistern ist nicht möglich).
Danach nie wieder Kabel.

1. ESP an der Zisterne abklemmen, per **Micro-USB-Datenkabel** an einen Windows-PC.
2. Falls kein COM-Port erscheint: USB-Seriell-Treiber installieren (meist **CH340** oder CP2102).
3. **Chrome** oder **Edge** öffnen (Web-Serial; Firefox kann es nicht).
4. **Weg A (empfohlen):** Dashboard `http://192.168.178.11:6052` → Karte **regenzisterne** →
   **Install → „Plug into this computer"** → COM-Port wählen → flashen.
5. **Weg B (Fallback):** Dashboard → **Install → „Manual download"** → `.factory.bin` laden →
   auf **https://web.esphome.io** → Connect → Datei wählen → Install.

Für das **erste** USB-Flashen immer das **factory**-Image nehmen (enthält den Bootloader). Wo die
Datei auf der Gbox liegt, steht in [`betrieb.md`](betrieb.md#firmware-datei).

## Schritt 3 — prüfen

1. Der ESP verbindet sich (`regenzisterne.local`) und sendet über MQTT-Discovery.
2. In HA erscheint automatisch ein **Gerät „Regenzisterne"** mit den Sensoren Liter/Füllstand/
   Abstand plus WLAN-Signal, Betriebszeit und IP.
3. Die Entity-IDs sind wieder `sensor.regenzisterne_liter/_abstand/_fullstand` —
   `group.sensor_zisterne` funktioniert unverändert.
4. Das Reconnect-Flappen im Mosquitto-Log ist weg.

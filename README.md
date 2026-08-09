# Regenzisterne-Füllstand über ESPHome

ESPHome-Firmware für den ESP8266 an der Regenwasser-Zisterne. Ein **HC-SR04-Ultraschallsensor**
misst den Abstand zur Wasseroberfläche; daraus werden Füllstand (%) und Liter berechnet und per
**MQTT mit Home-Assistant-Autodiscovery** veröffentlicht. Ersetzt die frühere
[opencistern](https://github.com/diefenbecker/opencistern)-Firmware.

> 📖 **Ausführliche Anleitungen im [Wiki](https://github.com/joschnurr/regenzisterne-esphome/wiki):**
> [Flashen](https://github.com/joschnurr/regenzisterne-esphome/wiki/Flashen) ·
> [Benötigte Werkzeuge](https://github.com/joschnurr/regenzisterne-esphome/wiki/Werkzeuge) ·
> [Von Grund auf neu aufbauen](https://github.com/joschnurr/regenzisterne-esphome/wiki/Neuaufbau) ·
> [Kalibrierung & Betrieb](https://github.com/joschnurr/regenzisterne-esphome/wiki/Betrieb) ·
> [Fehlersuche](https://github.com/joschnurr/regenzisterne-esphome/wiki/Fehlersuche)

## Warum der Umbau

| | opencistern (bisher) | ESPHome (neu) |
|---|---|---|
| MQTT-Verbindung | ~30 Reconnects/h („exceeded timeout") | stabil |
| HA-Anbindung | manuelle Sensoren in `mqtt_sensor.yaml` | Autodiscovery (Gerät + Entitäten) |
| Zugangsdaten | WLAN- **und** MQTT-Passwort im Klartext auf `http://<ESP>/cfg` | keine offene Konfigseite |
| Literanzeige | in 32-L-Stufen (Füllstand auf ganze % gerundet) | fein aufgelöst |
| Diagnose | – | WLAN-Signal, Betriebszeit, IP-Adresse |
| Updates | nur per USB | drahtlos (OTA) nach dem ersten Flashen |
| Konfiguration | im Gerät, nicht versioniert | `regenzisterne.yaml` im Git |

Die Entity-IDs bleiben **exakt gleich** (`sensor.regenzisterne_liter`, `_abstand`, `_fullstand`),
weil der ESPHome-Node `regenzisterne` heißt und die Sensoren `Liter`/`Abstand`/`Fullstand`. Damit
bleibt auch `group.sensor_zisterne` in Home Assistant unverändert.

## Repo-Struktur

| Pfad | Inhalt |
|---|---|
| `regenzisterne.yaml` | die vollständige ESPHome-Konfiguration (Rechenwerte, Pins, Sensoren) |
| `secrets.yaml` | WLAN-/AP-Zugang — **nicht** im Git (siehe `.gitignore`), aus `secrets.yaml.example` erzeugen |
| `secrets.yaml.example` | Vorlage mit Platzhaltern |
| `docs/umstieg.md` | Schritt-für-Schritt: alte HA-Sensoren entfernen → per USB flashen → prüfen |
| `docs/betrieb.md` | Betrieb, Wartung, Kalibrierung, Firmware-Datei, Rückweg |
| `docs/ausbau-heizungsdruck.md` | **Geplanter Ausbau:** Heizungsdruck per I2C-Sensor XDB401 mit an den Zisternen-ESP |

Der zugehörige Docker-Stack (ESPHome-Dashboard) liegt in der Gbox-Infrastruktur unter
`smarthome/stacks/esphome/` und mountet dieses Repo (`/home/johannes/regenzisterne-esphome`)
als sein `/config`-Verzeichnis. Das Dashboard ist erreichbar unter
**http://192.168.178.11:6052**.

## Quickstart

Der Wechsel von der Fremdfirmware geht **einmalig nur per USB** (danach nie wieder Kabel — Updates
laufen über OTA). Die Reihenfolge ist wichtig, damit HA keine doppelten Entitäten mit `_2`-Suffix
anlegt:

1. **Erst** die drei alten manuellen Sensoren aus `ha-config/mqtt_sensor.yaml` entfernen und MQTT
   neu laden.
2. **Dann** den ESP per USB mit dieser Firmware flashen.
3. Prüfen, dass in HA das Gerät „Regenzisterne" mit den gewohnten Entity-IDs erscheint.

Die ausführliche Anleitung mit allen Wegen (Dashboard bzw. `web.esphome.io`) steht in
**[`docs/umstieg.md`](docs/umstieg.md)**.

## Erststart nach dem Klonen

```bash
cp secrets.yaml.example secrets.yaml    # dann WLAN-Zugang eintragen
chmod 600 secrets.yaml
```

Bauen/prüfen über das Dashboard oder von Hand — siehe **[`docs/betrieb.md`](docs/betrieb.md)**.

## Sicherheitshinweis

opencistern zeigte auf `http://<ESP>/cfg` WLAN- und MQTT-Passwort im Klartext an jedes Gerät im
WLAN. Solange der alte ESP läuft, besteht das Leck fort — ein Grund, den Wechsel nicht zu lange
liegenzulassen. Das ESPHome-Image (`*.bin`) enthält ebenfalls das WLAN-Passwort und ist daher per
`.gitignore` vom Repo ausgeschlossen; nicht weitergeben.

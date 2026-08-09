# Betrieb, Wartung & Kalibrierung

## Updates (drahtlos)

Nach dem ersten USB-Flashen laufen alle weiteren Updates über OTA:
`regenzisterne.yaml` im Dashboard (`http://192.168.178.11:6052`) bearbeiten →
**Install → „Wirelessly"**.

## Werte ändern / kalibrieren

Die Rechengrundlage steht in `regenzisterne.yaml` unter `substitutions`:

| Wert | Bedeutung |
|---|---|
| `abst_boden` | Abstand Sensor → Tankboden (cm), also bei **leerem** Tank |
| `abst_max` | Abstand Sensor → maximaler Wasserstand (cm), also bei **vollem** Tank |
| `volumen` | Nutzvolumen der Zisterne (Liter) |

Die Rechnung ist linear:

```
Füllstand % = (Abstand − abst_boden) / (abst_max − abst_boden) × 100   (auf 0…100 begrenzt)
Liter       = Füllstand % / 100 × volumen
```

(aus opencistern übernommen, `zisterne/rechnen.ino` — aber ohne dessen grobe Rundung auf ganze %.)

## Hardware

- Board: **NodeMCU v2** (`esp8266: board: nodemcuv2`). Bei einem Wemos D1 mini stattdessen
  `d1_mini` (gleicher Chip).
- Sensor: **HC-SR04-Ultraschall**, `trigger_pin: GPIO4` (D2), `echo_pin: GPIO5` (D1) — dieselben
  Pins wie in der opencistern-Firmware.
- Messung alle 60 s, Median über 5 Werte gegen Ausreißer.

## Firmware-Datei

Kompiliert auf der Gbox im ESPHome-Build-Verzeichnis:

```
/home/johannes/regenzisterne-esphome/.esphome/build/regenzisterne/.pioenvs/regenzisterne/
    firmware.factory.bin   # erstes USB-Flashen (mit Bootloader)
    firmware.ota.bin       # drahtlose Updates
    firmware.bin           # reines Programm-Image
```

Am einfachsten über „Manual download" im Dashboard laden. Die `.bin` enthält das WLAN-Passwort:
nicht weitergeben, nicht ins Git (per `.gitignore` ausgeschlossen).

## Von Hand bauen (Gbox)

```bash
docker exec esphome esphome config  /config/regenzisterne.yaml   # nur Syntax prüfen
docker exec esphome esphome compile /config/regenzisterne.yaml   # kompilieren
```

Der erste Build lädt die Toolchain und dauert auf der 2-Kern-Gbox ~10–15 min. Läuft der Build
länger als die aufrufende Shell, mit `docker exec -d` (detached) starten, damit er nicht per SIGHUP
abbricht.

## Rückweg

Die alte `opencistern_1030.bin` liegt im Original-Repo
<https://github.com/diefenbecker/opencistern> und ließe sich per USB wieder aufspielen.

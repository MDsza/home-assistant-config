# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ KRITISCHER WORKFLOW - Source of Truth

**WICHTIG: Home Assistant Live-System (192.168.111.3) ist die EINZIGE Source of Truth!**

**Wolfgang arbeitet direkt in Home Assistant UI. Lokale Festplatte kann VERALTET sein!**

**Bei JEDER Arbeit an Config-Dateien:**

1. **PULL** vom Live-System → lokal (IMMER zuerst!)
   ```bash
   ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo cat /config/automations.yaml" > config/automations.yaml
   ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo cat /config/scenes.yaml" > config/scenes.yaml
   ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo cat /config/configuration.yaml" > config/configuration.yaml
   ```

2. **Lokal ändern** (Read, Edit, Write Tools)

3. **PUSH** zurück zum System (IMMER nach lokaler Änderung!)
   ```bash
   cat config/automations.yaml | ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo tee /config/automations.yaml > /dev/null"
   cat config/scenes.yaml | ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo tee /config/scenes.yaml > /dev/null"
   cat config/configuration.yaml | ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo tee /config/configuration.yaml > /dev/null"
   ```

4. **YAML Reload** (User muss manuell in HA UI: Developer Tools → YAML → Reload)

**NIEMALS lokal ändern ohne vorher zu pullen!**

## Overview

Home Assistant Smart Home Backup & Config Repository nach erfolgreicher Zigbee2MQTT 2.6.3 Migration.

**System:**
- Home Assistant OS 16.3 (VM auf TrueNAS Scale)
- IP: 192.168.111.3
- HA Core: 2025.11.1
- Zigbee2MQTT: 2.6.3 (migriert von 1.42.0)
- MQTT Broker: Mosquitto
- Zigbee Coordinator: TI CC2652 (zStack3x0)

**Hardware:**
- 10 Zigbee-Geräte (7x Philips Hue Dimmer V1/V2, Hue Tap Switch, Hue Tap Dial, Aqara Cube)
- 30+ Automationen
- 154 Szenen
- Standard Motion Sensor Blueprint: "Sensor Light" by Blacky (für alle Motion-basierten Automationen)

## 🎯 Standard Blueprint - Motion Sensor Automation

**WICHTIG: Dieser Blueprint ist die STANDARD-LÖSUNG für alle Motion-Sensor-basierten Automationen!**

**Blueprint:** "Sensor Light" by Blacky
**Import URL:** `https://gist.github.com/freakshock88/2311759ba64f929f6affad4c0a67110b`
**Community Thread:** https://community.home-assistant.io/t/turn-on-light-switch-scene-or-script-based-on-motion-and-illuminance-more-conditions/257085

### Features:
- ✅ Motion Sensor Trigger (einzeln oder Group)
- ✅ Auto-OFF Timer (X Minuten ohne Bewegung)
- ✅ Illuminance Conditions (Lux-basiert, nur bei Dunkelheit)
- ✅ Time-based Limits (Zeitfenster before/after)
- ✅ Sun Elevation (below/above horizon)
- ✅ Blocking Entities (Sleep Mode, Away Mode)
- ✅ Turn-off Blocking (verhindert Auto-OFF wenn gewünscht)
- ✅ Scene/Light/Switch/Script Support
- ✅ Kompatibel mit Zigbee2MQTT binary_sensor

### Required Helper Entities:

**WICHTIG:** Blueprint nutzt `input_number` Helper für dynamische Werte!

**Standard-Helper (bereits erstellt):**

1. **Auto-OFF Timer:**
   ```yaml
   Name: Treppe Auto-OFF Timer
   Entity: input_number.treppe_auto_off_timer
   Min: 0.1, Max: 5, Step: 0.1
   Unit: min
   Default: 0.5 (= 30 Sekunden)
   ```

2. **Lux Schwellwert:**
   ```yaml
   Name: Treppe Lux Schwellwert
   Entity: input_number.treppe_lux_schwellwert
   Min: 0, Max: 200, Step: 5
   Unit: lux
   Default: 50
   ```

### Blueprint Konfiguration - Minimal:

```yaml
Motion Sensor: binary_sensor.xxx_occupancy
Target Entity: light.xxx oder scene.xxx
Turn off wait time: input_number.xxx_auto_off_timer  # PFLICHT für Auto-OFF!
```

### Blueprint Konfiguration - Vollständig:

```yaml
# Required
Motion Sensor: binary_sensor.treppe_hue_motion_sensor_oben_occupancy
Target Entity: light.treppe_nightlight_hue_white_e14

# Optional - Auto-OFF
Turn off wait time: input_number.treppe_auto_off_timer

# Optional - Nur bei Dunkelheit
Illuminance sensor: sensor.treppe_hue_motion_sensor_oben_illuminance_lux
Illuminance cutoff: input_number.treppe_lux_schwellwert
Sun elevation: Below horizon

# Optional - Zeitfenster
Only run after: input_datetime.xxx_start
Only run before: input_datetime.xxx_end

# Optional - Blocking
Blocking entity: input_boolean.sleep_mode
Turn-off Blocking: input_boolean.guest_mode
```

### Wichtige Hinweise:

1. **Scene als Target:**
   - MUSS zusätzlich `target_off_entity` definieren!
   - target_off_entity = Light/Group die ausgeschaltet werden soll

2. **Auto-OFF funktioniert nur mit input_number Helper:**
   - NICHT als fester Wert im Blueprint!
   - Helper erlaubt dynamische Änderung ohne Automation-Edit

3. **Occupancy Timeout der Hue Sensoren:**
   - Standard: 15-60 Sekunden (je nach Sensor)
   - Blueprint-Timer startet NACH Sensor 'off'
   - Gesamt-Verzögerung = Sensor Timeout + Blueprint Timer

4. **Für mehrere Sensoren (OBEN + UNTEN):**
   - ENTWEDER: Separate Automationen pro Sensor
   - ODER: Binary Sensor Group als Motion Sensor

### Häufige Probleme:

**Licht geht nicht aus:**
- ❌ Kein `Turn off wait time` Helper definiert → Blueprint schaltet sofort aus bei Sensor 'off'
- ✅ `input_number` Helper erstellen + im Blueprint zuweisen

**Licht geht zu schnell aus:**
- Sensor Occupancy Timeout zu kurz (15s) + kein Blueprint Timer
- ✅ Blueprint Timer auf 0.5 min (30s) erhöhen

**Automation triggert nicht:**
- ❌ Falsche Entity ID (z.B. `_belegung` statt `_occupancy`)
- ✅ Developer Tools → States → korrekte Entity ID checken

## Repository-Struktur

```
/config/                    # Aktuelle HA Config (nicht versioniert)
├── automations.yaml        # ~40KB, 30+ Automationen
├── scenes.yaml             # ~157KB, 154 Szenen
├── configuration.yaml      # Haupt-Config mit InfluxDB Integration

/ha-config-backup/          # Versioniertes Backup
├── config/
│   ├── automations.yaml
│   ├── scenes.yaml
│   └── configuration.yaml
├── blueprints/
│   ├── z2m_hue_dimmer/     # V1 Hue Dimmer Blueprint (6 Geräte)
│   └── z2m_hue_dimmer_v2/  # V2 Hue Dimmer Blueprint (1 Gerät)
└── zigbee2mqtt/
    └── configuration.yaml  # MQTT-Passwort maskiert!

/config.backup_YYYYMMDD_HHMMSS/  # Zeitgestempelte Backups
```

## Common Commands

### SSH-Zugriff zum Live-System

```bash
# Connect (requires SSH key setup)
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222

# Root access
sudo -i

# View logs
ha addons logs 45df7312_zigbee2mqtt
tail -f /config/home-assistant.log
```

### Backup Commands (auf Live-System via SSH)

```bash
# Supervisor Backup (empfohlen vor großen Änderungen)
ha supervisor backups new --name "pre-change-$(date +%Y%m%d-%H%M)"

# Manuelle Config Backups
BACKUP_DIR="/root/z2m-backup-$(date +%Y%m%d-%H%M)"
mkdir -p $BACKUP_DIR

# Copy critical files
sudo cp /homeassistant/zigbee2mqtt/configuration.yaml $BACKUP_DIR/
sudo cp /homeassistant/zigbee2mqtt/database.db $BACKUP_DIR/
sudo cp /homeassistant/zigbee2mqtt/coordinator_backup.json $BACKUP_DIR/
sudo cp /config/automations.yaml $BACKUP_DIR/
sudo cp /config/scenes.yaml $BACKUP_DIR/
sudo cp /config/configuration.yaml $BACKUP_DIR/
```

### YAML Reload & Validation (auf Live-System)

```bash
# Config validieren BEVOR Reload
ha core check --config /config

# Via ha CLI (blockiert bis Start erfolgreich)
ha core restart --wait

# Nur YAML Configs neu laden (ohne Neustart)
# → HA UI → Developer Tools → YAML → Reload All

# Via MQTT (falls CLI nicht verfügbar)
mosquitto_pub -h 192.168.111.3 -u mqtt_user -P 'PASSWORD' \
  -t 'homeassistant/reload' -m ''
```

### Git Workflow (in diesem Repo)

```bash
# Status prüfen
git status

# Änderungen committen (config/ ist in .gitignore)
git add ha-config-backup/
git commit -m "Update: Beschreibung"

# WICHTIG: NIEMALS automatisch pushen! Nur auf Anfrage.
```

## Wichtige Pfade

### Live-System (192.168.111.3)

```
/homeassistant/zigbee2mqtt/
├── configuration.yaml       # Z2M Config mit MQTT Credentials
├── database.db              # Device-Pairing State
├── coordinator_backup.json  # Coordinator Backup (13KB)
├── state.json               # Runtime State
└── log/                     # Z2M Logs

/config/
├── automations.yaml         # HA Automationen
├── scenes.yaml              # HA Szenen
├── configuration.yaml       # HA Haupt-Config
└── blueprints/automation/   # Custom Blueprints
```

### Lokales Repo

```
/Users/wolfgang/MyCloud/TOOLs/HomeAssistant/
```

## Architektur & Kritische Konzepte

### Zigbee2MQTT 2.x Migration (November 2025)

**KRITISCH:** Z2M 2.x entfernt `sensor.*_action` Entities komplett.

**Alt (1.42.0):**
```yaml
trigger:
  - platform: state
    entity_id: sensor.dimmer_action
    to: "on_press_release"
```

**Neu (2.6.3):**
```yaml
trigger:
  - platform: mqtt
    topic: "zigbee2mqtt/Dimmer Name"
    payload: "on_press_release"
    value_template: "{{ value_json.action }}"
```

**Blueprint-Architektur:**
- MQTT-basierte Trigger (kein Entity State)
- `base_topic` Input (default: `zigbee2mqtt`)
- `controller` Input = Zigbee2MQTT Device Name (NICHT HA Entity Name!)
- Events direkt aus MQTT Payload `value_json.action`

### Hue Dimmer V1 vs V2 Hardware

**V1 Dimmer (6 Geräte):**
- Events: `button_1_press_release`, `button_1_hold`
- Blueprint: `z2m_hue_dimmer/hue_dimmer_z2m.yaml`

**V2 Dimmer (1 Gerät - "Noah NEU"):**
- Events: `button_1_press`, `button_1_hold` (OHNE `_release`)
- Blueprint: `z2m_hue_dimmer_v2/hue_dimmer_v2.yaml`

**Unterscheidung:** V2 hat andere Event-Namen in MQTT Payload!

### Light Groups + brightness_step Problem

**KRITISCH - NIEMALS brightness_step auf Light Groups!**

**Problem:**
```yaml
# FALSCH:
action:
  - service: light.turn_on
    target:
      entity_id: light.gruppe_mit_5_lampen
    data:
      brightness_step: -26  # Exponentiell kleiner werdende Schritte!
```

**Ursache:**
- HA wendet `-26` auf JEDE Lampe einzeln an
- Lampen mit niedrigem Brightness erreichen 0 schneller
- Verbleibende Lampen = scheinbar kleinere Schritte
- Nach mehreren Presses: Nur noch 1-2 Lampen dimmen

**Lösung - Device Actions:**
```yaml
# RICHTIG:
action:
  - device_id: abc123...
    domain: light
    entity_id: light.lampe_1
    type: brightness_decrease  # HA managed Steps!
  - device_id: def456...
    domain: light
    entity_id: light.lampe_2
    type: brightness_decrease
```

**Warum funktioniert das?**
- `type: brightness_decrease` = HA Device Action
- HA verwaltet Schrittgröße konsistent pro Lampe
- Unabhängig vom aktuellen Brightness-Level

### InfluxDB + Grafana Integration

**Config:** `configuration.yaml`
- InfluxDB 2 API auf `192.168.111.81:8086`
- Organisation: `HomeLab`
- Bucket: `homeassistant`
- Exports: sensor, binary_sensor, light, cover domains
- Excludes: persistent_notification, person

**Token:** In live Config (nicht maskiert im Backup)

## Sensitive Daten

**WICHTIG - Im Repo maskiert:**
- MQTT Passwort in `ha-config-backup/zigbee2mqtt/configuration.yaml`
- InfluxDB Token in `ha-config-backup/config/configuration.yaml`
- SSH Private Keys (niemals committen)

**Originale Werte:** 1Password / lokales Backup auf Live-System

**Best Practice:** Credentials auf Live-System in `secrets.yaml` auslagern, nur masked Placeholders committen

## Migration Historie & Learnings

### Versuch 1 (Januar 2025) - FEHLGESCHLAGEN

**Fehler:**
- Legacy-Mode aktiviert (`legacy_triggers: true`)
- Z2M 2.x akzeptiert Settings, aber KEINE Wirkung
- Action Sensors werden NICHT erstellt

**Resultat:** Rollback erforderlich

### Versuch 2 (November 2025) - ERFOLGREICH

**Timeline:**
- 18:00 Backup erstellt
- 18:01 Z2M Update 1.42.0 → 2.6.3
- 18:06 Blueprints installiert
- 18:08 Erste Automation migriert (Test erfolgreich)
- 18:15 Alle 7 Dimmer funktional
- 19:45 Alle 10 Geräte migriert

**Strategie:**
1. Z2M Update OHNE Legacy-Mode
2. MQTT-basierte Blueprints direkt ins Filesystem schreiben
3. Automationen via Python-Script migrieren
4. YAML Reload

**Key Learning:** Legacy-Mode ist KEINE Lösung! Direkte Blueprint-Migration erforderlich.

## Häufige Aufgaben

### Blueprint zu Live-System hinzufügen

```bash
# Via SSH:
sudo mkdir -p /config/blueprints/automation/neue_blueprint/
sudo nano /config/blueprints/automation/neue_blueprint/blueprint.yaml

# HA lädt automatisch nach YAML-Reload
```

### Automation aus Backup wiederherstellen

```bash
# Backup lokal prüfen
cat ha-config-backup/config/automations.yaml

# Auf Live-System kopieren (VORSICHT: überschreibt!)
scp -P 22222 -i ~/.ssh/claude_z2m_key \
  ha-config-backup/config/automations.yaml \
  hassiossh@192.168.111.3:/config/automations.yaml

# YAML neu laden (via UI)
```

### Z2M Config ändern

```bash
# WICHTIG: Immer Backup VORHER!
sudo cp /homeassistant/zigbee2mqtt/configuration.yaml /root/backup.yaml

# Edit (via SSH oder lokal via PULL → Edit → PUSH Workflow)
sudo nano /homeassistant/zigbee2mqtt/configuration.yaml

# Validate & Restart Z2M
ha addons restart a0d7b954_zigbee2mqtt

# Check logs (Z2M startet nicht bei invalider Config!)
ha addons logs a0d7b954_zigbee2mqtt | tail -n 50
```

### OTA Updates für Battery-Devices (Hue Tap Dial)

**Problem:** Battery Devices schlafen nach ~10 Sekunden

**Strategie:**
1. Device direkt am Coordinator platzieren (Link Quality 60+)
2. HA UI → Device → "Aktualisieren" klicken
3. **SOFORT** intensiv Buttons drücken / Dial drehen (10x schnell!)
4. Device bleibt ~30 Sekunden wach
5. OTA Transfer startet (läuft dann autonom ~20 Minuten)

**Nicht verwenden:**
- ❌ MQTT Command (`/set/update`) - oft nicht unterstützt
- ❌ Z2M Frontend (Port 8099) - nur Container-intern

## Troubleshooting

### Automation funktioniert nach Z2M Update nicht

**Prüfen:**
1. Nutzt Automation `sensor.*_action` Entity? → Migration erforderlich
2. Blueprint MQTT-basiert? → `base_topic` und `controller` korrekt?
3. Device Name in Z2M = Controller Input?

**Fix:**
- Automation auf MQTT Blueprint migrieren
- ODER: MQTT Device Trigger direkt verwenden

### Dimmer Steps werden kleiner (Brightness-Problem)

**Check:** Ist Entity eine Light Group?

**Fix:**
- Device Actions verwenden (`type: brightness_increase/decrease`)
- NICHT `brightness_step` auf Gruppen!

### Z2M startet nicht nach Config-Änderung

**Common Errors:**
```yaml
# FALSCH:
ota:  # Leere Zeile = invalid

# RICHTIG:
ota: {}
```

**Fix:** Config validieren, Backup wiederherstellen

### SSH Connection refused

**Prüfen:**
- Advanced SSH Add-on "Running"?
- Port 22222 in Add-on Network Config?
- Public Key korrekt in authorized_keys?

## Ressourcen

**Dokumentation (in diesem Repo):**
- `README.md` - Übersicht & Migration Timeline
- `ZIGBEE2MQTT_MIGRATION_ANALYSIS.md` - Detaillierte Analyse & Learnings
- `SSH_REMOTE_ACCESS.md` - SSH Setup Guide

**Externe Links:**
- Z2M Docs: https://www.zigbee2mqtt.io/
- Z2M Breaking Changes: https://github.com/Koenkk/zigbee2mqtt/blob/master/CHANGELOG.md
- HA Blueprint Exchange: https://community.home-assistant.io/c/blueprints-exchange

**Support:**
- Z2M Discord: https://discord.gg/dadfWMz3
- HA Forum: https://community.home-assistant.io/c/configuration/zigbee2mqtt
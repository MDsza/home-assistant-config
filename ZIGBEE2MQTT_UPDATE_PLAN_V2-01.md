# Zigbee2MQTT Update 1.42.0 → 2.6.3-1 - Vollständiger Migrations-Plan

**Erstellt:** 2025-11-09
**System:** Home Assistant OS (VM auf TrueNAS Scale)
**Add-on:** Zigbee2MQTT 1.42.0-2 → 2.6.3-1
**Coordinator:** Texas Instruments CC2652 (zStack3x0)

---

## PHASE A: Discovery & Analyse (Read-Only)

### A1. SSH-Zugang einrichten

1. **Advanced SSH & Web Terminal Add-on konfigurieren:**
   - Configuration → Network: Port `22222` eintragen
   - Configuration → Authorized keys: Claude's SSH Public Key eintragen

2. **Router/Firewall:**
   - Port-Forwarding: `22222` → HA-VM Port `22`

3. **Test-Verbindung:**
   ```bash
   ssh root@<ha-ip> -p 22222
   ```

### A2. System-Analyse (Claude führt aus)

```bash
# Pfade verifizieren
ha addons info 45df7312_zigbee2mqtt
ls -la /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/

# Aktuelle Config lesen
cat /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/configuration.yaml

# Automationen analysieren
cat /config/automations.yaml

# Entities checken
ha states list | grep -E '(zigbee2mqtt|action|click|child_lock|illuminance_lux)'

# MQTT Broker Status
ha addons info core_mosquitto

# Gruppen in Config prüfen
grep -A 10 "^groups:" /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/configuration.yaml
```

### A3. Automationen-Inventur

Liste erstellen für:
- **`sensor.*_action` / `sensor.*_click`** → Werden in v2.x entfernt, müssen auf MQTT Device Triggers umgestellt werden
- **`illuminance_lux`** → Property-Rename zu `illuminance`
- **`lock.*child_lock`** → Entity-Typ wechselt zu `switch.*child_lock`
- **Gruppen** in `configuration.yaml` → Werden entfernt, müssen neu angelegt werden
- **Manuelle OTA Topics** → Neue Topic-Struktur

### A4. Risiko-Bewertung & Go/No-Go

Basierend auf Findings:
- Anzahl betroffener Automationen
- Kritikalität der Gruppen
- Custom Integrationen
- → Update-Strategie anpassen/bestätigen

---

## PHASE B: Update-Durchführung

### B1. Backups erstellen

```bash
# Timestamp-Backup-Directory
BACKUP_DIR="/root/z2m-backup-$(date +%Y%m%d-%H%M)"
mkdir -p $BACKUP_DIR

# Kritische Z2M Files
cp /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/configuration.yaml $BACKUP_DIR/
cp /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/database.db $BACKUP_DIR/
cp /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/coordinator_backup.json $BACKUP_DIR/

# Automations sichern
cp /config/automations.yaml $BACKUP_DIR/

# Permissions sicherstellen
chmod -R 600 $BACKUP_DIR/*

# Backup verifizieren
ls -lh $BACKUP_DIR/
```

**Zusätzlich:**
- HA Snapshot erstellen via UI: `Einstellungen → System → Backups → Backup erstellen`

### B2. Update durchführen

1. **UI Navigation:**
   - Supervisor → Zigbee2MQTT → Tab "Informationen"

2. **Option aktivieren:**
   - ✓ "Backup der letzten Version behalten"

3. **Update starten:**
   - Button "Aktualisieren" klicken

4. **Logs live monitoren:**
   ```bash
   ha addons logs 45df7312_zigbee2mqtt --follow
   ```

   Erwartete Log-Ausgaben:
   - Add-on Stop
   - Image Pull: `2.6.3-1`
   - Config Migration läuft
   - Add-on Start
   - `Zigbee2MQTT started`

### B3. Migration-Log prüfen

```bash
cat /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/migration-1-to-2.log
```

**Checken:**
- ✓ Entfernte Keys:
  - `advanced.legacy_api`
  - `permit_join_timeout`
  - `groups` Definitionen
  - `advanced.report`
  - `advanced.soft_reset_timeout`

- ✓ Verschobene Keys:
  - `advanced.homeassistant_discovery_topic` → `homeassistant.discovery_topic`
  - `advanced.baudrate` → `serial.baudrate`
  - `advanced.rtscts` → `serial.rtscts`

- ✓ Umbenannte Keys:
  - `whitelist` → `passlist`
  - `ban` → `blocklist`

- ⚠️ Warnungen/Fehler dokumentieren

### B4. Validierung

#### MQTT Bridge Status

```bash
# Falls mosquitto_pub/sub auf HA verfügbar:
mosquitto_sub -h localhost -t 'zigbee2mqtt/bridge/info' -C 1

# Alternativ: MQTT Explorer von externem Rechner
# Topic: zigbee2mqtt/bridge/info
```

**Erwartete Ausgabe:**
```json
{
  "version": "2.6.3",
  "coordinator": {
    "type": "zStack3x0",
    "meta": {...}
  },
  "network": {...},
  "log_level": "info",
  "permit_join": false
}
```

#### HA Entities Check

**Developer Tools → States:**

```bash
# Via CLI:
ha states list | grep -E 'zigbee2mqtt'
```

**Prüfen:**
- ✓ `sensor.xxx_update_state` entfernt (erwartet)
- ✓ `sensor.xxx_update_available` entfernt (erwartet)
- ✓ `lock.xxx_child_lock` → jetzt `switch.xxx_child_lock`
- ✓ Attribute mit `illuminance_lux` → jetzt `illuminance`

#### Status Topic (Auto-Migriert)

- Z2M lauscht jetzt auf `homeassistant/status` (statt `hass/status`)
- HA sendet automatisch korrekt
- **Keine manuelle Aktion nötig**

### B5. Automationen anpassen

#### Pattern 1: Action/Click Entities (entfernt)

**ALT (funktioniert nicht mehr):**
```yaml
trigger:
  - platform: state
    entity_id: sensor.schalter_action
    to: 'single'
```

**NEU (MQTT Device Trigger):**
```yaml
trigger:
  - platform: device
    domain: mqtt
    device_id: <device_id_aus_phase_a>
    type: action
    subtype: single  # oder: double, long, etc.
```

**Device ID ermitteln:**
```bash
# Developer Tools → States → Device suchen
# Oder via CLI:
ha device list | grep -A 5 "schalter"
```

#### Pattern 2: Property-Renames

**Templates anpassen:**
```yaml
# ALT:
{{ state_attr('sensor.bewegungsmelder', 'illuminance_lux') }}

# NEU:
{{ state_attr('sensor.bewegungsmelder', 'illuminance') }}
```

**Entity-Namen:**
```yaml
# ALT:
sensor.bewegungsmelder_illuminance_lux

# NEU:
sensor.bewegungsmelder_illuminance
```

#### Pattern 3: Child-Lock Entity-Typ

**Prüfen:**
```yaml
# ALT (lock Entity):
service: lock.lock
target:
  entity_id: lock.thermostat_child_lock

# NEU (switch Entity):
service: switch.turn_on
target:
  entity_id: switch.thermostat_child_lock
```

### B6. Gruppen neu anlegen

Falls Gruppen aus `configuration.yaml` entfernt wurden:

#### Option A: Z2M Frontend
1. Z2M Web-UI öffnen
2. Gruppen → "+ Gruppe hinzufügen"
3. Geräte zuweisen

#### Option B: MQTT
```bash
# Gruppe erstellen
mosquitto_pub -t 'zigbee2mqtt/bridge/request/group/add' \
  -m '{"friendly_name": "Wohnzimmer Lichter"}'

# Mitglied hinzufügen
mosquitto_pub -t 'zigbee2mqtt/bridge/request/group/members/add' \
  -m '{"group": "Wohnzimmer Lichter", "device": "lampe_1"}'
```

---

## PHASE C: Post-Update Tests

### C1. Funktionstests

**Checkliste:**
- [ ] Alle Zigbee-Devices in Z2M sichtbar
- [ ] Devices erreichbar (Test: Licht schalten)
- [ ] Automationen ausgelöst (Test: Bewegungsmelder)
- [ ] Gruppen steuerbar
- [ ] MQTT Bridge online (`zigbee2mqtt/bridge/state` = "online")
- [ ] HA Discovery funktioniert (neue Devices werden erkannt)
- [ ] OTA Updates verfügbar (prüfen via Z2M UI)

**Test-Commands:**
```bash
# Bridge State
mosquitto_sub -t 'zigbee2mqtt/bridge/state' -C 1

# Device verfügbar?
mosquitto_sub -t 'zigbee2mqtt/lampe_1' -C 1

# Gruppensteuerung
mosquitto_pub -t 'zigbee2mqtt/Wohnzimmer Lichter/set' -m '{"state": "ON"}'
```

### C2. 24h-Monitoring

**Logs prüfen:**
```bash
# Errors/Warnings
ha addons logs 45df7312_zigbee2mqtt | grep -E '(error|warn)'

# Performance
ha addons stats 45df7312_zigbee2mqtt
```

**HA Logbuch:**
- Einstellungen → System → Protokolle
- Filter: `zigbee2mqtt`

**Bereitschaft:**
- Rollback-Prozedur dokumentiert
- Backups verfügbar halten (7 Tage)

---

## Rollback-Strategie

### Bei kritischen Problemen innerhalb 24h

```bash
# 1. Add-on stoppen
ha addons stop 45df7312_zigbee2mqtt

# 2. Alte Version reinstallieren
ha addons install --version 1.42.0-2 45df7312_zigbee2mqtt

# 3. Backups zurückspielen
BACKUP_DIR="/root/z2m-backup-YYYYMMDD-HHMM"  # Anpassen!

cp $BACKUP_DIR/configuration.yaml \
   /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/

cp $BACKUP_DIR/database.db \
   /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/

cp $BACKUP_DIR/coordinator_backup.json \
   /mnt/data/supervisor/addons/data/45df7312_zigbee2mqtt/

# 4. Automations restore
cp $BACKUP_DIR/automations.yaml /config/

# 5. Add-on starten
ha addons start 45df7312_zigbee2mqtt

# 6. HA Core neu laden (für Automationen)
ha core restart

# 7. Validierung
ha addons logs 45df7312_zigbee2mqtt --follow
mosquitto_sub -t 'zigbee2mqtt/bridge/state' -C 1
```

### Rollback via HA Snapshot

Falls Add-on-Rollback nicht funktioniert:
1. Einstellungen → System → Backups
2. Snapshot von vor Update auswählen
3. "Wiederherstellen" → Teilwiederherstellung
4. Add-ons: Nur Zigbee2MQTT ✓
5. Bestätigen

---

## Breaking Changes - Detaillierte Übersicht

### 1. Kritisch (Aktion ERFORDERLICH)

#### Home Assistant Discovery Cleanup
- **Status Topic:** `hass/status` → `homeassistant/status`
- **Entfernte Entities:**
  - `sensor.*_action`
  - `sensor.*_click`
  - `sensor.*_update_state`
  - `sensor.*_update_available`
- **Child Lock:** `lock.*` → `switch.*` Entity-Typ
- **Legacy Triggers:** Entfernt → MQTT Device Triggers nutzen
- **Legacy Entity Attributes:** Nicht mehr verfügbar

**Aktion:**
- Automationen auf MQTT Device Triggers umstellen
- Child-Lock Service-Calls anpassen
- Templates ohne Legacy-Attributes

#### Gruppen-Management
- **Entfernt:** `permit_join_timeout`
- **Verhalten:** HA Permit-Join schaltet nach max. 254s automatisch ab
- **Gruppen in YAML:** Nicht mehr unterstützt

**Aktion:**
- Gruppen via Frontend/MQTT neu anlegen
- Automationen für Permit-Join-Timeout entfernen

#### MQTT Error Responses
- **Änderung:** Bei `status: "error"` ist `data` leer
- **Fehlertext:** Jetzt in `error` Field

**Aktion:**
- Custom MQTT-Scripts anpassen
- Fehlerbehandlung prüfen

### 2. Moderat (Prüfen empfohlen)

#### Property-Renames (60+ Geräte betroffen)
- `illuminance_lux` → `illuminance`
- `internal_temperature` → `internalTemperature` (Typo-Fix)
- Diverse `child_lock` Cleanups

**Aktion:**
- Templates/Automationen mit alten Property-Namen aktualisieren

#### OTA Update Rework
- **Neue Topics:**
  - `zigbee2mqtt/bridge/request/device/ota_update/check`
  - `zigbee2mqtt/bridge/request/device/ota_update/update`
  - `zigbee2mqtt/bridge/request/device/ota_update/schedule`
- **Features:** Downgrade-Support, neue Limits

**Aktion:**
- OTA-Automationen auf neue Topics umstellen
- Neue Config-Optionen prüfen (`image_block_response_delay`, `default_maximum_data_size`)

#### Legacy API Removal
- **Entfernt:** Komplette MQTT Legacy API (seit 1.17.0 deprecated)
- **Betroffen:** Custom Integrationen die Legacy-Format nutzen

**Aktion:**
- Custom MQTT-Integrationen auf neue API migrieren

### 3. Info (Auto-Migriert)

#### Automatische Config-Migrationen
```yaml
# Automatisch migriert durch Z2M:
advanced.homeassistant_discovery_topic → homeassistant.discovery_topic
advanced.baudrate → serial.baudrate
advanced.rtscts → serial.rtscts
whitelist → passlist
ban → blocklist
log_level: warn → log_level: warning
```

**Aktion:** Keine - wird automatisch umgeschrieben

#### Adapter-Konfiguration
- **Änderung:** `zstack` nicht mehr Default
- **Erforderlich:** Explizite Angabe in Config

**Aktion:** Migration setzt automatisch `serial.adapter: zstack` für TI-Coordinator

---

## Zeitplan & Ressourcen

| Phase | Dauer | Beschreibung |
|-------|-------|--------------|
| **A1** | 10 min | SSH-Setup + Test |
| **A2** | 15 min | System-Discovery & Config-Analyse |
| **A3** | 15 min | Automationen-Inventur |
| **A4** | 5 min | Risiko-Bewertung & Go/No-Go |
| **B1** | 5 min | Backups erstellen |
| **B2** | 10 min | Update durchführen |
| **B3** | 5 min | Migration-Log prüfen |
| **B4** | 10 min | Validierung (MQTT/Entities) |
| **B5** | 20-60 min | Automationen anpassen |
| **B6** | 10 min | Gruppen neu anlegen |
| **C1** | 15 min | Funktionstests |
| **C2** | 24h | Monitoring |
| **TOTAL** | **2-3h + 24h** | Aktive Arbeit + Beobachtung |

---

## Pre-Flight Checkliste

### Vor Start (Phase A)
- [ ] SSH-Zugang getestet (`ssh root@<ip> -p 22222`)
- [ ] Port 22222 in Router/Firewall freigegeben
- [ ] Claude's SSH-Key in Advanced SSH Add-on hinterlegt
- [ ] Wartungsfenster geblockt (3 Stunden)
- [ ] MQTT-Client verfügbar (mosquitto/MQTT Explorer)

### Nach Phase A (Discovery)
- [ ] Automationen-Liste erstellt
- [ ] Gruppen identifiziert
- [ ] Risiken bewertet
- [ ] **Go-Entscheidung getroffen**

### Nach Phase B (Update)
- [ ] Backups erstellt (Files + Snapshot)
- [ ] Update erfolgreich abgeschlossen
- [ ] Migration-Log geprüft (keine kritischen Errors)
- [ ] MQTT Bridge online
- [ ] HA Entities validiert

### Nach Phase C (Tests)
- [ ] Alle Funktionstests bestanden
- [ ] Keine Errors in Logs (24h)
- [ ] Automationen funktionieren
- [ ] Gruppen steuerbar
- [ ] **SSH-Zugang widerrufen** (Port schließen + Key entfernen)

---

## POST-UPDATE: Blueprint Migration Strategie

**STATUS:** ⚠️ **KRITISCH** - 8 Automationen fallen nach Update aus bis Blueprints migriert sind!

### Betroffene Automationen

| Device | Blueprint Status | Migration | Zeitaufwand |
|--------|------------------|-----------|-------------|
| **6x Hue Dimmer** | ⚠️ INKOMPATIBEL | Neue Blueprint | 60 min |
| **1x Aqara Cube** | ✅ KOMPATIBEL | Vermutlich keine Aktion | 5 min Test |
| **1x Hue Tap Dial** | ⚠️ UPDATE | Neue Blueprint v2.0 | 15 min |
| **1x Hue Tap Switch** | ⚠️ INKOMPATIBEL | Manuelle Migration | 20 min |

**Gesamt-Downtime:** ~2 Stunden (Familie kann Lichter NICHT schalten!)

---

### 1. Aqara Cube (PRIORITY: HIGH - sollte funktionieren)

**Test nach Update:**
```yaml
# Automation-ID prüfen: '1651870296527'
# Blueprint: SirGoodenough/Zigbee2MQTT - Xiaomi Cube Controller
```

**Aktion:**
1. Cube-Automation testen (drehen, kippen, etc.)
2. Falls funktioniert: ✅ DONE
3. Falls nicht: Blueprint updaten auf T1-Pro Version

**Blueprint-URL (falls Update nötig):**
```
https://github.com/SirGoodenough/HA_Blueprints/blob/master/Automations/Zigbee2MQTT-Aqara-Magic-Cube-T1-Pro-CTP-R01-Xiaomi-Lumi.yaml
```

---

### 2. Hue Tap Dial Switch (PRIORITY: MEDIUM)

**Betroffene Automation:**
- ID: (zu identifizieren)
- Entity: `sensor.bad_hue_tap_dial_switch_black_action`

**Migration:**
1. Alte Blueprint sichern
2. Neue Blueprint v2.0 importieren:
   ```
   https://gist.github.com/freakshock88/672fc91e6981da4ca0c49e71b0c05032
   ```
3. Automation auf neue Blueprint umstellen
4. Testen: Button press + Dial rotation

**Zeitaufwand:** 15 Minuten

---

### 3. Hue Tap Switch (PRIORITY: HIGH - Dach Licht)

**Betroffene Automation:**
- ID: '1651944919324'
- Entity: `sensor.dach_switch_hue_tab_action`
- Funktion: 4-Button Scene-Control (Dach)

**Migration (Manuelle Device Trigger):**

```yaml
# NEU: Device Trigger statt Sensor
trigger:
  # Button 1
  - platform: device
    domain: mqtt
    device_id: <device_id>  # Via Developer Tools → Devices ermitteln
    type: action
    subtype: button_1_single

  # Button 2-4 analog
```

**Schritte:**
1. Device ID ermitteln: Developer Tools → States → "Dach - SWITCH - HUE TAB" suchen
2. Alte Automation deaktivieren
3. Neue Automation mit Device Triggers erstellen
4. Actions aus alter Automation kopieren
5. Testen alle 4 Buttons

**Zeitaufwand:** 20 Minuten

---

### 4. Hue Dimmer Switches (PRIORITY: CRITICAL - 6 Stück!)

**Betroffene Automationen:**
1. Sabine - Hue Dimmer (ID: '1651952303530')
2. Noah - Hue Dimmer (ID: '1651957558052')
3. Wohnzimmer - Hue Dimmer (ID: zu identifizieren)
4. Küche - Hue Dimmer (ID: zu identifizieren)
5. Wolfgang - Hue Dimmer (ID: zu identifizieren)
6. Noah - Hue Dimmer 2 (ID: zu identifizieren)

**Aktuelle Blueprint:** EPMatt/philips_324131092621.yaml (INKOMPATIBEL mit Z2M 2.x)

**Neue Blueprint:** "Very Easy Custom Philips Hue Dimmer Switch for Z2M"
```
https://community.home-assistant.io/t/very-easy-custom-philips-hue-dimmer-switch-for-z2m-zigbee2mqtt/521205
```

**Migration pro Dimmer (~10 min):**

1. **Blueprint importieren** (einmalig):
   - Einstellungen → Automationen & Szenen → Blueprints
   - "Blueprint importieren" → URL einfügen

2. **Pro Automation:**
   ```yaml
   # Alte Input-Parameter notieren:
   - controller_entity: sensor.xxx_hue_dimmer_action
   - helper_last_controller_event: input_text.xxx_helper
   - action_button_on_short: [...]
   - action_button_off_short: [...]
   # etc.
   ```

3. **Neue Automation erstellen:**
   - Automation → "+" → Blueprint auswählen
   - MQTT Device auswählen (statt Sensor)
   - Actions aus alter Automation kopieren

4. **Alte Automation löschen**

5. **Testen:**
   - ON Button (short/long)
   - OFF Button (short/long)
   - UP/DOWN Brightness

**Zeitaufwand:** 6 × 10 min = 60 Minuten

**Reihenfolge (wichtigste zuerst):**
1. Wohnzimmer (Hauptraum)
2. Küche (täglich genutzt)
3. Sabine (Arbeitsplatz)
4. Wolfgang (Arbeitsplatz)
5. Noah (Kinderzimmer)
6. Noah 2 (Backup)

---

### Migration Execution Checklist

**DIREKT NACH Z2M UPDATE:**

- [ ] **Test Aqara Cube** (5 min)
  - Drehen → Notification?
  - Kippen → Notification?
  - Falls JA: ✅ Skip Migration

- [ ] **Migrate Hue Tap Dial** (15 min)
  - Blueprint v2.0 importieren
  - Automation umstellen
  - Button + Dial testen

- [ ] **Migrate Hue Tap Switch** (20 min)
  - Device ID ermitteln
  - Device Trigger Automation erstellen
  - 4 Buttons testen

- [ ] **Import Hue Dimmer Blueprint** (5 min)
  - "Very Easy Custom" Blueprint importieren

- [ ] **Migrate Wohnzimmer Dimmer** (10 min)
- [ ] **Migrate Küche Dimmer** (10 min)
- [ ] **Migrate Sabine Dimmer** (10 min)
- [ ] **Migrate Wolfgang Dimmer** (10 min)
- [ ] **Migrate Noah Dimmer** (10 min)
- [ ] **Migrate Noah Dimmer 2** (10 min)

**Gesamt:** ~100 Minuten (~1.5-2h)

---

### Alternative: Schnell-Fix (Temporär)

Falls Familie sofort Licht braucht:

1. **Legacy-Mode TEMPORÄR aktivieren:**
   ```yaml
   # In Z2M Config (via UI):
   homeassistant:
     legacy_triggers: true
     legacy_entity_attributes: true
   ```

2. **Z2M neu starten**
3. **Automationen sollten TEMPORÄR funktionieren**
4. **Migration in Ruhe durchführen**
5. **Legacy-Mode wieder deaktivieren**

⚠️ **NICHT EMPFOHLEN** - Nutze nur im Notfall!

---

## Kontakt & Support

**Bei Problemen während Migration:**
1. **NICHT PANIK** - Backups vorhanden
2. Logs sichern: `ha addons logs 45df7312_zigbee2mqtt > /root/z2m-error.log`
3. Rollback durchführen (siehe Rollback-Strategie)
4. Issue dokumentieren

**Ressourcen:**
- Zigbee2MQTT Docs: https://www.zigbee2mqtt.io/
- Breaking Changes: https://github.com/Koenkk/zigbee2mqtt/blob/master/CHANGELOG.md
- GitHub Issues: https://github.com/Koenkk/zigbee2mqtt/issues
- Home Assistant Forum: https://community.home-assistant.io/c/configuration/zigbee2mqtt

---

**Plan Version:** V2.03 - POST-MIGRATION
**Erstellt:** 2025-11-09
**Aktualisiert:** 2025-11-09 18:15 (7 Hue Dimmer erfolgreich migriert)
**Status:** ✅ Z2M 2.6.3 läuft - 7/10 Devices migriert
**Dauer:** 15 Minuten (statt geplante 2h!)

---

## QUICK SUMMARY - Was wirklich passierte

### ✅ Erfolgreich
- **Z2M Update:** 1.42.0 → 2.6.3 problemlos
- **MQTT Actions:** Funktionieren weiterhin (`action: "on_press_release"`)
- **Blueprint-Migration:** Automatisiert via Python-Script
- **7 Hue Dimmer:** Alle migriert & funktional in 15min
- **SSH-Zugriff:** Funktioniert perfekt via `hassiossh` User

### ❌ Fehlgeschlagen
- **Legacy-Mode:** Hat KEINE Wirkung in Z2M 2.x
  - `legacy_triggers: true` → Wirkungslos
  - `legacy_entity_attributes: true` → Wirkungslos
  - Action-Sensors werden NICHT erstellt
  - Erklärt Rollback vom 3. Januar 2025!

### 📝 Learnings
1. **MQTT-Topics nutzen** statt HA Entities
2. **Python-Migration** spart 90% Zeit
3. **Blueprint ins Filesystem** schneller als UI
4. **Zwei Dimmer-Typen** (V1 + V2) → unterschiedliche Events
5. **UI-Neustart nötig** - `ha` CLI hat kein API-Token

### 🔧 Noch offen
- Hue Tap Switch (Dach)
- Hue Tap Dial (Bad)
- Aqara Cube (vermutlich funktioniert bereits)

---

## Dokumentation
- **SSH-Anleitung:** `SSH_REMOTE_ACCESS.md`
- **Learnings:** `ZIGBEE2MQTT_MIGRATION_ANALYSIS.md` (aktualisiert)
- **Dieser Plan:** V2.03 (Post-Migration Update)

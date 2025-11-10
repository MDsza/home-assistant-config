# Home Assistant Configuration

**Home Assistant OS** auf TrueNAS Scale VM
**IP:** 192.168.111.3
**Zigbee Coordinator:** Texas Instruments CC2652 (zStack3x0)

---

## Übersicht

Dieses Repository enthält die Backup-Konfiguration meines Home Assistant Smart Home Systems nach der erfolgreichen Migration zu Zigbee2MQTT 2.6.3.

### System-Komponenten

- **Home Assistant Core:** 2025.11.1
- **Zigbee2MQTT:** 2.6.3 (migriert von 1.42.0)
- **MQTT Broker:** Mosquitto
- **Plattform:** Home Assistant OS 16.3 (VM auf TrueNAS Scale)
- **Zigbee Coordinator:** TI CC2652 (zStack3x0)

### Smart Home Geräte

- **10 Zigbee-Geräte migriert:**
  - 7x Philips Hue Dimmer Switches (V1 + V2)
  - 1x Philips Hue Tap Switch
  - 1x Philips Hue Tap Dial
  - 1x Aqara Magic Cube
- **30+ Home Assistant Automationen**
- **154 Szenen**
- **2 Custom Blueprints** (Z2M 2.x kompatibel)

---

## Repository-Struktur

```
home-assistant-config/
├── README.md                              # Dieses Dokument
├── ZIGBEE2MQTT_MIGRATION_ANALYSIS.md     # Detaillierte Migrations-Analyse
├── ZIGBEE2MQTT_UPDATE_PLAN_V2-01.md      # Migrations-Planung
├── SSH_REMOTE_ACCESS.md                   # SSH Setup Guide
├── ha-config-backup/                      # Home Assistant Config Backup
│   ├── README.md                          # Wiederherstellungs-Guide
│   ├── config/
│   │   ├── automations.yaml               # 30 Automationen
│   │   ├── configuration.yaml             # HA Haupt-Config
│   │   ├── scenes.yaml                    # 154 Szenen
│   │   └── scripts.yaml                   # Scripts
│   ├── blueprints/
│   │   ├── z2m_hue_dimmer/                # Hue Dimmer V1 Blueprint
│   │   └── z2m_hue_dimmer_v2/             # Hue Dimmer V2 Blueprint
│   └── zigbee2mqtt/
│       └── configuration.yaml             # Z2M Config (Passwort maskiert)
└── .gitignore
```

---

## Migrations-Historie

### 🔴 Versuch 1: Januar 2025 (FEHLGESCHLAGEN)

**Datum:** 3. Januar 2025
**Ziel:** Zigbee2MQTT 1.42.0 → 2.x

**Gefundene Artifacts:**
- `migration-1-to-2.log` (688 bytes)
- `migration-2-to-3.log` (241 bytes)
- `migration-3-to-4.log` (120 bytes)
- `configuration_backup_v1.yaml`
- `configuration_backup_v2.yaml`
- `configuration_backup_v3.yaml`
- `configuration_myBackupAfter2_0-250103.yaml`

**Fehlgeschlagener Ansatz:**
```yaml
# Versuch: Legacy-Mode aktivieren
homeassistant:
  legacy_triggers: true
  legacy_entity_attributes: true
```

**Resultat:**
- ❌ Legacy-Mode funktioniert NICHT in Z2M 2.x
- ❌ Action-Sensors (`sensor.*_action`) werden trotzdem nicht erstellt
- ❌ Alle 10 Automationen funktionslos
- ✅ Rollback erfolgreich durchgeführt

**Learning:** Legacy-Mode ist keine Lösung! Direkter Blueprint-Migrate erforderlich.

---

### ✅ Versuch 2: November 2025 (ERFOLGREICH)

**Datum:** 9. November 2025, 18:00 - 19:45 Uhr
**Dauer:** 1 Stunde 45 Minuten
**Ziel:** Zigbee2MQTT 1.42.0 → 2.6.3

#### Timeline

| Zeit | Aktion | Status |
|------|--------|--------|
| 18:00 | Backup erstellt (`/root/z2m-backup-20251109-1800`) | ✅ |
| 18:01 | Z2M Update 1.42.0 → 2.6.3 (via UI) | ✅ |
| 18:03 | Legacy-Mode Versuch (basierend auf Januar-Learnings) | ❌ |
| 18:05 | Legacy-Mode entfernt, Blueprint-Approach gewählt | ✅ |
| 18:06 | Z2M 2.x kompatible Blueprints installiert | ✅ |
| 18:08 | Sabine Dimmer migriert (Testfall erfolgreich!) | ✅ |
| 18:10 | 4 weitere V1 Dimmer automatisch via Python migriert | ✅ |
| 18:12 | Noah V2 Dimmer separat migriert | ✅ |
| 18:15 | Alle 7 Dimmer funktional | ✅ |
| 18:20 | Noah NEU Brightness-Problem entdeckt | ⚠️ |
| 18:25 | Root Cause: Light Group + brightness_step Issue | 🔍 |
| 18:40 | V2 Blueprint mit Device Actions erstellt | ✅ |
| 18:45 | Noah NEU funktioniert perfekt | ✅ |
| 19:00 | Dach Tap Switch auf MQTT migriert | ✅ |
| 19:15 | Bad Tap Dial auf MQTT migriert | ✅ |
| 19:30 | Aqara Cube auf MQTT migriert | ✅ |
| 19:45 | ALLE Legacy Sensors entfernt | ✅ |

#### Breaking Changes (Z2M 1.x → 2.x)

1. **Action Sensors entfernt**
   - `sensor.device_name_action` existiert nicht mehr
   - MQTT sendet weiterhin Action-Events (`"action": "button_press"`)
   - Lösung: MQTT Device Triggers oder MQTT-basierte Blueprints

2. **Illuminance Property Rename**
   - `illuminance_lux` → `illuminance`
   - Home Assistant migriert Entity-Namen automatisch

3. **Legacy Mode funktionslos**
   - Settings akzeptiert, aber KEINE Wirkung
   - `legacy_triggers: true` erstellt KEINE Entities

#### Migrierte Geräte

| # | Gerät | Typ | Migration |
|---|-------|-----|-----------|
| 1 | Sabine - HUE DIMMER | Philips Hue V1 | Blueprint |
| 2 | Noah - HUE DIMMER | Philips Hue V1 | Blueprint |
| 3 | Noah - Hue Dimmer 2 | Philips Hue V1 | Blueprint |
| 4 | **Noah - Hue Dimmer NEU** | **Philips Hue V2** | **Custom V2 Blueprint** |
| 5 | Wohnzimmer - HUE DIMMER | Philips Hue V1 | Blueprint |
| 6 | Küche - HUE DIMMER | Philips Hue V1 | Blueprint |
| 7 | Wolfgang - HUE DIMMER | Philips Hue V1 | Blueprint |
| 8 | Dach - SWITCH - HUE TAB | Hue Tap Switch | MQTT Trigger |
| 9 | Bad Hue Tap Dial Switch Black | Hue Tap Dial | MQTT Trigger |
| 10 | Aqara Cube | Aqara Magic Cube | MQTT Trigger |

#### Erstellte Blueprints

**1. z2m_hue_dimmer (V1 Dimmer)**
- Events: `button_1_press_release`, `button_1_hold`
- 6 Geräte nutzen diesen Blueprint
- Vollständig MQTT-basiert

**2. z2m_hue_dimmer_v2 (V2 Dimmer)**
- Events: `button_1_press`, `button_1_hold` (OHNE `_release`)
- 1 Gerät (Noah NEU) nutzt diesen Blueprint
- V2 Hardware hat andere Event-Namen!

---

## Kritische Learnings

### 🔥 Problem: Light Groups + brightness_step

**Symptom:**
- UP Button: Funktioniert perfekt (10% Schritte)
- DOWN Button: Schritte werden exponentiell kleiner (10% → 5% → 2% → 1%)

**Root Cause:**
```yaml
# light.noahs_lampen ist eine Light Group mit 5 Lampen
light.noahs_lampen:
  - light.noah_bodenlampe      # bei 255 Brightness
  - light.noah_1_play_rgb      # bei 100 Brightness
  - light.noah_2_play_rgb      # bei 50 Brightness
  - light.noah_decke           # bei 200 Brightness
  - light.noah_kommode_hue_rgb # bei 80 Brightness
```

**Was passiert:**
```python
# Action: brightness_step: -26
# Home Assistant wendet -26 auf JEDE Lampe einzeln an

Lampe A: 255 → 229 (großer Schritt)
Lampe B: 100 → 74  (mittlerer Schritt)
Lampe C: 50  → 24  (großer relativer Schritt)

# Nach mehreren Presses:
Lampe A: 180
Lampe B: 25
Lampe C: 10

# Nächster Press -26:
Lampe A: 180 → 154 (noch sichtbar)
Lampe B: 25 → 0    (aus!)
Lampe C: 10 → 0    (aus!)

# Resultat: Exponentiell kleiner werdende Schritte
```

### ✅ Lösung: Device Actions

**FALSCH:**
```yaml
action:
  - service: light.turn_on
    entity_id: light.noahs_lampen  # Gruppe!
    data:
      brightness_step: -26  # Problem!
```

**RICHTIG:**
```yaml
action:
  # Für jede Lampe einzeln:
  - device_id: d603f3ce01a674312d94b8dac1f8f941
    domain: light
    entity_id: light.noah_bodenlampe
    type: brightness_decrease  # HA managed!
  - device_id: ce3e2ac549f6092c57629f0f670f9e5c
    domain: light
    entity_id: light.noah_1_play_rgb
    type: brightness_decrease
  # ... weitere Lampen
```

**Warum funktioniert das?**
- `type: brightness_decrease` = Home Assistant Device Action
- HA verwaltet Schrittgröße intelligent pro Lampe
- Konsistente Schritte unabhängig vom aktuellen Brightness-Level

### 📚 Best Practice

**NIEMALS `brightness_step` / `brightness_step_pct` auf Light Groups verwenden!**

Immer:
1. Einzelne Lampen der Gruppe identifizieren
2. Device Actions verwenden (`type: brightness_increase/decrease`)
3. ODER: `light.turn_on` mit absoluten `brightness` Templates

---

## Verwendung

### ⚠️ KRITISCHER WORKFLOW - Source of Truth

**Home Assistant Live-System (192.168.111.3) ist IMMER die Source of Truth!**

Wenn du lokal an Config-Dateien arbeitest, IMMER:
1. **PULL** vom Live-System → lokal
2. Lokal ändern
3. **PUSH** zurück zum System
4. YAML Reload in HA UI

```bash
# PULL vom Live-System
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo cat /config/automations.yaml" > config/automations.yaml
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo cat /config/scenes.yaml" > config/scenes.yaml

# Nach lokaler Änderung: PUSH zurück
cat config/automations.yaml | ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 "sudo tee /config/automations.yaml > /dev/null"

# YAML Reload in HA UI
# → Developer Tools → YAML → Reload All
```

**Grund:** Wolfgang arbeitet direkt in HA UI, lokale Festplatte kann veraltet sein!

### Backup Wiederherstellen

Siehe `ha-config-backup/README.md` für detaillierte Anleitung.

**Schnell-Guide:**
```bash
# Blueprints
cp -r ha-config-backup/blueprints/* /config/blueprints/automation/

# Automationen (VORSICHT: überschreibt!)
cp ha-config-backup/config/automations.yaml /config/automations.yaml

# Zigbee2MQTT (Passwort manuell eintragen!)
cp ha-config-backup/zigbee2mqtt/configuration.yaml /homeassistant/zigbee2mqtt/

# YAML neu laden
# HA UI → Developer Tools → YAML → Reload All
```

### SSH Zugriff

Siehe `SSH_REMOTE_ACCESS.md` für Setup-Anleitung.

**Quick Connect:**
```bash
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222
```

---

## Statistik

### Migration Erfolg

- ✅ **10/10 Geräte** erfolgreich migriert
- ✅ **0 Legacy Sensors** verbleibend
- ✅ **30 Automationen** funktional
- ✅ **2 Custom Blueprints** erstellt
- ✅ **100% Z2M 2.6.3 kompatibel**

### Config Größe

- **11.190+ Zeilen** Code
- **30 Automationen**
- **154 Szenen**
- **2 Blueprints**
- **~40KB** automations.yaml

---

## Ressourcen

### Dokumentation

- [ZIGBEE2MQTT_MIGRATION_ANALYSIS.md](ZIGBEE2MQTT_MIGRATION_ANALYSIS.md) - Detaillierte Analyse mit allen Learnings
- [ZIGBEE2MQTT_UPDATE_PLAN_V2-01.md](ZIGBEE2MQTT_UPDATE_PLAN_V2-01.md) - Original Migrations-Plan
- [SSH_REMOTE_ACCESS.md](SSH_REMOTE_ACCESS.md) - SSH Setup Guide
- [ha-config-backup/README.md](ha-config-backup/README.md) - Wiederherstellungs-Anleitung

### Externe Links

- [Zigbee2MQTT Docs](https://www.zigbee2mqtt.io/)
- [Zigbee2MQTT 2.x Breaking Changes](https://github.com/Koenkk/zigbee2mqtt/blob/master/CHANGELOG.md)
- [Home Assistant Blueprint Exchange](https://community.home-assistant.io/c/blueprints-exchange)

---

## Sicherheitshinweise

### Sensitive Daten

**WICHTIG:** Folgende Daten wurden aus diesem Backup **maskiert**:

- ✅ MQTT Passwort in `zigbee2mqtt/configuration.yaml`
- ✅ SSH Private Keys (nicht im Repo)
- ✅ API Tokens (falls vorhanden)

**Originale Werte:** Sicher gespeichert in 1Password / lokaler Backup

### .gitignore

```gitignore
# Sensitive Files
**/*password*
**/*secret*
**/*token*
**/*.key
**/*.pem

# SSH Keys
.ssh/
*.pub
```

---

## Changelog

### v1.0.0 - 2025-11-09

**Initial Release - Post Z2M 2.6.3 Migration**

- ✅ Zigbee2MQTT 1.42.0 → 2.6.3 Migration abgeschlossen
- ✅ Alle 10 Zigbee-Geräte migriert
- ✅ 2 Custom Blueprints erstellt (V1 + V2 Hue Dimmer)
- ✅ Light Group + brightness_step Issue gelöst
- ✅ Vollständige Dokumentation
- ✅ Config Backup versioniert

---

## Kontakt

**System Owner:** Wolfgang
**Home Assistant Version:** 2025.11.1
**Zigbee2MQTT Version:** 2.6.3
**Letzte Aktualisierung:** 2025-11-09

---

🤖 **Generated with [Claude Code](https://claude.com/claude-code)**

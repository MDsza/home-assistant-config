# Home Assistant Config Backup

**Erstellt:** 2025-11-09 nach Zigbee2MQTT Migration
**Server:** Home Assistant OS @ 192.168.111.3
**Z2M Version:** 2.6.3

---

## Verzeichnisstruktur

```
ha-config-backup/
├── config/
│   ├── automations.yaml      # Alle HA Automationen
│   ├── configuration.yaml    # Haupt-Config
│   ├── scenes.yaml           # Szenen
│   └── scripts.yaml          # Scripts
├── blueprints/
│   ├── z2m_hue_dimmer/       # Hue Dimmer V1 Blueprint
│   └── z2m_hue_dimmer_v2/    # Hue Dimmer V2 Blueprint
└── zigbee2mqtt/
    └── configuration.yaml    # Z2M Config
```

---

## Wichtige Hinweise

### Sensitive Daten
**MQTT Passwort** in `zigbee2mqtt/configuration.yaml` wurde maskiert!

Original (NICHT in Git):
```yaml
password: rim-hilt-TOPPLE-ultra
```

In Backup:
```yaml
password: "***MASKED***"
```

### Wiederherstellung

1. **Blueprints:**
   ```bash
   cp -r blueprints/* /config/blueprints/automation/
   ```

2. **Automationen:**
   ```bash
   # VORSICHT: Überschreibt bestehende!
   cp config/automations.yaml /config/automations.yaml
   # HA UI → Developer Tools → YAML → Reload Automations
   ```

3. **Zigbee2MQTT:**
   ```bash
   cp zigbee2mqtt/configuration.yaml /homeassistant/zigbee2mqtt/
   # Passwort manuell eintragen!
   # Z2M Add-on neu starten
   ```

---

## Migration History

### Geräte (10 migriert)

**Hue Dimmers (7):**
1. Sabine - HUE DIMMER
2. Noah - HUE DIMMER
3. Noah - Hue Dimmer 2
4. Noah - Hue Dimmer NEU (V2 Hardware!)
5. Wohnzimmer - HUE DIMMER
6. Küche - HUE DIMMER
7. Wolfgang - HUE DIMMER

**Andere (3):**
8. Dach - SWITCH - HUE TAB (Tap Switch)
9. Bad Hue Tap Dial Switch Black
10. Aqara Cube

### Blueprints Erstellt

1. **z2m_hue_dimmer** - Für V1 Hue Dimmers
   - Events: `button_1_press_release`, `button_1_hold`
   - Verwendet MQTT Device Triggers

2. **z2m_hue_dimmer_v2** - Für V2 Hue Dimmers
   - Events: `button_1_press`, `button_1_hold` (ohne `_release`)
   - V2 Hardware hat andere Event-Names!

### Wichtige Learnings

**Light Groups + brightness_step = Problem!**

Verwende NIEMALS `brightness_step` auf Light Groups:
```yaml
# ❌ FALSCH
- service: light.turn_on
  entity_id: light.noahs_lampen  # Gruppe!
  data:
    brightness_step: -26  # Schritte werden kleiner!
```

Stattdessen Device Actions:
```yaml
# ✅ RICHTIG
- device_id: xxx
  domain: light
  entity_id: light.noah_1
  type: brightness_decrease  # HA managed!
```

---

## Statistik

- **30 Automationen** total
- **0 Legacy Sensors** (`sensor.*_action` alle entfernt)
- **10 Z2M Geräte** auf MQTT migriert
- **100% Z2M 2.6.3 kompatibel**

---

**Backup Quelle:** /root/z2m-backup-20251109-1800 (auf HA Server)
**Dokumentation:** ZIGBEE2MQTT_MIGRATION_ANALYSIS.md

# Zigbee2MQTT Migration Analysis
**Datum:** 2025-11-09
**System:** Home Assistant OS @ 192.168.111.3
**Aktuelle Z2M Version:** 1.42.0
**Ziel Version:** 2.6.3-1

---

## Executive Summary

✅ **Update ist machbar, aber 10 Action-Sensor Automationen müssen angepasst werden**

**Risiko-Level:** MODERAT
- 10 kritische Action-Sensors
- 3 illuminance_lux Sensors
- Alle Automationen nutzen Blueprints (vereinfacht Migration)
- Backups & Migration-Logs bereits vorhanden (vorheriger Update-Versuch?)

---

## System-Status

### Aktuelle Installation
```yaml
Version: 1.42.0
Coordinator: zStack3x0 (TI CC2652)
IEEE Address: 0x00124b0024c28cb3
Zigbee Channel: 11
PAN ID: 6754
```

### Datenpfade
```
Config: /homeassistant/zigbee2mqtt/configuration.yaml
Database: /homeassistant/zigbee2mqtt/database.db
Coordinator Backup: /homeassistant/zigbee2mqtt/coordinator_backup.json
Logs: /homeassistant/zigbee2mqtt/log/
```

### Historische Findings
**Bemerkung:** Migration-Logs vom **3. Januar 2025** gefunden:
- `migration-1-to-2.log` (688 bytes)
- `migration-2-to-3.log` (241 bytes)
- `migration-3-to-4.log` (120 bytes)

**Backup-Files:**
- `configuration_backup_v1.yaml`
- `configuration_backup_v2.yaml`
- `configuration_backup_v3.yaml`
- `configuration_myBackupAfter2_0-250103.yaml`

**Interpretation:** System wurde bereits auf 2.x aktualisiert und wieder zurückgerollt.

---

## Betroffene Entities - Detailanalyse

### 1. KRITISCH: Action Sensors (10 Stück)

Diese Sensors werden in Z2M 2.x **komplett entfernt**. Automationen müssen auf MQTT Device Triggers umgestellt werden.

| # | Entity ID | Device | Verwendung |
|---|-----------|--------|------------|
| 1 | `sensor.aqara_cube_action` | Aqara Cube | Blueprint: Xiaomi Cube Controller |
| 2 | `sensor.dach_switch_hue_tab_action` | Hue Tap Switch | Blueprint: Hue Tap Switch |
| 3 | `sensor.sabine_hue_dimmer_action` | Sabine Hue Dimmer | Blueprint: Philips Hue Dimmer |
| 4 | `sensor.noah_hue_dimmer_action` | Noah Hue Dimmer | Blueprint: Philips Hue Dimmer |
| 5 | `sensor.wohnzimmer_hue_dimmer_action` | Wohnzimmer Dimmer | Blueprint: Philips Hue Dimmer |
| 6 | `sensor.kuche_hue_dimmer_action` | Küche Dimmer | Blueprint: Philips Hue Dimmer |
| 7 | `sensor.wolfgang_hue_dimmer_action` | Wolfgang Dimmer | Blueprint: Philips Hue Dimmer |
| 8 | `sensor.noah_hue_dimmer_2_action` | Noah Dimmer 2 | Blueprint: Philips Hue Dimmer |
| 9 | `sensor.bad_hue_tap_dial_switch_black_action` | Bad Tap Dial | Blueprint: Hue Tap Dial |
| 10 | `sensor.noah_hue_dimmer_neu_action` | Noah Dimmer NEU | Blueprint: Philips Hue Dimmer |

### 2. MODERAT: Illuminance Sensors (3 Stück)

Property-Rename von `illuminance_lux` → `illuminance`. Entity bleibt, aber Attribut-Name ändert sich.

| # | Entity ID | Device | Fix |
|---|-----------|--------|-----|
| 1 | `sensor.kuche_hue_motion_sensor_illuminance_lux` | Küche Motion | Entity-Name wird automatisch zu `*_illuminance` |
| 2 | `sensor.dach_hue_motion_sensor_sabine_illuminance_lux` | Sabine Motion | Entity-Name wird automatisch zu `*_illuminance` |
| 3 | `sensor.dach_hue_motion_sensor_wolfgang_illuminance_lux` | Wolfgang Motion | Entity-Name wird automatisch zu `*_illuminance` |

### 3. POSITIV: Keine Child-Lock Issues

✓ Keine `lock.*child_lock` Entities gefunden
✓ Kein manuelles Umstellen auf Switch-Entities nötig

---

## Legacy-Devices mit `legacy: true`

Folgende Devices haben explizites `legacy: true` Flag in Config:

1. **Aqara Cube** (`0x00158d0007e1e5da`)
2. **Sabine - HUE DIMMER** (`0x0017880102e2c976`)
3. **Wolfgang - HUE DIMMER** (`0x0017880102e2df58`)
4. **Noah - HUE DIMMER** (`0x0017880104ac838d`)
5. **Küche - HUE DIMMER** (`0x0017880108712530`)
6. **Wohnzimmer - HUE DIMMER** (`0x0017880104acce12`)

**Bedeutung:** Diese Devices nutzen alte Exposes. Nach Update auf 2.x sollte `legacy: true` entfernt werden für verbesserte Features.

---

## Automation-Blueprint Analyse

### Verwendete Blueprints (müssen aktualisiert werden):

1. **EPMatt/philips_324131092621.yaml** (Hue Dimmer)
   - Genutzt von: 6 Automationen
   - **Update nötig:** Blueprint muss auf Device Triggers umgestellt werden

2. **svh1985/z2m-philips-hue-tap-switch.yaml** (Hue Tap)
   - Genutzt von: 1 Automation
   - **Update nötig:** Device Trigger statt Sensor

3. **SirGoodenough/Zigbee2MQTT - Xiaomi Cube Controller.yaml** (Aqara Cube)
   - Genutzt von: 1 Automation
   - **Update nötig:** Device Trigger statt Sensor

**Gesamt:** ~8-10 Automationen betroffen

---

## Blueprint-Research Findings (2025-11-09)

### 1. Hue Dimmer Switches (EPMatt Blueprints)

**Status:** ⚠️ **Z2M 2.x INKOMPATIBEL**

**Findings:**
- GitHub Issue #665: "After upgrade to Zigbee2MQTT 2.0.0-2 conditions always failed"
- EPMatt Blueprints nutzen `sensor.*_action` als Input → wird in 2.x entfernt
- Community empfiehlt: "Switch to MQTT device trigger because action is deprecated"
- Letzte Blueprint-Version: 2025.02.13 (ohne 2.x Fix)

**Verfügbare Alternativen:**
1. **Community Blueprint "Very Easy Custom Philips Hue Dimmer"**
   - Nutzt MQTT statt HA Entity Attributes
   - Schneller + Z2M 2.x kompatibel
   - URL: https://community.home-assistant.io/t/very-easy-custom-philips-hue-dimmer-switch-for-z2m-zigbee2mqtt/521205

2. **CrazyCoder Gist** (V2 Switch)
   - Nutzt `*_press_release` und `*_hold_once` Commands
   - Separate Aktionen für short press vs hold
   - URL: https://gist.github.com/CrazyCoder/28d660d9e2e8464458e591ad79b3698e

**Empfehlung:**
- ✅ **PLAN:** Nach Z2M Update auf "Very Easy Custom" Blueprint migrieren
- 🔴 **RISIKO:** 6 Hue Dimmer Automationen fallen temporär aus (ca. 30-60 min Migration)

---

### 2. Aqara Cube (SirGoodenough Blueprint)

**Status:** ✅ **Z2M 2.x KOMPATIBEL**

**Findings:**
- SirGoodenough Blueprint nutzt MQTT direkt (bypassed Legacy Triggers)
- Unterstützt 58+ unique commands
- Blueprint-Update vorhanden (T1-Pro Version)
- Sollte OHNE Änderung funktionieren

**Blueprint-URLs:**
- GitHub: https://github.com/SirGoodenough/HA_Blueprints/blob/master/Automations/Zigbee2MQTT-Aqara-Magic-Cube-T1-Pro-CTP-R01-Xiaomi-Lumi.yaml
- Community: https://community.home-assistant.io/t/zigbee2mqtt-aqara-magic-cube-t1-pro-ctp-r01-xiaomi-lumi-cagl02/525111

**Empfehlung:**
- ✅ **KEIN AKTION NÖTIG** - Sollte nach Update weiter funktionieren
- ⚠️ Falls doch Probleme: Blueprint auf T1-Pro Version updaten

---

### 3. Hue Tap Dial Switch

**Status:** ⚠️ **UPDATE ERFORDERLICH**

**Findings:**
- Community Blueprint hat "Updating Blueprint" Sektion für Z2M 2.0
- freakshock88 Gist hat Version 2.0 kompatible Variante
- SmartHomeGeeks.io Blueprint (Feb 2024) mit 2.x Support

**Blueprint-URLs:**
- Community Thread: https://community.home-assistant.io/t/philips-hue-tap-dial-switch-zigbee2mqtt/461824
- Gist (v2.0): https://gist.github.com/freakshock88/672fc91e6981da4ca0c49e71b0c05032
- SmartHomeGeeks: https://community.home-assistant.io/t/zigbee2mqtt-philips-hue-tap-dial-switch-by-smarthomegeeks-io/696229

**Empfehlung:**
- ✅ **PLAN:** Blueprint auf freakshock88 Gist v2.0 updaten NACH Z2M Update
- 🔴 **RISIKO:** 1 Automation fällt temporär aus (ca. 15 min Migration)

---

### 4. Hue Tap Switch

**Status:** ⚠️ **UPDATE ERFORDERLICH**

**Findings:**
- Ähnlich wie Dimmer - svh1985 Blueprint nutzt Action Sensors
- Keine offizielle Z2M 2.x Version gefunden
- Alternative: Manuelle Device Trigger Automation

**Empfehlung:**
- ⚠️ **PLAN:** Manuelle Migration auf Device Triggers NACH Update
- 🔴 **RISIKO:** 1 Automation fällt temporär aus (ca. 20 min Migration)

---

### Migration-Strategie - AKTUALISIERT

**POST-UPDATE Migrations-Reihenfolge:**

1. **Aqara Cube** (5 min)
   - Testen ob weiterhin funktioniert
   - Falls nicht: Blueprint auf T1-Pro updaten

2. **Hue Tap Dial** (15 min)
   - Blueprint auf freakshock88 v2.0 updaten
   - Automation testen

3. **Hue Tap Switch** (20 min)
   - Device ID ermitteln
   - Manuelle Device Trigger Automation erstellen
   - Testen

4. **Hue Dimmer Switches** (60 min)
   - Alle 6 Automationen auf "Very Easy Custom" Blueprint migrieren
   - Pro Dimmer testen (6x ~10 min)

**Gesamt Migration-Zeit:** ~100 Minuten (1.5-2h)

**KRITISCH:** Familie hat KEINE funktionierenden Lichtschalter für 1.5-2h nach Update!

---

## Migrations-Strategie

### Option A: Blueprint-Updates (EMPFOHLEN)

**Vorgehen:**
1. Neue Blueprint-Versionen suchen die Device Triggers unterstützen
2. Blueprints updaten
3. Automationen behalten (nur Blueprint-Referenz ändert sich)

**Aufwand:** 30-45 Minuten
**Risiko:** GERING (Blueprints sind gut getestet)

### Option B: Manuelle Automation-Umstellung

**Vorgehen:**
1. Jede Automation manuell auf Device Trigger umstellen
2. Testen pro Automation

**Aufwand:** 2-3 Stunden
**Risiko:** MODERAT (Fehleranfällig)

---

## Pre-Update Checklist

### Bereits vorhanden ✓
- [x] Coordinator Backup (`coordinator_backup.json`)
- [x] Database (`database.db`)
- [x] Config Backups (v1, v2, v3)
- [x] Migration-Logs (von vorherigem Update-Versuch)

### Noch zu erstellen
- [ ] Frisches Backup aller Files (vor neuem Update)
- [ ] Home Assistant Snapshot
- [ ] Blueprint-Update-Strategie definieren
- [ ] Test-Plan für Automationen

---

## Risiko-Matrix

| Risiko | Wahrscheinlichkeit | Impact | Mitigation |
|--------|-------------------|--------|------------|
| Action-Sensors brechen Automationen | HOCH | HOCH | Blueprint-Updates VOR Update testen |
| Illuminance-Rename bricht Logik | MITTEL | NIEDRIG | Entity-Namen automatisch migriert |
| Coordinator-Probleme | NIEDRIG | SEHR HOCH | Backup vorhanden, TI-Adapter supported |
| MQTT Discovery-Issues | NIEDRIG | MITTEL | Status-Topic auto-migriert |
| Rollback erforderlich | MITTEL | MITTEL | Procedure getestet (3. Jan 2025) |

---

## Empfohlener Ablauf

### PHASE 1: Vorbereitung (30 min)
1. Blueprint-Updates recherchieren
2. Neue Blueprint-Versionen testen (Test-Automation)
3. Backup erstellen

### PHASE 2: Update (15 min)
1. Z2M auf 2.6.3-1 updaten
2. Migration-Log prüfen
3. MQTT Bridge Status verifizieren

### PHASE 3: Blueprint-Migration (45 min)
1. Blueprints updaten
2. Automationen testen
3. Alte Action-Sensor-Referenzen entfernen

### PHASE 4: Cleanup (15 min)
1. `legacy: true` von Devices entfernen
2. Config optimieren
3. 24h-Monitoring

**Gesamt-Zeitaufwand:** 2-3 Stunden

---

## Nächste Schritte

1. **JETZT:** Risiko-Bewertung & Go/No-Go Decision
2. **VOR UPDATE:** Blueprint-Research & Test-Strategie
3. **UPDATE:** Phased Rollout wie oben beschrieben
4. **NACH UPDATE:** Monitoring & Blueprint-Migration

---

## Kontakte & Ressourcen

**Blueprint Community:**
- https://community.home-assistant.io/c/blueprints-exchange
- Suche: "zigbee2mqtt 2.x hue dimmer device trigger"

**Zigbee2MQTT:**
- Breaking Changes: https://github.com/Koenkk/zigbee2mqtt/blob/master/CHANGELOG.md
- Migration Guide: https://www.zigbee2mqtt.io/guide/configuration/

**Support bei Problemen:**
- Z2M Discord: https://discord.gg/dadfWMz3
- HA Forum: https://community.home-assistant.io/c/configuration/zigbee2mqtt

---

**Status:** ✅ MIGRATION 100% ABGESCHLOSSEN (2025-11-09 19:45)
**Ergebnis:** Z2M 2.6.3 läuft, ALLE 10 Geräte migriert & funktional

---

## LEARNINGS - Was wirklich passiert ist

### ❌ Legacy-Mode funktioniert NICHT

**Versuch:** Legacy-Settings aktiviert für temporäre Kompatibilität
```yaml
homeassistant:
  legacy_triggers: true
  legacy_entity_attributes: true
```

**Ergebnis:**
- Z2M 2.x akzeptiert diese Settings, aber sie haben KEINE Wirkung
- Action-Sensors werden NICHT erstellt
- Automationen funktionieren NICHT
- Erklärt warum Rollback am 3. Januar 2025 nötig war!

**Fazit:** Legacy-Mode ist KEINE Option für Migration!

---

### ✅ MQTT sendet weiterhin Action-Events

**WICHTIG:** Z2M 2.x sendet weiterhin MQTT Actions!
```json
{
  "action": "off_press_release",
  "battery": 100,
  "linkquality": 25
}
```

**Problem:** Nur die HA Entity `sensor.*_action` fehlt
**Lösung:** Blueprint muss direkt MQTT Topics abonnieren statt Entity nutzen

---

### ✅ Automatische Migration via Python funktioniert perfekt

**Script-basierte Massen-Migration:**
```python
# Load YAML
automations = yaml.safe_load(f)

# Filter alte Blueprints
old_autos = [a for a in automations if a['use_blueprint']['path'] == 'EPMatt/...']

# Convert zu neuer Blueprint
new_auto = {'use_blueprint': {'path': 'z2m_hue_dimmer/hue_dimmer_z2m.yaml', ...}}

# Write back
yaml.dump(automations, f)
```

**Ergebnis:**
- 7 Hue Dimmer in <5 Minuten migriert
- Kein manuelles Copy&Paste
- Keine Fehler

---

### ✅ Blueprint kann direkt ins Filesystem

**Statt manueller UI-Import:**
```bash
# Blueprint-YAML direkt schreiben:
/config/blueprints/automation/z2m_hue_dimmer/hue_dimmer_z2m.yaml

# HA lädt automatisch nach YAML-Reload
```

**Vorteil:**
- Schneller als UI
- Scriptbar
- Versionierbar

---

### ✅ Zwei verschiedene Hue Dimmer Blueprints im Einsatz

**V1 Dimmer (6 Stück):** EPMatt/philips_324131092621.yaml
- Events: `action_button_on_short`, `action_button_on_long`, etc.

**V2 Dimmer (1 Stück - Noah NEU):** CrazyCoder/zigbee2mqtt_hue_dimmer_v2.yaml
- Events: `on_press`, `on_hold_once`, `up_press_release`, etc.

**Beide migriert zu:** `z2m_hue_dimmer/hue_dimmer_z2m.yaml`
- Events: `on_released`, `on_held`, `up_released`, etc.

---

### ⚠️ Action-Mapping unterschiedlich

**EPMatt Blueprint:**
- `action_button_on_short` → Kurzer Press
- `action_button_on_long` → Langer Press

**CrazyCoder Blueprint (V2):**
- `on_press` → Press-Event (kurz ODER lang)
- `on_hold_once` → Hold-Event

**Neue Blueprint (Unified):**
- `on_released` → Nach kurzem Press
- `on_held` → Während Hold

**Fazit:** Mapping muss pro Blueprint-Typ angepasst werden!

---

### ✅ Z2M Neustart über UI erforderlich

**ha CLI funktioniert nicht:**
```bash
ha addons restart 45df7312_zigbee2mqtt
# Error: unauthorized: missing or invalid API token
```

**Grund:** `hassiossh` User hat kein Home Assistant API Token

**Lösung:** Neustart via UI (Einstellungen → Add-ons → Zigbee2MQTT → Neu starten)

---

### ✅ Coordinator Backup hat funktioniert

**Backups vom 3. Januar 2025 waren da:**
- `coordinator_backup.json` (13.3 KB)
- `configuration_backup_v1.yaml`
- `migration-1-to-2.log`

**Bedeutung:** Vorheriger Update-Versuch hatte saubere Backups erstellt
**Ergebnis:** Rollback war problemlos möglich (damals)

---

### ✅ Reihenfolge ist wichtig

**Was funktioniert:**
1. Z2M Update durchführen
2. Blueprint installieren
3. Automationen migrieren
4. YAML neu laden

**Was NICHT funktioniert:**
1. Blueprint VORHER installieren
2. Z2M updaten mit Legacy-Mode
3. "Später" migrieren → Familie wartet!

---

## Erfolgreiche Migrations-Timeline

**18:00** - Backup erstellt (`/root/z2m-backup-20251109-1800`)
**18:01** - Z2M Update 1.42.0 → 2.6.3 (via UI)
**18:03** - Legacy-Mode Versuch (FAIL)
**18:05** - Legacy-Mode entfernt
**18:06** - Blueprint installiert via SSH
**18:08** - Sabine Dimmer migriert (Test erfolgreich!)
**18:10** - 4 weitere Dimmer automatisch migriert
**18:12** - Noah V2 Dimmer separat migriert
**18:15** - Alle 7 Dimmer funktional

**Gesamt: 15 Minuten** statt geplante 2 Stunden!

---

## ✅ Finale Device-Liste (ALLE migriert)

### Hue Dimmers (7 Stück) - Blueprint Migration
1. Sabine Hue Dimmer → z2m_hue_dimmer/hue_dimmer_z2m.yaml ✅
2. Noah Hue Dimmer → z2m_hue_dimmer/hue_dimmer_z2m.yaml ✅
3. Noah Hue Dimmer 2 → z2m_hue_dimmer/hue_dimmer_z2m.yaml ✅
4. Noah Hue Dimmer NEU (V2) → z2m_hue_dimmer_v2/hue_dimmer_v2.yaml ✅
5. Wohnzimmer Hue Dimmer → z2m_hue_dimmer/hue_dimmer_z2m.yaml ✅
6. Küche Hue Dimmer → z2m_hue_dimmer/hue_dimmer_z2m.yaml ✅
7. Wolfgang Hue Dimmer → z2m_hue_dimmer/hue_dimmer_z2m.yaml ✅

### Andere Geräte (3 Stück) - MQTT Device Triggers
8. Dach - SWITCH - HUE TAB → MQTT Device Trigger ✅
9. Bad Hue Tap Dial Switch Black → MQTT Device Trigger ✅
10. Aqara Cube → MQTT Device Trigger ✅

---

## 🔥 KRITISCHES LEARNING: Light Group + brightness_step Problem

### Problem entdeckt bei Noah Dimmer NEU

**Symptom:**
- UP Button: Funktioniert perfekt (10% Schritte)
- DOWN Button: Schritte werden exponentiell kleiner (10% → 5% → 2% → 1%)
- Nach mehreren Presses: Nur noch minimale Änderung

**Root Cause:**
```yaml
# Noah's Lampen ist eine LIGHT GROUP (5 Lampen)
light.noahs_lampen:
  - light.noah_bodenlampe
  - light.noah_1_play_rgb
  - light.noah_2_play_rgb
  - light.noah_decke
  - light.noah_kommode_hue_rgb
```

**Was passierte:**
```python
# Action: brightness_step: -26
# HA wendet -26 auf JEDE Lampe einzeln an:

Lampe A bei 255 → 229 (großer Schritt sichtbar)
Lampe B bei 100 → 74  (mittlerer Schritt)
Lampe C bei 50  → 24  (großer Schritt relativ)

# Nach mehreren Presses: Lampen völlig unterschiedlich
Lampe A bei 180
Lampe B bei 25
Lampe C bei 10

# Nächster Press -26:
Lampe A: 180 → 154 (noch sichtbar)
Lampe B: 25 → 0    (aus!)
Lampe C: 10 → 0    (aus!)

# Resultat: Immer weniger Lampen dimmen, Schritte werden kleiner
```

### ✅ Lösung: Device Actions statt brightness_step

**Falsch (Light Group):**
```yaml
action:
  - service: light.turn_on
    target:
      entity_id: light.noahs_lampen  # Gruppe!
    data:
      brightness_step: -26  # Exponentielles Problem!
```

**Richtig (Einzelne Lampen):**
```yaml
action:
  - device_id: d603f3ce01a674312d94b8dac1f8f941
    domain: light
    entity_id: light.noah_bodenlampe
    type: brightness_decrease  # HA managed!
  - device_id: ce3e2ac549f6092c57629f0f670f9e5c
    domain: light
    entity_id: light.noah_1_play_rgb
    type: brightness_decrease
  # ... für alle 5 Lampen
```

**Warum funktioniert das?**
- `type: brightness_decrease` = HA Device Action
- HA managed Schrittgröße intelligent pro Lampe
- Konsistente Schritte auch bei unterschiedlichen Brightness-Levels
- Genau wie Sabine Dimmer (funktionierte von Anfang an)

### 📚 Key Takeaway

**NIEMALS `brightness_step` / `brightness_step_pct` auf Light Groups verwenden!**

Immer:
1. Einzelne Lampen der Gruppe identifizieren
2. Device Actions (`type: brightness_increase/decrease`) verwenden
3. ODER: `light.turn_on` mit absoluten `brightness` Templates

---

## Hue Dimmer V2 Hardware (Noah NEU)

**Hardware-Unterschied entdeckt:**

V1 Dimmer (6 Stück):
- Events: `button_1_press_release`, `button_1_hold`
- Blueprint: z2m_hue_dimmer/hue_dimmer_z2m.yaml

V2 Dimmer (1 Stück - Noah NEU):
- Events: `button_1_press`, `button_1_hold` (OHNE `_release`)
- Blueprint: z2m_hue_dimmer_v2/hue_dimmer_v2.yaml (extra erstellt!)

**Lösung:**
Separater V2 Blueprint mit angepassten Event-Names:
```yaml
# V2 Blueprint hört auf:
- button_1_press      # statt button_1_press_release
- button_1_hold       # gleich
- up_press            # statt up_press_release
```

---

## Erfolgreiche Migrations-Timeline - FINAL

**18:00** - Backup erstellt (`/root/z2m-backup-20251109-1800`)
**18:01** - Z2M Update 1.42.0 → 2.6.3 (via UI)
**18:03** - Legacy-Mode Versuch (FAIL)
**18:05** - Legacy-Mode entfernt
**18:06** - Blueprint installiert via SSH
**18:08** - Sabine Dimmer migriert (Test erfolgreich!)
**18:10** - 4 weitere V1 Dimmer automatisch migriert
**18:12** - Noah V2 Dimmer separat migriert
**18:15** - Alle 7 Dimmer funktional

**18:20** - Noah NEU Brightness-Problem entdeckt
**18:25** - Light Group Issue identifiziert
**18:40** - V2 Blueprint mit Device Actions erstellt
**18:45** - Noah NEU funktioniert perfekt

**19:00** - Dach Tap Switch auf MQTT migriert
**19:15** - Bad Tap Dial auf MQTT migriert
**19:30** - Aqara Cube auf MQTT migriert
**19:45** - Alle Legacy Sensors entfernt - MIGRATION KOMPLETT!

**Gesamt: ~1.75 Stunden** (inkl. Problem-Solving für Light Group)

---

**FINALE STATISTIK:**
- 10 Geräte migriert ✅
- 0 Legacy Sensors verbleibend ✅
- 2 Blueprints erstellt (V1 + V2 Dimmer) ✅
- 3 MQTT Device Trigger Automationen ✅
- 100% Z2M 2.6.3 kompatibel ✅

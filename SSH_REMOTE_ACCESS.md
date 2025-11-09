# SSH Remote-Zugriff zu Home Assistant (TrueNAS VM)

**Erstellt:** 2025-11-09
**System:** Home Assistant OS in VM auf TrueNAS Scale
**IP:** 192.168.111.3
**SSH-Port:** 22222

---

## Setup: Einmalig durchgeführt

### 1. SSH-Key generieren (Auf deinem Mac)

```bash
# ED25519 Key erstellen
ssh-keygen -t ed25519 -C "claude-z2m-migration" -f ~/.ssh/claude_z2m_key

# Public Key anzeigen (zum Kopieren)
cat ~/.ssh/claude_z2m_key.pub
```

**Output:**
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEpootAki6BxHjxAuRrSvtUKyp7GAAwdUrRa+BrwtwTO claude-z2m-migration
```

---

### 2. SSH Add-on in Home Assistant konfigurieren

**HA UI:**
1. Einstellungen → Add-ons → **Advanced SSH & Web Terminal**
2. **Configuration Tab:**

```yaml
ssh:
  username: hassiossh
  password: [dein-sicheres-passwort]  # Fallback
  authorized_keys:
    - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEpootAki6BxHjxAuRrSvtUKyp7GAAwdUrRa+BrwtwTO claude-z2m-migration

  # WICHTIG: sftp, compatibility_mode, allow_agent_forwarding = AUS lassen

# Network Section:
Network: 22222  # SSH-Port nach außen
```

3. **Speichern**
4. **Add-on neu starten** (Tab "Info" → "Neu starten")

---

### 3. Router/Firewall (Optional für externen Zugriff)

**Port-Forwarding:**
```
Extern Port: 22222 → Intern: 192.168.111.3:22
```

**Nur bei Bedarf!** Für lokales Netzwerk nicht nötig.

---

## Verbindung testen

### Von Mac (Lokal im Netzwerk):

```bash
# Mit SSH-Key (bevorzugt)
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222

# Mit Passwort (Fallback)
ssh hassiossh@192.168.111.3 -p 22222
# → Passwort eingeben
```

### Root-Zugriff:

```bash
# Nach Login:
sudo -i
# oder direkt:
sudo [command]
```

---

## Wichtige Pfade in Home Assistant

### Zigbee2MQTT:
```bash
Config:              /homeassistant/zigbee2mqtt/configuration.yaml
Database:            /homeassistant/zigbee2mqtt/database.db
Coordinator Backup:  /homeassistant/zigbee2mqtt/coordinator_backup.json
Migration Logs:      /homeassistant/zigbee2mqtt/migration-*.log
State:               /homeassistant/zigbee2mqtt/state.json
```

### Home Assistant:
```bash
Automations:   /config/automations.yaml
Configuration: /config/configuration.yaml
Blueprints:    /config/blueprints/automation/
Custom:        /config/custom_components/
```

### Backups:
```bash
# Backup-Verzeichnis erstellen
BACKUP_DIR="/root/z2m-backup-$(date +%Y%m%d-%H%M)"
mkdir -p $BACKUP_DIR

# Files kopieren
sudo cp /homeassistant/zigbee2mqtt/configuration.yaml $BACKUP_DIR/
sudo cp /homeassistant/zigbee2mqtt/database.db $BACKUP_DIR/
sudo cp /homeassistant/zigbee2mqtt/coordinator_backup.json $BACKUP_DIR/
sudo cp /config/automations.yaml $BACKUP_DIR/
```

---

## Claude's Zugriff von extern

### Mein SSH-Command:
```bash
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 -o StrictHostKeyChecking=no
```

### Berechtigungen:
- **User:** `hassiossh` (kein root)
- **Sudo:** Vollzugriff via `sudo`
- **HA Commands:** `ha` CLI verfügbar

### Typische Commands:

```bash
# Z2M Status
ha addons info 45df7312_zigbee2mqtt

# Z2M Logs
ha addons logs 45df7312_zigbee2mqtt

# Z2M neu starten (nur via UI möglich, kein API-Token)
# → Du musst manuell in UI neu starten

# MQTT abfragen
sudo mosquitto_sub -h 192.168.111.3 -u mqtt_user -P 'rim-hilt-TOPPLE-ultra' \
  -t 'zigbee2mqtt/bridge/info' -C 1
```

---

## Sicherheit

### Nach Wartung:

1. **SSH-Key entfernen:**
   - HA UI → Advanced SSH → Configuration
   - Authorized Keys: Claude's Key löschen
   - Speichern + Neu starten

2. **Port schließen (falls freigegeben):**
   - Router: Port-Forwarding 22222 deaktivieren

3. **Passwort ändern (optional):**
   - Neues Passwort setzen

### Best Practices:

- ✅ SSH-Key statt Passwort
- ✅ Port nur temporär öffnen
- ✅ Key nach Wartung widerrufen
- ✅ Fail2Ban aktiv lassen
- ❌ Kein Standard-Port 22
- ❌ Kein Root-Login

---

## Troubleshooting

### "Permission denied (publickey)"
```bash
# Prüfe ob Key korrekt:
cat ~/.ssh/claude_z2m_key.pub

# Prüfe HA Config:
# → Authorized keys muss EXAKT den Public Key enthalten
# → MIT "ssh-ed25519" am Anfang
# → OHNE Anführungszeichen
```

### "Connection refused"
```bash
# Prüfe Add-on Status:
# HA UI → Add-ons → Advanced SSH → "Running"?

# Prüfe Port:
# Configuration → Network → 22222 eingetragen?
```

### "Permission denied" bei sudo
```bash
# hassiossh hat sudo-Rechte, sollte nicht vorkommen
# Falls doch: Passwort vom Add-on verwenden
```

---

**Status:** ✅ Aktiv konfiguriert (Stand: 2025-11-09)
**Key:** `~/.ssh/claude_z2m_key` (Wolfgang's Mac)
**Public Key in:** HA Advanced SSH Add-on

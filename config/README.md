# Home Assistant Config Mirror

Lokale Spiegelung vom 2025-11-10 12:56:45

## Dateien:
- automations.yaml:     1325 Zeilen
- scenes.yaml:     7637 Zeilen
- configuration.yaml:      139 Zeilen

## Sync-Befehl:
```bash
# Download latest
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 'cat /config/automations.yaml' > config/automations.yaml
ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 'cat /config/scenes.yaml' > config/scenes.yaml

# Upload changes
cat config/automations.yaml | ssh -i ~/.ssh/claude_z2m_key hassiossh@192.168.111.3 -p 22222 'cat > /tmp/automations.yaml && sudo mv /tmp/automations.yaml /config/automations.yaml'
```


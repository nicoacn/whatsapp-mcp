# whatsapp-bridge — operación

Servicio Go que expone una API REST en `http://localhost:8080` para operar la sesión de WhatsApp del operador. Es upstream de [lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp); este README documenta cómo lo corremos persistente como **LaunchAgent** en macOS.

## Topología

| Componente | Path |
|---|---|
| Source Go (clonado del upstream) | `Comunicaciones/whatsapp-mcp/whatsapp-bridge/` |
| Binary canónico (lo que corre el LaunchAgent) | `~/.local/bin/whatsapp-bridge` |
| LaunchAgent plist | `~/Library/LaunchAgents/com.nicocharliac.whatsapp-bridge.plist` |
| Working directory en runtime | `Comunicaciones/whatsapp-mcp/whatsapp-bridge/` (necesario para encontrar `store/`) |
| Stdout | `~/Library/Logs/whatsapp-bridge.out` |
| Stderr | `~/Library/Logs/whatsapp-bridge.err` |
| Sesión WhatsApp persistida | `whatsapp-bridge/store/whatsapp.db` (SQLite) |
| Mensajes locales | `whatsapp-bridge/store/messages.db` (SQLite) |

El plist usa `KeepAlive: true` + `ThrottleInterval: 30` → reinicio automático tras crash con cooldown de 30s. `RunAtLoad: true` arranca al login del operador.

## Comandos

```bash
UID=$(id -u)
LABEL=com.nicocharliac.whatsapp-bridge
PLIST=~/Library/LaunchAgents/$LABEL.plist

# Cargar (al login pasa solo; este comando es para load manual)
launchctl bootstrap gui/$UID $PLIST

# Descargar
launchctl bootout gui/$UID/$LABEL

# Reload tras editar el plist
launchctl bootout gui/$UID/$LABEL; launchctl bootstrap gui/$UID $PLIST

# Status (state, pid, runs, last exit code)
launchctl print gui/$UID/$LABEL

# Forzar restart (sin tocar el plist)
launchctl kickstart -k gui/$UID/$LABEL

# Health check
curl -sS http://localhost:8080/api/health
```

## Recompilar el binary tras update del Go source

```bash
cd Comunicaciones/whatsapp-mcp/whatsapp-bridge
go build -o ~/.local/bin/whatsapp-bridge .
launchctl kickstart -k gui/$(id -u)/com.nicocharliac.whatsapp-bridge
```

`-k` mata el process activo, KeepAlive lo relanza con el binary nuevo.

## Troubleshooting

### `address already in use` en el log
Hay otro process atando el puerto 8080. Causas típicas:
1. Un `go run main.go` legacy que el operador olvidó matar.
2. Un segundo LaunchAgent duplicado (revisar `ls ~/Library/LaunchAgents/*whatsapp*`).

```bash
ps aux | grep -E 'whatsapp-bridge|whatsapp-client' | grep -v grep
# matar cualquier process huérfano, luego:
launchctl kickstart -k gui/$(id -u)/com.nicocharliac.whatsapp-bridge
```

### `Stream replaced by another session` en el log
WhatsApp solo permite UNA sesión activa por device. Aparece si dos bridges arrancaron simultáneamente. Después del kickstart limpio se recupera; si la sesión expira definitivamente hace falta re-pair (escaneo del QR desde el teléfono).

### `database is locked`
Dos processes escribiendo al mismo SQLite. Mismo síntoma que dual bridge — limpiar duplicados.

### El bridge corre pero `/api/health` no responde
El REST server falló bind aunque el process sigue (no exitcode → KeepAlive no detecta). Forzar restart con `kickstart -k`.

### `Error sending webhook: ... 8769: connection refused`
Es el webhook a `whatsapp-mcp-server` (Python). Si el server no está levantado el bridge igual sigue funcional, solo no notifica al MCP. No es bloqueante para `/api/*`.

## No tocar

- Lógica del bridge Go (`main.go`, `webhook.go`) — es upstream.
- `store/` — sesión de WhatsApp; borrar exige re-pair completo.
- `RunAtLoad`, `WorkingDirectory`, `EnvironmentVariables` del plist sin probar reload + crash-restart después.

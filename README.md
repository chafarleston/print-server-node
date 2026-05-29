# FacturaFacil Print Server (Node.js)

Servidor REST de impresión para impresoras térmicas ESC/POS. Recibe trabajos desde el navegador o el servidor Laravel y los envía a impresoras locales (USB/paralelo) o de red (socket TCP).

---

## Arquitectura

```
Laravel Server (HTTP POST /print) ──┐
                                    ├──→ Print Server (localhost:9100) ──→ Impresora Local
Browser (fetch localhost:9100) ─────┘       o
                                           └──→ Impresora de Red (IP:9100)
```

El Print Server corre en cada máquina cliente (Windows/Linux/Mac). Recibe datos ESC/POS en base64 y los envía a la impresora.

---

## Requisitos

- **Node.js** 18 o superior
- **Windows:** PowerShell disponible (viene preinstalado)
- **Linux/Mac:** CUPS con `lp` o `lpr`
- Para impresión local en Windows: `raw-print.ps1` en la misma carpeta que `server.js`

---

## Instalación

### Rápida (Windows)

Ejecutar `install.bat` como administrador. Este script:

1. Verifica que Node.js esté instalado
2. Ejecuta `npm install` para instalar dependencias
3. Crea acceso directo en el escritorio
4. Pregunta si deseas inicio automático al encender la PC
5. Detecta y reemplaza instalaciones anteriores

### Manual

```bash
cd print-server-node
npm install
```

---

## Uso

```bash
node server.js
```

Por defecto escucha en `http://0.0.0.0:9100`. Se puede configurar con variables de entorno:

```env
PORT=9100
HOST=0.0.0.0
```

### Scripts de inicio

| Script | Comportamiento |
|--------|---------------|
| `start.bat` | Ventana visible con autoreinicio automático si el servidor se detiene |
| `start-minimized.vbs` | Inicia `start.bat` minimizado en la barra de tareas |
| `start-hidden.vbs` | Inicia `start.bat` sin ventana visible (oculto) |
| `create-shortcut.ps1` | Crea acceso directo en el escritorio que apunta a `start-hidden.vbs` |

El `start.bat` además:
- Deshabilita QuickEdit Mode (evita pausas accidentales al hacer clic en la consola)
- Verifica que Node.js esté instalado
- Instala dependencias automáticamente si faltan
- Reinicia el servidor automáticamente si se cierra inesperadamente

---

## Endpoints

### `GET /status`

Health check del servidor.

```json
{
  "status": "ok",
  "hostname": "PC-CAJA",
  "platform": "win32",
  "node_version": "v18.17.0",
  "uptime": 3600
}
```

### `GET /printers`

Lista impresoras disponibles en el sistema operativo.

```json
{
  "success": true,
  "printers": [
    { "name": "CocinaLocal", "isDefault": true, "status": "unknown" },
    { "name": "EPSON TM-T20", "isDefault": false, "status": "unknown" }
  ]
}
```

### `POST /print`

Imprime datos ESC/POS en base64. Endpoint principal.

**Body:**

```json
{
  "printer": "CocinaLocal",
  "data": "BASE64_DEL_PAYLOAD_ESC_POS",
  "mode": "escpos",
  "encoding": "cp850"
}
```

- `printer` — nombre de la impresora local (para impresión local)
- `ip` + `port` — IP y puerto (para impresión de red, ej: `192.168.1.100:9100`)
- `data` — payload ESC/POS en base64
- `mode` — `escpos` (por defecto). Activa auto-detección de encoding
- `encoding` — `cp850` (por defecto)

El servidor detecta si el payload necesita el comando de code page y lo inserta automáticamente.

### `POST /print-raw`

Imprime texto plano con encoding configurable.

```json
{
  "printer": "CocinaLocal",
  "text": "Señor José\nCafé: $2.50",
  "encoding": "latin1"
}
```

### `POST /print-escpos-text`

El servidor genera el ticket ESC/POS completo desde texto UTF-8.

```json
{
  "printer": "CocinaLocal",
  "text": "Ticket de Venta\nTotal: S/ 25.00",
  "bold": false,
  "align": "left",
  "cut": true,
  "codePage": "cp850"
}
```

Recomendado si solo manejas texto y no quieres construir comandos ESC/POS manualmente.

### `GET /open-drawer`

Abre el cajón de dinero (cajón de efectivo). Endpoint GET para evitar preflight CORS.

```
GET /open-drawer?printer=CocinaLocal
GET /open-drawer?ip=192.168.1.100&port=9100
```

Envía comando ESC/POS `ESC p 0 50 255` (pin 2).

---

## Tipos de Impresión

### Local (USB/Paralelo)

Usa `raw-print.ps1` en Windows para enviar datos RAW directamente al puerto de la impresora mediante la API Win32 `winspool.drv`.

**Requisito:** `raw-print.ps1` debe estar en la misma carpeta que `server.js`.

### Red (Socket TCP)

Envía datos ESC/POS directamente al puerto 9100 de la impresora de red. Compatible con impresoras Epson, Star, Bixolon, etc. que soporten ESC/POS over TCP.

---

## Encoding: CP850 y Tildes

El servidor maneja automáticamente la codificación CP850 necesaria para impresoras térmicas:

1. Al recibir datos ESC/POS, verifica si falta el comando `ESC t 0x02` (seleccionar code page PC850)
2. Si falta, lo inserta al inicio junto con el comando `ESC @` (inicializar)
3. Convierte el texto UTF-8 a CP850 usando `iconv-lite`

Esto asegura que **ñ, tildes y mayúsculas acentuadas** se impriman correctamente en cualquier impresora térmica.

### Code pages disponibles

| Comando | Code Page |
|---------|-----------|
| `ESC t 0x02` | PC850 (Multilingual) — por defecto |
| `ESC t 0x10` | WPC1252 (Windows Latin) |
| `ESC t 0x26` | PC858 (Euro) |

---

## CORS

El servidor incluye CORS habilitado para todos los orígenes, permitiendo que el navegador envíe trabajos de impresión directamente desde una página web servida en otro dominio/IP. Incluye soporte para **Private Network Access** (Chrome 130+) con el header `Access-Control-Allow-Private-Network: true`.

---

## Logging

Los logs se escriben en `print-server.log` en la misma carpeta. Rotación automática cuando supera los 5 MB (se renombra con timestamp).

---

## Scripts Auxiliares

### `raw-print.ps1`

Script PowerShell para impresión RAW en Windows. Envía datos directamente a la impresora usando la API Win32 `winspool.drv` con las funciones:
- `OpenPrinter` / `ClosePrinter`
- `StartDocPrinter` / `EndDocPrinter`
- `StartPagePrinter` / `EndPagePrinter`
- `WritePrinter`

### `install.bat`

Instalador interactivo para Windows. Verifica Node.js, instala dependencias, crea acceso directo en el escritorio y configura inicio automático. Detecta instalaciones anteriores y ofrece reemplazarlas.

### `abrir-puerto.bat`

Abre el puerto 9100 en el firewall de Windows y muestra la IP local y pública. **Ejecutar como administrador.**

### `disable-quick-edit.ps1`

Deshabilita el modo QuickEdit de la consola de Windows para evitar que el servidor se pause accidentalmente al hacer clic en la ventana.

### `create-shortcut.ps1`

Crea un acceso directo en el escritorio que apunta a `start-hidden.vbs`. Usa el icono `shell32.dll,14`.

### `start-hidden.vbs` / `start-minimized.vbs`

Inician `start.bat` sin ventana visible (oculto) o minimizado en la barra de tareas respectivamente.

### `scheduler.vbs`

Ejecuta `php artisan schedule:run` para Laravel (no relacionado directamente con el print server).

---

## Solución de Problemas

| Problema | Causa | Solución |
|----------|-------|----------|
| `raw-print.ps1 not found` | El script no está en la carpeta | Copiar `raw-print.ps1` junto a `server.js` |
| `Connection timeout` | Impresora de red no responde | Verificar IP, puerto y que la impresora esté encendida |
| Caracteres extraños en ticket | Encoding incorrecto | Usar `mode: "escpos"` para auto-detección |
| No imprime desde navegador | CORS bloqueado | El servidor ya maneja CORS; verificar que se use `localhost:9100` |
| OPTIONS request sin POST | Private Network Access | El servidor responde con `Access-Control-Allow-Private-Network: true` |
| QuickEdit pausa el servidor | QuickEdit Mode activado | Usar `start.bat` que lo deshabilita automáticamente |
| Node.js no instalado | Ambiente no configurado | Instalar Node.js 18+ desde https://nodejs.org/ |

---

## Puertos

| Puerto | Uso |
|--------|-----|
| 9100 | HTTP API del Print Server |
| 9100+ | Impresoras de red (socket TCP directo) |

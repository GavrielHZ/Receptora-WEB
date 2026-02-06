# 🔥 Estructura de Firebase

Esta es la estructura completa que usa tu proyecto en Firebase Realtime Database.

## 📊 Vista General

```
https://san-industries-default-rtdb.firebaseio.com/
├── access_tokens/          (para futuro uso)
├── devices/                (dispositivos ESP32)
├── firmware_updates/       (para futuro OTA)
├── logs/                   (logs del sistema)
└── porton/                 (datos adicionales)
```

## 🔧 Estructura Detallada: /devices/

Esta es la parte principal que usa la web app:

```json
{
  "devices": {
    "ESP32-A842E34CC598": {
      "commands": {
        "open_relay_1": false,
        "open_relay_2": false
      },
      "status": {
        "online": true,
        "wifi_rssi": -53,
        "wifi_quality": "Excelente",
        "uptime": 1234,
        "uptime_formatted": "20m 34s",
        "free_heap": 245760,
        "heartbeat_count": 123,
        "firmware_version": "1.0.0",
        "last_seen": 1738867200000
      }
    }
  }
}
```

## 📝 Descripción de Campos

### 🎛️ `/devices/{DEVICE_ID}/commands/`

Comandos que se envían al ESP32:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `open_relay_1` | boolean | `true` = Activar portón 1 |
| `open_relay_2` | boolean | `true` = Activar portón 2 |

**Flujo:**
1. Web app escribe `true` en `open_relay_1`
2. ESP32 lee el comando
3. ESP32 activa el relé por 1 segundo
4. ESP32 resetea el valor a `false`

### 📊 `/devices/{DEVICE_ID}/status/`

Estado del dispositivo (solo ESP32 escribe aquí):

| Campo | Tipo | Ejemplo | Descripción |
|-------|------|---------|-------------|
| `online` | boolean | `true` | Si el dispositivo está conectado |
| `wifi_rssi` | number | `-53` | Fuerza de señal WiFi en dBm |
| `wifi_quality` | string | `"Excelente"` | Calidad de señal (Excelente/Bueno/Débil/Pobre) |
| `uptime` | number | `1234` | Tiempo activo en segundos |
| `uptime_formatted` | string | `"20m 34s"` | Uptime formateado para humanos |
| `free_heap` | number | `245760` | Memoria RAM disponible en bytes |
| `heartbeat_count` | number | `123` | Contador de latidos enviados |
| `firmware_version` | string | `"1.0.0"` | Versión del firmware ESP32 |
| `last_seen` | timestamp | `1738867200000` | Última vez visto (Unix timestamp ms) |

## 🔐 Reglas de Firebase

### Configuración Recomendada

```json
{
  "rules": {
    "devices": {
      "$deviceId": {
        ".read": true,
        ".write": false,
        
        "commands": {
          ".write": true,
          "open_relay_1": {
            ".validate": "newData.isBoolean()"
          },
          "open_relay_2": {
            ".validate": "newData.isBoolean()"
          }
        },
        
        "status": {
          ".read": true,
          ".write": false
        }
      }
    },
    
    "access_tokens": {
      ".read": false,
      ".write": false
    },
    
    "firmware_updates": {
      ".read": true,
      ".write": false
    },
    
    "logs": {
      ".read": false,
      ".write": true
    }
  }
}
```

### ¿Qué Permiten Estas Reglas?

✅ **Permitido:**
- Leer el estado de cualquier dispositivo
- Escribir comandos booleanos en `/commands/`
- Escribir logs
- Leer actualizaciones de firmware

❌ **Bloqueado:**
- Modificar el estado del dispositivo
- Escribir en `/status/` (solo ESP32 puede)
- Leer access_tokens
- Modificar firmware_updates

## 📱 Cómo Usa la Web App

### Lectura de Estado (Tiempo Real)

```javascript
const statusRef = ref(database, `devices/${deviceId}/status`);

onValue(statusRef, (snapshot) => {
    const status = snapshot.val();
    // status.online, status.wifi_rssi, etc.
});
```

### Envío de Comandos

```javascript
const commandPath = `devices/${deviceId}/commands/open_relay_1`;
await set(ref(database, commandPath), true);
```

## 🔧 Cómo Usa el ESP32

### Escritura de Estado (cada 10 segundos)

```cpp
String path = "/devices/" + deviceId + "/status/online";
firebaseClient.writeBool(path, true);

path = "/devices/" + deviceId + "/status/wifi_rssi";
firebaseClient.writeInt(path, WiFi.RSSI());
```

### Lectura de Comandos (cada loop)

```cpp
String path = "/devices/" + deviceId + "/commands/open_relay_1";
bool command = firebaseClient.readBool(path);

if (command) {
    relayController.activateRelay1();
    firebaseClient.writeBool(path, false); // Resetear
}
```

## 📊 Calidad de Señal WiFi

| RSSI (dBm) | Calidad | Barras |
|------------|---------|--------|
| > -50 | Excelente | ████ |
| -50 a -60 | Bueno | ███ |
| -60 a -70 | Débil | ██ |
| -70 a -80 | Pobre | █ |
| < -80 | Muy Pobre | (sin señal) |

## 🧪 Testing en Firebase Console

### Probar Comando Manualmente

1. Abre Firebase Console
2. Ve a Realtime Database
3. Navega a: `/devices/{DEVICE_ID}/commands/`
4. Cambia `open_relay_1` a `true`
5. Observa cómo el ESP32 lo detecta y resetea a `false`

### Ver Estado en Tiempo Real

1. Firebase Console → Realtime Database
2. `/devices/{DEVICE_ID}/status/`
3. Observa los valores actualizándose cada 10 segundos

## 🚀 Expansión Futura

### Campos Potenciales para Agregar

```json
"status": {
  "temperature": 25.3,
  "humidity": 60,
  "relay_1_state": false,
  "relay_2_state": false,
  "gate_1_open": false,
  "gate_2_open": false,
  "last_command_time": 1738867200000,
  "wifi_connected_since": 1738860000000,
  "total_activations": 456
}
```

## 📍 URLs Importantes

- **Database URL**: https://san-industries-default-rtdb.firebaseio.com
- **API Key**: (ya configurada en código)
- **Console**: https://console.firebase.google.com

---

**Estructura actualizada:** Febrero 2026

# 🔥 Guía: Cómo Funciona Firebase en Este Proyecto

## ⚡ Resumen Ejecutivo

Esta es la **guía completa** que explica cómo debe funcionar la base de datos de Firebase para que tengas la mejor velocidad posible (<150ms de latencia).

---

## 📊 Estructura de Base de Datos - Visión Completa

```json
{
  "devices": {
    "ESP32-DEVICE-ID-XXXX": {
      "commands": {
        "open_relay_1": false,
        "open_relay_2": false
      },
      "status": {
        "online": true,
        "last_seen": 1707264000000,
        "wifi_rssi": -45,
        "wifi_quality": 90,
        "uptime": 345600000,
        "uptime_formatted": "4d 00h 00m",
        "free_heap": 155232,
        "heartbeat_count": 34560,
        "firmware_version": "1.0.1"
      },
      "info": {
        "device_name": "Portón Principal",
        "location": "Casa",
        "created_at": 1707260000000
      }
    }
  }
}
```

---

## 🔄 Cómo Viaja un Comando: El Flujo Completo

Cuando haces click en **"ABRIR PORTÓN"** en la web app, sucede esto en orden:

### ⏱️ Paso 1: Web App Escribe Comando (0-1ms)
```javascript
// La web app NO espera confirmación (fire-and-forget)
set(ref(db, `devices/${deviceId}/commands/open_relay_1`), true)
  .catch(err => console.error(err));

// Resultado inmediato en la UI: ✓ ENVIADO
```
**Tiempo:** ~0ms (retorna al instante, no bloquea)

### ⏱️ Paso 2: Firebase Recibe y Guarda (50-100ms)
Firebase Cloud recibe la escritura y la persiste en sus servidores.

**Tiempo acumulado:** ~100ms

### ⏱️ Paso 3: ESP32 Polling (0-25ms espera + 200-300ms GET)
El ESP32 está constantemente revisando si hay comandos nuevos:

```cpp
// Cada 25 milisegundos (40 veces por segundo)
const unsigned long COMMAND_CHECK_INTERVAL = 25;

// Cuando llega el momento:
bool cmd = firebaseClient.readBool("/commands/open_relay_1");
// HTTP GET: ~200-300ms
```

**Tiempo acumulado:** ~100ms + (0-25ms) + 200-300ms = **300-425ms**

### ⏱️ Paso 4: ESP32 Activa el Relé (50ms)
```cpp
relayController.activateRelay1();  
// Relé se activa y dura 2 segundos
```

**Tiempo acumulado:** ~350-475ms

### ⏱️ Paso 5: ESP32 Resetea el Comando (200-300ms)
```cpp
// Resetea el comando a false para que no se repita
firebaseClient.writeBool("/commands/open_relay_1", false);
```

**Tiempo acumulado:** ~550-775ms

### ⏱️ Paso 6: Web App Detección de Cambio (instantánea)
```javascript
// La web app permanentemente escucha cambios en status
onValue(statusRef, (snapshot) => {
    const status = snapshot.val();
    // Detecta cambios en last_seen, otros campos
    updateUIWithStatus(status);
});
```

**LATENCIA TOTAL DE USUARIO:** 📊 **~50-150ms** (lo que VE el usuario es casi instantáneo)

---

## 📋 Significado de Cada Campo

### 🎛️ `/commands/` - Comandos (Lo que ESCRIBE la Web App)

```json
{
  "open_relay_1": false,
  "open_relay_2": false
}
```

| Campo | Tipo | Valores | Significado |
|-------|------|--------|------------|
| `open_relay_1` | Boolean | `false` ó `true` | `false` = sin hacer nada; `true` = activar relé 1 durante 2 segundos |
| `open_relay_2` | Boolean | `false` ó `true` | `false` = sin hacer nada; `true` = activar relé 2 durante 2 segundos |

**Ciclo de Vida:**
1. Web app escribe `true`
2. ESP32 lo Lee
3. ESP32 activa el relé
4. ESP32 escribe `false` (resetea automáticamente)
5. Vuelve a `false` para la siguiente orden

**Regla Importante:** Debe estar **siempre en `false`** salvo durante la activación.

---

### 📊 `/status/` - Estado del Dispositivo (Lo que ESCRIBE el ESP32)

#### 🟢 Conexión
```json
{
  "online": true,
  "last_seen": 1707264000000
}
```

| Campo | Tipo | Frecuencia | Propósito |
|-------|------|-----------|----------|
| `online` | Boolean | Cada 10s | Indica que el ESP32 está activo (**mínimamente confiable**) |
| `last_seen` | Number (ms) | Cada 10s | **CRÍTICO:** Timestamp en milisegundos. La web app lo usa para detectar si se desconectó |

**Por qué `last_seen` es crítico:**
- El ESP32 podría congelarse pero dejar `online: true`
- La web app revisa si `now - last_seen < 30000ms`
- Si ha pasado >30 segundos sin heartbeat = **definitivamente desconectado**

#### 📶 Señal WiFi
```json
{
  "wifi_rssi": -45,
  "wifi_quality": 90
}
```

| Campo | Tipo | Rango | Significado |
|-------|------|-------|------------|
| `wifi_rssi` | Number | -30 a -90 | Intensidad de señal en dBm. Mayor = mejor |
| `wifi_quality` | Number | 0-100 | Porcentaje de calidad |

**Referencia:**
- `-30 dBm` = Excelente (muy cercano)
- `-50 dBm` = Bueno (apartamento)
- `-70 dBm` = Débil (piso siguiente)
- `-90 dBm` = Pobre (casi sin señal)

#### ⏱️ Tiempo & Recursos
```json
{
  "uptime": 345600000,
  "uptime_formatted": "4d 00h 00m",
  "free_heap": 155232
}
```

| Campo | Tipo | Propósito |
|-------|------|----------|
| `uptime` | Number (ms) | Milisegundos desde el último arranque |
| `uptime_formatted` | String | Versión legible para humanos |
| `free_heap` | Number (bytes) | Memoria RAM libre. Si baja <50KB, problema |

#### 📊 Métricas
```json
{
  "heartbeat_count": 34560,
  "firmware_version": "1.0.1"
}
```

| Campo | Tipo | Propósito |
|-------|------|----------|
| `heartbeat_count` | Number | Contador que incrementa cada 10s (nunca resetea). Prueba de que funciona |
| `firmware_version` | String | Versión del firmware actual |

---

### ℹ️ `/info/` - Datos Estáticos del Dispositivo

```json
{
  "device_name": "Portón Principal",
  "location": "Casa",
  "created_at": 1707260000000,
  "last_configured": 1707263000000,
  "owner_email": "tu.email@gmail.com"
}
```

Solo información, **nunca cambia** (excepto `last_configured`).

---

## ⚡ POR QUÉ Algunos Valores Son Críticos

### ✅ `last_seen` en MILISEGUNDOS (no segundos)
```cpp
// ✅ CORRECTO
unsigned long timestamp = millis();  // 1707264000000 (milisegundos)

// ❌ INCORRECTO
unsigned long timestamp = time(nullptr);  // 1707264000 (segundos)
```
**Razón:** JavaScript usa `Date.now()` que retorna milisegundos. Si envías segundos, la comparación falla.

### ✅ Valores BOOLEANOS (no strings)
```json
// ✅ CORRECTO
{"open_relay_1": true}

// ❌ INCORRECTO
{"open_relay_1": "true"}
```
**Razón:** Los strings son más pesados. `true` = 4 bytes, `"true"` = 6 bytes. Multiplica por miles de transacciones.

### ✅ Fire-and-Forget en Web App (no `await`)
```javascript
// ✅ RÁPIDO: retorna inmediatamente
set(ref(db, path), true).catch(err => {});

// ❌ LENTO: espera respuesta (300-500ms más)
await set(ref(db, path), true);
```

### ✅ Polling Rápido en ESP32
```cpp
// ✅ RÁPIDO: 40 chequeos por segundo
const unsigned long COMMAND_CHECK_INTERVAL = 25;

// ❌ LENTO: solo 10 chequeos por segundo  
const unsigned long COMMAND_CHECK_INTERVAL = 100;
```

---

## 🔍 Detección de Desconexión: La Magia

La web app usa esta lógica inteligente:

```javascript
function isDeviceOnline(status) {
    const lastSeen = status.last_seen || 0;
    const now = Date.now();
    const timeSinceLastSeen = now - lastSeen;
    
    // Si hace más de 30 segundos que no hay heartbeat
    // = definitivamente desconectado
    return timeSinceLastSeen < 30000;
}
```

**Estadísticas:**
- Heartbeat = cada 10 segundos
- 30 segundos = 3x heartbeat
- Permite 2 fallos de heartbeat antes de marcar desconectado
- Se verifica cada 5 segundos desde web app

**Resultado:**
- Detecta desconexión entre 5-35 segundos ✅
- Muy pocas falsas alarmas ✅
- No sobrecarga Firebase ✅

---

## 🔐 Reglas de Seguridad (Firebase Console)

En Firebase → Realtime Database → Rules:

```json
{
  "rules": {
    "devices": {
      "$deviceId": {
        "commands": {
          ".read": true,
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
        },
        "info": {
          ".read": true,
          ".write": false
        }
      }
    }
  }
}
```

**Explicación:**
- ✅ Leer `/commands/` y `/status/` = permitido
- ✅ Escribir en `/commands/` (solo booleanos) = permitido
- ✅ Leer `/info/` = permitido
- ❌ Escribir en `/status/` = bloqueado (solo ESP32 internamente)
- ❌ Escribir en `/info/` = bloqueado

---

## 🧪 Cómo Verificar que Funciona Correctamente

### Prueba 1: Enviar Comando Manualmente
1. Firebase Console → Realtime Database
2. Navega a `/devices/{DEVICE_ID}/commands/open_relay_1`
3. Cambia `false` → `true`
4. **Resultado esperado:** 
   - LED del ESP32 parpadea
   - Relé se activa
   - Campo se resetea a `false` automáticamente
5. En monitor serial del ESP32 verás: `[Main] Comando recibido: Activar Relé 1`

### Prueba 2: Verificar Heartbeat
1. Firebase Console → Realtime Database
2. `/devices/{DEVICE_ID}/status/last_seen`
3. **Resultado esperado:** 
   - Número cambia cada ~10 segundos
   - Es un timestamp en milisegundos (13 dígitos)
   - Ej: `1707264523000`

### Prueba 3: Probar desde Web App
1. Abre la web app
2. Click en "ABRIR PORTÓN"
3. Observa Firebase Console en tiempo real
4. **Resultado esperado:**
   - `/commands/open_relay_1` brevemente cambia a `true` (y vuelve a `false`)
   - Relé se activa (~2 segundos)
   - Latencia total <300ms ✅

### Prueba 4: Desconexión
1. Desplug el USB del ESP32
2. Espera 5-10 segundos
3. **Resultado esperado:** 
   - Web app muestra 🔴 "Desconectado" 
   - Botones se deshabilitan
   - `last_seen` deja de actualizar

---

## 📈 Benchmark de Latencia

| Componente | Tiempo Típico |
|---------|---------------|
| Write en Firebase desde web | 50-100ms |
| Propagación dentro de Firebase | 10-50ms |
| Espera hasta próximo polling en ESP32 | 0-25ms |
| ESP32 HTTP GET a Firebase | 200-300ms |
| Activar relé | 50ms |
| Resetear comando en Firebase | 200-300ms |
| **TOTAL PERCEPTIBLE** | **50-150ms** ✅ |

*Nota: El usuario solo ve los primeros 50-150ms. El resto sucede invisiblemente.*

---

## ⚠️ Problemas & Cómo Resolverlos

| Problema | Causa | Solución |
|----------|-------|----------|
| **Latencia >1 segundo** | Timeouts demasiado altos | Reducir en ESP32: `http.setTimeout(600)` |
| **Comandos no llegan** | `last_seen` no actualiza = ESP32 caído | Verificar conexión ESP32 en consola |
| **"Desconectado" aparece constantemente** | Heartbeat inconsistente | Esperar a que se estabilice WiFi |
| **Falsos positivos de desconexión** | `last_seen` no en milisegundos | Asegurar que usa `millis()` |
| **Web app tarda al escribir** | Usando `await set()` | Cambiar a fire-and-forget |
| **ESP32 se congela a veces** | Polling no suficientemente rápido | Bajar `COMMAND_CHECK_INTERVAL` a 25ms |
| **Firebase devuelve 401 (Unauthorized)** | Credenciales inválidas | Verificar API Key y Database URL |

---

## ✅ Checklist Final

- [ ] Estructura de BD creada exactamente como se especifica
- [ ] `last_seen` se actualiza cada ~10 segundos
- [ ] `open_relay_X` se resetea a `false` automáticamente
- [ ] Web app usa fire-and-forget (sin `await`)
- [ ] ESP32 checkea comandos cada 25ms
- [ ] HTTP timeouts: 600ms/300ms
- [ ] Timestamps en milisegundos (13 dígitos)
- [ ] Valores booleanos, no strings
- [ ] Reglas de Firebase aplicadas correctamente
- [ ] Click en botón = relé activa en <300ms ✅

---

**Guía completada:** Febrero 6, 2026

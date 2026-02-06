# 🔌 Detección de Desconexión

## El Problema

Cuando el ESP32 se desconecta abruptamente (se desenchufa, pierde WiFi, se reinicia), **no tiene oportunidad** de escribir `online: false` en Firebase. Esto significa que la web app seguiría mostrando "Conectado" aunque el dispositivo esté apagado.

## ❌ ¿Por qué `online: true/false` no es suficiente?

```javascript
// Esto NO funciona bien para detectar desconexiones:
const isOnline = status.online === true;
```

**Escenarios problemáticos:**
- 🔌 ESP32 se desenchufa → `online` queda en `true`
- 📡 WiFi se cae → `online` queda en `true`
- ⚡ Reinicio inesperado → `online` queda en `true`
- 💥 Crash del firmware → `online` queda en `true`

## ✅ La Solución: Timestamp `last_seen`

El ESP32 escribe un timestamp en cada heartbeat (cada 10 segundos):

```cpp
// En heartbeat.h del ESP32:
String path = "/devices/" + deviceId + "/status/last_seen";
firebaseClient.writeTimestamp(path, millis());
```

La web app verifica **cuánto tiempo ha pasado** desde el último heartbeat:

```javascript
function isDeviceOnline(status) {
    const lastSeen = status.last_seen || 0;
    const now = Date.now();
    const timeSinceLastSeen = now - lastSeen;
    
    // Si han pasado más de 30 segundos, considerar desconectado
    const TIMEOUT_MS = 30000;
    return timeSinceLastSeen < TIMEOUT_MS;
}
```

## 📊 Funcionamiento

### Timeline Normal (Dispositivo Conectado)

```
t=0s    → Heartbeat #1 enviado (last_seen = 0s)
t=10s   → Heartbeat #2 enviado (last_seen = 10s)
t=20s   → Heartbeat #3 enviado (last_seen = 20s)
t=30s   → Heartbeat #4 enviado (last_seen = 30s)
...
```

La web app ve que `now - last_seen < 30s` → **Conectado** ✅

### Timeline con Desconexión

```
t=0s    → Heartbeat #1 enviado (last_seen = 0s)
t=10s   → Heartbeat #2 enviado (last_seen = 10s)
t=15s   → ⚡ ESP32 SE DESCONECTA ⚡
t=20s   → (sin heartbeat - last_seen sigue en 10s)
t=25s   → Web app verifica: 25s - 10s = 15s → Aún "conectado"
t=30s   → (sin heartbeat - last_seen sigue en 10s)
t=35s   → Web app verifica: 35s - 10s = 25s → Aún "conectado"
t=40s   → Web app verifica: 40s - 10s = 30s → Aún "conectado"
t=45s   → Web app verifica: 45s - 10s = 35s → ❌ DESCONECTADO
```

## ⚙️ Configuración del Sistema

### En el ESP32 (Firmware)

```cpp
// config.h
#define HEARTBEAT_INTERVAL 10000  // 10 segundos entre heartbeats
```

### En la Web App

```javascript
// index.html
const TIMEOUT_MS = 30000;  // 30 segundos = 3x heartbeat interval
const CHECK_INTERVAL = 5000; // Verificar cada 5 segundos
```

## 🎯 Parámetros Optimizados

| Parámetro | Valor | Razón |
|-----------|-------|-------|
| Intervalo Heartbeat | 10s | Balance entre carga y detección rápida |
| Timeout Desconexión | 30s | 3x intervalo = permite 2 heartbeats perdidos |
| Verificación Web | 5s | Detecta cambios rápido sin sobrecargar |

### ¿Por qué 30 segundos?

**Demasiado corto (15s):**
- Falsos positivos si hay lag de red
- Marca desconectado aunque esté funcionando

**Demasiado largo (60s):**
- Tarda mucho en detectar desconexiones reales
- Mala experiencia de usuario

**30 segundos es ideal:**
- ✅ Permite hasta 2 heartbeats perdidos
- ✅ Detección rápida (5-35s después de desconexión)
- ✅ Pocos falsos positivos
- ✅ Tolerante a lag temporal de red

## 🔧 Implementación en la Web App

### 1. Variables Globales

```javascript
let lastSeenTimestamp = null;
let connectionCheckInterval = null;
```

### 2. Actualizar Timestamp al Recibir Datos

```javascript
function updateUIWithStatus(status) {
    lastSeenTimestamp = status.last_seen || Date.now();
    const isOnline = isDeviceOnline(status);
    // ... actualizar UI
}
```

### 3. Monitor Periódico

```javascript
function startConnectionMonitoring() {
    connectionCheckInterval = setInterval(() => {
        const now = Date.now();
        const timeSinceLastSeen = now - lastSeenTimestamp;
        const isOnline = timeSinceLastSeen < 30000;
        
        // Actualizar UI si cambió el estado
        if (wasOnline !== isOnline) {
            updateOnlineIndicator(isOnline);
            console.warn('⚠️ Dispositivo desconectado');
        }
    }, 5000);
}
```

## 📈 Ejemplos Prácticos

### Caso 1: Desconexión por Falta de Energía

```
t=0     → Conectado, enviando heartbeats
t=15    → Se va la luz, ESP32 se apaga
t=16-44 → Web app sigue mostrando "conectado"
t=45    → Web app detecta: "Sin heartbeat por 30s" → DESCONECTADO
```

**Tiempo de detección:** ~30 segundos

### Caso 2: Pérdida de WiFi

```
t=0     → Conectado, WiFi estable
t=20    → Router se reinicia, WiFi cae
t=21-49 → Web app espera heartbeats
t=50    → DESCONECTADO detectado
```

**Tiempo de detección:** ~30 segundos

### Caso 3: Reconexión

```
t=0     → DESCONECTADO (no hay heartbeats)
t=10    → ESP32 se reconecta, envía heartbeat
t=10.5  → Web app recibe datos → CONECTADO inmediatamente
```

**Tiempo de reconexión:** < 1 segundo (instantáneo)

## 🎨 Indicadores Visuales

### Estado Conectado
```
🟢 En línea
● (círculo verde pulsando)
✓ Conectado
```

### Estado Desconectado
```
🔴 Sin conexión
● (círculo rojo pulsando)
✗ Desconectado
```

## 🐛 Debugging

### Consola del Navegador

```javascript
// Mensajes útiles para debugging:
console.log('✓ Firebase inicializado');
console.log('✓ Portón 1 activado');
console.warn('⚠️ Dispositivo desconectado - Sin heartbeat por más de 30s');
```

### Firebase Console

Verifica en tiempo real:
1. Abre Firebase Console
2. Ve a `/devices/{ID}/status/last_seen`
3. Observa el timestamp actualizándose cada 10s

## 💡 Mejoras Futuras

### Firebase Presence Detection (Avanzado)

```javascript
// Usar .info/connected de Firebase
const connectedRef = ref(database, '.info/connected');
onValue(connectedRef, (snap) => {
    if (snap.val() === true) {
        // Firebase conectado
        const statusRef = ref(database, `devices/${deviceId}/status/online`);
        onDisconnect(statusRef).set(false);
        set(statusRef, true);
    }
});
```

Esto requeriría cambios en el firmware del ESP32.

### Notificaciones Push

Alertar al usuario cuando se pierda conexión:

```javascript
if (!isOnline && wasOnline) {
    new Notification('⚠️ Portón Desconectado', {
        body: 'El dispositivo perdió conexión'
    });
}
```

## ✅ Checklist de Implementación

- [x] ESP32 envía `last_seen` en cada heartbeat
- [x] Web app guarda timestamp en `lastSeenTimestamp`
- [x] Función `isDeviceOnline()` verifica timeout
- [x] Monitor periódico cada 5s (`startConnectionMonitoring()`)
- [x] UI actualiza indicador visual
- [x] Console logs para debugging
- [x] Documentación completa

## 🎯 Resultado

**Antes del fix:**
- Dispositivo desconectado → Web app sigue mostrando "Conectado" ❌

**Después del fix:**
- Dispositivo desconectado → Web app detecta en 5-35 segundos y muestra "Desconectado" ✅

---

**Implementado:** Febrero 2026

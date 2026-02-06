# 🚪 Control de Portones - Web App

Aplicación web para controlar portones eléctricos de forma remota mediante ESP32 y Firebase.

## 🌐 Demo en Vivo

Una vez subida a GitHub Pages, tu app estará disponible en:
```
https://tu-usuario.github.io/nombre-repo/
```

## ✨ Características

- ✅ **Auto-conecta** al dispositivo ESP32 automáticamente
- ✅ **Tiempo real** - Estado actualizado constantemente
- ✅ **Responsive** - Funciona en PC, tablet y celular
- ✅ **Control de 2 portones** independientes
- ✅ **Dashboard con métricas**:
  - Estado online/offline
  - Señal WiFi (RSSI con barras visuales)
  - Tiempo de actividad (uptime)
  - Memoria disponible
  - Contador de latidos
  - Versión del firmware

## 🚀 Instalación en GitHub Pages

### Paso 1: Crear Repositorio

1. Ve a [github.com/new](https://github.com/new)
2. Nombre: `gate-control` (o tu preferido)
3. Visibilidad: Public
4. Presiona "Create repository"

### Paso 2: Subir Archivos

En PowerShell (desde esta carpeta):

```powershell
git init
git add .
git commit -m "Web app para control de portones"
git branch -M main
git remote add origin https://github.com/TU-USUARIO/gate-control.git
git push -u origin main
```

### Paso 3: Activar GitHub Pages

1. En GitHub: Settings → Pages
2. Source: "Deploy from a branch"
3. Branch: "main"
4. Save

¡Listo! En 1-2 minutos estará disponible.

## ⚙️ Configuración

### Credenciales de Firebase

Las credenciales ya están configuradas en `index.html` (línea 279):

```javascript
const firebaseConfig = {
    apiKey: "AIzaSyAfpyKv1k0dNcOT4XiC-0ZKC75rngM8eCE",
    databaseURL: "https://san-industries-default-rtdb.firebaseio.com"
};
```

Si necesitas cambiarlas, edita estas líneas antes de subir a GitHub.

## 📱 Uso

1. Abre la URL de tu GitHub Pages
2. La app se conecta automáticamente
3. Espera a ver el Device ID y estado
4. Presiona los botones para activar los portones

## 🏗️ Estructura de Firebase

La app espera esta estructura en Firebase:

```
devices/
  └─ ESP32-XXXXXXXXXXXX/
      ├─ commands/
      │  ├─ open_relay_1: boolean
      │  └─ open_relay_2: boolean
      └─ status/
          ├─ online: boolean
          ├─ wifi_rssi: number
          ├─ uptime_formatted: string
          ├─ free_heap: number
          ├─ heartbeat_count: number
          └─ firmware_version: string
```

## 🔐 Seguridad

### Reglas de Firebase Recomendadas

En Firebase Console → Database → Rules:

```json
{
  "rules": {
    "devices": {
      "$deviceId": {
        ".read": true,
        ".write": false,
        "commands": {
          ".write": true,
          "open_relay_1": { ".validate": "newData.isBoolean()" },
          "open_relay_2": { ".validate": "newData.isBoolean()" }
        },
        "status": {
          ".read": true,
          ".write": false
        }
      }
    }
  }
}
```

Esto permite:
- ✅ Leer el estado del dispositivo
- ✅ Escribir comandos a los relés
- ❌ No permite modificar configuración
- ❌ No permite escribir en status (solo ESP32)

## 📂 Archivos Incluidos

- `index.html` - Aplicación web completa
- `README.md` - Este archivo
- `QUICKSTART.md` - Guía rápida en 2 minutos
- `FIREBASE_STRUCTURE.md` - Estructura de la base de datos

## 🛠️ Requisitos

- Navegador moderno (Chrome, Firefox, Safari, Edge)
- Conexión a Internet
- Firebase Realtime Database configurado
- ESP32 con firmware funcionando

## 🐛 Solución de Problemas

### "No hay dispositivos configurados"
- Verifica que el ESP32 esté enviando datos a Firebase
- Abre Firebase Console y confirma que existe `/devices/{id}/`

### "Desconectado" siempre
- Revisa las credenciales en `index.html`
- Confirma que el ESP32 tiene WiFi
- Verifica que `online: true` esté en Firebase

### Los botones no funcionan
- Revisa las reglas de Firebase
- Abre la consola del navegador (F12) para ver errores
- Confirma que tienes permisos de escritura en `/commands/`

## 💡 Tips

- **Agregar a inicio**: En móvil, usa "Agregar a pantalla de inicio"
- **Bookmark**: Guarda la URL para acceso rápido
- **Compartir**: Comparte la URL con otros usuarios
- **Actualizar**: Recarga la página (F5) si hay problemas

## 📞 Soporte

Los errores aparecen en:
1. Consola del navegador (F12 → Console)
2. Box de error rojo en la app
3. Firebase Console → Database → Rules Simulator

## 📄 Licencia

Código de ejemplo - Libre para usar y modificar

---

**Creado para San Industries** 🏭

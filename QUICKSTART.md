# ⚡ Quick Start - 2 Minutos

## 1️⃣ Crear Repositorio en GitHub

Ve a: [github.com/new](https://github.com/new)

- **Name**: `gate-control`
- **Public**: ✅
- **Create repository**

## 2️⃣ Subir con PowerShell

Abre PowerShell **en esta carpeta** y ejecuta:

```powershell
git init
git add .
git commit -m "Web app control de portones"
git branch -M main
git remote add origin https://github.com/TU-USUARIO/gate-control.git
git push -u origin main
```

⚠️ **Reemplaza `TU-USUARIO`** con tu usuario de GitHub

## 3️⃣ Activar GitHub Pages

En tu repositorio de GitHub:

1. **Settings** (⚙️)
2. **Pages** (menú lateral)
3. **Source**: Deploy from a branch
4. **Branch**: main
5. **Save** 💾

## ✅ ¡Listo!

Tu app estará en: `https://tu-usuario.github.io/gate-control/`

Espera 1-2 minutos para que se active.

## 🎯 Ejemplo Completo

Si tu usuario es `johnsmith`:

```powershell
git remote add origin https://github.com/johnsmith/gate-control.git
git push -u origin main
```

URL final: `https://johnsmith.github.io/gate-control/`

## 🔗 Compartir

Envía la URL a quien quieras que controle los portones.

---

**¡Ya está!** Abre la URL en cualquier dispositivo 📱💻

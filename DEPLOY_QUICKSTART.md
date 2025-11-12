# 🚀 Despliegue Rápido / Quick Deploy

## ❌ Error Actual / Current Error

```
ImportError: libGL.so.1: cannot open shared object file: No such file or directory
```

## ✅ Solución Inmediata / Immediate Solution

### Opción 1: Reconstruir con nuevo Dockerfile (Recomendado)

```bash
# 1. Detener el contenedor actual
docker-compose down

# 2. Reconstruir (esto instalará las dependencias del sistema)
docker-compose build --no-cache

# 3. Iniciar de nuevo
docker-compose up -d

# 4. Ver logs
docker-compose logs -f
```

### Opción 2: Solo cambiar requirements.txt

Si ya tienes el contenedor corriendo:

```bash
# 1. Detener
docker stop identificaciondeplantas
docker rm identificaciondeplantas

# 2. Asegurarte que requirements.txt use opencv-python-headless
# Ya está actualizado en el archivo!

# 3. Reconstruir imagen
docker build -t plantid-api .

# 4. Ejecutar
docker run -d \
  --name identificaciondeplantas \
  -p 8000:8000 \
  -v $(pwd)/plantid/models:/app/plantid/models:ro \
  --restart unless-stopped \
  plantid-api

# 5. Verificar
docker logs -f identificaciondeplantas
```

---

## 📋 Archivos Creados

| Archivo | Propósito |
|---------|-----------|
| `Dockerfile` | Configuración Docker con dependencias del sistema |
| `docker-compose.yml` | Orquestación fácil |
| `.dockerignore` | Optimizar build |
| `requirements-server.txt` | Dependencias alternativas para servidor |
| `DEPLOYMENT.md` | Guía completa de despliegue |

---

## 🎯 Comandos Esenciales

```bash
# Ver si está corriendo
docker ps

# Ver logs
docker logs -f identificaciondeplantas

# Reiniciar
docker restart identificaciondeplantas

# Ver recursos usados
docker stats identificaciondeplantas

# Ejecutar comando dentro del contenedor
docker exec -it identificaciondeplantas bash

# Verificar que funciona
curl http://localhost:8000/health
```

---

## 🔍 ¿Por qué ocurre este error?

OpenCV (`opencv-python`) requiere librerías gráficas del sistema:
- `libGL.so.1` - OpenGL
- `libglib2.0` - GLIB
- Y otras...

Las imágenes Docker mínimas (como `python:3.10-slim`) NO incluyen estas librerías.

**Soluciones:**
1. ✅ Usar `opencv-python-headless` (sin GUI)
2. ✅ Instalar dependencias del sistema en Dockerfile
3. ❌ Usar imagen base completa (muy grande)

---

## 📦 Diferencia entre versiones OpenCV

| Paquete | Tamaño | GUI | Servidor |
|---------|--------|-----|----------|
| `opencv-python` | ~60MB | ✅ | ❌ Necesita deps |
| `opencv-python-headless` | ~30MB | ❌ | ✅ Perfecto |

Para tu API, `opencv-python-headless` es **perfecto** porque no necesitas ventanas gráficas.

---

## ✅ Verificación

Después de desplegar, verifica:

```bash
# 1. Health check
curl http://localhost:8000/health

# 2. Probar con una imagen
curl -X POST "http://localhost:8000/identify/quick" \
  -F "file=@test_image.jpg"
```

Respuesta esperada:

```json
{
  "status": 0,
  "message": "True",
  "inference_time": 0.1234,
  "chinese_name": "一串红",
  "latin_name": "Salvia splendens",
  "probability": 0.9523
}
```

---

## 🐛 Si aún no funciona

```bash
# 1. Ver logs completos
docker logs identificaciondeplantas 2>&1 | less

# 2. Verificar que el modelo existe
docker exec identificaciondeplantas ls -lh /app/plantid/models/

# 3. Probar manualmente dentro del contenedor
docker exec -it identificaciondeplantas python -c "import cv2; print(cv2.__version__)"

# 4. Verificar memoria
docker stats identificaciondeplantas
```

---

**¡Listo! Tu API debería funcionar ahora! 🎉**


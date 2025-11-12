# 🚀 Guía de Despliegue / Deployment Guide

## 🐳 Solución Rápida con Docker (Recomendado)

### Método 1: Docker Compose (Más Fácil)

```bash
# 1. Clonar o subir el proyecto al servidor
cd /path/to/quarrying-plant-id

# 2. Construir y ejecutar con un solo comando
docker-compose up -d

# 3. Ver logs
docker-compose logs -f

# 4. Detener el servicio
docker-compose down
```

### Método 2: Docker Manual

```bash
# 1. Construir la imagen
docker build -t plantid-api .

# 2. Ejecutar el contenedor
docker run -d \
  --name identificaciondeplantas \
  -p 8000:8000 \
  -v $(pwd)/plantid/models:/app/plantid/models:ro \
  plantid-api

# 3. Ver logs
docker logs -f identificaciondeplantas

# 4. Detener el contenedor
docker stop identificaciondeplantas
docker rm identificaciondeplantas
```

---

## ❌ Error: libGL.so.1 no encontrado

### Causa
OpenCV necesita librerías gráficas del sistema que no están disponibles en contenedores Docker mínimos.

### ✅ Solución 1: Usar opencv-python-headless (Recomendado)

Cambiar en `requirements.txt`:

```diff
- opencv-python>=4.4
+ opencv-python-headless>=4.4
```

O usar el archivo especial para servidores:

```bash
pip install -r requirements-server.txt
```

### ✅ Solución 2: Instalar dependencias del sistema

En el servidor Ubuntu/Debian:

```bash
apt-get update
apt-get install -y \
    libgl1-mesa-glx \
    libglib2.0-0 \
    libsm6 \
    libxext6 \
    libxrender-dev \
    libgomp1
```

En Alpine Linux:

```bash
apk add --no-cache \
    libstdc++ \
    libgomp \
    libx11 \
    libxext \
    mesa-gl
```

---

## 📦 Despliegue sin Docker

### Opción A: Systemd (Linux)

1. **Crear archivo de servicio:**

```bash
sudo nano /etc/systemd/system/plantid.service
```

2. **Contenido del archivo:**

```ini
[Unit]
Description=Plant Identification API
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/plantid
Environment="PATH=/opt/plantid/venv/bin"
ExecStart=/opt/plantid/venv/bin/uvicorn app:app --host 0.0.0.0 --port 8000 --workers 4
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

3. **Activar y ejecutar:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable plantid
sudo systemctl start plantid
sudo systemctl status plantid
```

### Opción B: Con Gunicorn + Nginx

1. **Instalar dependencias:**

```bash
pip install gunicorn
```

2. **Ejecutar con Gunicorn:**

```bash
gunicorn app:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --timeout 120 \
  --access-logfile - \
  --error-logfile -
```

3. **Configurar Nginx como proxy reverso:**

```nginx
server {
    listen 80;
    server_name tu-dominio.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Aumentar límite de tamaño de archivo
        client_max_body_size 16M;
    }
}
```

---

## 🔒 Configuración de Producción

### 1. Variables de Entorno

Crear archivo `.env`:

```env
# Server
HOST=0.0.0.0
PORT=8000
WORKERS=4

# Security
ALLOWED_ORIGINS=https://tu-dominio.com,https://www.tu-dominio.com

# Performance
MAX_FILE_SIZE=16777216  # 16MB
```

### 2. Modificar app.py para producción

```python
import os
from dotenv import load_dotenv

load_dotenv()

# CORS para producción
allowed_origins = os.getenv("ALLOWED_ORIGINS", "*").split(",")
app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 3. Optimizaciones

**a) Usar múltiples workers:**

```bash
# Con Gunicorn
gunicorn app:app \
  --workers $(nproc) \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000
```

**b) Habilitar cache de respuestas:**

```bash
pip install fastapi-cache2
```

**c) Usar GPU si está disponible:**

```bash
pip uninstall onnxruntime
pip install onnxruntime-gpu
```

---

## 📊 Monitoreo

### Healthcheck Endpoint

```bash
curl http://localhost:8000/health
```

Respuesta esperada:

```json
{
  "status": "healthy",
  "model_loaded": true,
  "supported_species": 4066,
  "supported_genus": 1234,
  "supported_family": 567
}
```

### Logs con Docker

```bash
# Ver logs en tiempo real
docker-compose logs -f

# Ver últimas 100 líneas
docker-compose logs --tail=100

# Filtrar por servicio
docker-compose logs plantid-api
```

---

## 🔧 Solución de Problemas

### Problema: Contenedor se reinicia constantemente

```bash
# Ver logs
docker logs identificaciondeplantas

# Ver eventos
docker events

# Verificar recursos
docker stats identificaciondeplantas
```

### Problema: Modelo no se carga

```bash
# Verificar que el modelo existe
ls -lh plantid/models/

# Verificar permisos
chmod -R 755 plantid/models/
```

### Problema: Timeout en requests

Aumentar timeout en Nginx:

```nginx
proxy_connect_timeout 300s;
proxy_send_timeout 300s;
proxy_read_timeout 300s;
```

---

## 🌐 Despliegue en Cloud

### AWS EC2

```bash
# 1. Instalar Docker
sudo yum update -y
sudo yum install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user

# 2. Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 3. Clonar y ejecutar
git clone <tu-repo>
cd quarrying-plant-id
docker-compose up -d
```

### Google Cloud Run

```bash
# 1. Build y push a Container Registry
gcloud builds submit --tag gcr.io/tu-proyecto/plantid-api

# 2. Deploy
gcloud run deploy plantid-api \
  --image gcr.io/tu-proyecto/plantid-api \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 2Gi \
  --timeout 300
```

### DigitalOcean App Platform

Crear archivo `app.yaml`:

```yaml
name: plantid-api
services:
- name: api
  dockerfile_path: Dockerfile
  source_dir: /
  github:
    repo: tu-usuario/quarrying-plant-id
    branch: master
  http_port: 8000
  instance_count: 1
  instance_size_slug: basic-xs
  routes:
  - path: /
```

---

## 📝 Checklist de Despliegue

- [ ] Modelo ONNX presente en `plantid/models/`
- [ ] Usar `opencv-python-headless` en servidores
- [ ] Configurar CORS correctamente
- [ ] Establecer límite de tamaño de archivo
- [ ] Configurar múltiples workers
- [ ] Implementar HTTPS (Let's Encrypt)
- [ ] Configurar healthcheck
- [ ] Configurar logs
- [ ] Establecer auto-restart
- [ ] Monitoreo de recursos

---

## 🎯 Comandos Útiles

```bash
# Docker Compose
docker-compose up -d          # Iniciar
docker-compose down           # Detener
docker-compose restart        # Reiniciar
docker-compose logs -f        # Ver logs
docker-compose ps             # Ver estado

# Docker
docker ps                     # Contenedores activos
docker images                 # Imágenes
docker system prune -a        # Limpiar todo

# Systemd
sudo systemctl start plantid
sudo systemctl stop plantid
sudo systemctl restart plantid
sudo systemctl status plantid
sudo journalctl -u plantid -f
```

---

**¡Tu API está lista para producción! 🚀**


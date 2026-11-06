# MonitoreoIndustrial

API REST desarrollada con **FastAPI** y **MongoDB** para gestionar lecturas de sensores industriales (**temperatura**, **presión** y **vibración**), desplegable en arquitectura **monolítica** y **escalable con Nginx** mediante **Docker Compose**.

**Asignatura:** Escalamiento de Aplicaciones — IU Digital de Antioquia

---

## 1. Tecnologías utilizadas

- **Python 3.11+**, **FastAPI**, **Uvicorn**, **Pydantic**
- **MongoDB 8** y **PyMongo**
- **Docker**, **Docker Compose** y **Docker Hub**
- **Nginx** (Balanceador de cargas *Round-Robin*)

---

## 2. Estructura del proyecto

```text
MonitoreoIndustrial/
├── app/
│   ├── __init__.py
│   ├── main.py                     # Endpoints REST y validaciones Pydantic
│   └── database.py                 # Conexión a MongoDB y conversión de ObjectId
├── docker/
│   └── nginx.conf                  # Configuración del balanceador de cargas Nginx
├── screenshots/                    # Capturas de evidencia
├── requirements.txt                # Dependencias de Python
├── Dockerfile                      # Imagen Docker de la API
├── .dockerignore
├── .env.example
├── docker-compose.monolitico.yml   # Arquitectura monolítica (API + MongoDB)
├── docker-compose.escalable.yml    # Arquitectura escalable (Nginx + 2 APIs + MongoDB)
└── README.md
```

---

## 3. Instalación y ejecución local

1. **Crear y activar el entorno virtual:**
   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```
2. **Instalar dependencias:**
   ```powershell
   pip install -r requirements.txt
   ```
3. **Configurar variables de entorno:**
   ```powershell
   Copy-Item .env.example .env
   ```
4. **Iniciar la API (con MongoDB activo en el puerto `27017`):**
   ```powershell
   uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
   ```
   - **API:** `http://localhost:8000`
   - **Swagger UI:** `http://localhost:8000/docs`
   - **MongoDB Compass:** `mongodb://[::1]:27017/` (o `mongodb://localhost:27017/`)

---

## 4. Endpoints y validaciones

| Método | Endpoint | Descripción |
| :--- | :--- | :--- |
| `GET` | `/` | Información general de la API e instancia activa. |
| `GET` | `/salud` | Estado de la API y conexión con MongoDB (`ping`). |
| `POST` | `/sensores/` | Registra un nuevo sensor validando tipo, rango, ubicación y fecha. |
| `GET` | `/sensores/` | Consulta todos los sensores registrados. |
| `GET` | `/sensores/?tipo=presion` | Filtra los sensores por su tipo. |
| `PUT` | `/sensores/?tipo=presion` | Actualiza **todos** los registros que coincidan con ese tipo. |
| `DELETE` | `/sensores/?tipo=presion` | Elimina **todos** los registros que coincidan con ese tipo. |

> **Nota:** Las operaciones `PUT` y `DELETE` filtran por el parámetro `tipo` (`update_many` y `delete_many`), por lo que afectan a todos los sensores almacenados que coincidan con dicho tipo.

### Validaciones implementadas (Pydantic) y manejo de errores
- **Tipos permitidos:** `temperatura`, `presion`, `vibracion`.
- **Rangos válidos:**
  - `temperatura`: `-10` a `100` °C
  - `presion`: `0` a `500` kPa
  - `vibracion`: `0` a `5` mm/s
- **Campos obligatorios:** `ubicacion` (texto no vacío) y `fecha` (formato ISO 8601).
- **Respuestas de error:** `422` (datos o rangos inválidos), `404` (sin registros para el tipo indicado) y `503` (fallo de conexión con MongoDB). Los identificadores `_id` (`ObjectId`) se serializan automáticamente a texto en el campo `id`.

### Ejemplo JSON (`POST /sensores/` y `PUT /sensores/?tipo=temperatura`)
```json
{
  "tipo": "temperatura",
  "valor": 25.5,
  "ubicacion": "Planta 1 - Línea de Ensamble",
  "fecha": "2026-11-05T15:30:00Z"
}
```

---

## 5. Ejecución con Docker Compose

### A. Arquitectura Monolítica (`docker-compose.monolitico.yml`)
Levanta 1 contenedor para FastAPI (`puerto 8000`) y 1 contenedor para MongoDB (`puerto 27017`) con volumen persistente:
```powershell
docker compose -f docker-compose.monolitico.yml up -d --build
```
- **Probar API:** `http://localhost:8000/docs`
- **Detener:** `docker compose -f docker-compose.monolitico.yml down`

### B. Arquitectura Escalable con Balanceador Nginx (`docker-compose.escalable.yml`)
Levanta 1 contenedor MongoDB (`puerto 27018`), 2 instancias internas de FastAPI (`api_1` y `api_2`, sin puertos expuestos al host) y **Nginx** como balanceador de cargas en el `puerto 8080`:
```powershell
docker compose -f docker-compose.escalable.yml up -d --build
```
- **Probar API balanceada:** `http://localhost:8080/docs`
- **Verificar el balanceo Round-Robin en PowerShell:**
  ```powershell
  1..6 | ForEach-Object { (Invoke-RestMethod "http://localhost:8080/salud").instancia }
  ```
- **Detener:** `docker compose -f docker-compose.escalable.yml down`

---

## 6. Construcción y publicación en Docker Hub

Reemplaza `<TU_USUARIO>` por tu usuario de Docker Hub:

```powershell
# 1. Construir la imagen
docker build -t monitoreo-industrial-api:1.0.0 .

# 2. Iniciar sesión en Docker Hub
docker login

# 3. Etiquetar la imagen
docker tag monitoreo-industrial-api:1.0.0 <TU_USUARIO>/monitoreo-industrial-api:1.0.0
docker tag monitoreo-industrial-api:1.0.0 <TU_USUARIO>/monitoreo-industrial-api:latest

# 4. Publicar la imagen
docker push <TU_USUARIO>/monitoreo-industrial-api:1.0.0
docker push <TU_USUARIO>/monitoreo-industrial-api:latest
```
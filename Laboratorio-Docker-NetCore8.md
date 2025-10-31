# 🧪 Laboratorio: Docker y Docker Compose para .NET Core 8

## 📚 Objetivo del Laboratorio
Aprender a crear un Dockerfile optimizado y un Docker Compose para una Web API en .NET Core 8.

---

## 📂 Estructura del Proyecto

```
webapi/
├── webapi.sln
├── webapi.csproj
├── Program.cs
├── Controllers/
├── Dockerfile              ← Vamos a crear este
├── docker-compose.yml      ← Vamos a crear este
└── .dockerignore          ← Vamos a crear este
```

---

## 🎯 Parte 1: Crear el Dockerfile

### Paso 1: Crear el archivo `Dockerfile`

En la carpeta raíz del proyecto `webapi/`, crea un archivo llamado `Dockerfile` (sin extensión) con el siguiente contenido:

```dockerfile
# ============================================
# ETAPA 1: BUILD
# ============================================
# Usamos la imagen del SDK para compilar
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copiar el .csproj y restaurar dependencias
# Esto se hace primero para aprovechar la caché de Docker
COPY ["webapi.csproj", "./"]
RUN dotnet restore "webapi.csproj"

# Copiar todo el código fuente
COPY . .

# Compilar la aplicación en modo Release
RUN dotnet build "webapi.csproj" -c Release -o /app/build

# ============================================
# ETAPA 2: PUBLISH
# ============================================
# Publicar la aplicación optimizada
FROM build AS publish
RUN dotnet publish "webapi.csproj" -c Release -o /app/publish /p:UseAppHost=false

# ============================================
# ETAPA 3: RUNTIME (Imagen final)
# ============================================
# Usamos la imagen ligera de runtime (solo para ejecutar)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Copiar los archivos publicados desde la etapa anterior
COPY --from=publish /app/publish .

# Exponer el puerto 8080 (puerto por defecto en .NET 8)
EXPOSE 8080

# Comando para ejecutar la aplicación
ENTRYPOINT ["dotnet", "webapi.dll"]
```

### 📖 Explicación del Dockerfile:

| Sección | Descripción |
|---------|-------------|
| **ETAPA 1: BUILD** | Usa imagen SDK (600+ MB) para compilar el código |
| **ETAPA 2: PUBLISH** | Genera archivos optimizados para producción |
| **ETAPA 3: RUNTIME** | Usa imagen ligera (200 MB) solo con lo necesario para ejecutar |
| **Multi-Stage** | Resultado: imagen final pequeña y eficiente |

**¿Por qué 3 etapas?**
- Solo la imagen final se guarda
- No incluye herramientas de desarrollo
- Más segura y rápida

---

## 🎯 Parte 2: Crear el Docker Compose

### Paso 2: Crear el archivo `docker-compose.yml`

En la misma carpeta raíz `webapi/`, crea el archivo `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # ============================================
  # Servicio: Web API
  # ============================================
  webapi:
    # Configuración de build
    build:
      context: .              # Directorio actual
      dockerfile: Dockerfile  # Archivo Dockerfile a usar
    
    # Nombre del contenedor
    container_name: webapi-container
    
    # Mapeo de puertos (host:container)
    ports:
      - "5000:8080"          # http://localhost:5000
      - "5001:8081"          # https://localhost:5001 (opcional)
    
    # Variables de entorno
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:8080
      - ASPNETCORE_HTTP_PORTS=8080
    
    # Red para comunicación entre servicios
    networks:
      - webapi-network
    
    # Política de reinicio
    restart: unless-stopped
    
    # Health check (opcional)
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

# ============================================
# Redes
# ============================================
networks:
  webapi-network:
    driver: bridge
```

### 📖 Explicación del Docker Compose:

| Configuración | Descripción |
|--------------|-------------|
| **version** | Versión del formato de Docker Compose |
| **build** | Indica cómo construir la imagen |
| **ports** | Puerto 5000 (tu PC) → 8080 (contenedor) |
| **environment** | Variables de entorno para la app |
| **networks** | Red virtual para servicios |
| **restart** | Reinicia automáticamente si falla |
| **healthcheck** | Verifica que la app esté funcionando |

---

## 🎯 Parte 3: Optimizar con .dockerignore

### Paso 3: Crear el archivo `.dockerignore`

En la carpeta raíz `webapi/`, crea el archivo `.dockerignore`:

```
# ============================================
# Carpetas de compilación
# ============================================
bin/
obj/
out/
publish/

# ============================================
# Archivos de usuario y configuración
# ============================================
*.user
*.suo
*.userosscache
*.sln.docstates
.vs/
.vscode/

# ============================================
# Archivos de Git
# ============================================
.git/
.gitignore
.gitattributes

# ============================================
# Archivos de Docker
# ============================================
Dockerfile
docker-compose.yml
.dockerignore
*.md

# ============================================
# Archivos temporales y logs
# ============================================
*.log
*.tmp
*.cache
TestResults/

# ============================================
# Dependencias (se restauran en el build)
# ============================================
packages/
node_modules/
```

### 📖 ¿Para qué sirve .dockerignore?

- ✅ **Acelera** el proceso de build
- ✅ **Reduce** el tamaño del contexto de Docker
- ✅ **Evita** copiar archivos innecesarios
- ✅ **Mejora** la seguridad (no copia .git, etc.)

---

## 🚀 Parte 4: Comandos para Ejecutar

### Opción A: Usando Docker Compose (Recomendado)

```bash
# 1. Construir y ejecutar (primera vez)
docker-compose up --build

# 2. Ejecutar en segundo plano (detached)
docker-compose up -d

# 3. Ver logs en tiempo real
docker-compose logs -f webapi

# 4. Ver logs de los últimos 100 líneas
docker-compose logs --tail=100 webapi

# 5. Detener los servicios
docker-compose down

# 6. Detener y eliminar volúmenes
docker-compose down -v

# 7. Reconstruir sin caché (cuando hay cambios importantes)
docker-compose build --no-cache

# 8. Ver estado de los contenedores
docker-compose ps

# 9. Reiniciar el servicio
docker-compose restart webapi

# 10. Entrar al docker 
# Entrar al servicio webapi
docker-compose exec webapi bash
```

### Opción B: Usando Docker directamente

#### Paso 1: Crear la imagen

```bash
# Construir la imagen con un nombre y tag
docker build -t webapi:1.0 .

# Construir sin usar caché
docker build --no-cache -t webapi:1.0 .

# Construir con un tag específico
docker build -t webapi:latest -t webapi:v1.0.0 .

# Ver las imágenes creadas
docker images

# Ver información detallada de la imagen
docker inspect webapi:1.0
```

#### Paso 2: Ejecutar el contenedor

```bash
# Ejecutar el contenedor en primer plano
docker run -p 5000:8080 webapi:1.0

# Ejecutar en segundo plano (detached)
docker run -d -p 5000:8080 --name webapi-container webapi:1.0

# Ejecutar con variables de entorno
docker run -d -p 5000:8080 \
  -e ASPNETCORE_ENVIRONMENT=Development \
  -e ASPNETCORE_URLS=http://+:8080 \
  --name webapi-container \
  webapi:1.0

# Ejecutar con reinicio automático
docker run -d -p 5000:8080 --restart unless-stopped --name webapi-container webapi:1.0
```

#### Paso 3: Gestionar el contenedor

```bash
# Ver contenedores en ejecución
docker ps

# Ver todos los contenedores (incluso detenidos)
docker ps -a

# Ver logs del contenedor
docker logs webapi-container

# Ver logs en tiempo real
docker logs -f webapi-container

# Detener el contenedor
docker stop webapi-container

# Iniciar el contenedor
docker start webapi-container

# Reiniciar el contenedor
docker restart webapi-container

# Eliminar el contenedor
docker rm webapi-container

# Eliminar el contenedor forzadamente (si está corriendo)
docker rm -f webapi-container
```

#### Paso 4: Limpiar imágenes y contenedores

```bash
# Eliminar una imagen específica
docker rmi webapi:1.0

# Eliminar imagen forzadamente
docker rmi -f webapi:1.0

# Eliminar contenedores detenidos
docker container prune

# Eliminar imágenes sin usar
docker image prune

# Eliminar todo (contenedores, imágenes, redes, volúmenes)
docker system prune -a

# Ver espacio usado por Docker
docker system df
```

---

## 🔍 Parte 5: Verificar que Funciona

### 1. Verificar que el contenedor está corriendo

```bash
docker ps
```

Deberías ver algo como:
```
CONTAINER ID   IMAGE          PORTS                    NAMES
abc123def456   webapi         0.0.0.0:5000->8080/tcp   webapi-container
```

### 2. Acceder a la API

Abre tu navegador y visita:

- **Swagger UI**: http://localhost:5000/swagger
- **Health Check**: http://localhost:5000/health (si está configurado)
- **Endpoint de ejemplo**: http://localhost:5000/api/...

### 3. Probar con curl

```bash
# Probar endpoint
curl http://localhost:5000/api/weatherforecast

# Probar health check
curl http://localhost:5000/health
```

---

## 🛠️ Parte 6: Personalización y Consejos

### Cambiar el puerto

En `docker-compose.yml`, modifica la sección `ports`:

```yaml
ports:
  - "8080:8080"  # Cambiar 5000 por el puerto que quieras
```

### Modo Producción

Cambia el environment en `docker-compose.yml`:

```yaml
environment:
  - ASPNETCORE_ENVIRONMENT=Production
  - ASPNETCORE_URLS=http://+:8080
```

### Agregar Variables de Entorno Personalizadas

```yaml
services:
  webapi:
    # ... configuración existente ...
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:8080
      - MiVariable=MiValor
      - ConnectionStrings__DefaultConnection=tu-connection-string-aqui
```

---

## 📊 Parte 7: Ventajas del Multi-Stage Build

| Ventaja | Descripción |
|---------|-------------|
| **Tamaño reducido** | Imagen final ~200 MB vs ~600 MB |
| **Más segura** | No incluye herramientas de desarrollo |
| **Más rápida** | Descarga y deploy más veloces |
| **Mejor práctica** | Separación clara entre build y runtime |

---

## 🐛 Parte 8: Solución de Problemas Comunes

### Problema 1: Puerto ya en uso

```bash
Error: bind: address already in use
```

**Solución**: Cambia el puerto en docker-compose.yml o detén el servicio que usa el puerto:

```bash
# Windows
netstat -ano | findstr :5000

# Linux/Mac
lsof -i :5000
```

### Problema 2: Cambios no se reflejan

**Solución**: Reconstruir sin caché:

```bash
docker-compose down
docker-compose build --no-cache
docker-compose up
```

### Problema 3: Contenedor se detiene inmediatamente

**Solución**: Ver los logs para identificar el error:

```bash
docker-compose logs webapi
```

### Problema 4: Error "Cannot find project file"

**Solución**: Verifica que el archivo .csproj esté en el contexto del build y que el nombre coincida en el Dockerfile:

```dockerfile
COPY ["webapi.csproj", "./"]
```

### Problema 5: Imagen muy grande

**Solución**: Asegúrate de usar multi-stage build y que el .dockerignore esté correctamente configurado.

---

## ✅ Checklist del Laboratorio

- [ ] Crear archivo `Dockerfile` en la raíz del proyecto
- [ ] Crear archivo `docker-compose.yml` en la raíz
- [ ] Crear archivo `.dockerignore` en la raíz
- [ ] Ejecutar `docker-compose up --build`
- [ ] Verificar que el contenedor está corriendo con `docker ps`
- [ ] Acceder a http://localhost:5000/swagger
- [ ] Probar un endpoint de la API
- [ ] Ver los logs con `docker-compose logs -f`
- [ ] Detener los servicios con `docker-compose down`

---

## 🎓 Conceptos Clave Aprendidos

1. **Multi-Stage Build**: Optimiza el tamaño de la imagen
2. **Docker Compose**: Orquesta múltiples servicios fácilmente
3. **.dockerignore**: Mejora el rendimiento del build
4. **Puertos**: Mapeo entre host y contenedor
5. **Networks**: Comunicación entre servicios
6. **Environment Variables**: Configuración por ambiente

---

## 📚 Recursos Adicionales

- [Documentación oficial de Docker](https://docs.docker.com/)
- [.NET Docker Images](https://hub.docker.com/_/microsoft-dotnet)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Best Practices para .NET en Docker](https://docs.microsoft.com/dotnet/core/docker/build-container)

---

## 🎯 Siguiente Paso

Una vez completado este laboratorio, estás listo para:
- Agregar más microservicios al docker-compose
- Integrar bases de datos
- Configurar redes entre servicios
- Deploy en Kubernetes

---

**¡Felicidades! Has completado el laboratorio de Docker para .NET Core 8** 🎉

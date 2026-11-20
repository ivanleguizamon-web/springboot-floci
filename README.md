# Spring Boot + Floci (AWS RDS MySQL local)

Backend básico en Spring Boot conectado a una base de datos MySQL
emulada con [Floci](https://floci.io/), un emulador local de AWS que corre sobre Docker.

**Autor:** Tu Nombre

## Servicios de Floci analizados

| Servicio | Uso en el proyecto |
|---|---|
| RDS (MySQL) | Base de datos relacional del backend (**usado**) |
| S3 | Posible almacenamiento de archivos (no usado) |
| SQS | Posible cola de mensajes para procesos asíncronos (no usado) |

## Requisitos

- Docker Desktop
- Java 21
- AWS CLI v2
- Git

## Estructura

```
mi-proyecto/
├── docker-compose.yml
├── .env.example
├── .gitignore
├── README.md
├── evidencias/
└── backend/        (proyecto Spring Boot)
```

## Instrucciones

Los comandos están escritos para PowerShell (Windows).

### 1. Levantar Floci

```powershell
docker compose up -d
docker ps
```

Floci expone la API de AWS en el puerto `4566` y el proxy de RDS en `7001-7099`.

### 2. Variables de entorno de AWS

```powershell
$env:AWS_ENDPOINT_URL="http://localhost:4566"
$env:AWS_DEFAULT_REGION="us-east-1"
$env:AWS_ACCESS_KEY_ID="test"
$env:AWS_SECRET_ACCESS_KEY="test"
```

Las variables disponibles están en `.env.example`. Las variables de la sesión
se pierden al cerrar la terminal, hay que volver a cargarlas.

### 3. Crear la base de datos MySQL en Floci

```powershell
aws rds create-db-instance --db-instance-identifier appdb --db-instance-class db.t3.micro --engine mysql --master-username admin --master-user-password admin12345 --db-name appdb --allocated-storage 20
aws rds describe-db-instances --query "DBInstances[0].[DBInstanceStatus,Endpoint.Port]"
```

El estado debe ser `available`. El puerto devuelto (`7001`) es el que usa el backend.

### 4. Ejecutar el backend

```powershell
$env:DB_HOST="localhost"
$env:DB_PORT="7001"
$env:DB_NAME="appdb"
$env:DB_USER="admin"
$env:DB_PASSWORD="admin12345"
$env:SERVER_PORT="8081"
cd backend
.\mvnw spring-boot:run
```

Debe aparecer `HikariPool-1 - Added connection` y `Started BackendApplication`.

### 5. Probar la API

```powershell
# Estado
Invoke-RestMethod -Uri http://localhost:8081/actuator/health

# Crear
Invoke-RestMethod -Uri http://localhost:8081/usuarios -Method Post -ContentType "application/json" -Body '{"nombre":"Ana"}'

# Listar
Invoke-RestMethod -Uri http://localhost:8081/usuarios

# Actualizar
Invoke-RestMethod -Uri http://localhost:8081/usuarios/1 -Method Put -ContentType "application/json" -Body '{"nombre":"Ana Maria"}'

# Eliminar
Invoke-RestMethod -Uri http://localhost:8081/usuarios/1 -Method Delete
```

### 6. Verificar directamente en MySQL

```powershell
docker exec -it $(docker ps --filter "name=floci-aws-rds-db" -q) mysql -uadmin -padmin12345 appdb -e "SELECT * FROM usuario;"
```

## Endpoints

| Método | Ruta | Acción |
|---|---|---|
| GET | `/actuator/health` | Estado de la aplicación |
| POST | `/usuarios` | Crear usuario |
| GET | `/usuarios` | Listar usuarios |
| GET | `/usuarios/{id}` | Obtener un usuario |
| PUT | `/usuarios/{id}` | Actualizar usuario |
| DELETE | `/usuarios/{id}` | Eliminar usuario |

## Configuración por variables de entorno

`application.properties` lee los datos de conexión de variables de entorno,
con valores por defecto para desarrollo local:

```properties
spring.datasource.url=jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:7001}/${DB_NAME:appdb}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=${DB_USER:admin}
spring.datasource.password=${DB_PASSWORD:admin12345}
server.port=${SERVER_PORT:8081}
```

Las credenciales son de prueba, solo válidas para el emulador local.

## Evidencias

**Floci y MySQL corriendo**
![docker ps](evidencias/01-docker-ps.png)

**Base de datos creada en RDS (Floci)**
![RDS](evidencias/02-rds-describe.png)

**Backend conectado a la base de datos**
![Spring Boot](evidencias/03-spring-started.png)

**Health check**
![Health](evidencias/04-health.png)

**Operaciones CRUD**
![CRUD](evidencias/05-crud.png)

**Datos guardados en MySQL**
![MySQL](evidencias/06-mysql-select.png)

## Problemas frecuentes

| Error | Solución |
|---|---|
| `NoRegion` en AWS CLI | Cargar las variables de entorno de AWS (paso 2) |
| `DBInstanceAlreadyExists` | La base ya fue creada, continuar con el paso 4 |
| Puerto 8080 ocupado | Usar `SERVER_PORT=8081` |
| El backend no conecta a la base | Verificar que `docker-compose.yml` publica `7001-7099` y que `DB_PORT` coincide con el puerto de `describe-db-instances` |
| `400 Bad Request` con `curl.exe` | Usar `Invoke-RestMethod` (problema de comillas en PowerShell) |
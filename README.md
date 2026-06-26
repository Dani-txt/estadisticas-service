# Estadísticas Service (VidalCasino 2.0)

## Descripción

**Estadísticas Service** es el microservicio encargado de generar estadísticas y dashboards para **VidalCasino 2.0**, desarrollado con **Python** y **FastAPI**.

El servicio opera en modo **solo lectura**, consumiendo la información almacenada en la base de datos compartida para calcular indicadores, métricas y reportes utilizados por el casino.

---

# Arquitectura e Integración

## Base de Datos

El microservicio se conecta directamente a una base de datos **PostgreSQL** compartida con el servicio principal (`casino-backend`).

Consulta información de las siguientes tablas:

* `usuarios`
* `transacciones`
* `apuestas`

El servicio es de **solo lectura**, por lo que no crea, modifica ni elimina registros de la base de datos.

## Autenticación

El servicio no posee un sistema de autenticación propio.

Valida los tokens JWT utilizando el mismo `JWT_SECRET` configurado en el backend principal.

---

# API

## Prefijo de rutas

```text
/api/estadisticas
```

## Documentación Swagger

```text
/docs
```

---

# Endpoints

| Método | Endpoint                     | Descripción                                                                                                       |
| ------ | ---------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| GET    | `/api/estadisticas/mias`     | Obtiene los KPIs del usuario autenticado, desglose por tipo y evolución del saldo.                                |
| GET    | `/api/estadisticas/globales` | Obtiene métricas globales del casino, incluyendo usuarios, GGR, principales jugadores y estadísticas de apuestas. |
| GET    | `/livez`                     | Liveness Probe. Verifica que el contenedor continúe en ejecución. Retorna `200 OK`.                               |
| GET    | `/readyz`                    | Readiness Probe. Verifica la conexión con PostgreSQL. Retorna `200 OK` o `503 Service Unavailable`.               |

---

# Desarrollo Local

## Requisitos

* Python 3.12 o superior
* PostgreSQL accesible con las tablas compartidas creadas por `casino-backend`

---

## 1. Crear el entorno virtual

```bash
python -m venv .venv
```

### Linux / macOS

```bash
source .venv/bin/activate
```

### Windows

```bash
.venv\Scripts\activate
```

---

## 2. Instalar dependencias

```bash
pip install -r requirements.txt
```

---

## 3. Configurar variables de entorno

Copiar el archivo de ejemplo:

```bash
cp .env.example .env
```

Configurar las variables correspondientes al entorno local.

**Importante:** nunca subir el archivo `.env` al repositorio.

---

## 4. Ejecutar el servidor

```bash
uvicorn app.main:app --reload --port 8006
```

El servicio estará disponible en:

```text
http://localhost:8006
```

La documentación Swagger estará disponible en:

```text
http://localhost:8006/docs
```

---

# Estructura del Proyecto

```text
estadisticas-service/
├── app/
│   ├── main.py
│   ├── db.py
│   ├── auth.py
│   └── ...
│
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── hpa.yaml
│
├── .github/
│   └── workflows/
│       └── deploy.yaml
│
├── Dockerfile
├── .dockerignore
├── requirements.txt
├── .env.example
└── README.md
```

---

# Despliegue en Kubernetes

Los manifiestos de Kubernetes se encuentran en la carpeta:

```text
k8s/
```

## deployment.yaml

Configura:

* Número de réplicas.
* Recursos (`requests` y `limits`).
* `livenessProbe`.
* `readinessProbe`.

## service.yaml

Expone el microservicio mediante un servicio de tipo:

```text
ClusterIP
```

para permitir la comunicación interna dentro del clúster.

## hpa.yaml

Configura el **Horizontal Pod Autoscaler (HPA)**.

Escalado configurado:

* Mínimo: 2 réplicas.
* Máximo: 6 réplicas.
* Basado en utilización de CPU.

---

# Contenedorización

El proyecto utiliza un `Dockerfile` basado en:

```text
python:3.12-slim
```

Además, incorpora un archivo `.dockerignore` para excluir archivos innecesarios durante la construcción de la imagen, como:

* `.venv`
* `__pycache__`
* archivos temporales

---

# Estrategia de Ramas

El proyecto sigue un flujo **polirepo** con tres ramas principales.

## main

Rama estable del proyecto.

## dev

Desarrollo diario, implementación de funcionalidades y pruebas locales.

## deploy

Cada `push` o `merge` sobre esta rama dispara automáticamente el pipeline de despliegue.

---

# Pipeline CI/CD

El workflow se encuentra en:

```text
.github/workflows/deploy.yaml
```

El pipeline realiza las siguientes etapas.

## 1. Construcción

Genera la imagen Docker utilizando tres etiquetas:

* `vX.Y.Z`
* `latest`
* `${{ github.sha }}`

## 2. Publicación

Publica la imagen en un repositorio privado de **Amazon ECR**.

## 3. Despliegue

Actualiza automáticamente el clúster **Amazon EKS** ejecutando:

```bash
kubectl apply
```

sobre los manifiestos del directorio `k8s/`.

---

# Pruebas de Carga

Se realizaron pruebas de carga para validar el comportamiento del microservicio desplegado en **Amazon EKS**.

Las pruebas permitieron verificar:

* Disponibilidad del servicio bajo carga.
* Correcto funcionamiento de las sondas de salud.
* Escalado automático mediante el Horizontal Pod Autoscaler.
* Distribución de carga entre las réplicas del Deployment.

---

# Seguridad

Las credenciales sensibles no se almacenan en el código fuente.

## Kubernetes Secrets

Se utilizan para almacenar:

* Credenciales de PostgreSQL.
* Variables de entorno.
* `JWT_SECRET`.

## GitHub Secrets

Las credenciales temporales de AWS Academy utilizadas por el pipeline de CI/CD se almacenan de forma cifrada mediante **GitHub Secrets**.

No se permiten credenciales en texto plano dentro del repositorio ni en los manifiestos de Kubernetes.

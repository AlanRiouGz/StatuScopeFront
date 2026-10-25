# Frontend StatuScope en Harness

Este repositorio tiene una pipeline independiente del backend. Ambos se despliegan en el mismo
namespace `statusscope-local`, donde la base MySQL se prepara desde el repositorio del backend.

## Valores acordados

- Registro Harness: `statuscopefront`.
- Repositorio de imagen: `statusscope-frontend`.
- Servicio Kubernetes: `statusscope-frontend`; NodePort predeterminado: `30080`.
- Backend dentro del namespace: `statusscope-backend:8080`.
- API del navegador: `/api`, que Nginx reenvía al backend quitando ese prefijo.

## Pipeline

Importar `.harness/pipeline-template.yml` como **Pipeline Template** y crear una pipeline a partir
de él. Seleccionar el repositorio de frontend en Codebase y conservar `Docker_infra` si es el
conector correcto. El registro es un parámetro con valor predeterminado `statuscopefront`:
usar su identificador real si Harness asigna uno distinto al nombre visible.

Crear un servicio CD Kubernetes cuya fuente de artefacto sea Harness Artifact Registry,
registro `statuscopefront`, repositorio `statusscope-frontend` y tag `<+pipeline.sequenceId>`.
Configurar manifiestos desde `k8s/manifests` y valores desde `k8s/values.yml`, usando la misma
rama/revisión que CI. El entorno/infraestructura debe apuntar a Docker Desktop y `statusscope-local`.
El Secret `statusscope-registry` debe permitir descargar imágenes del registro privado.

La pipeline instala dependencias con `npm ci`, ejecuta los tests de Node de Vitest, exporta Expo,
construye `Dockerfile.ci`, ejecuta los escáneres habilitados, publica y despliega canary/rolling.
`Dockerfile.ci` empaqueta el resultado ya compilado. Las pruebas no dependen de `STO_scans`.
La instalación respeta `.npmrc` (`legacy-peer-deps=true`) y no descarga Cypress para estos tests.
Los tests de Storybook/Cypress no se incluyen en esta primera pipeline.

Proporcionar estos cuatro valores de la aplicación **Web** de Firebase:

- `Firebase_api_key` -> `EXPO_PUBLIC_FIREBASE_API_KEY`.
- `Firebase_auth_domain` -> `EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN`.
- `Firebase_project_id` -> `EXPO_PUBLIC_FIREBASE_PROJECT_ID`.
- `Firebase_app_id` -> `EXPO_PUBLIC_FIREBASE_APP_ID`.

Son configuración pública del cliente, incluida en el JavaScript generado. No usar aquí el JSON
ni la clave privada del servicio Firebase Admin del backend. El proyecto Firebase debe coincidir.
Expo incorpora estas variables al compilar; cambiar variables del pod después no cambia la web.

## Comprobación local

```powershell
$env:CYPRESS_INSTALL_BINARY='0'
npm ci --no-audit --no-fund
npx vitest run --project unit
$env:EXPO_PUBLIC_API_URL='/api'
# Establecer además las cuatro variables públicas de Firebase Web.
npm run build:web
docker build -f Dockerfile.ci -t statusscope-frontend:local .
```

El Dockerfile normal permite compilar dentro de Docker usando los mismos valores como `--build-arg`.
`/healthz` comprueba Nginx. `/api/q/health/ready` comprueba también el backend y MySQL.
Para otro entorno en el mismo clúster, cambiar el NodePort: no puede repetirse entre namespaces.

La configuración Nginx toma el DNS de `/etc/resolv.conf` al iniciar y resuelve el backend en tiempo
de ejecución, por lo que puede arrancar aunque el backend todavía no exista.
En Docker sin Kubernetes, ambos contenedores deben compartir una red y el backend debe tener el
alias `statusscope-backend`.

La guía completa de base de datos, secretos, migraciones y configuración conjunta está en
`HARNESS_DEPLOYMENT.md` del repositorio del backend.

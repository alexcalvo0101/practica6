# Apuntes de examen — Infraestructura y Procesos de Soporte

> Guía de supervivencia paso a paso. Pensada para tener abierta durante el examen.
> Cubre Práctica 6 (CI con GitHub Actions) y Práctica 7 (CD con Docker + Kubernetes).

---

## ÍNDICE RÁPIDO

1. [Conceptos clave (para no perderte)](#0-conceptos-clave)
2. [PRÁCTICA 6 — CI con GitHub Actions](#practica-6)
3. [PRÁCTICA 7 — CD con Docker + Kubernetes](#practica-7)
4. [Subir un proyecto a GitHub desde cero](#subir-a-github)
5. [Catálogo de errores y cómo resolverlos](#errores)
6. [Chuletas de comandos](#chuletas)

---

<a name="0-conceptos-clave"></a>
## 1. CONCEPTOS CLAVE (léelos antes de empezar)

| Concepto | Qué es en una frase |
|----------|---------------------|
| **CI** (Integración Continua) | Cada push compila y testea el código automáticamente (Práctica 6). |
| **CD** (Despliegue Continuo) | Cada push además despliega la app automáticamente (Práctica 7). |
| **Workflow** | Fichero `.yml` en `.github/workflows/` que define los pasos automáticos. |
| **`run` vs `uses`** | `run` = comando de terminal. `uses` = Action ya hecha por otro. |
| **`on`** | El disparador: cuándo se ejecuta el workflow (push, pull_request). |
| **`job`** | Un trabajo. Corre en una máquina virtual (o en tu máquina con self-hosted). |
| **`steps`** | Los pasos dentro del job, en orden. |
| **Docker image** | "Foto" congelada de tu app lista para ejecutarse en cualquier sitio. |
| **Docker Hub** | Sitio donde se publican las imágenes Docker (como GitHub pero de imágenes). |
| **Kubernetes (k8s)** | Orquestador que ejecuta y mantiene vivas tus imágenes Docker. |
| **Deployment** | Recurso k8s: "ejecuta mi imagen con X réplicas". |
| **Service** | Recurso k8s: "expón mi app en un puerto accesible". |
| **Self-hosted runner** | Programa que ejecuta el workflow EN TU MÁQUINA (necesario para k8s local). |

**REGLA DE ORO DEL EXAMEN:** el enunciado te da enlaces a repos de GitHub.
No te tienes que saber nada de memoria — **entra a esos enlaces, busca la sección
"Usage" o "Basic Usage", copia el bloque y adáptalo**. Eso es exactamente lo que
se espera que hagas.

---

<a name="practica-6"></a>
## 2. PRÁCTICA 6 — CI con GitHub Actions

### Objetivo
Workflow que: se ejecuta en push/PR → compila → ejecuta tests → genera informe →
badge de estado en README → badge de cobertura Jacoco.

### PASO 0 — Crear el fichero
El workflow va SIEMPRE en esta ruta exacta:
```
.github/workflows/ci.yml
```
⚠️ **CUIDADO:** no debe haber carpetas intermedias. Si ves algo como
`.github/modernize/java-upgrade/workflows/ci.yml` está MAL. Muévelo a
`.github/workflows/ci.yml` directamente.

### PASO 1 — Esqueleto + disparador (`on`)
El enunciado dice "push o pull request". Se traduce así:

```yaml
name: workflow

on:
  push:
    branches: ["**"]
  pull_request:
    branches: ["**"]
```
⚠️ Las `["**"]` SIEMPRE con comillas. Sin comillas YAML da error.
`"**"` significa "cualquier rama".

### PASO 2 — El job y la máquina
```yaml
jobs:
  mi-job:
    runs-on: ubuntu-latest

    steps:
```

### PASO 3 — Preparar la máquina (checkout + Java)
La máquina virtual empieza VACÍA. Hay que: (1) traer tu código, (2) instalar Java.
Ambas son Actions ya hechas → `uses`.

```yaml
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
```
- `checkout` = descarga tu código en la máquina.
- `setup-java` = instala Java. `with` = los parámetros (versión y proveedor).

### PASO 4 — Permisos + compilar + tests
El enunciado dice literalmente: compilar con `./mvnw compile`, y antes de los tests
"darle permisos de escritura con chmod". Son comandos de terminal → `run`.

```yaml
      - name: Permisos de ejecucion
        run: chmod +x mvnw

      - name: Compilar
        run: ./mvnw compile --no-transfer-progress

      - name: Permisos de escritura
        run: chmod +w mvnw

      - name: Ejecutar tests
        run: ./mvnw verify --no-transfer-progress
```
- `chmod +x mvnw` → da permiso de EJECUCIÓN (en Linux el zip no lo conserva).
- `chmod +w mvnw` → da permiso de ESCRITURA (lo pide el enunciado explícitamente).
- `./mvnw verify` → lanza Surefire (tests unitarios) + Failsafe (tests IT) + Jacoco.

### PASO 5 — Informe de tests (enlace github-test-reporter)
Entra al enlace del enunciado → sección "Basic Usage" → copia y adapta.
El report-path apunta a los XML que genera Maven:

```yaml
      - name: Publish Test Report
        uses: ctrf-io/github-test-reporter@v1
        with:
          report-path: 'target/surefire-reports/*.xml'
          github-report: true
        if: always()
```
- **`if: always()`** = ejecuta este paso AUNQUE los tests fallen. Es clave: si fallan
  los tests es justo cuando MÁS quieres ver el informe. Sin esto, si algo falla nunca
  verías el informe.

### PASO 6 — Badge de cobertura Jacoco (enlace jacoco-badge-generator)
Entra al enlace → "Basic Action Syntax" → copia. Quita el paso de Maven (ya lo tienes):

```yaml
      - name: Generate JaCoCo Badge
        uses: cicirello/jacoco-badge-generator@v2
        with:
          generate-branches-badge: true
```
- Por defecto busca el CSV en `target/site/jacoco/jacoco.csv` (ruta estándar de Maven),
  no hace falta especificarlo.

### PASO 7 — ⚠️ COMMIT del badge (ESTO SE OLVIDA Y ROMPE EL BADGE)
La Action GENERA el badge pero NO lo sube al repo. Lo dice su propio README:
*"The action doesn't commit the badge file."* Hay que añadir un paso que haga commit.
Está en el "Example Workflow 1" del enlace. Copia y cambia nombre/email:

```yaml
      - name: Commit the badge (if it changed)
        run: |
          if [[ `git status --porcelain` ]]; then
            git config --global user.name 'TU NOMBRE'
            git config --global user.email 'TU_EMAIL@uma.es'
            git add -A
            git commit -m "Autogenerated JaCoCo coverage badge"
            git push
          fi
```

### PASO 8 — ⚠️ PERMISOS del workflow (SI NO, ERROR 128)
Si el `git push` da **"exit code 128"** es que el workflow no tiene permiso de escritura.
Arréglalo en GitHub (NO en el código):
```
Repo → Settings → Actions → General → Workflow permissions
→ marcar "Read and write permissions" → Save
```
Luego: Actions → el workflow fallido → "Re-run jobs" → "Re-run all jobs"
(no hace falta otro commit).

### PASO 9 — Badges en el README
Van AL PRINCIPIO del `README.md`. El de build lo genera GitHub solo. El de coverage
lo genera Jacoco (mira su README, sección "Adding the Badges"):

```markdown
[![Build Status](https://github.com/TU_USUARIO/TU_REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/TU_USUARIO/TU_REPO/actions/workflows/ci.yml)

![Coverage](.github/badges/jacoco.svg)
```

### RESULTADO ESPERADO
- Badge **workflow: passing** (verde) → compila y tests OK.
- Badge **coverage: XX%** → puede salir rojo si la cobertura es baja (ej: 18.5%).
  El rojo NO es error, solo significa poca cobertura porque los tests están casi vacíos.

### YAML COMPLETO PRÁCTICA 6 (referencia)
```yaml
name: workflow

on:
  push:
    branches: ["**"]
  pull_request:
    branches: ["**"]

jobs:
  mi-job:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Permisos de ejecucion
        run: chmod +x mvnw

      - name: Compilar
        run: ./mvnw compile --no-transfer-progress

      - name: Permisos de escritura
        run: chmod +w mvnw

      - name: Ejecutar tests
        run: ./mvnw verify --no-transfer-progress

      - name: Publish Test Report
        uses: ctrf-io/github-test-reporter@v1
        with:
          report-path: 'target/surefire-reports/*.xml'
          github-report: true
        if: always()

      - name: Generate JaCoCo Badge
        uses: cicirello/jacoco-badge-generator@v2
        with:
          generate-branches-badge: true

      - name: Commit the badge (if it changed)
        run: |
          if [[ `git status --porcelain` ]]; then
            git config --global user.name 'TU NOMBRE'
            git config --global user.email 'TU_EMAIL@uma.es'
            git add -A
            git commit -m "Autogenerated JaCoCo coverage badge"
            git push
          fi
```

---

<a name="practica-7"></a>
## 3. PRÁCTICA 7 — CD con Docker + Kubernetes

### Objetivo
Workflow que: compila → genera imagen Docker → la sube a Docker Hub →
se conecta a Kubernetes local (vía self-hosted runner) → despliega automáticamente.

### ⚠️ ORDEN REAL (lo dice el final del enunciado)
> "Antes de definir el workflow, será necesario crear la imagen del contenedor de
> forma local y realizar un despliegue inicial en Kubernetes (Deployment + Service)."

Así que el orden es:
1. Dockerfile + imagen local
2. Subir imagen a Docker Hub (manual)
3. Despliegue inicial manual en Kubernetes (Deployment + Service)
4. Self-hosted runner
5. Secrets de Docker Hub
6. Workflow ci.yml
7. Validar: hacer un cambio → push → se actualiza solo

---

### PARTE A — DOCKER

#### A.1 — Crear el `Dockerfile` (en la raíz, junto al pom.xml)
3 líneas, cada una hace una cosa:
```dockerfile
FROM eclipse-temurin:21-jdk
COPY target/Practica6-0.0.1-SNAPSHOT.jar app.jar
CMD ["java", "-jar", "app.jar"]
```
- `FROM` → "usa Java 21" (la base).
- `COPY` → "mete el JAR dentro de la imagen" y lo renombra a app.jar.
- `CMD` → "arráncalo con java -jar". (También vale `ENTRYPOINT`, es casi lo mismo).

⚠️ El nombre del JAR sale del `pom.xml`: `artifactId` + `-` + `version` + `.jar`.
En este proyecto: `Practica6-0.0.1-SNAPSHOT.jar`. Verifícalo siempre.

#### A.2 — Generar el JAR (Maven)
```bash
./mvnw package -DskipTests
```
- `-DskipTests` = sin tests, va más rápido. Debe terminar en **BUILD SUCCESS**.
- El JAR queda en la carpeta `target/`.

#### A.3 — Construir la imagen
```bash
docker build -t TU_USUARIO_DOCKERHUB/practica6:latest .
```
- El `.` final = "el Dockerfile está en la carpeta actual".
- `-t` = nombre (tag) de la imagen.

#### A.4 — Login en Docker Hub
```bash
docker login
```
- Te pide usuario y contraseña. **MEJOR usar un Access Token** (lo pidieron en el examen).
- Crear token: hub.docker.com → tu usuario → Account Settings → Security →
  New Access Token → Generate → **CÓPIALO, solo se ve una vez**.

#### A.5 — Subir la imagen
```bash
docker push TU_USUARIO_DOCKERHUB/practica6:latest
```
- Comprueba en hub.docker.com que aparece. → **ESTA ES UNA CAPTURA DEL ENTREGABLE**.

---

### PARTE B — KUBERNETES (despliegue inicial manual)

#### B.0 — Comprobar que Kubernetes está vivo
```bash
kubectl get nodes
```
Debe salir algo como:
```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   92s   v1.32.2
```
Si sale **Ready** → perfecto, sigue. Si da error → ver sección de errores.

#### B.1 — Crear `deployment.yml` (Deployment + Service en un mismo fichero)
Se pueden poner los dos recursos separados por `---`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: practica6
spec:
  replicas: 1
  selector:
    matchLabels:
      app: practica6
  template:
    metadata:
      labels:
        app: practica6
    spec:
      containers:
        - name: practica6
          image: TU_USUARIO_DOCKERHUB/practica6:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: practica6
spec:
  type: NodePort
  selector:
    app: practica6
  ports:
    - port: 8080
      targetPort: 8080
```
**Qué hace cada uno (para explicarlo en el examen):**
- **Deployment** = "ejecuta mi imagen Docker con 1 réplica en el puerto 8080".
- **Service** = "expón la app en el puerto 8080 para que sea accesible".
- `selector.app` debe coincidir con `labels.app` para que el Service encuentre el pod.

#### B.2 — Desplegar en Kubernetes
```bash
kubectl apply -f deployment.yml
```
Debe decir "created".

#### B.3 — Comprobar que corre
```bash
kubectl get pods
```
Debe salir STATUS **Running** y READY **1/1**:
```
NAME                         READY   STATUS    RESTARTS   AGE
practica6-64dbd898db-jjt5x   1/1     Running   0          22s
```

---

### PARTE C — SELF-HOSTED RUNNER

#### ¿Por qué se necesita?
Tu Kubernetes está en TU máquina local. Los servidores de GitHub NO pueden acceder a él.
El runner local hace de PUENTE: el workflow corre en tu máquina y sí puede hablar con k8s.

#### C.1 — Crear el runner
```
Repo → Settings → Actions → Runners → New self-hosted runner
```
GitHub te da una lista de comandos (download + config + run). Ejecútalos en la terminal
en orden. El último (`./run.cmd` o `run.sh`) arranca el runner. Déjalo corriendo.
(Para que esté siempre activo se puede instalar como servicio).

---

### PARTE D — SECRETS DE DOCKER HUB

No se pone el token en texto plano en el yml. Se guarda como Secret:
```
Repo → Settings → Secrets and variables → Actions → New repository secret
```
Crea dos:
- `DOCKER_USERNAME` → tu usuario de Docker Hub
- `DOCKER_TOKEN` → el token que generaste antes

Se usan en el yml con la sintaxis `${{ secrets.NOMBRE }}`.

---

### PARTE E — EL WORKFLOW (ci.yml de la Práctica 7)

```yaml
name: CI/CD

on:
  push:
    branches: ["main"]

jobs:
  deploy:
    runs-on: self-hosted     # CLAVE: corre en TU máquina, no en GitHub

    steps:
      - uses: actions/checkout@v4

      - name: Login Docker Hub
        run: docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_TOKEN }}

      - name: Build imagen
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/practica6:latest .

      - name: Push imagen
        run: docker push ${{ secrets.DOCKER_USERNAME }}/practica6:latest

      - name: Desplegar en Kubernetes
        run: kubectl apply -f deployment.yml
```
- **`runs-on: self-hosted`** = lo más importante. Hace que corra en tu máquina y
  pueda acceder a tu Kubernetes local.

⚠️ Si la imagen no se actualiza al hacer push, fuerza el rollout:
```bash
kubectl rollout restart deployment/practica6
```

---

### PARTE F — VALIDAR EL DESPLIEGUE (lo pide el enunciado)
1. Haz un cambio cualquiera en la app (ej: cambia un texto en un controlador).
2. `git add . && git commit -m "cambio" && git push`
3. El workflow se dispara solo → construye imagen → la sube → actualiza k8s.
4. Comprueba con `kubectl get pods` que el pod se ha recreado (AGE bajo).

### ENTREGABLES PRÁCTICA 7
- Captura de los contenedores subidos a Docker Hub.
- Captura del workflow generado (el yml + ejecución en verde en Actions).

---

<a name="subir-a-github"></a>
## 4. SUBIR UN PROYECTO A GITHUB DESDE CERO

⚠️ **NO TE OLVIDES DE CREAR EL REPO** (te pasó en las dos prácticas).

1. github.com → botón verde **New** → nombre → **Public** → Create repository.
2. En la terminal, dentro de la carpeta del proyecto:
```bash
git init
git add .
git commit -m "primer commit"
git branch -M main
git remote add origin https://github.com/TU_USUARIO/TU_REPO.git
git push -u origin main
```

### ⚠️ Si el proyecto YA tenía otro git enlazado (ramas de otra asignatura)
Síntoma: `git branch` muestra ramas raras (dev, modulo5...).
Solución — desvincular y enlazar al repo nuevo:
```bash
git checkout main                 # asegúrate de estar en main (tu código está aquí)
git remote remove origin          # quita el repo viejo
git remote add origin https://github.com/TU_USUARIO/TU_REPO.git
git push -u origin main
```
- Tu código NO se pierde, solo estabas en otra rama.
- Las ramas viejas (dev, modulo5) no se suben salvo que lo hagas a propósito.

---

<a name="errores"></a>
## 5. CATÁLOGO DE ERRORES (los que te han pasado)

| Error / síntoma | Causa | Solución |
|-----------------|-------|----------|
| `ci.yml` en carpeta rara (`.github/modernize/...`) | Carpetas intermedias creadas por error | Mover a `.github/workflows/ci.yml` |
| YAML inválido en `branches: [**]` | Faltan comillas | Poner `["**"]` |
| `git push` → **exit code 128** en Actions | Workflow sin permiso de escritura | Settings → Actions → General → Workflow permissions → Read and write |
| Badge de coverage roto en README | La Action no commitea el SVG sola | Añadir paso "Commit the badge" (ver Paso 7 P6) |
| **No valid CTRF reports found** | El reporter no encuentra los XML | Verificar `report-path: 'target/surefire-reports/*.xml'`. No es crítico si el resto va bien |
| `kubectl get nodes` → **EOF / timeout** | Kubernetes aún arrancando o colgado | Esperar; si no, reiniciar Docker Desktop |
| Docker Desktop atascado en **"Starting Kubernetes"** | WSL2 / Resource Saver / recursos | `wsl --shutdown` en PowerShell + reiniciar Docker Desktop |
| No deja dar **Apply** al quitar Resource Saver | Límites gestionados por WSL2 | Usar `wsl --shutdown` directamente, no tocar Apply |
| Settings → Reset Kubernetes Cluster no carga | Clúster colgado | `wsl --shutdown` y reiniciar Docker Desktop |
| `git branch` muestra ramas de otra asignatura | Carpeta con git viejo enlazado | `git remote remove origin` y enlazar al repo nuevo |
| Olvidaste crear el repo en GitHub | — | Crearlo y hacer el push (sección 4) |
| Imagen no se actualiza en k8s tras push | Caché / mismo tag latest | `kubectl rollout restart deployment/practica6` |
| Coverage badge en ROJO | Cobertura baja (ej. 18.5%) | NO es un error. Es normal con tests vacíos |

### El truco que arregla casi todo en Docker Desktop / Kubernetes (Windows)
```bash
wsl --shutdown
```
(ejecútalo en PowerShell, luego reinicia Docker Desktop)

---

<a name="chuletas"></a>
## 6. CHULETAS DE COMANDOS

### Maven
```bash
./mvnw compile --no-transfer-progress     # compilar
./mvnw verify --no-transfer-progress      # compilar + tests + jacoco
./mvnw package -DskipTests                # generar el JAR sin tests
```

### Docker
```bash
docker build -t USUARIO/practica6:latest .   # construir imagen
docker login                                 # login (usar token como password)
docker push USUARIO/practica6:latest         # subir a Docker Hub
docker images                                # listar imágenes locales
```

### Kubernetes
```bash
kubectl get nodes                            # ver el clúster
kubectl apply -f deployment.yml              # desplegar
kubectl get pods                             # ver pods corriendo
kubectl get services                         # ver services
kubectl rollout restart deployment/practica6 # forzar actualización
kubectl delete -f deployment.yml             # borrar el despliegue
kubectl logs <nombre-del-pod>                # ver logs de un pod
```

### Git
```bash
git init
git add .
git commit -m "mensaje"
git branch -M main
git remote add origin URL_DEL_REPO
git push -u origin main
git remote remove origin                     # desvincular repo viejo
git checkout main                            # cambiar a rama main
git branch                                   # ver ramas
git status                                   # ver estado
```

### Windows / Docker Desktop (rescate)
```bash
wsl --shutdown                               # en PowerShell, arregla k8s colgado
```

---

## RECORDATORIO FINAL ANTES DE ENTREGAR

**Práctica 6:**
- [ ] `ci.yml` en `.github/workflows/`
- [ ] Workflow en verde (passing) en la pestaña Actions
- [ ] Badge de build y badge de coverage en el README
- [ ] Captura del repo con badges + tests pasados

**Práctica 7:**
- [ ] Imagen subida a Docker Hub (captura)
- [ ] Deployment + Service corriendo (`kubectl get pods` en Running)
- [ ] Self-hosted runner activo
- [ ] Secrets `DOCKER_USERNAME` y `DOCKER_TOKEN` creados
- [ ] Workflow con `runs-on: self-hosted`
- [ ] Validado: un cambio + push actualiza la app sola
- [ ] Captura del workflow + contenedores en Docker Hub

> Si te atascas: respira, entra al enlace que da el enunciado, busca "Usage",
> copia y adapta. No hay que saberse nada de memoria.

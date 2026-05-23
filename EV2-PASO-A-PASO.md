# EV2 - Paso a paso (guía completa con captura de evidencias)

Guía probada end-to-end. Cada bloque indica **qué capturar** para el informe PDF (📸 = screenshot).

> **Carpeta sugerida para evidencias**: crear `evidencias-EV2/` en cualquier lugar. Subcarpetas por bloque:
> ```
> evidencias-EV2/
>   00-prereq/
>   01-provision-ec2/
>   02-cluster/
>   03-demo-api/
>   04-nginx/
>   05-apache/
>   06-rollback/
>   07-estado-final/
>   08-dockerhub/
>   09-github-actions/
> ```
>
> **Total: 33 capturas**. Para el informe usar ~10-15 más representativas; resto es backup.

---

## Arquitectura final

```
3 repos GitHub        →  GitHub Actions  →  Docker Hub  →  SSH  →  k3s/EC2  →  NodePort
─────────────────        ──────────────     ──────────     ──     ──────────    ────────
Primary (recetas)        provision lab      bvnjitsss/     scp    pod nginx     30080
Runner  (app+nginx)      build+push+deploy  demo-api       apply  pod demo-api  30090
Apache  (apache)         build+push+deploy  apache-demo           pod apache    30100
```

---

## 0. Pre-requisitos

- [ ] Cuenta GitHub `BenjaminDuran` con los 3 repos:
  - `AUY1104-BenjaminDuran-Primary` (recetas reutilizables)
  - `AUY1104-BenjaminDuran-Runner` (app ramo + nginx)
  - `AUY1104-BenjaminDuran-Apache` (servidor Apache)
- [ ] Cuenta Docker Hub `bvnjitsss` + token con permisos Read/Write/Delete
- [ ] AWS Learner Lab activo
- [ ] `labsuser.pem` descargado en `C:/Users/drafi/`

### 📸 EVIDENCIAS BLOQUE 0 (1 captura)
- **00-docker-hub-token.png** — pantalla Docker Hub con token nuevo creado (oculta el valor del token)

---

## 1. Levantar EC2 + k3s vía workflow (NO manual)

> **Por qué workflow**: instala k3s con `--tls-san <IP>` para que `kubectl` externo no falle TLS. Manual no lo hace.

### 1.1 Configurar secrets en Primary

GitHub → repo Primary → Settings → Secrets and variables → Actions → Secrets:

| Secret | Valor | Dónde sacarlo |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | string | Learner Lab → AWS Details → AWS CLI |
| `AWS_SECRET_ACCESS_KEY` | string | mismo lugar |
| `AWS_SESSION_TOKEN` | string | mismo lugar (expira ~4h) |
| `EA2_SSH_PRIVATE_KEY` | contenido completo `labsuser.pem` | con `-----BEGIN...-----END-----` |
| `DOCKER_USERNAME` | `bvnjitsss` | Docker Hub |
| `DOCKER_PASSWORD` | token Docker Hub | hub.docker.com → Account → Personal access tokens |

### 1.2 Ejecutar workflow

- GitHub → repo Primary → Actions → **"Ej K8S desde alumno"** → Run workflow
- Inputs (defaults sirven):
  - `aws_region`: `us-east-1`
  - `instance_type`: `t3.small`
  - `k8s_stack`: `k3s`
  - `provision_dockerhub`: `true`
- Run workflow

Esperar ~5-8 min. Al final del job, sección **"Resumen (IP + hints)"** muestra IP pública.

### 1.3 Anotar IP

Ejemplo: `34.237.218.17`. La vas a usar varias veces.

### 1.4 Validar SSH al nodo

Desde tu PC:
```bash
ssh -i C:/Users/drafi/labsuser.pem ubuntu@<IP>
```

Dentro nodo:
```bash
sudo k3s kubectl get nodes
sudo k3s kubectl get pods -A
```

Debe mostrar 1 nodo `Ready` + pods sistema (coredns, traefik, metrics-server, local-path-provisioner).

### 📸 EVIDENCIAS BLOQUE 1 (4 capturas)
- **01-primary-secrets.png** — Settings → Secrets and variables → Actions (lista nombres, valores ocultos)
- **02-workflow-ej-k8s-running.png** — Actions corriendo el workflow `Ej K8S desde alumno`
- **03-workflow-resumen-ip.png** — final del workflow, sección "EA2 — MV lista" con IP pública
- **04-aws-ec2-running.png** — consola AWS EC2 mostrando instancia `ea2-k8s-sandbox` Running con IP pública

### 📸 EVIDENCIAS BLOQUE 2 - cluster operativo (3 capturas)
- **05-ssh-conexion.png** — terminal con `ssh -i labsuser.pem ubuntu@<IP>` exitoso, prompt `ubuntu@ip-xxx`
- **06-kubectl-get-nodes.png** — `sudo k3s kubectl get nodes` mostrando nodo `Ready` + versión k3s
- **07-kubectl-pods-sistema.png** — `sudo k3s kubectl get pods -A` mostrando pods coredns, traefik, metrics-server, local-path-provisioner

---

## 2. Configurar Runner (app ramo)

### 2.1 Secrets + Variables

GitHub → repo Runner → Settings → Secrets and variables → Actions:

**Secrets:**
- `DOCKER_USERNAME` = `bvnjitsss`
- `DOCKER_PASSWORD` = token Docker Hub
- `EA2_SSH_PRIVATE_KEY` = contenido `labsuser.pem`

**Variables:**
- `K3S_SERVER_PUBLIC_IP` = IP del paso 1.3

### 2.2 Disparar primer deploy (sin nginx aún - para validar cadena)

```bash
cd C:/Users/drafi/AUY1104-BenjaminDuran-Runner
git tag v2.0.0
git push origin v2.0.0
```

> Si tag ya existe, usar siguiente: `v2.0.1`, etc.

### 2.3 Monitorear

GitHub → repo Runner → Actions → workflow `client.yaml`. 3 jobs esperados (~5-7 min):
1. `deps-and-test` ✅
2. `build-and-push` ✅
3. `deploy-to-k8s` ✅

### 2.4 Validar

```bash
# Desde nodo
sudo k3s kubectl get pods
sudo k3s kubectl get svc
curl http://<IP>:30090/health
```

Esperado: pod `demo-api-xxx` Running, service NodePort 30090, JSON respuesta `/health`.

### 📸 EVIDENCIAS BLOQUE 3 - demo-api (5 capturas)
- **08-runner-secrets.png** — Settings Runner mostrando los 3 secrets + variable `K3S_SERVER_PUBLIC_IP`
- **09-runner-tag-push.png** — terminal local con `git tag v2.0.0 && git push origin v2.0.0`
- **10-runner-actions-3jobs.png** — pestaña Actions con `client.yaml` y 3 jobs verdes (`deps-and-test`, `build-and-push`, `deploy-to-k8s`)
- **11-kubectl-demo-api.png** — desde nodo: `kubectl get pods` + `kubectl get svc` mostrando demo-api Running + NodePort 30090
- **12-curl-health.png** — `curl http://<IP>:30090/health` mostrando JSON respuesta

---

## 3. Agregar nginx al Runner (puerto 30080)

### 3.1 Crear manifests

Archivos ya presentes en `k8s/`:
- `k8s/nginx-deployment.yaml` (image `nginx:1.27-alpine`)
- `k8s/nginx-service.yaml` (NodePort 30080)

> Si no existen, crearlos copiando del repo actual.

### 3.2 Commit + tag

```bash
cd C:/Users/drafi/AUY1104-BenjaminDuran-Runner
git add k8s/nginx-deployment.yaml k8s/nginx-service.yaml
git commit -m "feat: agregar nginx NodePort 30080"
git tag v2.0.2
git push origin main --tags
```

### 3.3 Validar

```bash
# Desde nodo
sudo k3s kubectl get pods
sudo k3s kubectl get svc
curl http://<IP>:30080
```

Esperado: pod `nginx-xxx` Running, service nginx NodePort 30080, HTML "Welcome to nginx!".

### 📸 EVIDENCIAS BLOQUE 4 - nginx (4 capturas)
- **13-nginx-yaml-files.png** — VSCode/explorador mostrando `k8s/nginx-deployment.yaml` + `k8s/nginx-service.yaml` creados
- **14-nginx-tag-push.png** — terminal con `git tag v2.0.2 && git push origin main --tags`
- **15-actions-nginx-verde.png** — Actions del run nuevo con 3 jobs verdes
- **16-curl-nginx.png** — terminal con `curl http://<IP>:30080` mostrando HTML "Welcome to nginx!"

---

## 4. Crear repo Apache (puerto 30100)

Estructura ya creada en `C:/Users/drafi/AUY1104-BenjaminDuran-Apache/`:
```
Dockerfile              # httpd:2.4-alpine
index.html              # página con datos pareja
package.json            # test trivial
k8s/deployment.yaml     # apache-demo
k8s/service.yaml        # NodePort 30100
.github/workflows/client.yaml
README.md, .gitignore
```

### 4.1 Editar `index.html`

Reemplazar `[INTEGRANTE 2]` por nombre real (en nuestro caso: `Gastón Mardones`).

### 4.2 Crear repo en GitHub

github.com → New repository:
- Nombre: `AUY1104-BenjaminDuran-Apache`
- Public
- Sin README/gitignore/license

### 4.3 Init + push

```bash
cd C:/Users/drafi/AUY1104-BenjaminDuran-Apache
git init
git add .
git commit -m "feat: estructura inicial Apache EV2"
git branch -M main
git remote add origin https://github.com/BenjaminDuran/AUY1104-BenjaminDuran-Apache.git
git push -u origin main
```

### 4.4 Configurar secrets/variables

GitHub → repo Apache → Settings → Secrets and variables → Actions:

**Secrets:**
- `DOCKER_USERNAME` = `bvnjitsss`
- `DOCKER_PASSWORD` = mismo token
- `EA2_SSH_PRIVATE_KEY` = mismo contenido `labsuser.pem`

**Variables:**
- `K3S_SERVER_PUBLIC_IP` = `<IP>`

### 4.5 Disparar deploy

```bash
git tag v1.0.0
git push origin v1.0.0
```

### 4.6 Monitorear + validar

GitHub → Apache → Actions → 3 jobs verdes (~5-7 min).

```bash
# Desde nodo
sudo k3s kubectl get pods
sudo k3s kubectl get svc
curl http://<IP>:30100
```

Esperado: pod `apache-demo-xxx` Running, service NodePort 30100, HTML con nombres pareja.

### 📸 EVIDENCIAS BLOQUE 5 - Apache (6 capturas)
- **17-apache-repo-creado.png** — GitHub mostrando repo `AUY1104-BenjaminDuran-Apache` recién creado
- **18-apache-estructura.png** — VSCode mostrando estructura local (Dockerfile, index.html, k8s/, .github/workflows/, README.md)
- **19-apache-push-inicial.png** — terminal con `git push -u origin main` exitoso
- **20-apache-secrets.png** — Settings repo Apache con secrets + variable
- **21-apache-actions-verde.png** — Actions repo Apache con 3 jobs verdes
- **22-curl-apache.png** — terminal con `curl http://<IP>:30100` mostrando HTML con nombres pareja

---

## 5. Gotcha conocido: `:latest` + cambios en HTML/código

Si después de cambiar código, push y tag, **el navegador muestra versión vieja**:

**Causa**: `deployment.yaml` usa `image: bvnjitsss/apache-demo:latest`. `kubectl apply` ve mismo manifest → no reinicia pod. Imagen nueva existe en Hub pero pod sigue con la cacheada.

**Fix** (desde nodo k3s):
```bash
sudo k3s kubectl rollout restart deployment/apache-demo
sudo k3s kubectl rollout status deployment/apache-demo
```

`rollout restart` mata pod viejo → crea nuevo → como `imagePullPolicy: Always`, baja imagen fresca.

Aplica para `demo-api` también si cambias código sin cambiar tag de imagen en YAML.

---

## 6. Practicar rollback (evidencia obligatoria informe)

Desde nodo k3s:

```bash
# 1. Ver historial inicial (solo revisión 1)
sudo k3s kubectl rollout history deployment/demo-api

# 2. Forzar revisión 2 (cambio imagen a tag específico)
sudo k3s kubectl set image deployment/demo-api api=bvnjitsss/demo-api:v1.0.0
sudo k3s kubectl rollout status deployment/demo-api

# 3. Confirmar imagen cambió
sudo k3s kubectl describe deployment demo-api | grep "Image:"

# 4. Rollback (vuelve a :latest)
sudo k3s kubectl rollout undo deployment/demo-api
sudo k3s kubectl rollout status deployment/demo-api

# 5. Confirmar imagen volvió
sudo k3s kubectl describe deployment demo-api | grep "Image:"

# 6. Historial final (rev 2 + rev 3)
sudo k3s kubectl rollout history deployment/demo-api

# 7. Validar app sigue viva
curl http://<IP>:30090/health
```

### 📸 EVIDENCIAS BLOQUE 6 - Rollback (1 captura larga, scrolleable o pegada)
- **23-rollback-completo.png** — terminal con TODO el flujo:
  1. `kubectl rollout history deployment/demo-api` (rev 1)
  2. `kubectl set image deployment/demo-api api=bvnjitsss/demo-api:v1.0.0`
  3. `kubectl rollout status` (rolled out)
  4. `kubectl describe | grep Image:` (muestra v1.0.0)
  5. `kubectl rollout undo deployment/demo-api`
  6. `kubectl rollout status` (rolled out)
  7. `kubectl describe | grep Image:` (muestra latest de vuelta)
  8. `kubectl rollout history` (rev 2 + rev 3)
  9. `curl http://<IP>:30090/health` (200 OK, app viva)

---

## 7. Capturar evidencias generales

### 7.1 Desde nodo k3s

```bash
sudo k3s kubectl get nodes
sudo k3s kubectl get pods -A
sudo k3s kubectl get svc -A

curl http://<IP>:30080
curl http://<IP>:30090/health
curl http://<IP>:30100
```

### 7.2 Desde navegador

Abrir las 3 URLs:
- `http://<IP>:30080` → nginx welcome
- `http://<IP>:30090/health` → JSON `{"status":"ok"}` (o lo que devuelva)
- `http://<IP>:30100` → HTML Apache con nombres pareja

### 7.3 Desde GitHub Actions

Pestaña Actions de los 3 repos con runs verdes.

### 7.4 Desde Docker Hub

`hub.docker.com` → `bvnjitsss/demo-api` + `bvnjitsss/apache-demo` mostrando tags.

### 📸 EVIDENCIAS BLOQUE 7 - estado final (5 capturas)
- **24-cluster-completo.png** — terminal nodo con los 3 outputs (`get nodes`, `get pods -A`, `get svc -A`) en una sola captura
- **25-curl-3-servicios.png** — terminal con los 3 `curl` consecutivos
- **26-navegador-30080.png** — browser en `http://<IP>:30080` → página nginx
- **27-navegador-30090.png** — browser en `http://<IP>:30090/health` → JSON
- **28-navegador-30100.png** — browser en `http://<IP>:30100` → página Apache con nombres pareja

### 📸 EVIDENCIAS BLOQUE 8 - Docker Hub (2 capturas)
- **29-dockerhub-demo-api.png** — `hub.docker.com/r/bvnjitsss/demo-api/tags` mostrando tags publicados
- **30-dockerhub-apache.png** — `hub.docker.com/r/bvnjitsss/apache-demo/tags` mostrando tags

### 📸 EVIDENCIAS BLOQUE 9 - GitHub Actions historial (3 capturas)
- **31-actions-primary.png** — Primary → Actions → lista con `Ej K8S desde alumno` verde
- **32-actions-runner.png** — Runner → Actions → lista con varios `client.yaml` verdes
- **33-actions-apache.png** — Apache → Actions → lista con `client.yaml` verde

---

## 8. Informe PDF (4-6 págs)

### Estructura

1. **Resumen del trabajo**
   - Qué implementamos: 3 servicios en k3s con pipelines automáticas
   - Herramientas: GitHub Actions, Docker, Docker Hub, k3s, AWS EC2, Terraform (workflow profe)
   - Servicios: nginx (30080), demo-api ramo (30090), Apache (30100)
   - Relación con DevOps: ciclo completo CI/CD, IaC parcial, despliegue declarativo, rollback

2. **Repositorios**
   - Primary: `https://github.com/BenjaminDuran/AUY1104-BenjaminDuran-Primary` - recetas reutilizables
   - Runner: `https://github.com/BenjaminDuran/AUY1104-BenjaminDuran-Runner` - app ramo + nginx
   - Apache: `https://github.com/BenjaminDuran/AUY1104-BenjaminDuran-Apache` - servidor Apache

3. **Arquitectura** (ver diagrama sección 0 + `DEPLOY-FLOW.md`)
   - Trigger: push tag `v*.*.*`
   - Job 1: tests
   - Job 2: docker build + push Hub
   - Job 3: SSH + scp k8s/ + `kubectl apply`
   - Resultado: pod expuesto vía NodePort

4. **Evidencias técnicas** (screenshots paso 7)

5. **Tabla servicios**

   | Puerto | Servicio | URL | Imagen | Estado |
   |---|---|---|---|---|
   | 30080 | nginx | `http://<IP>:30080` | `nginx:1.27-alpine` (pública) | Funcionando |
   | 30090 | App ramo | `http://<IP>:30090/health` | `bvnjitsss/demo-api:latest` | Funcionando |
   | 30100 | Apache | `http://<IP>:30100` | `bvnjitsss/apache-demo:latest` | Funcionando |

6. **Rollback** (paso 6 + screenshot)
   - Comando: `kubectl rollout undo deployment/<nombre>`
   - Alternativa: editar `image: tag` en YAML + tag nuevo
   - Evidencia: historial mostrando rev 2 → rev 3 después del undo

7. **2 estrategias de despliegue** (sugerencia)
   - **Rolling Update** (default Kubernetes, conecta directo con el lab):
     - Qué: actualización gradual pod-por-pod
     - Cómo: k8s reemplaza réplicas una por una, manteniendo servicio vivo
     - Ventajas: zero downtime, simple
     - Desventajas: 2 versiones conviven brevemente
     - Riesgos: estado compartido (sesiones, DB schema incompatible)
     - Cuándo: APIs stateless, varias réplicas
     - Continuidad: alta (sin caída)
     - Empresa: API interna con 5 réplicas + readiness probes + `kubectl rollout undo` como red de seguridad

   - **Blue/Green**:
     - Qué: dos entornos idénticos, solo uno recibe tráfico
     - Cómo: deploy a "green" mientras "blue" sirve. Switch atómico (cambio Service selector).
     - Ventajas: rollback instantáneo, validación previa
     - Desventajas: doble costo infra
     - Riesgos: drift entre entornos
     - Cuándo: cambios riesgosos, releases grandes
     - Continuidad: máxima
     - Empresa: backend de pagos con SLO 99.99% — validar green con sintético + canary % antes del switch

8. **Aplicación empresarial** (sección 7 expandida)
   - Mismos puntos pero con: tipo app, usuarios afectados, monitoreo (Prometheus/Grafana), métricas (error rate, latency p95), automatización pipeline

---

## 9. Entregables finales (checklist)

- [ ] Informe PDF (4-6 págs)
- [ ] URL Primary
- [ ] URL Runner
- [ ] URL Apache
- [ ] URL Docker Hub `bvnjitsss/demo-api`
- [ ] URL Docker Hub `bvnjitsss/apache-demo`
- [ ] Links runs Actions exitosos (3)
- [ ] IP pública
- [ ] URLs validación (30080, 30090/health, 30100)
- [ ] Outputs `kubectl get nodes/pods -A/svc -A`
- [ ] Explicación + evidencia rollback
- [ ] Análisis 2 estrategias + aplicación empresarial

---

## 10. Bloqueador conocido: sesión Learner Lab expira

Cada ~4h:
1. Re-iniciar lab
2. Re-launchear EC2 (IP nueva)
3. Actualizar `K3S_SERVER_PUBLIC_IP` en **Runner + Apache**
4. Actualizar `AWS_*` en Primary (sesión nueva)
5. Re-disparar workflows con tag nuevo (`v2.0.3`, etc) o `workflow_dispatch`

**Antes de la presentación**: dejar levantado lab + verificar las 3 URLs responden 5 min antes.

---

## 11. Orden recomendado total

| Bloque | Tiempo aprox |
|---|---|
| 0 - Pre-requisitos | 10 min |
| 1 - Provisionar EC2 + k3s | 10 min |
| 2 - Runner secrets + deploy inicial | 15 min |
| 3 - Agregar nginx + deploy | 10 min |
| 4 - Repo Apache desde cero + deploy | 25 min |
| 5 - Fix `:latest` si aplica | 5 min |
| 6 - Practicar rollback + capturar | 10 min |
| 7 - Capturar todas las evidencias | 20 min |
| 8 - Escribir informe | 90-180 min |

**Total práctico**: ~1.5h-2h laboratorio + 2-3h informe.

---

## 12. Atajos útiles

```bash
# Re-ver IP actual EC2 (en consola AWS o último run del workflow)

# Conexión rápida SSH
ssh -i C:/Users/drafi/labsuser.pem ubuntu@<IP>

# Estado completo cluster (1 comando)
sudo k3s kubectl get all -A

# Re-disparar workflow sin tag nuevo
# GitHub → Actions → workflow → Run workflow → branch main

# Forzar restart pod (cuando :latest no se refresca)
sudo k3s kubectl rollout restart deployment/<nombre>
```

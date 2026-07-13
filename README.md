# TechMarket Orders — Despliegue Blue-Green con remediación automática

**Evaluación Final Transversal · AUY1104 Ciclo de Vida del Software II**
Benjamín Durán Bustos · Sección AUY1104-003D · Docente: Andrés Sánchez

Microservicio `demo-api` —que representa al servicio crítico **Orders** del caso
TechMarket— desplegado sobre Kubernetes con estrategia **Blue-Green**,
**validación de salud** previa al cambio de tráfico y **rollback automático**
ante cualquier fallo.

---

## 1. Arquitectura

### 1.1 Los dos repositorios

| Repositorio | Rol | Contiene |
|---|---|---|
| **AUY1104-BenjaminDuran-Runner** (este) | Cliente | La app, el Dockerfile, los tests y los manifiestos de Kubernetes. Su workflow solo *invoca* la plantilla |
| **AUY1104-BenjaminDuran-Primary** | Plantillas (SharedWorkflows) | La receta reutilizable de CI/CD que hace todo el trabajo |

El repositorio cliente **no contiene ni un comando de Docker o `kubectl`**. Su
workflow, [client.yaml](.github/workflows/client.yaml), son 27 líneas que solo
declaran *qué* desplegar y *dónde*; el *cómo* vive en la plantilla. Si mañana
nace otro microservicio, se copian esas 27 líneas y ya tiene CI/CD completo.

### 1.2 Dónde corre cada cosa

Hay **tres máquinas distintas** en juego, y confundirlas es la fuente habitual de
malentendidos:

```
┌──────────────────────┐   git push origin v2.4.1
│  Runner de GitHub    │   ← VM efímera en la nube de GitHub.
│  (ubuntu-latest)     │     Aquí corren los tests y el `docker build`.
└──────────┬───────────┘     Se destruye al terminar.
           │ docker push
           ▼
┌──────────────────────┐
│     Docker Hub       │   ← El registry. La imagen no viaja de GitHub
│  bvnjitsss/demo-api  │     al clúster: pasa por aquí.
└──────────┬───────────┘
           │ image pull
           ▼
┌──────────────────────────────────────────────────┐
│  EC2 (AWS) · Kubernetes K3s                      │
│                                                  │
│  El runner entra por SSH y ejecuta AQUÍ todos    │
│  los `kubectl` y todos los health checks. El     │
│  runner no tiene kubeconfig ni acceso al clúster.│
└──────────────────────────────────────────────────┘
```

> **Sobre EKS y ECR.** El clúster corre sobre **K3s en EC2** y el registry es
> **Docker Hub**. Ambos cumplen exactamente el mismo rol que Amazon EKS y Amazon
> ECR: K3s implementa la API estándar de Kubernetes, de modo que cada
> `Deployment`, `Service` y `kubectl patch` de este proyecto funcionaría sin
> cambios sobre EKS.

### 1.3 Los objetos en el clúster

```
              Service demo-api            (NodePort 30090)
              selector: {app: demo-api, version: green}
                          │
                          │  ← el tráfico va SOLO a un color
                          ▼
    ┌──────────────────────────────┐   ┌──────────────────────────────┐
    │  Deployment demo-api-green   │   │  Deployment demo-api-blue    │
    │  labels: version=green       │   │  labels: version=blue        │
    │  2 réplicas · v2.4.1         │   │  2 réplicas · v2.4.0         │
    │  ► SIRVIENDO USUARIOS        │   │  ► encendido, SIN tráfico    │
    └──────────────────────────────┘   └──────────────────────────────┘
                                          (la red de seguridad: revertir
                                           es devolverle el selector)
```

Los dos colores **coexisten**. Quién recibe tráfico lo decide una sola cosa: el
campo `selector.version` del Service.

---

## 2. La plantilla reutilizable

Toda la lógica vive en `deploy-api.yaml`, en el repo Primary. Se invoca con
`uses:` y tiene tres jobs encadenados, cada uno como compuerta del siguiente:

```
client.yaml (este repo)
    └─► deploy-api.yaml (Primary)
            │
            ├─ 1. deps-and-test     npm install + npm test
            │                       ✗ → la imagen NUNCA se construye
            │
            ├─ 2. build-and-push    docker build + push a Docker Hub
            │                       ✗ → nada llega al clúster
            │
            └─ 3. deploy-to-k8s     Blue-Green (§3) + rollback (§5)
```

### 2.1 Parametrización

Nada del proyecto está escrito dentro de la plantilla. Todo lo que varía se pasa
como input desde el repo cliente:

| Input / Secret | Origen | Para qué |
|---|---|---|
| `image-name` | `with:` en client.yaml | Nombre de la imagen en Docker Hub |
| `image-tag` | `${{ github.ref_name }}` — el tag de Git | Versión a construir y desplegar |
| `k3s-server-public-ip` | Variable de repo `K3S_SERVER_PUBLIC_IP` | A qué clúster desplegar |
| `DOCKER_USERNAME` / `DOCKER_PASSWORD` | Secrets | Publicar en el registry |
| `EA2_SSH_PRIVATE_KEY` | Secret | Entrar al nodo K3s |

Cambiar `k3s-server-public-ip` basta para desplegar el mismo código a otro
clúster sin tocar una línea. La IP va como **variable** y no como secreto porque
una IP pública no es un secreto — y así se puede leer en los logs al depurar.

### 2.2 Variables de entorno dinámicas inyectadas al clúster

El pipeline inyecta la versión al contenedor **en tiempo de despliegue**, sin
recompilar la imagen:

```bash
kubectl set image deployment/demo-api-$NUEVO api=bvnjitsss/demo-api:v2.4.1
kubectl set env   deployment/demo-api-$NUEVO APP_VERSION=v2.4.1
```

Sumado a `RELEASE_COLOR`, que va fijo en cada manifiesto, la app las devuelve en
`/health` ([src/lib/ejemplo.js](src/lib/ejemplo.js)):

```json
{"ok":true,"servicio":"auy1104-api-ejemplo","color":"green","version":"v2.4.1"}
```

Esto **no es cosmético**. Es lo que permite que la validación de salud confirme
que el pod que responde es realmente el release nuevo, y no el viejo (§4, punto
b). Y es lo que hace visible el switch durante la demostración: el JSON en
`http://<IP>:30090/health` pasa de `"color":"blue"` a `"color":"green"` sin
recargar nada más que la página.

### 2.3 Componentes externos

| Componente | Versión | Por qué este y no un script propio |
|---|---|---|
| `actions/checkout` | `@v4` | Oficial de GitHub. Maneja el `GITHUB_TOKEN` y el clonado correctamente |
| `docker/login-action` | `@v2` | Oficial de Docker. **Enmascara las credenciales en los logs** y limpia la sesión al terminar el job; un `docker login` manual las filtraría |
| `webfactory/ssh-agent` | `@v0.9.0` | De terceros. Fijada a una **versión exacta**, no a una rama móvil: maneja la clave privada del clúster y no queremos que su código cambie sin aviso |

Criterio general: **todo anclado a una versión**. Una action referenciada como
`@main` es una dependencia que puede cambiar bajo tus pies — inaceptable en algo
que toca credenciales.

---

## 3. La estrategia: Blue-Green

### 3.1 Qué cambió respecto de la versión anterior

Antes, `k8s/` tenía **un** Deployment y un Service con selector `app: demo-api`.
El pipeline hacía `kubectl apply -f k8s/` y Kubernetes reemplazaba los pods de
forma gradual: eso es un **Rolling Update** (la estrategia por defecto cuando el
Deployment no declara ninguna).

Ahora hay **dos** Deployments y el selector del Service incluye `version`:

| | Antes (Rolling Update) | Ahora (Blue-Green) |
|---|---|---|
| Deployments | `demo-api` | `demo-api-blue` + `demo-api-green` |
| Selector del Service | `{app: demo-api}` | `{app: demo-api, version: blue}` |
| Cómo se mueve el tráfico | Kubernetes reemplaza pods | `kubectl patch` del selector |
| Rollback | `kubectl rollout undo` | `kubectl patch` del selector |

Esa línea de más en el selector es **todo el cambio conceptual**. Con el selector
antiguo, cualquier pod con la etiqueta `app: demo-api` recibía tráfico, sin
importar su versión; los pods viejos y los nuevos convivían dentro del mismo
Service. Con `version`, el Service apunta a **un solo color a la vez**, y el color
inactivo puede correr en paralelo sin recibir una sola petición.

### 3.2 El flujo del despliegue

1. **Detectar el color activo.** Se le pregunta al clúster leyendo el selector:
   ```bash
   kubectl get service demo-api -o jsonpath='{.spec.selector.version}'
   ```
   El clúster es la única fuente de verdad; no hay archivo de estado que se pueda
   desincronizar. Si el Service no existe, es el primer despliegue y el color
   nuevo es `blue`.

2. **Desplegar el color inactivo** con la imagen recién publicada
   (`set image` + `set env` + `rollout status`). El Service sigue apuntando al
   color estable: **nadie ve la versión nueva todavía**.

3. **Validar su salud** (§4), golpeándolo directamente por su IP interna.

4. **El switch:**
   ```bash
   kubectl patch service demo-api -p \
     '{"spec":{"selector":{"app":"demo-api","version":"green"}}}'
   ```
   Kubernetes recalcula los endpoints del Service al instante. Como los pods del
   color nuevo **ya estaban `Ready`**, no existe ni un milisegundo sin backend
   disponible. De ahí el cero downtime.

5. **Verificar** sobre el tráfico real (NodePort 30090) que responde el color
   nuevo.

6. El color viejo **queda encendido** como red de seguridad.

### 3.3 Por qué Blue-Green y no otra

`Orders` es un servicio **crítico y transaccional**. Eso descarta las otras
opciones por razones concretas, no por gusto:

- **All-in-once / Recreate** implica una ventana con **cero pods disponibles**.
  Para un servicio de órdenes, cada segundo de esa ventana es una venta perdida.
- **Rolling Update** mezcla ambas versiones dentro del *mismo* Service: durante el
  reemplazo, una fracción de usuarios **sí le pega a la versión nueva antes de que
  nadie la haya validado**. Si está rota, esos usuarios ya vieron el error, y el
  `rollout undo` llega tarde.
- **Canary** acota el daño, pero no lo elimina: por diseño, un porcentaje de
  usuarios reales *es* el conejillo de indias. Además, sin un Ingress con pesos, el
  porcentaje de tráfico se aproxima con la proporción de réplicas, lo cual es
  impreciso. En un servicio que mueve dinero, exponer aunque sea al 5% a una
  versión no validada es una decisión de negocio, no técnica.
- **Blue-Green** es la única en la que **ningún usuario real toca la versión nueva
  hasta que pasó todas las validaciones**. El precio es tener dos entornos vivos a
  la vez; para `Orders`, ese costo es trivial frente al de una caída.

### 3.4 Análisis comparativo

| | All-in-once | Rolling Update | Canary | **Blue-Green** |
|---|---|---|---|---|
| **Uptime durante el deploy** | ❌ Downtime total | ✅ Sin downtime | ✅ Sin downtime | ✅ Sin downtime |
| **Usuarios expuestos a la versión sin validar** | 100 % | Fracción creciente | % acotado (5–20 %) | **0 %** |
| **Costo de infraestructura** | 1× | ~1,2× (`maxSurge`) | ~1,2× | **2×** mientras conviven |
| **Velocidad del rollback** | Redespliegue completo (minutos) | `rollout undo`: recrea pods (30–60 s) | Escalar el canary a 0 (segundos) | **`patch` del selector: milisegundos** |
| **¿El rollback causa downtime?** | Sí, otra vez | No, pero es lento | No | **No, y es instantáneo** |
| **Complejidad operativa** | Mínima | Baja | Alta (necesita Ingress con pesos para ser preciso) | Media |
| **Cuándo conviene** | Batch, mantención programada | Servicios internos tolerantes a fallos | Validar features con usuarios reales | **Servicios críticos donde el error no es opción** |

**Sobre el costo real en el clúster:** con 2 réplicas por color, tener ambos
colores vivos son 4 pods de una app Node mínima. En una `t3.small` es
despreciable. **Estamos comprando reversión instantánea con RAM barata** — ese es
el trade-off, y es un buen negocio.

---

## 4. La validación de salud

Es el guardia entre *"desplegado"* y *"recibiendo usuarios"*. Se ejecuta contra el
pod nuevo **por su IP interna**, que los usuarios no alcanzan porque el Service no
apunta a él. Si algo está mal, **el usuario nunca lo vio**.

Tres comprobaciones:

| | Qué comprueba | Qué fallo caza |
|---|---|---|
| **a) Disponibilidad** | `GET /health` responde `200` (con reintentos: el pod recién arrancó) | El pod no arranca, se cae, o el puerto está mal |
| **b) Identidad** | El JSON reporta `"color"` igual al color que se acaba de desplegar | Estar validando el pod **viejo** sin darse cuenta y dar por buena una versión que ni siquiera se probó |
| **c) Contrato funcional** | `GET /api/suma?a=2&b=3` devuelve `"resultado":5` | **Una regresión de lógica**: la app está "sana" pero responde mal |

> **(b) y (c) son las que un `rollout status` jamás detectaría.** Una app puede
> arrancar perfecto, quedar `Ready`, y devolver 500 o resultados incorrectos:
> Kubernetes la daría por buena. Ese era exactamente el hueco del pipeline
> anterior.
>
> Nótese además que **(c) es independiente de los tests unitarios**. Si un
> desarrollador introduce un bug *y ajusta el test para que pase* —cosa que ocurre—
> los tests se vuelven cómplices del bug. La comprobación de contra el pod real es
> la única línea de defensa que sobrevive a eso.

---

## 5. La remediación automática

Conviene distinguir el mecanismo de **detección** del de **acción**: son cosas
distintas y van separadas a propósito.

### 5.1 Detección — quién se da cuenta

| Capa | Mecanismo | Dónde está | Qué caza |
|---|---|---|---|
| 1 | `readinessProbe` sobre `/health` | [k8s/deployment-blue.yaml](k8s/deployment-blue.yaml) | El pod nunca pasa a `Ready` → `CrashLoopBackOff`, puerto errado, 500 al arrancar |
| 2 | `livenessProbe` sobre `/health` | mismo archivo | La app se cuelga **después** de arrancar → kubelet reinicia el contenedor. Esto es *auto-healing de Kubernetes*, distinto del rollback del pipeline |
| 3 | `kubectl rollout status --timeout=90s` | paso 2 del deploy | Traduce lo anterior en un fallo del pipeline. Caza `ImagePullBackOff` |
| 4 | **Validación de salud HTTP** (§4) | paso 3 del deploy | 500 y regresiones de lógica — **lo que las probes no ven** |
| 5 | Verificación post-switch | paso 5 del deploy | Un switch que no se confirma |

### 5.2 Acción — qué hace el sistema

Se dispara con **`if: failure()`**. No está condicionado a un paso concreto, así
que cubre el fallo de *cualquier* etapa. Esto es deliberado: no hay que enumerar
los modos de fallo por adelantado, de modo que un fallo imprevisto —uno que nadie
anticipó al escribir el pipeline— también queda cubierto.

El paso de rollback primero **diagnostica**: clasifica la causa raíz en lenguaje
humano (`ImagePullBackOff`, `CrashLoopBackOff`, `CreateContainerConfigError`, o
fallo de validación) e imprime los logs del contenedor. Después **remedia**:

**Caso A — el fallo ocurre ANTES del switch** (el escenario habitual):

El selector **nunca se movió**. El color estable jamás dejó de servir. La
remediación se reduce a escalar a 0 el color defectuoso y fallar el pipeline.

> **Impacto en usuarios: cero.** Nadie vio nunca la versión rota.
> *El rollback más rápido es el que no hace falta hacer.*

**Caso B — el fallo ocurre DESPUÉS del switch:**

```bash
kubectl patch service demo-api -p \
  '{"spec":{"selector":{"app":"demo-api","version":"blue"}}}'   # ← de vuelta al estable
```

Como el color viejo sigue corriendo intacto, el tráfico vuelve **al instante**. Es
un cambio de regla de enrutamiento, no un redespliegue.

**En ambos casos el rollback se verifica:** se consulta `/health` en el NodePort
30090 y se confirma que responde `200` con el color estable. *Un rollback que no se
comprueba es una suposición, no una remediación.*

Caso borde: si el color estable no existe (primer despliegue), no hay a dónde
volver. Se retira el color defectuoso y se falla, sin fingir una recuperación que
no ocurrió.

### 5.3 Notificación

El job queda en rojo en GitHub Actions, con el correo de notificación que eso
dispara, y el log del paso de rollback contiene el informe completo del incidente:
causa raíz, acción tomada y confirmación del estado final del servicio.

### 5.4 El flujo completo

```
git push origin v2.4.1
  │
  ├─ npm test ─────────────✗──► ALTO. La imagen nunca se construye.
  ├─ docker build + push ──✗──► ALTO. Nada llega al clúster.
  │
  └─ deploy Blue-Green   (dentro de la EC2, vía SSH)
       │
       │   Service ──► BLUE (v2.4.0)   ← usuarios, intactos todo el tiempo
       │
       ├─ 1. detectar color activo → blue
       ├─ 2. desplegar GREEN (v2.4.1), sin tráfico
       │        rollout status ──────────✗──┐  DETECCIÓN
       ├─ 3. VALIDACIÓN DE SALUD ─────────✗─┤  DETECCIÓN
       │        (200 · identidad · contrato) │
       │                                     ▼
       │                        ACCIÓN: scale green=0
       │                        El selector nunca se movió.
       │                        → Impacto en usuarios: CERO
       │
       ├─ 4. SWITCH: kubectl patch svc → version=green
       ├─ 5. verificar tráfico real ──────✗──► ACCIÓN: patch de vuelta a blue
       │                                       (blue seguía vivo → instantáneo)
       ▼
     ✅ GREEN sirviendo. BLUE queda encendido como red de seguridad.
```

### 5.5 Impacto en el negocio (MTTR)

| | Pipeline anterior (Rolling Update) | Pipeline actual (Blue-Green) |
|---|---|---|
| **Detección** | Solo `rollout status`. Un 500 pasaba desapercibido | 5 capas, incluida validación HTTP funcional |
| **Usuarios que ven el error** | Los que pegan a los pods nuevos durante el reemplazo | **Ninguno** (fallo pre-switch) |
| **MTTR** | `rollout undo` → recrear pods: **30–60 s** | `patch` del selector: **< 1 s** |
| **Intervención humana** | Ninguna | Ninguna |

Para TechMarket la traducción es concreta: **una imagen defectuosa ya no puede
tumbar Orders**. El peor caso pasó de ser *"downtime parcial de ~1 minuto mientras
se revierte"* a *"el deploy falla y nadie se entera, salvo el equipo"*.

---

## 6. Escenarios de error contemplados

| Escenario | Cómo se manifiesta | Quién lo detecta | Qué ocurre |
|---|---|---|---|
| Test roto | `npm test` falla | Job `deps-and-test` | La imagen no se construye |
| Dockerfile roto (imagen base inexistente) | `docker build` falla | Job `build-and-push` | Nada llega al clúster |
| Imagen o tag inexistente | `ImagePullBackOff` | `rollout status` | Rollback. El tráfico nunca se movió |
| Excepción al arrancar | `CrashLoopBackOff` | `readinessProbe` → `rollout status` | Rollback |
| Error de configuración (probe mal apuntada) | Pods `Running` pero nunca `Ready` | `rollout status` | Rollback |
| App responde 500 | Pod no pasa a `Ready` | `readinessProbe` + validación (a) | Rollback |
| **Regresión de lógica** | Pod `Ready` y sano, pero `/api/suma` responde mal | **Validación (c)** | Rollback **sin que el tráfico se mueva** |
| ConfigMap o Secret ausente | `CreateContainerConfigError` | `rollout status` | Rollback |
| App se cuelga ya en producción | Deja de responder tras estar viva | `livenessProbe` | **Kubernetes reinicia el pod solo** |

---

## 7. Cómo se opera

### 7.1 Desplegar

```bash
git tag v2.4.1
git push origin v2.4.1
```

El tag dispara el workflow. El color de destino se alterna solo: v2.4.0 → blue,
v2.4.1 → green, v2.4.2 → blue…

### 7.2 Configuración requerida

**Secrets** (Settings → Secrets and variables → Actions):

| Secret | Qué es |
|---|---|
| `DOCKER_USERNAME` | Usuario de Docker Hub |
| `DOCKER_PASSWORD` | Token de acceso de Docker Hub |
| `EA2_SSH_PRIVATE_KEY` | Clave privada SSH del nodo K3s |

**Variables** (misma pantalla, pestaña *Variables*):

| Variable | Qué es |
|---|---|
| `K3S_SERVER_PUBLIC_IP` | IP pública de la EC2 |

El Security Group debe tener abierto el **puerto 30090** (el NodePort del Service).

### 7.3 Inspeccionar el clúster

```bash
ssh ubuntu@<IP>

# ¿Qué color está sirviendo? (mirar la columna SELECTOR)
sudo k3s kubectl get svc demo-api -o wide

# Los dos colores conviviendo
sudo k3s kubectl get pods -l app=demo-api --label-columns=version

# Quién responde ahora mismo
curl -s http://127.0.0.1:30090/health
```

### 7.4 Rollback manual de emergencia

Si el rollback automático fallara, revertir es **un comando**:

```bash
sudo k3s kubectl patch service demo-api -p \
  '{"spec":{"selector":{"app":"demo-api","version":"blue"}}}'
```

---

## 8. La API

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/health` | Estado del servicio. **Devuelve el color y la versión que están sirviendo** |
| `GET` | `/api/saludo` | Saludo en JSON; query opcional `nombre` |
| `GET` | `/api/suma` | Suma `a` + `b`. **Es el contrato funcional que valida el pipeline** |
| `POST` | `/api/suma` | Igual, con los operandos en el cuerpo |
| `POST` | `/api/echo` | Devuelve el cuerpo recibido |

Cualquier otra ruta responde **404** con `{"error": "Ruta no encontrada"}`.

Ejecución local:

```bash
npm install
npm test      # Jest + supertest, con cobertura
npm start     # http://localhost:3000
```

Fuera del clúster, `/health` devuelve `"color":"local"` y `"version":"dev"`.

---

## 9. Estructura del repositorio

```
├── .github/workflows/
│   └── client.yaml            Invoca la plantilla. No contiene lógica de deploy.
├── k8s/
│   ├── deployment-blue.yaml   Color BLUE  (labels version=blue)
│   ├── deployment-green.yaml  Color GREEN (labels version=green)
│   └── service.yaml           EL INTERRUPTOR: su selector decide quién sirve
├── src/
│   ├── index.js               Rutas Express
│   └── lib/ejemplo.js         Lógica pura, incluido healthPayload()
├── tests/                     Jest + supertest
└── Dockerfile
```

La plantilla reutilizable vive en
[AUY1104-BenjaminDuran-Primary](https://github.com/BenjaminDuran/AUY1104-BenjaminDuran-Primary).

---

## 10. Declaración de uso de Inteligencia Artificial

Para el desarrollo de este trabajo se utilizó **Claude (Anthropic)** como
herramienta de apoyo en:

- La discusión de alternativas de diseño entre las estrategias Canary y
  Blue-Green, y de los criterios técnicos para justificar la elección.
- La redacción de los pasos del workflow de GitHub Actions y de los manifiestos de
  Kubernetes.
- La estructura y redacción de esta documentación.

El diseño de la solución, las decisiones de arquitectura, la verificación del
funcionamiento sobre el clúster real y la validación de todo el contenido aquí
presentado son responsabilidad del autor. Todo el código fue revisado, ejecutado y
comprobado antes de incorporarse al repositorio.

---

## 11. Referencias

Amazon Web Services. (2024). *Amazon Elastic Kubernetes Service: User guide*.
https://docs.aws.amazon.com/eks/latest/userguide/

Fowler, M. (2010, 1 de marzo). *BlueGreenDeployment*. martinfowler.com.
https://martinfowler.com/bliki/BlueGreenDeployment.html

GitHub. (2024). *Reusing workflows*. GitHub Docs.
https://docs.github.com/en/actions/using-workflows/reusing-workflows

Humble, J., & Farley, D. (2010). *Continuous delivery: Reliable software releases
through build, test, and deployment automation*. Addison-Wesley.

Kim, G., Humble, J., Debois, P., & Willis, J. (2016). *The DevOps handbook: How to
create world-class agility, reliability, and security in technology organizations*.
IT Revolution Press.

Kubernetes. (2024). *Configure liveness, readiness and startup probes*.
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

Kubernetes. (2024). *Service*.
https://kubernetes.io/docs/concepts/services-networking/service/

Rancher Labs. (2024). *K3s: Lightweight Kubernetes*. https://docs.k3s.io/

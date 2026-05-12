# Cómo funciona el deploy a Kubernetes (paso a paso)

## Concepto base

Tenemos **dos repos**:

- **Primary** (`SharedWorkflows`) → "biblioteca" con recetas reutilizables. NO tiene código de app.
- **Runner** (`SharedClient`) → repo del proyecto. Tiene código, Dockerfile, manifests K8s, y un workflow corto que **llama** a la receta del Primary.

Idea: la receta vive en un solo lugar (Primary). Cualquier repo cliente la invoca sin copiarse los pasos. Igual que importar una librería en código.

---

## ¿Qué dispara todo?

Cuando hacemos:

```bash
git tag v1.0.0
git push origin v1.0.0
```

El archivo `.github/workflows/client.yaml` en Runner tiene esto:

```yaml
on:
  push:
    tags:
      - 'v*.*.*'
```

Significa: "GitHub Actions, ejecuta este workflow **solo cuando alguien empuje un tag tipo vX.Y.Z**". El tag `v1.0.0` cumple → arranca.

---

## ¿Qué hace `client.yaml`?

Solo una cosa: **delegar**. Su único job es:

```yaml
jobs:
  usar-receta-compartida:
    uses: BenjaminDuran/AUY1104-BenjaminDuran-Primary/.github/workflows/deploy-api.yaml@main
    with:
      image-name: demo-api
      image-tag: ${{ github.ref_name }}      # = "v1.0.0"
      k3s-server-public-ip: ${{ vars.K3S_SERVER_PUBLIC_IP }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      EA2_SSH_PRIVATE_KEY: ${{ secrets.EA2_SSH_PRIVATE_KEY }}
```

Traducido a humano:

> "GitHub, ve al repo Primary, rama main, ejecuta el workflow `deploy-api.yaml`. Pásale estos tres valores (`with`) y estos tres secretos."

`uses:` es el equivalente a un `import` o `include` de otro workflow.

Los **secretos** y **variables** que pasa salen del Runner (Settings → Secrets and variables → Actions). Por eso configuramos:

- `DOCKER_USERNAME`, `DOCKER_PASSWORD`, `EA2_SSH_PRIVATE_KEY` (secrets)
- `K3S_SERVER_PUBLIC_IP` (variable — no es secreto, es solo una IP)

---

## ¿Qué hace `deploy-api.yaml` en Primary?

Es la "receta real". Tiene **3 jobs encadenados**. Si uno falla, los siguientes no corren (eso es lo que hace `needs:`).

### Job 1: `deps-and-test`

```yaml
- uses: actions/checkout@v4    # baja código del repo que invocó
- run: npm install
- run: npm test
```

Trae el código del Runner (no del Primary), instala dependencias Node y corre los tests. Si los tests fallan → todo se detiene. No queremos publicar imagen rota.

### Job 2: `build-and-push`

```yaml
needs: deps-and-test           # solo corre si el job anterior pasó
- uses: actions/checkout@v4
- uses: docker/login-action@v2 # login a Docker Hub con DOCKER_USERNAME/PASSWORD
- run: |
    IMAGEN_COMPLETA="${{ secrets.DOCKER_USERNAME }}/${{ inputs.image-name }}"
    docker build \
      -t "${IMAGEN_COMPLETA}:${{ inputs.image-tag }}" \
      -t "${IMAGEN_COMPLETA}:latest" \
      .
    docker push --all-tags "${IMAGEN_COMPLETA}"
```

Qué hace cada línea:

1. Login a Docker Hub (cuenta `bvnjitsss`)
2. Construye la imagen Docker usando el `Dockerfile` del Runner
3. La etiqueta DOS veces:
   - `bvnjitsss/demo-api:v1.0.0` (versión específica)
   - `bvnjitsss/demo-api:latest` (siempre apunta a la última)
4. Sube ambas tags al Docker Hub

Resultado: cualquier máquina del mundo puede descargar la imagen con `docker pull bvnjitsss/demo-api:v1.0.0`.

### Job 3: `deploy-to-k8s` (el más importante)

```yaml
needs: build-and-push
- uses: webfactory/ssh-agent@v0.9.0
  with:
    ssh-private-key: ${{ secrets.EA2_SSH_PRIVATE_KEY }}
- run: ssh ubuntu@${{ inputs.k3s-server-public-ip }} 'echo ok'
- uses: actions/checkout@v4         # baja código del Runner OTRA VEZ
- run: |
    test -d k8s || { echo "::error::Falta carpeta k8s/"; exit 1; }
    ssh ubuntu@<IP> 'sudo rm -rf /tmp/k8s-deploy'
    scp -r k8s ubuntu@<IP>:/tmp/k8s-deploy
    ssh ubuntu@<IP> 'sudo k3s kubectl apply -f /tmp/k8s-deploy/'
- run: ssh ubuntu@<IP> 'sudo k3s kubectl get pods -o wide'
```

**Paso por paso:**

**a) Cargar la clave SSH en memoria del runner**

```yaml
uses: webfactory/ssh-agent@v0.9.0
```

La clave `EA2_SSH_PRIVATE_KEY` (guardada en secrets) se carga en un agente SSH temporal en la VM que corre el workflow. Así los siguientes pasos pueden hacer `ssh ...` sin pedir password.

**b) Probar conexión**

```bash
ssh ubuntu@<IP_K3S> 'echo ok'
```

Conecta al nodo EC2 y ejecuta `echo ok`. Si falla, todo el job falla. Validación temprana.

**c) Bajar el código del Runner**

```yaml
uses: actions/checkout@v4
```

Trae los archivos del Runner (incluyendo la carpeta `k8s/`) a la VM temporal de GitHub Actions.

**d) Verificar que existe carpeta `k8s/`**

```bash
test -d k8s || { echo "::error::Falta carpeta k8s/"; exit 1; }
```

Si no existe, aborta con error claro.

**e) Limpiar versión anterior en el nodo**

```bash
ssh ubuntu@<IP> 'sudo rm -rf /tmp/k8s-deploy'
```

Borra la carpeta `/tmp/k8s-deploy` vieja en el nodo k3s. Garantiza que si quitamos un archivo, no queda colgado.

**f) Copiar manifests al nodo**

```bash
scp -r k8s ubuntu@<IP>:/tmp/k8s-deploy
```

`scp` = "secure copy". Copia la carpeta `k8s/` completa (con `deployment.yaml`, `service.yaml`, lo que sea) desde la VM de Actions al servidor k3s, en `/tmp/k8s-deploy`.

**g) Aplicar al clúster Kubernetes**

```bash
ssh ubuntu@<IP> 'sudo k3s kubectl apply -f /tmp/k8s-deploy/'
```

Conecta al nodo y corre `kubectl apply` apuntando a la carpeta entera. **`kubectl apply -f <carpeta>/` aplica TODOS los archivos `.yaml` que estén ahí.**

Esto significa:

- Si hay `deployment.yaml` → crea/actualiza Deployment `demo-api`
- Si hay `service.yaml` → crea/actualiza Service `demo-api`
- Si agregamos `nginx-deployment.yaml` → crea/actualiza Deployment `nginx`
- Si agregamos `nginx-service.yaml` → crea/actualiza Service `nginx`

**`apply` es declarativo:** describimos el "estado deseado" (1 réplica de demo-api con imagen X) y Kubernetes hace lo necesario para llegar a ese estado. Si ya existe igual → no hace nada. Si difiere → actualiza. Si no existe → crea.

**h) Verificar resultado**

```bash
ssh ubuntu@<IP> 'sudo k3s kubectl get pods -o wide'
```

Lista los pods del clúster para confirmar que están corriendo.

---

## ¿Qué pasa en el nodo k3s cuando llega el `apply`?

Imaginamos el clúster como un "sistema operativo de contenedores":

1. **`kubectl apply -f deployment.yaml`** → le dice al API server de Kubernetes: "quiero 1 pod corriendo la imagen `bvnjitsss/demo-api:latest` con puerto 3000 expuesto"
2. El **scheduler** decide en qué nodo correrlo (en k3s de 1 nodo, siempre el mismo)
3. El **kubelet** del nodo descarga la imagen de Docker Hub (`docker pull bvnjitsss/demo-api:latest`) y arranca el contenedor
4. Las **probes** chequean `/health` cada cierto tiempo. Si falla → reinicia el pod
5. **`kubectl apply -f service.yaml`** → crea un "puerto fijo" 30090 en el nodo que reenvía tráfico al pod (NodePort)

Resultado: `http://<IP_nodo>:30090/health` → llega al pod → responde 200.

---

## Resumen del flujo completo

```
Vos hacés:
  git tag v1.0.1
  git push origin v1.0.1
        │
        ▼
GitHub detecta tag v*.*.* en Runner
        │
        ▼
Ejecuta .github/workflows/client.yaml (Runner)
        │
        │ uses: Primary/.../deploy-api.yaml
        ▼
Ejecuta deploy-api.yaml (Primary) con inputs+secrets del Runner
        │
        ├─ Job 1: tests (Runner code)        ──┐
        │                                       │ falla → STOP
        ├─ Job 2: docker build + push    ◄──── pasa
        │   • Docker Hub: bvnjitsss/demo-api:v1.0.1
        │   • Docker Hub: bvnjitsss/demo-api:latest
        │
        ├─ Job 3: deploy a k3s
        │   1. SSH al nodo EC2 (IP de K3S_SERVER_PUBLIC_IP)
        │   2. scp carpeta k8s/ → /tmp/k8s-deploy
        │   3. kubectl apply -f /tmp/k8s-deploy/
        │   4. kubectl get pods (verificación)
        │
        ▼
Pod nuevo corriendo en el nodo, accesible en :30090
```

---

## Por qué este diseño

- **Reutilizable:** muchos repos cliente pueden usar la misma receta sin copiarla
- **Centralizado:** si cambia la forma de buildear (ej. agregar lint), tocamos solo Primary
- **Declarativo:** el estado del clúster está en Git (la carpeta `k8s/`). Si alguien rompe algo en el clúster a mano, el siguiente tag deploy lo arregla
- **Seguro:** secretos viven en cada repo, no en código
- **Automático:** push tag → 5-10 min → cambio en producción

---

## Frase mental para clase

> "Cuando empujo un tag, GitHub copia la carpeta `k8s/` al servidor por SSH y le dice a Kubernetes que aplique todos los YAML. Si agrego un manifest, se aplica también automáticamente."

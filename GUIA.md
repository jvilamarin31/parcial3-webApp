# 🚀 Guía de Despliegue y Pruebas - Requerimiento 3.1
**Proyecto 3: Infraestructura resiliente con Kubernetes**  
*Despliegue de microservicios con HPA y Probes.*

## 📋 1. Descripción del Requerimiento
**R3.1:** Desplegar una aplicación de mínimo 3 microservicios en Kubernetes, cada uno con mínimo 2 réplicas, HPA configurado y liveness/readiness probes.
*   **Criterio de aceptación:** Aplicación funcional; HPA demostrado bajo carga artificial con `k6` o `wrk`.

---

## 🛠️ 2. Requisitos Previos (Instalación de Herramientas)
Antes de empezar, asegúrate de tener **Docker** instalado y corriendo. Luego, instala las siguientes herramientas en tu máquina (probado en Linux Mint / Ubuntu):

### 2.1 Instalar Minikube
```bash
# 1. Descargar el archivo .deb oficial
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb

# 2. Instalar el paquete en el sistema
sudo dpkg -i minikube_latest_amd64.deb
```

### 2.2 Instalar kubectl
```bash
# Paso 1: Instalar dependencias previas
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# Paso 2: Crear el directorio de claves y agregar la GPG key de Kubernetes
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Paso 3: Agregar el repositorio oficial a la lista de apt
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list

# Paso 4: Actualizar la lista de paquetes e instalar kubectl
sudo apt-get update
sudo apt-get install -y kubectl
```

### 2.3 Instalar wrk (Herramienta para pruebas de carga)
```bash
sudo apt install wrk
```

---

## 📂 3. Estructura de los Archivos de Kubernetes
En la carpeta `k8s/` del repositorio se encuentran los manifiestos:
*   `01-databases.yaml`: Despliega las 3 bases de datos MySQL (con volúmenes persistentes) usando imágenes personalizadas de Docker Hub que ya incluyen los scripts `.sql`.
*   `02-microservices.yaml`: Despliega los 4 servicios (frontend, users, products, orders) con 2 réplicas cada uno, configurando variables de entorno, límites de CPU/Memoria y los *Probes* (`/health`).
*   `03-hpa.yaml`: Configura los Autoescaladores para cada servicio (mínimo 2, máximo 5 pods, escalando al 70% de uso de CPU).

---

## 🚀 4. Paso a Paso para el Despliegue

### 4.1 Iniciar el clúster de Minikube
*(Nota: El flag `--force-systemd=true` es crucial en Linux para evitar errores de conexión con el API Server).*
```bash
minikube start --driver=docker --cpus=6 --memory=7168 --force-systemd=true
```

### 4.2 Habilitar el servidor de métricas
Este complemento es **obligatorio** para que el HPA pueda leer el consumo de CPU.
```bash
minikube addons enable metrics-server
```

### 4.3 Aplicar los manifiestos
Ejecuta los comandos en este orden exacto:
```bash
kubectl apply -f k8s/01-databases.yaml
kubectl apply -f k8s/02-microservices.yaml
kubectl apply -f k8s/03-hpa.yaml
```

---

## 🔍 5. Verificación del Despliegue

### 5.1 Verificar los Pods
Ejecuta:
```bash
kubectl get pods
```
**¿Qué esperar?** 
*   Los microservicios deben mostrar `READY 2/2` (o `1/1` dependiendo de si están escalando) y estado `Running`.
*   Las bases de datos deben mostrar `READY 1/1`.

### 5.2 Verificar el Autoescalador (HPA)
Ejecuta:
```bash
kubectl get hpa
```
**¿Qué esperar?** 
*   Debes ver 4 HPAs (frontend, users, products, orders).
*   En la columna `TARGETS` debe aparecer el porcentaje actual (ej. `1%/70%` o `2%/70%`). Si sale `<unknown>`, espera 1 minuto y vuelve a consultar.

### 5.3 Probar la Aplicación Web
Abre un túnel hacia el frontend:
```bash
kubectl port-forward svc/frontend 8080:5001
```
Abre tu navegador en **`http://localhost:8080`**.
*   **Usuario:** `juan` / **Contraseña:** `123`
*   Navega por el catálogo y crea una orden para verificar que la comunicación entre microservicios y las bases de datos funciona perfectamente.

---

## 🔥 6. Prueba de Estrés (Demostración del HPA)
Para demostrar que el HPA funciona (Criterio de aceptación R3.1), haremos una prueba de carga.

1.  **Terminal 1 (Monitoreo):** Deja el port-forward corriendo y abre otra terminal. Ejecuta:
    ```bash
    kubectl get hpa -w
    ```
    *(El `-w` deja el comando en modo "watch" o tiempo real).*

2.  **Terminal 2 (Carga):** Abre una tercera terminal y lanza el ataque de peticiones:
    ```bash
    # -t4 (4 hilos), -c100 (100 conexiones concurrentes), -d60s (duración 60 segundos)
    wrk -t4 -c100 -d60s http://localhost:8080/api/products
    ```

3.  **Observa la magia:** 
    *   En la Terminal 1, verás cómo el `TARGETS` del frontend y microproducts sube (por encima del 70%).
    *   Verás cómo la columna `REPLICAS` aumenta de `2` a `3`, `4` o `5` en tiempo real.
    *   *(Toma una captura de pantalla de esto, es evidencia clave para la sustentación).*

---

## 🛑 7. Apagar el Entorno
Cuando termines de probar, puedes detener el clúster para liberar los recursos de tu máquina:
```bash
minikube stop
```
Si deseas borrar todo por completo y empezar de cero:
```bash
minikube delete
```
# 🛡️ Requerimiento 3.2: Service Mesh (Istio), mTLS y Kiali

## 1. Requisitos Previos
Antes de desplegar Istio en el clúster, el entorno local de Kubernetes debe estar activo y contar con las herramientas de análisis instaladas.

```bash
# Iniciar el clúster si se encuentra detenido
minikube start --driver=docker --cpus=6 --memory=7168 --force-systemd=true

# Instalar jq (necesario para leer los JSON de Istio y extraer los certificados)
sudo apt update && sudo apt install -y jq
```

---

## 2. Instalación de Istio y Herramientas de Visualización

La instalación se divide en la descarga de la CLI en el host (sin necesidad de tener el clúster iniciado) y el despliegue del plano de control e instrumental en Minikube (requiere el clúster activo).

### A. Descargar e instalar la CLI de Istio (Local en el Host)
```bash
# Descargar e instalar el cliente de Istio de forma global en /usr/local/bin
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
sudo cp bin/istioctl /usr/local/bin/
cd .. && rm -rf istio-*
```

### B. Instalar Istio y Add-ons (En el Clúster - Requiere Minikube activo)
```bash
# 1. Instalar el plano de control y gateways de Istio
istioctl install --set profile=demo -y

# 2. Instalar los dashboards de observabilidad (Kiali, Prometheus, Grafana, Jaeger)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.31/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.31/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.31/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.31/samples/addons/jaeger.yaml
```

---

## 3. Habilitar el Sidecar e Inyectar Istio

Para que Istio gestione el tráfico, se habilita la inyección automática del proxy Envoy en los Pods del namespace `default`:

```bash
# 1. Habilitar la inyección automática en el namespace default
kubectl label namespace default istio-injection=enabled

# 2. Reiniciar los microservicios para inyectar el proxy de Istio
kubectl rollout restart deployment microusers microproducts microorders frontend

# 3. Verificar que los pods digan READY 2/2 (App + Proxy Envoy)
kubectl get pods
```

---

## 4. Configurar mTLS en Modo STRICT

Por defecto, Istio opera en modo `PERMISSIVE`. Para forzar que **todo** el tráfico de la red interna se transmita de forma estrictamente cifrada mediante Mutual TLS, se aplica la siguiente política:

```bash
kubectl apply -f k8s/04-mtls-strict.yaml
```

*Verificación de estado de mTLS en un pod específico:*
```bash
istioctl x describe pod <nombre-del-pod>
```
(El resultado debe mostrar Workload mTLS mode: STRICT)

### 4.1. Inspección de Certificados mTLS (Evidencia de la Rúbrica)
Para demostrar que el tráfico está realmente cifrado, extraemos e inspeccionamos el certificado inyectado en el proxy Envoy:

```bash
# Reemplazar <nombre-del-pod> por el nombre real del pod
istioctl proxy-config secret <nombre-del-pod> -o json | \
jq -r '.dynamicActiveSecrets[] | select(.name=="default") | .secret.tlsCertificate.certificateChain.inlineBytes' | \
base64 -d | openssl x509 -text -noout
```

---

## 5. Acceder a Kiali y Visualizar la Topología

Para verificar el correcto flujo de la malla de servicios y los certificados mTLS:

```bash
# Abrir el dashboard de Kiali en el navegador
istioctl dashboard kiali
```

### Pasos de visualización:
1. En la interfaz web de Kiali, ir a la sección **Traffic Graph**.
2. En la barra superior, seleccionar el namespace **`default`** y ajustar el filtro de tiempo a **Last 1m**.
3. Realizar peticiones navegando o comprando en la aplicación (`http://localhost:8080`) para observar las conexiones en tiempo real, las métricas de tráfico y los candados de seguridad activos en la malla.

# 🛡️ Requerimiento 3.3: Circuit Breaker (Outlier Detection)

## ⚠️ Nota Importante sobre la Configuración
Istio tiene una regla de precedencia estricta: si un `VirtualService` inyecta fallos (como los retries del R3.4), este puede interferir con la detección de errores del Circuit Breaker. 
**Para probar este requerimiento correctamente, es obligatorio NO tener aplicado el archivo `06-resiliencia-virtual-service.yaml`.**

```bash
# Asegurarse de que el archivo de retries NO esté aplicado
kubectl delete -f k8s/06-resiliencia-virtual-service.yaml --ignore-not-found
```

---

## 1. Aplicar las Reglas de Resiliencia
Vamos a aplicar el `DestinationRule` que configura el Circuit Breaker y un `EnvoyFilter` (ubicado en la carpeta de pruebas) que inyecta errores 503 **directamente en el servidor**.

```bash
kubectl apply -f k8s/05-circuit-breaker.yaml
kubectl apply -f k8s/testing/98-envoy-fault-server.yaml
```

---

## 2. Preparar el Entorno Visual
Abre una terminal para exponer el frontend y otra para Kiali:

```bash
# Terminal 1: Exponer el frontend
kubectl port-forward svc/frontend 8080:5001

# Terminal 2: Abrir Kiali
istioctl dashboard kiali
```

---

## 3. El "Antes" (Estado Saludable)
Verifica el estado actual de los pods de `microproducts` desde la perspectiva del `frontend`:

```bash
istioctl proxy-config endpoint $(kubectl get pod -l app=frontend -o jsonpath="{.items[0].metadata.name}") --cluster "outbound|5003||microproducts.default.svc.cluster.local"
```

> **¿Qué esperar?** Verás las dos IPs de los pods con el estado `OUTLIER CHECK: OK`. 

---

## 4. Generar el Caos
1. Ve a tu navegador en `http://localhost:8080`.
2. Inicia sesión y entra a la sección de **Productos**.
3. Haz clic en el botón **"Get Products"** (o en **"Create Product"**) unas 10 a 15 veces rápidamente.
4. *(Nota: Presiona **F12** en tu navegador, ve a la pestaña **Network (Red)** y verás las peticiones fallando con el código HTTP **503**).*

---

## 5. El "Después" (Circuit Breaker Activado)
Vuelve a ejecutar **exactamente el mismo comando** del Paso 3:

```bash
istioctl proxy-config endpoint $(kubectl get pod -l app=frontend -o jsonpath="{.items[0].metadata.name}") --cluster "outbound|5003||microproducts.default.svc.cluster.local"
```

> **¿Qué esperar?** Al menos uno de los pods ahora mostrará `OUTLIER CHECK: FAILED`. Esto significa que Istio detectó los 5 errores consecutivos y expulsó temporalmente al pod del balanceo de carga.

---

## 6. Evidencia Numérica (Métricas del Proxy)
Para obtener la prueba definitiva del aislamiento de instancias, ejecuta el siguiente comando:

```bash
kubectl exec -it $(kubectl get pod -l app=frontend -o jsonpath="{.items[0].metadata.name}") -c istio-proxy -- pilot-agent request GET stats | grep -E "ejections_enforced_total|ejections_consecutive_5xx|ejections_enforced_consecutive_5xx|ejections_overflow"
```

> **¿Qué esperar?** Verás contadores mayores a 0, por ejemplo:
> * `ejections_enforced_total: 3` (El pod fue expulsado 3 veces).
> * `ejections_consecutive_5xx: 5` (Se disparó la expulsión al alcanzar 5 errores 503 consecutivos).

---

## 7. Limpieza del Entorno
Una vez capturadas las evidencias, es **obligatorio** eliminar el inyector de fallos:

```bash
kubectl delete -f k8s/testing/98-envoy-fault-server.yaml
```

---

# ⏱️ Requerimiento 3.4: Retries, Timeouts y Backoff Exponencial

## 1. Configuración Aplicada
Se aplica el `VirtualService` que configura un timeout global de 5 segundos, 3 reintentos con un timeout de 1 segundo por intento. Istio aplica **backoff exponencial** de forma automática entre reintentos.

```bash
# Asegurarse de que el Circuit Breaker (05) no interfiera
kubectl delete -f k8s/05-circuit-breaker.yaml --ignore-not-found

# Aplicar las reglas de reintentos
kubectl apply -f k8s/06-resiliencia-virtual-service.yaml

# Aplicar el inyector de retraso (delay) de 2 segundos
kubectl apply -f k8s/testing/98-envoy-fault-delay.yaml
```

---

## 2. Preparar el Entorno Visual
Abre una terminal para exponer el frontend y otra para Kiali:

```bash
# Terminal 1: Exponer el frontend
kubectl port-forward svc/frontend 8080:5001

# Terminal 2: Abrir Kiali
istioctl dashboard kiali
```

---

## 3. Prueba de Fault Injection (Delay y Timeout)
1. Ve a tu navegador (`http://localhost:8080`) y presiona **F12** para abrir las herramientas de desarrollo.
2. Ve a la pestaña **Network (Red)**.
3. Entra a la sección de **Productos** y haz clic en **"Get Products"**.
4. **Comportamiento:** Verás que la petición tarda varios segundos (aprox. 4000ms) y finalmente devuelve un error **`504 Gateway Timeout`** (o `200 OK` si logra responder en un reintento). Esto ocurre porque el delay (2s) supera el `perTryTimeout` (1s), forzando a Istio a cortar y reintentar la petición.

---

## 4. Evidencia Numérica (Métricas de Reintentos)
Para demostrar que Istio está reintentando las peticiones internamente, ejecuta este comando:

```bash
kubectl exec -it $(kubectl get pod -l app=frontend -o jsonpath="{.items[0].metadata.name}") -c istio-proxy -- pilot-agent request GET stats | grep "microproducts.*upstream_rq_retry"
```

> **¿Qué esperar?** Verás contadores que aumentan con cada clic que hiciste en el navegador:
> * `...upstream_rq_retry: 95` (Número total de reintentos).
> * `...upstream_rq_retry_backoff_exponential: 95` (**Prueba clave:** demuestra que se aplicó espera exponencial entre reintentos).
> * `...upstream_rq_retry_limit_exceeded: 31` (Peticiones que fallaron definitivamente tras agotar los 3 intentos).

---

## 5. Evidencia Visual en Kiali
En Kiali -> **Traffic Graph**, con el filtro en `Last 1m`, verás las líneas de conexión hacia `microproducts` en color **rojo** y el porcentaje de error elevado, demostrando el estrés al que fue sometido el sistema.

---

## 6. Limpieza del Entorno
Borra el inyector de delay y restaura las configuraciones de producción:

```bash
kubectl delete -f k8s/testing/98-envoy-fault-delay.yaml
kubectl apply -f k8s/05-circuit-breaker.yaml
```

# 💥 Requerimiento 3.5: Ingeniería del Caos con LitmusChaos

## 1. Requisitos Previos
Para este requerimiento, utilizaremos LitmusChaos sin su interfaz gráfica (ChaosCenter) para optimizar los recursos de nuestro Minikube (evitando el consumo de MongoDB). Es necesario tener instalado `helm` en el host.

```bash
# Instalar Helm en el host (si no se tiene)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh

# Añadir el repositorio de Litmus
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
```

---

## 2. Instalación del Operador de Litmus y los Experimentos

Vamos a instalar únicamente el "Core" (el motor de ejecución) y descargar las 5 "recetas" de caos requeridas por la rúbrica.

```bash
# 1. Crear el namespace y desplegar el operador (sin la UI pesada)
kubectl create namespace litmus
helm install chaos-core litmuschaos/litmus-core --namespace litmus

# 2. Aplicar los permisos necesarios (RBAC)
kubectl apply -f k8s/testing/litmus-rbac.yaml

# 3. Descargar los 5 experimentos de caos en el namespace de la app
for fault in pod-delete pod-network-latency pod-cpu-hog pod-network-loss pod-memory-hog; do
  kubectl apply -n default -f "https://hub.litmuschaos.io/api/chaos/master?file=faults/kubernetes/${fault}/fault.yaml"
done

# 4. Verificar que los 5 experimentos estén listos
kubectl get chaosexperiments -n default
```

---

## 3. Experimento 1: Pod Delete (Eliminación de Pods)

**Hipótesis:** *"Si se elimina un pod del microservicio `microproducts`, Kubernetes lo recreará automáticamente gracias a las réplicas configuradas, manteniendo el servicio 100% disponible para el usuario."*

### A. Configuración (ChaosEngine)
En el repositorio ya se encuentra creado el archivo `k8s/testing/chaos-pod-delete.yaml`, el cual contiene la configuración para inyectar este fallo. Este archivo le indica a Litmus que ataque a los pods del deployment `microproducts` durante 60 segundos, eliminando un pod cada 10 segundos.

### B. Ejecutar el Caos y Monitorear
Aplica el archivo y abre terminales de monitoreo para ver la magia de la auto-recuperación de Kubernetes:

```bash
# Terminal 1: Lanzar el caos
kubectl apply -f k8s/testing/chaos-pod-delete.yaml

# Terminal 2: Ver los pods muriendo y renaciendo
kubectl get pods -l app=microproducts -w

# Terminal 3: Ver el estado del motor de caos
kubectl get chaosengine -w
```
*Mientras se ejecuta (verás cómo los pods pasan a `Terminating` y luego se crean nuevos en `Pending` -> `Running`), ingresa al frontend (`http://localhost:8080`) y haz clic en "Get Products". La aplicación debe seguir respondiendo sin caerse.*

### C. Obtener el Veredicto
Espera a que el experimento termine por completo (el `chaosengine` pasará a estado `completed`, esto toma aprox. 1 a 2 minutos). Luego ejecuta este comando para obtener el resultado oficial de Litmus:

```bash
kubectl get chaosresult pod-delete-engine-pod-delete -o yaml
```

**¿Qué buscar en la salida?** 
* `phase: Completed`
* `verdict: Pass`
* `probeSuccessPercentage: "100"`

### D. Limpieza
Una vez terminado y documentado, borra el motor de caos para dejar todo listo para el próximo experimento:

```bash
kubectl delete -f k8s/testing/chaos-pod-delete.yaml
```

## 3. Experimento 2: Network Delay (Latencia de Red)

**Hipótesis:** *"Al inyectar 2 segundos de latencia en el microservicio `microorders`, las peticiones experimentarán demoras (Timeouts). Sin embargo, gracias a las políticas de resiliencia configuradas en Istio (Reintentos y Circuit Breaker), el sistema manejará la falla, expulsará temporalmente los pods afectados y se recuperará automáticamente sin colapsar."*

### A. Comparación con el Steady State (Estado Estacionario)
*   **Steady State (Antes del caos):** Las peticiones a `/api/orders` responden en un promedio de **9ms a 10ms** con código HTTP **200**.
*   **Estado durante el caos:** Las peticiones registran un tiempo de respuesta de **~5000ms** devolviendo errores **504** (Gateway Timeout), seguido de fallos inmediatos **503** (Service Unavailable por Circuit Breaker).
*   **Estado posterior al caos (Recuperación):** Las peticiones vuelven a responder en **~9ms** con código **200**.

### B. Ejecución y Monitoreo (Paso a Paso)

**1. Preparar el entorno (Terminal 1):**
Abre el túnel hacia el frontend para poder acceder a la aplicación desde el navegador:
```bash
kubectl port-forward svc/frontend 8080:5001
```

**2. Monitorear el caos (Terminal 2):**
Deja este comando corriendo para ver el estado del motor de caos. Verás cómo pasa a `active` y luego a `Completed`:
```bash
kubectl get chaosengine -w
```

**3. Lanzar el caos (Terminal 3):**
Aplica el archivo YAML que inyecta la latencia:
```bash
kubectl apply -f k8s/testing/chaos-network-delay.yaml
```

### C. Generar Tráfico y Observar
1. Ve a tu navegador en `http://localhost:8080`.
2. Inicia sesión y entra a la sección **Order Management** (Órdenes).
3. Presiona **F12** para abrir las herramientas de desarrollador y ve a la pestaña **Network (Red)**.
4. Haz clic en el botón **"Get Orders"** varias veces durante el minuto que dura el caos.

**Secuencia esperada en la pestaña Network:**
*   **Peticiones 200 OK (rápidas):** Justo al inicio, antes de que la latencia se propague.
*   **Peticiones 504 Gateway Timeout (~5000ms):** La latencia de 2s supera el `perTryTimeout` de Istio (1s). Istio reintenta 3 veces, pero al agotar el `timeout` global de 5s, devuelve el error 504.
*   **Peticiones 503 Service Unavailable (rápidas):** El **Circuit Breaker** (R3.3) detecta los errores 504 y expulsa temporalmente los pods, rechazando nuevas peticiones de inmediato para no colgar al usuario.
*   **Peticiones 200 OK (rápidas):** Al finalizar los 60s del caos, la red se restaura y el sistema vuelve a la normalidad.

### D. Obtener el Veredicto (Evidencia)
Espera a que el comando de la Terminal 2 muestre que el proceso ha terminado, o ejecuta este comando para ver la prueba oficial de Litmus:

```bash
kubectl get chaosresult network-delay-engine-pod-network-latency -o yaml
```

**¿Qué buscar?**
*   `verdict: Pass`
*   `probeSuccessPercentage: "100"`
*   En la sección `targets`, deben aparecer los 2 pods de `microorders` atacados.

*(Toma capturas de pantalla de la pestaña Network y de esta salida de terminal).*

### E. Limpieza
Una vez terminado y documentado, borra el motor de caos para dejar el clúster limpio:
```bash
kubectl delete -f k8s/testing/chaos-network-delay.yaml
```

## 5. Experimento 3: CPU Hog (Saturación de CPU)

**Hipótesis:** *"Al inyectar una carga artificial de 1 núcleo de CPU en el microservicio `frontend`, el uso de CPU superará el umbral del 70% configurado en el HPA (R3.1). Kubernetes creará automáticamente nuevas réplicas para distribuir la carga, manteniendo la aplicación altamente disponible."*

### A. Comparación con el Steady State
*   **Steady State (Antes):** El HPA reporta **~3% de CPU** y mantiene **2 réplicas**.
*   **Estado durante el caos:** El HPA reporta picos de **hasta 430% de CPU** y escala dinámicamente de **2 a 5 réplicas**.
*   **Estado posterior:** Al terminar el caos, el uso de CPU baja a **~3%** y, tras el tiempo de enfriamiento, el HPA reduce las réplicas nuevamente a **2**.

### B. Ejecución y Monitoreo (Paso a Paso)
**1. Terminal 1 (Exponer la app):**
```bash
kubectl port-forward svc/frontend 8080:5001
```

**2. Terminal 2 (Monitorear el HPA en tiempo real):**
```bash
kubectl get hpa -w
```

**3. Terminal 3 (Monitorear la creación de nuevos pods):**
```bash
kubectl get pods -w | grep frontend
```

**4. Terminal 4 (Lanzar el caos):**
```bash
kubectl apply -f k8s/testing/chaos-cpu-hog.yaml
```

### C. Resultados Observados
*(Nota para la sustentación: El HPA tarda un poco en reaccionar porque el `metrics-server` recolecta datos cada 15 segundos).*
Mientras se navegaba por la aplicación, se observó cómo el HPA detectó el pico de estrés y creó nuevos pods de forma automática:
```text
frontend-hpa  Deployment/frontend  cpu: 4%/70%    2   5   2   162m
frontend-hpa  Deployment/frontend  cpu: 430%/70%  2   5   2   163m
frontend-hpa  Deployment/frontend  cpu: 430%/70%  2   5   4   163m
frontend-hpa  Deployment/frontend  cpu: 84%/70%   2   5   5   164m
frontend-hpa  Deployment/frontend  cpu: 3%/70%    2   5   5   165m
```
La aplicación web nunca dejó de responder, demostrando que el autoescalado absorbió el ataque.

### D. Obtener el Veredicto (Evidencia)
Espera a que el experimento termine y ejecuta:
```bash
kubectl get chaosresult cpu-hog-engine-pod-cpu-hog -o yaml
```
*   **Veredicto:** `verdict: Pass`
*   **Targets:** Los 2 pods originales fueron atacados y revertidos a su estado normal.

### E. Limpieza
```bash
kubectl delete -f k8s/testing/chaos-cpu-hog.yaml
```

## 3. Experimento 4: Network Loss (Pérdida de Paquetes)

**Hipótesis:** *"Al inyectar una pérdida del 50% de paquetes de red en `microusers`, las peticiones experimentarán fallos intermitentes (errores 504) y demoras significativas. Sin embargo, gracias a las políticas de reintento configuradas en Istio (R3.4), algunas peticiones lograrán completarse exitosamente tras varios intentos, demostrando la tolerancia a fallos de la malla de servicios."*

### A. Comparación con el Steady State
*   **Steady State (Antes):** Peticiones estables con respuestas de **~7ms a 10ms** (200 OK).
*   **Estado durante el caos:** Comportamiento errático. Se observan errores **504 Gateway Timeout** (al cumplirse el `timeout` de 5s de Istio) mezclados con **200 OK** lentos (de 1.5s a 4.3s) que lograron salvarse gracias a los reintentos.
*   **Estado posterior:** La red se restaura y las peticiones vuelven a su latencia normal de **~7ms**.

### B. Ejecución y Monitoreo (Paso a Paso)
**1. Terminal 1 (Exponer la app):**
```bash
kubectl port-forward svc/frontend 8080:5001
```
**2. Terminal 2 (Monitorear el caos):**
```bash
kubectl get chaosengine -w
```
**3. Terminal 3 (Lanzar el caos):**
```bash
kubectl apply -f k8s/testing/chaos-network-loss.yaml
```

### C. Generar Tráfico
1. Ir al navegador en `http://localhost:8080/users`.
2. Hacer clic en "Get Users" repetidamente durante el minuto que dura el caos.
3. **Observación clave en la pestaña Network:** Se observa cómo la red falla (504) pero Istio interviene reintentando, resultando en peticiones exitosas pero con latencias altas (ej. 2650ms) antes de volver a la normalidad.

### D. Obtener el Veredicto (Evidencia)
```bash
kubectl get chaosresult network-loss-engine-pod-network-loss -o yaml
```
*   **Veredicto:** `verdict: Pass`
*   **Targets:** Atacados y revertidos los 2 pods de `microusers`.

### E. Limpieza
```bash
kubectl delete -f k8s/testing/chaos-network-loss.yaml
```

## 3. Experimento 5: Memory Hog (Saturación de Memoria)

**Hipótesis:** *"Al inyectar un consumo de memoria (300MB) superior al límite configurado en el Deployment de `microorders` (256Mi), el kernel de Linux matará el contenedor (OOMKilled). Kubernetes lo reiniciará automáticamente y, gracias al balanceo de carga y al Circuit Breaker de Istio, el sistema sufrirá una interrupción mínima y se recuperará en segundos."*

### A. Comparación con el Steady State
*   **Steady State (Antes):** Peticiones rápidas de **~9ms a 10ms** (200 OK).
*   **Estado durante el caos:** Los pods son asesinados por falta de memoria (`OOMKilled`). Istio detecta la caída y devuelve errores rápidos **503** (4-5ms) durante un breve lapso.
*   **Estado posterior:** Kubernetes reinicia los pods en 1-12 segundos. Las peticiones vuelven a ser **200 OK** (~10ms).

### B. Ejecución y Monitoreo (Paso a Paso)
**1. Terminal 1 (Exponer la app):**
```bash
kubectl port-forward svc/frontend 8080:5001
```
**2. Terminal 2 (Monitorear reinicios y OOMKilled):**
```bash
kubectl get pods -w | grep microorders
```
**3. Terminal 3 (Lanzar el caos):**
```bash
kubectl apply -f k8s/testing/chaos-memory-hog.yaml
```

### C. Resultados Observados (Evidencia)
Mientras se hacían peticiones desde el navegador, la terminal mostró cómo los pods morían y renacían:
```text
microorders-68b9cbfc59-kzx8c  1/2  OOMKilled  3 (73m ago)  3h22m
microorders-68b9cbfc59-kzx8c  1/2  Running    4 (1s ago)   3h22m
microorders-68b9cbfc59-t4bxq  1/2  OOMKilled  3 (73m ago)  3h22m
microorders-68b9cbfc59-t4bxq  1/2  Running    4 (1s ago)   3h22m
```
En la pestaña **Network**, se observó una ráfaga de 5 errores `503` (mientras los pods reiniciaban), seguida inmediatamente por peticiones `200 OK` exitosas.

### D. Obtener el Veredicto
```bash
kubectl get chaosresult memory-hog-engine-pod-memory-hog -o yaml
```
*   **Veredicto:** `verdict: Pass`
*   **Targets:** Ambos pods fueron atacados y revertidos.

### E. Limpieza
```bash
kubectl delete -f k8s/testing/chaos-memory-hog.yaml
```

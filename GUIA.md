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
minikube start --driver=docker --cpus=2 --memory=4096 --force-systemd=true
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
minikube start --driver=docker --cpus=2 --memory=4096

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
Vamos a aplicar el `DestinationRule` que configura el Circuit Breaker y un `EnvoyFilter` que inyecta errores 503 **directamente en el servidor** (para que el frontend reciba errores reales de red).

```bash
kubectl apply -f k8s/05-circuit-breaker.yaml
kubectl apply -f k8s/98-envoy-fault-server.yaml
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
3. Haz clic en el botón **"Get Products"** (o en **"Create Product"** llenando los campos) unas 10 a 15 veces rápidamente.
4. *(Nota: La interfaz no mostrará un mensaje de error visible porque el JavaScript del frontend solo imprime los errores en la consola. Para ver que los errores 503 realmente están llegando, puedes presionar **F12** en tu navegador, ir a la pestaña **Network (Red)** y verás las peticiones fallando con el código HTTP **503**).*

---

## 5. El "Después" (Circuit Breaker Activado)
Vuelve a ejecutar **exactamente el mismo comando** del Paso 3:

```bash
istioctl proxy-config endpoint $(kubectl get pod -l app=frontend -o jsonpath="{.items[0].metadata.name}") --cluster "outbound|5003||microproducts.default.svc.cluster.local"
```

> **¿Qué esperar?** Al menos uno de los pods ahora mostrará `OUTLIER CHECK: FAILED`. Esto significa que Istio detectó los 5 errores consecutivos y expulsó temporalmente al pod del balanceo de carga para proteger el sistema. *(Toma una captura de esto).*

---

## 6. Evidencia Numérica (Métricas del Proxy)
Para obtener la prueba definitiva del aislamiento de instancias, ejecuta el siguiente comando que extrae las métricas del proxy Envoy:

```bash
kubectl exec -it $(kubectl get pod -l app=frontend -o jsonpath="{.items[0].metadata.name}") -c istio-proxy -- pilot-agent request GET stats | grep -E "ejections_enforced_total|ejections_consecutive_5xx|ejections_enforced_consecutive_5xx|ejections_overflow"
```

> **¿Qué esperar?** Verás contadores mayores a 0, por ejemplo:
> * `ejections_enforced_total: 3` (El pod fue expulsado 3 veces).
> * `ejections_consecutive_5xx: 5` (Se disparó la expulsión al alcanzar 5 errores 503 consecutivos).
> * `ejections_overflow: 4` (Demuestra que la regla `maxEjectionPercent: 50` evitó deshabilitar más de la mitad de la infraestructura).

---

## 7. Evidencia Visual en Kiali
En la interfaz de Kiali:
1. Dirígete a **Traffic Graph**.
2. Selecciona el namespace `default` y ajusta el rango de tiempo a `Last 1m`.
3. Activa las opciones de **Traffic Animation** y **Security** en el menú *Display*.

> **¿Qué observar?** Las líneas de conexión hacia `microproducts` se tornarán de color **rojo** con un icono de **rayo** ⚡, representando el fallo de peticiones y la activación del mecanismo de protección.

---

## 8. Limpieza del Entorno
Una vez capturadas las evidencias, es **obligatorio** eliminar el inyector de fallos para restablecer la comunicación habitual del sistema:

```bash
kubectl delete -f k8s/98-envoy-fault-server.yaml
```

*(El manifiesto `05-circuit-breaker.yaml` puede permanecer aplicado, ya que constituye la regla de resiliencia estándar para producción).*

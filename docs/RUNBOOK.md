# Runbook de Respuesta a Incidentes (Proyecto 3)

Este documento proporciona los procedimientos de diagnóstico y recuperación para los incidentes simulados durante las pruebas de Chaos Engineering en la arquitectura de microservicios desplegada en Kubernetes con Istio.

---

## 1. Herramientas de Diagnóstico Previas

Antes de investigar cualquier incidente, el ingeniero de guardia debe verificar el acceso a las siguientes herramientas ejecutando cada comando en una terminal independiente:

### Topología de red y errores visuales
```bash
istioctl dashboard kiali
```

### Trazas distribuidas (detección de cuellos de botella)
```bash
kubectl port-forward svc/tracing -n istio-system 16686:80
```

### Dashboard de resiliencia (Grafana)
```bash
kubectl port-forward svc/grafana -n istio-system 3000:3000
```

### Comandos base de Kubernetes
```bash
kubectl get pods
kubectl get hpa
```

---

## 2. Incidente 1: Caída de Pods (Pod Delete / Memory Hog)

### Síntomas reportados
* La aplicación web devuelve errores `HTTP 500` / `HTTP 503` de forma intermitente.
* Los usuarios reportan fallos repentinos durante la navegación.

### Procedimiento de diagnóstico

1. **Estado de los Pods:**
   ```bash
   kubectl get pods
   ```
   * **Criterio de búsqueda:** Pods con estado `Error`, `OOMKilled`, `Terminating` o con un número elevado en la columna `RESTARTS`.

2. **Eventos del Pod (en caso de fallos persistentes):**
   ```bash
   kubectl describe pod <nombre-del-pod>
   ```
   * **Criterio de búsqueda:** En la sección **Events**, identificar mensajes como `OOMKilled` (insuficiencia de memoria) o `Killing` (acciones de Litmus/Chaos).

### Procedimiento de recuperación

* **Pods en estado `OOMKilled` o `Terminating`:**
  Kubernetes aplicará los mecanismos de autorrecuperación (*Self-healing*) reiniciando el pod automáticamente. Realice el seguimiento con:
  ```bash
  kubectl get pods -w
  ```
  Verifique hasta que el estado cambie a `Running 2/2`.

* **Pods en estado `Error` persistente:**
  Examine los registros para identificar la causa raíz y fuerce el reinicio si es necesario:
  ```bash
  kubectl logs <nombre-del-pod>
  kubectl rollout restart deployment <nombre-del-deployment>
  ```

---

## 3. Incidente 2: Degradación de Red (Network Delay / Network Loss)

### Síntomas reportados
* Tiempos de respuesta en las peticiones superiores a 5 segundos.
* El navegador devuelve errores `504 Gateway Timeout` o `503 Service Unavailable`.

### Procedimiento de diagnóstico

1. **Análisis de trazas distribuidas (Jaeger):**
   * Acceda a Jaeger > Servicio `frontend.default` > Filtre por trazas con duración mayor a 2 segundos.
   * **Criterio de búsqueda:** Identifique el *span* de mayor duración. Si `microproducts` o `microorders` presentan bloques extensos, allí se localiza el cuello de botella.

2. **Topología de red (Kiali):**
   * Acceda a **Traffic Graph**.
   * **Criterio de búsqueda:** Enlaces de conexión en color rojo (errores) o amarillo (latencia elevada).

3. **Circuit Breaker (Grafana):**
   * En el Dashboard de Resiliencia, revise el panel **Circuit Breaker Ejections**.
   * **Criterio de búsqueda:** Un pico en la gráfica indica que Istio ha aislado pods por tasa de error.

### Procedimiento de recuperación

* **Activación de Circuit Breaker:**
  El sistema se autorrecupera. Espere a que transcurra el periodo de `baseEjectionTime` (30 segundos) para que Istio reanude el tráfico hacia los pods afectados.

* **Latencia originada en Base de Datos:**
  Compruebe si existen consultas lentas. *(Propuesta a futuro: Implementar una capa de caché con Redis).*

* **Persistencia del fallo:**
  Valide la existencia de experimentos de caos activos en el clúster:
  ```bash
  kubectl get chaosengine
  ```
  En caso afirmativo, proceda a su eliminación:
  ```bash
  kubectl delete chaosengine --all
  ```

---

## 4. Incidente 3: Saturación de Recursos (CPU Hog)

### Síntomas reportados
* La aplicación responde con lentitud sin llegar a fallar.
* El clúster experimenta niveles altos de utilización de recursos.

### Procedimiento de diagnóstico

1. **Autoescalador Horizontal de Pods (HPA):**
   ```bash
   kubectl get hpa
   ```
   * **Criterio de búsqueda:** Métrica `TARGETS` de CPU por encima del 70% o cercana al 100%.

2. **Conteo de réplicas:**
   ```bash
   kubectl get pods -l app=frontend
   ```
   * **Criterio de búsqueda:** Presencia de más de 2 pods activos (indicador de que el HPA ha ejecutado el escalado).

### Procedimiento de recuperación

* **Acción inmediata:** No requiere intervención manual. El HPA escalará automáticamente hasta un máximo de 5 réplicas para absorber la carga.
* **Acción post-incidente:** Si el HPA alcanza el límite (5 réplicas) y la lentitud persiste, se debe escalar la capacidad de los nodos del clúster (aumentar CPU/RAM en Minikube o en el proveedor cloud).

---

## 5. Registro del Incidente (Post-Mortem)

Al finalizar la atención del evento, el ingeniero a cargo debe documentar los siguientes puntos:

1. Hora de inicio y duración total del incidente.
2. Servicio o microservicio afectado.
3. Causa raíz identificada (mediante Jaeger, Kiali o Grafana).
4. Acciones correctivas aplicadas (automáticas por K8s/Istio o manuales).

---

## 6. Matriz de Cobertura de Caos vs. Incidentes

Este runbook cubre los 5 experimentos de LitmusChaos requeridos, agrupados según los síntomas presentados:

| Experimento de Caos | Incidente en el Runbook | Síntoma Principal |
| :--- | :--- | :--- |
| `pod-delete` | Incidente 1 (Caída de Pods) | Pod en estado `Terminating`. |
| `pod-memory-hog` | Incidente 1 (Caída de Pods) | Pod en estado `OOMKilled`. |
| `pod-network-latency` | Incidente 2 (Degradación de Red) | Latencia alta (errores HTTP 504). |
| `pod-network-loss` | Incidente 2 (Degradación de Red) | Errores intermitentes (HTTP 503/504). |
| `pod-cpu-hog` | Incidente 3 (Saturación de Recursos) | HPA escalando al 400% de consumo de CPU. |

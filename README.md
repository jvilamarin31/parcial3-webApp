# Proyecto 3: Infraestructura Resiliente con Kubernetes, Service Mesh y Chaos Engineering

**Asignatura:** Computación en la Nube
**Integrantes:** 
*   Juan Esteban Vila Marín
*   Juan David Cuero Katar
*   Jorge Andres Medina Urriata
*   Juan Jose Solarte Pabon
*   Alejandro Rodriguez Cortes

## Descripción del Proyecto
Este repositorio contiene la implementación de una arquitectura de microservicios desplegada en Kubernetes (Minikube) y potenciada con **Istio** como Service Mesh. El sistema implementa patrones de resiliencia (Circuit Breaker, Retries, Timeouts, mTLS) y ha sido validado mediante **Chaos Engineering** con **LitmusChaos** para garantizar su tolerancia a fallos. Además, cuenta con un stack completo de observabilidad (Kiali, Grafana, Jaeger) y documentación técnica detallada.

---

## Estructura del Repositorio

El repositorio está organizado de la siguiente manera para facilitar la navegación y reproducibilidad del entorno:

```text
parcial3-webApp/
├── README.md                          # Este archivo (Visión general del proyecto)
├── LICENSE                            # Licencia del proyecto
├── docker-compose.yml                 # Orquestación local (Parte 1 del curso)
├── frontend/                          # Código fuente del Frontend (Flask)
├── microUsers/                        # Código fuente del servicio de Usuarios
├── microProducts/                     # Código fuente del servicio de Productos
├── microOrders/                       # Código fuente del servicio de Órdenes
│
├── docs/                              # Documentación y Evidencias
│   ├── GUIA.md                        # Guía paso a paso para desplegar y probar los requerimientos
│   ├── RUNBOOK.md                     # Manual de operaciones e incidentes (R3.8)
│   ├── INFORME_CAOS.md                # Informe detallado de los 5 experimentos de LitmusChaos (R3.5)
│   ├── INFORME_OBSERVABILIDAD.md      # Análisis de trazas (Jaeger) y métricas (Grafana) (R3.6 y R3.7)
│   └── img/                           # Capturas de pantalla usadas en los informes
│
└── k8s/                               # Manifiestos de Kubernetes (Producción)
    ├── 01-databases.yaml              # Despliegue de las 3 BD MySQL (con volúmenes persistentes)
    ├── 02-microservices.yaml          # Deployments y Services de los 4 microservicios
    ├── 03-hpa.yaml                    # Horizontal Pod Autoscalers (Autoescalado por CPU)
    ├── 04-mtls-strict.yaml            # PeerAuthentication de Istio (Cifrado estricto mTLS)
    ├── 05-circuit-breaker.yaml        # DestinationRule de Istio (Outlier Detection)
    ├── 06-resiliencia-virtual-service.yaml # VirtualServices de Istio (Retries y Timeouts)
    ├── 07-telemetry.yaml              # Configuración de Tracing para Jaeger
    ├── grafana/                       # Dashboard JSON exportado
    │   └── istio-resilience-dashboard.json
    └── testing/                       # Manifiestos de Pruebas y Caos (NO aplicar en producción)
        ├── litmus-rbac.yaml           # Permisos para el runner de Litmus
        ├── chaos-pod-delete.yaml      # ChaosEngine 1
        ├── chaos-network-delay.yaml   # ChaosEngine 2
        ├── chaos-cpu-hog.yaml         # ChaosEngine 3
        ├── chaos-network-loss.yaml    # ChaosEngine 4
        ├── chaos-memory-hog.yaml      # ChaosEngine 5
        └── 98-envoy-*.yaml            # Inyectores de fallos (EnvoyFilters) para pruebas de Istio
```

---

## Documentación Clave

Para entender a profundidad la solución y cómo fue probada, consulte los siguientes documentos ubicados en la carpeta `docs/`:

1.  **[GUIA.md](./docs/GUIA.md):** El paso a paso técnico de cómo instalar herramientas, desplegar la arquitectura y ejecutar las pruebas de cada uno de los 8 requerimientos.
2.  **[INFORME_CAOS.md](./docs/INFORME_CAOS.md):** La evidencia y el análisis de los 5 experimentos de Chaos Engineering, incluyendo hipótesis, ejecución, resultados y comparación con el estado estacionario.
3.  **[INFORME_OBSERVABILIDAD.md](./docs/INFORME_OBSERVABILIDAD.md):** El análisis profundo de las trazas distribuidas en Jaeger (identificación del cuello de botella) y la configuración del dashboard de resiliencia en Grafana.
4.  **[RUNBOOK.md](./docs/RUNBOOK.md):** El manual operativo (Runbook) que un ingeniero de guardia puede usar para diagnosticar y recuperar el sistema ante 3 tipos de incidentes simulados.

---

## Tecnologías Utilizadas

*   **Orquestación:** Kubernetes (Minikube)
*   **Service Mesh:** Istio (mTLS, Circuit Breaker, Retries)
*   **Chaos Engineering:** LitmusChaos
*   **Observabilidad:** Kiali (Topología), Grafana + Prometheus (Métricas), Jaeger (Trazas)
*   **Contenedores:** Docker (imágenes personalizadas alojadas en Docker Hub)
*   **Lenguaje:** Python (Flask) / JavaScript (Frontend)

---

## Inicio Rápido

Para levantar todo el ecosistema desde cero y ver la magia de la resiliencia, siga estos comandos básicos (para el detalle completo vaya a la **[GUIA.md](./docs/GUIA.md)**):

```bash
# 1. Clonar el repositorio
git clone https://github.com/TU-USUARIO/parcial3-webApp.git
cd parcial3-webApp

# 2. Iniciar el clúster (se recomiendan al menos 6 CPUs y 7GB de RAM)
minikube start --driver=docker --cpus=6 --memory=7168 --force-systemd=true
minikube addons enable metrics-server

# 3. Aplicar los manifiestos base de la aplicación
kubectl apply -f k8s/01-databases.yaml
kubectl apply -f k8s/02-microservices.yaml
kubectl apply -f k8s/03-hpa.yaml

# 4. Instalar Istio y aplicar las reglas de resiliencia
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
kubectl rollout restart deployment microusers microproducts microorders frontend
kubectl apply -f k8s/04-mtls-strict.yaml
kubectl apply -f k8s/05-circuit-breaker.yaml
kubectl apply -f k8s/06-resiliencia-virtual-service.yaml

# 5. Exponer la aplicación
kubectl port-forward svc/frontend 8080:5001
# Abrir http://localhost:8080 (Usuario: juan / Contraseña: 123)
```

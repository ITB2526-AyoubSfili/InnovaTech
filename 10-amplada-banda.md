# 2.3 Comprobaciones de Ancho de Banda

---

## Speedtest — Velocidad hacia Internet

Instalamos un cliente de speedtest para medir la velocidad de conexión hacia el mundo exterior.

```bash
ubuntu@ip-10-0-1-60:~$ sudo apt install speedtest-cli -y
ubuntu@ip-10-0-1-60:~$ speedtest-cli
```

Resultados obtenidos:

| Métrica | Resultado |
|---|---|
| Descarga | `749.82 Mbit/s` |
| Subida | `8.31 Mbit/s` |
| Ping | `10.617 ms` |

---

## iPerf3 — Rendimiento de Red Interno

Instalamos `iperf3` para medir el rendimiento de red interno o directo entre dos máquinas específicas. A diferencia de speedtest, **no mide internet**, sino la capacidad máxima de transferencia y el estrés del cable o la red privada que une ambos puntos.

```bash
ubuntu@ip-10-0-1-60:~$ sudo apt install iperf3 -y
```

Resultados obtenidos tras el test de 10 segundos:

| Métrica | Resultado |
|---|---|
| Velocidad media | `313 Mbps` estables de transferencia directa |
| Volumen total transferido | `375 MB` en 10 segundos |
| Pérdida de paquetes | `432` pérdidas al inicio; **estabilización completa a partir del segundo 2** |

---

## Análisis por Servicio

### Requisitos Mínimos vs. Resultados Reales

**Streaming de Audio (MP3)**
Solo requiere `0.2 Mbps` y el servidor ofrece `749 Mbps` de bajada y `8.31 Mbps` de subida. La latencia de `10.6 ms` está muy por debajo del límite de `500 ms`.

**Streaming de Vídeo (HLS 1080p)**
Al límite. Para la descarga va perfecto con `749 Mbps` sobre los `8 Mbps` necesarios, pero para la emisión la máquina ofrece `8.31 Mbps`, rozando el mínimo requerido de `8 Mbps`. **No hay margen para más de un directo simultáneo.**

**Videoconferencia (Jitsi)**
Justo. Requiere `4 Mbps` de subida y el servidor ofrece `8.31 Mbps`, por lo que soporta la llamada de forma individual. La latencia de `10.6 ms` es excelente frente al límite de `150 ms`.

---

### Tabla Comparativa — Resumen

| Servicio | Descarga (Pedida / Real) | Subida (Pedida / Real) | Latencia (Límite / Real) | Estado |
|---|---|---|---|---|
| Audio MP3 | 0.2 / 749 Mbps | 0.2 / 8.31 Mbps | < 500 / 10.6 ms | Excelente |
| Vídeo 1080p | 8.0 / 749 Mbps | 8.0 / 8.31 Mbps | < 200 / 10.6 ms | Muy Justo |
| Videollamada | 4.0 / 749 Mbps | 4.0 / 8.31 Mbps | < 150 / 10.6 ms |  Aceptable |

---

## ¿Cómo funciona?

| Parámetro | Valor | Valoración |
|---|---|---|
| Bajada (749 Mbps) | Excelente | Soporta a cientos de usuarios viendo vídeos a la vez |
| Latencia (10.6 ms) | Perfecta | Las videollamadas van en tiempo real, sin retrasos perceptibles |
| Subida (8.31 Mbps) | Muy justa | Si varios usuarios transmiten en directo a la vez, el servidor se colapsará |

---

## Mejoras Propuestas

**Máquina más grande (`t3.medium`):** Aumenta la velocidad de salida para que más usuarios puedan transmitir en directo de forma simultánea.

**Red de distribución de contenido (CDN):** Reparte los vídeos desde servidores cercanos al usuario, reduciendo la carga directa sobre la instancia EC2.

**Priorización de tráfico (QoS):** Da preferencia a las llamadas en vivo frente al vídeo grabado cuando la red se satura.

**Monitoreo con alertas en tiempo real:** Notifica automáticamente si el servidor se está acercando a su límite de capacidad.

---

## Conclusión

El servidor cumple para arrancar, pero la **velocidad de subida (`8.31 Mbps`) es el cuello de botella**. Para un uso real con múltiples usuarios emitiendo en directo de forma simultánea, será necesario escalar la potencia de la instancia en AWS.

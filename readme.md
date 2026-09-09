# 🚘 Asistente de Estacionamiento Autónomo para Cochera (Zero-Power-Draw Idle)

Sistema de guiado visual de alta precisión para garajes basado en **ESPHome** sobre un microcontrolador **ESP8266 (NodeMCU v2)**. El dispositivo opera de forma **100% independiente** (sin requerir Home Assistant, dependencias de nube ni conexión continua a red Wi-Fi), ofreciendo consumo de corriente nulo ($0\text{ W}$) en reposo mediante un sistema de encendido magnético autónomo.

---

## 📋 Descripción del Proyecto y Arquitectura

El asistente utiliza tres sensores ultrasónicos (HC-SR04), tres módulos de diodo láser para marcar puntos de referencia visuales en el vehículo y un matriz de LED redondos de $8\times32$ píxeles gobernada por cuatro controladores MAX7219 encadenados.

### Principales Características
* **Zero Power Draw en Reposo:** La alimentación general de $5\text{V}$ pasa a través de un relé comandado por un sensor magnético de lámina (reed switch) ubicado en el portón del garaje. Al cerrarse el portón o no detectar el vehículo, la alimentación se interrumpe por completo.
* **Tolerancia a Fallos de Red:** Operación autónoma en modo AP (Punto de Acceso) local. Sin retardos por desconexiones o bloqueos por fallas de Wi-Fi.
* **Alineación Simétrica de Precisión:** Muescas estáticas centrales y barra móvil proporcional de $2\times6$ píxeles en la pantalla para evitar colisiones laterales con espejos retrovisores o columnas.
* **Respuesta Dinámica según Sentido de Marcha:** Detecta automáticamente si el vehículo está ingresando (aproximación) o saliendo (retroceso), modificando el flujo visual de la pantalla.
* **Filtrado Peatonal:** Requiere la lectura simultánea de ambos sensores laterales para activar el modo de alineación, evitando falsas alarmas por paso de personas.
* **Modo STOP Invertido y Parpadeante:** Inversión de hardware completa de la matriz LED alternando a alta frecuencia en zona crítica ($\le 10\text{ cm}$).
* **Transición Inteligente a READY:** Apagado de alertas y cambio a `READY` tras un tiempo estático configurable ($10\text{ s}$) o en arranques con el vehículo ya estacionado.

---

## 📐 Estados del Sistema de Guiado Visual

| Estado | Condición de Activación | Representación Matriz LED ($8\times32$) | Simulación Web UI |
| :--- | :--- | :--- | :--- |
| **Arranque / Reposo** | Auto estático al encender o fuera de rango | Texto fijo **`READY`** | `[   R E A D Y   ]` |
| **Retroceso / Salida** | Distancia de fondo aumentando ($> +1\text{ cm}$) | Chevrons animados bajando **`v`** | `[       v       ]` |
| **Alineación Lateral** | Ambos sensores laterales $\le 50\text{ cm}$ | Muesca central `:` + Barra móvil $2\times6\text{px}$ `║` | `[   ║     :       >>  ]` |
| **Aproximación Normal** | Distancia de fondo entre $50\text{ cm}$ y $150\text{ cm}$ | Distancia en cm + Chevrons subiendo `^` | `«   120 cm   »` |
| **Precaución / Alerta**| Distancia de fondo entre $10\text{ cm}$ y $50\text{ cm}$ | Scroll ralentizado y parpadeo de brillo | `«   30 cm   »` |
| **STOP Crítico** | Distancia de fondo $\le 10\text{ cm}$ | Texto **`STOP`** con Inversión Parpadeante | `[  S T O P  ]` |
| **Inactividad Post-STOP**| Vehículo estático $\le 10\text{ cm}$ durante $> 10\text{ s}$ | Pasa de `STOP` a **`READY`** de bajo consumo | `[   R E A D Y   ]` |

---

## 🔌 Hardware y Esquemática de Conexionado

### Componentes Requeridos
1. **Microcontrolador:** NodeMCU v2 (ESP8266).
2. **Pantalla:** Matriz de LED MAX7219 ($8\times32$ píxeles, 4 módulos).
3. **Sensores de Distancia:** 3x HC-SR04 (Ultrasonido).
4. **Lásers de Posición:** 3x Módulos de Diodo Láser de $5\text{V}$.
5. **Control de Energía:** Módulo Relé de $5\text{V}$ (Canal Normal Cerrado/Abierto según reed switch).
6. **Protección Lógica:** Divisores de tensión ($1.5\text{ k}\Omega$ y $1.8\text{ k}\Omega$) para adaptar las salidas de $5\text{V}$ del `Echo` de los HC-SR04 al nivel de $3.3\text{V}$ del ESP8266.
7. **Infraestructura de Cableado:** Cable de red UTP Cat 5e de cobre.

### Asignación de Pines (ESP8266 NodeMCU)

```text
                  ┌───────────────────────────────┐
                  │    NodeMCU v2 (ESP8266)       │
                  ├───────────────────────────────┤
                  │ GPIO12 (D6) ──────► Trigger (Todos HC-SR04)
                  │ GPIO14 (D5) ──────► Echo (Fondo) / SPI CLK
                  │ GPIO5  (D1) ──────► Echo (Izquierda)
                  │ GPIO4  (D2) ──────► Echo (Derecha)
                  │ GPIO13 (D7) ──────► SPI MOSI (DIN MAX7219)
                  │ GPIO15 (D8) ──────► SPI CS (CS MAX7219)
                  └───────────────────────────────┘

```

### Adaptación de Voltaje (Divisor Resistivo por PIN Echo)

Debido a que los sensores HC-SR04 entregan pulsos de $5\text{V}$ en su pin `Echo` y los pines GPIO del ESP8266 operan a un máximo de $3.3\text{V}$, se debe intercalar un divisor resistivo en cada línea `Echo`:

```text
Echo HC-SR04 (5V) ───[ 1.5 kΩ ]───┬───► GPIO ESP8266 (3.12V Seguro)
                                  │
                               [ 1.8 kΩ ]
                                  │
                                 GND

```

### Distribución de Pares en Cable UTP Cat 5e

```text
 [ Par 1: Azul / Blanco-Azul ]     ──► Alimentación (+5V VCC / GND)
 [ Par 2: Naranja / Blanco-Naranja] ──► Bus SPI MAX7219 (DIN / CS)
 [ Par 3: Verde / Blanco-Verde ]   ──► Línea de Trigger Unificada (GPIO12)
 [ Par 4: Marrón / Blanco-Marrón ] ──► Línea de Echo Filtrada (GPIO14/5/4)

```

---

## 🛠️ Compilación e Instalación

1. **Requisitos Previos:** Tener instalado **ESPHome** mediante CLI o Docker.
2. **Archivos Necesarios:**
* Archivo de configuración: `asistente_cochera.yaml`.
* Fuente tipográfica bitmap: Descargar el archivo **`TomThumb.bdf`** y colocarlo exactamente en la misma carpeta que el archivo `.yaml`.


3. **Compilación y Flasheo Inicial:**
Conecta el NodeMCU v2 por USB a tu equipo y ejecuta:
```bash
esphome run asistente_cochera.yaml

```


4. **Modificaciones Posteriores:**
Al iniciar, el dispositivo genera un Punto de Acceso Wi-Fi llamado `Asistente-Cochera-AP`. Puedes conectarte desde cualquier teléfono o PC e ingresar a `http://192.168.4.1` para ajustar los umbrales de distancia y visualizar la simulación de la pantalla en tiempo real.

```

<ElicitationsGroup message="¿Deseas dar por finalizada la documentación o profundizar en algún área?">
  <Elicitation label="Ver guía de instalación física y montaje" query="Proporciona recomendaciones para el montaje físico de la matriz y los sensores en la pared de la cochera."/>
</ElicitationsGroup>

```

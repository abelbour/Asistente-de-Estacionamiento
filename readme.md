# 🚘 Asistente de Estacionamiento Autónomo para Cochera (Zero-Power-Draw Idle)

Sistema de guiado visual de alta precisión para garajes basado en **ESPHome** sobre un microcontrolador **ESP8266 (NodeMCU v2)**. El dispositivo opera de forma **100% independiente** (sin requerir Home Assistant, dependencias de nube ni conexión continua a red Wi-Fi), ofreciendo consumo de corriente nulo ($0\text{ W}$) en reposo mediante un sistema de encendido magnético autónomo.

---

## 📋 Descripción del Proyecto y Arquitectura

El asistente utiliza tres sensores ultrasónicos (HC-SR04), tres módulos de diodo láser para marcar puntos de referencia visuales en el vehículo y una matriz de LED de $8\times32$ píxeles gobernada por cuatro controladores MAX7219 encadenados (DOUT→DIN).

### Principales Características
* **Zero Power Draw en Reposo:** La alimentación general de $5\text{V}$ pasa a través de un relé comandado por un sensor magnético de lámina (reed switch) ubicado en el portón del garaje. Al cerrarse el portón o no detectar el vehículo, la alimentación se interrumpe por completo.
* **Tolerancia a Fallos de Red:** Operación autónoma en modo AP (Punto de Acceso) local. Sin retardos por desconexiones o bloqueos por fallas de Wi-Fi.
* **Alineación Simétrica de Precisión:** Arriba ticks de límite del portón + barra de posición (el tick contrario se vuelve chevrón de corrección en desvío); abajo marca de centro fija y distancias de ambos lados en dígitos 3x5 dibujados contra los márgenes — para evitar colisiones laterales con espejos retrovisores o columnas.
* **Respuesta Dinámica según Sentido de Marcha:** Detecta automáticamente si el vehículo está ingresando (aproximación) o saliendo (retroceso), modificando el flujo visual de la pantalla.
* **Filtrado Peatonal:** Requiere la lectura simultánea de ambos sensores laterales para activar el modo de alineación, evitando falsas alarmas por paso de personas.
* **Suavizado de Lecturas:** Cada HC-SR04 publica en cm (`unit_of_measurement: "cm"`) con filtro de mediana de 5 muestras y disparo secuencial por turnos (round-robin cada 70 ms: fondo → izquierda → derecha, cada sensor se mide cada 210 ms) para que ningún eco interfiera con otro.
* **Modo STOP Invertido y Parpadeante:** Inversión fija de la matriz LED (`invert_on_off`) en zona crítica ($\le 10\text{ cm}$): fondo encendido / texto apagado, con parpadeo de brillo entre intensidad 12 y 7 (pico de consumo contenido).
* **Transición Inteligente a READY:** Apagado de alertas y cambio a `READY` tras un tiempo estático configurable ($10\text{ s}$) o en arranques con el vehículo ya estacionado.

---

## 📐 Estados del Sistema de Guiado Visual

| Estado | Condición de Activación | Representación Matriz LED ($8\times32$) | Simulación Web UI |
| :--- | :--- | :--- | :--- |
| **Arranque / Reposo** | Auto estático al encender o fuera de rango | Texto fijo **`READY`** (fuente Spleen 5x8) | `[   R E A D Y   ]` |
| **Retroceso / Salida** | Distancia de fondo aumentando ($> +1\text{ cm}$) | Doble chevrón gráfico `vv` bajando (laterales, scroll `/3`) | `[      vv      ]` |
| **Alineación Lateral** | Ambos sensores laterales $\le 50\text{ cm}$ (+ histéresis 2 cm) | Arriba: ticks de límite $1\times2\text{px}$ en bordes + barra de progreso sólida desde el centro (largo = magnitud del desvío; 5 px de alto, o 3 px si chocaría con los números); en desvío (> `umbral_desvio_chev`, histéresis 1 cm) el tick contrario se reemplaza por triple mini-chevrón 2x3 en zona de 13 px del lado libre apuntando la corrección (marcha hacia el borde). Abajo: distancias 3x5 contra los márgenes con marca de centro fija de $2\times2\text{px}$ abajo (filas 6–7, p. ej. `30 | 30`) | `[ 45cm << : ║ 30cm ]` (valores en vivo) |
| **Aproximación Normal** | Distancia de fondo entre $50\text{ cm}$ y $150\text{ cm}$ | Distancia en cm + doble chevrón gráfico `^^` subiendo (scroll rápido `/4`) | `«   120 cm   »` |
| **Precaución / Alerta**| Distancia de fondo entre $10\text{ cm}$ y $50\text{ cm}$ | Scroll ralentizado (`/8`) y parpadeo de brillo en Alerta ($\le 20\text{ cm}$) | `«   30 cm   »` |
| **STOP Crítico** | Distancia de fondo $\le 10\text{ cm}$ | Texto **`STOP`** con inversión fija y parpadeo de brillo (12 ↔ 7) | `[  S T O P  ]` |
| **Peligro Poste Lateral** | Algún lateral $\le 15\text{ cm}$ (estando en alineación, con switch activado) | Alterna pantalla de alineación con **`STOP`** invertido fijo (500 ms cada una) | `[  S T O P  ]` / valores en vivo |
| **Inactividad Post-STOP**| Vehículo estático $\le 10\text{ cm}$ durante $> 10\text{ s}$ | Pasa de `STOP` a **`READY`** de bajo consumo | `[   R E A D Y   ]` |
| **Sin lectura** | Sensor fondo `NaN` o $\le 0$ | Texto **`READY` parpadeante lento** (distinguible del `READY` fijo = sistema OK; conserva temporizadores) | `[ SIN LECTURA ]` |

> La matriz se refresca cada 100 ms (`update_interval`) con scroll automático desactivado (`scroll_enable: false`); las velocidades `/3`, `/4` y `/8` de la tabla están calculadas sobre esa base. Los dígitos 3x5 de alineación y las flechas `^v` son gráficos dibujados con `it.line`, no fuente.

### Precedencia de estados (orden de evaluación en el `lambda`)

1. Arranque con auto estacionado → `READY`.
2. Retroceso (solo si **no** hay presencia lateral).
3. **Alineación lateral: exclusiva y prioritaria** — si ambos laterales detectan el auto, se muestra alineación aunque el fondo esté en zona STOP o aproximación.
4. STOP / inactividad post-STOP.
5. Aproximación por distancia de fondo.
6. Reposo.

> Consecuencia de seguridad: la alineación ante espejos/columnas prevalece sobre la distancia de fondo. El modo STOP solo aparece cuando los laterales están libres.

## 🎚️ Calibración de Umbrales (interfaz web `http://192.168.4.1`)

| Parámetro Web UI | ID en YAML | Default | Rango / Paso | Efecto |
| :--- | :--- | :---: | :---: | :--- |
| Tiempo Inactividad STOP (s) | `tiempo_inactividad_stop` | 10 s | 3–60 / 1 | Espera estático en STOP antes de pasar a `READY` |
| Umbral Lateral (cm) | `umbral_lateral` | 50 | 10–100 / 5 | Distancia que activa cada sensor lateral |
| Umbral Desvío Chevron (cm) | `umbral_desvio_chev` | 5 | 1–20 / 0.5 | Diferencia izq–der que dispara los chevrones de corrección (con histéresis de 1 cm) |
| Umbral Poste Lateral (cm) | `umbral_poste` | 15 | 5–40 / 1 | Algún lateral por debajo: alterna alineación con `STOP` invertido fijo (histéresis 2 cm) |
| Umbral Inicio (cm) | `umbral_inicio` | 150 | 50–250 / 10 | Distancia a la que empieza la aproximación (tope físico ~200 cm del HC-SR04) |
| Umbral Precaucion (cm) | `umbral_precaucion` | 50 | 20–100 / 5 | Scroll lento de chevrones por debajo de este valor |
| Umbral Alerta (cm) | `umbral_alerta` | 20 | 10–40 / 2 | Parpadeo de brillo + arranque estacionado por debajo de este valor |
| Umbral STOP (cm) | `umbral_stop` | 10 | 5–20 / 1 | Zona crítica con inversión parpadeante |

> Mantener coherencia: STOP < Alerta ≤ Precaución < Inicio. Valores incoherentes (p. ej. STOP > Alerta) dejan estados inalcanzables.

> El switch **Alerta Poste Lateral** (web UI, activado por defecto en cada arranque) habilita o silencia la alternancia con `STOP` sin tocar el umbral. Con el switch apagado, el peligro de poste solo se ve en los números de la pantalla de alineación.

> Histéresis de 2 cm en todos los umbrales (fondo y laterales): se entra al estado con el valor nominal y se sale recién con valor + 2 cm (p. ej. STOP entra a ≤ 10 y sale a > 12). Absorbe el ruido del HC-SR04 y evita parpadeo de estados en las fronteras.

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
                  │ GPIO16 (D0) ──────► Trigger Fondo
                  │ GPIO0  (D3) ──────► Trigger Izquierda
                  │ GPIO15 (D8) ──────► Trigger Derecha
                  │ GPIO12 (D6) ──────► Echo Fondo
                  │ GPIO5  (D1) ──────► Echo Izquierda
                  │ GPIO4  (D2) ──────► Echo Derecha
                  │ GPIO14 (D5) ──────► SPI CLK (dedicado MAX7219)
                  │ GPIO13 (D7) ──────► SPI MOSI (DIN MAX7219)
                  │ GPIO2  (D4) ──────► SPI CS (CS MAX7219)
                  └───────────────────────────────┘

```

### Nota sobre relé y láseres (lógica cableada, sin software)

El módulo relé de $5\text{V}$ y los 3 diodos láser funcionan con lógica cableada directa a la alimentación ($5\text{V}$) comandada por el interruptor magnético (*reed switch*) del portón, sin requerir intervención por software ni pines de datos del ESP8266. Al abrirse el portón se energiza el sistema completo; al cerrarse o salir el vehículo se corta por completo ($0\text{ W}$ en reposo).

### Adaptación de Voltaje (Divisor Resistivo por PIN Echo)

Debido a que los sensores HC-SR04 entregan pulsos de $5\text{V}$ en su pin `Echo` y los pines GPIO del ESP8266 operan a un máximo de $3.3\text{V}$, se debe intercalar un divisor resistivo en cada línea `Echo`:

```text
Echo HC-SR04 (5V) ───[ 1.5 kΩ ]───┬───► GPIO ESP8266 (2.73V Seguro)
                                   │
                                [ 1.8 kΩ ]
                                   │
                                  GND

```

> $5 \times 1.8/(1.5+1.8) = 2.73\text{ V}$, por encima del VIH (~2.5 V) pero con poco margen ante ruido en cable largo. Para mayor margen se recomienda **1 kΩ / 2 kΩ → 3.33 V**.

> El echo del sensor de fondo va a **GPIO12** y no a GPIO16: en ESP8266 el GPIO16 es un pin especial RTC sin soporte de interrupciones (el componente ultrasonic mide el pulso de echo por interrupción) y tampoco sirve como entrada fiable. Como salida de trigger, GPIO16 funciona sin problema.

### Distribución de Pares en Cable UTP Cat 5e

```text
 [ Par 1: Azul / Blanco-Azul ]     ──► Alimentación (+5V VCC / GND)
 [ Par 2: Naranja / Blanco-Naranja] ──► Bus SPI MAX7219 (DIN / CS)
 [ Par 3: Verde / Blanco-Verde ]   ──► Líneas de Trigger dedicadas (GPIO16/0/15)
 [ Par 4: Marrón / Blanco-Marrón ] ──► Líneas de Echo filtradas (GPIO12/5/4)

```

> Con 4 módulos encadenados a 3.3 V la doc de ESPHome advierte que puede hacer falta un level-converter si el brillo es bajo o hay flicker. Al probar en banco, si el texto sale espejado o rotado, ajustar `reverse_enable`, `rotate_chip` o `flip_x` del bloque `display:`.

### 🧰 Recomendaciones de Montaje Físico

* **Matriz LED:** a la altura de los ojos del conductor y centrada con el eje del vehículo, visible con el auto en movimiento.
* **Sensor fondo:** a la altura del paragolpe, perpendicular a la dirección de entrada, sin objetos intermedios.
* **Sensores laterales:** a la altura de los espejos retrovisores, apuntando a los flancos del auto donde el espacio es crítico (columnas, paredes).
* **Entorno:** evitar sol directo sobre los HC-SR04 y superficies absorbentes (telas, espuma) en la línea de medición; el ultrasonido rebota mejor en superficies duras y perpendiculares.
* **Reed switch + relé:** imán en la hoja móvil del portón y reed en el marco, con el relé interrumpiendo los $5\text{ V}$ generales (ver nota de lógica cableada).

### 🩺 Diagnóstico Rápido

| Síntoma | Causa probable | Acción |
| :--- | :--- | :--- |
| `[ SIN LECTURA ]` permanente | Echo fondo sin señal o auto fuera de alcance (> 2 m) | Revisar cableado/divisor de GPIO12, `esphome logs parking.yaml` |
| Texto espejado o rotado | Orden de encadenado DOUT→DIN invertido | Probar `reverse_enable`, `rotate_chip` o `flip_x` |
| Brillo bajo / flicker | Caída de tensión con 4 chips a 3.3 V o cable UTP muy largo | Level-converter, alimentar matriz con 5 V dedicados |
| No aparece el AP | Bootloop o falta de alimentación | Verificar pines de strapping (GPIO0/2/15) y fuente 5 V |
| Barra lateral inestable | Crosstalk o umbral lateral muy alto | Verificar round-robin en logs, bajar `umbral_lateral` |
| No sale de STOP a READY | Movimiento/vibración reinicia el temporizador o tiempo alto | Subir el auto a punto muerto, bajar `tiempo_inactividad_stop` |
| `« -- cm »` o valores fijos | Sensor colgado o mediana sin muestras nuevas | Revisar trigger correspondiente, reiniciar el NodeMCU |

---

## 🛠️ Compilación e Instalación

1. **Requisitos Previos:** Tener instalado **ESPHome** mediante CLI o Docker.
2. **Archivos Necesarios:**
* Archivo de configuración: `parking.yaml`.
* Fuente tipográfica bitmap: El archivo **`spleen-5x8.bdf`** debe estar en la misma carpeta que el archivo `.yaml` (solo se cargan los glifos `READYSTOP0123456789 ` para ahorrar RAM; chevrones `^v` y dígitos 3x5 son gráficos dibujados con `it.line`, no fuente).


3. **Compilación y Flasheo Inicial:**
Conecta el NodeMCU v2 por USB a tu equipo y ejecuta:
```bash
esphome run parking.yaml

```


4. **Primer Arranque y Red:**
El equipo sale de fábrica sin WiFi válido, así que levanta el Punto de Acceso `Asistente-Cochera-AP` (clave `ClaveSegura123`; cámbiala en la sección `wifi:` del YAML, igual que la clave `ota:`). Conéctate desde el teléfono, abre `http://192.168.4.1` (portal cautivo) y carga tu red WiFi. Desde entonces el equipo se une a tu red y responde siempre en **`http://garage.local`** (mDNS; funciona en iPhone, macOS, Windows 10+ y Chrome en Android). Si el router no está disponible, vuelve solo al modo AP.
Puedes conectarte desde cualquier teléfono o PC a `http://garage.local` (o a `http://192.168.4.1` en modo AP) para ajustar los umbrales de distancia y visualizar la simulación de la pantalla en tiempo real. Para actualizaciones sin USB usa `esphome run parking.yaml` con el dispositivo en red (módulo `ota:` habilitado).

5. **Verificación de Funcionamiento:**
Al energizar debe mostrar `READY` fijo. Acerca una mano al sensor de fondo: la web debe mostrar la distancia bajando en cm y la matriz el número con chevrones `^^` subiendo; al alejarla, chevrones `vv` bajando; a $\le 10\text{ cm}$ sostenidos, `STOP` invertido parpadeante. Con `esphome logs parking.yaml` puedes ver el estado interno en tiempo real.

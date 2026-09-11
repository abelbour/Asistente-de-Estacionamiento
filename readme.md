# 🚘 Asistente de Estacionamiento Autónomo para Cochera (Zero-Power-Draw Idle)

Sistema de guiado visual de alta precisión para garajes basado en **ESPHome** sobre un microcontrolador **ESP8266 (NodeMCU v2)**. El dispositivo opera de forma **100% independiente** (sin requerir Home Assistant, dependencias de nube ni conexión continua a red Wi-Fi), ofreciendo consumo de corriente nulo ($0\text{ W}$) en reposo mediante un sistema de encendido magnético autónomo.

---

## 📋 Descripción del Proyecto y Arquitectura

El asistente utiliza tres sensores ultrasónicos (HC-SR04), tres módulos de diodo láser para marcar puntos de referencia visuales en el vehículo y una matriz de LED de $8\times32$ píxeles gobernada por cuatro controladores MAX7219 encadenados (DOUT→DIN).

### Principales Características
* **Zero Power Draw en Reposo:** La alimentación general de $5\text{V}$ pasa a través de un relé comandado por un sensor magnético de lámina (reed switch) ubicado en el portón del garaje. Al cerrarse el portón o no detectar el vehículo, la alimentación se interrumpe por completo.
* **Tolerancia a Fallos de Red:** Operación autónoma en modo AP (Punto de Acceso) local. Sin retardos por desconexiones o bloqueos por fallas de Wi-Fi.
* **Alineación Simétrica de Precisión:** Barra de progreso sólida desde el centro + doble-chevrón 3x5 del lado libre en desvío + distancias 3x5 contra los márgenes con calado en negativo — para evitar colisiones laterales con espejos retrovisores o columnas.
* **Respuesta Dinámica según Sentido de Marcha:** Detecta automáticamente si el vehículo está ingresando (aproximación) o saliendo (retroceso), modificando el flujo visual de la pantalla.
* **Filtrado Peatonal:** Requiere la lectura simultánea de ambos sensores laterales para activar el modo de alineación, evitando falsas alarmas por paso de personas.
* **Suavizado de Lecturas:** Cada HC-SR04 publica en cm (`unit_of_measurement: "cm"`) con filtro de mediana de 5 muestras y disparo secuencial por turnos (round-robin cada 70 ms: fondo → izquierda → derecha, cada sensor se mide cada 210 ms) para que ningún eco interfiera con otro.
* **Modo STOP Invertido y Parpadeante:** Inversión fija de la matriz LED (`invert_on_off`) en zona crítica ($\le 10\text{ cm}$): fondo encendido / texto apagado, con parpadeo de brillo entre intensidad 12 y 7 (pico de consumo contenido).
* **Transición Inteligente a READY:** Apagado de alertas y cambio a `READY` tras un tiempo estático configurable ($10\text{ s}$) o en arranques con el vehículo ya estacionado.

---

## 📐 Estados del Sistema de Guiado Visual

| Estado | Condición de Activación | Representación Matriz LED ($8\times32$) | Simulación Web UI |
| :--- | :--- | :--- | :--- |
| **Arranque / Reposo** | Auto estático al encender o fuera de rango | Prompt retro **`READY_`** con cursor parpadeante (500 ms) | `[   R E A D Y   ]` |
| **Retroceso / Salida** | Distancia de fondo aumentando ($> +1\text{ cm}$) | Doble chevrón gráfico `vv` bajando (laterales, scroll `/3`) | `[      vv      ]` |
| **Alineación Lateral** | Ambos sensores laterales $\le 50\text{ cm}$ (+ histéresis 2 cm) | Arriba: barra de progreso sólida desde el centro a toda altura (filas 0–7, largo = magnitud del desvío); en desvío (> `umbral_desvio_chev`, histéresis 1 cm) doble-chevrón 3x5 del lado libre a la altura de los dígitos (filas 1–5), en el hueco entre centro y número, apuntando la corrección. Abajo: distancias 3x5 centradas verticalmente (filas 1–5) contra los márgenes; los píxeles tapados por la barra se ven invertidos (calado en negativo, p. ej. `30 | 30`) | `[ 45cm << : ║ 30cm ]` (valores en vivo) |
| **Aproximación Normal** | Distancia de fondo entre $50\text{ cm}$ y $150\text{ cm}$ | Distancia 3x5 a la izq + chevron `^` 7 px a la der con scroll `/4` + barra de progreso L→R (vacía en inicio, llena en stop; cala en negativo lo que tapa) | `«   120 cm   »` |
| **Precaución / Alerta**| Distancia de fondo entre $10\text{ cm}$ y $50\text{ cm}$ | Idem anterior con scroll ralentizado (`/8`) y parpadeo de brillo en Alerta ($\le 20\text{ cm}$, 12↔7 como en STOP pero sin inversión) | `«   30 cm   »` |
| **STOP Crítico** | Distancia de fondo $\le 10\text{ cm}$ | Texto **`STOP`** con inversión fija y parpadeo de brillo (12 ↔ 7) | `[  S T O P  ]` |
| **Peligro Poste Lateral** | Algún lateral $\le 15\text{ cm}$ (estando en alineación, con switch activado) | Alterna pantalla de alineación con **`STOP`** invertido fijo (500 ms cada una) | `[  S T O P  ]` / valores en vivo |
| **Inactividad Post-STOP**| Vehículo estático $\le 10\text{ cm}$ durante $> 10\text{ s}$ | Pasa de `STOP` a prompt **`READY_`** (sin alertas, brillo máximo) | `[   R E A D Y   ]` |
| **Sin lectura** | Sensor fondo `NaN` o $\le 0$ **y** sin presencia lateral (sin eco: sensor tapado o fuera de alcance) | Igual que reposo: prompt **`READY_`** (conserva temporizadores, no congela inactividad) | `[   R E A D Y   ]` |

> La matriz se refresca cada 100 ms (`update_interval`) con scroll automático desactivado (`scroll_enable: false`); las velocidades `/3`, `/4` y `/8` de la tabla están calculadas sobre esa base. Los gráficos (dígitos 3x5 y chevrones) se renderizan mediante primitivas de píxeles/mapas de bits directos (`draw_pixel_at`), sin depender de archivos de fuentes `.bdf`.

### Precedencia de estados (orden de evaluación en el `lambda`)

1. Alineación lateral (no depende del eco de fondo: prioritaria absoluta, con sub-fase Poste).
2. Lectura inválida de fondo → `READY_` como reposo (solo si no hay presencia lateral).
3. Arranque con auto estacionado → `READY`.
4. Retroceso (solo si **no** hay presencia lateral).
5. STOP / inactividad post-STOP.
6. Aproximación por distancia de fondo.
7. Reposo.

> Consecuencia de seguridad: la alineación ante espejos/columnas prevalece sobre todo, incluso si el fondo pierde el eco (`NaN`): el guiado lateral nunca se aborta por una pérdida momentánea del rebote trasero. El modo STOP de fondo solo aparece cuando los laterales están libres.

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
5. **Control de Energía:** Módulo relé HW-482 de $5\text{ V}$ (1 canal, optoacoplado, disparo LOW, bobina ~70 mA, contactos SPDT 10 A, diodo flyback incluido en placa, jumper JD-VCC de fábrica sin tocar).
6. **Protección Lógica:** Divisores de tensión ($1\text{ k}\Omega$ arriba y $1.8\text{ k}\Omega$ abajo) para adaptar las salidas de $5\text{V}$ del `Echo` de los HC-SR04 al nivel de $3.3\text{V}$ del ESP8266.
7. **Infraestructura de Cableado:** Cable de red UTP Cat 5e de cobre (un solo tendido fondo→portón, 7/8 hilos usados).

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

### ⚡ Arquitectura de Energía (corte total, $0\text{ W}$ en reposo)

Topología: fuente $5\text{ V}$/2 A y módulo HW-482 junto al MCU en el **fondo**; en el **portón** solo laterales + reed (nodo tonto). El reed va en serie con el VCC del módulo (por el Par4 del UTP) y el pin IN va puenteado a GND (disparo LOW permanente):

```text
FONDO                                              PORTÓN
fuente 5V ──┬──► COM relé (módulo HW-482, jumper JD-VCC puesto)
            ├──► Par4A ──► reed ──► Par4B ──► VCC módulo
            │       (IN del módulo puenteado a GND: siempre "disparado" con VCC)
            ├──► GND módulo
            └──► NO ──► riel 5V nodo fondo (MCU/display/fondo)
                     └──► Par1 ──► VCC laterales
```

* **Portón abierto** (imán junto al reed) → módulo energizado → IN ya en LOW → relé pega → viven laterales + fondo.
* **Portón cerrado** → reed abre → módulo apagado (hasta su LED de power muere) → todo muerto. Consumo en reposo: **cero real**.
* El reed maneja ~80 mA totales del módulo (bobina + opto + LEDs), dentro de su rating típico 0.5 A. Sin transistor ni GPIO.
* **Sin diodo externo**: el HW-482 ya trae flyback en placa en paralelo con la bobina.
* **Bulk 470 µF + 100 nF** en el riel del nodo fondo (patas cortas): cabalgan el rebote de contactos al energizar y las ráfagas WiFi del ESP8266 (~400 mA). Una fuente de 2 A sobra en promedio pero no responde en microsegundos; para eso están los capacitores locales.
* Boot de 2–4 s al abrir el portón antes del `READY_` (el auto igual espera al portón).
* Los LEDs del módulo (power + canal) sirven de diagnóstico a simple vista: con portón abierto, ambos encendidos.

### Nota sobre relé y láseres (lógica cableada, sin software)

Los 3 diodos láser funcionan con lógica cableada directa a la alimentación ($5\text{V}$) comandada por el mismo riel conmutado, sin requerir intervención por software ni pines de datos del ESP8266.

### Adaptación de Voltaje (Divisor Resistivo por PIN Echo)

Debido a que los sensores HC-SR04 entregan pulsos de $5\text{V}$ en su pin `Echo` y los pines GPIO del ESP8266 operan a un máximo de $3.3\text{V}$, se debe intercalar un divisor resistivo en cada línea `Echo`:

```text
Echo HC-SR04 (5V) ───[ 1 kΩ ]───┬───► GPIO ESP8266 (3.21V Seguro)
                                 │
                              [ 1.8 kΩ ]
                                 │
                                GND

```

> $5 \times 1.8/(1.0+1.8) = 3.21\text{ V}$: por debajo del máximo ($3.3\text{ V}$) y con buen margen sobre el VIH (~2.5 V), incluso con cable largo. Corriente del divisor en pulso: ~1.8 mA, sin carga para el Echo.

> El echo del sensor de fondo va a **GPIO12** y no a GPIO16: en ESP8266 el GPIO16 es un pin especial RTC sin soporte de interrupciones (el componente ultrasonic mide el pulso de echo por interrupción) y tampoco sirve como entrada fiable. Como salida de trigger, GPIO16 funciona sin problema.

### Distribución de Pares en Cable UTP Cat 5e (un tendido fondo→portón, 8/8 hilos)

```text
 [ Par 1: Azul / Blanco-Azul ]     ──► 5V conmutado + GND (potencia laterales)
 [ Par 2: Naranja / Blanco-Naranja] ──► Trig Izq (GPIO0) + Echo Izq (GPIO5)
 [ Par 3: Verde / Blanco-Verde ]   ──► Trig Der (GPIO15) + Echo Der (GPIO4)
 [ Par 4: Marrón / Blanco-Marrón ] ──► Lazo reed: 5V ida + retorno a VCC módulo (~80 mA)

```
(8/8 hilos usados, sin reserva.)

* Los divisores de echo van **atrás, junto al MCU** (protegen el GPIO donde entra la señal).
* Los pulsos de trigger/echo por ~6 m de Cat5e no requieren cambios de timings (retardos de ns, flancos tolerables).
* El fondo (display + MCU + sensor fondo + SPI) se cablea en corto directo, sin UTP.

> Con 4 módulos encadenados a 3.3 V la doc de ESPHome advierte que puede hacer falta un level-converter si el brillo es bajo o hay flicker. Al probar en banco, si el texto sale espejado o rotado, ajustar `reverse_enable`, `rotate_chip` o `flip_x` del bloque `display:`.

### 🧰 Recomendaciones de Montaje Físico

* **Nodo fondo (pared de fondo):** matriz LED a la altura de los ojos del conductor y centrada con el eje del vehículo + NodeMCU + sensor fondo a la altura del paragolpe, todo en corto directo (SPI y fondo sin UTP). Prever acceso USB al NodeMCU (primer flasheo y recovery).
* **Nodo portón:** laterales a la altura de los espejos retrovisores apuntando a los flancos + reed en el marco con imán en la hoja móvil (calibrar para contacto cerrado con portón abierto) + nada más (sin MCU ni relé ahí).
* **Módulo HW-482:** atrás junto al MCU, IN puenteado a GND, jumper JD-VCC de fábrica. Sus LEDs (power + canal) son los testigos: con portón abierto, ambos encendidos.
* **Sensores multi-modo:** los módulos marcados HC-SR04 con pads R4/R5 se usan en modo clásico de 2 hilos (**pads abiertos**). No puentear R4 (I2C), R5 (UART) ni R4+R5 ("1-WIRE" single-bus): ningún modo alternativo tiene soporte nativo en ESPHome y el diseño actual los necesita en modo trigger/echo.
* **Entorno:** evitar sol directo sobre los HC-SR04 y superficies absorbentes (telas, espuma) en la línea de medición; el ultrasonido rebota mejor en superficies duras y perpendiculares.

### 🩺 Diagnóstico Rápido

| Síntoma | Causa probable | Acción |
| :--- | :--- | :--- |
| No enciende al abrir el portón | Reed descalibrado o módulo sin 5V | Mirar LEDs del HW-482 (power+canal con portón abierto); verificar continuidad del lazo reed con multímetro |
| Resets al cerrar el portón o en STOP | Flyback o bulk insuficientes | Verificar el módulo HW-482 (diodo interno de fábrica) y los capacitores de 470 µF + 100 nF en el riel de fondo |
| Fondo sin respuesta (siempre `READY_`) | Echo fondo sin señal o fuera de alcance (> 2 m) | Revisar cableado/divisor de GPIO12, `esphome logs parking.yaml --device /dev/ttyUSB0` |
| Texto espejado o rotado | Orden de encadenado DOUT→DIN invertido | Probar `reverse_enable`, `rotate_chip` o `flip_x` |
| Brillo bajo / flicker | Caída de tensión con 4 chips a 3.3 V o cable UTP muy largo | Level-converter, alimentar matriz con 5 V dedicados |
| No aparece el AP | Bootloop o falta de alimentación | Verificar pines de strapping (GPIO0/2/15) y fuente 5 V |
| Barra de progreso lateral inestable | Crosstalk o umbral lateral muy alto | Verificar round-robin en logs, bajar `umbral_lateral` |
| No sale de STOP a READY | Movimiento/vibración reinicia el temporizador o tiempo alto | Subir el auto a punto muerto, bajar `tiempo_inactividad_stop` |
| `« -- cm »` o valores fijos | Sensor colgado o mediana sin muestras nuevas | Revisar trigger correspondiente, reiniciar el NodeMCU |

---

## 🛠️ Compilación e Instalación

1. **Requisitos Previos:** Tener instalado **ESPHome** mediante CLI o Docker.
2. **Archivos Necesarios:**
* Archivo de configuración: `parking.yaml` (autocontenido: los 10 glifos 5x8 `READYSTOP_` van embebidos como tablas en el `lambda`, sin `font:` ni archivos `.bdf` que subir; el viejo `spleen-5x8.bdf` ya no se referencia y puede borrarse del proyecto).


3. **Compilación y Flasheo Inicial:**
Conecta el NodeMCU v2 por USB a tu equipo y ejecuta:
```bash
esphome run parking.yaml

```


4. **Primer Arranque y Red:**
El YAML no trae ninguna credencial: ni WiFi ni OTA. Al primer arranque el equipo levanta el Punto de Acceso abierto `Asistente-Cochera-AP` (solo existe sin router, para el setup inicial). Conéctate desde el teléfono, abre `http://192.168.4.1` (portal cautivo) y carga tu red WiFi: queda guardada en flash y no hay que repetirlo. Desde entonces el equipo se une a tu red y responde siempre en **`http://garage.local`** (mDNS; funciona en iPhone, macOS, Windows 10+ y Chrome en Android). Si el router no está disponible, vuelve solo al modo AP. Nota: el OTA queda sin clave (cualquiera en tu LAN podría flashear el equipo); en una red hogareña normal es aceptable, en red compartida conviene agregar `password` al bloque `ota:`.
Puedes conectarte desde cualquier teléfono o PC a `http://garage.local` (o a `http://192.168.4.1` en modo AP) para ajustar los umbrales de distancia y visualizar la simulación de la pantalla en tiempo real. Para actualizaciones sin USB usa `esphome run parking.yaml` con el dispositivo en red (módulo `ota:` habilitado).

5. **Verificación de Funcionamiento:**
Al energizar debe mostrar el prompt `READY_` (con cursor parpadeante; si no hay eco, igual: sin estado de fallo dedicado). En la web (`Sensor Fondo/Izquierda/Derecha`, en cm, 0 decimales, ~5 Hz por round-robin) verifica las lecturas en vivo: `Unknown` = sin eco (normal sin obstáculo), y la mediana tarda ~1 s en asentarse al mover la mano (normal, no es lag de red). Prueba de rango mínimo: mano a 5, 10, 15 y 30 cm del fondo — si a ≤10 cm lee estable, los módulos responden como clásico y el tema R4/R5 queda archivado. Prueba de potencia: con portón cerrado, multímetro en el riel = 0 V; abierto, secuencia mano→barra L→R→`STOP`. Con `esphome logs parking.yaml --device /dev/ttyUSB0` (ajusta el puerto) puedes ver el estado interno en tiempo real.

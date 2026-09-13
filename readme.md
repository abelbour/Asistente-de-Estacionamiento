# 🚘 Asistente de Estacionamiento Autónomo para Cochera (Zero-Power-Draw Idle)

Sistema de guiado visual de alta precisión para garajes basado en **ESPHome** sobre un microcontrolador **ESP8266 (NodeMCU v2)**. El dispositivo opera de forma **100% independiente** (sin requerir Home Assistant, dependencias de nube ni conexión continua a red Wi-Fi), ofreciendo consumo de corriente nulo ($0\text{ W}$) en reposo mediante un sistema de encendido magnético autónomo.

---

## 📋 Descripción del Proyecto y Arquitectura

El asistente utiliza tres sensores ultrasónicos (HC-SR04), tres módulos de diodo láser para marcar puntos de referencia visuales en el vehículo y una matriz de LED de $8\times32$ píxeles gobernada por cuatro controladores MAX7219 encadenados (DOUT→DIN).

### Principales Características
* **Zero Power Draw en Reposo:** La alimentación general de $5\text{V}$ pasa a través de un relé comandado por un sensor magnético de lámina (reed switch) ubicado en el portón del garaje. Al cerrarse el portón o no detectar el vehículo, la alimentación se interrumpe por completo.
* **Tolerancia a Fallos de Red:** Operación autónoma en modo AP (Punto de Acceso) local. Sin retardos por desconexiones o bloqueos por fallas de Wi-Fi.
* **Alineación Simétrica de Precisión:** Barra de progreso sólida desde el centro + marquesina 7x5 del usuario del lado libre en desvío + distancias 3x6 contra los márgenes con calado en negativo — para evitar colisiones laterales con espejos retrovisores o columnas.
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
| **Retroceso / Salida** | Distancia de fondo aumentando ($> +1\text{ cm}$) | Misma marquesina `^` invertida verticalmente (baja) en laterales x=1 y x=24, ~300 ms/frame | `[      vv      ]` |
| **Alineación Lateral** | Ambos sensores laterales $\le 50\text{ cm}$ (+ histéresis 2 cm) | Arriba: barra de progreso sólida desde el centro a toda altura (filas 0–7, largo = magnitud del desvío); en desvío (> `umbral_desvio_chev`, histéresis 1 cm) marquesina 7×5 del usuario a ~100 ms/frame (tablas `CHEVDER` verbatim del GIF, espejada a la izquierda) del lado libre a la altura de los dígitos (filas 1–5), apuntando la corrección. Abajo: distancias 3x5 centradas verticalmente (filas 1–5) contra los márgenes; los píxeles tapados por la barra se ven invertidos (calado en negativo, p. ej. `30 | 30`) | `[ 45cm << : ║ 30cm ]` (valores en vivo) |
| **Aproximación Normal** | Distancia de fondo entre $50\text{ cm}$ y $150\text{ cm}$ | Distancia 3x6 a la izq + marquesina vertical `^` 7×8 del usuario (tablas `CHEVUP` verbatim del GIF, ~100 ms/frame; x2 en precaución/alerta) + barra de progreso 0–23 L→R (100% en x=23, col 24 libre; cala en negativo números, no el chevrón) | `«   120 cm   »` |
| **Precaución / Alerta**| Distancia de fondo entre $10\text{ cm}$ y $50\text{ cm}$ | Idem anterior con marquesina a mitad de velocidad (x2) y parpadeo de brillo en Alerta ($\le 20\text{ cm}$, 12↔7 como en STOP pero sin inversión) | `«   30 cm   »` |
| **STOP Crítico** | Distancia de fondo $\le 10\text{ cm}$ | Texto **`STOP`** gigante como mapa de bits libre de 32×8 a todo ancho (tabla `STOP32`, bit 31 = x0) con inversión fija y parpadeo de brillo (12 ↔ 7) | `[  S T O P  ]` |
| **Peligro Poste Lateral** | Algún lateral $\le 15\text{ cm}$ (estando en alineación, con switch activado) | Alterna pantalla de alineación con **`STOP`** invertido fijo (500 ms cada una) | `[  S T O P  ]` / valores en vivo |
| **Inactividad Post-STOP**| Vehículo estático $\le 10\text{ cm}$ durante $> 10\text{ s}$ | Pasa de `STOP` a prompt **`READY_`** (sin alertas, brillo máximo) | `[   R E A D Y   ]` |
| **Sin lectura** | Sensor fondo `NaN` o $\le 0$ **y** sin presencia lateral (sin eco: sensor tapado o fuera de alcance) | Igual que reposo: prompt **`READY_`** (conserva temporizadores, no congela inactividad) | `[   R E A D Y   ]` |

> La matriz se refresca cada 100 ms (`update_interval`); la marquesina `^` corre con velocidad continua interpolada entre umbrales (1 frame/tick en `Umbral Inicio`, 0.5 en `Umbral Precaución`, sin saltos; fase fraccional acumulada) y la marquesina lateral a ~100 ms/frame. Los gráficos (dígitos 3x6 del usuario en `num.gif`, chevrones y STOP gigante) se renderizan mediante primitivas de píxeles/mapas de bits directos (`draw_pixel_at`), sin depender de archivos de fuentes `.bdf`. Los números contra el margen derecho van 1 px más a la derecha para quedar contra el borde (2 dígitos en x=25, 1 dígito en x=29).

### Máquina de estados (enum `Modo` en el `lambda` del display)

El `lambda` está estructurado en tres fases: **leer entradas → `evalua_modo()` → `dibuja_modo()` + publicar textos**. Los 12 modos del enum son: `REPOSO_READY`, `ARRANQUE_LISTO`, `RETROCESO`, `ALIN_CENTRADA`, `ALIN_DESV_DER`, `ALIN_DESV_IZQ`, `APROX_NORMAL`, `APROX_PRECAUCION`, `APROX_ALERTA`, `STOP_CRITICO`, `POSTE_PELIGRO`, `INACTIVIDAD_READY`. La precedencia y la histéresis (±2 cm, ±1 cm en desvíos) son las de la tabla de abajo; el refactor desde la cadena de `if…return` es bit-idéntico en comportamiento (mismos píxeles y mismos textos ante las mismas lecturas).

### Modo Prueba (`select` "Modo Prueba" en la UI web de ESPHome)

Entidad `select` template con `Automático` (default, **sin `restore_value`**: tras un corte del portón siempre arranca en `Automático`, nunca queda un estado forzado) + 12 opciones forzadas: `READY`, `Retroceso`, `Alineación centrada`, `Alineación corregir izq/der`, `Aproximación Normal/Precaución/Alerta`, `STOP`, `Peligro Poste`, `Inactividad`, `Manual`.

* En prueba se **pausa el round-robin** de los HC-SR04 (el `interval` de 70 ms retorna sin disparar) y se publican **distancias sintéticas** coherentes (p. ej. alineación 30/30 o 45/30, aproximación 120/35/15, STOP 8), así el SVG y las tarjetas del dashboard muestran valores consistentes con la matriz.
* La opción **Manual** usa en cambio las distancias de los `number` **Prueba Fondo/Izquierda/Derecha** (`0 = sin eco`, sin `restore_value`) y las pasa por la **máquina de estados real** (con su histéresis y temporizadores), republicando a los sensores solo cuando alguna cambia. Los sintéticos (fijos y manuales) se publican con bypass de filtros (`internal_send_state_to_frontend`): ya vienen en cm y `publish_state` los re-escalaría x100 + mediana. Sirve para ver cómo reacciona el sistema ante cualquier combinación, incluyendo laterales desparejos o pérdida de eco. En el dashboard (Estado avanzado) aparecen sus sliders solo con `Manual` activo.
* No se tocan histéresis ni temporizadores reales; al volver a `Automático` se resiembran `distancia_fondo_previa` y `estatico_desde_ms` para no disparar falsos retrocesos ni saltos a inactividad.
* `Modo del Sistema` lleva el sufijo `(Prueba)`; parpadeos y alternancias usan los mismos contadores que en producción, así la prueba se ve igual que el estado real.

### Botón Reiniciar

Entidad `button` (`platform: restart`, nombre `Reiniciar`) en la interfaz web propia de ESPHome: reinicia el NodeMCU desde el navegador (útil tras cambiar umbrales o para salir de cualquier estado sin cortar el portón).

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
| Ancho Portón (cm) | `ancho_porton` | 200 | 100–400 / 5 | Solo esquema del dashboard (persiste reboot) |
| Ancho Auto (cm) | `ancho_auto` | 170 | 100–250 / 5 | Solo esquema del dashboard (persiste reboot) |
| Ancho Auto con espejos (cm) | `ancho_auto_espejos` | 190 | 100–300 / 1 | Total con espejos, solo esquema (debe superar carrocería; persiste reboot) |
| Nombre Cochera | `nombre_cochera` (text) | Cochera | 1–64 car. | Título del dashboard web (solo etiqueta; persiste reboot) |
| Largo Garage (cm) | `largo_garage` | 500 | 300–800 / 10 | Solo esquema del dashboard (persiste reboot) |
| Largo Auto (cm) | `largo_auto` | 420 | 250–600 / 10 | Solo esquema del dashboard (persiste reboot) |
| Prueba Fondo (cm) | `prueba_fondo` | 0 = sin eco | 0–400 / 1 | Distancia manual de fondo en Modo Prueba `Manual` (sin restore) |
| Prueba Izquierda/Der (cm) | `prueba_izq`/`prueba_der` | 0 = sin eco | 0–200 / 1 | Distancias manuales laterales en Modo Prueba `Manual` (sin restore) |

> Mantener coherencia: STOP < Alerta ≤ Precaución < Inicio. Valores incoherentes (p. ej. STOP > Alerta) dejan estados inalcanzables.

> El switch **Alerta Poste Lateral** (web UI, activado en el primer arranque) habilita o silencia la alternancia con `STOP` sin tocar el umbral. Con el switch apagado, el peligro de poste solo se ve en los números de la pantalla de alineación.

> Umbrales, switch y dimensiones persisten reboot (`restore_value`): lo calibrado por web sobrevive al corte del portón.

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
7. **Infraestructura de Cableado:** Cable de red UTP Cat 5e de cobre (un solo tendido fondo→portón, 8/8 hilos usados).

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
| Valores congelados en web | Sensor colgado o mediana sin muestras nuevas | Revisar trigger correspondiente, reiniciar el NodeMCU |

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

---

## 📱 Dashboard web (`web/`)

Interfaz de monitoreo amigable (mobile-first, tema claro/oscuro, español) que habla con el equipo vía REST + Event Source de ESPHome (`https://esphome.io/web-api/`). Carpeta autocontenida y sin compilación: `index.html` + `app.js` + `styles.css` (+ `manifest.webmanifest`, `sw.js`, `icon.svg` para PWA/QR). Copiarla al repo de hosting cuando se defina.

* Requiere en firmware `web_server.allowed_origins: ["*"]` + `enable_private_network_access: true` (ya configurados) + reflasheo; si no, el navegador bloquea todo por CORS/PNA. La interfaz propia del equipo está fijada en `version: 1` (liviana para el ESP8266, solo respaldo/depuración; la API no depende de la versión, pero v1 se elimina en 2027.1.0 y habrá que migrar a v2).
* Acceso por `http://garage.local` (editable, con escaneo de subred y `?garage=` para compartir). Si la página va por HTTPS, permitir contenido inseguro para el sitio (patrón probado del reloj).
* Topbar mínima (título = `Nombre Cochera` + LEDs **TX**/**RX** + menú ⋮; sin subtítulo): la pill de conexión solo se muestra sin conexión, conectado vive dentro del menú. Menú con Conexión (con estado), Registro (diálogo modal con Limpiar), Sistema, Estado avanzado, Compartir, Tema y Configuración. Nombre también en Configuración → Cochera y en la pestaña.
* Cuerpo directo sin tarjetas ni títulos: matriz 32×8 con **puntos redondos** arriba (color de LED configurable en Configuración → Pantalla, solo vista local), esquema cenital abajo y badge de modo centrado debajo (sin texto de simulacro ni pie de página).
* Esquema cenital a escala: fondo arriba y portón abajo, interior 20% más ancho que el portón (vano) con muros gruesos en un solo trazo (uniones suaves, al ras de los postes), umbrales como anillos por tramo entre umbrales (sin líneas, cada uno con su color sombreado y su distancia en vertical al borde interno intercalado izq/der, centrada en su tramo y sin "cm"), lecturas 50% más grandes y FUERA de las paredes (laterales a los costados, fondo arriba y con su línea al centro) en verde→amarillo→rojo según desvío y umbrales (fondo por tier; laterales: lado cercano ≤ poste en rojo, desvío > chevron en amarillo de ese lado), cotas SÓLIDAS solo para lo medido por sensores (con terminaciones perpendiculares) y línea DE PUNTOS solo entre cada cota y su número para vincularlas; dimensiones fijas sin líneas y semitransparentes, auto que se funde a transparente donde asome del portón (máscara con degradado), zona objetivo gris (morro a Umbral STOP, tamaño = Largo/Ancho Auto, sombreada con borde punteado semitransparente) y auto con los paths vectoriales reales de `temp/car.svg` a escala (sin PNGs externos ni gradientes, ver `web/car.svg`; carrocería = `Ancho Auto`, total = `Ancho Auto con espejos` del firmware, solo vista: los sensores miran por debajo de los espejos; el tope lateral también usa el total con espejos) posicionado por fondo + laterales (oculto sin detección), cotas numéricas junto a cada elemento (sin palabras, rotadas donde va) y las 3 distancias en grande sobre el gráfico (se eliminaron las tarjetas individuales). Si el fondo aún no tiene eco pero los laterales sí, el auto se estima con la cola en el portón; sin ningún eco solo queda la zona objetivo.
* Diálogo **Configuración** (sliders + número por campo, con **bloqueo** ante umbrales incoherentes o espejos ≤ carrocería): umbrales, Alineación, `Alerta Poste Lateral`, Dimensiones (incl. con espejos), Pantalla y Cochera (nombre).
* Modal **Sistema**: IP, WiFi (SSID/señal/BSSID/MAC), versión ESPHome, fecha de firmware, uptime, motivo de reinicio, CPU, heap/bloque/fragmentación/loop, mensajes y latencia (telemetría `debug` + `uptime` + `wifi_signal` + `wifi_info` + `version` del firmware).
* Diálogo **Estado avanzado** (estilo del proyecto del reloj): `select` **Modo Prueba** (las 13 opciones del firmware vía `POST /select/Modo Prueba/set?option=`, con badge `PRUEBA · …` en la tarjeta Posición; con `Manual` aparecen sliders de distancias manuales) + botón **Reiniciar equipo** (con confirmación, `POST /button/Reiniciar/press`, como el `modalRebootBtn` del reloj).
* Diálogo **Firmware**: compara la fecha de compilación instalada contra lo publicado en GitHub (configurar `FW_REPO` en `app.js`) y permite subir un `.bin` directo al equipo (`POST /update`, plataforma OTA `web_server` habilitada en firmware).

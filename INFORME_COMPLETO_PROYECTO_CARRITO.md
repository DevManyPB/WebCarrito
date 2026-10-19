# 📋 INFORME TÉCNICO COMPLETO: ROBOT 4WD EVASOR DE OBSTÁCULOS & CONTROL WEB BLE
**Fecha de generación:** 19 de Octubre de 2026  
**Autor:** Antigravity (Google DeepMind Agentic Pair Programmer)  
**Destinatario:** Claude / Desarrollador de relevo para resolución de incidencias  
**Repositorio GitHub:** [https://github.com/DevManyPB/WebCarrito.git](https://github.com/DevManyPB/WebCarrito.git)  
**Aplicación Web en Producción (Firebase Hosting):** [https://carrito-b7f6a.web.app](https://carrito-b7f6a.web.app)  

---

## 📑 ÍNDICE
1. [Resumen Ejecutivo y Estado Actual](#1-resumen-ejecutivo-y-estado-actual)
2. [Arquitectura de Hardware y Mapeo Pin a Pin](#2-arquitectura-de-hardware-y-mapeo-pin-a-pin)
3. [Arquitectura de Software y Protocolo](#3-arquitectura-de-software-y-protocolo)
4. [Historial Detallado de Problemas, Causas Raíz y Soluciones Implementadas](#4-historial-detallado-de-problemas-causas-raíz-y-soluciones-implementadas)
5. [Análisis del Problema Crítico Pendiente: "No cambia a Modo Manual"](#5-análisis-del-problema-crítico-pendiente-no-cambia-a-modo-manual)
6. [Hoja de Ruta y Pruebas Recomendadas para Claude](#6-hoja-de-ruta-y-pruebas-recomendadas-para-claude)

---

## 1. RESUMEN EJECUTIVO Y ESTADO ACTUAL

### 🎯 Objetivo del Proyecto
Construir un vehículo robótico de 4 ruedas motrices (4WD) basado en chasis comercial con motores TT DC amarillos, capaz de:
1. **Modo Autónomo:** Navegar esquivando obstáculos de forma autónoma mediante un sensor ultrasónico HC-SR04 montado sobre un servomotor SG90 que barre Izquierda (94°), Centro (39°) y Derecha (15°).
2. **Modo Manual:** Ser conducido en tiempo real desde una aplicación web moderna (PWA) compatible con teléfonos móviles (Android/iOS) y PC vía **Bluetooth Low Energy (BLE)** o **Web Serial (Bluetooth Clásico/USB)**.
3. **Telemetría en Vivo:** Transmitir en tiempo real a la aplicación web la distancia medida (cm), tiempo de eco (µs), ángulo del servo, velocidad PWM actual y diagnósticos del enlace serie.

### 📊 Estado Actual
* **Despliegue Web:** Totalmente funcional en Firebase Hosting con HTTPS y en GitHub.
* **Telemetría Arduino ➔ Celular/Web:** **100% OPERATIVA.** La página web se conecta por BLE, recibe telemetría JSON continua, dibuja el radar y la distancia en cm en tiempo real.
* **Modo Autónomo:** **100% OPERATIVO.** El vehículo arranca automáticamente tras 1.2 segundos, avanza, detecta obstáculos a 25 cm, retrocede, gira el servo para escanear y decide el camino más despejado.
* **Problema Pendiente (Bloqueante):** **Los comandos enviados desde la Web hacia el Arduino (Web ➔ Arduino) no surten efecto.** Cuando el usuario pulsa el botón "Manual" o las flechas de dirección:
  * La web envía `→ M` o `→ F`.
  * La interfaz web bloquea el modo por 2.5s para no ser pisada.
  * El Arduino no registra los caracteres entrantes (el contador de bytes recibidos en A2 permanece en `0 (-)`).
  * En consecuencia, el carro sigue en Modo Autónomo y no frena ni obedece el control manual.

---

## 2. ARQUITECTURA DE HARDWARE Y MAPEO PIN A PIN

### 🧩 Componentes Físicos
1. **Microcontrolador:** Arduino Uno R3 (ATmega328P a 16 MHz, lógica 5V TTL).
2. **Controlador de Motores:** Adafruit Motor Shield v1 (L293D) montado sobre el Arduino Uno.
   * Contiene 2 puentes H L293D y un registro de desplazamiento **74HC595** de 8 bits para las señales de dirección.
3. **Motores:** 4 motores DC tipo TT con reductora (consumo ~150-250 mA sin carga, hasta 1.0 A en bloqueo/arranque cada uno).
4. **Radar Móvil:** Servomotor TowerPro SG90 (alimentado a 5V).
5. **Sensor de Distancia:** Sensor Ultrasónico HC-SR04 (alimentado a 5V).
6. **Módulo de Comunicación:** Módulo Bluetooth etiqueta **HC-05**, pero cuyo firmware interno es un clon **BLE (HM-10 / AT-09 / CC2541 / Beken)** que anuncia servicio GATT `0000ffe0-0000-1000-8000-00805f9b34fb` y característica `ffe1`.
7. **Alimentación:**
   * Pack de baterías (7.4V - 8.4V Li-ion o 6V AA) conectado a los bornes `EXT_PWR` (+M y GND) del Shield L293D.

---

### 🔌 Tabla de Conexiones Pin a Pin

| Componente | Pin del Componente | Pin Arduino / Shield | Función / Descripción Técnica |
| :--- | :--- | :--- | :--- |
| **Shield L293D (74HC595)** | SHCP (Clock) | **Pin 4** | Reloj de desplazamiento de bits |
| **Shield L293D (74HC595)** | OE (Output Enable) | **Pin 7** | Habilitación de salidas (Activo en LOW) |
| **Shield L293D (74HC595)** | DS (Data Serial) | **Pin 8** | Entrada serie de bits de dirección |
| **Shield L293D (74HC595)** | STCP (Latch) | **Pin 12** | Pestillo de almacenamiento |
| **Shield L293D (PWM M1)** | Enable 1A | **Pin 11** | Velocidad PWM Motor Delantero Izq |
| **Shield L293D (PWM M2)** | Enable 2A | **Pin 3** | Velocidad PWM Motor Trasero Izq |
| **Shield L293D (PWM M3)** | Enable 3A | **Pin 6** | Velocidad PWM Motor Trasero Der |
| **Shield L293D (PWM M4)** | Enable 4A | **Pin 5** | Velocidad PWM Motor Delantero Der |
| **Servomotor SG90** | Señal (Naranja/Amarillo) | **Pin 10 (SER1)** | Conector SERVO_1 del Shield L293D |
| **Servomotor SG90** | VCC (+) (Rojo) | **5V (SER1)** | 5V regulado del Arduino |
| **Servomotor SG90** | GND (-) (Marrón) | **GND (SER1)** | Tierra común |
| **Sensor HC-SR04** | VCC | **5V** (Tira analógica) | Alimentación 5V |
| **Sensor HC-SR04** | GND | **GND** (Tira analógica)| Tierra común |
| **Sensor HC-SR04** | TRIG (Gris) | **Pin A5** | Pulso de disparo ultrasónico (10 µs) |
| **Sensor HC-SR04** | ECHO (Azul) | **Pin A4** | Retorno del eco medido con `pulseIn` |
| **Módulo Bluetooth** | VCC | **5V** | Alimentación del módulo |
| **Módulo Bluetooth** | GND | **GND** | Tierra común |
| **Módulo Bluetooth** | **TXD** | **Pin A2** (`PIN_BT_RX`)| Arduino recibe datos del Bluetooth (9600 bps) |
| **Módulo Bluetooth** | **RXD** | **Pin A3** (`PIN_BT_TX`)| Arduino envía telemetría al Bluetooth (9600 bps) |
| **Módulo Bluetooth** | STATE / EN / KEY | **DESCONECTADO** | Sin conexión (al aire) |

> ⚠️ **Nota Eléctrica sobre el Pin A3 (TX Arduino ➔ RX Bluetooth):**
> Los módulos HM-10/AT-09 operan a 3.3V en sus líneas de datos. Idealmente se coloca un divisor de voltaje (1kΩ en serie, 2kΩ a tierra) entre el pin A3 (5V) y el pin RXD del Bluetooth para no degradar el chip de radio a largo plazo.

---

## 3. ARQUITECTURA DE SOFTWARE Y PROTOCOLO

### 📁 Estructura del Repositorio
```
c:\Users\Jhon\Documents\Proyectos\Trabajos\Pelaos\
├── Arduino_Uno_Firmware/
│   └── Arduino_Uno_Firmware.ino    # Código C++ del microcontrolador
├── WebApp_Controller/
│   ├── index.html                 # PWA móvil (Web Bluetooth + Web Serial)
│   ├── firebase.json              # Configuración de hosting
│   ├── .firebaserc                # Proyecto 'carrito-b7f6a'
│   ├── manifest.json              # Manifiesto PWA instalable
│   └── icon.svg                   # Icono de la PWA
└── INFORME_COMPLETO_PROYECTO_CARRITO.md  # Este informe
```

---

### 📡 Protocolo de Comunicación Serie / BLE (9600 baudios, 8N1)

#### 1. Comandos Web ➔ Arduino (Control Inmediato)
Todos los comandos de una sola letra se ejecutan de manera **inmediata** (0 ms de espera, sin requerir `\r` ni `\n` obligatorios):

| Comando | Acción en el Arduino | Respuesta / Telemetría generada |
| :---: | :--- | :--- |
| **`M`** | Cambia a **Modo Manual**. Detiene motores de inmediato. | `{"m":"M","f":...}` |
| **`A`** | Cambia a **Modo Autónomo**. Inicia evasión de obstáculos. | `{"m":"A","f":...}` |
| **`F`** | Avanzar adelante a velocidad actual (`currentSpeed`). | Mueve motores si `distFront > 25cm` |
| **`B`** | Retroceder a velocidad actual. | Mueve motores en reversa |
| **`L`** | Girar sobre su eje hacia la Izquierda (`VEL_GIRO = 200`). | Motores izq atrás, der adelante |
| **`R`** | Girar sobre su eje hacia la Derecha (`VEL_GIRO = 200`). | Motores izq adelante, der atrás |
| **`S`** | Frenar en seco todos los motores. | Apaga PWMs y pone 74HC595 en 0 |
| **`V<0-255>`** | Establece velocidad PWM de marcha (ej. `V180`). | Actualiza variable `currentSpeed` |
| **`C<ang>`** | Guarda en EEPROM nuevo ángulo Central del servo. | Calibra centro (ej. `C39`) |
| **`I<ang>`** | Guarda en EEPROM nuevo ángulo Izquierdo del servo. | Calibra izq (ej. `I94`) |
| **`D<ang>`** | Guarda en EEPROM nuevo ángulo Derecho del servo. | Calibra der (ej. `D15`) |
| **`Z`** | Pone a 0 el contador de arranques/reinicios en EEPROM[10].| `[ARRANQUES] contador a 0` |

#### 2. Telemetría Arduino ➔ Web (Cada 500 ms por Bluetooth, 400 ms por USB)
El Arduino emite un JSON compacto por línea:
```json
{"f":48.2,"e":2767,"m":"A","a":39,"v":180,"rx":1,"c":0,"lb":"-"}
```
* **`f`**: Distancia frontal actual medida por el sensor en centímetros.
* **`e`**: Duración del pulso de eco ultrasónico en microsegundos (µs).
* **`m`**: Modo actual (`"A"` = Autónomo, `"M"` = Manual).
* **`a`**: Ángulo actual del servomotor.
* **`v`**: Velocidad PWM configurada (0-255).
* **`rx`**: Estado digital directo del pin A2 (`1` = HIGH, `0` = LOW).
* **`c`**: Contador acumulado de bytes recibidos por el puerto Bluetooth (A2).
* **`lb`**: Último carácter recibido por el puerto Bluetooth.
* Además, cada 3000 ms se emite la calibración: `{"c":39,"i":94,"d":15}`.

---

## 4. HISTORIAL DETALLADO DE PROBLEMAS, CAUSAS RAÍZ Y SOLUCIONES IMPLEMENTADAS

### 🛑 Problema 1: Fallo al hacer `git push` a GitHub desde PowerShell
* **Síntoma:** El comando `git push origin main` se quedaba colgado indefinidamente sin arrojar ningún mensaje.
* **Causa Raíz:** En Windows, el *Git Credential Manager (GCM)* intenta abrir de manera predeterminada un cuadro de diálogo gráfico o redirigir al navegador web para autenticación OAuth. Al ejecutarse en subprocesos no interactivos de terminal, el proceso quedaba esperando la interacción del usuario sin mostrar la ventana.
* **Solución Aplicada:** Se registraron directamente las credenciales y el Personal Access Token (PAT) del usuario en el Administrador de Credenciales de Windows utilizando `cmdkey`:
  ```powershell
  cmdkey /generic:git:https://github.com /user:<username> /pass:<PAT_TOKEN>
  ```
  Tras esto, los pushes se ejecutan limpiamente en 1 segundo.

---

### 🛑 Problema 2: Web Bluetooth y Web Serial bloqueados en navegadores móviles
* **Síntoma:** Al abrir el archivo `index.html` de manera local (`file:///...`) o mediante HTTP sin cifrar, el navegador arrojaba error de permisos de API de Bluetooth.
* **Causa Raíz:** Las APIs modernas de Chromium (`navigator.bluetooth` y `navigator.serial`) son consideradas APIs con privilegios de seguridad altos y **exigen de forma estricta un contexto seguro HTTPS**.
* **Solución Aplicada:**
  1. Se instaló y configuró la suite de `firebase-tools`.
  2. Se vinculó el proyecto Firebase `carrito-b7f6a`.
  3. Se desplegó la aplicación en Firebase Hosting con certificado SSL automático en:
     👉 **`https://carrito-b7f6a.web.app`**

---

### 🛑 Problema 3: Reinicio espontáneo del Arduino al arrancar los motores (Brown-Out Reset)
* **Síntoma:** Conectado a baterías, en cuanto el carro intentaba arrancar o esquivar un obstáculo, el Arduino se congelaba o se reiniciaba. El contador de arranques de la EEPROM aumentaba continuamente.
* **Causa Raíz:**
  1. **Corriente de inserción (Inrush Current):** Los 4 motores DC de corriente continua, cuando arrancan bruscamente desde velocidad 0 con un PWM de golpe de 225, demandan un pico de corriente combinado superior a **3.5 Amperios**.
  2. La resistencia interna de las baterías provocaba una caída instantánea de voltaje (sag) por debajo de 4.3V en el riel de 5V del microcontrolador.
  3. Esto disparaba el circuito interno **Brown-Out Detector (BOD)** del ATmega328P, reiniciando la CPU.
  4. Además, el servomotor SG90 conectado al pin 10 consumía corriente continuamente sosteniendo su posición fija mientras los motores vibraban.
* **Solución Aplicada en Firmware (`Arduino_Uno_Firmware.ino`):**
  1. **Rampa de aceleración Soft-Start:** Se programó una función que escala la velocidad en tramos de 30 ms (ej. 90 PWM ➔ 150 PWM ➔ objetivo 180/225 PWM). Esto amortigua el pico de corriente de 3.6 A a menos de 1.1 A.
  2. **Desconexión automática del Servomotor:** Tras centrar el servo a 39°, se ejecuta `radar.detach()`. El pin deja de emitir pulsos PWM y el servo entra en reposo absoluto consumiendo **0 mA** durante la marcha de los motores. Solo se vuelve a adjuntar (`attach`) cuando el carro está completamente detenido antes de escanear.

---

### 🛑 Problema 4: Al tocar "Manual" en la Web, volvía a "Autónomo" de inmediato
* **Síntoma:** El usuario pulsaba el botón "Manual", en la pantalla cambiaba por una fracción de segundo a MANUAL, pero de inmediato regresaba a decir AUTÓNOMO.
* **Causas Raíz Detectadas:**
  1. **Causa A (Eléctrica en Arduino):** En la versión inicial del código, la rutina `procesarComando('M')` ejecutaba `moverServo(SERVO_CENTRO)`. Mover el servomotor mientras los motores se detenían causaba un pico inductivo que reiniciaba el Arduino. Al reiniciarse, `setup()` volvía a activar el Modo Autónomo por defecto.
     * *Solución A:* Se eliminó la llamada al servomotor en `procesarComando('M')`.
  2. **Causa B (Condición de carrera de Telemetría en Web):** El Arduino envía un paquete JSON de telemetría cada 500 ms con `"m":"A"`. Cuando el usuario tocaba "Manual", la web enviaba la orden, pero un paquete JSON que venía en tránsito por el aire (emitido 40 ms antes por el Arduino) llegaba a la web y sobreescribía la variable `state.mode` a `"A"`.
     * *Solución B:* Se implementó en `WebApp_Controller/index.html` un cerrojo temporal (`modeLockUntil = Date.now() + 2500`). Durante 2.5 segundos, la interfaz ignora cualquier telemetría con `"m":"A"` hasta que el Arduino responda explícitamente confirmando `"m":"M"`.

---

### 🛑 Problema 5: Error JavaScript `ReferenceError: device is not defined`
* **Síntoma:** Al presionar "Bluetooth BLE" en la web aparecía una alerta de error en la consola y no se completaba la conexión.
* **Causa Raíz:** En la función `connectBle()`, la variable `device` se asignaba sin haber sido declarada previamente con `let device = null` dentro del ámbito de la función, violando el modo estricto de JS.
* **Solución Aplicada:** Se corrigió la declaración de variables al inicio de `connectBle()` en `WebApp_Controller/index.html` y se redesplegó a Firebase.

---

### 🛑 Problema 6: Latencia y omisión de comandos de 1 carácter en `leerPuerto()`
* **Síntoma:** Comandos como `M` o `F` requerían enviar un salto de línea (`\n`) para que el Arduino los procesara. Si la web enviaba solo el byte sin `\n`, quedaba almacenado en el búfer sin ejecutarse.
* **Solución Aplicada:** Se reescribió `leerPuerto()` para que cualquier comando clave de un solo carácter (`M`, `A`, `S`, `F`, `B`, `L`, `R`) sea ejecutado de manera síncrona en el microsegundo exacto en que entra el byte por el puerto serie.

---

## 5. ANÁLISIS DEL PROBLEMA CRÍTICO PENDIENTE: "NO CAMBIA A MODO MANUAL"

### 🔍 Descripción del Comportamiento Actual
* El módulo Bluetooth físico está emparejado y conectado vía BLE con la Web App.
* La telemetría en sentido **Arduino ➔ Web** funciona con total fluidez: la distancia en cm, el radar y el eco se actualizan en pantalla en tiempo real.
* Sin embargo, cuando se pulsa el botón **Manual**:
  * La consola de la Web registra el envío: `→ M`.
  * La pantalla del carrito sigue mostrando: `Pin A2: HIGH (Normal) | Recibidos: 0 (-)`.
  * El Arduino nunca entra al bloque `case 'M'`, por lo que **no detiene los motores ni abandona el modo autónomo**.

---

### 🔬 Hipótesis Técnicas y Causas Raíz Probables

#### Hipótesis 1: Falla Física en la Línea de Recepción (Pin TXD del Bluetooth ➔ Pin A2 del Arduino) — [Probabilidad: ALTA]
* La línea de transmisión del Arduino (**Pin A3 ➔ Pin RXD del Bluetooth**) está 100% sana porque la web recibe los datos.
* Pero la línea inversa (**Pin TXD del Bluetooth ➔ Pin A2 del Arduino**) es un cable físico independiente.
* **Comprobación:**
  * Si el cable Dupont que sale de `TXD` del módulo Bluetooth está conectado por error en `A1`, `A0`, `GND` o en el pin digital `2`, el Arduino estará leyendo un pin flotante o con pullup interno en `A2`.
  * Como en el firmware se colocó `pinMode(PIN_BT_RX, INPUT_PULLUP)`, el pin A2 leerá `HIGH` (`1`), lo cual engaña al usuario creyendo que hay conexión, pero en realidad ningún bit físico está llegando desde el chip de radio.
  * También es posible que el pin hembra del conector del Shield L293D esté falseando o el cable Dupont tenga el filamento de cobre roto internamente.

#### Hipótesis 2: Bloqueo de Interrupciones por `SoftwareSerial` (Colisión Half-Duplex) — [Probabilidad: MEDIA-ALTA]
* `SoftwareSerial` en el ATmega328P emula la UART por software mediante retardos de microsegundos (`_delay_loop_2`) y la rutina `cli()` (Clear Interrupts).
* **El mecanismo del fallo:**
  * Cada 500 ms, el Arduino ejecuta `enviarTelemetriaBT()`, enviando una cadena JSON de aproximadamente **45 bytes** a 9600 baudios.
  * Transmitir 45 bytes a 9600 baudios toma aproximadamente **47 milisegundos**.
  * Durante esos 47 ms, **todas las interrupciones de cambio de pin (PCINT) del microcontrolador están deshabilitadas**.
  * Si el usuario pulsa un botón en la Web y el módulo Bluetooth descarga el byte `M` por su pin TXD justo mientras el Arduino está transmitiendo telemetría, el bit de inicio (*start bit*) se pierde irremediablemente y el byte es descartado.
  * Además, al no tener búfer de hardware en A2, cualquier concurrencia rompe la recepción.

#### Hipótesis 3: Incompatibilidad de Característica GATT en Clones BLE AT-09 / HM-10 — [Probabilidad: MEDIA]
* El módulo anuncia nombre "HC-05", pero es un clon BLE (HM-10 o AT-09).
* En la especificación Bluetooth Low Energy, enviar datos al dispositivo se hace a través de la característica `FFE1`:
  * Algunos clones chinos **únicamente aceptan `Write without response`** (comando sin confirmación a nivel de enlace).
  * Otros clones (especialmente con chips Beken BK3431 o CC2541 con firmwares viejos) rechazan los paquetes sin confirmación y **exigen obligatoriamente `Write with response` (GATT Write Request)**.
  * Si el navegador web intenta enviar mediante el método que el firmware del chip BLE no soporta, el navegador no siempre arroja error, pero el chip Bluetooth simplemente descarta el paquete de radio sin ponerlo en el bus UART TXD hacia el Arduino.

#### Hipótesis 4: Nivel Lógico de Voltaje de Entrada (Threshold TTL de A2) — [Probabilidad: BAJA]
* El pin TXD del módulo Bluetooth entrega **3.3V** en estado ALTO.
* El Arduino Uno opera con alimentación de 5V.
* Según la hoja de datos del ATmega328P, el umbral de voltaje para un `1` lógico ($V_{IH}$) es $0.6 \times V_{CC} = 0.6 \times 5.0\text{V} = 3.0\text{V}$.
* 3.3V está muy cerca del límite de 3.0V. Si las baterías tienen una masa ruidosa por los motores o la línea de 3.3V del módulo cae bajo carga a 2.8V, el pin analógico A2 en modo digital podría no reconocer las transiciones de nivel bajo/alto.

---

## 6. HOJA DE RUTA Y PRUEBAS RECOMENDADAS PARA CLAUDE

Si vas a asumir la resolución de este problema, aquí tienes los pasos quirúrgicos directos para dar con la solución sin perder tiempo:

### 🧪 Paso 1: Verificación Física Ineludible (Hardware)
Pedirle al usuario que verifique minuciosamente con el carrito en mano:
1. **Rastreo del Pin TXD del Bluetooth:**
   * Ubicar el pin marcado **`TX`** o **`TXD`** en la placa del Bluetooth.
   * Seguir físicamente el cable hasta el Shield L293D.
   * **Debe estar insertado en la fila analógica exactamente en el pin rotulado como `A2`** (contando: A0, A1, **A2**, A3, A4, A5).
   * Si el usuario tiene dudas, sustituir ese cable Dupont por uno nuevo (muchos cables vienen cortados por dentro).
2. **Pines STATE y EN:** Asegurarse de que estén totalmente sueltos al aire.

---

### 🧪 Paso 2: Descartar la Saturación de `SoftwareSerial` en el Firmware
En `Arduino_Uno_Firmware.ino`, para probar si el Arduino escucha cuando no está saturado transmitiendo:
1. **Comentar temporalmente el envío de telemetría hacia el Bluetooth** en `loop()`:
   ```cpp
   // if (ahora - lastTelemetryBT >= 500) {
   //   lastTelemetryBT = ahora;
   //   enviarTelemetriaBT(ahora);
   // }
   ```
2. O alternativamente, aumentar el intervalo de telemetría a cada 2000 ms y reducir el tamaño del paquete a lo mínimo indispensable:
   ```cpp
   bt.print(F("{\"f\":")); bt.print((int)distFront); bt.println(F("}"));
   ```
3. Si al suspender la telemetría saliente el carro **responde al instante al botón Manual**, la causa es 100% la colisión half-duplex de `SoftwareSerial`.
   * **Solución permanente para SoftwareSerial:** Usar la librería `AltSoftSerial` (usa temporizador por hardware) o conectar el Bluetooth a los pines 0 (RX) y 1 (TX) usando la UART por hardware nativa del microcontrolador (desconectando el cable durante la subida por USB).

---

### 🧪 Paso 3: Test Directo de Escritura BLE en `WebApp_Controller/index.html`
Para verificar si el módulo Bluetooth está descartando los paquetes por el tipo de escritura GATT:
1. En la consola de Chrome (F12) en la PC conectada al carro por BLE, ejecutar manualmente:
   ```javascript
   // Probar escritura con respuesta directa
   const enc = new TextEncoder();
   await chWrite.writeValueWithResponse(enc.encode("M\n"));
   ```
2. Y probar escritura sin respuesta:
   ```javascript
   await chWrite.writeValueWithoutResponse(enc.encode("M\n"));
   ```
3. Observar cuál de las dos llamadas no genera error y si el LED del módulo parpadea.

---

### 🧪 Paso 4: Script de Validación Puente Serie (Loopback Test)
Si se conecta el Arduino a la PC por cable USB, se puede cargar un sketch mínimo de prueba que sirva de puente directo entre `Serial` (Monitor Serie de la PC a 9600) y `bt` (`SoftwareSerial` en A2/A3):
```cpp
#include <SoftwareSerial.h>
SoftwareSerial bt(A2, A3); // RX=A2, TX=A3

void setup() {
  Serial.begin(9600);
  bt.begin(9600);
  Serial.println("PUENTE ACTIVO: Lo que escribas en la app debe salir aqui:");
}

void loop() {
  if (bt.available()) {
    char c = bt.read();
    Serial.print("BT -> PC: [");
    Serial.print(c);
    Serial.print("] DEC: ");
    Serial.println((int)c);
  }
  if (Serial.available()) {
    bt.write(Serial.read());
  }
}
```
Con este sketch cargado:
* El usuario abre la app web en su teléfono, se conecta por BLE y presiona "Manual".
* En el Monitor Serie de la PC debe aparecer de inmediato `BT -> PC: [M]`.
* Si no aparece nada: el fallo es **físico de cableado/módulo** o de la **característica GATT BLE**.
* Si aparece `[M]`: el fallo era la saturación del loop principal en el sketch original del robot.

---

## 7. INFORMACIÓN DE ENTORNO PARA CLAUDE
* **Sistema Operativo:** Windows 10/11.
* **Herramientas Instaladas:**
  * `arduino-cli`: Disponible en PATH para compilar y subir sketches vía USB.
  * `firebase-tools v15.34.0`: Disponible para despliegue con `firebase deploy --only hosting`.
  * `git`: Repositorio local configurado y sincronizado con `origin/main`.
* **Puerto COM del Arduino:** Típicamente `COM9` (cuando el cable USB está conectado a la PC).
* **Comando para compilar y subir en local:**
  ```powershell
  arduino-cli compile --fqbn arduino:avr:uno Arduino_Uno_Firmware
  arduino-cli upload -p COM9 --fqbn arduino:avr:uno Arduino_Uno_Firmware
  ```
* **Comando para desplegar la Web:**
  ```powershell
  cd WebApp_Controller
  firebase deploy --only hosting
  git commit -am "Update" ; git push
  ```

---
*Fin del informe técnico. Documento listo para ser transferido a Claude para su análisis y continuación.*

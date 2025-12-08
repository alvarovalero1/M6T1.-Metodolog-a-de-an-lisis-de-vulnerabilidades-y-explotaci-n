# M6T1.-Metodología de Análisis de Vulnerabilidades y Explotación

**Autor:** [Álvaro Valero]  

## 1. Introducción
Este proyecto documenta la metodología técnica aplicada para la identificación, análisis y explotación de vulnerabilidades en software binario. El objetivo es demostrar un flujo de trabajo profesional, desde la preparación del entorno hasta la confirmación de la ejecución arbitraria de código (RCE), utilizando como objetivo de estudio el software **VulnServer**.

## 2. Preparación del Entorno (The Arsenal)
Para garantizar un análisis eficiente y reproducible, se ha automatizado el despliegue del laboratorio utilizando scripts de PowerShell, asegurando la disponibilidad de las siguientes herramientas clave:

*   **VulnServer:** Binario objetivo diseñado intencionalmente con vulnerabilidades de corrupción de memoria.
*   **Immunity Debugger:** Depurador principal utilizado para el análisis dinámico y la inspección de registros (CPU Registers) en tiempo real.
*   **Python 3:** Motor de ejecución para los scripts de *fuzzing* y *exploit development*.
*   **Netcat:** Herramienta de red para la interacción inicial y pruebas de conectividad.
*   <img width="872" height="284" alt="image" src="https://github.com/user-attachments/assets/3bddd2c6-5ff0-4109-b139-507b315b9ac0" />


## 3. Metodología de Análisis

### Fase 1: Identificación (Fuzzing)
El primer paso consistió en someter al binario a un proceso de *Fuzzing* (pruebas de estrés). Utilizando un script personalizado en Python, se enviaron cadenas de caracteres de longitud incremental al comando `TRUN` del servidor.

**Resultado:** El servicio dejó de responder tras recibir un paquete cercano a los 2000 bytes, confirmando una posible vulnerabilidad de desbordamiento de búfer (*Buffer Overflow*).

<img width="349" height="324" alt="image" src="https://github.com/user-attachments/assets/86d4a08f-958d-4ac7-98ed-9b10cef3f791" />

'''

### Fase 2: Análisis Dinámico y Crash
Con el depurador **Immunity Debugger** adjunto al proceso (`Attach`), replicamos el envío del paquete malicioso. El análisis del estado de la CPU en el momento del fallo reveló que el registro **EIP** (Instruction Pointer) fue sobrescrito, provocando una violación de acceso.

### Fase 3: Control del Flujo de Ejecución (EIP Overwrite)
Para confirmar la explotabilidad, refinamos el *payload* enviando una cadena compuesta íntegramente por caracteres 'A' (`\x41`).

**Evidencia Técnica:**
Como se observa en la siguiente captura, el registro EIP muestra el valor `41414141`. Esto demuestra que tenemos control total sobre la instrucción que el procesador intentará ejecutar a continuación.

A continuación se envió un payload de prueba `2003 'A' + 'BBBB'`, observándose `EIP = 0x42424242` en el debugger, lo que demuestra que el flujo de ejecución pasa a estar completamente bajo control del atacante.

<img width="185" height="46" alt="image" src="https://github.com/user-attachments/assets/1496405c-46d2-4088-9af1-6fc516815d3b" />
<img width="436" height="25" alt="image" src="https://github.com/user-attachments/assets/d3f2acaf-de3c-410e-962b-d6abe48b6b1d" />

### Fase 4: Ejemplo práctico completo en VulnServer (TRUN)

1. **Fuzzing y detección del fallo**  
   Se desarrolló un fuzzer en Python que enviaba cadenas crecientes al comando `TRUN /.:/`. El servicio dejó de responder alrededor de los ~2000 bytes, confirmando un posible stack-based buffer overflow.

2. **Cálculo del offset hasta EIP**  
   Se generó un patrón cíclico y se envió tras 2000 bytes de relleno.  
   El crash mostró en Immunity un EIP con el valor `0x41316141`, que se correlacionó con una posición concreta del patrón, obteniendo un offset efectivo de **2003 bytes** para este entorno y prefijo.
   <img width="344" height="23" alt="image" src="https://github.com/user-attachments/assets/53dca074-cbb6-4305-afc7-70cedb6dced5" />
   Posteriormente se verificó enviando `2003 'A' + 'BBBB'` y observando `EIP = 0x42424242` en el debugger, demostrando control total del puntero de instrucción.
   <img width="345" height="32" alt="image" src="https://github.com/user-attachments/assets/825f9d51-0406-4362-a3da-08d8bf45f22e" />

4. **Localización de un gadget de salto (`JMP ESP`)**  
   En la librería `essfunc.dll` (sin ASLR) se identificó una instrucción `JMP ESP` estable en la dirección `0x625011AF`, que permite redirigir la ejecución desde EIP hacia el buffer controlado en la pila.

5. **Prueba de ejecución de código (PoC)**  
   Como prueba de concepto de RCE se construyó un payload formado por:  
   `padding (2003 bytes) + dirección JMP ESP (0x625011AF) + NOPs + bytes \xCC (INT3)`.  
   Al enviarlo, Immunity se detuvo mostrando una interrupción `INT3` en el código inyectado, lo que confirma que el proceso estaba ejecutando instrucciones controladas por el atacante.
   <img width="292" height="48" alt="Evidencia_4_RCE_INT3" src="https://github.com/user-attachments/assets/3d6f7fec-912b-4884-a4ff-a108a92705d0" />

Este ejemplo demuestra de forma práctica las fases de la metodología propuesta: descubrimiento (fuzzing), análisis del crash, cálculo de offset, manipulación de EIP y ejecución de código controlado.

## 4. Aproximación a Vulnerabilidades Desconocidas (0-Days)
Si nos enfrentáramos a un software desconocido (0‑day) sin documentación pública, la metodología evolucionaría de la siguiente manera:

1. **Ingeniería inversa (static analysis):** Utilizar herramientas como **Ghidra** o **IDA Pro** para desensamblar el binario, buscando funciones inseguras (`strcpy`, `strcat`, etc.) y puntos de entrada potenciales.
2. **Binary diffing:** Si existiera una versión parcheada, usar **BinDiff** para comparar binarios y localizar cambios relacionados con la corrección de vulnerabilidades. 
3. **Fuzzing instrumentado:** Emplear herramientas como **WinAFL** o **Boofuzz** para aumentar la cobertura de código y priorizar rutas que aún no han sido exploradas, mejorando la probabilidad de descubrir fallos explotables.

## 5. Conclusiones
A través de este ejercicio práctico se ha validado que la metodología clásica de *stack-based buffer overflow* continúa siendo la base fundamental para entender la explotación de binarios. La combinación de fuzzing, análisis dinámico, cálculo de offset, búsqueda de gadgets (`JMP ESP`) y pruebas de ejecución de código controlado proporciona un marco sólido y reproducible para el análisis de vulnerabilidades.

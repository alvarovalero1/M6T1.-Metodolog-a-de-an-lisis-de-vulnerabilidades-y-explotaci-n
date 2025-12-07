# M6T1.-Metodolog-a-de-an-lisis-de-vulnerabilidades-y-explotaci-n
# Metodología de Análisis de Vulnerabilidades y Explotación

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

<img width="185" height="46" alt="image" src="https://github.com/user-attachments/assets/1496405c-46d2-4088-9af1-6fc516815d3b" />
<img width="436" height="25" alt="image" src="https://github.com/user-attachments/assets/d3f2acaf-de3c-410e-962b-d6abe48b6b1d" />

## 4. Aproximación a Vulnerabilidades Desconocidas (0-Days)
Si nos enfrentáramos a un software desconocido (0-day) sin documentación pública, mi metodología evolucionaría de la siguiente manera:

1.  **Ingeniería Inversa (Static Analysis):** Antes de *fuzzear*, utilizaría herramientas como **Ghidra** o **IDA Pro** para desensamblar el binario. Buscaría funciones inseguras conocidas (como `strcpy` o `strcat`) en la tabla de importaciones para identificar puntos débiles teóricos.
2.  **Binary Diffing:** Si existiera una versión parcheada del software, usaría **BinDiff** para comparar los binarios. Las diferencias en el código suelen señalar directamente dónde estaba la vulnerabilidad y cómo fue corregida, ahorrando semanas de investigación.
3.  **Fuzzing Instrumentado:** En lugar de un *fuzzer* básico de red, implementaría **WinAFL** o **Boofuzz** para tener métricas de cobertura de código y asegurar que estoy probando todas las rutas posibles del programa.

## 5. Conclusiones
A través de este ejercicio práctico, se ha validado que la metodología clásica de *Stack-Based Buffer Overflow* sigue siendo la base fundamental para entender la explotación de binarios. 

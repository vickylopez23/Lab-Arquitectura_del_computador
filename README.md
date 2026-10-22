# Arquitectura de Computadoras — Laboratorios (FaMAF, UNC) 2023

Laboratorios de la materia **Arquitectura de Computadoras** de la Licenciatura en Ciencias de la Computación (FaMAF, UNC), 2023.

**Grupo:** Victoria López, Tomás Ponce y Tomás Ordoñez.

| Lab | Tema | Herramientas |
|-----|------|--------------|
| 1 | Procesador LEGv8 segmentado: nueva instrucción, riesgos de datos y de control | SystemVerilog, Quartus, ModelSim |
| 2 | Evaluación de microarquitecturas: cachés, *loop unrolling*, predictores de saltos, in-order vs. out-of-order | gem5, ensamblador ARMv8 |

---

## Laboratorio 1 — Procesador LEGv8 con pipeline

Se parte del procesador ARM (LEGv8) segmentado en 5 etapas del libro de Patterson & Hennessy (`PipelinedProcessorPatterson-Modules/`) y se lo extiende en tres ejercicios incrementales.

### Ejercicio 1 — Pipeline funcionando e instrucción `MOVZ` (`lab1_ej_1/`)

- Se dejó funcionando el pipeline siguiendo la guía del laboratorio.
- Como los programas de prueba crecieron (por los `NOP` necesarios sin manejo de riesgos), se amplió la memoria de instrucciones (`imem.sv`) a **10 bits de dirección (1024 palabras)** y se conectaron esos bits en `processor_arm.sv`.
- Implementación de **`MOVZ Rd, #imm16, LSL #shift`** (shift de 0, 16, 32 o 48), sin agregar módulos ni señales nuevas:
  - `maindec.sv`: decodificación de `MOVZ` y sus señales de control.
  - `aludec.sv`: cuatro nuevos valores de `alucontrol`, uno por cada posición posible del `LSL`.
  - `signext.sv`: para `MOVZ` el inmediato no se extiende en signo; se ubica en los 16 bits menos significativos y el resto queda en 0.
  - `alu.sv`: coloca el inmediato en el bloque de 16 bits que indica el `LSL` y pone el resto en 0.
- Se probó con un programa específico de `MOVZ` (que guarda en memoria `0x11`, `0xF_0000`, `0xFFFF_0000_0000` y `0xF_0000_0000_0000`) y con el programa principal de la cátedra, para verificar que el resto de las instrucciones siguiera funcionando.

### Ejercicio 2 — Riesgos de datos: forwarding y stall (`lab1_ej_2/`)

Se agregaron tres módulos nuevos con sus señales de control y datos:

- **`forwarding_unit.sv`**: adelanta resultados desde las etapas MEM y WB hacia los operandos de la ALU en EX. Ejemplo verificado en la simulación: un `SUB` seguido de un `STUR` que usa su resultado; se activa `ForwardB` y el operando llega desde EX/MEM.
- **`hazard_detection_unit.sv`**: detecta el caso *load-use*, que no se puede resolver con forwarding (por ejemplo, un `LDUR` seguido de un `ADD` que usa el dato cargado), y genera la señal **Stall**, que frena el fetch y la decodificación un ciclo hasta que el dato esté disponible para adelantarlo.
- **`instruction_register_decoder.sv`**: extrae los campos `Rd`, `Rn` y `Rm` de las instrucciones en cada etapa para que las dos unidades anteriores puedan evaluar sus condiciones.

### Ejercicio 3 — Riesgos de control: flush (`lab1_ej_3/`)

- A partir de `PCSrc`, que se resuelve en la etapa MEM, se genera la señal **Flush**.
- Cuando se toma un salto, se ponen en 0 las señales de control de los registros ID/EX y EX/MEM, y el registro IF/ID recibe una instrucción con todos los bits en 0.
- Así, ninguna de las instrucciones cargadas por error detrás del salto llega a modificar el estado del procesador. Se verificó con un `CBZ` seguido de dos `STUR` y un `ADD`.

### Cómo correr el Lab 1

1. Abrir el proyecto `Lab1.qpf` de la carpeta del ejercicio en Quartus y simular con ModelSim usando `processor_tb.sv`.
2. El programa que se carga en la memoria de instrucciones está en `progs/test_code.txt` (copia de `simulation/modelsim/progs/test_code.txt`).
3. Para generar `test_code.txt` a partir de un programa en ensamblador se usa **La Olla** (`opcode-loader/`), la herramienta de la cátedra: se escribe el `.s` y se compila con `olla.sh`. Ahí también están los programas de prueba de cada ejercicio (`lab1_ejN_test_code*`).

El análisis completo, con los diagramas del procesador y las capturas de las waves, está en `Informe_Lab1.pdf`.

---

## Laboratorio 2 — Microarquitecturas con gem5

Se simularon distintas configuraciones de procesador con **gem5** para medir ciclos, ciclos ociosos, CPI, hits de caché y aciertos del predictor de saltos.

### Ejercicio 1 — DAXPY (suma y producto de vectores)

- **Caché de datos:** se comparó una caché de 8, 16, 32, 64 y 128 KB con 1, 2, 4 y 8 vías.
  - Con 1 vía solo rinden bien las de 64 y 128 KB; las más chicas tienen muy pocos hits.
  - Con 2 vías mejoran mucho las cachés de 8, 16 y 32 KB: menos ciclos, menos ciclos ociosos y más hits. El mejor CPI se obtuvo con 2 vías y entre 8 y 64 KB.
  - Con 4 y 8 vías se maximizan los hits pero aumentan los ciclos totales; el grupo lo atribuye a imprecisión del simulador.
- **Loop unrolling** (`daxpy_optimizado.s`): se desenrolló el loop en 4 iteraciones, lo que redujo la cantidad de ciclos en el procesador in-order.
- **In-order vs. out-of-order:** con la versión optimizada, el procesador fuera de orden redujo considerablemente los ciclos y los ciclos ociosos.

### Ejercicio 2 — Simulación física (`simFisica.s`)

- Con una caché de 32 KB de 1, 2, 4 y 8 vías y predictor de saltos local, la configuración más rápida fue la de **2 vías**.
- **Predictores de saltos:**
  - Predictor local: tasa de fallos del **0,42 %** (1413 fallos).
  - Predictor por torneo: tasa de fallos del **0,22 %** (735 fallos), casi el doble de preciso. Esto se explica porque el código tiene saltos condicionales dentro de los ciclos más internos que un predictor global predice mejor.
- **In-order vs. out-of-order** (ambos con predictor por torneo): el procesador fuera de orden usó casi la mitad de los ciclos, con un CPI casi a la mitad y ciclos ociosos reducidos en órdenes de magnitud.

### Estructura del Lab 2

- `lab2/`: entorno de simulación provisto por la cátedra.
  - `run_simulation.sh`: corre la simulación eligiendo benchmark y configuración de procesador.
  - `scripts/`: configuración de gem5 (`in_order.py`, `out_of_order.py`, `cpu_config.toml`, `se.py`, `devices.py`) y `stat-collect.py` para recolectar métricas.
  - `benchmarks/`: programas en ensamblador (`daxpy.s`, `daxpy_optimizado.s`, `simFisica.s`, `bubbleSort.s`) y sus imágenes.
  - `codebuilder/`: `Makefile` para compilar los programas.
  - `Dockerfile`: imagen con gem5. Ver `lab2/README.md` para crear configuraciones nuevas.
- `lab2_entrega/`: entrega final, con los programas `daxpy.s`, `daxpy_optimizado.s` y `simFisica.s` y el informe `Informe Laboratorio 2.pdf`, que tiene todos los gráficos y el análisis.

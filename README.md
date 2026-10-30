# Proyecto de Sistemas Operativos

## Proyecto 3: Tablas de páginas en acción (xv6)

### Integrantes

- Jeronimo Escobar
- Sebastian Cañon
- Yohn Gaviria

### Curso

Sistemas Operativos

### Semestre

2026-2

---

# Descripción

Se extendió xv6-riscv (commit `06aad25`) con tres mecanismos que usan la tabla de páginas de cada proceso:

- **Parte A: página compartida y** `ugetpid()`**.** Cada proceso recibe una página física propia que el kernel mapea en la dirección virtual `USYSCALL` (la página justo debajo de `TRAPFRAME`) con permisos `PTE_R | PTE_U`. El kernel guarda ahí el pid y `ugetpid()` lo lee como una variable, sin llamada al sistema. La página se libera junto con el proceso, y el heap (`growproc()` y `sys_sbrk()`) ahora termina en `USYSCALL` en lugar de `TRAPFRAME`.
- **Parte B:** `vmprint()`**.** Llamada al sistema que imprime, con un recorrido recursivo al estilo de `freewalk()`, las PTEs válidas de los tres niveles Sv39 del proceso: dirección virtual donde empieza el rango, valor de la PTE y dirección física.
- **Parte C:** `pgaccess()`**.** Llamada al sistema que, para hasta 64 páginas desde `base`, construye una máscara con las páginas cuyo bit `PTE_A` está encendido (leídas o escritas desde la última consulta), la copia al usuario con `copyout()` y limpia el bit.

---



# Archivos modificados



## kernel/


| Archivo                  | Cambio                                                                                                                                                    | Por qué                                                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `memlayout.h`            | Define `USYSCALL` (`TRAPFRAME - PGSIZE`) y `struct usyscall { int pid; }` dentro de `#ifndef __ASSEMBLER__`. Actualiza el comentario del mapa de memoria. | Fija la dirección de la página compartida y su contenido. La estructura va protegida porque el archivo también lo incluye código en ensamblador (`trampoline.S`). |
| `proc.h`                 | Campo `struct usyscall *usyscall` en `struct proc`.                                                                                                       | Guarda el puntero a la página física para mapearla y liberarla.                                                                                                   |
| `proc.c`                 | `allocproc()` reserva la página con `kalloc()` y guarda el pid. `proc_pagetable()` la mapea en `USYSCALL` con `PTE_R                                      | PTE_U`y deshace los mapeos anteriores si falla.`freeproc()`la libera.`proc_freepagetable()`la desmapea sin liberarla.`growproc()`compara contra`USYSCALL`.        |
| `sysproc.c`              | `sys_sbrk()` compara contra `USYSCALL`. Nuevas `sys_vmprint()` y `sys_pgaccess()`.                                                                        | Partes A, B y C.                                                                                                                                                  |
| `vm.c`                   | `vmprint()` y su auxiliar recursiva `vmprint_level()`.                                                                                                    | Parte B.                                                                                                                                                          |
| `riscv.h`                | `#define PTE_A (1L << 6)`.                                                                                                                                | Parte C: bit de acceso de la especificación RISC-V.                                                                                                               |
| `syscall.h`, `syscall.c` | `SYS_vmprint` (23) y `SYS_pgaccess` (24), con sus `extern` y entradas en la tabla.                                                                        | Registro de las nuevas llamadas.                                                                                                                                  |
| `defs.h`                 | Prototipo de `vmprint(pagetable_t)`.                                                                                                                      | Para que `sysproc.c` pueda llamarla.                                                                                                                              |




## user/


| Archivo   | Cambio                                                                             | Por qué                                                   |
| --------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `ulib.c`  | Incluye `kernel/memlayout.h` e implementa `ugetpid()`.                             | Parte A: lee el pid de `USYSCALL` sin llamada al sistema. |
| `user.h`  | Declara `ugetpid()`, `vmprint()` y `pgaccess()` con las firmas exactas de la guía. | Los usan `pgtbltest` y los programas de usuario.          |
| `usys.pl` | `entry("vmprint")` y `entry("pgaccess")`.                                          | Genera los stubs de las llamadas al sistema.              |


---



# Integración con xv6

1. Clonar xv6-riscv y hacer `git switch -c proyecto3 06aad25`.
2. Copiar los archivos entregados en sus rutas.
3. Agregar `$U/_pgtbltest` a UPROGS en el Makefile (después de `$U/_sync\`), copiar `pgtbltest.c` y la versión del curso de `usertests.c` a `user/`.
4. Compilar con `make qemu`.
5. Ejecutar `pgtbltest` y `usertests -q`.

Para regenerar la evidencia: `python3 ejecutar_pruebas.py` desde la raíz de xv6.

---



# Respuestas a las preguntas



## 1. Interpretación de vmprint()

Salida de `vmprint()` del programa `pgtbltest` (la prueba `vmprint`). Los permisos salen de los 10 bits bajos de cada PTE: V=0x1, R=0x2, W=0x4, X=0x8, U=0x10, A=0x40 (accedida), D=0x80 (modificada). A y D los enciende el hardware.


| Dirección virtual            | PTE                          | Flags         | Qué contiene                                                                                                                                       |
| ---------------------------- | ---------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0x0000`, `0x1000`, `0x2000` | `...05b`, `...85b`, `...c1b` | V R X U       | **Texto** (código del programa y datos de solo lectura). Se ejecuta y se lee, pero no se escribe: ni el programa puede modificar su propio código. |
| `0x3000`                     | `...4d7`                     | V R W U       | **Datos y bss.** Lectura y escritura para el proceso.                                                                                              |
| `0x4000`                     | `...407`                     | V R W (sin U) | **Página de guarda.** Está mapeada, pero sin `PTE_U`, así que cualquier acceso desde modo usuario falla.                                           |
| `0x5000`                     | `...8d7`                     | V R W U       | **Pila** de usuario (una página).                                                                                                                  |
| `0x9000`                     | `...cd7`                     | V R W U       | Página de heap de la región perezosa (`sbrklazy`) que el proceso tocó. Las otras 7 no aparecen porque no tienen memoria.                           |
| `0x3fffffd000`               | `...013`                     | V R U         | **USYSCALL**: el proceso solo puede leerla.                                                                                                        |
| `0x3fffffe000`               | `...8c7`                     | V R W (sin U) | **Trapframe**: lo usa el trampolín al entrar y salir del kernel; el usuario no la ve.                                                              |
| `0x3ffffff000`               | `...84b`                     | V R X (sin U) | **Trampolín**: código de entrada y salida de traps; ejecutable pero no accesible desde modo usuario.                                               |


En los niveles intermedios (las líneas con uno y dos `..`) las PTEs solo tienen `V`: apuntan a la tabla del nivel siguiente y no son hojas. Los saltos de direcciones (de `0x9000` a `0x3fc0000000`) muestran que el espacio de direcciones es disperso: lo que no está mapeado no ocupa tablas.

## 2. Página de guarda

La página de guarda no tiene `PTE_U` para que el usuario no pueda acceder a ella: toda lectura, escritura o ejecución desde modo usuario produce un page fault. Está entre los datos y la pila. La pila crece hacia abajo, así que si un programa la desborda (recursión infinita, arreglos locales enormes), la siguiente escritura cae en la página de guarda, el hardware genera la excepción y `usertrap()` mata al proceso. Sin la guarda, la pila invadiría en silencio los datos del programa y corrompería su memoria. La prueba `stacktest` de `usertests` verifica esto.

## 3. Otras llamadas que se pueden acelerar

Una llamada se puede acelerar con una página compartida de solo lectura si solo **devuelve información** que el kernel puede dejar ya calculada, que no cambia el estado del kernel y que es seguro mostrar al proceso.

- `getpid()`: el pid no cambia en toda la vida del proceso. Es el caso implementado.
- `uptime()`: también se podría, con una página única (compartida por todos los procesos) en la que el manejador de la interrupción de reloj actualice el contador de ticks. Es posible pero menos directo: hay que mantener la página al día en cada tick.

No se pueden acelerar las que **modifican estado del kernel** o necesitan privilegio: `fork`, `exit`, `wait`, `kill`, `exec`, `sbrk` (cambian procesos y tablas de páginas); `open`, `close`, `read`, `write`, `pipe`, `dup`, `link`, `unlink`, `mkdir`, `mknod`, `chdir`, `sync` (usan tablas de archivos, inodos, discos y dispositivos, con bloqueos); `pause` (necesita dormir al proceso). `fstat` solo lee, pero depende del inodo y requiere sus bloqueos, así que tampoco cabe en una página estática.

## 4. Riesgo de USYSCALL con PTE_W

Con `PTE_W`, el proceso podría escribir un pid falso y `ugetpid()` mentiría, lo que engañaría a bibliotecas o programas que usan el pid (para nombres de archivos temporales, para `kill`, etc.). El riesgo real es **que el kernel confíe en esa página**: si algún día lee de ahí en lugar de usar `p->pid`, un proceso malicioso controlaría datos del kernel (escalada de privilegios o confusión de identidad). Aun si hoy el kernel no la lee, una página escribible invita a ese error al agregar campos nuevos. Por eso se mapea `PTE_R | PTE_U` y la prueba `usyscall_solo_lectura` verifica que escribir mata al proceso.

## 5. pgaccess() y páginas perezosas

Una página de `sbrklazy()` que no se ha tocado todavía **no tiene memoria física ni PTE válida**. Como el proceso nunca la accedió, el bit `PTE_A` ni siquiera existe, y 0 es la respuesta correcta a "¿se usó desde la última consulta?". `pgaccess()` no debe asignarle memoria porque es una operación de **observación**: si consultar una página la creara, la llamada cambiaría lo que mide y destruiría el ahorro de la asignación perezosa (con hasta 64 páginas por llamada, un proceso podría consumir memoria sin usarla). Además, una página creada por el kernel no habría sido accedida por el proceso. Por eso `pgaccess()` usa `walk(..., alloc = 0)` y salta las PTEs no válidas, y `vmprint()` sigue sin mostrar esas páginas.

---



# Resultados de las pruebas

Ejecutadas en QEMU (`qemu-system-riscv64` 8.2.2) con `python3 ejecutar_pruebas.py`. La salida completa está en `pruebas/`.

- `pgtbltest`**:** 11 de 11 pruebas OK (Partes A, B y C), y revisión del formato de `vmprint()` OK.
- `usertests -q` (versión del curso): `ALL TESTS PASSED`, incluidas las pruebas de asignación perezosa; sin `lost some free pages` ni `panic`.
- `make qemu` compila sin advertencias del compilador. El enlazador de la toolchain usada muestra `kernel/kernel has a LOAD segment with RWX permissions`, aviso de la versión de `ld` que también aparece con el xv6 original.

---



# Decisiones de diseño

- **Una página por proceso, liberada en** `freeproc()`**.** `proc_freepagetable()` solo desmapea `USYSCALL` (sin liberar) para no liberarla dos veces. Esto además hace correcto `exec()`: la tabla nueva vuelve a mapear la misma página (el pid no cambia) y la vieja se desmapea sin tocarla. `fork()` no necesita copiarla porque `uvmcopy()` solo copia hasta `p->sz` y cada hijo recibe la suya en `allocproc()`.
- `vmprint()` **recursivo con la estructura de** `freewalk()`**.** El nivel de profundidad determina cuántos `" .."` se imprimen, y la dirección virtual se calcula como `base + (índice << PXSHIFT(nivel))`. Una PTE sin `R`, `W` ni `X` es un puntero a la tabla siguiente. Usa `printk()`, el `printf` de esta versión del kernel.
- `pgaccess()` **en dos pasadas.** Primero solo lee `PTE_A`; luego copia el resultado con `copyout()`, y **solo si esa copia tuvo éxito** limpia los bits. Así, un puntero de resultado inválido devuelve -1 sin perder accesos.
- `sfence_vma()` **tras limpiar** `PTE_A`**.** Si la TLB conserva la PTE con A=1, el hardware no vuelve a encender el bit en el siguiente acceso y se perderían accesos nuevos.
- **Validación de argumentos sin desbordes.** El rango se valida como `base <= sz` y `npages * PGSIZE <= sz - base`, para que `base + npages * PGSIZE` no pueda dar la vuelta.
- **Heap hasta** `USYSCALL`**.** Se cambió la comparación en `growproc()` y `sys_sbrk()`, que cubre tanto `sbrk` normal como `sbrklazy`.

---



# Limitaciones conocidas

- `pgaccess()` depende de que el hardware encienda `PTE_A` automáticamente (QEMU lo hace). En hardware que lance excepción en lugar de actualizar A, habría que manejarlo en el kernel.
- `vmprint()` imprime línea por línea con `printk()`; si otro proceso imprime a la vez, sus líneas pueden intercalarse.
- Solo se aceleró `getpid()`; las otras llamadas se discuten en la pregunta 3 pero no se implementan, como pide el enunciado.
- Probado únicamente en QEMU 8.2.2 con `gcc-riscv64-linux-gnu`.

---



# Declaración de uso de IA

Se usó Claude para revisión de código y sugerencias de mejora.
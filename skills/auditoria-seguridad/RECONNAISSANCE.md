# Reconocimiento

### Fase 1: mapear la fuente y planificar la cobertura

El padre inicializa `run-metadata.json`, aplica la compuerta de presupuesto de `SKILL.md` y crea
las carpetas de los agentes y la tabla compartida antes de cazar. Si la compuerta no pasa, se
anota el estado incompleto en la metadata y **no se lanza ningún agente de reconocimiento**.

El reconocimiento lee el objetivo y lo que esté disponible de la máquina: **no contacta
despliegues, ni servicios de identidad, ni registries, ni APIs de terceros.**

Se lanzan varios agentes de investigación en paralelo (con la herramienta de delegación). Devuelven
hechos estructurados al padre y **no escriben archivos**.

## Los cuatro agentes de reconocimiento

**1a — Producto, stack y operación local**

```text
Leé el objetivo en <target>. Sin acceso a red. Devolvé:
1. Qué producto es, quiénes son sus usuarios y operadores, y qué acciones normales tocan la
   seguridad (publicar al aire, borrar medios, cambiar la config del Core, escribir en un equipo).
2. Lenguajes, frameworks, sistema de build, runtimes y los modos de despliegue visibles en el repo.
3. Puntos de entrada y fronteras de subsistema, con rutas relativas al repo.
4. Los comandos exactos de build y test que podrían correr sin red con lo ya instalado, dónde
   escribirían, y qué entradas del objetivo procesan. NO los corras en el reconocimiento.
5. Software o protocolo comparable, visible en la documentación y las dependencias locales. Si no
   hay comparación fundada en la fuente, decilo.
6. Qué herramientas o hechos del runtime faltan y limitan la verificación local.
Devolvé sólo hechos de la fuente, con archivo:línea relativo al repo.
```

**1b — Principales, autoridad y controles**

```text
Leé todo el código que establece identidad, autorización, aislamiento y privilegio. Mapeá:
1. Cada principal de menor confianza y las acciones que tiene por diseño.
2. La autenticación o identidad en cada superficie de entrada.
3. La autorización por recurso y el alcance por dueño.
4. La autoridad de proceso, navegador, trabajo, CI, plugin, modelo/herramienta, dispositivo o IPC.
5. Los cambios de privilegio, confirmación, revocación, recuperación y los caminos de respaldo.
6. Qué controles se ven en la fuente y cuáles dependen de un hecho de despliegue que no se observa.
Devolvé las fronteras de confianza y dónde está cada control, con archivo:línea. No infieras
alcance real en vivo.
```

**1c — Superficies de entrada, copias y sumideros**

```text
Inventariá cada lugar visible en la fuente por donde entra algo externo o de menor confianza:
HTTP/navegador, mensajes/protocolo, archivos/archivo comprimido/documento, CLI/entorno/config,
plugins y dependencias, eventos de nube, contexto de modelo y argumentos de herramienta, y los
dispositivos locales en juego (por ejemplo el receptor de TV y su telnet).
Por cada superficie: seguí las transformaciones importantes, las copias guardadas o derivadas, y
los sumideros que importan para la seguridad. Anotá los límites visibles y los caminos paralelos
que llevan al mismo efecto.
Devolvé rutas relativas al repo y números de línea. Sé completo, pero no ejecutes ni mandes datos.
```

**1d — Visibilidad de ejecución y despliegue**

```text
Leé los tests, las definiciones de build, los manifiestos y la configuración de despliegue que
mantenemos. Devolvé:
1. Los tests chicos o fixtures que ya existen y podrían validar una frontera con datos de prueba,
   sin salir a la red y sin instalar nada.
2. Los procesos que se pueden observar en modo lectura en esta VM (contenedores propios, unidades,
   puertos, logs) y qué muestra cada uno.
3. Los comandos que descargarían dependencias, publicarían algo, tocarían APIs pagas o afectarían
   estado compartido: marcalos como PROHIBIDOS para esta corrida.
4. Los controles de despliegue que la fuente no puede establecer y que, si son decisivos, van como
   needs_validation.
5. Si la plataforma local puede ejecutar algo del objetivo SIN red, con entorno vacío y límites
   explícitos. En nuestra VM no hay sandbox del sistema operativo: asumí que NO se puede y decilo.
6. Qué parte de la verificación va a quedar como needs_validation por esa limitación.
```

Si hay modos de despliegue o subsistemas distintos que estos cuatro no cubren, se agrega un agente
enfocado. **Si no alcanza el presupuesto, no se lanza**: el área queda como unidad `deferred` con
el motivo y el hueco se declara en el informe.

## Lo que dejaron las corridas anteriores

Antes de elegir el trabajo, el padre lee todos los `coverage-ledger.json` y `findings.json` que
haya de la misma materia:

- Comparar ubicaciones, controles, condiciones e identidad de cada registro y unidad previos contra
  **el código actual**.
- Un `confirmado` previo se arrastra sólo si su código, sus condiciones y su evidencia siguen
  valiendo. Se lo liga a una unidad `planned` con `prior_status: "prior_confirmed_same_source"` y
  sólo esa causa raíz se excluye de la caza. El verificador que lo re-chequee pasa a ser el dueño
  de esa unidad y su chequeo es el primero de la unidad.
- Si el código cambió, se crea una unidad de revalidación `prior_confirmed_changed_source`. Esa
  causa raíz **no** se excluye: el veredicto anterior no vale por sí solo.
- Todo `needs_validation`, `deferred`, `blocked`, `out_of_scope` y lo que cambió es trabajo actual.
  Esos estados dan prioridad; **nunca** sirven para deduplicar ni para suprimir.
- Un `needs_validation` que sigue bloqueado se arrastra sólo si el código actual sostiene su traza,
  con `prior_status: "prior_needs_validation"`, y conserva su bloqueo.
- Un `rechazado` previo queda como afirmación vieja salvo que la evidencia actual cambie la traza:
  suprime esa afirmación, no la revisión de la unidad.
- Si la tabla previa falta o es incompatible, se anota — no se trata como "cobertura vacía".

## Resumen de arquitectura y elección de compañeros

El padre sintetiza `<salida>/architecture.md`, con un tope duro de unas 1.000 palabras. Tiene que
incluir:

1. Producto, principales, autoridad normal y recursos protegidos.
2. La línea base de software comparable, cuando sea fundada: qué compromisos acepta el comparable.
   Sirve para calibrar esfuerzo y severidad, **nunca** para descartar un hallazgo demostrado.
3. Stack, caminos de despliegue visibles y los límites de build/test sin red.
4. Superficies de entrada y los caminos fuente→sumidero o de ciclo de vida que importan.
5. Fronteras de confianza y el control más fuerte visible en cada una.
6. Rutas de arranque, relativas al repo.
7. Huecos previos, objetivos de revalidación, y las confirmaciones del mismo código que se excluyen.
8. Un resumen corto de compañeros elegidos según [ATTACK-CLASSES.md](ATTACK-CLASSES.md): archivos
   elegidos y las fronteras que los exigen.

Las decisiones por unidad (bloque común, compañeros elegidos y bloques excluidos con su motivo)
viven en la tabla, no en `architecture.md`. No se elige un compañero porque el lenguaje aparezca en
el nombre: se lo elige porque el reconocimiento **encontró la frontera** de su sección "cuándo usar
este archivo".

## La tabla de cobertura determinista

El padre escribe `<salida>/coverage-ledger.json` como un arreglo JSON de nivel superior. Se deriva
**una unidad por cada combinación material** de superficie de entrada, frontera de confianza,
subsistema y clase de ataque aplicable, con la granularidad del perfil (`rapida` usa un solo
identificador de subsistema; `profunda` agrega modos del ciclo de vida).

Cada dimensión tiene una etiqueta legible y un valor estable derivado de la fuente en
`canonical_refs`. Se usa la misma referencia canónica para el mismo objeto entre corridas, aunque
cambie la etiqueta visible. Sirven como referencia: la ruta de entrada más el alcance exportado,
una ruta o identidad definida en el código, el control que define una frontera, el paquete, y la
referencia exacta al bloque de clase de ataque.

Una referencia de bloque es `ARCHIVO.md#` más el texto exacto del encabezado tal como está en ese
archivo (hoy, en español: `Disciplina central`, `Movimientos universales`, `Reglas de validación`).
**No** se derivan referencias pasando la etiqueta a minúsculas ni a slug.

`coverage_id` se deriva sin pérdida:

1. Cada referencia tiene que ser Unicode NFC, con contenido visible, sin caracteres de control ni
   separadores, sin espacios alrededor.
2. Sus bytes UTF-8 se codifican con percent-encoding de RFC 3986: sólo quedan sin escapar
   `A-Z a-z 0-9 - . _ ~`; todo lo demás va como `%HH` en mayúsculas.
3. Se unen las referencias codificadas de `surface`, `boundary`, `subsystem` y `attack_class` con
   `::`; si hay ciclo de vida, se agrega al final.

Para el perfil `rapida`, el subsistema usa el valor fijo
`profile/quick/all-in-scope-subsystems`. En una referencia o ID **no** van número de ola, agente,
veredicto, severidad ni línea. Las unidades se ordenan lexicográficamente por `coverage_id` antes de
cada asignación. **Se falla ante cualquier ID duplicado** (si dos duplicados tienen campos
semánticos distintos, es una colisión de identidad: no se fusionan ni se pisan en silencio). El
validador también rechaza una misma tupla semántica con referencias canónicas distintas.

Cada unidad registra:

```json
{
  "coverage_id": "...",
  "canonical_refs": {
    "surface": "src/app/api/example/route.ts#POST",
    "boundary": "src/lib/sesion.ts#requiereSesion",
    "subsystem": "src/app/api",
    "attack_class": "ATTACK-CLASSES.md#Control de acceso"
  },
  "surface": "...",
  "boundary": "...",
  "subsystem": "...",
  "attack_class": "...",
  "starting_paths": ["src/app/api/example/route.ts"],
  "ordinary_attack_class_block": "ATTACK-CLASSES.md#Control de acceso",
  "selected_companion_blocks": ["WEB-PROTOCOL-AND-AUTH.md#Disciplina central"],
  "excluded_blocks": [{"block": "FILE.md#seccion", "reason": "..."}],
  "prior_status": "none",
  "attempts": [],
  "wave": 1,
  "status": "planned",
  "agent_id": null,
  "reviewed_paths": [],
  "local_checks": [],
  "result_fingerprints": [],
  "unresolved": []
}
```

Los nombres de los campos se mantienen **en inglés** porque los validadores y el esquema los exigen:
cambiar la prosa no cambia las claves.

Cuando hay ciclo de vida material, se agregan `canonical_refs.lifecycle` y el campo legible
`lifecycle`. `ordinary_attack_class_block` es nulo sólo si no aplica ningún bloque común. La lista
de compañeros elegidos incluye cada clase aplicable y sus tres secciones de compañero.

### La tabla de estados (se respeta tal cual)

| Estado | `agent_id` | `reviewed_paths` / `local_checks` | `result_fingerprints` | `unresolved` |
|---|---|---|---|---|
| `planned` | nulo | vacíos | vacíos | vacíos |
| `not_applicable`, `out_of_scope`, `deferred` | nulo | vacíos | vacíos | motivo no vacío |
| `in_progress` | dueño canónico | vacíos | vacíos | vacíos |
| `blocked` | dueño canónico | ambos con evidencia parcial propia | vacíos | bloqueo no vacío |
| `covered` | dueño canónico | ambos no vacíos | vacíos | vacíos |
| `candidate` | dueño canónico | ambos no vacíos | no vacíos | opcional |

Los IDs de agente son canónicos, en minúsculas y matchean `^[a-z0-9][a-z0-9_-]{0,63}$` (no son
nombres de dispositivo de Windows). Cada chequeo registra su propio `agent_id` y sus
`reviewed_paths`; los de la unidad son exactamente la unión. Un chequeo de código usa
`artifact: null`.

**En nuestra versión no hay artefactos promovidos desde un sandbox**: un chequeo `local` es la
salida de un comando nuestro con datos de prueba, transcripta en el informe con el comando exacto.
Nada de `agents/<id>/artifacts/`.

**La tabla es la afirmación de cobertura.** Un resumen, un conteo de agentes o una frase tipo
"revisé la autenticación" **no** son evidencia de cobertura. La fase 2 cierra unidades sólo con las
rutas y los chequeos del resultado estructurado de un cazador.

## Validación

```sh
node <skill-dir>/validate-coverage-ledger.cjs <salida>/coverage-ledger.json
```

Corre después de sembrar la tabla, después de cada actualización del padre y antes de la fase 6.
Todos los errores se arreglan **antes** de asignar trabajo o de afirmar cobertura.

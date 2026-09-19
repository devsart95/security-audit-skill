# Caza de vulnerabilidades

### Fase 2: olas de caza guiadas por la cobertura

El padre asigna las unidades `planned` de la tabla a los agentes cazadores. Se usan los cazadores
necesarios para cubrir las unidades sin mezclar fronteras que no tienen relación. Un cazador puede
tener varias unidades del mismo subsistema; **ninguna unidad queda sin asignar en silencio**: la
que el presupuesto no alcanza queda `deferred` con el motivo exacto.

Cuando un perfil o un presupuesto limita la cantidad de cazadores, las unidades se asignan por
prioridad y el motivo del orden queda en la tabla. El orden es: (1) superficies sin autenticación o
de menor confianza antes que las autenticadas; (2) fronteras que protegen lo más valioso
(credenciales, datos de otro, ejecución de código, autoridad de publicación); (3) huecos de
corridas previas, objetivos de revalidación y código que cambió; (4) las clases que históricamente
dan hallazgos en este tipo de objetivo. Los empates se rompen por `coverage_id`.

Antes de lanzar: las unidades asignadas pasan a `in_progress`, se fija un `agent_id` canónico y se
le crea su carpeta de trabajo. Los cazadores leen el código y el contexto que les da el padre,
escriben sólo en su carpeta y **devuelven un único resultado estructurado**. Nunca escriben los
archivos compartidos, ni el código del objetivo, ni la carpeta de otro.

## El prompt del cazador (lo que hay que incluir, en este orden)

1. **Dos frases de rol**: el objetivo es encontrar fallas de una invariante de seguridad con
   evidencia en el código, y devolver **exactamente un objeto JSON** con el contrato del final.
2. `architecture.md` **textual**.
3. Los IDs de cobertura asignados, el subsistema, la frontera, las rutas de arranque relativas al
   repo y el mapa de asignación de cada unidad.
4. **Los bloques de clase textuales**: el bloque común elegido de `ATTACK-CLASSES.md` y, por cada
   compañero elegido, su `Disciplina central`, las clases elegidas, `Movimientos universales` y
   `Reglas de validación`. **No se manda sólo el nombre del bloque.**
5. Los bloques excluidos, con el motivo de cada exclusión.
6. El método de caza (abajo) y las reglas de validación (abajo).
7. Las confirmaciones previas del mismo código que se excluyen (huella, título y causa raíz), y los
   IDs de cobertura de otros cazadores que no debe duplicar.
8. Las rutas de su carpeta, el ID seguro, y el contrato del resultado estructurado, incluidas las
   ramas `confirmed` y `needs_validation` de `report-schema.json` **textuales**.

Un prompt puede elegir varios compañeros cuando el mismo camino cruza varios dominios. El alcance
es la obligación de cobertura, no permiso para duplicar trabajo excluido. Si aparece una frontera
distinta, se devuelve en `uncovered` para que el padre cree la unidad y la asigne en la ola
siguiente.

### Método de caza — va en todos los prompts de cazador

```text
## Método defensivo de búsqueda

Tu objetivo es encontrar fallas de una invariante de seguridad con evidencia en el código, y el
arreglo más chico. NO busques ampliar el daño más allá del resultado de frontera.

## Reglas del terreno (nuestras, no negociables)

- NO contactes despliegues, endpoints externos, APIs de terceros, ni servicios compartidos. No
  sondees internet, ni los paneles de terceros, ni producción, ni la LAN del receptor.
- NO ejecutes código que no sea nuestro: ni el firmware, ni binarios del fabricante, ni builds o
  dependencias que hagan falta instalar. En esta VM no hay sandbox del sistema operativo.
- SÍ podés leer el código, leer la configuración, y OBSERVAR EN MODO LECTURA lo propio:
  `curl` a 127.0.0.1, `sudo docker ps|inspect|logs`, `ss -ltnp`, `ps`, `systemctl --user status`,
  `journalctl --user`, `git log`, `df`/`du`. Y podés correr **nuestros** scripts con datos de
  prueba, en `work/tmp`, sin red.
- Si para confirmar hace falta ejecutar algo de un tercero: NO lo hagas. Devolvé
  `needs_validation` con el bloqueo exacto.
- NUNCA pongas un valor de credencial en tu resultado. Ni real, ni inventado que parezca real.
- No toques el canal de TV al aire: es producción interna.

LEÉ EL CÓDIGO A FONDO. Seguí cada entrada asignada por el parseo, la identidad, la autorización, la
normalización, el estado, las copias derivadas y el sumidero final. Leé los caminos hermanos, los
viejos, los de lote, los de reintento, los de cancelación, los de migración y los de error que
producen el mismo efecto. Compará los controles hermanos por equivalencia, no sólo por presencia, y
compará lo que un componente garantiza con lo que el siguiente asume.

TRABAJÁ DESDE UNA INVARIANTE CONCRETA:
1. Nombrá el principal de menor confianza y con qué arranca.
2. Nombrá el valor, la acción, la transición de estado o el selector de recurso que se acepta.
3. Ubicá el control que debería rechazarlo, atarlo, aislarlo, limitarlo o revocarlo.
4. Trazá el camino exacto del código después de esa decisión.
5. Frená en el efecto más chico: un registro de prueba de más, un valor de retorno equivocado, un
   efecto observado en un recurso compartido propio.
6. Decí el cambio de código y el caso de regresión que sostienen la invariante.

LÍMITE DE PROFUNDIDAD: trazá sólo caminos que puedan llegar a tu frontera, o cuyas garantías esa
frontera use. Cortá una línea de investigación apenas la invariante quede resuelta en cualquier
sentido y anotala en tu resultado —cubierta, candidata o bloqueada, o una entrada `uncovered`— en
vez de seguir buscando.

PROBÁ LOS CAMINOS TRISTES Y LAS DISCORDANCIAS. Revisá ausente, vacío, cero, negativo, máximo, por
encima del límite, duplicado, codificación mezclada, vencido, revocado, reordenado, concurrente,
migrado a medias, dependencia caída y estado de vuelta atrás, sólo donde la interfaz los acepte.
Compará la canonicalización y las unidades en cada traspaso de parser o de política. En problemas de
varios pasos, cada salida es un prerrequisito: si un prerrequisito no está establecido, anotá un
bloqueo.

Cuando un candidato alto o crítico revele una causa raíz reutilizable, buscá variantes léxicas,
estructurales y lógicas en los caminos de tus unidades. Consolidá la misma causa raíz, pero
establecé las condiciones y el impacto de cada variante por separado. No investigues unidades de
otro cazador.

USÁ EL CHEQUEO LOCAL MÁS ANGOSTO QUE RESUELVA LA AFIRMACIÓN, y sólo dentro de lo permitido arriba.
Preferí un test que ya exista, un arnés mínimo sobre el código nuestro, un fixture chico, un
escenario de carrera determinista, o leer la política. No instales ni descargues nada.

Registrá la entrada exacta, el comando y el resultado mínimo. Del entorno, anotá sólo lo necesario
para reproducirlo, sin variables del ambiente ni estado de autenticación. Si el chequeo no se puede
hacer, no lo hagas: va como needs_validation con ese bloqueo exacto.

Un hecho de despliegue, navegador, proveedor, sistema operativo, proxy, paquete, secreto o
identidad que esté fuera del código **no prueba nada en ningún sentido**. Si uno de esos hechos es
decisivo, devolvé un needs_validation con la observación exacta que falta y el chequeo seguro que la
resolvería (que puede hacerse desde otra máquina, o mirando una configuración).

Nunca estreses la disponibilidad, ni invoques un servicio vivo, ni uses una credencial real, ni
publiques nada, ni sigas más allá del efecto mínimo observado.
```

### Reglas de validación — van en todos los prompts de cazador

```text
## Compuerta del candidato

1. Un candidato necesita la traza completa de código con rutas relativas al repo y evidencia de la
   causa raíz que afirma, incluido el control más fuerte visible en el código.
2. Un registro `confirmado` propuesto necesita un resultado local acotado y observado, impacto con
   sentido al cruzar una frontera, condiciones completas, y ninguna capa que lo impida a la vista.
3. No conviertas un cierre inesperado en ejecución de código, un trabajo normal en caída del
   servicio, ni una acción del mismo principal en aumento de privilegio.
4. Si un hecho necesario no se ve en el código ni se puede observar localmente, va
   `needs_validation`. Nombralo exacto; sin severidad y sin completarlo con especulación.
5. Una buena práctica que falta y no afecta a ningún principal ni recurso es exclusión o nota de
   endurecimiento, no hallazgo. Un candidato refutado por el código no es needs_validation.
6. La misma huella para la misma causa raíz en todos los estados. Tiene que matchear
   `^[A-Za-z0-9][A-Za-z0-9._:/@+-]*$` y no incluir línea, ola, agente, severidad ni veredicto.
7. Devolvé el arreglo de candidatos vacío cuando no sobreviva ninguno.
```

## Resultado estructurado del cazador

Se devuelve **exactamente un objeto JSON, sin prosa alrededor**. Las claves van en inglés porque el
padre y los validadores las esperan así:

```json
{
  "units": [
    {
      "coverage_id": "el ID asignado",
      "disposition": "covered|candidate|blocked",
      "reviewed_paths": ["ruta/relativa/al/repo"],
      "checks": [
        {
          "agent_id": "dueño canónico del chequeo",
          "reviewed_paths": ["ruta/relativa"],
          "invariant": "el control concreto que se revisó",
          "method": "source|local",
          "result": "qué estableció el código o el chequeo acotado",
          "artifact": null
        }
      ],
      "candidate_fingerprints": [],
      "unresolved": []
    }
  ],
  "candidates": [],
  "hardening": ["nota concreta que no es hallazgo"],
  "uncovered": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["ruta/relativa"],
      "reason": "por qué necesita su propia unidad de cobertura"
    }
  ]
}
```

`artifact` es siempre `null` en nuestra versión: no hay artefactos promovidos desde un sandbox. La
evidencia de un chequeo `local` se transcribe **en el campo `result` y en el informe**, con el
comando exacto.

Cada entrada de `candidates` tiene la forma del esquema salvo que usa `proposed_verdict` en lugar de
`verdict`:

- `proposed_verdict: "confirmed"` — incluye todos los campos que pide la rama `confirmed` del
  esquema salvo `verdict`: `fingerprint`, título, descripción, `root_cause`, `intended_behavior`,
  `trace` ordenada, `evidence`, `conditions`, `execution`, `remediation`, `severity` y
  `confidence`. `payloads` lleva la entrada mínima de prueba. `observed_result` es lo que
  efectivamente salió. **La severidad general no puede superar el impacto observado.**
- `proposed_verdict: "needs_validation"` — los campos de esa rama salvo `verdict`: `fingerprint`,
  título, descripción, `claimed_root_cause`, `trace`, `evidence`, `blockers` no vacío, y
  `validation_plan` con al menos un paso `local` o `deployment` aplicable. **Sin severidad, sin
  execution, sin remediation.**

Cada ID asignado aparece exactamente una vez en `units`. Una unidad `covered` necesita dueño,
`reviewed_paths` y `checks` no vacíos, ningún hecho sin resolver y ningún candidato. Una unidad
`candidate` es la única que lleva huellas ligadas. Una `blocked` es una revisión parcial con dueño,
rutas y chequeos no vacíos, y hechos sin resolver. Todas las rutas son relativas al repo. Una traza
de varios pasos arranca en `entrypoint`, termina en `sink` y etiqueta los pasos intermedios como
`propagation`.

## Consolidación del padre y actualización de la tabla

El padre valida cada resultado de unidad, lo mapea a un `coverage_id` asignado y actualiza **sólo**
esa unidad. Rechaza IDs duplicados o ausentes, IDs inseguros, chequeos de código con artefacto, y
cualquier artefacto que no corresponda. Copia las `reviewed_paths`, los `checks` a `local_checks`,
los fingerprints ligados y los hechos sin resolver. Guarda la lista `hardening` de cada cazador en
un campo de contabilidad aparte, para que la fase 6 la pueda reportar. **Una conclusión fallida o
mal formada deja la unidad en `planned` para reasignar.**

Las unidades que el presupuesto o el perfil no alcanzaron pasan a `deferred`, sin evidencia y con
motivo; la evidencia parcial **no** se esconde dentro de `deferred`. Después de actualizar, corre el
validador de la tabla: **una tabla inválida no puede dirigir otra asignación.**

Los candidatos se consolidan por huella y después por causa raíz. Una causa raíz que expone varios
caminos es **un** candidato, con la traza completa más fuerte. Controles faltantes distintos pero
independientes usan huellas distintas. Las huellas duplicadas se registran en la unidad y no se
mandan dos veces a validación.

## Olas de crítico de cobertura

Inmediatamente después de cada ola de caza, el padre gasta la invocación reservada en **un crítico
de cobertura fresco**: recibe `architecture.md`, la tabla completa con los mapas de asignación, los
candidatos con su estado, y el resumen de huecos previos. Lee el código pero **no escribe ni
ejecuta** nada.

Tiene que devolver exactamente este JSON:

```json
{
  "missing_units": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["ruta/relativa"],
      "selected_companion_blocks": ["FILE.md#seccion"],
      "excluded_blocks": [{"block": "FILE.md#seccion", "reason": "..."}],
      "reason": "hueco de cobertura fundado en el código"
    }
  ],
  "reassign_ids": ["id-existente-que-no-cerro"],
  "resolved_prior_leads": ["fingerprint"],
  "stop": false
}
```

El crítico busca: puntos de entrada sin mapear, caminos paralelos sin revisar, modos del ciclo de
vida faltantes, clases elegidas sin unidad, exclusiones sin fundamento, unidades cerradas sin rutas
ni chequeos, y huecos previos que ninguna unidad atiende. **Propone cobertura, no hallazgos.** El
`stop` es su propia evaluación: `true` sólo si no acepta ninguna `missing_units` ni `reassign_ids`.

El padre rechaza unidades fuera del alcance o del terreno permitido, deriva los IDs canónicos de
las aceptadas y las deduplica. **Ante una colisión de ID canónico, se falla** en lugar de fusionar.
Para cada `reassign_id` legítimo con evidencia viva, se agrega el registro terminal exacto al
archivo `attempts` de la unidad con el motivo del crítico, y la siguiente asignación usa un dueño
fresco y evidencia vacía. La ola se incrementa y la tabla se valida antes de asignar de nuevo.

En `estandar` y `profunda`, cuando el crítico posterior a la ola no acepta trabajo nuevo y no quedan
unidades `planned`, se gasta la invocación reservada en un crítico final distinto. **La cobertura se
considera completa sólo cuando ese crítico tampoco encuentra trabajo.** Si encuentra, se encola y se
repite el ciclo.

El perfil acota el ciclo: una corrida `rapida` tiene exactamente una ola de caza y una pasada de
crítico final; lo que aparezca después queda `deferred` con el motivo del perfil.

Un presupuesto lo acota igual: antes de cada ola se compara lo que queda contra los cazadores, la
reserva de validación y los críticos. Si las reservas obligatorias no entran, **no se lanza ningún
cazador de esa ola** y sus unidades quedan `deferred` con el motivo. Nunca se usa un tope de
agentes como evidencia de cobertura completa.

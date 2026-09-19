# Validación, salida estructurada, verificación e informes

### Fase 3: validar cada candidato, de forma independiente

Después de la pasada limpia del crítico de cobertura —o de una parada temprana anotada— el padre
consolida los candidatos por huella y causa raíz. Cada candidato único (`confirmado` o
`needs_validation` propuesto) va a **un verificador fresco que no lo cazó**.

Un verificador puede leer lo que produjo el cazador, pero **tiene que volver a leer cada ubicación
citada del código actual** y reproducir por su cuenta cualquier chequeo decisivo que pueda hacer de
forma segura (dentro de las reglas del terreno).

Se le da: el candidato, los chequeos y las rutas de la unidad ligada, los hechos de arquitectura
necesarios para interpretar el camino, los bloques de validación del compañero que correspondan, el
límite de ejecución permitido, las ramas `confirmed`, `needs_validation` y `rejected` de
`report-schema.json` **textuales**, y los registros previos con la misma huella. **No recibe la
conclusión de otro verificador.**

#### Prompt del verificador de candidatos

```text
Este candidato no lo escribiste vos. Tratá de refutarlo desde el código del repo y desde evidencia
local acotada.

Reglas del terreno: no contactes despliegues ni servicios externos o compartidos; no sondees
internet ni terceros. En esta VM NO hay sandbox del sistema operativo, así que no ejecutes código
que no sea nuestro. SÍ podés leer el código, observar en modo lectura lo propio (curl a 127.0.0.1,
docker inspect, ss, logs) y correr nuestros scripts con datos de prueba en work/tmp, sin red.

Si la confirmación depende de ejecutar algo de un tercero, no lo hagas: dejá el `needs_validation`
con el bloqueo exacto.

1. Verificá cada traza y cada archivo de evidencia, el número de línea positivo, el alcance y la
   descripción. Confirmá que la primera entrada sea un punto de entrada real de menor confianza y
   que la última sea el sumidero o el efecto de frontera que se afirma.
2. Reconstruí los controles más fuertes visibles en el código sobre ese camino: validación,
   identidad, autorización, normalización, ciclo de vida, framework y contención.
3. Para un candidato `confirmado`: reproducí por tu cuenta el resultado mínimo observado cuando sea
   posible, respetando el límite de ejecución. Verificá entradas, forma de la interfaz, condiciones
   y a quién afecta. No infieras un resultado más fuerte ni sigas después de eso.
4. Verificá que la probabilidad, el impacto, la confianza y el arreglo propuesto coincidan **sólo**
   con lo que la evidencia establece.
5. Para un candidato `needs_validation`: decidí si el bloqueo está realmente fuera del código y de
   lo observable. Si el código refuta la traza, se rechaza. Si el hecho que falta sigue siendo
   decisivo, se mantiene, con planes locales y de observación por el dueño que sean exactos y no
   destructivos.
6. Conservá la huella para la misma causa raíz en todos los estados.

Devolvé exactamente un objeto JSON, sin prosa alrededor:
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
donde el registro coincide exactamente con la rama del veredicto en el esquema que se incluyó en
este prompt. Un registro corregido reemplaza la redacción del cazador.
```

Un verificador puede **promover** un `needs_validation` a `confirmado` sólo después de establecer
por su cuenta el camino completo y el resultado observado acotado. Y **degrada** una confirmación
propuesta a `needs_validation` cuando un hecho de despliegue o de runtime sigue sin conocerse. Usa
`rejected` cuando el código, el comportamiento local, un control visible, la falta de impacto real o
un prerrequisito imposible refutan la afirmación.

**`needs_validation` nunca es el estacionamiento de una idea especulativa.**

El padre comprueba que cada verificador haya devuelto la misma huella, salvo que haya identificado
una causa raíz genuinamente distinta. Aplica las correcciones, anota la decisión en cada unidad
ligada y asegura **un registro final por huella**. Un resultado mal formado o envuelto en prosa **se
descarta sin arreglarlo**: se vuelve a correr con un verificador fresco si el presupuesto lo
permite; si no, queda como candidato sin validar en la tabla, bajo la regla de corrida incompleta.

Si el presupuesto no alcanza para validar todo: se corta la caza, se valida mientras alcance, y la
corrida queda `incomplete` con el motivo. **Un candidato sin validar no entra a `findings.json` bajo
ningún veredicto.**

### Fase 4: escribir y validar `findings.json`

El padre escribe todos los registros decididos en `<salida>/findings.json`, ordenados por huella:

- **`confirmed`**: vulnerabilidades fundadas en el código, con evidencia de ejecución local
  completa, condiciones, remediación concreta, probabilidad/impacto/severidad general, y confianza.
- **`needs_validation`**: candidatos fundados en el código con un bloqueo exacto sin resolver y al
  menos un plan aplicable (local u observación por el dueño).
- **`rejected`**: candidatos refutados durante la validación, conservados para que una corrida
  futura no repita la afirmación sin evidencia nueva.

Leé `report-schema.json` **inmediatamente antes de escribir**. Usa `additionalProperties: false`: no
se llevan campos del envoltorio del cazador a un registro. Los tres contratos son distintos:

- `confirmed` usa `root_cause`, `intended_behavior`, `conditions`, `execution`, `remediation`,
  `severity` y `confidence`. **No** usa `claimed_root_cause`, `blockers`, `validation_plan` ni
  `reason`. `execution` describe la interfaz propia del objetivo, y `observed_result` no está vacío.
- `needs_validation` usa `claimed_root_cause`, `trace`, `evidence`, `blockers` y al menos un campo
  no vacío de `validation_plan.local` o `validation_plan.deployment`. **No** usa severidad,
  execution, remediation, reason, ni causa raíz confirmada.
- `rejected` usa `claimed_root_cause`, `trace`, `evidence` y `reason`. **No** usa severidad,
  execution, remediation, blockers ni plan de validación.

Todos los registros llevan huella estable, título, descripción y rutas relativas al repo. Una traza
de varios pasos arranca en `entrypoint`, termina en `sink` y usa `propagation` sólo entre medio.
**La severidad general no puede superar el impacto demostrado.**

```sh
node <skill-dir>/validate-findings.cjs <salida>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <salida>/coverage-ledger.json
```

Se arregla **todo** error estructural y semántico antes de seguir. Que el validador pase prueba el
formato y la consistencia con la tabla, nada más.

### Fase 5: verificar los registros finales con ojos frescos

Se lanza un verificador fresco por cada registro final `confirmed` y `needs_validation`, en
paralelo. Este verificador revisa **el registro estructurado**, no la redacción del cazador, y se
mantiene dentro del terreno permitido.

En una corrida `rapida`, las fases 3 y 5 se juntan: el verificador de la fase 3 también hace estos
chequeos y devuelve el registro final con forma de esquema. Los otros perfiles mantienen las dos
pasadas separadas. **En ningún perfil se saltea la revisión independiente de un `confirmed`.**

Para un `confirmed` tiene que revisar:

1. Cada ruta, línea, alcance y operación descritos en la traza y la evidencia.
2. La interfaz de entrada real y la forma exacta de la entrada.
3. Cada condición, paso de parser o política, capa que impida el efecto, y resultado local observado.
4. A quién o a qué afecta, y el impacto demostrado.
5. La separación de severidad: probabilidad realista, impacto demostrado, y total no mayor al impacto.
6. La estrategia de remediación: que el arreglo sostenga la invariante **sin mover la confianza de
   lugar**.

Para un `needs_validation`:

1. Que el camino del código sea real y sostenga sólo la causa raíz que se afirma.
2. Que cada bloqueo listado sea decisivo y no se pueda responder ya de forma local.
3. Que el candidato nombre una frontera y un resultado posible concreto, no una preocupación genérica.
4. Que haya al menos un campo del plan de validación, y exacto. `local` usa un fixture acotado;
   `deployment` le pide al dueño que observe una configuración, identidad, ruta, política o hecho
   del runtime. **No se inventa un plan para un contexto que no aplica, y nunca se manda tráfico de
   auditoría a un despliegue.**
5. Que la huella coincida con los registros previos y actuales de la misma causa raíz.

Cada verificador devuelve exactamente un objeto JSON, sin prosa:
`{"decision":"verified","fingerprint":"..."}` o
`{"decision":"replace","reason":"...","record":{...}}`.

Un reemplazo que **promueva** el veredicto (sobre todo a `confirmed`) o que cambie materialmente la
causa raíz, la traza, la entrada de ejecución, el resultado observado, el impacto o la severidad
**no se aplica como final**: se le da a un verificador nuevo que no haya cazado, ni validado en la
fase 3, ni propuesto ese reemplazo. Si no hay presupuesto o independencia, el registro disputado
**se saca de `findings.json`**, su unidad queda como candidato sin resolver, y la corrida queda
`incomplete` con el motivo exacto. Sólo las correcciones de redacción o de número de línea que no
cambian el significado se pueden aplicar directo.

Después de cada reemplazo aplicado se corren los dos validadores otra vez y se actualiza la decisión
en las unidades ligadas. `run_status: "complete"` sólo se pone cuando **cada candidato de la tabla
tiene un veredicto final independiente** y cada registro retenido pasó la fase 5.

No se verifican sólo los `confirmado`: un `needs_validation` engañoso le hace perder tiempo al dueño
y puede dejar en pie una premisa falsa.

### Fase 6: los informes, derivados de los registros finales

Sólo después de que la fase 5 pase para todos los registros retenidos, se escribe la prosa a partir
de los registros finales, la tabla y las notas de endurecimiento. Una corrida incompleta puede
informar los registros verificados, pero **tiene que identificar cada candidato sin resolver** y no
puede presentarlo como hallazgo. **La prosa nunca cambia un veredicto, una severidad, un bloqueo ni
el impacto demostrado.**

#### `REPORT.md`

1. Perfil de la corrida, alcance y presupuesto, con los agentes gastados contra los planificados;
   la referencia de código; la aclaración de que la ejecución fue **sólo código propio y observación
   en modo lectura** (no hay sandbox); el uso de corridas previas; y la cobertura diferida y fuera
   de alcance, explícita. Una corrida `rapida`, acotada o incompleta **dice claramente que es
   parcial**. Si el presupuesto impidió correr un crítico obligatorio, se dice cuál y **no se
   afirma cobertura limpia**.
2. Un resumen corto de la postura de seguridad.
3. Una tabla de hallazgos confirmados: severidad, título, frontera afectada y el resultado observado
   en una línea.
4. Cada confirmado: ubicación en el código, principal de menor confianza, reproducción acotada con
   la interfaz propia del objetivo, condiciones, resultado real, impacto, por qué esa prioridad, y
   el arreglo más chico.
5. Una tabla aparte de **NECESITA VALIDACIÓN**, con el título de cada pista, la traza en el código,
   el bloqueo exacto, el próximo paso local acotado y el chequeo seguro que puede hacer el dueño.
   **Sin severidad y sin llamarlo vulnerabilidad confirmada.**
6. Notas de endurecimiento y patrones positivos del código.
7. Resumen de cobertura de la tabla: cubiertas, candidatas, bloqueadas y diferidas, más las
   exclusiones importantes y el resultado final del crítico.

Los `rejected` no se describen como hallazgos: sus huellas se mencionan sólo si explican una
discrepancia previa o una decisión de cobertura.

#### `FINDINGS-DETAIL.md`

Por cada registro confirmado `media`, `alta` o `crítica`, se copia la traza completa y la
reproducción acotada:

- la traza ordenada, relativa al repo, y la evidencia;
- el principal de prueba y el recurso de prueba afectado;
- la entrada, invocación o fixture, y las instrucciones acotadas exactas;
- la salida observada y la invariante de seguridad que prueba;
- las condiciones y la contención;
- la remediación a nivel de código y el caso de regresión.

#### `NEEDS-VALIDATION.md`

Por cada registro sin resolver, se copia la traza, la evidencia verificada, el bloqueo exacto, la
frontera afectada y cada plan aplicable. Quedan como **pistas priorizadas, sin severidad**. No se
convierten en instrucciones para probar contra un despliegue vivo, ni se supone el hecho que falta.

HTTP es **una** interfaz posible, no la de por defecto: un hallazgo puede reproducirse con una
llamada a una función, un fixture, un comando, o mirando una configuración. No se exige un endpoint,
una cuenta externa ni un entorno vivo que el objetivo no tenga.

**El informe va en proporción a la evidencia.** Una corrida limpia puede tener cero confirmados: se
dice el resultado y los límites de cobertura sin inventar hallazgos de relleno.

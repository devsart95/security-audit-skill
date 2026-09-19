# security-audit → auditoría de seguridad de DevSar

Fork de [cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill)
(MIT), adaptado a nuestro entorno.

El método original convierte a un agente en auditor: orquesta agentes aislados por seis fases —
reconocimiento, caza guiada por cobertura, validación adversarial de candidatos, salida
estructurada, verificación independiente y informe— y no deja reportar nada sin traza de código y
sin que otro agente haya intentado refutarlo. **Ese método se conserva entero.** Lo que cambia es
el terreno.

## Qué adaptamos

| | |
|---|---|
| **Nombre y idioma** | `auditoria-seguridad`, en español rioplatense, con los términos de nuestro trabajo |
| **El entorno** | [ENTORNO-DEVSAR.md](skills/auditoria-seguridad/ENTORNO-DEVSAR.md) es nuevo: qué corre en la VM, por dónde escucha, dónde están las zonas sensibles y **qué no se toca** |
| **Una clase propia** | [EXPOSICION-Y-TUNELES.md](skills/auditoria-seguridad/EXPOSICION-Y-TUNELES.md): túneles efímeros, puertos en `0.0.0.0`, el Core, la UI y la cadena de despliegue |
| **Sin sandbox** | El original exige un sandbox del sistema operativo para ejecutar código del objetivo. Acá no hay: la regla pasó a ser **no ejecutar código ajeno**, observar en modo lectura lo propio, y mandar a `needs_validation` todo lo que necesite ejecutar algo de un tercero |
| **Secretos** | Regla explícita: el informe no lleva un solo valor de credencial; se nombran el archivo y su permiso |
| **Alcance** | Nuestros repos, nuestra VM, nuestros servicios. Nada de terceros, ni de internet, ni de producción |
| **Clases de ataque** | [ATTACK-CLASSES.md](skills/auditoria-seguridad/ATTACK-CLASSES.md) reescrito para nuestro stack (Next.js, Docker, Python, git, agentes) |

## Qué se sacó

- `DESKTOP-MOBILE-AND-LOCAL-IPC.md` — no hacemos apps nativas ni IPC local.
- `PROTOCOLS-RPC-AND-MESSAGING.md` — no usamos gRPC, brokers ni colas; lo que hay de HTTP está en
  las clases web.

## Qué queda heredado del original

- Los **validadores** y el esquema, que **son la parte que hace que el método no sea un prompt
  lindo**: `report-schema.json`, `validate-findings.cjs` y `validate-coverage-ledger.cjs`. Cero
  dependencias, se corren con `node`, y **cortan el proceso si el informe no cierra**. No se tocaron
  a propósito: validan **estructura**, no prosa, así que las claves de los JSON siguen en inglés y
  la prosa de este skill está en español. Los dos se verifican con sus propios tests:

  ```sh
  node validate-findings.test.cjs            # 34 pruebas
  node validate-coverage-ledger.test.cjs     # 31 pruebas
  ```

- **Los nombres de archivo y las claves del contrato.** El ledger y los `findings.json` se llaman
  así y usan claves en inglés (`coverage_id`, `fingerprint`, `root_cause`, `needs_validation`…)
  porque el esquema y los validadores las exigen. Traducir la prosa no cambia las claves.

## Estado de la adaptación

Todo el texto del skill —los tres archivos de método, los siete compañeros de dominio y las dos
clases propias— está en español rioplatense y aterrizado en nuestro entorno. El método de seis
fases, los contratos y las tablas de estados se conservan tal cual vinieron.

Lo que se adaptó de fondo, además del idioma: **la ejecución**. El original gira alrededor de un
sandbox del sistema operativo (ejecutar el objetivo con red cortada, entorno vacío y límites) y de
un procedimiento de once pasos para promover artefactos desde ese sandbox. Acá no hay sandbox, así
que ese aparato se reemplazó por la regla opuesta: **no se ejecuta código ajeno**, se observa en
modo lectura lo propio, y lo que necesite ejecutar algo de un tercero queda como `needs_validation`
con el bloqueo exacto. Eso toca `SKILL.md`, `HUNTING.md` (el prompt del cazador) y
`VALIDATION-AND-REPORTING.md` (el del verificador).


## Cómo se usa

Con un agente que pueda lanzar subagentes (el nuestro, por `delegate_task`), desde el repo a
auditar:

```
auditá la seguridad de work/restreamer-ui y dejá el informe en ~/auditorias/
```

```
revisá si la exposición por túnel de la UI filtra algo sin sesión
```

El skill se activa cuando el pedido es de seguridad. Una pregunta puntual usa el **modo consulta**
(no crea directorios ni artefactos); "auditá", "pentest" o "revisión completa" usa el **modo
auditoría** con las seis fases.

La salida de una auditoría completa va a `~/auditorias/<repo>/corrida-<N>/` con `architecture.md`,
`coverage-ledger.json`, `findings.json`, `REPORT.md`, `FINDINGS-DETAIL.md` y
`NEEDS-VALIDATION.md`. **Nunca dentro del repo auditado.**

## Requisitos

- Node.js (para los dos validadores; están probados con sus propios `.test.cjs`).
- Un agente con delegación de subagentes para el modo completo. Sin subagentes las fases 2 y 3 se
  hacen en serie y el informe **debe decir** que no hubo independencia entre el que encontró y el
  que verificó.

## Licencia

MIT — ver [LICENSE](LICENSE). El método y las clases base son de Cloudflare, Inc.; la adaptación
al entorno de DevSar es nuestra.

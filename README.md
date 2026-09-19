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

## Qué queda heredado (y qué falta)

Los archivos de método siguen como vinieron del original, en inglés, porque funcionan y son la
referencia del formato: `RECONNAISSANCE.md`, `HUNTING.md`, `VALIDATION-AND-REPORTING.md`,
`WEB-PROTOCOL-AND-AUTH.md`, `CLIENT-SIDE.md`, `CLOUD-AND-DEPLOYMENT.md`,
`DATA-ISOLATION-AND-LIFECYCLE.md`, `SUPPLY-CHAIN-AND-RELEASE.md`,
`RESOURCE-EXHAUSTION-AND-AVAILABILITY.md`, `MEMORY-SAFETY-AND-BINARY.md`, `AI-AND-LLM.md`.

Y los validadores, que **son la parte que hace que el método no sea un prompt lindo**:

- `report-schema.json` — el esquema de los tres veredictos.
- `validate-findings.cjs` y `validate-coverage-ledger.cjs` — cero dependencias, se corren con
  `node` y **cortan el proceso si el informe no cierra**.

Pendiente para la segunda pasada: traducir esos textos y reescribir los prompts de cazador para
nuestro idioma y nuestro listón de evidencia. Los validadores no hace falta tocarlos: validan
estructura, no prosa.

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

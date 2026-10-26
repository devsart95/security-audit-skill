---
name: auditoria-seguridad
description: Auditoría de seguridad y revisión de vulnerabilidades para los proyectos de DevSar (restreamer-ui, freesky, scripts y la VM), con la exposición propia de por medio (túneles, el Core, contenedores, credenciales). Usar para preguntas de seguridad, revisiones puntuales, investigación de vulnerabilidades, o una auditoría completa con informe. El trabajo completo de seis fases corre sólo si el pedido es explícito (auditoría, pentest, revisión de punta a punta) o si se piden los artefactos.
---

# Auditoría de seguridad — DevSar

Fork de [cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill) (MIT), adaptado
a nuestro entorno. El método es el de ellos: **encontrar fallas de una frontera de confianza real y
entregar la evidencia en el código, la reproducción segura, la prioridad y el arreglo más chico**. Lo
que cambia es el terreno: nuestros repos, nuestra VM, nuestros servicios y nuestras reglas.

Un candidato sin principal afectado, recurso afectado ni resultado de seguridad concreto **no es un
hallazgo**.

## Reglas del terreno (esto es lo que no se negocia acá)

1. **No hay sandbox del sistema operativo.** El skill original exige uno para ejecutar código del
   objetivo; nosotros no lo tenemos. Entonces: **no se ejecuta código que no sea nuestro**, y lo
   nuestro corre sólo con datos de prueba y sin salir a internet.
   - Permitido: leer el código; correr **nuestros** scripts y contenedores; `curl` a
     `127.0.0.1`; `docker inspect`/`ps`; `ss`/`ps`; `git log`; leer archivos.
   - Prohibido: ejecutar el firmware (`dvbapp`, `upgrade`), correr dependencias o builds de
     terceros, levantar el Core ajeno, descargar e instalar nada para la prueba.
   - Si para confirmar hace falta algo de eso, el hallazgo queda **`needs_validation`** con el
     bloqueo exacto y un plan de verificación que sí se pueda hacer (mirar en la Mac de Justino,
     leer una configuración, observar un log).
2. **Nada de terceros, ni de internet.** No se escanean ni sondean otros hosts, ni los paneles
   IPTV, ni `tata.local.net.py`, ni la VM 202 `integrax`, ni producción. Un `curl` a nuestro
   propio servicio expuesto por túnel **sí** (eso es exactamente lo que hay que revisar).
3. **El informe no lleva secretos.** Ni valores, ni tokens, ni claves, ni la firma de la escribana.
   Se nombra el campo, el archivo y su permiso; el valor no se copia. Todo log de FFmpeg o del Core
   pasa por `redactSecrets` antes de ir al informe.
4. **Sin cuota ni gasto**: la auditoría no usa Claude Code ni APIs pagas. Si un paso la necesita,
   se propone y se espera.
5. **La evidencia manda.** Todo hallazgo confirmado se apoya en una traza de código con rutas
   relativas al repo y, si se reprodujo, en el comando exacto y el resultado mínimo observado. Lo
   que no se puede mostrar, no se afirma.

## Modos

- **Modo consulta** (por defecto): preguntas de seguridad, revisiones puntuales, triaje, "¿esto es
  un problema?". Se usan las partes que hagan falta. No se crea directorio de auditoría ni se
  escriben artefactos.
- **Modo auditoría completa**: cuando el pedido es explícito (auditoría, pentest, "revisá todo",
  o se piden los informes). Corren las seis fases y se escriben los archivos de abajo.

Si el pedido puede ser cualquiera de los dos, **una sola pregunta** antes de crear nada.

## Montaje de una auditoría completa

- **Objetivo**: el repo o la zona a auditar (ruta absoluta). Con `work/restreamer-ui` se audita la
  UI; con `work/freesky/repo`, el sistema del receptor.
- **Nombre**: el del repo.
- **Salida**: `~/auditorias/<repo>/corrida-<N>/`, **fuera del objetivo** y con `N` el primer entero
  libre. Nunca dentro del repo (y nunca en `/tmp`).
- **Referencia de fuente**: el commit revisado y si el árbol tiene cambios sin commitear.
- **Perfil**: `rapida` (una pasada), `estandar` (por defecto) o `profunda` (subsistema por
  subsistema, dos pasadas). El perfil cambia el ancho y la redundancia, **nunca el listón de
  evidencia**.

### Quién escribe qué

Sólo el agente principal escribe los archivos compartidos: `run-metadata.json`, `architecture.md`,
`coverage-ledger.json`, `findings.json`, `REPORT.md`, `FINDINGS-DETAIL.md`,
`NEEDS-VALIDATION.md`. Cada cazador o verificador (subagente) recibe su carpeta
`agentes/<id>/` y **no toca** los compartidos, ni el código del objetivo, ni la carpeta de otro.

Los subagentes se lanzan con la herramienta de delegación; si no hay subagentes disponibles, las
fases 2 y 3 se hacen **secuencialmente en la misma sesión** y el informe lo dice (la independencia
del verificador no se puede cumplir sin agentes frescos: en ese caso se declara).

## Las seis fases

1. **Reconocimiento** — mapa de arquitectura, fronteras de confianza, superficies de entrada,
   evidencia previa y la primera tabla de cobertura determinista (`architecture.md` y
   `coverage-ledger.json`). Ver [RECONNAISSANCE.md](RECONNAISSANCE.md).
2. **Caza guiada por cobertura** — se asignan cazadores por unidad de la tabla y críticos de
   cobertura buscan los huecos. Ver [HUNTING.md](HUNTING.md) y las clases de
   [ATTACK-CLASSES.md](ATTACK-CLASSES.md).
3. **Validación de candidatos** — cada candidato único va a un **verificador fresco** que intenta
   **refutarlo**. Ver [VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md).
4. **Salida estructurada** — `findings.json` con los tres veredictos (`confirmado`,
   `needs_validation`, `rechazado`), validado contra `report-schema.json` con
   `node validate-findings.cjs`; y la cobertura validada con `validate-coverage-ledger.cjs`.
5. **Verificación independiente de registros** — agentes frescos vuelven a verificar las
   afirmaciones finales sobre el código.
6. **Informe** — `REPORT.md`, `FINDINGS-DETAIL.md` y `NEEDS-VALIDATION.md` derivados de los
   registros verificados, **sin instrucciones que apunten a un servicio vivo de terceros**.

El validador de cobertura corre después de crear la tabla y después de cada actualización; el de
hallazgos, en la fase 4 y después de cada reemplazo de la fase 5.

No se corta a mitad de fase: o están los tres informes y los dos validadores pasan, o queda
`run_status: "incomplete"` con el motivo exacto y el hueco declarado en el informe.

## Qué se audita en nuestro caso

| Objetivo | Clases que aplican |
|---|---|
| `restreamer-ui` (Next.js + el Core datarhei) | web y autenticación, cliente/navegador, exposición por túnel, datos y credenciales, agotamiento de recursos |
| La VM y su despliegue (Docker, túneles, unidades, watchdog) | [EXPOSICION-Y-TUNELES.md](EXPOSICION-Y-TUNELES.md) y [CLOUD-AND-DEPLOYMENT.md](CLOUD-AND-DEPLOYMENT.md) |
| Agentes y skills (nosotros mismos) | [AI-AND-LLM.md](AI-AND-LLM.md) |
| Scripts y firmware propio | [ATTACK-CLASSES.md](ATTACK-CLASSES.md) y [MEMORY-SAFETY-AND-BINARY.md](MEMORY-SAFETY-AND-BINARY.md) para el binario del fabricante |
| Dependencias y entrega | [SUPPLY-CHAIN-AND-RELEASE.md](SUPPLY-CHAIN-AND-RELEASE.md) |
| Datos guardados (credenciales, firmas, planillas) | [DATA-ISOLATION-AND-LIFECYCLE.md](DATA-ISOLATION-AND-LIFECYCLE.md) |
| Nuestros servicios propios | [WEB-PROTOCOL-AND-AUTH.md](WEB-PROTOCOL-AND-AUTH.md) y [CLIENT-SIDE.md](CLIENT-SIDE.md) |

El mapa concreto de nuestra VM, nuestros servicios y la lista de lo que no se toca está en
[ENTORNO-DEVSAR.md](ENTORNO-DEVSAR.md): **leerlo antes de la fase 1**.

## Principios

### Exigir frontera y resultado

Para cada candidato: principal de menor confianza, entrada o acción aceptada, control previsto,
frontera cruzada, principal o recurso afectado, y resultado concreto (observado o que el dueño
pueda observar). No se eleva a hallazgo una buena práctica faltante, un comportamiento de
despliegue supuesto, un crash genérico ni un daño a uno mismo.

### Prioridad distinta de certeza

Sólo los `confirmado` llevan severidad, y la severidad **no puede superar el impacto demostrado**.
`needs_validation` es una hipótesis de frontera concreta que está bloqueada, no una vulnerabilidad
de baja confianza: **no lleva severidad**.

Anclas de severidad (igual que el original):

- **crítica** — un actor no autenticado consigue ejecución de código, acceso total al
  almacenamiento, o la toma de cuentas arbitrarias.
- **alta** — derrota por completo un control explícito con consecuencias reales: saltarse la
  autenticación, leer o escribir datos de otro, ejecutar script almacenado que afecta a otros,
  ejecución autenticada de código, o parar un servicio compartido sin autenticarse.
- **media** — violación real de frontera de alcance limitado, precondiciones poco comunes, o
  consecuencias acotadas a pocos recursos.
- **baja** — revelar interior no secreto, o un efecto que exige esfuerzo sostenido para poco.
- **informativa** — observación confirmada de impacto mínimo, útil como prerrequisito de otro
  hallazgo.

El discriminador alta/media: ¿el resultado demostrado **derrota por completo** un control explícito
para una acción con consecuencias reales, o sólo lo debilita? Si no se puede decir el daño
concreto, la severidad es menor de lo que parece.

### Validación adversarial

El agente que verifica un hallazgo **nunca** es el que lo encontró.

### Recomendar el arreglo más chico que sirva

Para cada confirmado: la invariante que el código tiene que sostener y el cambio de fuente más
angosto que la sostenga en el último punto de decisión confiable. Preferir cambios concretos con
ruta relativa al repo y un test de regresión antes que consejo genérico. **La auditoría describe
arreglos: no modifica el código del objetivo.**

### Varias corridas mejoran la cobertura

En las pruebas de Cloudflare, una sola corrida encontró cerca de la mitad de lo que encontraron
varias. Ninguna corrida agota el objetivo, y el informe lo dice.

## Anti-patrones

1. Desvíos de checklist presentados como vulnerabilidades.
2. Consejo de defensa en profundidad sin violación alcanzable de una frontera.
3. Probar contra un servicio vivo o compartido cuando alcanza con código y datos de prueba.
4. Suponer comportamiento de despliegue, proxy, navegador o identidad que no está en la fuente.
5. Tratar autoridad del mismo principal, o daño a uno mismo, como resultado de frontera.
6. Reportar un efecto del parser o del runtime más fuerte que el efecto observado.
7. Resultados del cazador en prosa, que no se pueden deduplicar ni verificar.
8. Darle severidad a un `needs_validation`.
9. Escribir el informe antes de la verificación independiente, o dejar que la prosa y el JSON
   digan cosas distintas.
10. **Poner un valor de credencial en el informe, aunque sea en un ejemplo.**
11. **Tocar un servicio de terceros o de producción "para ver si es explotable".**
12. **Contar como hallazgo algo que ya está mitigado por otra capa** (si A lo impide, la falta de B
    es una nota de endurecimiento, no una vulnerabilidad).

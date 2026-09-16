# rn-review-pipeline

Pipeline multiagéntica de revisión de código y testing para repos de React
Native (Expo). Vive en un único sitio y se reutiliza desde cualquier repo de
la cuenta vía un GitHub Actions **reusable workflow** — no hay que
reconfigurar la lógica en cada proyecto nuevo.

## Qué hace

En cada Pull Request:

1. **`rn-reviewer`** (Opus) revisa el diff: bugs, malas prácticas de
   React/RN/Expo, seguridad, convenciones del repo.
2. **`rn-test-writer`** (Sonnet) escribe/actualiza tests Jest + React Native
   Testing Library para el código nuevo, y añade un flujo Maestro (`e2e/`)
   cuando el cambio toca navegación o un flujo de usuario. Si el repo no
   tiene Jest/RNTL configurado, lo monta la primera vez que corre.
3. **`rn-test-runner`** (Sonnet) ejecuta de verdad `npm test` (y typecheck).
   Si algo falla, corrige el test si el test estaba mal (máx. 3 intentos), o
   reporta el bug real sin tocar el código de producto para forzar un pass.
4. Los tests nuevos se commitean a la rama del PR. Se deja un comentario con
   el resumen y se genera un veredicto (`pass`/`fail`) que determina si el
   job de GitHub Actions termina en verde o en rojo.

**Límite conocido**: el flujo Maestro se *escribe* pero no se ejecuta en CI
(no hay un emulador Android/iOS configurado en este workflow todavía) — eso
sería una fase siguiente si hace falta e2e ejecutado automáticamente.

## Protección contra loops infinitos

Cuando el pipeline pushea tests nuevos a la rama del PR, ese push dispara un
nuevo evento `synchronize` — sin cuidado, eso podría re-disparar el pipeline
indefinidamente. Dos garantías, ninguna basada en "confiar en que el agente
obedezca":

1. **Un solo push por PR, forzado por el propio YAML.** Un step
   determinístico (`Check for prior pipeline commit`) revisa el historial de
   git buscando un commit previo del propio pipeline en este PR. El step que
   hace `git commit`/`git push` tiene un `if:` que se salta por completo si
   ya existe uno — así el agente escriba archivos igual en esa corrida, no
   hay ningún paso que los suba. No depende de que el modelo decida no
   pushear (aunque también se le indica eso en el prompt como refuerzo).
2. **Circuit breaker numérico, independiente del agente.** Antes de instalar
   o correr Claude, un step cuenta cuántos workflow runs ya existen para ese
   PR vía la API de GitHub; si supera 5, el job falla inmediatamente sin
   gastar nada. Es el techo real, absoluto, para cualquier caso raro que las
   dos protecciones anteriores no contemplen.

En el flujo normal esto da como mucho 2 corridas por evento humano: la que
revisa y pushea tests, y una de verificación sobre ese mismo push que ya no
puede pushear de nuevo.

## Cómo integrar un repo nuevo

Este pipeline corre con tu **suscripción de Claude (Pro/Max)**, no con una
API key de pago por token:

1. En tu máquina (con la CLI de Claude Code y sesión iniciada en tu cuenta
   Pro/Max), genera un token: `claude setup-token`. Copia el valor que
   imprime.
2. En el repo del proyecto RN (no aquí), añade ese valor como secret
   `CLAUDE_CODE_OAUTH_TOKEN` en Settings → Secrets and variables → Actions.
   (Si mueves tus repos a una organización, puedes ponerlo una sola vez como
   secret de organización para que cubra todos los repos.)
3. Crea `.github/workflows/pr-review.yml`:

   ```yaml
   name: PR Review

   on:
     pull_request:
       types: [opened, synchronize, reopened]

   jobs:
     rn-review:
       uses: dkdk22/rn-review-pipeline/.github/workflows/rn-multiagent-review.yml@main
       secrets:
         CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
   ```

4. Una vez que el workflow haya corrido al menos una vez, ve a Settings →
   Branches → Branch protection rule para tu rama principal, y marca como
   required el check **`rn-review-pipeline`** (no el nombre del job de
   Actions) — es un commit status que el propio run publica explícitamente
   sobre el commit final, así que siempre cubre el SHA real del PR incluso
   cuando el pipeline le agrega commits propios.

Nota: el token de `claude setup-token` puede expirar con el tiempo — si el
workflow empieza a fallar en el paso de autenticación, genera uno nuevo y
actualiza el secret. También cuenta contra los límites de uso de tu plan
Pro/Max, igual que usar Claude Code interactivamente.

Eso es todo — actualizar `agents/*.md` o `prompts/orchestrator.md` aquí
mejora el pipeline en todos los repos que lo usan, sin tocar nada en ellos.

## Nota sobre repos privados

Si `rn-review-pipeline` es privado, GitHub solo permite llamarlo como
reusable workflow desde otros repos de la misma cuenta/organización con los
permisos adecuados. Si algún repo cliente no puede resolver el `uses:`,
verifica los permisos de acceso entre repos en Settings, o considera hacer
este repo público (no contiene nada sensible: son prompts y workflows).

## Estructura

```
agents/               subagentes (.claude/agents/*.md): reviewer, test-writer, test-runner
prompts/orchestrator.md   prompt del agente orquestador (top-level)
.github/workflows/rn-multiagent-review.yml   workflow reutilizable (workflow_call)
```

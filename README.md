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

## Cómo integrar un repo nuevo

En el repo del proyecto RN (no aquí):

1. Añade el secret `ANTHROPIC_API_KEY` en Settings → Secrets and variables →
   Actions (repo secret; si mueves tus repos a una organización, puedes
   ponerlo como secret de organización una sola vez para todos).
2. Crea `.github/workflows/pr-review.yml`:

   ```yaml
   name: PR Review

   on:
     pull_request:
       types: [opened, synchronize, reopened]

   jobs:
     rn-review:
       uses: dkdk22/rn-review-pipeline/.github/workflows/rn-multiagent-review.yml@main
       secrets:
         ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
   ```

3. Una vez que el workflow haya corrido al menos una vez, ve a Settings →
   Branches → Branch protection rule para tu rama principal, y marca como
   required el check `RN Multiagent Review (reusable) / rn-review`.

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

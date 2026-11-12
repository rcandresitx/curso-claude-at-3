---
name: implement-plan
description: Implementa el plan que está como comentario de una issue de GitHub (se indica por número o URL), aplicando TDD estricto, tarea a tarea, en un git worktree propio dentro de .claude/worktrees y con una rama <num_issue>-<breve-descripcion> creada desde development. Deja una PR creada para su revisión. Úsala cuando pidan implementar, ejecutar o desarrollar el plan de una issue.
argument-hint: <num_issue o url de la issue>
---

# implement-plan

Implementa el plan de la issue indicada en `$ARGUMENTS` tarea a tarea, con TDD estricto, marcando cada tarea en el plan y haciendo un commit por tarea. Trabaja en un worktree aislado y termina dejando una PR abierta para revisión; **nunca la mergees**.

## Pasos

### 1. Localiza y lee el plan en la issue

- El usuario debe indicar el número o la URL de la issue en `$ARGUMENTS` o en la conversación. Si no, pregúntala antes de continuar.
- Lee la issue con `gh issue view <num_o_url> --json number,title,state,comments`. Si `gh` falla (no instalado, sin autenticar, issue inexistente), informa del error y detente.
- Localiza el comentario que contiene el plan (publicado por la skill `plan-feature`: tiene las secciones Objetivo, Alcance, Archivos afectados, Tareas, Criterios de aceptación y Dudas y decisiones tomadas). Si hay varios, usa el más reciente; si no hay ninguno, informa al usuario y sugiérele ejecutar `plan-feature`; no inventes el plan.
- Lee `CLAUDE.md` y respeta sus convenciones (capas controllers/services/repositories/models/utils, funciones en snake_case, prettier sin `;` y con comillas simples, eslint, tests con vitest).
- Si en "Dudas y decisiones tomadas" hay dudas abiertas que bloquean la implementación, pregúntalas al usuario antes de empezar.

### 2. Prepara la rama y el worktree

El nombre de la rama es `<num_issue>-<breve-descripcion>` (kebab-case, minúsculas, sin tildes, de 2 a 5 palabras, derivado del título de la issue; ej.: `42-paginacion-vendehumos`).

1. Ejecuta `git status --short` y `git branch --list development main`. Si `development` **no existe**, créala a partir de `main` sin cambiar de rama: `git branch development main`. No la publiques.
2. Comprueba que `.claude/worktrees/` está ignorado (`git check-ignore .claude/worktrees`). Si no lo está, añádelo a `.git/info/exclude` (no modifiques `.gitignore` sin avisar).
3. Crea el worktree y la rama desde `development`: `git worktree add .claude/worktrees/<rama> -b <rama> development`. **Todo el trabajo se hace dentro de ese worktree**, no en el árbol principal (ejecuta los comandos con `cd .claude/worktrees/<rama>`; si hace falta, `npm install` allí).
4. Si la rama o el worktree ya existen, no los recrees ni sobrescribas: pregunta al usuario si continuar en ellos (para retomar un plan a medias; en ese caso salta las tareas ya marcadas `- [x]`).
5. Guarda el plan del comentario en `plans/<rama>.md` dentro del worktree (créalo si no existe); se commiteará junto con la primera tarea.
6. **Varios agentes**: si el plan tiene tareas independientes que conviene repartir entre varios agentes, crea un worktree y una rama por agente (`<rama>-<sufijo>`, p. ej. `-api`, `-web`) con `git worktree add .claude/worktrees/<rama>-<sufijo> -b <rama>-<sufijo> <rama>`, cada agente trabaja solo en el suyo, y al terminar integras las ramas en `<rama>` (merge sin fast-forward, sin `--force`) antes de abrir la PR. Si no hay paralelismo real, usa un único worktree.

### 3. Implementa cada tarea con TDD estricto

Recorre las tareas `- [ ]` **en orden**, de una en una. Para cada tarea:

1. **Red**: escribe primero el test (vitest) que describe el comportamiento de la tarea. Ejecútalo (`npx vitest run <archivo>`) y **comprueba que falla** por la razón esperada (no por un error de sintaxis o de import). No escribas código de producción antes de ver el test en rojo.
2. **Green**: escribe el código **mínimo** necesario para que el test pase, en la capa que corresponda. Ejecuta el test y confirma que pasa.
3. **Refactor**: limpia el código manteniendo los tests en verde. Aplica prettier y eslint a los archivos tocados.
4. **Verifica**: ejecuta la suite completa (`npm test`) y comprueba que nada anterior se ha roto. Si algo falla, corrígelo antes de seguir.
5. **Marca la tarea como hecha** en el archivo del plan, cambiando `- [ ]` por `- [x]` solo para esa tarea y solo cuando el test pase con el código implementado.
6. **Commit**: invoca el slash command `/commit` (con la herramienta Skill, skill `commit`) pasando como argumento un contexto breve con el número y título de la tarea. El commit debe incluir el código, el test y el plan actualizado. No hagas `git commit` a mano ni `git push` en este paso.

Tareas sin comportamiento testable (instalar dependencias, crear archivos de configuración, documentar en `CLAUDE.md`, etc.):
- No inventes tests artificiales. Verifica el resultado con un comando observable (p. ej. `npm ls`, `npx eslint .`, `npx prettier --check .`, arrancar el comando) y, si pasa, marca la tarea y haz commit igualmente.
- Si la tarea es la que configura vitest y aún no existe, instálalo y configúralo primero, y usa su ejecución como verificación.

### 4. Cierre y PR

- Cuando todas las tareas estén en `- [x]`, recorre los **criterios de aceptación** del plan y comprueba cada uno (tests, ejecución real si procede). Marca `- [x]` los que se cumplan e informa de los que no.
- Ejecuta `git status` y `git log --oneline development..HEAD` en el worktree para confirmar que el árbol está limpio y hay un commit por tarea.
- Publica la rama (`git push -u origin <rama>`) y crea la PR contra `development` con `gh pr create --base development --head <rama> --title "<título>" --body-file <fichero>`. El cuerpo debe incluir `Closes #<num_issue>`, un resumen de los cambios, los criterios de aceptación cumplidos/pendientes, las desviaciones respecto al plan y terminar con la línea de atribución indicada en el contexto, si la hay. Crea la PR como **ready for review** (o `--draft` si quedaron tareas o criterios pendientes).
- No mergees la PR ni actives auto-merge: debe revisarla otra persona. No elimines el worktree hasta que la PR se haya mergeado o el usuario lo pida.
- Responde con: rama, worktree, URL de la PR, tareas completadas, resultado de la suite de tests y cualquier decisión o desviación respecto al plan.

### 5. Avisa por Slack

- Como último paso, envía un mensaje al canal de Slack `curso-claude-ed3` indicando que el plan ya se ha implementado. Usa la herramienta MCP de Slack (`mcp__slack__slack_post_message` para publicar el mensaje en el canal con el id "C0CGZ01R9M2"; si no está cargada, cárgala antes con ToolSearch).
- El mensaje debe incluir el número de issue, la rama y la URL de la PR (ej.: `Ya se ha implementado la issue 42 (rama 42-paginacion-vendehumos); PR lista para revisión: <url>.`). Si quedaron tareas sin completar o criterios sin cumplir, indícalo en el mensaje en lugar de decir que está terminado.
- No incluyas datos privados (emails, rutas locales, secretos). Si no se puede enviar el mensaje, informa al usuario del fallo, sin reintentar en bucle.

## Reglas

- Una tarea = un ciclo TDD = un commit. No agrupes varias tareas en un commit ni dejes tareas hechas sin commitear.
- Nunca marques una tarea como hecha si su test no pasa.
- No te saltes el paso rojo del test ni escribas código "por adelantado" de tareas posteriores.
- Si una tarea no se puede completar o el plan es incorrecto, para, explica el problema y pregunta al usuario; no improvises cambios fuera del alcance del plan.
- `git push` solo de la rama del plan, y solo en el paso de cierre para crear la PR. Nunca `--force`, `--amend`, `--no-verify` ni modifiques la configuración de git.
- No mergees la PR. No toques `main`, salvo para crear `development` a partir de ella.
- No trabajes fuera del worktree creado para la tarea.

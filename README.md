# ai-code-review-workflows

Revisión automática de pull requests con Claude, como un workflow reutilizable de GitHub Actions. Cada repositorio agrega un archivo pequeño que lo llama; el comportamiento, el modelo y los límites viven aquí, en un solo lugar.

Claude **solo lee y comenta**: publica un comentario por cada problema en la línea donde está, o un resumen cuando no encuentra nada. No puede hacer commits, pushes ni merge. La decisión de fusionar sigue siendo humana.

## Cómo funciona

```text
ealpuche/ai-code-review-workflows            (este repo, público)
└── .github/workflows/claude-pr-review.yml   workflow reutilizable
              ▲
              │ uses: ...@v1
              │
Cada repositorio revisado
├── .github/workflows/revision-ia.yml        dispara la revisión al abrir un PR hacia main
└── .github/claude-review.md                 política de revisión propia del repo
```

En cada PR, el workflow:

1. Comprueba que exista una credencial de Anthropic y que el PR no sea borrador ni venga de un fork.
2. Lee `.github/claude-review.md` **desde la rama base** (un PR no puede cambiar las reglas con las que se revisa a sí mismo) y la entrega a Claude como memoria de usuario (`~/.claude/CLAUDE.md`). No se escribe en el `CLAUDE.md` del repo porque la Action lo restaura desde la rama base antes de correr.
3. Arma un **paquete de revisión ciega** en `.revision-ia/`: la descripción del PR sin firmas de autoría ("Generated with Claude Code", `Co-Authored-By`), el diff contra el merge-base y la lista de archivos. Claude no recibe ni puede consultar los comentarios del PR, así que no lo influyen otras revisiones (Copilot, revisiones anteriores).
4. Corre Claude con [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action) usando el prompt de revisión de este repositorio (`modo: propio`, por defecto). Reglas principales: leer el diff completo, juzgar solo lo observable (si falta información, marcar EVIDENCIA INSUFICIENTE), recorrer una lista de casos límite, verificar las afirmaciones de la documentación contra el código, revisar las guardas de operaciones destructivas, respetar la sección de fuera de alcance del PR salvo riesgos graves y no emitir un hallazgo BLOQUEANTE o IMPORTANTE sin un caso que lo reproduzca. Publica un comentario en línea por hallazgo (H1, H2… con severidad y confianza) y **siempre** un comentario "## Revisión de Claude" con el SHA, la tabla de hallazgos y un veredicto: APROBAR, APROBAR CON CAMBIOS o RECHAZAR.
5. Si la Action falla antes de que Claude arranque, por ejemplo por una caída de red al instalar sus dependencias, espera 30 segundos y lo intenta una vez más. Si Claude ya arrancó, no reintenta: el error se repetiría y costaría dos veces. Si vuelve a fallar, deja en el resumen del job una tabla con las causas típicas y qué hacer.

## Seguridad

| Control | Cómo |
| --- | --- |
| Sin escritura en el código | `contents: read`. Herramientas permitidas en `modo: propio`: leer archivos (`Read`, `Glob`, `Grep`), `gh pr comment` y comentarios en línea. No tiene `gh pr view` ni `gh pr diff`: todo lo que necesita está en el paquete. Sin `Write`, `Edit`, shell libre, commits ni merge. |
| Token de GitHub mínimo | Se usa `github.token` del propio run, no un token personal ni la GitHub App de Claude. |
| Credencial de Anthropic | Solo como secreto, pasado de forma explícita. Nunca en el YAML. |
| Forks | Excluidos: GitHub no les entrega secretos y no se usa `pull_request_target`. |
| Copias del repo | El caller corre solo si el dueño es el esperado, para que repos creados desde una plantilla no hereden un check que falla. |
| Costo acotado | `--max-turns`, `timeout-minutes` por intento y por job, y una sola revisión activa por PR (`concurrency`). |
| Dependencias fijas | `claude-code-action` y `checkout` fijadas por SHA. Dependabot propone las actualizaciones cada semana como PR. |

Los archivos, comentarios y descripciones del PR se tratan como datos, no como instrucciones.

## Instalar en un repositorio

### 1. Credencial de Anthropic

Con suscripción de Claude (Pro, Max, Team o Enterprise), en tu computadora:

```bash
claude setup-token
```

Copia el token y guárdalo como secreto `CLAUDE_CODE_OAUTH_TOKEN` en el repositorio:

```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo OWNER/REPO
```

`gh` pide el valor y no lo deja en el historial. En GitHub Free los secretos de organización no llegan a repositorios privados, así que se crea uno por repositorio.

Con API key de [Claude Console](https://platform.claude.com) el secreto se llama `ANTHROPIC_API_KEY` y en el caller se pasa con ese nombre. Configura solo una de las dos credenciales.

### 2. Caller

Copia [`ejemplos/revision-ia.yml`](ejemplos/revision-ia.yml) a `.github/workflows/revision-ia.yml` y ajusta el dueño en la condición `if`.

### 3. Política del repositorio

Crea `.github/claude-review.md` con lo que importa revisar en ese proyecto. Hay un ejemplo en [`ejemplos/politicas/ejemplo.md`](ejemplos/politicas/ejemplo.md). Se usa un archivo en `.github/` y no un `CLAUDE.md` en la raíz para que no se copie a quien use el repo como plantilla.

## Versiones

Los callers apuntan a `@v1`. Para publicar una versión:

```bash
git tag v1
git push origin v1
```

Cada versión lleva su etiqueta exacta (`v1.1.0`) y la etiqueta `v1` se mueve a la última versión compatible, igual que en las Actions oficiales. Un cambio que rompa la interfaz (inputs o secretos) va en `v2`. Si prefieres no recibir cambios sin revisarlos, fija el caller a `@v1.1.0` o al SHA completo de un commit.

| Versión | Cambios |
| --- | --- |
| `v1.3.0` | Revisión ciega con prompt propio en español como modo por defecto (`modo: propio`): paquete con descripción sin firmas de autoría y diff contra el merge-base, sin acceso a los comentarios del PR; reglas de evidencia, casos límite, documentación normativa y operaciones destructivas; hallazgos H1… con severidad y confianza; siempre publica un resumen con veredicto. El plugin oficial queda como `modo: plugin`. |
| `v1.2.0` | `max_turns` por defecto sube de 8 a 30. El reintento solo ocurre si Claude no llegó a arrancar. El caller de ejemplo ya no fija `model` ni `max_turns` para heredar los valores del workflow central. |
| `v1.1.1` | La política se lee de la rama base y no de `base.sha`, que GitHub no actualiza al reabrir un PR. |
| `v1.1.0` | Reintento ante fallas temporales, Actions fijadas por SHA con Dependabot, input `debug`, resumen de fallos en el job, política cargada como memoria de usuario. |
| `v1` | Primera versión. |

## Inputs

| Input | Por defecto | Para qué |
| --- | --- | --- |
| `model` | `claude-sonnet-5-5` | Equilibrio entre costo, rapidez y calidad. Para PRs de seguridad o arquitectura, `claude-opus-5-5`. |
| `max_turns` | `30` | Tope de turnos de Claude por revisión. El plugin usa varios agentes y con menos de ~20 no termina en PRs grandes. |
| `policy_path` | `.github/claude-review.md` | Ruta de la política dentro del repositorio que llama. |
| `modo` | `propio` | `propio`: prompt en español de este repo, siempre deja resumen. `plugin`: plugin oficial `code-review`, que solo marca bugs evidentes del diff, publica en inglés y puede omitir un PR sin dejar comentario. |
| `debug` | `false` | Muestra la salida completa de Claude en el log. Solo para diagnosticar y por poco tiempo: el log puede incluir el contenido del PR, y en repos públicos los logs son públicos. |

## Límites conocidos

- Revisa al **abrir, reabrir o marcar listo** un PR, no en cada push. Para revisar de nuevo tras cambios, cierra y reabre el PR. En `modo: propio` cada revisión publica su propio resumen; en `modo: plugin` el plugin omite PRs que ya tienen un comentario suyo.
- Los PRs desde forks no se revisan.
- El token de `claude setup-token` dura un año. Al pegarlo, la terminal puede partirlo en dos renglones: guárdalo con `printf '%s' "$T" | gh secret set ...` para que llegue completo y sin saltos de línea.
- El reintento puede repetir comentarios si el primer intento alcanzó a publicar algo antes de fallar. Es poco probable, porque casi todas las fallas ocurren antes de que Claude empiece.
- Con token de suscripción hay reportes de errores 401 en Actions ([claude-code-action#1614](https://github.com/anthropics/claude-code-action/issues/1614)). Si aparece, cambia a `ANTHROPIC_API_KEY`: el workflow ya acepta las dos credenciales.
- El check es informativo. No lo marques como obligatorio en la protección de `main` hasta medir su estabilidad.
- Si el código lo escribió Claude, la revisión con Claude puede compartir sus puntos ciegos. El prompt lo mitiga, pero no lo elimina: las pruebas automáticas (pytest, linters) siguen siendo el control principal, y conviene una segunda opinión de otro proveedor en los PRs de alto riesgo.
- No se usa un repomix del merge-base: Claude lee los archivos completos del repositorio con `Read`, `Glob` y `Grep`, lo que escala mejor que meter todo el repo en el contexto. Lo que ve es el estado del PR; el estado anterior se deduce del diff.
- PRs muy grandes: si GitHub no entrega el diff completo, el job falla con un mensaje claro en vez de revisar a medias.
- El veredicto (RECHAZAR incluido) no bloquea el merge: el check queda en verde mientras la revisión corra bien.
- Claude no ejecuta el código del PR. Es a propósito: el job tiene la credencial de Anthropic y el código del PR no se corre junto a ella. Las pruebas van en otro workflow sin secretos.

## Evaluar contra otro revisor

Corre ambos revisores durante dos o tres semanas sobre los mismos PRs, sin que ninguno sea obligatorio, y registra: hallazgos verdaderos, falsos positivos, bugs importantes omitidos, comentarios duplicados, costo y tiempo por PR.

## Licencia

MIT.

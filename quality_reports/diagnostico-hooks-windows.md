# Diagnóstico: `ledger-coverage` y `hook-battery` fallan en Windows

**Fecha:** 2026-10-21
**Revisión leída y probada:** `fcc3036` (los mismos fallos ya estaban en `4c8bace`)
**Equipo:** Windows 11 Pro, Git Bash (MSYS), Python 3.12
**Estado:** diagnóstico. **No se ha aplicado ningún arreglo.**

## Resumen

`./scripts/backtest.sh` da exit 1 por dos comprobaciones:

- `ledger-coverage`: 20 fallos.
- `hook-battery`: 45 casos fallidos.

Las otras ocho comprobaciones están en verde.

Casi todo son fallos de las propias comprobaciones al ejecutarse en Windows. **La excepción es la causa A: un fallo real del guard `root-of-trust-guard.py`, que afecta al uso real** — `git -C .claude -C hooks clean -fd` borraría los hooks sin que el guard lo bloquee.

---

## 1. `ledger-coverage` (20 fallos)

**Causa.** `scripts/check-ledger-coverage.py:185` (`named_by`) y `:216` (lambda `rel`) usan `os.path.relpath`, que en Windows devuelve rutas con `\` (`.claude\hooks\notify.sh`). El ledger y `git ls-files` escriben las rutas con `/`, así que no hay coincidencias:

- "runs as … but the ledger does not name it": 7 fallos (`rel.endswith(tok)` con `tok` en formato `/`).
- "not tracked by git": 13 fallos (`r in tracked`, donde `tracked` viene de `git ls-files`).

Los archivos sí están en el ledger y sí están en git (comprobado con `git ls-files` y `grep` sobre `LEDGER.md`).

**Arreglo propuesto.** En las dos líneas, normalizar el resultado:

```python
os.path.relpath(path, ROOT).replace(os.sep, "/")
```

**Comprobado.** Ejecuté el script con `os.path.relpath` parcheado en memoria de esa forma y terminó con exit 0: "Ledger and running checks agree: 23 registered check(s) accounted for, every named checker on disk, every wiring live". Los avisos que quedan son las filas "Not yet qualified", que son deuda ya registrada y no cuentan como fallo.

---

## 2. `hook-battery` (45 casos): cinco causas

| Causa | Casos | Nº | Comprobado |
|---|---|---|---|
| A. Composición de `-C` con `\` en el guard | a54 | 1 | Sí |
| B. Rutas MSYS en el `cwd` de los eventos | c1, c9, c15, c16, c18, c23, c24, c26, c27, c29, c39, c43, c44, c46, c48, c50, c65, c66 | 18 | c1 directamente; el resto se deduce por los mismos fixtures |
| C. `gh` y `git` falsos sin extensión | h2, h5, h6, h7, i1–i5, i7–i9, c51, c52, c55 | 16 | Por el código y los mensajes de error |
| D. Sin symlinks nativos | c57–c64, h9 | 9 | Sí |
| E. En Windows se ignora `HOME` | f8, f16 | 2 | Sí |
| **Total** | | **45** | |

### A. Fallo real del guard: `-C` compuestos (a54)

**Causa.** En `.claude/hooks/root-of-trust-guard.py`, `_compose_dash_c` une los `-C` con `os.path.join`. En Windows, `['.claude', 'hooks']` da `'.claude\\hooks'`, y `is_protected` solo reconoce rutas con `/`.

**Prueba:**

```
_compose_dash_c(['.claude', 'hooks'])  -> '.claude\\hooks'
is_protected('.claude\\hooks')          -> False
is_protected('.claude/hooks')           -> True
```

El caso a22 (`git -C .claude/hooks clean -fd`, un solo `-C`) sí se deniega. El a54 sigue sin denegarse aunque `cwd` vaya en formato Windows, así que no se debe a la causa B.

**Impacto.** Afecta al uso real en Windows: `git -C .claude -C hooks clean -fd` pasaría sin bloqueo y borraría los hooks.

**Arreglo propuesto.** En `_compose_dash_c`, unir con `posixpath.join`, o normalizar `\` a `/` antes de llamar a `is_protected`.

- El propio guard protege el archivo, así que editarlo exige la vía de escape documentada.
- Según `.claude/rules/issue-ledger.md`, este fallo debería tener un issue en GitHub. No se ha abierto.

### B. Rutas MSYS en el `cwd` de los eventos (18 casos)

**Causa.** `TMPDIR` en Git Bash vale `/c/Users/CARLOS~1/AppData/Local/Temp`, así que `$TMP` y `$ROOT` llegan a los eventos JSON en formato MSYS. El Python de Windows no entiende esas rutas (`os.path.isdir('/c/Users/...')` da `False`). Como no encuentra el repositorio, `git-guardrails.py` deja pasar el comando, tal como está diseñado.

**Prueba (c1, `git merge feature-x` en un repositorio con un archivo sin seguimiento):**

```
cwd=/c/Users/CARLOS~1/AppData/Local/Temp/probe.igoM  -> (salida vacía: permitido)
cwd=C:/Users/CARLOS~1/AppData/Local/Temp/probe.igoM  -> permissionDecision: deny
```

c39, c48 y c50 son la cara inversa: esperan silencio y reciben un deny ("names a repository this guard cannot resolve"), porque las rutas de `-C` del comando también están en formato MSYS.

**Arreglo propuesto.** Al principio de `scripts/hook-battery.sh`, después de crear `TMP` y fijar `ROOT`:

```bash
if command -v cygpath >/dev/null 2>&1; then
    TMP="$(cygpath -m "$TMP")"; ROOT="$(cygpath -m "$ROOT")"
fi
```

(Hay que comprobar el nombre exacto de las variables y el orden al aplicarlo.)

**Nota.** En el uso real, Claude Code probablemente manda `cwd` en formato Windows, así que lo más seguro es que B solo afecte a la batería. No está comprobado.

### C. `gh` y `git` falsos sin extensión (16 casos)

**Causa.** La batería crea dobles de `gh` (`$TMP/fakegh/gh`, para los casos h e i) y de `git` (`shim-slow`, `shim-fail` y `shim-stalled`, para c51, c52 y c55) como scripts de bash sin extensión. En Windows:

- `shutil.which("gh")` solo encuentra ejecutables con una extensión de `PATHEXT`. Por eso aparece "the GitHub CLI (gh) is not on PATH".
- `subprocess` con `["git", ...]` y `CreateProcess` no ejecuta scripts sin extensión. Por eso se usa el `git` real y no el doble.

**Arreglo propuesto, dos opciones:**

- **(a)** Cambiar el código de producción para que `open-issues.py`, `file-issue.py` y `git-guardrails.py` llamen a la ruta completa que devuelve `shutil.which()`, y que la batería cree además envoltorios `gh.cmd` y `git.cmd`.
- **(b) Recomendada.** Que la batería marque esos casos como **SKIP en Windows**, con el motivo y el recuento a la vista, y no como PASS. El fallo está en cómo se construye la prueba, no en los hooks.

### D. Sin symlinks nativos (9 casos)

**Causa.** El Modo de desarrollador de Windows está desactivado (no existe la clave `AppModelUnlock\AllowDevelopmentWithoutDevLicense`). Por eso `ln -s` en Git Bash copia en vez de enlazar, y con `MSYS=winsymlinks:nativestrict` da "Operation not permitted".

- c57–c64: "the sym fixtures are not in the defect state".
- h9: `ln -s "$(command -v python3)" "$NOGH/python3"` deja una copia sin `.exe` que no se puede ejecutar, de ahí "hook exited 127".

**Arreglo propuesto.** SKIP visible en Windows cuando no se pueda crear un symlink nativo, igual que en C(b). La alternativa es activar el Modo de desarrollador y ejecutar con `MSYS=winsymlinks:nativestrict`, pero depende de cómo esté configurado cada equipo.

### E. En Windows se ignora `HOME` (2 casos)

**Causa.** `session-handoff.py` guarda su estado en `Path.home() / ".claude" / "sessions" / <hash>`. En Windows, `Path.home()` usa `USERPROFILE` e ignora `HOME`. La función `hand()` de la batería solo pasa `HOME="$home"`, así que los casos f no reciben un directorio aislado y comparten el `~/.claude/sessions` real.

**Prueba.** Con `HOME=/x`, `os.path.expanduser('~')` da `C:\Users\Carlos Zapata`.

**Arreglo propuesto.** En `hand()`, pasar también `USERPROFILE="$home"`.

**Efecto secundario.** En `~/.claude/sessions/` hay unas 15 carpetas de estado con hash creadas hoy. Al menos parte son restos de las ejecuciones de la batería. No se ha borrado nada.

---

## Resultado esperado

| Arreglos aplicados | Fallos de `hook-battery` |
|---|---|
| Ninguno | 45 |
| A + B + E | 25 |
| A + B + E + C(b) + D | 0, con 25 casos marcados como SKIP |

Con el arreglo de la sección 1, `ledger-coverage` pasaría a verde (comprobado). Las cifras de `hook-battery` son estimaciones que solo se confirmarán al aplicar los cambios y volver a ejecutar `./scripts/backtest.sh`.

## Orden recomendado

1. A: cerrar el fallo real del guard y abrir su issue.
2. Sección 1: `ledger-coverage`.
3. B y E: cambios pequeños en la batería.
4. C(b) y D: SKIP visibles en Windows.

# Fase 1 — Mascota viva: plan de implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Que Aura, un durazno animado, flote en el escritorio de la Mac. Reacciona con emociones a lo
que pasa con los videos de VideoQA (llega, se revisa, sale con errores o aprobado), se duerme
cuando no hay nada, enseña un globito con el estado al hacerle clic y revisa los videos que le
sueltan encima.

**Architecture:** Hay dos repos.
- En `videoqa` (Python) se añaden un marcador de "revisión en curso", el comando
  `videoqa estado --json` y la skill `/aura:mascota`, que compila e instala la app.
- En `aura-app` (Swift) hay dos partes:
  - Una librería pura, `AuraNucleo`, con la máquina de estados, las frases, el contrato JSON,
    las diferencias entre fotos del estado y el cliente del motor. Se prueba entera.
  - Un ejecutable, `Aura`, con el `NSPanel` flotante, el dibujo del durazno en SwiftUI, la
    vigilancia de carpetas con FSEvents y el globito.

  La app nunca lee las carpetas para decidir. Siempre pregunta a `videoqa estado --json` y
  compara con la foto anterior.

**Tech Stack:** Swift 5.10 tools (compila con Swift 6.1, modo de lenguaje 5), SwiftUI + AppKit,
swift-testing (`import Testing`), CoreServices FSEvents, ServiceManagement. Python 3.12 + pytest en
`videoqa`.

**Spec:** `docs/superpowers/specs/2026-10-17-aura-mascota-design.md`. Fase 1 de la sección 11.

## Global Constraints

- macOS 14 o superior, chip Apple: `platforms: [.macOS(.v14)]`, `LSMinimumSystemVersion` 14.0.
- Todo texto visible en español, de tú, tierno y breve. Los emojis, con moderación (🍑 🥺 ✨).
- La Mac de destino tiene 8 GB de RAM. La app debe quedar por debajo de 60 MB en reposo y nunca
  arrancar procesos en bucle: cada refresco se agrupa con espera de ≥ 1,5 s y hay un sondeo cada
  20 s.
- Sin dependencias externas en Swift: solo frameworks de Apple.
- Colores del durazno: cuerpo `#F7B9AE`, brillo `#FBD3C7`, línea `#E7998D`, mejillas `#F29C9C`,
  hoja `#8CC08A` (nervio `#6FA56D`), tallo `#8A5A3B`, visor `#2B2340`, ojos y boca `#CDBEFF`.
- Emociones (valores exactos): `feliz, dormida, sorprendida, revisando, preocupada, celebrando,
  editando, escuchando, triste`.
- 120 s sin actividad, sin revisión en curso y sin globo abierto → `dormida`.
- Contrato JSON: cada salida lleva `"version": 1`. Si la versión es otra, la app no la usa y pide
  `/aura:actualizar`.
- Videos válidos: extensiones `.mp4 .mov .m4v`, sin distinguir mayúsculas, igual que
  `videoqa/watcher.py`.
- Nunca se pasa un nombre de video por una shell: se usa `Process` con un array de argumentos.
- Código de salida de `videoqa estado --json`: 0 si todo bien; **4** si no hay configuración
  (stderr `SIN_CONFIG`). Un motor viejo sin el subcomando sale con 2 y `invalid choice`.
- Commits que terminan con `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

**Decisiones de este plan que precisan el spec:**
- En la fase 1, el clic abre un **globito de estado** (cola, en curso, últimos resultados con
  "Abrir"). La ventana de chat completa llega en la fase 2.
- El primer arranque comprueba el motor con `videoqa estado --json`, **no** con `videoqa doctor`,
  porque `doctor` gasta uso de Claude en cada arranque.
- El **bostezo antes de dormirse** (Parte 1 del spec) se aplaza a la fase 2. En la fase 1 se
  duerme sin bostezar: no afecta a nada más y evita otro estado transitorio.
- Las notificaciones de macOS se aplazan a la fase 2. En la fase 1, Aura es visible siempre, y
  cuando está silenciada se enterará al despertar porque el estado ya habrá cambiado.

## Review Focus

1. **La carpeta de Drive está en `~/Library/CloudStorage`** y ahí FSEvents puede no avisar. Aun
   así ella espera que Aura reaccione: el sondeo de 20 s lo cubre. Si un video llega y termina
   entre dos sondeos, igual tiene que salir la reacción final (test en Task B3).
2. **Nombres con tildes, eñes y espacios** ("promo otoño ñ.mp4") atraviesan el JSON, la frase y
   la revisión al soltarlos sin romperse ni quedar escapados (tests en A2, B1 y B5).
3. **El motor se cae a mitad de una revisión**: el marcador `en_curso.json` no puede dejar a
   Aura "revisando" para siempre (pid muerto → `null`, test en A1; `revisionSeDetuvo`, test en
   B3).
4. **Al abrir la app con resultados viejos** no debe haber una fiesta de celebraciones, y si
   terminan varios a la vez una revisión con errores no puede quedar tapada por una aprobada
   (tests en B3 y B4).
5. **Falta el motor o la configuración**: Aura se pone triste con el paso concreto, pero no
   repite el globito cada 20 s (tests en B4 y B5).

## Mapa de archivos

**`videoqa` (rama `feat/aura-estado`, sobre `main`):**
- Crear `videoqa/en_curso.py`: marcador de revisión en curso. Escribe y lee
  `~/.videoqa/en_curso.json`.
- Modificar `videoqa/pipeline.py`: `process_video` pasa a ser un envoltorio que marca y llama al
  cuerpo actual, renombrado `_process_video`.
- Crear `videoqa/estado.py`: la foto del estado (`foto()`, `leer_reporte()`,
  `automatico_encendido()`).
- Modificar `videoqa/cli.py`: subcomando `estado --json`.
- Crear `plugins/aura/skills/mascota/SKILL.md`: compila e instala Aura.app.
- Modificar `plugins/aura/skills/actualizar/SKILL.md`: recompila la app si está instalada.
- Modificar `plugins/aura/.claude-plugin/plugin.json` y `.claude-plugin/marketplace.json`:
  versión 1.6.0.
- Modificar `tests/unit/test_plugin_layout.py`: añadir `mascota` a `SIDE_EFFECT_SKILLS`.
- Tests: `tests/unit/test_en_curso.py` y `tests/unit/test_estado.py`.

**`aura-app` (rama `feat/fase1-mascota`):**
- `Package.swift`
- `Sources/AuraNucleo/`:
  - `Emocion.swift`
  - `EstadoMotor.swift`: contrato JSON y `ErrorMotor`.
  - `Frases.swift`
  - `EventoAura.swift`
  - `Diferencias.swift`
  - `EstadoAura.swift`
  - `MotorCLI.swift`: con `Ejecutor` y `EjecutorReal`.
  - `Utilidades.swift`: `AccionSoltar`, `Vigilancia.relevante`, `Mirada` y `Parpadeo`.
- `Sources/Aura/`:
  - `main.swift`
  - `Delegado.swift`
  - `Controlador.swift`
  - `Vigia.swift`
  - `MascotaPanel.swift`
  - `VistaPanel.swift`
  - `MascotaView.swift`: dibujo y caras.
  - `GloboEstado.swift`
  - `Icono.swift`
  - `Color+Hex.swift`
- `Tests/AuraNucleoTests/`: un archivo por unidad del núcleo, más `Fixtures.swift`.
- `Recursos/Info.plist`
- `scripts/empaquetar.sh`
- `README.md`: cómo compilar y probar.

---

## Parte A — `videoqa`

Prepara la rama una sola vez antes de A1:

```bash
cd ~/videoqa && git checkout main && git pull --ff-only && git checkout -b feat/aura-estado
uv run pytest tests/unit -q   # línea base: todo verde
```

### Task A1: Marcador de revisión en curso

**Files:**
- Create: `videoqa/en_curso.py`
- Modify: `videoqa/pipeline.py` (la línea `def process_video(` se renombra y se añade el envoltorio justo debajo de la clase `Result`)
- Test: `tests/unit/test_en_curso.py`

**Interfaces:**
- Produces: `en_curso.marcar(video: Path)` (context manager) y `en_curso.leer() -> dict | None`
  (`{"video": str, "desde": str ISO}`), que usa A2.

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/test_en_curso.py
import json
import os
from pathlib import Path

from videoqa import en_curso
from videoqa.config import videoqa_home


def test_marca_mientras_dura_y_la_borra_al_salir(tmp_path):
    video = tmp_path / "promo otoño ñ.mp4"
    with en_curso.marcar(video):
        dato = en_curso.leer()
        assert dato["video"] == "promo otoño ñ.mp4"
        assert dato["desde"][:4].isdigit()
    assert en_curso.leer() is None


def test_borra_la_marca_aunque_falle(tmp_path):
    try:
        with en_curso.marcar(tmp_path / "v.mp4"):
            raise RuntimeError("se cayó")
    except RuntimeError:
        pass
    assert not (videoqa_home() / "en_curso.json").exists()


def test_marca_de_proceso_muerto_no_cuenta():
    ruta = videoqa_home() / "en_curso.json"
    ruta.parent.mkdir(parents=True, exist_ok=True)
    ruta.write_text(json.dumps({"video": "v.mp4", "desde": "2026-10-20T10:00:00", "pid": 999999}))
    assert en_curso.leer() is None


def test_pid_invalido_o_json_roto_no_cuenta():
    ruta = videoqa_home() / "en_curso.json"
    ruta.parent.mkdir(parents=True, exist_ok=True)
    ruta.write_text(json.dumps({"video": "v.mp4", "desde": "x", "pid": -1}))
    assert en_curso.leer() is None
    ruta.write_text("{roto")
    assert en_curso.leer() is None


def test_process_video_deja_la_marca(tmp_path, monkeypatch):
    from videoqa import pipeline
    visto = {}

    def cuerpo(video, settings, rules, runner, sheet=None, modo="completo", veredicto_text=None):
        visto["durante"] = en_curso.leer()
        return pipeline.Result("approved", [], None)

    monkeypatch.setattr(pipeline, "_process_video", cuerpo)
    res = pipeline.process_video(tmp_path / "clip.mp4", None, {}, None)
    assert res.status == "approved"
    assert visto["durante"]["video"] == "clip.mp4"
    assert en_curso.leer() is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/videoqa && uv run pytest tests/unit/test_en_curso.py -q`
Expected: FAIL, `ImportError: cannot import name 'en_curso'`.

- [ ] **Step 3: Write minimal implementation**

```python
# videoqa/en_curso.py
"""Marca en disco qué video se está revisando, para que la app de Aura lo sepa.

La marca lleva el pid del proceso: si el motor se cae a mitad, la marca queda
huérfana y `leer()` la ignora en vez de dejar a Aura "revisando" para siempre.
"""
from __future__ import annotations

import json
import os
from contextlib import contextmanager
from datetime import datetime
from pathlib import Path
from typing import Iterator

from videoqa.config import videoqa_home

NOMBRE = "en_curso.json"


def _ruta() -> Path:
    return videoqa_home() / NOMBRE


def _vivo(pid: int) -> bool:
    if pid <= 0:  # os.kill(-1, 0) preguntaría por TODOS los procesos
        return False
    try:
        os.kill(pid, 0)
    except ProcessLookupError:
        return False
    except PermissionError:
        return True
    return True


@contextmanager
def marcar(video: Path) -> Iterator[None]:
    ruta = _ruta()
    ruta.parent.mkdir(parents=True, exist_ok=True)
    ruta.write_text(json.dumps({"video": Path(video).name,
                                "desde": datetime.now().isoformat(timespec="seconds"),
                                "pid": os.getpid()}, ensure_ascii=False), encoding="utf-8")
    try:
        yield
    finally:
        ruta.unlink(missing_ok=True)


def leer() -> dict | None:
    try:
        data = json.loads(_ruta().read_text(encoding="utf-8"))
    except (FileNotFoundError, json.JSONDecodeError, UnicodeDecodeError):
        return None
    if not isinstance(data, dict):
        return None
    try:
        pid = int(data.get("pid", -1))
    except (TypeError, ValueError):
        return None
    if not _vivo(pid) or "video" not in data or "desde" not in data:
        return None
    return {"video": str(data["video"]), "desde": str(data["desde"])}
```

En `videoqa/pipeline.py`, renombra la función existente
`def process_video(video: Path, settings: Settings, rules: dict, runner: Runner, sheet: SheetWriter | None = None,`
a `def _process_video(` (mismo cuerpo, sin tocar nada más) y añade **encima** de ella:

```python
def process_video(video: Path, settings: Settings, rules: dict, runner: Runner, sheet: SheetWriter | None = None,
                  modo: str = "completo", veredicto_text: str | None = None) -> Result:
    """Revisa un video. Mientras dura, deja `en_curso.json` para que la app de Aura lo vea."""
    with en_curso.marcar(video):
        return _process_video(video, settings, rules, runner, sheet=sheet, modo=modo, veredicto_text=veredicto_text)
```

y en los imports de `pipeline.py` añade `from videoqa import en_curso`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest tests/unit/test_en_curso.py tests/unit -q`
Expected: PASS. La suite entera sigue verde: los tests existentes usan `process_video` con la misma firma.

- [ ] **Step 5: Commit**

```bash
git add videoqa/en_curso.py videoqa/pipeline.py tests/unit/test_en_curso.py
git commit -m "feat: marca de revisión en curso para la app de Aura

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task A2: `videoqa estado --json`

**Files:**
- Create: `videoqa/estado.py`
- Modify: `videoqa/cli.py` (función `cmd_estado` junto a `cmd_hallazgos`, y el parser junto a `respaldo`)
- Test: `tests/unit/test_estado.py`

**Interfaces:**
- Consumes: `en_curso.leer()` (A1), `watcher.list_videos(entrada)`, `profiles.is_client_folder(name)`, `config.load_all_settings()`, y `Settings.entrada`, `.con_errores` y `.aprobado`.
- Produces: el JSON del contrato, que consume B1:

```json
{"version": 1, "drive_root": "/abs", "automatico": false,
 "en_curso": {"video": "a.mp4", "desde": "2026-10-20T10:00:00"} | null,
 "pendientes": [{"video": "a.mp4", "cliente": null}],
 "recientes": [{"video": "b.mp4", "cliente": "Sushi CD", "estado": "approved|rejected|error",
                "bloqueantes": 0, "advertencias": 1, "revisado": "2026-10-20T10:05:13",
                "carpeta": "/abs/03_Aprobado/Sushi CD/b"}]}
```

`recientes` va ordenado de más nuevo a más viejo, con un máximo de 10. `revisado` es el mtime de
`reporte.md` con precisión de segundos.

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/test_estado.py
import json
import os
import time

from videoqa import estado
from videoqa.cli import main


def _drive(tmp_path):
    drive = tmp_path / "drive"
    main(["init", "--drive-root", str(drive)])
    return drive


def _resultado(raiz, nombre, icono, label, bloq, adv, cliente=None, edad=0):
    carpeta = (raiz / cliente / nombre) if cliente else (raiz / nombre)
    carpeta.mkdir(parents=True)
    rep = carpeta / "reporte.md"
    rep.write_text(f"# {icono} {nombre}.mp4 — {label}\n"
                   f"Revisado: 2026-10-20 10:00 · Duración 0:35 · 1080×1920 · {bloq} · {adv}\n\n", encoding="utf-8")
    t = time.time() - edad
    os.utime(rep, (t, t))
    return carpeta


def test_foto_con_pendientes_resultados_y_clientes(tmp_path):
    drive = _drive(tmp_path)
    (drive / "01_Entrada" / "promo otoño ñ.mp4").write_bytes(b"x")
    (drive / "01_Entrada" / "Sushi CD").mkdir()
    (drive / "01_Entrada" / "Sushi CD" / "reel3.MOV").write_bytes(b"x")
    (drive / "01_Entrada" / "notas.txt").write_text("no es video")
    _resultado(drive / "02_Con_errores", "viejo", "🔴", "NO APROBADO", "2 bloqueantes", "1 advertencia", edad=100)
    _resultado(drive / "03_Aprobado", "nuevo", "🟢", "APROBADO", "0 bloqueantes", "0 advertencias", cliente="Sushi CD")
    _resultado(drive / "02_Con_errores", "pend", "⏸️", "PENDIENTE (sin criterio de Claude)", "1 bloqueante", "0 advertencias", edad=50)

    from videoqa.config import load_all_settings
    foto = estado.foto(load_all_settings(), automatico=True)

    assert foto["version"] == 1 and foto["automatico"] is True and foto["en_curso"] is None
    assert foto["drive_root"] == str(drive)
    assert sorted((p["video"], p["cliente"]) for p in foto["pendientes"]) == [("promo otoño ñ.mp4", None), ("reel3.MOV", "Sushi CD")]
    assert [r["video"] for r in foto["recientes"]] == ["nuevo.mp4", "pend.mp4", "viejo.mp4"]
    nuevo, pend, viejo = foto["recientes"]
    assert nuevo["estado"] == "approved" and nuevo["cliente"] == "Sushi CD"
    assert pend["estado"] == "error" and pend["bloqueantes"] == 1
    assert viejo["estado"] == "rejected" and viejo["bloqueantes"] == 2 and viejo["advertencias"] == 1
    assert viejo["carpeta"].endswith("02_Con_errores/viejo")
    assert len(viejo["revisado"]) == 19


def test_ignora_reportes_ilegibles_y_limita_a_diez(tmp_path):
    drive = _drive(tmp_path)
    roto = drive / "02_Con_errores" / "roto"
    roto.mkdir(parents=True)
    (roto / "reporte.md").write_text("cualquier cosa\n", encoding="utf-8")
    for i in range(12):
        _resultado(drive / "03_Aprobado", f"v{i:02d}", "🟢", "APROBADO", "0 bloqueantes", "0 advertencias", edad=i)
    from videoqa.config import load_all_settings
    foto = estado.foto(load_all_settings(), automatico=False)
    assert len(foto["recientes"]) == 10
    assert foto["recientes"][0]["video"] == "v00.mp4"


def test_automatico_lee_launchctl():
    class P:
        def __init__(self, code, out): self.returncode, self.stdout = code, out
    assert estado.automatico_encendido(run=lambda *a, **k: P(0, "state = running")) is True
    assert estado.automatico_encendido(run=lambda *a, **k: P(113, "")) is False
    def explota(*a, **k): raise OSError("no launchctl")
    assert estado.automatico_encendido(run=explota) is False


def test_cli_estado_json(tmp_path, monkeypatch, capsys):
    _drive(tmp_path)
    monkeypatch.setattr(estado, "automatico_encendido", lambda: False)
    capsys.readouterr()
    assert main(["estado", "--json"]) == 0
    datos = json.loads(capsys.readouterr().out)
    assert datos["version"] == 1 and datos["pendientes"] == []


def test_cli_estado_sin_config(tmp_path, monkeypatch, capsys):
    monkeypatch.setenv("VIDEOQA_CONFIG", str(tmp_path / "no_existe.yaml"))
    assert main(["estado", "--json"]) == 4
    salida = capsys.readouterr()
    assert salida.out == "" and "SIN_CONFIG" in salida.err
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/unit/test_estado.py -q`
Expected: FAIL, `ImportError: cannot import name 'estado'`.

- [ ] **Step 3: Write minimal implementation**

```python
# videoqa/estado.py
"""Foto del estado de la revisión, en JSON, para la app de Aura (`videoqa estado --json`).

La app compara una foto con la anterior para saber qué pasó (llegó un video,
empezó o terminó una revisión). `version` sube solo si el formato cambia de forma
incompatible.
"""
from __future__ import annotations

import os
import re
import subprocess
from datetime import datetime
from pathlib import Path
from typing import Callable

from videoqa import en_curso
from videoqa.config import Settings
from videoqa.profiles import is_client_folder
from videoqa.watcher import list_videos

VERSION = 1
MAX_RECIENTES = 10
_TITULO = re.compile(r"^# (🟢|🔴|⏸️) (.+?) — ")
_BLOQ = re.compile(r"(\d+) bloqueantes?\b")
_ADV = re.compile(r"(\d+) advertencias?\b")
_ESTADO = {"🟢": "approved", "🔴": "rejected", "⏸️": "error"}


def leer_reporte(carpeta: Path, cliente: str | None) -> dict | None:
    reporte = carpeta / "reporte.md"
    try:
        lineas = reporte.read_text(encoding="utf-8").splitlines()[:2]
        mtime = reporte.stat().st_mtime
    except (FileNotFoundError, UnicodeDecodeError):
        return None
    m = _TITULO.match(lineas[0]) if lineas else None
    if not m:
        return None
    segunda = lineas[1] if len(lineas) > 1 else ""
    b, a = _BLOQ.search(segunda), _ADV.search(segunda)
    return {"video": m.group(2), "cliente": cliente, "estado": _ESTADO[m.group(1)],
            "bloqueantes": int(b.group(1)) if b else 0, "advertencias": int(a.group(1)) if a else 0,
            "revisado": datetime.fromtimestamp(mtime).isoformat(timespec="seconds"),
            "carpeta": str(carpeta)}


def _carpetas_resultado(raiz: Path) -> list[tuple[Path, str | None]]:
    if not raiz.is_dir():
        return []
    out: list[tuple[Path, str | None]] = []
    for d in raiz.iterdir():
        if not d.is_dir() or d.name.startswith("."):
            continue
        if (d / "reporte.md").exists():
            out.append((d, None))
        elif is_client_folder(d.name):
            out += [(c, d.name) for c in d.iterdir() if c.is_dir() and (c / "reporte.md").exists()]
    return out


def automatico_encendido(run: Callable = subprocess.run) -> bool:
    try:
        p = run(["launchctl", "print", f"gui/{os.getuid()}/com.videoqa.watcher"],
                capture_output=True, text=True, timeout=5)
    except (OSError, subprocess.TimeoutExpired):
        return False
    return p.returncode == 0 and "state = running" in (p.stdout or "")


def foto(todas: list[Settings], automatico: bool) -> dict:
    pendientes: list[dict] = []
    recientes: list[dict] = []
    for s in todas:
        for v in list_videos(s.entrada):
            pendientes.append({"video": v.name, "cliente": v.parent.name if v.parent != s.entrada else None})
        for raiz in (s.con_errores, s.aprobado):
            for carpeta, cliente in _carpetas_resultado(raiz):
                r = leer_reporte(carpeta, cliente)
                if r:
                    recientes.append(r)
    recientes.sort(key=lambda r: r["revisado"], reverse=True)
    return {"version": VERSION, "drive_root": str(todas[0].drive_root), "automatico": automatico,
            "en_curso": en_curso.leer(), "pendientes": pendientes, "recientes": recientes[:MAX_RECIENTES]}
```

En `videoqa/cli.py`, junto a `cmd_hallazgos`:

```python
def cmd_estado(args) -> int:
    """Foto del estado en JSON para la app de Aura. Sale con 4 si no hay configuración."""
    from videoqa import estado
    try:
        todas = load_all_settings()
    except FileNotFoundError:
        print("SIN_CONFIG", file=sys.stderr)
        return 4
    print(json.dumps(estado.foto(todas, estado.automatico_encendido()), ensure_ascii=False))
    return 0
```

Y en el parser, justo antes de `p = sub.add_parser("doctor", ...)`:

```python
    p = sub.add_parser("estado", help="estado de la cola en JSON (para la app de Aura)")
    p.add_argument("--json", action="store_true", required=True)
    p.set_defaults(fn=cmd_estado)
```

Comprueba que `cli.py` ya importa `json`, `sys` y `load_all_settings`. Si falta alguno,
añádelo. El log va a stderr (`setup_logging`), así que stdout queda limpio y es solo el JSON.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest tests/unit/test_estado.py tests/unit -q`
Expected: PASS.

Prueba también con tus datos reales:

```bash
uv run videoqa estado --json | python3 -m json.tool | head -30
```

Expected: JSON con tus carpetas de `~/Desktop/videos`.

- [ ] **Step 5: Commit**

```bash
git add videoqa/estado.py videoqa/cli.py tests/unit/test_estado.py
git commit -m "feat: videoqa estado --json para la app de Aura

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task A3: Skill `/aura:mascota`, recompilar al actualizar y versión 1.6.0

**Files:**
- Create: `plugins/aura/skills/mascota/SKILL.md`
- Modify: `plugins/aura/skills/actualizar/SKILL.md` (nuevo paso entre el 5 y el 6)
- Modify: `plugins/aura/.claude-plugin/plugin.json` y `.claude-plugin/marketplace.json` (`"version": "1.5.0"` → `"1.6.0"`)
- Test: `tests/unit/test_plugin_layout.py` (`SIDE_EFFECT_SKILLS`)

**Interfaces:**
- Consumes: el script `~/aura-app/scripts/empaquetar.sh` (B8), que deja `~/aura-app/build/Aura.app`.

> **Nota de acceso:** `hongs05/aura-app` es privado. En la Mac de ella, `git clone` pedirá
> credenciales de GitHub. Antes de entregar hay que hacer el repo público (no tiene secretos) o
> iniciar sesión con `gh` en su Mac. Esta skill usa la URL HTTPS pública y, si el clon falla por
> permisos, lo explica.

- [ ] **Step 1: Write the failing test**

En `tests/unit/test_plugin_layout.py` cambia:

```python
SIDE_EFFECT_SKILLS = {"instalar", "ajustar", "activar-automatico", "actualizar", "respaldo", "mascota"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/unit/test_plugin_layout.py -q`
Expected: FAIL en `test_hay_skills`, porque `mascota` no existe.

- [ ] **Step 3: Write the skill and bump the version**

```markdown
---
description: Instala o pone al día a Aura, la mascota de escritorio que te acompaña mientras se revisan los videos. Úsalo con "instala la mascota" o "quiero a Aura en el escritorio".
disable-model-invocation: true
allowed-tools: Bash(ls:*) Bash(swift --version) Bash(xcode-select --install) Bash(git clone:*) Bash(git -C ~/aura-app:*) Bash(bash ~/aura-app/scripts/empaquetar.sh) Bash(pkill -x Aura) Bash(ditto:*) Bash(open:*) Read
---

# Instalar a Aura en el escritorio

Habla en español informal (tú), sin jerga. Avisa al empezar: "Voy a traer a Aura a tu escritorio.
Tarda unos minutos la primera vez."

## Paso 1 — Comprobar el motor

`ls ~/videoqa/pyproject.toml`. Si no existe: "Primero hay que instalar la revisión: escribe
`/aura:instalar`." Y termina.

## Paso 2 — Herramientas para compilar

`swift --version`. Si falla, ejecuta `xcode-select --install` y dile: "Se abrió una ventana para
instalar las herramientas de Apple. Dale a Instalar, espera a que termine y vuelve a escribir
`/aura:mascota`." Y termina.

## Paso 3 — Traer el código

- Si `~/aura-app` no existe: `git clone https://github.com/hongs05/aura-app.git ~/aura-app`.
- Si existe: `git -C ~/aura-app pull --ff-only`.

Si el clon falla con "Repository not found" o pide usuario: "No tengo permiso para descargar a
Aura. Pídele a quien te pasó la herramienta que te dé acceso." Y termina.

## Paso 4 — Compilar e instalar

    bash ~/aura-app/scripts/empaquetar.sh
    pkill -x Aura
    ditto ~/aura-app/build/Aura.app ~/Applications/Aura.app
    open ~/Applications/Aura.app

`pkill` puede decir que no había nada abierto: no es un error, no lo menciones. Si
`empaquetar.sh` falla, no muestres el error entero. Di "No pude prepararla, inténtalo otra vez
en un rato" y termina.

## Paso 5 — Contarlo

> 🍑 ¡Aura ya está en tu escritorio! Arrástrala a la esquina que quieras.
> - Se duerme cuando no hay videos y se despierta cuando llega uno.
> - Hazle clic para ver cómo va la cola.
> - Suéltale un video encima y lo reviso.
> - Clic derecho → "Abrir al iniciar sesión" para que esté siempre contigo.
```

En `plugins/aura/skills/actualizar/SKILL.md`:
- Añade a `allowed-tools`: ` Bash(ls:*) Bash(git -C ~/aura-app:*) Bash(bash ~/aura-app/scripts/empaquetar.sh) Bash(pkill -x Aura) Bash(ditto:*) Bash(open:*)`.
- Inserta antes de "## Paso 6 — Contar qué cambió":

```markdown
## Paso 5b — Poner al día a la mascota (si está instalada)

`ls ~/Applications/Aura.app`. Si no existe, sáltalo sin mencionarlo. Si existe:

    git -C ~/aura-app pull --ff-only
    bash ~/aura-app/scripts/empaquetar.sh
    pkill -x Aura
    ditto ~/aura-app/build/Aura.app ~/Applications/Aura.app
    open ~/Applications/Aura.app

Si algo falla, di "Aura sigue con la versión anterior; no pasa nada" y sigue con el paso 6.
```

Cambia `"version": "1.5.0"` por `"version": "1.6.0"` en los dos manifiestos.

- [ ] **Step 4: Run tests**

Run: `uv run pytest tests/unit -q`
Expected: PASS, incluidos los checks de frontmatter de `mascota`.

- [ ] **Step 5: Commit**

```bash
git add plugins .claude-plugin tests/unit/test_plugin_layout.py
git commit -m "feat: /aura:mascota instala la app de escritorio (aura 1.6.0)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

No hagas merge a `main` todavía: la rama se integra al final, después de la verificación
manual de B8.

---

## Parte B — `aura-app`

Prepara la rama una sola vez antes de B1:

```bash
cd ~/aura-app && git checkout main && git pull --ff-only && git checkout -b feat/fase1-mascota
```

### Task B1: Paquete, emociones y contrato JSON

**Files:**
- Create: `Package.swift`, `Sources/AuraNucleo/Emocion.swift`, `Sources/AuraNucleo/EstadoMotor.swift`, `Sources/Aura/main.swift` (provisional, una línea)
- Test: `Tests/AuraNucleoTests/Fixtures.swift`, `Tests/AuraNucleoTests/EstadoMotorTests.swift`

**Interfaces:**
- Produces: `Emocion`; `EstadoMotor` con `version`, `driveRoot`, `automatico`, `enCurso: EnCurso?`, `pendientes: [Pendiente]` y `recientes: [ResultadoVideo]`; `EstadoMotor.decodificar(_ data: Data) throws -> EstadoMotor` (lanza `ErrorMotor.versionDesconocida`); `ErrorMotor` (`sinMotor`, `sinConfigurar`, `versionDesconocida`, `fallo(String)`); `Pendiente.clave` y `ResultadoVideo.clave`; `ResultadoVideo.Resultado` (`aprobado`, `conErrores`, `pendiente`); `nombreVisible(_ video: String) -> String`.

- [ ] **Step 1: Write the failing test**

```swift
// Package.swift
// swift-tools-version:5.10
import PackageDescription

let package = Package(
    name: "Aura",
    platforms: [.macOS(.v14)],
    targets: [
        .target(name: "AuraNucleo"),
        .executableTarget(name: "Aura", dependencies: ["AuraNucleo"]),
        .testTarget(name: "AuraNucleoTests", dependencies: ["AuraNucleo"]),
    ]
)
```

```swift
// Sources/Aura/main.swift  (provisional hasta B7)
print("Aura")
```

```swift
// Tests/AuraNucleoTests/Fixtures.swift
import Foundation
@testable import AuraNucleo

enum Fixtures {
    static let estadoJSON = """
    {"version": 1, "drive_root": "/Users/x/Drive/Videos", "automatico": true,
     "en_curso": {"video": "promo otoño ñ.mp4", "desde": "2026-10-20T10:00:00"},
     "pendientes": [{"video": "promo otoño ñ.mp4", "cliente": null},
                    {"video": "reel3.MOV", "cliente": "Sushi CD"}],
     "recientes": [{"video": "nuevo.mp4", "cliente": "Sushi CD", "estado": "approved",
                    "bloqueantes": 0, "advertencias": 0, "revisado": "2026-10-20T10:05:13",
                    "carpeta": "/Users/x/Drive/Videos/03_Aprobado/Sushi CD/nuevo"},
                   {"video": "viejo.mp4", "cliente": null, "estado": "rejected",
                    "bloqueantes": 2, "advertencias": 1, "revisado": "2026-10-20T09:00:00",
                    "carpeta": "/Users/x/Drive/Videos/02_Con_errores/viejo"}]}
    """.data(using: .utf8)!

    static func resultado(_ video: String, _ estado: ResultadoVideo.Resultado, bloqueantes: Int = 0,
                          advertencias: Int = 0, revisado: String = "2026-10-20T10:00:00") -> ResultadoVideo {
        ResultadoVideo(video: video, cliente: nil, estado: estado, bloqueantes: bloqueantes,
                       advertencias: advertencias, revisado: revisado, carpeta: "/d/" + video)
    }

    static func motor(pendientes: [String] = [], enCurso: String? = nil, recientes: [ResultadoVideo] = [],
                      automatico: Bool = false) -> EstadoMotor {
        EstadoMotor(version: 1, driveRoot: "/d", automatico: automatico,
                    enCurso: enCurso.map { EnCurso(video: $0, desde: "2026-10-20T10:00:00") },
                    pendientes: pendientes.map { Pendiente(video: $0, cliente: nil) }, recientes: recientes)
    }
}
```

```swift
// Tests/AuraNucleoTests/EstadoMotorTests.swift
import Foundation
import Testing
@testable import AuraNucleo

@Test func decodificaElContratoConTildes() throws {
    let e = try EstadoMotor.decodificar(Fixtures.estadoJSON)
    #expect(e.driveRoot == "/Users/x/Drive/Videos")
    #expect(e.automatico)
    #expect(e.enCurso?.video == "promo otoño ñ.mp4")
    #expect(e.pendientes.map(\.clave) == ["/promo otoño ñ.mp4", "Sushi CD/reel3.MOV"])
    #expect(e.recientes[0].estado == .aprobado)
    #expect(e.recientes[1].estado == .conErrores)
    #expect(e.recientes[1].bloqueantes == 2)
}

@Test func versionDistintaSeRechaza() {
    let json = #"{"version": 2, "otra": "cosa"}"#.data(using: .utf8)!
    #expect(throws: ErrorMotor.versionDesconocida) { try EstadoMotor.decodificar(json) }
}

@Test func estadoDesconocidoEsUnErrorDeDecodificacion() throws {
    var texto = String(decoding: Fixtures.estadoJSON, as: UTF8.self)
    texto = texto.replacingOccurrences(of: "\"approved\"", with: "\"otro\"")
    #expect(throws: (any Error).self) { try EstadoMotor.decodificar(Data(texto.utf8)) }
}

@Test func nombreVisibleQuitaLaExtension() {
    #expect(nombreVisible("promo otoño ñ.mp4") == "promo otoño ñ")
    #expect(nombreVisible("sin_extension") == "sin_extension")
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/aura-app && swift test`
Expected: FAIL en la compilación, `cannot find 'EstadoMotor' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Sources/AuraNucleo/Emocion.swift
public enum Emocion: String, CaseIterable, Codable, Sendable {
    case feliz, dormida, sorprendida, revisando, preocupada, celebrando, editando, escuchando, triste
}
```

```swift
// Sources/AuraNucleo/EstadoMotor.swift
import Foundation

/// Lo que devuelve `videoqa estado --json` (contrato versión 1).
public struct EstadoMotor: Decodable, Equatable, Sendable {
    public static let versionSoportada = 1
    public let version: Int
    public let driveRoot: String
    public let automatico: Bool
    public let enCurso: EnCurso?
    public let pendientes: [Pendiente]
    public let recientes: [ResultadoVideo]

    enum CodingKeys: String, CodingKey {
        case version, automatico, pendientes, recientes
        case driveRoot = "drive_root"
        case enCurso = "en_curso"
    }

    public init(version: Int, driveRoot: String, automatico: Bool, enCurso: EnCurso?,
                pendientes: [Pendiente], recientes: [ResultadoVideo]) {
        self.version = version; self.driveRoot = driveRoot; self.automatico = automatico
        self.enCurso = enCurso; self.pendientes = pendientes; self.recientes = recientes
    }

    public static func decodificar(_ data: Data) throws -> EstadoMotor {
        struct SoloVersion: Decodable { let version: Int }
        let v = try JSONDecoder().decode(SoloVersion.self, from: data)
        guard v.version == versionSoportada else { throw ErrorMotor.versionDesconocida }
        return try JSONDecoder().decode(EstadoMotor.self, from: data)
    }
}

public struct EnCurso: Decodable, Equatable, Sendable {
    public let video: String
    public let desde: String
    public init(video: String, desde: String) { self.video = video; self.desde = desde }
}

public struct Pendiente: Decodable, Equatable, Sendable {
    public let video: String
    public let cliente: String?
    public init(video: String, cliente: String?) { self.video = video; self.cliente = cliente }
    public var clave: String { (cliente ?? "") + "/" + video }
}

public struct ResultadoVideo: Decodable, Equatable, Sendable {
    public enum Resultado: String, Decodable, Sendable {
        case aprobado = "approved", conErrores = "rejected", pendiente = "error"
    }
    public let video: String
    public let cliente: String?
    public let estado: Resultado
    public let bloqueantes: Int
    public let advertencias: Int
    public let revisado: String
    public let carpeta: String

    public init(video: String, cliente: String?, estado: Resultado, bloqueantes: Int, advertencias: Int,
                revisado: String, carpeta: String) {
        self.video = video; self.cliente = cliente; self.estado = estado; self.bloqueantes = bloqueantes
        self.advertencias = advertencias; self.revisado = revisado; self.carpeta = carpeta
    }

    /// Identifica UNA revisión: el mismo video revisado otra vez tiene otro `revisado`.
    public var clave: String { carpeta + "|" + revisado }
    public var cositas: Int { bloqueantes + advertencias }
}

public enum ErrorMotor: Error, Equatable, Sendable {
    case sinMotor, sinConfigurar, versionDesconocida
    case fallo(String)
}

public func nombreVisible(_ video: String) -> String {
    (video as NSString).deletingPathExtension
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add Package.swift Sources Tests
git commit -m "feat: paquete Swift, emociones y contrato de videoqa estado

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task B2: Libro de frases

**Files:**
- Create: `Sources/AuraNucleo/Frases.swift`
- Test: `Tests/AuraNucleoTests/FrasesTests.swift`

**Interfaces:**
- Produces: `Situacion` (enum `CaseIterable`); `Frases(aleatorio: @escaping @Sendable (Int) -> Int = { Int.random(in: 0..<$0) })`; `mutating func decir(_ s: Situacion, _ valores: [String: String] = [:]) -> String`; `func cositas(_ n: Int) -> String`.

- [ ] **Step 1: Write the failing test**

```swift
// Tests/AuraNucleoTests/FrasesTests.swift
import Testing
@testable import AuraNucleo

@Test func todasLasSituacionesTienenAlMenosDosVariantes() {
    for s in Situacion.allCases {
        #expect((Frases.textos[s]?.count ?? 0) >= 2, "faltan variantes de \(s)")
    }
}

@Test func noRepiteLaMismaVarianteSeguida() {
    var f = Frases(aleatorio: { _ in 0 })
    let a = f.decir(.saludo)
    let b = f.decir(.saludo)
    #expect(a != b)
}

@Test func rellenaLosValoresSinDejarLlaves() {
    var f = Frases(aleatorio: { _ in 0 })
    for s in Situacion.allCases {
        let t = f.decir(s, ["video": "promo otoño", "cositas": "3 cositas", "k": "3", "m": "1"])
        #expect(!t.contains("{"), "\(s) dejó una llave: \(t)")
    }
}

@Test func cositasEnSingularYPlural() {
    #expect(cositas(1) == "1 cosita")
    #expect(cositas(3) == "3 cositas")
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test`
Expected: FAIL, `cannot find 'Frases' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Sources/AuraNucleo/Frases.swift
public enum Situacion: String, CaseIterable, Sendable {
    case saludo, llego, revisando, conErrores, variosConErrores, aprobado, variosAprobados
    case pendiente, seDetuvo, noEsVideo, sinMotor, sinConfigurar, versionDesconocida, fallo, despierta
}

public func cositas(_ n: Int) -> String { n == 1 ? "1 cosita" : "\(n) cositas" }

/// Lo que dice Aura. Escrito a mano, varias variantes por situación, nunca la misma dos veces seguidas.
public struct Frases: Sendable {
    static let textos: [Situacion: [String]] = [
        .saludo: ["Hola 🍑 ¿Revisamos algo juntas?", "¡Aquí estoy! Avísame cuando subas algo.", "Hola, hola ✨"],
        .llego: ["¡Oh! Llegó «{video}».", "¡Uy, un video nuevo! «{video}»", "«{video}» acaba de llegar 👀"],
        .revisando: ["Mirando «{video}» segundo a segundo…", "Revisando «{video}»… un momentito.", "Estoy con «{video}» 🔍"],
        .conErrores: ["Encontré {cositas} en «{video}» 🥺", "«{video}» tiene {cositas} para arreglar. ¿Las vemos?", "Ay… {cositas} en «{video}». Nada grave, ¡tú puedes!"],
        .variosConErrores: ["Revisé {k} videos y en {m} encontré cositas 🥺", "Listos {k} videos; en {m} hay algo que retocar."],
        .aprobado: ["¡«{video}» está listo para publicar! ✨", "¡Perfecto! «{video}» quedó aprobado 🍑", "¡Sí! «{video}» sin problemas."],
        .variosAprobados: ["¡Los {k} videos están listos! ✨", "¡{k} aprobados de una! 🍑"],
        .pendiente: ["«{video}» quedó pendiente: me faltó el criterio de Claude.", "No pude terminar «{video}». Quedó pendiente."],
        .seDetuvo: ["Se me cortó la revisión de «{video}» 🥺", "«{video}» se quedó a medias. Lo intentamos otra vez."],
        .noEsVideo: ["Eso no es un video 🥺", "Mmm, solo sé revisar videos (mp4, mov)."],
        .sinMotor: ["Me falta el motor 🥺 Escribe /aura:instalar en Claude.", "No encuentro la revisión. Escribe /aura:instalar en Claude."],
        .sinConfigurar: ["Falta elegir la carpeta. Escribe /aura:instalar en Claude.", "Todavía no sé dónde están tus videos. Escribe /aura:instalar en Claude."],
        .versionDesconocida: ["Hay que ponerme al día: escribe /aura:actualizar en Claude.", "Estoy desactualizada 🥺 Escribe /aura:actualizar en Claude."],
        .fallo: ["Algo no salió bien… lo intento en un ratito.", "Tropecé con algo 🥺 Lo vuelvo a intentar pronto."],
        .despierta: ["¡Ya desperté! 🍑", "Mmm… ¿me perdí de algo?"],
    ]

    private var ultima: [Situacion: Int] = [:]
    private let aleatorio: @Sendable (Int) -> Int

    public init(aleatorio: @escaping @Sendable (Int) -> Int = { Int.random(in: 0..<$0) }) {
        self.aleatorio = aleatorio
    }

    public mutating func decir(_ s: Situacion, _ valores: [String: String] = [:]) -> String {
        let opciones = Self.textos[s] ?? [""]
        var i = aleatorio(opciones.count) % opciones.count
        if opciones.count > 1, i == ultima[s] { i = (i + 1) % opciones.count }
        ultima[s] = i
        return valores.reduce(opciones[i]) { $0.replacingOccurrences(of: "{\($1.key)}", with: $1.value) }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/AuraNucleo/Frases.swift Tests/AuraNucleoTests/FrasesTests.swift
git commit -m "feat: libro de frases de Aura

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task B3: Eventos y diferencias entre fotos

**Files:**
- Create: `Sources/AuraNucleo/EventoAura.swift`, `Sources/AuraNucleo/Diferencias.swift`
- Test: `Tests/AuraNucleoTests/DiferenciasTests.swift`

**Interfaces:**
- Consumes: `EstadoMotor`, `Pendiente.clave`, `ResultadoVideo.clave` (B1).
- Produces: `EventoAura` (`videoLlego(String)`, `revisionEmpezo(String)`, `revisionesTerminaron([ResultadoVideo])`, `revisionSeDetuvo(String)`, `noEsVideo`, `globoAbierto`, `globoCerrado`, `errorMotor(ErrorMotor)`, `motorBien`, `silenciar(hasta: Date)`); `Diferencias.eventos(antes: EstadoMotor?, despues: EstadoMotor) -> [EventoAura]`. El orden de los eventos es: llegadas, terminadas o detenida, y por último la revisión que empieza.

- [ ] **Step 1: Write the failing test**

```swift
// Tests/AuraNucleoTests/DiferenciasTests.swift
import Testing
@testable import AuraNucleo

@Test func primeraFotoNoCelebraLoViejo() {
    let foto = Fixtures.motor(pendientes: ["a.mp4"], recientes: [Fixtures.resultado("v.mp4", .aprobado)])
    #expect(Diferencias.eventos(antes: nil, despues: foto) == [])
}

@Test func primeraFotoConRevisionEnCursoLaSigue() {
    #expect(Diferencias.eventos(antes: nil, despues: Fixtures.motor(enCurso: "a.mp4")) == [.revisionEmpezo("a.mp4")])
}

@Test func llegaUnVideoConTildes() {
    let antes = Fixtures.motor()
    let despues = Fixtures.motor(pendientes: ["promo otoño ñ.mp4"])
    #expect(Diferencias.eventos(antes: antes, despues: despues) == [.videoLlego("promo otoño ñ.mp4")])
}

@Test func cicloCompletoEntreDosSondeosIgualReacciona() {
    let r = Fixtures.resultado("rapido.mp4", .conErrores, bloqueantes: 1)
    let ev = Diferencias.eventos(antes: Fixtures.motor(), despues: Fixtures.motor(recientes: [r]))
    #expect(ev == [.revisionesTerminaron([r])])
}

@Test func variasTerminadasVanEnUnSoloEvento() {
    let viejo = Fixtures.resultado("viejo.mp4", .aprobado, revisado: "2026-10-20T08:00:00")
    let a = Fixtures.resultado("a.mp4", .aprobado, revisado: "2026-10-20T10:01:00")
    let b = Fixtures.resultado("b.mp4", .conErrores, bloqueantes: 2, revisado: "2026-10-20T10:02:00")
    let ev = Diferencias.eventos(antes: Fixtures.motor(recientes: [viejo]),
                                 despues: Fixtures.motor(recientes: [b, a, viejo]))
    #expect(ev == [.revisionesTerminaron([b, a])])
}

@Test func revisionQueDesapareceSinResultadoSeDetuvo() {
    let ev = Diferencias.eventos(antes: Fixtures.motor(pendientes: ["a.mp4"], enCurso: "a.mp4"),
                                 despues: Fixtures.motor(pendientes: ["a.mp4"]))
    #expect(ev == [.revisionSeDetuvo("a.mp4")])
}

@Test func terminaUnoYEmpiezaOtroEnOrden() {
    let ra = Fixtures.resultado("a.mp4", .aprobado)
    let ev = Diferencias.eventos(antes: Fixtures.motor(pendientes: ["a.mp4", "b.mp4"], enCurso: "a.mp4"),
                                 despues: Fixtures.motor(pendientes: ["b.mp4"], enCurso: "b.mp4", recientes: [ra]))
    #expect(ev == [.revisionesTerminaron([ra]), .revisionEmpezo("b.mp4")])
}

@Test func elMismoVideoRevisadoOtraVezEsNuevo() {
    let v1 = Fixtures.resultado("a.mp4", .conErrores, bloqueantes: 1, revisado: "2026-10-20T09:00:00")
    let v2 = Fixtures.resultado("a.mp4", .aprobado, revisado: "2026-10-20T11:00:00")
    #expect(Diferencias.eventos(antes: Fixtures.motor(recientes: [v1]), despues: Fixtures.motor(recientes: [v2]))
            == [.revisionesTerminaron([v2])])
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test`
Expected: FAIL, `cannot find 'Diferencias' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Sources/AuraNucleo/EventoAura.swift
import Foundation

public enum EventoAura: Equatable, Sendable {
    case videoLlego(String)
    case revisionEmpezo(String)
    case revisionesTerminaron([ResultadoVideo])
    case revisionSeDetuvo(String)
    case noEsVideo
    case globoAbierto
    case globoCerrado
    case errorMotor(ErrorMotor)
    case motorBien
    case silenciar(hasta: Date)
}
```

```swift
// Sources/AuraNucleo/Diferencias.swift
/// Qué pasó entre dos fotos de `videoqa estado --json`.
public enum Diferencias {
    public static func eventos(antes: EstadoMotor?, despues: EstadoMotor) -> [EventoAura] {
        guard let antes else {
            // Primera foto: solo seguimos una revisión ya en marcha; lo viejo no se celebra.
            return despues.enCurso.map { [.revisionEmpezo($0.video)] } ?? []
        }
        var ev: [EventoAura] = []
        let previos = Set(antes.pendientes.map(\.clave))
        ev += despues.pendientes.filter { !previos.contains($0.clave) }.map { .videoLlego($0.video) }

        let vistos = Set(antes.recientes.map(\.clave))
        let nuevos = despues.recientes.filter { !vistos.contains($0.clave) }
        if !nuevos.isEmpty {
            ev.append(.revisionesTerminaron(nuevos))
        } else if let c = antes.enCurso, despues.enCurso == nil {
            ev.append(.revisionSeDetuvo(c.video))
        }

        if let c = despues.enCurso, c != antes.enCurso {
            ev.append(.revisionEmpezo(c.video))
        }
        return ev
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/AuraNucleo/EventoAura.swift Sources/AuraNucleo/Diferencias.swift Tests/AuraNucleoTests/DiferenciasTests.swift
git commit -m "feat: eventos de Aura a partir de dos fotos del motor

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task B4: Máquina de estados `EstadoAura`

**Files:**
- Create: `Sources/AuraNucleo/EstadoAura.swift`
- Test: `Tests/AuraNucleoTests/EstadoAuraTests.swift`

**Interfaces:**
- Consumes: `Emocion`, `EventoAura`, `ResultadoVideo`, `ErrorMotor`, `nombreVisible` (B1, B3), y `Frases`, `Situacion`, `cositas` (B2).
- Produces: `EstadoAura(ahora: Date, frases: Frases = Frases())`. Propiedades `emocion: Emocion`, `frase: String?`, `fraseID: Int` (sube con cada frase nueva), `revisando: Bool` y `silenciada: Bool`. Métodos `mutating recibir(_ e: EventoAura, ahora: Date)` y `mutating tick(ahora: Date)`. Constante `EstadoAura.segundosParaDormir = 120`.

- [ ] **Step 1: Write the failing test**

```swift
// Tests/AuraNucleoTests/EstadoAuraTests.swift
import Foundation
import Testing
@testable import AuraNucleo

private let t0 = Date(timeIntervalSince1970: 1_000_000)
private func at(_ s: TimeInterval) -> Date { t0.addingTimeInterval(s) }
private func nueva() -> EstadoAura { EstadoAura(ahora: t0, frases: Frases(aleatorio: { _ in 0 })) }

@Test func empiezaFelizYSaluda() {
    let a = nueva()
    #expect(a.emocion == .feliz)
    #expect(a.frase != nil && a.fraseID == 1)
}

@Test func seDuermeALosDosMinutosYNoAntes() {
    var a = nueva()
    a.tick(ahora: at(119)); #expect(a.emocion == .feliz)
    a.tick(ahora: at(120)); #expect(a.emocion == .dormida)
    #expect(a.frase == nil)
}

@Test func noSeDuermeMientrasRevisaNiConElGloboAbierto() {
    var a = nueva()
    a.recibir(.revisionEmpezo("a.mp4"), ahora: at(1))
    a.tick(ahora: at(1000)); #expect(a.emocion == .revisando)
    var b = nueva()
    b.recibir(.globoAbierto, ahora: at(1))
    b.tick(ahora: at(1000)); #expect(b.emocion == .escuchando)
}

@Test func llegaUnVideoSeSorprendeYLuegoRevisa() {
    var a = nueva()
    a.tick(ahora: at(200)); #expect(a.emocion == .dormida)
    a.recibir(.videoLlego("promo otoño ñ.mp4"), ahora: at(201))
    #expect(a.emocion == .sorprendida)
    #expect(a.frase?.contains("promo otoño ñ") == true)
    #expect(a.frase?.contains(".mp4") == false)
    a.recibir(.revisionEmpezo("promo otoño ñ.mp4"), ahora: at(202))
    #expect(a.emocion == .sorprendida)
    a.tick(ahora: at(203.1))
    #expect(a.emocion == .revisando)
    #expect(a.frase?.contains("promo otoño ñ") == true)
}

@Test func unRechazadoNoQuedaTapadoPorUnAprobado() {
    var a = nueva()
    a.recibir(.revisionesTerminaron([Fixtures.resultado("a.mp4", .aprobado),
                                     Fixtures.resultado("b.mp4", .conErrores, bloqueantes: 2)]), ahora: at(5))
    #expect(a.emocion == .preocupada)
    #expect(a.frase?.contains("2") == true && a.frase?.contains("1") == true)
    a.tick(ahora: at(13)); #expect(a.emocion == .feliz)
}

@Test func unAprobadoCelebraSeisSegundos() {
    var a = nueva()
    a.recibir(.revisionEmpezo("a.mp4"), ahora: at(1))
    a.recibir(.revisionesTerminaron([Fixtures.resultado("a.mp4", .aprobado)]), ahora: at(10))
    #expect(a.emocion == .celebrando && !a.revisando)
    a.tick(ahora: at(15.9)); #expect(a.emocion == .celebrando)
    a.tick(ahora: at(16)); #expect(a.emocion == .feliz)
}

@Test func unaCositaEnSingular() {
    var a = nueva()
    a.recibir(.revisionesTerminaron([Fixtures.resultado("a.mp4", .conErrores, bloqueantes: 1)]), ahora: at(1))
    #expect(a.frase?.contains("1 cosita ") == true || a.frase?.contains("1 cosita") == true)
    #expect(a.frase?.contains("1 cositas") == false)
}

@Test func elMismoErrorDelMotorNoSeRepite() {
    var a = nueva()
    a.recibir(.errorMotor(.sinMotor), ahora: at(1))
    let id = a.fraseID
    #expect(a.emocion == .triste)
    a.recibir(.errorMotor(.sinMotor), ahora: at(21))
    #expect(a.fraseID == id)
    a.tick(ahora: at(500)); #expect(a.emocion == .triste)
    a.recibir(.motorBien, ahora: at(501))
    #expect(a.emocion == .feliz)
}

@Test func silenciadaDuermeHastaLaHoraYLuegoDespierta() {
    var a = nueva()
    a.recibir(.silenciar(hasta: at(3600)), ahora: at(0))
    #expect(a.emocion == .dormida && a.silenciada)
    a.recibir(.videoLlego("a.mp4"), ahora: at(10))
    a.recibir(.revisionEmpezo("a.mp4"), ahora: at(11))
    #expect(a.emocion == .dormida)
    a.tick(ahora: at(3600))
    #expect(!a.silenciada && a.emocion == .revisando && a.frase != nil)
}

@Test func abrirElGloboDespiertaAunqueEsteSilenciada() {
    var a = nueva()
    a.recibir(.silenciar(hasta: at(3600)), ahora: at(0))
    a.recibir(.globoAbierto, ahora: at(5))
    #expect(a.emocion == .escuchando && !a.silenciada)
    a.recibir(.globoCerrado, ahora: at(6))
    #expect(a.emocion == .feliz)
}

@Test func revisionCortadaPoneTristeUnRato() {
    var a = nueva()
    a.recibir(.revisionEmpezo("a.mp4"), ahora: at(1))
    a.recibir(.revisionSeDetuvo("a.mp4"), ahora: at(2))
    #expect(a.emocion == .triste && !a.revisando)
    a.tick(ahora: at(10)); #expect(a.emocion == .feliz)
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `swift test`
Expected: FAIL, `cannot find 'EstadoAura' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Sources/AuraNucleo/EstadoAura.swift
import Foundation

/// Qué siente Aura y qué dice. Pura: sin reloj propio ni E/S; el tiempo entra por `ahora`.
public struct EstadoAura: Sendable {
    public static let segundosParaDormir: TimeInterval = 120

    public private(set) var emocion: Emocion = .feliz
    public private(set) var frase: String?
    public private(set) var fraseID = 0
    public private(set) var revisando = false
    public var silenciada: Bool { silenciadaHasta != nil }

    private var videoEnCurso: String?
    private var ultimaActividad: Date
    private var transitoriaHasta: Date?
    private var silenciadaHasta: Date?
    private var errorActual: ErrorMotor?
    private var globoAbierto = false
    private var frases: Frases

    public init(ahora: Date, frases: Frases = Frases()) {
        ultimaActividad = ahora
        self.frases = frases
        decir(.saludo)
    }

    private var reposo: Emocion { revisando ? .revisando : (globoAbierto ? .escuchando : .feliz) }
    private var callada: Bool { silenciadaHasta != nil || errorActual != nil }

    private mutating func decir(_ s: Situacion, _ v: [String: String] = [:]) {
        frase = frases.decir(s, v)
        fraseID += 1
    }

    private mutating func mostrar(_ e: Emocion, por segundos: TimeInterval, ahora: Date) {
        emocion = e
        transitoriaHasta = ahora.addingTimeInterval(segundos)
    }

    private static func situacion(de e: ErrorMotor) -> Situacion {
        switch e {
        case .sinMotor: return .sinMotor
        case .sinConfigurar: return .sinConfigurar
        case .versionDesconocida: return .versionDesconocida
        case .fallo: return .fallo
        }
    }

    public mutating func recibir(_ evento: EventoAura, ahora: Date) {
        switch evento {
        case .silenciar(let hasta):
            silenciadaHasta = hasta; transitoriaHasta = nil; emocion = .dormida; frase = nil
            return
        case .errorMotor(let e):
            guard errorActual != e else { return }
            errorActual = e; transitoriaHasta = nil; ultimaActividad = ahora
            if silenciadaHasta == nil { emocion = .triste; decir(Self.situacion(de: e)) }
            return
        case .motorBien:
            guard errorActual != nil else { return }
            errorActual = nil
            if silenciadaHasta == nil { emocion = reposo; frase = nil }
            return
        case .globoAbierto:
            globoAbierto = true; silenciadaHasta = nil; ultimaActividad = ahora
            if errorActual == nil, transitoriaHasta == nil { emocion = .escuchando }
            return
        case .globoCerrado:
            globoAbierto = false; ultimaActividad = ahora
            if !callada, transitoriaHasta == nil { emocion = reposo }
            return
        default:
            break
        }

        ultimaActividad = ahora
        switch evento {
        case .videoLlego(let v):
            guard !callada else { return }
            decir(.llego, ["video": nombreVisible(v)])
            mostrar(.sorprendida, por: 2, ahora: ahora)
        case .revisionEmpezo(let v):
            revisando = true; videoEnCurso = v
            guard !callada, transitoriaHasta == nil else { return }
            emocion = .revisando
            decir(.revisando, ["video": nombreVisible(v)])
        case .revisionesTerminaron(let rs):
            revisando = false; videoEnCurso = nil
            guard !callada, !rs.isEmpty else { return }
            reaccionar(a: rs, ahora: ahora)
        case .revisionSeDetuvo(let v):
            revisando = false; videoEnCurso = nil
            guard !callada else { return }
            decir(.seDetuvo, ["video": nombreVisible(v)])
            mostrar(.triste, por: 8, ahora: ahora)
        case .noEsVideo:
            guard !callada else { return }
            decir(.noEsVideo)
            mostrar(.sorprendida, por: 3, ahora: ahora)
        default:
            break
        }
    }

    private mutating func reaccionar(a rs: [ResultadoVideo], ahora: Date) {
        let conErrores = rs.filter { $0.estado == .conErrores }
        let pendientes = rs.filter { $0.estado == .pendiente }
        if rs.count == 1, let r = rs.first {
            let video = nombreVisible(r.video)
            switch r.estado {
            case .conErrores:
                decir(.conErrores, ["video": video, "cositas": cositas(r.cositas)])
                mostrar(.preocupada, por: 8, ahora: ahora)
            case .aprobado:
                decir(.aprobado, ["video": video])
                mostrar(.celebrando, por: 6, ahora: ahora)
            case .pendiente:
                decir(.pendiente, ["video": video])
                mostrar(.triste, por: 8, ahora: ahora)
            }
        } else if !conErrores.isEmpty {
            decir(.variosConErrores, ["k": "\(rs.count)", "m": "\(conErrores.count)"])
            mostrar(.preocupada, por: 8, ahora: ahora)
        } else if let p = pendientes.first {
            decir(.pendiente, ["video": nombreVisible(p.video)])
            mostrar(.triste, por: 8, ahora: ahora)
        } else {
            decir(.variosAprobados, ["k": "\(rs.count)"])
            mostrar(.celebrando, por: 6, ahora: ahora)
        }
    }

    public mutating func tick(ahora: Date) {
        if let hasta = silenciadaHasta {
            guard ahora >= hasta else { return }
            silenciadaHasta = nil; ultimaActividad = ahora
            emocion = errorActual == nil ? reposo : .triste
            decir(.despierta)
            return
        }
        if errorActual != nil { return }
        if let t = transitoriaHasta {
            guard ahora >= t else { return }
            transitoriaHasta = nil
            emocion = reposo
            if revisando, let v = videoEnCurso { decir(.revisando, ["video": nombreVisible(v)]) }
            return
        }
        if emocion != .dormida, !revisando, !globoAbierto,
           ahora.timeIntervalSince(ultimaActividad) >= Self.segundosParaDormir {
            emocion = .dormida
            frase = nil
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `swift test`
Expected: PASS, toda la suite.

- [ ] **Step 5: Commit**

```bash
git add Sources/AuraNucleo/EstadoAura.swift Tests/AuraNucleoTests/EstadoAuraTests.swift
git commit -m "feat: máquina de estados de las emociones de Aura

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task B5: Cliente del motor y utilidades puras

**Files:**
- Create: `Sources/AuraNucleo/MotorCLI.swift`, `Sources/AuraNucleo/Utilidades.swift`
- Test: `Tests/AuraNucleoTests/MotorCLITests.swift`, `Tests/AuraNucleoTests/UtilidadesTests.swift`

**Interfaces:**
- Consumes: `EstadoMotor.decodificar` y `ErrorMotor` (B1).
- Produces:
  - `SalidaProceso(codigo: Int32, salida: Data, error: Data)`.
  - `protocol Ejecutor: Sendable { func ejecutar(_ programa: URL, _ argumentos: [String], entorno: [String: String]) async throws -> SalidaProceso }` y su implementación `EjecutorReal()`.
  - `MotorCLI(ejecutor:uv:proyecto:home:)`, `MotorCLI.predeterminado()`, `MotorCLI.buscarUV(home:existe:)`, `estado() async -> Result<EstadoMotor, ErrorMotor>`, `revisar(_ video: URL) async -> Result<Void, ErrorMotor>` y `MotorCLI.interpretarEstado(_:)`.
  - `AccionSoltar.decidir(_ url: URL, automatico: Bool) -> AccionSoltar` (`copiarAEntrada(URL)`, `revisar(URL)`, `ignorar`).
  - `Vigilancia.relevante(_ ruta: String, raiz: String) -> Bool`.
  - `Mirada.desplazamiento(raton: CGPoint, centro: CGPoint, maximo: CGFloat = 4) -> CGVector`.
  - `Parpadeo.cerrado(en t: Double) -> Bool`.

- [ ] **Step 1: Write the failing tests**

```swift
// Tests/AuraNucleoTests/MotorCLITests.swift
import Foundation
import Testing
@testable import AuraNucleo

final class EjecutorFalso: Ejecutor, @unchecked Sendable {
    var respuesta: SalidaProceso
    var lanzar = false
    private(set) var llamadas: [(URL, [String], [String: String])] = []
    init(_ r: SalidaProceso) { respuesta = r }
    func ejecutar(_ programa: URL, _ argumentos: [String], entorno: [String: String]) async throws -> SalidaProceso {
        llamadas.append((programa, argumentos, entorno))
        if lanzar { throw CocoaError(.fileNoSuchFile) }
        return respuesta
    }
}

private func salida(_ codigo: Int32, _ out: String = "", _ err: String = "") -> SalidaProceso {
    SalidaProceso(codigo: codigo, salida: Data(out.utf8), error: Data(err.utf8))
}

private func proyectoFalso() throws -> URL {
    let d = FileManager.default.temporaryDirectory.appendingPathComponent("vqa-\(UUID().uuidString)")
    try FileManager.default.createDirectory(at: d, withIntermediateDirectories: true)
    FileManager.default.createFile(atPath: d.appendingPathComponent("pyproject.toml").path, contents: Data())
    return d
}

@Test func interpretaLosCodigosDeSalida() {
    let ok = MotorCLI.interpretarEstado(SalidaProceso(codigo: 0, salida: Fixtures.estadoJSON, error: Data()))
    #expect((try? ok.get())?.automatico == true)
    #expect(MotorCLI.interpretarEstado(salida(4, "", "SIN_CONFIG")) == .failure(.sinConfigurar))
    #expect(MotorCLI.interpretarEstado(salida(2, "", "error: argument cmd: invalid choice: 'estado'")) == .failure(.versionDesconocida))
    #expect(MotorCLI.interpretarEstado(salida(0, #"{"version": 9}"#)) == .failure(.versionDesconocida))
    if case .failure(.fallo) = MotorCLI.interpretarEstado(salida(0, "no es json")) {} else { Issue.record("esperaba fallo") }
    if case .failure(.fallo) = MotorCLI.interpretarEstado(salida(1, "", "Traceback…")) {} else { Issue.record("esperaba fallo") }
}

@Test func llamaAUvConArgumentosYPath() async throws {
    let ej = EjecutorFalso(SalidaProceso(codigo: 0, salida: Fixtures.estadoJSON, error: Data()))
    let proyecto = try proyectoFalso()
    let cli = MotorCLI(ejecutor: ej, uv: URL(fileURLWithPath: "/opt/homebrew/bin/uv"), proyecto: proyecto,
                       home: URL(fileURLWithPath: "/Users/ella"))
    _ = await cli.estado()
    let (programa, args, entorno) = try #require(ej.llamadas.first)
    #expect(programa.path == "/opt/homebrew/bin/uv")
    #expect(args == ["run", "-q", "--project", proyecto.path, "videoqa", "estado", "--json"])
    #expect(entorno["PATH"]?.hasPrefix("/opt/homebrew/bin:") == true)
    #expect(entorno["HOME"] == "/Users/ella")
}

@Test func sinUvOSinProyectoEsSinMotor() async throws {
    let ej = EjecutorFalso(salida(0))
    let sinUv = MotorCLI(ejecutor: ej, uv: nil, proyecto: try proyectoFalso(), home: URL(fileURLWithPath: "/x"))
    #expect(await sinUv.estado() == .failure(.sinMotor))
    let sinProyecto = MotorCLI(ejecutor: ej, uv: URL(fileURLWithPath: "/u"),
                               proyecto: URL(fileURLWithPath: "/no/existe"), home: URL(fileURLWithPath: "/x"))
    #expect(await sinProyecto.estado() == .failure(.sinMotor))
    #expect(ej.llamadas.isEmpty)
}

@Test func revisarPasaLaRutaTalCualYAceptaEspera() async throws {
    let ej = EjecutorFalso(salida(3))
    let cli = MotorCLI(ejecutor: ej, uv: URL(fileURLWithPath: "/u"), proyecto: try proyectoFalso(), home: URL(fileURLWithPath: "/x"))
    let video = URL(fileURLWithPath: "/Users/ella/Desktop/promo otoño ñ.mp4")
    if case .success = await cli.revisar(video) {} else { Issue.record("exit 3 (en espera) cuenta como éxito") }
    #expect(ej.llamadas.first?.1.suffix(2) == ["run", "/Users/ella/Desktop/promo otoño ñ.mp4"])
    ej.respuesta = salida(1, "", "ERROR: algo")
    if case .failure(.fallo) = await cli.revisar(video) {} else { Issue.record("esperaba fallo") }
}

@Test func buscaUvEnElOrdenEsperado() {
    let home = URL(fileURLWithPath: "/Users/ella")
    #expect(MotorCLI.buscarUV(home: home, existe: { $0 == "/Users/ella/.local/bin/uv" })?.path == "/Users/ella/.local/bin/uv")
    #expect(MotorCLI.buscarUV(home: home, existe: { _ in true })?.path == "/opt/homebrew/bin/uv")
    #expect(MotorCLI.buscarUV(home: home, existe: { _ in false }) == nil)
}

@Test func ejecutorRealDevuelveSalidaGrandeSinBloquearse() async throws {
    let r = try await EjecutorReal().ejecutar(URL(fileURLWithPath: "/bin/sh"),
                                              ["-c", "head -c 300000 /dev/zero; printf 'ñ' >&2; exit 3"], entorno: [:])
    #expect(r.codigo == 3)
    #expect(r.salida.count == 300_000)
    #expect(String(decoding: r.error, as: UTF8.self) == "ñ")
}
```

```swift
// Tests/AuraNucleoTests/UtilidadesTests.swift
import CoreGraphics
import Foundation
import Testing
@testable import AuraNucleo

@Test func soltarSoloAceptaVideos() {
    let mp4 = URL(fileURLWithPath: "/x/promo otoño ñ.MP4")
    #expect(AccionSoltar.decidir(mp4, automatico: true) == .copiarAEntrada(mp4))
    #expect(AccionSoltar.decidir(mp4, automatico: false) == .revisar(mp4))
    #expect(AccionSoltar.decidir(URL(fileURLWithPath: "/x/notas.txt"), automatico: false) == .ignorar)
    #expect(AccionSoltar.decidir(URL(string: "https://x.com/a.mp4")!, automatico: false) == .ignorar)
}

@Test func vigilanciaSoloAvisaDeLoQueImporta() {
    let raiz = "/Users/ella/Library/CloudStorage/Drive/Videos"
    #expect(Vigilancia.relevante(raiz + "/01_Entrada/a.mp4", raiz: raiz))
    #expect(Vigilancia.relevante(raiz + "/03_Aprobado/Sushi CD/a/reporte.md", raiz: raiz + "/"))
    #expect(Vigilancia.relevante("/Users/ella/.videoqa/en_curso.json", raiz: raiz))
    #expect(!Vigilancia.relevante(raiz + "/01_Entrada/.a.mp4.tmp", raiz: raiz))
    #expect(!Vigilancia.relevante(raiz + "/_config/glosario.txt", raiz: raiz))
    #expect(!Vigilancia.relevante(raiz + "/01_EntradaX/a.mp4", raiz: raiz))
    #expect(!Vigilancia.relevante("/Users/ella/.videoqa/videoqa.log", raiz: raiz))
}

@Test func miradaApuntaAlRatonConLimite() {
    let lejos = Mirada.desplazamiento(raton: CGPoint(x: 1000, y: 0), centro: .zero)
    #expect(abs(lejos.dx - 4) < 0.001 && abs(lejos.dy) < 0.001)
    let arriba = Mirada.desplazamiento(raton: CGPoint(x: 0, y: 1000), centro: .zero)
    #expect(arriba.dy < 0)
    #expect(Mirada.desplazamiento(raton: .zero, centro: .zero) == .zero)
}

@Test func parpadeaDeVezEnCuandoYPoco() {
    var parpadeos = 0
    var anterior = false
    var t = 0.0
    while t < 60 {
        let c = Parpadeo.cerrado(en: t)
        if c && !anterior { parpadeos += 1 }
        anterior = c
        t += 0.01
    }
    #expect((8...16).contains(parpadeos))
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `swift test`
Expected: FAIL, `cannot find 'MotorCLI' in scope`.

- [ ] **Step 3: Write minimal implementation**

```swift
// Sources/AuraNucleo/MotorCLI.swift
import Foundation

public struct SalidaProceso: Sendable, Equatable {
    public let codigo: Int32
    public let salida: Data
    public let error: Data
    public init(codigo: Int32, salida: Data, error: Data) { self.codigo = codigo; self.salida = salida; self.error = error }
}

public protocol Ejecutor: Sendable {
    func ejecutar(_ programa: URL, _ argumentos: [String], entorno: [String: String]) async throws -> SalidaProceso
}

/// Ejecuta un proceso sin shell. La salida va a archivos temporales: con pipes, una salida
/// grande llenaría el búfer y el proceso se quedaría colgado.
public struct EjecutorReal: Ejecutor {
    public init() {}

    public func ejecutar(_ programa: URL, _ argumentos: [String], entorno: [String: String]) async throws -> SalidaProceso {
        let fm = FileManager.default
        let base = fm.temporaryDirectory.appendingPathComponent("aura-\(UUID().uuidString)")
        let out = base.appendingPathExtension("out"), err = base.appendingPathExtension("err")
        fm.createFile(atPath: out.path, contents: nil)
        fm.createFile(atPath: err.path, contents: nil)
        defer { try? fm.removeItem(at: out); try? fm.removeItem(at: err) }
        let hOut = try FileHandle(forWritingTo: out), hErr = try FileHandle(forWritingTo: err)

        let p = Process()
        p.executableURL = programa
        p.arguments = argumentos
        if !entorno.isEmpty { p.environment = entorno }
        p.standardOutput = hOut
        p.standardError = hErr
        p.standardInput = FileHandle.nullDevice
        try await withCheckedThrowingContinuation { (c: CheckedContinuation<Void, Error>) in
            p.terminationHandler = { _ in c.resume() }
            do { try p.run() } catch { p.terminationHandler = nil; c.resume(throwing: error) }
        }
        try hOut.close(); try hErr.close()
        return SalidaProceso(codigo: p.terminationStatus, salida: try Data(contentsOf: out), error: try Data(contentsOf: err))
    }
}

/// Habla con `videoqa` a través de `uv run`. Nunca usa una shell.
public struct MotorCLI: Sendable {
    public let ejecutor: any Ejecutor
    public let uv: URL?
    public let proyecto: URL
    public let home: URL

    public init(ejecutor: any Ejecutor, uv: URL?, proyecto: URL, home: URL) {
        self.ejecutor = ejecutor; self.uv = uv; self.proyecto = proyecto; self.home = home
    }

    public static func predeterminado() -> MotorCLI {
        let home = FileManager.default.homeDirectoryForCurrentUser
        return MotorCLI(ejecutor: EjecutorReal(),
                        uv: buscarUV(home: home, existe: { FileManager.default.isExecutableFile(atPath: $0) }),
                        proyecto: home.appendingPathComponent("videoqa"), home: home)
    }

    public static func buscarUV(home: URL, existe: (String) -> Bool) -> URL? {
        ["/opt/homebrew/bin/uv", "/usr/local/bin/uv", home.path + "/.local/bin/uv", home.path + "/.cargo/bin/uv"]
            .first(where: existe).map { URL(fileURLWithPath: $0) }
    }

    /// Las apps abiertas desde el Finder no heredan el PATH de Homebrew; ffmpeg lo necesita `run`.
    public var entorno: [String: String] {
        ["HOME": home.path, "LANG": "es_ES.UTF-8",
         "PATH": "/opt/homebrew/bin:/usr/local/bin:\(home.path)/.local/bin:/usr/bin:/bin:/usr/sbin:/sbin"]
    }

    func videoqa(_ args: [String]) async -> Result<SalidaProceso, ErrorMotor> {
        guard let uv, FileManager.default.fileExists(atPath: proyecto.appendingPathComponent("pyproject.toml").path) else {
            return .failure(.sinMotor)
        }
        do {
            return .success(try await ejecutor.ejecutar(uv, ["run", "-q", "--project", proyecto.path, "videoqa"] + args,
                                                        entorno: entorno))
        } catch {
            return .failure(.sinMotor)
        }
    }

    public func estado() async -> Result<EstadoMotor, ErrorMotor> {
        switch await videoqa(["estado", "--json"]) {
        case .failure(let e): return .failure(e)
        case .success(let s): return Self.interpretarEstado(s)
        }
    }

    /// `videoqa run` copia a 01_Entrada si el video viene de fuera. 0 = revisado; 3 = en espera de uso de Claude.
    public func revisar(_ video: URL) async -> Result<Void, ErrorMotor> {
        switch await videoqa(["run", video.path]) {
        case .failure(let e): return .failure(e)
        case .success(let s):
            if s.codigo == 0 || s.codigo == 3 { return .success(()) }
            return .failure(.fallo(String(String(decoding: s.error, as: UTF8.self).suffix(300))))
        }
    }

    public static func interpretarEstado(_ s: SalidaProceso) -> Result<EstadoMotor, ErrorMotor> {
        let err = String(decoding: s.error, as: UTF8.self)
        if s.codigo == 4 { return .failure(.sinConfigurar) }
        if err.contains("invalid choice") { return .failure(.versionDesconocida) }
        guard s.codigo == 0 else { return .failure(.fallo(String(err.suffix(300)))) }
        do {
            return .success(try EstadoMotor.decodificar(s.salida))
        } catch let e as ErrorMotor {
            return .failure(e)
        } catch {
            return .failure(.fallo("respuesta ilegible del motor"))
        }
    }
}
```

```swift
// Sources/AuraNucleo/Utilidades.swift
import CoreGraphics
import Foundation

public enum AccionSoltar: Equatable, Sendable {
    case copiarAEntrada(URL), revisar(URL), ignorar

    public static let extensiones: Set<String> = ["mp4", "mov", "m4v"]

    public static func decidir(_ url: URL, automatico: Bool) -> AccionSoltar {
        guard url.isFileURL, extensiones.contains(url.pathExtension.lowercased()) else { return .ignorar }
        return automatico ? .copiarAEntrada(url) : .revisar(url)
    }
}

/// Qué cambios de disco merecen preguntar otra vez al motor (el log cambia sin parar y no cuenta).
public enum Vigilancia {
    static let carpetas = ["/01_Entrada", "/02_Con_errores", "/03_Aprobado"]

    public static func relevante(_ ruta: String, raiz: String) -> Bool {
        if ruta.hasSuffix("/en_curso.json") { return true }
        let r = raiz.hasSuffix("/") ? String(raiz.dropLast()) : raiz
        guard ruta.hasPrefix(r + "/") else { return false }
        let resto = String(ruta.dropFirst(r.count))
        if (resto as NSString).lastPathComponent.hasPrefix(".") { return false }
        return carpetas.contains { resto == $0 || resto.hasPrefix($0 + "/") }
    }
}

/// Hacia dónde miran los ojos, en unidades del dibujo (y positiva = abajo, como SwiftUI).
public enum Mirada {
    public static func desplazamiento(raton: CGPoint, centro: CGPoint, maximo: CGFloat = 4) -> CGVector {
        let dx = raton.x - centro.x, dy = raton.y - centro.y   // pantalla de macOS: y crece hacia arriba
        let d = (dx * dx + dy * dy).squareRoot()
        guard d > 1 else { return .zero }
        let f = min(d / 200, 1) * maximo / d
        return CGVector(dx: dx * f, dy: -dy * f)
    }
}

/// Parpadeo irregular (cada ~5 s, con desfase) calculado solo a partir del tiempo.
public enum Parpadeo {
    public static let ciclo = 5.0
    public static let duracion = 0.12

    public static func cerrado(en t: Double) -> Bool {
        let n = (t / ciclo).rounded(.down)
        let desfase = (n * 2.718281828).truncatingRemainder(dividingBy: 1) * 3
        let dentro = t - n * ciclo - desfase
        return dentro >= 0 && dentro < duracion
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `swift test`
Expected: PASS, toda la suite.

- [ ] **Step 5: Commit**

```bash
git add Sources/AuraNucleo Tests/AuraNucleoTests
git commit -m "feat: cliente de videoqa y utilidades (soltar, vigilancia, mirada, parpadeo)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task B6: Dibujo del durazno y sus caras

**Files:**
- Create: `Sources/Aura/Color+Hex.swift`, `Sources/Aura/MascotaView.swift`, `Sources/Aura/Icono.swift`
- Modify: `Sources/Aura/main.swift` (flags `--icono` y `--muestras`)

**Interfaces:**
- Consumes: `Emocion` (B1); `Parpadeo.cerrado(en:)` (B5).
- Produces: `MascotaView(emocion: Emocion, mirada: @escaping () -> CGVector = { .zero }, animada: Bool = true)`; `Paleta` (colores); `Color(hex:)`; `Icono.exportar(a: URL) -> Bool`; `Icono.muestras(en: URL) -> Bool`. Las usan B7 y B8.

Esta tarea es visual: no hay tests unitarios. La verificación consiste en renderizar las 9 caras a PNG y mirarlas.

- [ ] **Step 1: Paleta y color hex**

```swift
// Sources/Aura/Color+Hex.swift
import SwiftUI

extension Color {
    init(hex: UInt32, opacidad: Double = 1) {
        self.init(.sRGB, red: Double((hex >> 16) & 0xFF) / 255, green: Double((hex >> 8) & 0xFF) / 255,
                  blue: Double(hex & 0xFF) / 255, opacity: opacidad)
    }
}

enum Paleta {
    static let cuerpo = Color(hex: 0xF7B9AE), brillo = Color(hex: 0xFBD3C7), linea = Color(hex: 0xE7998D)
    static let mejillas = Color(hex: 0xF29C9C), brazo = Color(hex: 0xEFA59A)
    static let hoja = Color(hex: 0x8CC08A), nervio = Color(hex: 0x6FA56D), tallo = Color(hex: 0x8A5A3B)
    static let visor = Color(hex: 0x2B2340), ojos = Color(hex: 0xCDBEFF)
    static let zzz = Color(hex: 0x9C8FD6), burbuja = Color(hex: 0xBDB6D6), gota = Color(hex: 0x9FD3F0)
}
```

- [ ] **Step 2: La vista del durazno**

El dibujo usa el sistema de coordenadas del boceto (200 × 220) y escala al tamaño de la vista.

```swift
// Sources/Aura/MascotaView.swift
import AuraNucleo
import SwiftUI

struct MascotaView: View {
    let emocion: Emocion
    var mirada: () -> CGVector = { .zero }
    var animada = true
    @Environment(\.accessibilityReduceMotion) private var reducirMovimiento

    var body: some View {
        Group {
            if animada && !reducirMovimiento {
                TimelineView(.animation(minimumInterval: emocion == .dormida ? 1.0 / 10 : 1.0 / 30)) { contexto in
                    lienzo(t: contexto.date.timeIntervalSinceReferenceDate)
                }
            } else {
                lienzo(t: 1)  // t = 1: ojos abiertos (en t = 0 cae un parpadeo)
            }
        }
        .aspectRatio(200.0 / 220.0, contentMode: .fit)
        .accessibilityLabel("Aura, \(emocion.rawValue)")
    }

    private func lienzo(t: Double) -> some View {
        Canvas { ctx, tamano in
            Durazno.dibujar(en: &ctx, tamano: tamano, emocion: emocion, mirada: mirada(), t: t)
        }
    }
}

enum Durazno {
    static func dibujar(en ctx: inout GraphicsContext, tamano: CGSize, emocion: Emocion, mirada: CGVector, t: Double) {
        let s = min(tamano.width / 200, tamano.height / 220)
        ctx.translateBy(x: (tamano.width - 200 * s) / 2, y: (tamano.height - 220 * s) / 2)
        ctx.scaleBy(x: s, y: s)

        var dy = sin(t * 2 * .pi / 3.2) * 3
        var escalaY = 1.0
        switch emocion {
        case .dormida: dy = 0; escalaY = 1 - 0.015 * (1 + sin(t * 2 * .pi / 4.5))
        case .celebrando: dy = -abs(sin(t * .pi / 0.6)) * 12
        default: break
        }

        ctx.fill(Path(ellipseIn: CGRect(x: 48, y: 199, width: 104, height: 14)), with: .color(.black.opacity(0.08)))

        var cuerpo = ctx
        cuerpo.translateBy(x: 0, y: dy)
        cuerpo.translateBy(x: 100, y: 202); cuerpo.scaleBy(x: 1, y: escalaY); cuerpo.translateBy(x: -100, y: -202)
        brazos(&cuerpo, emocion: emocion)
        fruta(&cuerpo)
        hoja(&cuerpo, t: t)
        cuerpo.fill(Path(roundedRect: CGRect(x: 52, y: 94, width: 96, height: 58), cornerRadius: 28), with: .color(Paleta.visor))
        cara(&cuerpo, emocion: emocion, mirada: mirada, cerrados: Parpadeo.cerrado(en: t))
        adornos(&ctx, emocion: emocion, t: t)
    }

    // MARK: cuerpo

    static let silueta: Path = {
        var p = Path()
        p.move(to: CGPoint(x: 100, y: 62))
        p.addCurve(to: CGPoint(x: 172, y: 140), control1: CGPoint(x: 150, y: 50), control2: CGPoint(x: 178, y: 95))
        p.addCurve(to: CGPoint(x: 100, y: 202), control1: CGPoint(x: 166, y: 182), control2: CGPoint(x: 134, y: 202))
        p.addCurve(to: CGPoint(x: 28, y: 140), control1: CGPoint(x: 66, y: 202), control2: CGPoint(x: 34, y: 182))
        p.addCurve(to: CGPoint(x: 100, y: 62), control1: CGPoint(x: 22, y: 95), control2: CGPoint(x: 50, y: 50))
        p.closeSubpath()
        return p
    }()

    static func fruta(_ ctx: inout GraphicsContext) {
        ctx.fill(silueta, with: .color(Paleta.cuerpo))
        var brillo = Path()
        brillo.move(to: CGPoint(x: 100, y: 66))
        brillo.addCurve(to: CGPoint(x: 164, y: 138), control1: CGPoint(x: 140, y: 60), control2: CGPoint(x: 166, y: 96))
        brillo.addCurve(to: CGPoint(x: 100, y: 80), control1: CGPoint(x: 150, y: 110), control2: CGPoint(x: 122, y: 84))
        brillo.closeSubpath()
        ctx.fill(brillo, with: .color(Paleta.brillo.opacity(0.7)))
        var surco = Path()
        surco.move(to: CGPoint(x: 100, y: 64))
        surco.addCurve(to: CGPoint(x: 98, y: 150), control1: CGPoint(x: 92, y: 90), control2: CGPoint(x: 92, y: 120))
        ctx.stroke(surco, with: .color(Paleta.linea), style: StrokeStyle(lineWidth: 3, lineCap: .round))
        for x in [58.0, 142.0] {
            ctx.fill(Path(ellipseIn: CGRect(x: x - 12, y: 161, width: 24, height: 14)), with: .color(Paleta.mejillas.opacity(0.6)))
        }
    }

    static func brazos(_ ctx: inout GraphicsContext, emocion: Emocion) {
        let arriba = emocion == .celebrando
        for (x, giro) in [(38.0, -20.0), (162.0, 20.0)] {
            var b = ctx
            b.translateBy(x: x, y: arriba ? 118 : 140)
            b.rotate(by: .degrees(arriba ? giro * 2.5 : giro))
            b.fill(Path(ellipseIn: CGRect(x: -13, y: -18, width: 26, height: 36)), with: .color(Paleta.brazo))
        }
    }

    static func hoja(_ ctx: inout GraphicsContext, t: Double) {
        var tallo = Path()
        tallo.move(to: CGPoint(x: 100, y: 64))
        tallo.addCurve(to: CGPoint(x: 108, y: 34), control1: CGPoint(x: 99, y: 52), control2: CGPoint(x: 102, y: 42))
        ctx.stroke(tallo, with: .color(Paleta.tallo), style: StrokeStyle(lineWidth: 5, lineCap: .round))
        var h = ctx
        h.translateBy(x: 106, y: 40); h.rotate(by: .degrees(sin(t * 1.3) * 4)); h.translateBy(x: -106, y: -40)
        var forma = Path()
        forma.move(to: CGPoint(x: 106, y: 40))
        forma.addCurve(to: CGPoint(x: 156, y: 30), control1: CGPoint(x: 120, y: 22), control2: CGPoint(x: 146, y: 22))
        forma.addCurve(to: CGPoint(x: 106, y: 40), control1: CGPoint(x: 146, y: 46), control2: CGPoint(x: 122, y: 52))
        forma.closeSubpath()
        h.fill(forma, with: .color(Paleta.hoja))
        var nervio = Path()
        nervio.move(to: CGPoint(x: 108, y: 40))
        nervio.addCurve(to: CGPoint(x: 154, y: 30), control1: CGPoint(x: 124, y: 34), control2: CGPoint(x: 140, y: 32))
        h.stroke(nervio, with: .color(Paleta.nervio), style: StrokeStyle(lineWidth: 2))
    }

    // MARK: cara

    struct Cara { var ojos = Path(); var ojosRellenos = false; var boca = Path(); var bocaRellena = false; var cejas = Path() }

    static func q(_ x1: CGFloat, _ y1: CGFloat, _ cx: CGFloat, _ cy: CGFloat, _ x2: CGFloat, _ y2: CGFloat) -> Path {
        var p = Path(); p.move(to: CGPoint(x: x1, y: y1)); p.addQuadCurve(to: CGPoint(x: x2, y: y2), control: CGPoint(x: cx, y: cy)); return p
    }
    static func linea(_ x1: CGFloat, _ y1: CGFloat, _ x2: CGFloat, _ y2: CGFloat) -> Path {
        var p = Path(); p.move(to: CGPoint(x: x1, y: y1)); p.addLine(to: CGPoint(x: x2, y: y2)); return p
    }
    static func circulo(_ x: CGFloat, _ y: CGFloat, _ r: CGFloat) -> Path {
        Path(ellipseIn: CGRect(x: x - r, y: y - r, width: 2 * r, height: 2 * r))
    }
    static func unir(_ partes: Path...) -> Path {
        var p = Path(); partes.forEach { p.addPath($0) }; return p
    }

    static func cara(para e: Emocion) -> Cara {
        var c = Cara()
        switch e {
        case .feliz:
            c.ojos = unir(q(70, 125, 81, 112, 92, 125), q(108, 125, 119, 112, 130, 125)); c.boca = q(91, 137, 100, 146, 109, 137)
        case .dormida:
            c.ojos = unir(q(70, 121, 81, 129, 92, 121), q(108, 121, 119, 129, 130, 121)); c.boca = linea(96, 139, 104, 139)
        case .sorprendida:
            c.ojos = unir(circulo(81, 120, 7), circulo(119, 120, 7)); c.ojosRellenos = true; c.boca = circulo(100, 140, 4)
        case .revisando:
            c.ojos = unir(circulo(88, 116, 6), circulo(124, 116, 6)); c.ojosRellenos = true; c.boca = q(95, 141, 101, 137, 107, 141)
        case .preocupada:
            c.ojos = unir(circulo(82, 123, 5), circulo(118, 123, 5)); c.ojosRellenos = true
            c.cejas = unir(linea(70, 111, 90, 115), linea(110, 115, 130, 111))
            var b = Path(); b.move(to: CGPoint(x: 89, y: 142))
            b.addQuadCurve(to: CGPoint(x: 100, y: 142), control: CGPoint(x: 94, y: 137))
            b.addQuadCurve(to: CGPoint(x: 111, y: 142), control: CGPoint(x: 106, y: 147))
            c.boca = b
        case .celebrando:
            c.ojos = unir(q(68, 124, 81, 108, 94, 124), q(106, 124, 119, 108, 132, 124))
            var b = q(89, 134, 100, 150, 111, 134); b.closeSubpath(); c.boca = b; c.bocaRellena = true
        case .editando:
            c.ojos = unir(circulo(84, 124, 5), circulo(116, 124, 5)); c.ojosRellenos = true; c.boca = linea(96, 140, 104, 140)
        case .escuchando:
            c.ojos = unir(circulo(81, 120, 6), circulo(119, 120, 6)); c.ojosRellenos = true; c.boca = q(92, 137, 100, 143, 108, 137)
        case .triste:
            c.ojos = unir(circulo(82, 122, 5), circulo(118, 122, 5)); c.ojosRellenos = true
            c.cejas = unir(linea(72, 113, 88, 110), linea(112, 110, 128, 113)); c.boca = q(91, 143, 100, 135, 109, 143)
        }
        return c
    }

    static func cara(_ ctx: inout GraphicsContext, emocion: Emocion, mirada: CGVector, cerrados: Bool) {
        let c = cara(para: emocion)
        let trazo = StrokeStyle(lineWidth: 5, lineCap: .round, lineJoin: .round)
        var ojos = ctx
        if c.ojosRellenos { ojos.translateBy(x: mirada.dx, y: mirada.dy) }
        if cerrados && emocion != .dormida {
            ojos.translateBy(x: 0, y: 120); ojos.scaleBy(x: 1, y: 0.12); ojos.translateBy(x: 0, y: -120)
        }
        if c.ojosRellenos { ojos.fill(c.ojos, with: .color(Paleta.ojos)) } else { ojos.stroke(c.ojos, with: .color(Paleta.ojos), style: trazo) }
        ctx.stroke(c.cejas, with: .color(Paleta.ojos), style: StrokeStyle(lineWidth: 4, lineCap: .round))
        if c.bocaRellena { ctx.fill(c.boca, with: .color(Paleta.ojos)) } else { ctx.stroke(c.boca, with: .color(Paleta.ojos), style: trazo) }
    }

    // MARK: adornos (no se mueven con el cuerpo)

    static func adornos(_ ctx: inout GraphicsContext, emocion: Emocion, t: Double) {
        switch emocion {
        case .dormida:
            let sube = (t * 6).truncatingRemainder(dividingBy: 8)
            for (texto, x, y, tam) in [("z", 150.0, 80.0, 16.0), ("z", 162.0, 64.0, 20.0), ("Z", 176.0, 46.0, 24.0)] {
                ctx.draw(Text(texto).font(.system(size: tam, weight: .semibold, design: .rounded)).foregroundColor(Paleta.zzz),
                         at: CGPoint(x: x, y: y - sube))
            }
        case .revisando:
            for (i, (x, y, r)) in [(160.0, 84.0, 4.0), (170.0, 70.0, 6.0), (182.0, 52.0, 9.0)].enumerated() {
                let pulso = 0.6 + 0.4 * sin(t * 3 - Double(i))
                ctx.fill(circulo(x, y, r), with: .color(Paleta.burbuja.opacity(pulso)))
            }
        case .preocupada:
            var gota = Path()
            gota.move(to: CGPoint(x: 156, y: 84))
            gota.addQuadCurve(to: CGPoint(x: 156, y: 102), control: CGPoint(x: 146, y: 98))
            gota.addQuadCurve(to: CGPoint(x: 156, y: 84), control: CGPoint(x: 166, y: 98))
            ctx.fill(gota, with: .color(Paleta.gota))
        case .celebrando:
            let colores: [UInt32] = [0xF2C14E, 0x7FC8A9, 0xE58FB0, 0x9C8FD6, 0xF59A3E, 0x7FC8A9]
            let sitios: [(Double, Double)] = [(30, 50), (160, 40), (50, 26), (140, 20), (178, 88), (14, 96)]
            for (i, (x, y)) in sitios.enumerated() {
                var c = ctx
                c.translateBy(x: x, y: y + sin(t * 4 + Double(i)) * 4)
                c.rotate(by: .degrees(t * 90 + Double(i) * 40))
                c.fill(Path(CGRect(x: -4, y: -2.5, width: 8, height: 5)), with: .color(Color(hex: colores[i])))
            }
        default:
            break
        }
    }
}
```

- [ ] **Step 3: Exportar icono y muestras**

```swift
// Sources/Aura/Icono.swift
import AppKit
import AuraNucleo
import SwiftUI

@MainActor
enum Icono {
    static func png(_ vista: some View, lado: CGFloat) -> Data? {
        let r = ImageRenderer(content: vista.frame(width: lado, height: lado))
        r.scale = 1
        guard let cg = r.cgImage else { return nil }
        return NSBitmapImageRep(cgImage: cg).representation(using: .png, properties: [:])
    }

    static func exportar(a url: URL) -> Bool {
        let vista = ZStack {
            RoundedRectangle(cornerRadius: 185, style: .continuous).fill(Color(hex: 0xEEE8FF)).padding(100)
            MascotaView(emocion: .feliz, animada: false).padding(170)
        }
        guard let datos = png(vista, lado: 1024) else { return false }
        return (try? datos.write(to: url)) != nil
    }

    static func muestras(en dir: URL) -> Bool {
        try? FileManager.default.createDirectory(at: dir, withIntermediateDirectories: true)
        return Emocion.allCases.allSatisfy { e in
            guard let datos = png(MascotaView(emocion: e, animada: false), lado: 240) else { return false }
            return (try? datos.write(to: dir.appendingPathComponent("\(e.rawValue).png"))) != nil
        }
    }
}
```

```swift
// Sources/Aura/main.swift  (B7 añade el arranque de la app debajo)
import AppKit
import AuraNucleo

_ = NSApplication.shared
let argumentos = CommandLine.arguments
if argumentos.count == 3, argumentos[1] == "--icono" {
    exit(MainActor.assumeIsolated { Icono.exportar(a: URL(fileURLWithPath: argumentos[2])) } ? 0 : 1)
}
if argumentos.count == 3, argumentos[1] == "--muestras" {
    exit(MainActor.assumeIsolated { Icono.muestras(en: URL(fileURLWithPath: argumentos[2])) } ? 0 : 1)
}
print("Aura: la app llega en la tarea B7")
```

- [ ] **Step 4: Verificar mirando las caras**

Run:

```bash
swift build && swift run Aura --muestras build/muestras && swift run Aura --icono build/icono.png && ls build/muestras
```

Expected: 9 PNG (`feliz.png` … `triste.png`) y `build/icono.png`. Ábrelas (`open build/muestras`) y
compáralas con el boceto aprobado:
- todas tienen el durazno rosado, la hoja, el visor oscuro y los ojos lavanda;
- cada emoción se distingue de un vistazo;
- `dormida` tiene "zzz", `revisando` burbujitas, `preocupada` una gota y `celebrando` confeti;
- nada se sale del lienzo.

Si algo se corta, ajusta las coordenadas en `adornos`, no la escala. `swift test` sigue verde.

- [ ] **Step 5: Commit**

```bash
git add Sources/Aura
git commit -m "feat: dibujo del durazno con sus nueve caras e icono

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task B7: La app: panel flotante, controlador, vigía y globito

**Files:**
- Create: `Sources/Aura/Delegado.swift`, `Sources/Aura/Controlador.swift`, `Sources/Aura/Vigia.swift`, `Sources/Aura/MascotaPanel.swift`, `Sources/Aura/VistaPanel.swift`, `Sources/Aura/GloboEstado.swift`
- Modify: `Sources/Aura/main.swift` (cambiar el `print` final por el arranque de la app)

**Interfaces:**
- Consumes:
  - `MotorCLI.predeterminado()`, `estado()`, `revisar(_:)`, `Diferencias.eventos`, `EstadoAura` (`recibir`, `tick`, `emocion`, `frase`, `fraseID`), `AccionSoltar.decidir`, `Vigilancia.relevante`, `Mirada.desplazamiento`, `nombreVisible` y `cositas` (B1–B5).
  - `MascotaView` y `Color(hex:)` (B6).
- Produces: el ejecutable `Aura`, que arranca como app accesoria (sin Dock) con el panel flotante. B8 lo empaqueta.

Esta parte es pegamento de AppKit y SwiftUI. Toda la lógica que se puede probar ya está en
`AuraNucleo`. La verificación se hace a mano en el Step 4.

- [ ] **Step 1: Vigía de carpetas (FSEvents)**

```swift
// Sources/Aura/Vigia.swift
import CoreServices
import Foundation

/// Avisa (en la cola principal) de cambios bajo `rutas`. La app decide si le importan.
final class Vigia {
    private var stream: FSEventStreamRef?
    private let alCambiar: ([String]) -> Void

    init?(rutas: [String], alCambiar: @escaping ([String]) -> Void) {
        self.alCambiar = alCambiar
        var contexto = FSEventStreamContext(version: 0, info: Unmanaged.passUnretained(self).toOpaque(),
                                            retain: nil, release: nil, copyDescription: nil)
        let callback: FSEventStreamCallback = { _, info, cuantos, rutas, _, _ in
            guard let info else { return }
            let vigia = Unmanaged<Vigia>.fromOpaque(info).takeUnretainedValue()
            let lista = (Unmanaged<CFArray>.fromOpaque(rutas).takeUnretainedValue() as? [String]) ?? []
            vigia.alCambiar(Array(lista.prefix(cuantos)))
        }
        let flags = FSEventStreamCreateFlags(kFSEventStreamCreateFlagUseCFTypes | kFSEventStreamCreateFlagFileEvents
                                             | kFSEventStreamCreateFlagNoDefer)
        guard let s = FSEventStreamCreate(nil, callback, &contexto, rutas as CFArray,
                                          FSEventStreamEventId(kFSEventStreamEventIdSinceNow), 1.0, flags) else { return nil }
        stream = s
        FSEventStreamSetDispatchQueue(s, .main)
        guard FSEventStreamStart(s) else {
            FSEventStreamInvalidate(s); FSEventStreamRelease(s); stream = nil
            return nil
        }
    }

    deinit {
        if let s = stream { FSEventStreamStop(s); FSEventStreamInvalidate(s); FSEventStreamRelease(s) }
    }
}
```

- [ ] **Step 2: Controlador**

```swift
// Sources/Aura/Controlador.swift
import AppKit
import AuraNucleo
import ServiceManagement

@MainActor
final class Controlador: ObservableObject {
    @Published private(set) var aura = EstadoAura(ahora: Date())
    @Published private(set) var motor: EstadoMotor?
    @Published var globoVisible = false {
        didSet { if globoVisible != oldValue { aura.recibir(globoVisible ? .globoAbierto : .globoCerrado, ahora: Date()) } }
    }
    weak var ventana: NSWindow?

    private let cli: MotorCLI
    private var vigia: Vigia?
    private var raizVigilada: String?
    private var relojes: [Timer] = []
    private var refrescoPendiente: DispatchWorkItem?
    private var refrescando = false
    private var otraVez = false
    private var colaRevisar: Task<Void, Never>?

    init(cli: MotorCLI) { self.cli = cli }

    func arrancar() {
        relojes = [
            Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
                MainActor.assumeIsolated { self?.aura.tick(ahora: Date()) }
            },
            Timer.scheduledTimer(withTimeInterval: 20, repeats: true) { [weak self] _ in
                MainActor.assumeIsolated { self?.programarRefresco(en: 0) }
            },
        ]
        programarRefresco(en: 0)
    }

    /// Agrupa avisos seguidos en un solo `videoqa estado` (cada llamada arranca Python).
    func programarRefresco(en segundos: Double = 1.5) {
        refrescoPendiente?.cancel()
        let item = DispatchWorkItem { [weak self] in Task { @MainActor in await self?.refrescar() } }
        refrescoPendiente = item
        DispatchQueue.main.asyncAfter(deadline: .now() + segundos, execute: item)
    }

    private func refrescar() async {
        if refrescando { otraVez = true; return }
        refrescando = true
        defer { refrescando = false }
        repeat {
            otraVez = false
            switch await cli.estado() {
            case .success(let nuevo):
                let ahora = Date()
                aura.recibir(.motorBien, ahora: ahora)
                for e in Diferencias.eventos(antes: motor, despues: nuevo) { aura.recibir(e, ahora: ahora) }
                vigilar(nuevo.driveRoot)
                motor = nuevo
            case .failure(let e):
                aura.recibir(.errorMotor(e), ahora: Date())
            }
        } while otraVez
    }

    private func vigilar(_ raiz: String) {
        let real = URL(fileURLWithPath: raiz).resolvingSymlinksInPath().path
        guard real != raizVigilada else { return }
        raizVigilada = real
        let videoqaHome = FileManager.default.homeDirectoryForCurrentUser.appendingPathComponent(".videoqa").path
        vigia = Vigia(rutas: [real, videoqaHome]) { [weak self] rutas in
            MainActor.assumeIsolated {
                if rutas.contains(where: { Vigilancia.relevante($0, raiz: real) }) { self?.programarRefresco() }
            }
        }
    }

    func soltar(_ urls: [URL]) {
        let automatico = motor?.automatico ?? false
        for url in urls {
            switch AccionSoltar.decidir(url, automatico: automatico) {
            case .ignorar:
                aura.recibir(.noEsVideo, ahora: Date())
            case .copiarAEntrada(let u):
                copiarAEntrada(u)
            case .revisar(let u):
                let previa = colaRevisar
                let cli = self.cli
                colaRevisar = Task { @MainActor [weak self] in
                    await previa?.value
                    if case .failure(let e) = await cli.revisar(u) { self?.aura.recibir(.errorMotor(e), ahora: Date()) }
                    self?.programarRefresco(en: 0)
                }
            }
        }
        programarRefresco()
    }

    /// Regla 2 del equipo: el mismo nombre en 01_Entrada es una versión nueva, así que reemplaza.
    private func copiarAEntrada(_ origen: URL) {
        guard let raiz = motor?.driveRoot else { return }
        let destino = URL(fileURLWithPath: raiz).appendingPathComponent("01_Entrada").appendingPathComponent(origen.lastPathComponent)
        guard destino.standardizedFileURL != origen.standardizedFileURL else { return }
        do {
            if FileManager.default.fileExists(atPath: destino.path) { try FileManager.default.removeItem(at: destino) }
            try FileManager.default.copyItem(at: origen, to: destino)
        } catch {
            aura.recibir(.errorMotor(.fallo("no pude copiar \(origen.lastPathComponent)")), ahora: Date())
        }
    }

    func dormirUnaHora() { aura.recibir(.silenciar(hasta: Date().addingTimeInterval(3600)), ahora: Date()) }

    var abreAlIniciar: Bool { SMAppService.mainApp.status == .enabled }

    func cambiarAbrirAlIniciar() {
        do {
            if abreAlIniciar { try SMAppService.mainApp.unregister() } else { try SMAppService.mainApp.register() }
        } catch {
            let alerta = NSAlert()
            alerta.messageText = "No pude cambiarlo"
            alerta.informativeText = "Puedes hacerlo en Ajustes del Sistema → General → Ítems de inicio."
            alerta.runModal()
        }
        objectWillChange.send()
    }

    func abrir(_ r: ResultadoVideo) {
        let carpeta = URL(fileURLWithPath: r.carpeta)
        let html = carpeta.appendingPathComponent("reporte.html")
        NSWorkspace.shared.open(FileManager.default.fileExists(atPath: html.path) ? html : carpeta.appendingPathComponent("reporte.md"))
    }

    /// Centro del durazno en coordenadas de pantalla (el durazno ocupa la parte baja del panel).
    func centroDelDurazno() -> CGPoint? {
        ventana.map { CGPoint(x: $0.frame.midX, y: $0.frame.minY + 75) }
    }
}
```

- [ ] **Step 3: Panel, vista, globito, delegado y arranque**

```swift
// Sources/Aura/MascotaPanel.swift
import AppKit
import SwiftUI

enum FaseArrastre { case cambio, fin }

enum Posicion {
    static func cargar(tamano: NSSize) -> NSPoint {
        let d = UserDefaults.standard
        if d.object(forKey: "auraX") != nil {
            let p = NSPoint(x: d.double(forKey: "auraX"), y: d.double(forKey: "auraY"))
            // Si la pantalla donde estaba ya no existe (monitor desconectado), vuelve a la esquina.
            if NSScreen.screens.contains(where: { $0.visibleFrame.insetBy(dx: -tamano.width / 2, dy: -tamano.height / 2).contains(p) }) {
                return p
            }
        }
        let v = NSScreen.main?.visibleFrame ?? NSRect(x: 0, y: 0, width: 1440, height: 900)
        return NSPoint(x: v.maxX - tamano.width - 24, y: v.minY + 24)
    }

    static func guardar(_ p: NSPoint) {
        UserDefaults.standard.set(p.x, forKey: "auraX")
        UserDefaults.standard.set(p.y, forKey: "auraY")
    }
}

/// Ventana transparente, sin bordes, siempre encima y que no activa la app al tocarla.
final class MascotaPanel: NSPanel {
    static let tamano = NSSize(width: 230, height: 270)
    private var inicioArrastre: (raton: NSPoint, origen: NSPoint)?

    init(controlador: Controlador) {
        super.init(contentRect: NSRect(origin: Posicion.cargar(tamano: Self.tamano), size: Self.tamano),
                   styleMask: [.borderless, .nonactivatingPanel], backing: .buffered, defer: false)
        isOpaque = false
        backgroundColor = .clear
        hasShadow = false
        level = .floating
        collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary, .stationary, .ignoresCycle]
        hidesOnDeactivate = false
        isMovable = false
        contentView = NSHostingView(rootView: VistaPanel(controlador: controlador, mover: { [weak self] in self?.mover($0) }))
    }

    override var canBecomeKey: Bool { true }
    override var canBecomeMain: Bool { false }

    private func mover(_ fase: FaseArrastre) {
        let raton = NSEvent.mouseLocation
        switch fase {
        case .cambio:
            if inicioArrastre == nil { inicioArrastre = (raton, frame.origin) }
            guard let i = inicioArrastre else { return }
            setFrameOrigin(NSPoint(x: i.origen.x + raton.x - i.raton.x, y: i.origen.y + raton.y - i.raton.y))
        case .fin:
            inicioArrastre = nil
            Posicion.guardar(frame.origin)
        }
    }
}
```

```swift
// Sources/Aura/VistaPanel.swift
import AppKit
import AuraNucleo
import SwiftUI

struct VistaPanel: View {
    @ObservedObject var controlador: Controlador
    let mover: (FaseArrastre) -> Void
    @State private var fraseVisible: String?
    @State private var encima = false

    var body: some View {
        VStack(spacing: 6) {
            Spacer(minLength: 0)
            if let f = fraseVisible {
                Text(f)
                    .font(.system(size: 13, weight: .medium, design: .rounded))
                    .foregroundStyle(Color(hex: 0x2B2340))
                    .multilineTextAlignment(.center)
                    .padding(.horizontal, 12).padding(.vertical, 8)
                    .background(RoundedRectangle(cornerRadius: 14, style: .continuous).fill(Color.white.opacity(0.96)))
                    .overlay(RoundedRectangle(cornerRadius: 14, style: .continuous).stroke(Color(hex: 0xCDBEFF), lineWidth: 1))
                    .frame(maxWidth: 210)
                    .transition(.opacity.combined(with: .scale(scale: 0.9, anchor: .bottom)))
            }
            MascotaView(emocion: controlador.aura.emocion, mirada: { mirada() })
                .frame(width: 136, height: 150)
                .scaleEffect(encima ? 1.08 : 1)
                .contentShape(Rectangle())
                .onTapGesture { controlador.globoVisible.toggle() }
                .gesture(DragGesture(minimumDistance: 3, coordinateSpace: .global)
                    .onChanged { _ in mover(.cambio) }
                    .onEnded { _ in mover(.fin) })
                .onDrop(of: [.fileURL], isTargeted: $encima) { proveedores in cargar(proveedores); return true }
                .popover(isPresented: $controlador.globoVisible, arrowEdge: .top) { GloboEstado(controlador: controlador) }
                .contextMenu {
                    Button("Dormir 1 hora") { controlador.dormirUnaHora() }
                    Button(controlador.abreAlIniciar ? "✓ Abrir al iniciar sesión" : "Abrir al iniciar sesión") {
                        controlador.cambiarAbrirAlIniciar()
                    }
                    Divider()
                    Button("Salir") { NSApp.terminate(nil) }
                }
        }
        .frame(width: MascotaPanel.tamano.width, height: MascotaPanel.tamano.height)
        .animation(.easeOut(duration: 0.25), value: fraseVisible)
        .animation(.spring(duration: 0.25), value: encima)
        .onChange(of: controlador.aura.fraseID) { _, _ in mostrarFrase() }
        .onChange(of: controlador.aura.frase) { _, nueva in if nueva == nil { fraseVisible = nil } }
        .onAppear { mostrarFrase() }
    }

    private func mostrarFrase() {
        let id = controlador.aura.fraseID
        fraseVisible = controlador.aura.frase
        guard fraseVisible != nil else { return }
        DispatchQueue.main.asyncAfter(deadline: .now() + 7) {
            if controlador.aura.fraseID == id { fraseVisible = nil }
        }
    }

    private func mirada() -> CGVector {
        guard let centro = controlador.centroDelDurazno() else { return .zero }
        return Mirada.desplazamiento(raton: NSEvent.mouseLocation, centro: centro)
    }

    private func cargar(_ proveedores: [NSItemProvider]) {
        for p in proveedores {
            _ = p.loadObject(ofClass: URL.self) { url, _ in
                guard let url else { return }
                DispatchQueue.main.async { controlador.soltar([url]) }
            }
        }
    }
}
```

```swift
// Sources/Aura/GloboEstado.swift
import AuraNucleo
import SwiftUI

struct GloboEstado: View {
    @ObservedObject var controlador: Controlador

    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            Text("Aura 🍑").font(.system(.headline, design: .rounded))
            if let m = controlador.motor {
                if let c = m.enCurso {
                    Label("Revisando «\(nombreVisible(c.video))»", systemImage: "magnifyingglass")
                }
                Label(m.pendientes.isEmpty ? "No hay nada pendiente" : "Pendientes: \(m.pendientes.count)", systemImage: "tray")
                if !m.recientes.isEmpty {
                    Divider()
                    Text("Últimos").font(.subheadline).foregroundStyle(.secondary)
                    ForEach(Array(m.recientes.prefix(5)), id: \.clave) { r in
                        HStack(spacing: 8) {
                            Text(icono(r))
                            Text(nombreVisible(r.video)).lineLimit(1).truncationMode(.middle)
                            Spacer()
                            Text(detalle(r)).font(.caption).foregroundStyle(.secondary)
                            Button("Abrir") { controlador.abrir(r) }.buttonStyle(.link)
                        }
                    }
                }
                Divider()
                Text(m.automatico ? "Revisión automática: encendida"
                                  : "Revisión automática: apagada. Suéltame un video y lo reviso.")
                    .font(.caption).foregroundStyle(.secondary)
            } else {
                Text(controlador.aura.frase ?? "Preguntando cómo va todo…").foregroundStyle(.secondary)
            }
        }
        .padding(16)
        .frame(width: 300)
    }

    private func icono(_ r: ResultadoVideo) -> String {
        switch r.estado { case .aprobado: "🟢"; case .conErrores: "🔴"; case .pendiente: "⏸️" }
    }

    private func detalle(_ r: ResultadoVideo) -> String {
        switch r.estado { case .aprobado: "listo"; case .conErrores: cositas(r.cositas); case .pendiente: "pendiente" }
    }
}
```

```swift
// Sources/Aura/Delegado.swift
import AppKit
import AuraNucleo

@MainActor
final class Delegado: NSObject, NSApplicationDelegate {
    private let controlador = Controlador(cli: .predeterminado())
    private var panel: MascotaPanel?

    func applicationDidFinishLaunching(_ notification: Notification) {
        let panel = MascotaPanel(controlador: controlador)
        controlador.ventana = panel
        panel.orderFrontRegardless()
        self.panel = panel
        controlador.arrancar()
    }
}
```

En `Sources/Aura/main.swift`, cambia la última línea (`print("Aura: …")`) por:

```swift
let app = NSApplication.shared
let delegado = MainActor.assumeIsolated { Delegado() }
app.delegate = delegado
app.setActivationPolicy(.accessory)
app.run()
```

- [ ] **Step 4: Compilar y probar a mano**

Run: `swift build && swift test && swift run Aura`
Expected: compila sin errores, los tests siguen verdes y aparece el durazno abajo a la derecha,
sin icono en el Dock.

Con la app abierta y tu `~/Desktop/videos` real, comprueba:
1. Saluda con un globito que desaparece en unos 7 s.
2. Al hacer clic en el espacio transparente que rodea al durazno, el clic llega a la ventana de
   detrás.
3. Al arrastrar el durazno se mueve. Ciérrala (clic derecho → Salir), ábrela otra vez y aparece
   en el mismo sitio.
4. Al hacerle clic se abre el globito de estado. "Abrir" muestra el reporte del video.
5. Mientras escribes en otra app, que Aura cambie de cara no te quita el foco.
6. Suelta un `.txt` encima y dice "Eso no es un video". Suelta un `.mp4` corto: se sorprende,
   pasa a revisando y al terminar celebra o se preocupa.
7. Tras 2 min sin actividad se duerme con "zzz". "Dormir 1 hora" la duerme al instante.
8. Memoria y CPU en reposo:
   `ps -o rss=,%cpu= -p $(pgrep -x Aura)`. Lo esperado es menos de 60000 KB y menos de 3% de CPU.

Corrige lo que falle antes de seguir.

- [ ] **Step 5: Commit**

```bash
git add Sources/Aura
git commit -m "feat: Aura flota en el escritorio y reacciona a la revisión

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task B8: Empaquetar `Aura.app`, verificación final e integración

**Files:**
- Create: `Recursos/Info.plist`, `scripts/empaquetar.sh`
- Modify: `README.md`

**Interfaces:**
- Consumes: el ejecutable `Aura` y `Aura --icono` (B6, B7).
- Produces: `build/Aura.app`, que la skill `/aura:mascota` (A3) copia a `~/Applications`.

- [ ] **Step 1: Info.plist**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CFBundleIdentifier</key><string>com.hongs05.aura</string>
  <key>CFBundleName</key><string>Aura</string>
  <key>CFBundleDisplayName</key><string>Aura</string>
  <key>CFBundleExecutable</key><string>Aura</string>
  <key>CFBundleIconFile</key><string>Aura</string>
  <key>CFBundlePackageType</key><string>APPL</string>
  <key>CFBundleShortVersionString</key><string>0.1.0</string>
  <key>CFBundleVersion</key><string>1</string>
  <key>LSMinimumSystemVersion</key><string>14.0</string>
  <key>LSUIElement</key><true/>
  <key>NSHumanReadableCopyright</key><string>Hecha con cariño 🍑</string>
</dict>
</plist>
```

- [ ] **Step 2: Script de empaquetado**

```bash
#!/usr/bin/env bash
# scripts/empaquetar.sh — compila y arma build/Aura.app (firma ad hoc; no necesita cuenta de Apple).
set -euo pipefail
cd "$(dirname "$0")/.."

swift build -c release
BIN=".build/release/Aura"
APP="build/Aura.app"
ICONSET="build/Aura.iconset"

rm -rf "$APP" "$ICONSET"
mkdir -p "$APP/Contents/MacOS" "$APP/Contents/Resources" "$ICONSET"
cp "$BIN" "$APP/Contents/MacOS/Aura"
cp Recursos/Info.plist "$APP/Contents/Info.plist"

"$BIN" --icono build/icono.png
for s in 16 32 128 256 512; do
  sips -z "$s" "$s" build/icono.png --out "$ICONSET/icon_${s}x${s}.png" >/dev/null
  d=$((s * 2))
  sips -z "$d" "$d" build/icono.png --out "$ICONSET/icon_${s}x${s}@2x.png" >/dev/null
done
iconutil -c icns "$ICONSET" -o "$APP/Contents/Resources/Aura.icns"

codesign --force --sign - "$APP"
echo "$APP"
```

Run: `chmod +x scripts/empaquetar.sh && bash scripts/empaquetar.sh`
Expected: imprime `build/Aura.app`; `codesign -dv build/Aura.app` dice `Signature=adhoc`.

- [ ] **Step 3: README**

Añade al final de `README.md`:

````markdown
## Compilar y probar

```bash
swift test                     # lógica (núcleo)
swift run Aura                 # abrir la mascota en desarrollo
swift run Aura --muestras build/muestras   # las 9 caras en PNG
bash scripts/empaquetar.sh     # build/Aura.app
```

En la Mac de la usuaria, la instala `/aura:mascota` (plugin `aura`, VideoQA ≥ 1.6.0), que necesita
`videoqa estado --json`.
````

- [ ] **Step 4: Verificación final con la app empaquetada**

```bash
pkill -x Aura || true
ditto build/Aura.app ~/Applications/Aura.app && open ~/Applications/Aura.app
```

Repite los puntos 1–8 de B7 Step 4 con la app empaquetada. Además:
- El icono del durazno aparece en Finder (`~/Applications`).
- Clic derecho → "Abrir al iniciar sesión" la añade en Ajustes del Sistema → General → Ítems de
  inicio. Si macOS lo rechaza por la firma ad hoc, sale el aviso con la ruta manual: anótalo
  como limitación, no lo bloquees.
- Con la rama de `videoqa` instalada (`cd ~/videoqa && git checkout feat/aura-estado && uv sync --extra macos`),
  todo lo anterior funciona contra tus carpetas reales.

- [ ] **Step 5: Commit**

```bash
git add Recursos scripts README.md
git commit -m "feat: empaquetado de Aura.app con icono y firma ad hoc

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 6: Integrar las dos ramas**

Usa superpowers:finishing-a-development-branch en cada repo. Primero `videoqa` (`feat/aura-estado` → `main`, versión 1.6.0) y después
`aura-app` (`feat/fase1-mascota` → `main`). Antes de que ella ejecute `/aura:mascota`, resuelve el
acceso al repo privado (ver la nota de A3).

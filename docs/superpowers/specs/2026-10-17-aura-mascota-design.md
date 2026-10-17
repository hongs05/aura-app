# Aura mascota — diseño

Fecha: 2026-10-17 · Estado: aprobado por partes en conversación, pendiente de revisión escrita

## 1. Para quién y para qué

**Usuaria:** la esposa del autor. Revisa y edita videos para redes en una agencia y no es técnica.
Ya usa VideoQA a través del plugin `aura` de Claude Code (`hongs05/videoqa`, versión 1.5.0).

**Qué es:** una compañera de escritorio para su Mac. Es un durazno animado que flota en una
esquina, muestra emociones según lo que pasa con sus videos y, con un clic, abre una ventana para
conversar, revisar y editar.

**Criterio de éxito:** que ella quiera tenerla abierta todos los días. Debe ser tierna, acogedora
y no estorbar. El trabajo de revisión y edición sigue funcionando aunque el chat no esté
disponible.

**Restricciones:**
- Su Mac tiene **8 GB de RAM** y chip Apple. Cada megabyte cuenta, y un modelo local tiene que
  ser pequeño y cargarse solo cuando se use.
- Todo en español, de tú.
- La revisión con Claude se paga con un plan Max. El objetivo es **gastar ≈ $50 al mes de uso
  equivalente** (unos 40 videos por semana).
- No se paga la cuenta de desarrollador de Apple ($99 al año): la app se compila en su Mac.

## 2. Decisiones tomadas

| Tema | Decisión | Por qué |
|---|---|---|
| Personaje | "Hermanita de Jico": un **durazno** rosado con visor oscuro y ojos lavanda | Hereda el estilo de Jico (Jicaro) pero es solo de ella; el lavanda encaja con el nombre "Aura" |
| Dónde vive | **Mascota de escritorio** flotante; con un clic se abre la ventana de chat y edición | Es lo que más se parece a "una compañera" |
| App | **Swift nativa** (SwiftUI + `NSPanel`) | Unos 40 MB de RAM; transparencia, "siempre encima", arrastrar y soltar y notificaciones son nativos |
| Animación | Dibujada en **código SwiftUI** detrás de un protocolo `MascotaView`; Rive opcional más adelante | No depende de alguien que sepa usar el editor de Rive |
| Motor | `videoqa` en Python, sin cambios de arquitectura. La app lo llama por línea de comandos | Reutiliza Whisper, OCR, juez, aprendizaje y respaldo local |
| Cerebro del chat | **Intercambiable**: Claude Sonnet 5.5 con esfuerzo bajo; si no hay uso o sesión, `qwen3.5:4b` en Ollama, cargado solo al escribir | Calidad sin trabar la Mac de 8 GB; el chat no se come el presupuesto del juez |
| Juez de revisión | Sigue en **Opus** (videoqa 1.5.0, juez ligero, ≈ $0,41 por video medido) | Elegido por el autor por ahora |
| Repos | **`aura-app`** (este repo) para la app; los comandos nuevos del motor van en **`videoqa`** | La app es independiente; la lógica de modelos y video ya vive en Python |
| Distribución | Se compila en su Mac con `swift build` desde un comando del plugin | Sin firma de Apple ni bloqueo de Gatekeeper |

## 3. Arquitectura

```
┌─────────────── aura-app (Swift, este repo) ───────────────┐
│  MascotaPanel (NSPanel flotante)   VentanaAura (SwiftUI)  │
│        │  emociones                    │ chat, editar     │
│  EstadoAura (máquina de estados) ◀── Eventos              │
│        ▲                                   │              │
│  Vigía (FSEvents)                  MotorCLI (Process)     │
└────────┼───────────────────────────────────┼──────────────┘
         │ carpetas de Drive + ~/.videoqa/jobs│ videoqa … --json
┌────────┴────────── videoqa (Python, otro repo) ───────────┐
│  run · estado · charlar · editar plan / aplicar           │
│  juez (claude -p / Opus) · cerebro (Sonnet → Ollama)      │
│  ffmpeg · Whisper · OCR · aprendizaje                     │
└───────────────────────────────────────────────────────────┘
```

Unidades de `aura-app`, cada una con una sola responsabilidad:

- **`EstadoAura`**: máquina de estados pura (sin UI ni E/S). Entra un evento y sale una emoción
  más una frase. Es lo más testeado.
- **`Vigia`**: observa con FSEvents `01_Entrada`, `02_Con_errores`, `03_Aprobado` y
  `~/.videoqa/jobs/*/state.json`, y produce eventos (`videoLlego`, `revisionEmpezo`,
  `revisionTermino(estado, n)`).
- **`MotorCLI`**: la única pieza que ejecuta procesos. Llama a
  `uv run --project ~/videoqa videoqa <comando> --json`, decodifica el JSON y traduce los errores.
- **`MascotaView`**: dibuja el durazno y anima la cara según la emoción. Es un protocolo con una
  implementación SwiftUI; Rive sería otra.
- **`MascotaPanel`**: el `NSPanel` transparente, sin bordes y siempre encima, que nunca roba el
  foco.
- **`VentanaAura`**: chat, tarjetas de acción con botón "Sí, hazlo" y progreso de la edición.
- **`Frases`**: libro de frases en español con varias variantes por situación. No usa IA.

## 4. Parte 1: Mascota y emociones

**Personaje** (colores de partida, salen del boceto aprobado):

| | |
|---|---|
| Cuerpo | `#F7B9AE`, con un brillo `#FBD3C7` y la línea del durazno en `#E7998D` |
| Mejillas | `#F29C9C` |
| Hoja | `#8CC08A` |
| Tallo | `#8A5A3B` |
| Visor | `#2B2340` |
| Ojos y boca | `#CDBEFF` (lavanda) |

Tamaño en pantalla: unos 120 px.

**Estados y disparadores:**

| Evento | Emoción | Comportamiento |
|---|---|---|
| Arranque, o despierta sin nada pendiente | `feliz` | Ojos `^ ^` y sonrisa: "Hola, ¿revisamos algo juntas?" |
| 2 min sin actividad (y ninguna revisión en curso) | `dormida` | Ojos `﹀﹀`, "zzz", respiración lenta; bosteza antes de dormirse |
| Archivo nuevo en `01_Entrada` | `sorprendida` → `revisando` | Ojos redondos, luego ojos a un lado con burbujitas de pensamiento |
| Revisión terminada con errores | `preocupada` | Cejas tiernas y globito: "Encontré N cositas 🥺" |
| Revisión terminada en verde | `celebrando` | Salta, confeti y "¡Listo para publicar!" |
| Edición en curso | `editando` | Teclea en su laptop y muestra el % |
| Ventana abierta | `escuchando` | Mira hacia la ventana |
| Error del motor | `triste` | Frase corta con qué hacer |

**Vida:** parpadeo con intervalos al azar (3–7 s), ojos que siguen el cursor, hoja que se mece
y respiración suave. Todo se desactiva si el sistema pide reducir el movimiento.

**Interacción:**
- Clic: abre la ventana.
- Arrastrar: la mueve y recuerda la posición.
- Soltar un video encima: lo copia a `01_Entrada` y, si la revisión automática está apagada,
  ejecuta `videoqa run`.
- Clic derecho: "Dormir 1 hora", "Abrir al iniciar sesión" y "Salir".

## 5. Parte 2: Edición (comandos nuevos en `videoqa`)

Módulo nuevo `videoqa/editar/`:
- `plan.py` construye un plan en JSON desde los artefactos del trabajo. No toca el video.
- `operaciones.py` contiene funciones puras que devuelven tramos o filtros.
- `render.py` es lo único que ejecuta ffmpeg.

| Operación | Datos | Qué hace |
|---|---|---|
| `silencios` | `technical.json` | Corta silencios de ≥ 2 s y deja 0,25 s de aire |
| `negros` | `technical.json` | Recorta pantallas negras y congeladas |
| `bloopers` | `findings_final.json` (tipo `blooper` con tramo) | Corta tomas fallidas |
| `subtitulos` | `transcript.json` | `.srt` y, opcionalmente, subtítulos quemados |
| `volumen` | — | `loudnorm` a −14 LUFS |
| `vertical` | `probe.json` | Recorte central o relleno desenfocado a 1080×1920 |

**Cambios en lo que ya existe en `videoqa`:**
- Activar `word_timestamps=True` en la transcripción para que los cortes caigan entre palabras.
- Un candado para que revisar y editar no se ejecuten a la vez.

**Comandos:**
- `videoqa editar plan <video> --ops silencios,bloopers --json` devuelve
  `{ "plan_id", "resumen", "tramos": [...], "duracion_antes", "duracion_despues" }`.
- `videoqa editar aplicar <plan_id> --json` emite líneas de progreso
  `{ "progreso": 0.42 }` y una final `{ "ok": true, "salida": "..." }`. Codifica con
  `h264_videotoolbox`.

**Seguridad:**
- El original se copia **antes** de cualquier cambio a `~/.videoqa/originales/<video>/vN.<ext>`.
- El resultado vuelve a `01_Entrada` **con el mismo nombre** (regla 2 del equipo), así que se
  revisa solo.
- Si ffmpeg falla, el original queda intacto y Aura se pone `triste`.

**No se arregla automáticamente:** la ortografía en pantalla, los colores de marca y el texto
tapado o que dura poco. Aura los entrega como **lista de arreglos a mano** (segundo, qué cambiar,
color correcto y foto). Ver el punto 4 de la Parte 5.

## 6. Parte 3: Cerebro y chat (comando nuevo en `videoqa`)

`videoqa charlar --json` recibe por stdin `{ "mensaje": "...", "historial": [...] }`, con como
mucho los últimos 10 turnos que guarda la app. Devuelve:

```json
{ "texto": "¡Ya lo vi! Tiene 2 silencios largos 🍑",
  "emocion": "revisando",
  "accion": { "tipo": "editar", "video": "promo_octubre", "ops": ["silencios"] },
  "cerebro": "claude" }
```

- `emocion` sale de la lista de emociones de la Parte 1.
- `accion` es `null` o una de estas: `revisar`, `estado`, `explicar_reporte`, `editar`,
  `corregir_hallazgo`, `revisar_texto` o `abrir_carpeta`.
- Se valida en Python contra un esquema. Una acción desconocida o mal formada se descarta y se
  queda solo el texto.
- **La app nunca ejecuta una acción sin el botón "Sí, hazlo".**

**Contexto que recibe el modelo:**
- Personalidad: tierna, acogedora, breve, en español, de tú, emojis con moderación.
- Un resumen del estado (cola y últimos 5 resultados).
- El historial.

Objetivo: unos 3–5 mil tokens por mensaje.

**Cadena de cerebros:**
1. Claude con `claude -p` en modo ligero (`--tools ""` y un `--system-prompt` propio), con
   `--model claude-sonnet-5-5 --effort low`. El modelo es configurable en `reglas.yaml` con
   `chat.modelo` y `chat.esfuerzo`.
2. Si hay `UsageLimitError` o la sesión caducó, pasa a Ollama con `juez_respaldo.modelo`
   (`qwen3.5:4b`), `keep_alive: 60s` y salida JSON.
3. Si no hay Ollama, devuelve `{"texto": "Ahora no puedo charlar 🥺, pero los botones siguen
   funcionando.", "emocion": "triste", "accion": null, "cerebro": "ninguno"}`.

El historial vive en `~/.videoqa/aura/chat.jsonl` y es de la app.

## 7. Parte 4: Instalación y distribución

- Requiere macOS 14 o superior, igual que VideoQA, y chip Apple.
- **Paquete Swift** en este repo, con un script `scripts/empaquetar.sh` que arma `Aura.app`
  (Info.plist, icono y `LSUIElement=1` para que no aparezca en el Dock).
- **Comando nuevo del plugin**, `/aura:mascota`, en `videoqa`:
  1. Clona o actualiza `aura-app` en `~/aura-app`.
  2. Ejecuta `swift build -c release` y `scripts/empaquetar.sh`.
  3. Copia el resultado a `~/Applications/Aura.app` y la abre.

  `/aura:actualizar` repite estos pasos si la app está instalada.
- **Primer arranque:** comprueba `~/videoqa`, `uv` y `videoqa doctor`. Si falta algo, Aura se
  pone `triste` y dice el paso concreto. Pide permiso de notificaciones. "Abrir al iniciar
  sesión" usa `SMAppService.mainApp`.
- **Revisión automática:** si el servicio launchd `com.videoqa.watcher` está activo, la app
  **solo observa**. Si no, soltar un video lanza `videoqa run`. Nunca hay dos revisiones a la vez.
- **Comando nuevo `videoqa estado --json`:** cola, revisión en curso, últimos resultados y si la
  automática está encendida. La app lo llama al arrancar y cuando el Vigía avisa de cambios.

## 8. Contrato entre repos

| Comando (`videoqa`) | Fase | Estado |
|---|---|---|
| `run <video>` | 1 | Existe |
| `estado --json` | 1 | Nuevo |
| `charlar --json` | 2 | Nuevo |
| `editar plan … --json` / `editar aplicar … --json` | 3 | Nuevo |

Cada salida JSON lleva `"version": 1`. La app comprueba la versión y, si no la entiende, pide
actualizar en vez de fallar en silencio.

## 9. Errores

| Situación | Qué ve ella |
|---|---|
| No existe `~/videoqa` o `uv` | Aura `triste`: "Me falta el motor. Escribe /aura:instalar en Claude." |
| Sesión de Claude caducada | Revisar: los videos se quedan pendientes (comportamiento actual). Chat: pasa a Ollama |
| Sin uso de Claude | El juez espera o usa el respaldo, según `reglas.yaml`. El chat pasa a Ollama |
| ffmpeg falla al editar | Original intacto y "No pude editarlo, ¿lo intento otra vez?" |
| JSON del motor de una versión que no entiende | "Hay que poner al día a Aura: escribe /aura:actualizar." |

## 10. Tests

- **Swift (XCTest):**
  - `EstadoAura`: cada evento produce su emoción, no se duerme durante una revisión, el tiempo
    de dormir con un reloj falso.
  - `MotorCLI`: decodifica salidas reales guardadas como fixtures y rechaza versiones
    desconocidas.
  - `Frases`: no repite la misma variante dos veces seguidas.
- **Python (pytest, en `videoqa`):**
  - `editar`: cada operación con trabajos de prueba y un render real con un video sintético de
    5 s (`slow`).
  - `charlar`: con un cerebro falso, validación del esquema y la cadena de respaldo.
  - `estado --json`.
- **Manual, antes de entregar:** transparencia y clics sobre la mascota, que no robe el foco,
  soltar un video, la secuencia dormir → despertar → celebrar, el consumo de RAM (< 60 MB en
  reposo) y el primer arranque en una cuenta limpia.

## 11. Fases

1. **Mascota viva:** `EstadoAura`, `Vigia`, `MascotaView`, `MascotaPanel`, `Frases`,
   `estado --json`, soltar videos, empaquetado y `/aura:mascota`.
2. **Chat:** `VentanaAura` (chat y tarjetas de acción) y `charlar`.
3. **Edición:** `editar` en `videoqa` y su interfaz en la ventana.
4. **Extras:** corrector previo de textos (`revisar_texto`) y, según el programa de edición que
   use, marcadores FCPXML para Premiere o Final Cut, o la lista detallada para CapCut.

## 12. Fuera de alcance

- Editor con línea de tiempo (ya existen otros, como OpenChatCut).
- Arreglar texto quemado en el video con IA de inpainting (no cabe en 8 GB).
- Voz o habla de Aura.
- Windows (`videoqa-win` queda aparte).
- Presupuesto mensual automático del juez (propuesto el 2026-10-17 y aplazado).

## 13. Preguntas abiertas

- **¿Con qué programa edita ella?** Define el punto 4 de la fase 4.
- **¿Subir `aura-app` a GitHub (privado) y con qué nombre?** Hace falta para que `/aura:mascota`
  lo clone en su Mac.

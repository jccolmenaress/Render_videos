# Fase 1 — Diseño: Baseline en ComfyUI

> Diseño aprobado (a primera vista) el 2026-07-18. Documento vivo: se itera.
> Contexto general: `docs/00-roadmap.md`. Fuentes: `docs/investigacion/fase-1-fuentes.md`.
> **Estado:** diseño aprobado; pendiente revisar investigación antes de crear las tareas
> detalladas (plan de implementación).

---

## 1. Objetivo y alcance

Un **workflow manual en la interfaz de ComfyUI** que toma 1 video musical de baja calidad y
devuelve uno mejorado a **1080p**, aplicando **restauración → upscaling → interpolación de FPS**,
conservando el **audio original intacto**. Se arma y corre a mano por la GUI (nada programático
todavía). Calidad de referencia: WinXvideo AI.

**Fuera de alcance (Fase 2+):** backend, API, frontend, cola de trabajos. Mejora de audio
(el audio solo se conserva, no se procesa).

**Objetivo secundario:** aprender y documentar cada decisión para repasar/explicar en entrevistas.

---

## 2. Arquitectura del entorno (la base)

ComfyUI corre dentro de un **contenedor Docker** basado en `nvcr.io/nvidia/pytorch:25.11-py3`,
replicando el patrón ya probado del proyecto de Unsloth del usuario (mismo base image, mismo
`TORCH_CUDA_ARCH_LIST="12.1"` para compilar hacia sm_121).

```
┌─ DGX Spark (GB10, aarch64, sm_121) ─────────────────────┐
│  Docker: imagen comfyui-dgx (FROM nvcr pytorch 25.11)   │
│  ├─ ComfyUI + ffmpeg                                     │
│  ├─ PyTorch con soporte GB10 (sm_121) ✅                 │
│  └─ main.py --listen 0.0.0.0 --port 8188  → navegador   │
│                                                          │
│  Volúmenes montados (persisten en el host):              │
│  ├─ /models       (Real-ESRGAN, RIFE, etc.)             │
│  ├─ /input        (videos a procesar)                    │
│  ├─ /output       (videos mejorados)                     │
│  ├─ /workflows    (los .json del pipeline)               │
│  └─ /custom_nodes (nodos extra, ej. interpolación)       │
└─────────────────────────────────────────────────────────┘
```

**Por qué contenedor y no arreglar el venv:** el venv actual (`comfyui-env`) tiene
`torch 2.10.0+cu130`, que avisa que NO soporta sm_121 (máx sm_120) → riesgo de crashes o salida
corrupta. El contenedor NGC trae PyTorch compilado para la GB10. Además es reproducible, aislado
y reutiliza una skill que el usuario ya domina.

**Desarrollo ágil dentro del contenedor:** lo que cambia seguido (workflows `.json`, modelos,
videos, y hasta el código de ComfyUI) vive en el **host** y se **monta** como volumen → los
cambios se ven al instante, sin reconstruir la imagen. Solo se reconstruye al cambiar
dependencias. Para el día a día no se toca Docker.

**Nota de red/seguridad:** `--listen 0.0.0.0` abre ComfyUI a la red local. Para la Fase 1 (uso
en la misma máquina/LAN) está bien; el endurecimiento (firewall, auth) es tema de la Fase 3.

---

## 3. El pipeline de ComfyUI (orden final de procesamiento)

```
Video entra
   │
   ▼
[Decodificar a frames]     ← nodos NATIVOS de ComfyUI (LoadVideo / GetVideoComponents)
   │
   ▼
[1. Restauración]          ← limpiar ruido/compresión ANTES de escalar
   │
   ▼
[2. Upscaling]             ← Real-ESRGAN → llevar a 1080p
   │
   ▼
[3. Interpolación FPS]     ← RIFE → movimiento fluido (ej. 24→60 fps)
   │
   ▼
[Recodificar a video]      ← H.264 / MP4 (CreateVideo)
   │
   ▼
[Remux del audio ORIGINAL] ← ffmpeg, copia de stream sin recomprimir
   │
   ▼
Video mejorado sale
```

**Decisión clave:** usar los **nodos de video nativos** de ComfyUI
(`LoadVideo` / `GetVideoComponents` / `CreateVideo`) en vez de VideoHelperSuite, para minimizar
custom nodes, que son el punto frágil en ARM64.

---

## 4. Orden de construcción (incremental)

Se construye y valida **una capa a la vez** (elección del usuario: mejor para aprender y depurar).
Cada paso deja un workflow `.json` guardado y funcionando antes de pasar al siguiente.

| Paso | Qué se arma | Riesgo | Resultado validable |
|---|---|---|---|
| **0** | Entorno: contenedor ComfyUI + ffmpeg; verificar que corre en la GB10 | Medio (sm_121) | ComfyUI abre y ejecuta un nodo simple sin errores |
| **1** | **Solo upscaling** (nativos + Real-ESRGAN) + remux audio | Bajo | Video 1080p más nítido, con su audio original |
| **2** | Agregar **restauración** antes del upscaling | Medio | Video más limpio (menos ruido/bloques) |
| **3** | Agregar **interpolación FPS** (RIFE) | Alto (cupy/ARM64) | Movimiento fluido a 60 fps |
| **4** | Ajuste final de salida (codec, resolución exacta) | Bajo | Salida pulida y consistente |

---

## 5. Modelos (a confirmar en detalle en cada sub-tarea)

- **Upscaling:** `RealESRGAN_x4plus.pth` (modelo general realista, para imagen real/personas;
  ~67 MB). Se coloca en `ComfyUI/models/upscale_models/`. Escala x4; se ajusta a 1080p exacto.
- **Restauración:** a definir en el Paso 2. Candidatos no-generativos: filtro de denoise/deblock
  de ffmpeg (`hqdn3d` / `nlmeans`) como pre-pass, o un modelo ESRGAN con denoise. Se investiga ahí.
- **Interpolación:** RIFE vía `ComfyUI-Frame-Interpolation` (Fannovel16). Si `cupy` da problemas
  en ARM64, plan B: `ComfyUI-Rife-Tensorrt`.

---

## 6. Salida y audio

- **Video:** H.264 en contenedor MP4 (compatibilidad universal).
- **Audio:** se extrae del original y se re-pega al final con ffmpeg **copiando el stream**
  (sin recomprimir → cero pérdida de calidad de sonido).

---

## 7. Testing / Definición de "hecho"

- Se procesa primero un **clip corto** (10–30 s) para validar el flujo rápido.
- Luego un **video completo** representativo (a conseguir del hermano del usuario).
- Criterios de aceptación:
  1. Resultado a **1080p visiblemente mejor** en una comparación antes/después.
  2. **Audio original intacto** y sincronizado.
  3. El proceso termina **sin errores**.
  4. Se registra el **tiempo de procesamiento** (dato clave para diseñar la Fase 2).

---

## 8. Limitaciones conocidas y riesgos

- **Flickering / consistencia temporal (IMPORTANTE):** aplicar un upscaler de imagen fija
  frame-por-frame (Real-ESRGAN) NO tiene conciencia temporal → puede aparecer parpadeo entre
  frames. Es una limitación conocida del enfoque baseline. Herramientas como WinXvideo AI/Topaz
  lo mitigan analizando grupos de frames. Si el parpadeo molesta, una iteración futura puede usar
  modelos de video dedicados (ej. SeedVR2) o procesar en chunks temporales. Para el baseline se
  acepta y se observa.
- **Memoria:** un video de varios minutos son miles de frames; cargarlos todos puede ser pesado
  aun con los 128 GB de la GB10. Mitigación: empezar con clips cortos; evaluar procesado por lotes.
- **cupy/ARM64:** la interpolación (Paso 3) puede requerir ajustes por la dependencia `cupy`.

---

## 9. Estructura de carpetas del proyecto

```
renderizar_videos/
├─ docs/
│  ├─ 00-roadmap.md
│  ├─ planeacion/
│  │  └─ fase-1-diseno.md              ← este documento
│  └─ investigacion/
│     └─ fase-1-fuentes.md             ← fuentes verificadas
├─ docker/
│  └─ Dockerfile                        (imagen ComfyUI para DGX)
├─ workflows/                           (los .json de cada paso)
├─ input/  output/  models/             (o montados desde la DGX)
└─ aprendizaje/                         (lecciones/aprendizajes por sesión;
                                         a futuro: HTML animado con d3/three.js/p5)
```

---

## 10. Próximo paso

Antes de crear el **plan de implementación detallado** (tareas y sub-tareas), el usuario va a:
1. Revisar `docs/investigacion/fase-1-fuentes.md` para empaparse del tema y detectar huecos.
2. (Opcional) Pasar el diseño por la skill de "grilling" para una segunda vuelta crítica.
3. Volver en otra sesión a discutir lo hallado y recién ahí generar las tareas detalladas.

# Fase 1 — Investigación y Fuentes

> Fuentes verificadas para el diseño de la Fase 1. Recopiladas y revisadas (2 pasadas) el
> 2026-07-18. Objetivo: que el usuario se empape del tema y detecte huecos antes de crear las
> tareas detalladas. Diseño relacionado: `docs/planeacion/fase-1-diseno.md`.

---

## 1. Hardware: DGX Spark (GB10 / sm_121 / aarch64)

**El punto más crítico.** La GB10 es Blackwell con compute capability **sm_121**; los PyTorch
"estándar" soportan hasta sm_120 → hay que usar un PyTorch/CUDA preparado (contenedor NGC
`nvcr.io/nvidia/pytorch:25.11-py3`, que es lo que el usuario ya usa para Unsloth).

Verificado en la máquina: `torch 2.10.0+cu130` en el venv actual **avisa** que la GB10 (sm_121)
excede su máximo soportado (sm_120) → riesgo de fallo/salida corrupta. De ahí la decisión del
contenedor.

- [natolambert/dgx-spark-setup](https://github.com/natolambert/dgx-spark-setup) — setup ML para DGX Spark (GB10, CUDA 13, aarch64).
- [ecarmen16/SparkyUI](https://github.com/ecarmen16/SparkyUI/) — ComfyUI + SageAttention adaptado a DGX Spark (GB10, ARM64, sm_121).
- [Triplany/comfyui-dgx-spark](https://github.com/Triplany/comfyui-dgx-spark) — kit de optimización + `verify.sh`. Recomienda torch 2.11.0+cu130 y ComfyUI master reciente. Lista de flags a evitar (`--gpu-only`, overrides fp16/bf16 globales, etc.).
- [Spark-comfyui — foro NVIDIA](https://forums.developer.nvidia.com/t/spark-comfyui-a-self-healing-comfyui-setup-for-dgx-spark/376542) — setup "auto-reparable".
- [Compatibilidad aarch64 — foro NVIDIA](https://forums.developer.nvidia.com/t/architecture-and-library-compatibility-on-aarch64/350389)
- [martimramos/dgx-spark-ml-guide](https://github.com/martimramos/dgx-spark-ml-guide) — guía PyTorch & CUDA para DGX Spark.
- [Soporte DGX Spark — foro PyTorch](https://discuss.pytorch.org/t/nvidia-dgx-spark-support/223677)

---

## 2. Upscaling (Real-ESRGAN)

- [Real-ESRGAN x4plus en ComfyUI (oficial)](https://comfy.org/p/supported-models/realesrgan-x4plus/) — página oficial del modelo soportado.
- [Descarga del modelo `RealESRGAN_x4plus.pth` (HuggingFace fofr/comfyui)](https://huggingface.co/fofr/comfyui/blob/main/upscale_models/RealESRGAN_x4plus.pth) — ~67 MB. Va en `ComfyUI/models/upscale_models/`.
- [Upscale Models — ComfyUI Wiki](https://comfyui-wiki.com/en/resource/upscale-models) — catálogo de modelos de magnificación.
- [The Complete AI Upscaling Handbook (blog oficial ComfyUI)](https://blog.comfy.org/p/upscaling-in-comfyui) — panorama general de upscaling.

---

## 3. Pipeline de video en ComfyUI (nodos)

- [Video upscaling in ComfyUI (tutorial oficial)](https://docs.comfy.org/tutorials/utility/video-upscale) — flujo con nodos nativos + `ImageUpscaleWithModel`.
- [Fast GAN Video Upscaler — workflow oficial](https://comfy.org/workflows/utility-gan_upscaler-b8c030205d86/) — workflow de ejemplo listo para descargar.
- [ComfyUI-VideoUpscale_WithModel (ShmuelRonen)](https://github.com/ShmuelRonen/ComfyUI-VideoUpscale_WithModel) — nodo *memory-efficient* para secuencias largas (útil si el video completo no entra en memoria).
- [Video Upscale & Restoration — Comfy with ComfyUI (nomadoor)](https://comfyui.nomadoor.net/en/ai-capabilities/video-upscale-restoration/) — explica bien el tema de consistencia temporal.

---

## 4. Restauración / Denoise / Deblock

- [AI Video Restoration and Enhancement — ComfyUI.org](https://comfyui.org/en/ai-video-restoration-and-enhancement) — restauración a nivel de frame.
- [Video Restoration — workflow oficial](https://comfy.org/workflows/558cc012978d-558cc012978d/)
- [Easy Video Upscaler + HD Detail Restoration (RunComfy)](https://www.runcomfy.com/comfyui-workflows/easy-video-upscaler-for-footage-in-comfyui-hd-detail-restoration)
- Nota: la restauración no-generativa (denoise/deblock clásico) puede hacerse como pre-pass con
  ffmpeg (`hqdn3d`, `nlmeans`). El deblock (artefactos de compresión) conviene "en bajo" para no
  borrar detalle.

---

## 5. Interpolación de FPS

- [Fannovel16/ComfyUI-Frame-Interpolation](https://github.com/Fannovel16/ComfyUI-Frame-Interpolation) — el set principal (RIFE/FILM). ⚠️ depende de `cupy` (posible fricción en ARM64).
- [ComfyUI-Rife-Tensorrt (yuvraj108c)](https://github.com/yuvraj108c/ComfyUI-Rife-Tensorrt) — plan B, RIFE con TensorRT (muy rápido).
- [ThunderFun/ComfyUI-RIFE-FILM-Only](https://github.com/ThunderFun/ComfyUI-RIFE-FILM-Only) — variante liviana (solo RIFE y FILM).

---

## 6. ComfyUI en Docker (headless + GPU)

- [ComfyUI Headless (docs oficiales)](https://mintlify.wiki/Comfy-Org/ComfyUI/guides/headless-mode) — `main.py --listen 0.0.0.0 --port 8188`.
- [mmartial/ComfyUI-Nvidia-Docker](https://github.com/mmartial/ComfyUI-Nvidia-Docker) — imagen ComfyUI para GPU NVIDIA con uid/gid configurable (buena referencia de Dockerfile/compose).
- [How to Run ComfyUI in Docker (oneuptime)](https://oneuptime.com/blog/post/2026-02-08-how-to-run-comfyui-in-docker-for-ai-image-generation/view)
- [ComfyUI with Docker: Reproducible GPU Inference (Markaicode)](https://markaicode.com/integrate/comfyui-with-docker/)
- Recordatorio: hace falta el **NVIDIA Container Toolkit** y `--gpus all`. Con `--listen 0.0.0.0`,
  cuidar firewall.

---

## 7. Referencia de calidad (la herramienta a igualar)

- [WinXvideo AI](https://www.winxdvd.com/es/winxvideo-ai/) — herramienta actual del hermano.
- [ComfyUI Topaz Video AI Nodes (Apatero, 2026)](https://apatero.com/blog/comfyui-topaz-video-enhancement-upscaling-2026) — cómo Topaz maneja upscaling/restauración por chunks de 16–32 frames para consistencia temporal (útil como referencia conceptual).

---

## 8. Huecos / preguntas abiertas para revisar antes de las tareas detalladas

1. **Consistencia temporal (flickering):** el baseline frame-por-frame puede parpadear. ¿Se
   acepta en la Fase 1 y se mejora después, o conviene evaluar ya un enfoque temporal? (El diseño
   asume: se acepta y se observa.)
2. **Modelo/método de restauración concreto:** ¿ffmpeg pre-pass vs nodo/modelo dedicado? Decidir
   al llegar al Paso 2.
3. **cupy en ARM64:** confirmar si `ComfyUI-Frame-Interpolation` compila/corre en la GB10 o si hay
   que ir directo al plan B (TensorRT).
4. **Memoria para videos largos:** ¿alcanza cargar todo en RAM unificada, o hay que usar el nodo
   *memory-efficient* / procesar por lotes desde el inicio?
5. **Resolución exacta:** Real-ESRGAN x4 sobre 480p da 1920×1440 aprox. → definir cómo ajustar a
   1080p exacto (downscale posterior, o escalar a factor no entero).
6. **ffmpeg dentro del contenedor:** confirmar que la imagen NGC lo trae o instalarlo en el
   Dockerfile.

---

## Cómo usar este documento

Leer las fuentes de la sección 1 (hardware) y 6 (Docker) primero — son las que desbloquean el
Paso 0. Las secciones 2–5 corresponden a cada capa del pipeline (se profundizan al llegar a cada
paso). La sección 8 son los puntos a cerrar antes de escribir las tareas detalladas.

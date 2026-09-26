# Roadmap — Renderizar / Mejorar Videos Musicales

> Documento de planificación de alto nivel. Define las 4 etapas del proyecto.
> Cada etapa tiene (o tendrá) su propio ciclo: **spec → plan → implementación**.
> Última actualización: 2026-06-23

---

## Objetivo del proyecto

Construir un sistema propio para **mejorar la calidad de videos musicales** (conciertos,
canciones; duración típica 1–10 min), reemplazando una herramienta comercial por un
pipeline propio que corre en hardware local.

Las mejoras a aplicar son:
- **Upscaling de resolución** (ej. SD → 1080p).
- **Restauración** (quitar ruido, artefactos de compresión, blur; des-entrelazado si aplica).
- **Interpolación de FPS** (ej. 24/25 fps → 60 fps para movimiento más fluido).

> Nota: NO se busca "mejora generativa" (re-inventar detalle con difusión) en el baseline.

### Objetivo secundario (igual de importante)
Aprender y **documentar cada decisión** para poder repasar y explicar el proyecto en una
entrevista técnica. Por eso planeamos por etapas y dejamos registradas las justificaciones.

---

## Contexto técnico

- **Servidor:** DGX Spark con superchip **NVIDIA GB10** (Grace-Blackwell, memoria unificada).
  Es la máquina que tiene la GPU y hace el procesamiento pesado.
- **ComfyUI:** instalado en `/home/okmisantos/Documentos/comfyui/ComfyUI`
  (venv en `comfyui-env`). Aún sin custom nodes de video ni modelos de upscale.
- **Carpeta del proyecto:** `/home/okmisantos/Documentos/comfyui/renderizar_videos`.
- **Herramienta de referencia (la que usa actualmente):** WinXvideo AI
  (https://www.winxdvd.com/es/winxvideo-ai/). El baseline debe igualar o superar su calidad.

---

## Decisiones de arquitectura ya tomadas

1. **Pipeline primero en ComfyUI como baseline**, antes de envolver nada en un backend.
   Hasta que el flujo no dé buena calidad a mano, no se construye servicio.
2. **Backend en FastAPI** que envuelve ese flujo y lo expone como endpoint.
3. **Procesamiento asíncrono (job queue), NO petición HTTP bloqueante.**
   Razón: el procesado frame-por-frame tarda minutos; una request HTTP no puede quedarse
   esperando. El flujo es: subir → recibir un `job_id` → consultar estado → descargar.
4. **Frontend = cliente liviano**, montable en cualquier PC local. No procesa; solo sube el
   video (binario, `multipart/form-data`), muestra progreso y descarga el resultado.
5. **Red (baseline):** PC local y DGX en la **misma LAN**; el frontend pega a la IP local de
   la DGX (ej. `http://192.168.x.x:8000`). Escenarios remotos (Tailscale/túnel) o exponer a
   internet quedan fuera del baseline.
6. **LangGraph: descartado.** El pipeline es determinista y lineal (video entra → pasos
   fijos → video sale). LangGraph sirve para orquestar agentes/LLMs con decisiones dinámicas;
   acá sería complejidad sin beneficio. ComfyUI ya ES el grafo de procesamiento.

---

## Las 4 etapas

### Fase 1 — Baseline en ComfyUI  *(siguiente a planear en detalle)*
**Meta:** un workflow de ComfyUI que toma 1 video y devuelve uno mejorado
(upscale + restauración + interpolación), con calidad comparable a WinXvideo AI.
Se corre a mano desde la interfaz de ComfyUI.

**Pendiente de definir en su spec:** resolución de salida objetivo, modelos concretos
(upscale/restauración/interpolación), custom nodes de video necesarios, manejo del audio
original (remux), formato/codec de salida.

**Definición de "hecho":** se procesa un video de prueba de principio a fin y el resultado
se ve claramente mejor, con el audio original intacto.

---

### Fase 2 — Backend FastAPI
**Meta:** envolver el workflow de la Fase 1 en un endpoint. Subir video → procesar (job
async) → descargar resultado.

**Definición de "hecho":** se puede disparar el procesamiento por HTTP (sin abrir ComfyUI a
mano), consultar el estado del job y descargar el video terminado.

---

### Fase 3 — Servir en la DGX + frontend local
**Meta:** el backend corriendo como servidor en la DGX, y una página web liviana en un PC
local que sube el video, muestra el progreso y descarga el resultado. Comunicación por LAN.

**Definición de "hecho":** desde otra PC de la red, subir un video y recibir el mejorado, sin
tocar la DGX directamente.

---

### Fase 4 — (Opcional) Escalar
**Meta:** cola de varios videos, soporte multiusuario, manejo de la GPU como recurso
compartido. Solo si hace falta. **YAGNI por ahora.**

---

## Estado actual

- [x] Roadmap de 4 fases definido y registrado (este documento).
- [x] **Fase 1 — diseño detallado** → `docs/planeacion/fase-1-diseno.md` (aprobado 2026-07-18).
- [x] Fase 1 — investigación/fuentes → `docs/investigacion/fase-1-fuentes.md`.
- [ ] Fase 1 — revisar investigación + (opcional) skill de grilling ← retomar acá.
- [ ] Fase 1 — plan de implementación (tareas detalladas).
- [ ] Fase 1 — implementación.
- [ ] Fases 2, 3, 4.

**Decisión de entorno (Fase 1):** ComfyUI en contenedor Docker sobre `nvcr.io/nvidia/pytorch:25.11-py3`
(la GB10 es sm_121 y el venv actual no la soporta bien). Ver diseño para detalles.

---

## Cómo retomar en la próxima sesión

> "Continuemos el proyecto de mejorar videos. Ver `docs/00-roadmap.md`. Vamos a hacer el
> brainstorming/diseño detallado de la **Fase 1 (baseline en ComfyUI)**."

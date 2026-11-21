# Longshot-Intel

Longshot **v3.0.0** con el parche Intel para la UHD 620: compila **solo `longshot_vk.dll`**
(Windows x64) con GitHub Actions (runner Ubuntu, cross-compile con llvm-mingw, la misma
receta de la CI de upstream) y la publica como artefacto y en la release
`vk-dll-v3.0.0-intel`.

## Cómo funciona

El workflow clona el código oficial de Longshot (tag `v3.0.0`, git.magicbean.dev) y le
aplica los archivos de `overlay/` antes de compilar:

- `overlay/gpu/vk/vk_draw.cpp`, `vk_draw.h`, `switches.def` — el parche.
- `overlay/tests/…` — los checks adaptados al parche.

## El parche

1. **P1 — Guardias SEH en las llamadas que no tenían.** `vkCreateGraphicsPipelines` ya
   tenía guardia (`guarded_graphics`), pero el crash al cargar la partida en la UHD 620
   (0xc0000005 dentro de `igvk64.dll`) escapaba por una llamada vecina sin guardia:
   - `vkCreatePipelineLayout` (`ensure_layout`) → `guarded_pipeline_layout`: si el driver
     revienta, el programa entra en cuarentena (`driver_faults`) y su draw se salta; el
     juego no se cierra.
   - `vkGetPipelineCacheData` (los 5 sitios) → `guarded_pipeline_cache_data`: si el driver
     revienta al leer la caché, la lectura se da por fallida, como ya trataban los sitios
     cualquier resultado que no fuera `VK_SUCCESS`.
   - `vkCreateShaderModule` ya estaba guardado en la v3.0.0 (`guarded_shader_module`).
2. **P2 — Sin optimizador en Intel integrada.** En el driver Intel de Windows sobre GPU
   integrada, los pipelines gráficos se crean con
   `VK_PIPELINE_CREATE_DISABLE_OPTIMIZATION_BIT` (el equivalente para gráficos de lo que
   upstream ya hace para compute en Intel). `RX_VK_INTEL_OPT=1` lo deshace.

La ABI de la DLL no cambia (`MKWVK_API_VERSION` intacta): la DLL parcheada sustituye a la
original en la carpeta del juego ya compilado.

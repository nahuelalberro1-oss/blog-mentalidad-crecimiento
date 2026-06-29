---
layout: post
title: "Post-mortem: inconsistencias de turnos al integrar la API de 2Workers"
date: 2026-06-29 09:00:00
categories: [backend, post-mortem, devops]
tags: [node, express, 2workers-api, pm2, planificacion-diaria]
---

# Post-mortem: inconsistencias de turnos al integrar la API de 2Workers

## Contexto

El dashboard **"Planificación Diaria x Cuadrilla"** es una herramienta interna que desarrollé para Horizon Seguridad, construida en **Node.js/Express** y desplegada con **PM2** en una máquina local de la empresa. Su función principal es mostrar, en una sola vista, la planificación diaria de cada cuadrilla de vigilancia, combinando la planificación manual que hacen los supervisores con los datos reales de asistencia que llegan desde la **API de 2Workers** (el sistema externo de gestión de personal que usa la empresa).

El dashboard corre como proceso persistente con PM2, haciendo polling periódico contra la API de 2Workers para refrescar el estado de cada turno (asignado, confirmado, en curso, ausente). Esta arquitectura —un proceso de larga duración consultando un sistema externo cada cierto intervalo— es simple en el papel, pero esconde varias formas de fallar silenciosamente, como quedó demostrado en este caso.

## Problema

A las pocas semanas de uso en producción, los supervisores empezaron a reportar que el dashboard mostraba guardias como "ausentes" cuando en realidad habían fichado entrada, y viceversa: guardias marcados como presentes que nunca habían llegado al puesto. La confianza en la herramienta empezó a caer justo cuando más se la necesitaba: en los cambios de turno, el momento de mayor actividad.

Al investigar, identifiqué tres causas combinadas, ninguna de las cuales era evidente por sí sola:

1. **Manejo de timezone inconsistente.** La API de 2Workers devuelve timestamps en UTC, pero el dashboard los comparaba contra la hora local sin conversión explícita en un punto del pipeline. El resultado: turnos que arrancaban cerca de la medianoche quedaban mal clasificados como "del día anterior" o "del día siguiente" según el momento exacto del polling.

2. **Polling no idempotente.** Cada ciclo de polling sobrescribía el estado completo de la cuadrilla en memoria, en lugar de aplicar un merge incremental. Si una request a 2Workers fallaba parcialmente (timeout en algunos registros, éxito en otros), el siguiente ciclo exitoso podía "revertir" datos correctos a un estado anterior incompleto.

3. **Reinicios silenciosos de PM2 enmascarando datos viejos.** PM2 estaba configurado para reiniciar el proceso automáticamente ante un crash, pero no había alerta visible cuando esto pasaba. El dashboard volvía a mostrar datos "frescos" en apariencia, aunque en realidad eran el último estado cacheado antes del crash, ahora desactualizado.

Ninguna de las tres causas, vista de forma aislada, explicaba el patrón completo de errores reportados. Fue la combinación —y el hecho de que cada una enmascaraba a las otras— lo que hizo que el diagnóstico tomara más tiempo del esperado.

## Acciones

El proceso de resolución combinó trabajo técnico con un ejercicio deliberado de post-mortem constructivo, siguiendo el espíritu de un post-mortem sin culpa (blameless):

**Diagnóstico:**
- Reproduje el problema de timezone agregando logging temporal de timestamps crudos (UTC) junto a los timestamps ya procesados, en cada etapa del pipeline. Esto confirmó la desalineación en los turnos nocturnos.
- Para el polling no idempotente, revisé los logs de PM2 cruzados con los timestamps de las requests fallidas a 2Workers, y pude correlacionar los "retrocesos" de estado reportados por los supervisores con ciclos de polling que habían tenido fallos parciales.
- Confirmé los reinicios silenciosos revisando el historial de PM2 (`pm2 logs` y los logs de reinicio), encontrando varios restarts que coincidían temporalmente con los reportes de datos desactualizados.

**Corrección:**
- Normalicé todos los timestamps a UTC en el punto de entrada del pipeline, y centralicé la conversión a hora local únicamente en la capa de presentación.
- Reemplacé el polling de sobrescritura completa por un merge incremental por turno, usando el ID de turno como clave, de forma que un fallo parcial en una request no pudiera revertir datos ya confirmados.
- Configuré alertas explícitas en PM2 ante reinicios (vía un hook que loguea a un canal de monitoreo), y agregué un timestamp visible de "última sincronización exitosa" en el dashboard, para que los supervisores pudieran detectar a simple vista cuándo los datos no eran confiables.

**Post-mortem constructivo:**
Una vez resuelto el incidente, documenté internamente una línea de tiempo del problema (cuándo empezó, cuándo se detectó, cuándo se resolvió) y las tres causas raíz, sin enfocar el análisis en "quién" había introducido cada una, sino en qué condiciones del sistema permitieron que pasaran desapercibidas. El objetivo no era encontrar responsables sino identificar qué controles faltaban —en este caso, observabilidad— para que el mismo tipo de falla no pudiera ocurrir de forma silenciosa otra vez.

## Aprendizajes

- **La observabilidad no es opcional en procesos de larga duración.** Un proceso con PM2 que se reinicia solo es robusto frente a crashes, pero sin alertas explícitas, ese mismo mecanismo de resiliencia puede ocultar problemas en lugar de solucionarlos.
- **El polling contra APIs externas necesita pensarse en términos de fallos parciales, no solo de éxito/error binario.** Diseñar para idempotencia desde el principio evita que un error de red se traduzca en una regresión de datos.
- **Los timestamps cruzando timezones son una fuente de bugs desproporcionadamente grande para lo simple que parece el problema.** Normalizar a UTC en el punto de entrada y convertir solo en la capa de presentación es una regla que debería aplicar por defecto, no como corrección posterior.
- **Un post-mortem sin culpa cambia la calidad del análisis.** Al sacar la pregunta de "quién" de la conversación, fue más fácil llegar a las tres causas reales en lugar de quedarse en la primera explicación plausible.

---

## Reflexión: feedback radicalmente sincero

Durante este proceso aplicué el principio de feedback radicalmente sincero principalmente hacia mi propio trabajo: en vez de cerrar el incidente apenas el síntoma visible desapareció (los reportes de los supervisores bajaron tras la primera corrección de timezone), me obligué a preguntar *por qué* ese fix no explicaba el patrón completo de errores, lo cual llevó a destapar las otras dos causas. Ser sincero conmigo mismo sobre que "funciona ahora" no es lo mismo que "está resuelto" fue lo que evitó dejar dos bugs latentes en producción.

También documenté el post-mortem de forma directa y sin atenuantes: nombré explícitamente la falta de observabilidad como una decisión de diseño insuficiente, no como un detalle menor, porque suavizar ese punto en el reporte habría hecho menos probable que se priorizara la corrección.
---

## Evidencia de control de versiones

El trabajo de corrección y la documentación de este post quedaron registrados en el repositorio público del proyecto, con el siguiente historial de commits:

- `chore: configura Jekyll para GitHub Pages`
- `feat: agrega index con layout home para listar posts`
- `docs: agrega post-mortem sincronizacion 2Workers`
- `fix: corrige ruta duplicada de _posts`
- `fix: agrega theme minima para soportar layout home y post`

Repositorio completo: [github.com/nahuelalberro1-oss/blog-mentalidad-crecimiento](https://github.com/nahuelalberro1-oss/blog-mentalidad-crecimiento)

Historial de commits: [github.com/nahuelalberro1-oss/blog-mentalidad-crecimiento/commits/main](https://github.com/nahuelalberro1-oss/blog-mentalidad-crecimiento/commits/main)

---
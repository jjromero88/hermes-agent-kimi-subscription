# Hermes Agent + Kimi Code por suscripción

*Read this in [English](README.md).*

> Guía para configurar **Hermes Agent** con tu **suscripción de Kimi** (Kimi Code), consumiendo la cuota de tu membresía en lugar de pagar por token.

Hermes Agent incluye de forma nativa el proveedor `kimi-coding`. Esto significa que si tienes una membresía de Kimi (por ejemplo, el plan de $19/mes), puedes generar una API key de **Kimi Code** y usarla directamente en Hermes: el consumo se descuenta de la cuota semanal de tu suscripción, sin facturación por token.

> ℹ️ **Entorno verificado.** Esta guía se escribió y probó sobre:
> - **Hermes Agent v0.19.0 (2026.7.20)**
> - **Kimi Code**, tal como estaba disponible el **2026-07-27** (es un servicio hosteado, sin un cliente versionado al que anclarse)
> - **Ubuntu Desktop 26.04 LTS** ("resolute")
>
> Los pasos dependen del comportamiento del proveedor `kimi-coding` en ese momento puntual. Si estás en una versión de Hermes muy distinta o mucho después en el tiempo, revisa con cuidado los Pasos 1 y 4 — tanto Kimi como Hermes publican actualizaciones frecuentes y el detalle de los proveedores puede cambiar.

---

## Tabla de contenidos

- [Cómo funciona](#cómo-funciona)
- [Requisitos](#requisitos)
- [Paso 1 — Actualizar Hermes Agent](#paso-1--actualizar-hermes-agent)
- [Paso 2 — Generar la API key de Kimi Code](#paso-2--generar-la-api-key-de-kimi-code)
- [Paso 3 — Guardar la key en el entorno de Hermes](#paso-3--guardar-la-key-en-el-entorno-de-hermes)
- [Paso 4 — Configurar el proveedor en config.yaml](#paso-4--configurar-el-proveedor-en-configyaml)
- [Paso 5 — Probar la configuración](#paso-5--probar-la-configuración)
- [Paso 6 — Reiniciar el servicio (instalaciones 24/7)](#paso-6--reiniciar-el-servicio-instalaciones-247)
- [Verificar el consumo de la suscripción](#verificar-el-consumo-de-la-suscripción)
- [Límites de la suscripción](#límites-de-la-suscripción)
- [Opcional: modelo de respaldo (fallback)](#opcional-modelo-de-respaldo-fallback)
- [Solución de problemas](#solución-de-problemas)
- [Referencias](#referencias)

---

## Cómo funciona

Tu membresía de Kimi incluye **Kimi Code**, un servicio de programación asistida que permite generar hasta **5 API keys** desde su consola web. Estas keys:

- Empiezan con el prefijo `sk-kimi-`.
- **No cobran por token**: consumen la cuota semanal de tu suscripción (se renueva automáticamente cada 7 días).
- Funcionan con agentes de terceros a través del endpoint `https://api.kimi.com/coding/v1`, con el modelo `kimi-for-coding`.

Hermes Agent incluye este endpoint preconfigurado bajo el proveedor `kimi-coding`, así que la configuración se reduce a dos archivos: `~/.hermes/.env` (la key) y `~/.hermes/config.yaml` (el proveedor).

> ⚠️ **No confundir** con las API keys de `platform.moonshot.ai`: esas pertenecen a la plataforma de pago por token de Moonshot y **no** descuentan de la suscripción. La única consola válida para este método es `kimi.com/code/console`.

## Requisitos

- Hermes Agent instalado, en una versión reciente (ver Paso 1).
- Una membresía activa de Kimi que incluya Kimi Code.
- Linux/macOS con acceso a terminal (los ejemplos usan Ubuntu).

## Paso 1 — Actualizar Hermes Agent

Antes de todo, actualiza Hermes a la última versión — las instalaciones muy antiguas solo conocían el endpoint clásico de Moonshot (pago por token) y no traían el proveedor `kimi-coding`:

```bash
hermes update
```

Verifica la versión instalada:

```bash
hermes version
```

No hay un número de versión mínimo estricto y documentado públicamente; basta con estar en una instalación actualizada recientemente. Si tras el Paso 4 el proveedor `kimi-coding` no es reconocido, la causa casi siempre es una instalación desactualizada (ver [Solución de problemas](#solución-de-problemas)).

## Paso 2 — Generar la API key de Kimi Code

1. Abre la consola de Kimi Code: **https://www.kimi.com/code/console**
2. Inicia sesión con la cuenta que tiene la membresía activa.
3. En la sección **API Keys**, haz clic en **"+ Create API Key"**.
4. Ponle un nombre descriptivo (por ejemplo `hermes-agent`) y créala.
5. **Copia la key inmediatamente** — solo se muestra completa una vez. Debe empezar con `sk-kimi-`.

## Paso 3 — Guardar la key en el entorno de Hermes

Agrega la key al archivo de entorno de Hermes (el comando crea el archivo si no existe):

```bash
echo 'KIMI_API_KEY=sk-kimi-TU-KEY-AQUI' >> ~/.hermes/.env
chmod 600 ~/.hermes/.env
```

Reemplaza `sk-kimi-TU-KEY-AQUI` por tu key real, sin espacios alrededor del `=` y conservando las comillas simples.

Verifica que quedó bien guardada (una sola línea `KIMI_API_KEY=`, sin duplicados):

```bash
grep KIMI_API_KEY ~/.hermes/.env
```

## Paso 4 — Configurar el proveedor en config.yaml

Haz un respaldo de tu configuración actual:

```bash
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
```

Edita el archivo:

```bash
nano ~/.hermes/config.yaml
```

Localiza el bloque `model:` al inicio del archivo. Este bloque existe sin importar qué proveedor tengas configurado — puede ser Nous, OpenRouter, Anthropic directo, OpenAI, un modelo local, etc. **Sustitúyelo íntegramente**, sin importar su contenido actual, por:

```yaml
model:
  default: kimi-for-coding
  provider: kimi-coding
```

Algunos ejemplos de cómo puede verse el bloque *antes* de reemplazarlo, según el proveedor que tuvieras (son solo ilustrativos — el tuyo puede ser distinto):

```yaml
# Ejemplo viniendo de Nous
model:
  default: deepseek/deepseek-v4-flash
  provider: nous
  base_url: https://inference-api.nousresearch.com/v1
```

```yaml
# Ejemplo viniendo de OpenRouter
model:
  default: anthropic/claude-opus-4.7
  provider: openrouter
```

Puntos importantes:

- Si tu bloque anterior tenía una línea `base_url`, **elimínala por completo** (no la dejes vacía): el proveedor `kimi-coding` ya conoce su endpoint y una URL ajena rompería la conexión. Esta regla aplica sin importar de qué proveedor vengas.
- No modifiques ninguna otra sección del archivo (`agent`, `terminal`, `memory`, etc.).
- Alternativa más guiada: en vez de editar el YAML a mano, puedes correr `hermes setup` o `hermes model` y seleccionar "Kimi / Moonshot" desde el asistente interactivo — hace el mismo cambio sin tocar el archivo directamente.

Guarda con `Ctrl+O`, `Enter` y sal con `Ctrl+X`.

## Paso 5 — Probar la configuración

```bash
hermes chat -q "Responde solo: PING" --max-turns 1
```

Si responde sin errores de autenticación (`401`/`403`) ni de modelo desconocido, Hermes ya está consumiendo de tu suscripción. ✅

## Paso 6 — Reiniciar el servicio (instalaciones 24/7)

Si Hermes corre como servicio permanente, reinícialo para que tome la nueva configuración. Con systemd (modo usuario):

```bash
systemctl --user list-units | grep -i hermes   # identifica el servicio
systemctl --user restart <nombre-del-servicio> # p. ej. hermes-gateway.service
```

Si corre en `tmux`/`screen` o en una sesión interactiva, cierra y vuelve a lanzar el proceso.

## Verificar el consumo de la suscripción

Entra de nuevo a **https://www.kimi.com/code/console** y revisa el panel:

- **Rate limit details**: se mueve casi de inmediato con las primeras peticiones (es la ventana de 5 horas).
- **Weekly usage**: refleja el consumo acumulado de la semana; con pruebas pequeñas puede seguir mostrando 0% por redondeo.
- **API Keys**: tu key debe aparecer con estado `Enabled`.

Si esos indicadores se mueven al usar Hermes, la configuración es correcta: estás consumiendo suscripción, no tokens.

## Límites de la suscripción

La cuota de Kimi Code funciona con dos medidores:

| Medidor | Comportamiento |
|---|---|
| Cuota semanal | Se renueva automáticamente cada 7 días |
| Rate limit | Ventana móvil de ~5 horas (aprox. 300–1,200 requests según el plan, hasta 30 concurrentes) |

En operación 24/7 es posible topar la ventana de 5 horas en picos de uso: verás errores `429` temporales que **se resuelven solos** al renovarse la ventana. No es un error de configuración.

## Opcional: modelo de respaldo (fallback)

Hermes soporta conmutación automática de proveedor ante errores `429`/`503`/`529`. Si tienes credenciales de otro proveedor (OpenRouter, Nous, etc.), puedes descomentar y adaptar la sección `fallback_model` que viene al final del `config.yaml`:

```yaml
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
```

Así, si tu cuota de Kimi se agota momentáneamente, Hermes sigue operando con el respaldo y vuelve a Kimi al liberarse la ventana.

> `fallback_model` (singular) es la forma clásica y sigue funcionando, pero Hermes también soporta `fallback_models` (plural, en forma de lista) si quieres encadenar más de un respaldo. Usa la que necesites según cuántos proveedores de respaldo tengas.

## Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `401` / `403` en la prueba | Key mal copiada, con espacios, o duplicada en `.env` | Revisa `~/.hermes/.env`: una sola línea `KIMI_API_KEY=sk-kimi-...` limpia |
| Proveedor `kimi-coding` no reconocido | Hermes desactualizado | `hermes update` y vuelve a intentar |
| Error de conexión / endpoint | Quedó una línea `base_url` de otro proveedor en el bloque `model:` | Elimínala y reinicia |
| La key no descuenta de la suscripción | Key generada en `platform.moonshot.ai` (pago por token) | Genera la key en `kimi.com/code/console` |
| `429` intermitentes en uso intensivo | Rate limit de la ventana de 5 horas | Esperar la renovación o configurar `fallback_model` |
| El servicio 24/7 sigue usando el proveedor anterior | No se reinició el proceso | `systemctl --user restart <servicio>` |

Si algo sale mal y necesitas volver atrás:

```bash
cp ~/.hermes/config.yaml.bak ~/.hermes/config.yaml
systemctl --user restart <nombre-del-servicio>
```

## Referencias

- [Kimi Code — Documentación oficial](https://www.kimi.com/code/docs/en/)
- [Kimi Code — Uso en agentes de terceros](https://www.kimi.com/code/docs/en/third-party-tools/other-coding-agents)
- [Hermes Agent — Proveedores soportados](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- [Hermes Agent — Repositorio en GitHub](https://github.com/NousResearch/hermes-agent)

---

*Esta guía es un aporte de la comunidad y no está afiliada a Moonshot AI ni a Nous Research. Los precios, límites y versiones mencionados corresponden a julio de 2026 y pueden cambiar.*

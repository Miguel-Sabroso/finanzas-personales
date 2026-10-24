# CLAUDE.md — Finanzas Personales

## Qué es este proyecto

App personal (Android) para ver de un vistazo mis finanzas: saldo total, saldo por cuenta, gastos mensuales por cuenta y gastos por categoría. Los datos llegan automáticamente de mis bancos a través de Enable Banking (PSD2).

- Cuentas conectadas: **Santander**, **Revolut** y **Trade Republic** (cuenta de efectivo).
- Uso personal y no comercial. La app de Enable Banking está en **modo restringido**: solo puede leer las cuentas vinculadas en su panel de control.
- El código es público en GitHub (proyecto de portfolio). Los datos y los secretos, nunca.

## Fase actual

**Fase 0 — Prueba de concepto.** Objetivo: confirmar que de las tres cuentas se pueden obtener saldos y movimientos por la API. Todavía no hay base de datos ni app móvil.

Fases siguientes (no adelantar trabajo de ellas sin que lo pida):
1. Backend en Supabase (tablas, Edge Functions, sincronización diaria).
2. App Android con Expo (login, resumen, movimientos).
3. Categorías (reglas + corrección manual) y detección de transferencias internas.
4. Pulido: avisos de renovación del consentimiento, gráficos, README con datos falsos.

## Stack

- **TypeScript en modo estricto** en todo el proyecto.
- **Deno** para scripts y para las Supabase Edge Functions (el mismo código sirve en ambos).
- **Zod** para validar todas las respuestas de APIs externas.
- Fase 1+: **Supabase** (Postgres, Auth, Edge Functions, pg_cron).
- Fase 2+: **Expo / React Native** para la app Android.

## Estructura del repositorio

```
supabase/
  functions/
    _shared/
      enable-banking/   # Cliente de Enable Banking: JWT, auth, sesiones, cuentas (reutilizable)
      sources/          # Interfaz BankDataSource y sus implementaciones
  migrations/           # (fase 1)
apps/
  mobile/               # App Expo (fase 2)
scripts/                # Scripts de la prueba de concepto y utilidades
docs/
  architecture.md       # Arquitectura actual, con diagrama
  decisions/            # Registro de decisiones (ADR): una por archivo, NNNN-titulo.md
```

## Principios de arquitectura

- **Fuentes de datos intercambiables.** Todo acceso a datos bancarios pasa por una interfaz `BankDataSource` (saldos y movimientos normalizados). Implementaciones: Enable Banking y entrada manual (respaldo). El resto de la app no sabe de dónde viene el dato.
- **Capas separadas:** cliente de API externa → normalización a modelos propios → lógica de negocio → persistencia/presentación. Nada de lógica de negocio dentro del cliente HTTP.
- **Dinero:** nunca `number` con decimales. Importes como string decimal de la API → enteros en céntimos (o `numeric` en Postgres). Siempre con su moneda.
- **Fechas:** guardar en UTC; mostrar en Europe/Madrid.
- **Sincronización idempotente:** ejecutar la sincronización dos veces no debe duplicar movimientos (upsert por identificador de transacción; si el banco no da uno estable, hash de campos clave).
- **Multiusuario desde el diseño:** aunque ahora el único usuario soy yo, todas las tablas llevan `user_id` y reglas RLS (fase 1).
- **Simplicidad:** sin abstracciones "por si acaso". Escalable significa fácil de cambiar, no complejo.

## Enable Banking (resumen técnico)

- Docs: https://enablebanking.com/docs/api/reference/
- Autenticación: JWT firmado con **RS256** con mi clave privada. Header: `typ: JWT`, `alg: RS256`, `kid: <APP_ID>`. Body: `iss: "enablebanking.com"`, `aud: "api.enablebanking.com"`, `iat`, `exp`. **TTL máximo 24 h.**
- Flujo de acceso a cuentas:
  1. `GET /aspsps` → lista de bancos con `maximum_consent_validity` (segundos).
  2. `POST /auth` con `aspsp`, `access.valid_until` (≤ ahora + `maximum_consent_validity`), `redirect_url`, `state`, `psu_type: "personal"` → devuelve una `url`.
  3. El titular se autentica en el banco y vuelve a `redirect_url?code=...&state=...`.
  4. `POST /sessions` con `{ code }` → `session_id` y lista de cuentas (`uid`). Parte de esa información solo se devuelve una vez: guardarla.
  5. `GET /accounts/{uid}/balances` y `GET /accounts/{uid}/transactions` (paginado con `continuation_key`).
- Redirect URL registrada para la fase 0: `https://localhost:8080/callback` (el navegador da error, pero el `code` se copia de la barra de direcciones).
- Vincular cuentas en el panel solo **activa** la app; el acceso a los datos se autoriza siempre por API.

## Seguridad (obligatorio)

- **Nunca** commitear: `.env`, `*.pem`, claves de Supabase, IDs de sesión, ni datos bancarios reales (movimientos, IBAN, saldos, capturas).
- La clave privada vive **fuera del repositorio**; el código la lee desde la ruta indicada en `.env`. En producción será un secreto de Supabase, **nunca** dentro de la app móvil.
- Mantener `.env.example` con los nombres de las variables, sin valores.
- Logs: no imprimir tokens, JWT, IBAN completos ni descripciones completas de movimientos. Enmascarar (ej. `ES12 **** **** 3456`).
- Las respuestas reales de la API que se guarden para estudiarlas van a `data/` (en `.gitignore`). Los tests usan **fixtures inventados**.

## Comandos

> Actualizar esta sección cuando se creen o cambien los comandos.

- `deno task poc` — ejecutar la prueba de concepto (fase 0)
- `deno task test` — tests
- `deno task check` — formato, lint y comprobación de tipos

## Forma de trabajar

- **Planificar antes de programar:** para cualquier cambio no trivial, proponer un plan y esperar mi aprobación.
- **Pasos pequeños** y verificables; tests junto al código que tengan lógica (JWT, parsing, normalización, cálculos de dinero).
- Ejecutar `deno task check` y los tests antes de dar algo por terminado.
- Cuando se tome una decisión de diseño relevante, añadir un ADR breve en `docs/decisions/` (contexto, opciones, decisión, consecuencias) y mantener `docs/architecture.md` al día.
- Commits pequeños con Conventional Commits (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`).
- Si algo es ambiguo o hay varias opciones razonables, **preguntar** en vez de suponer.

## Idioma y explicaciones

- Hablarme en **español** y explicar el **porqué** de cada decisión: quiero entender la arquitectura y cómo funciona todo, no solo obtener el resultado.
- Código, identificadores, comentarios, mensajes de commit y README en **inglés** (proyecto de portfolio).
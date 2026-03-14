# ADR-003 — JWT propio en Etapa 1, Auth0 en Etapa 2

**Estado:** Aceptado
**Fecha:** 2026-03-13
**Decisores:** Equipo VGM

---

## Contexto

VGM Core usa Auth0 desde el inicio. VGM Core Geo necesita autenticación propia para poder funcionar y venderse de forma independiente, sin requerir que el cliente tenga Auth0 configurado.

El namespace JWT ya fue acordado con Mauricio: `https://vgmcoregeo.com` — independiente del namespace de VGM Core (`https://vgmcore.com`).

## Decisión

VGM Core Geo arranca con **JWT propio** generado internamente. El diseño interno ya está preparado para conectar Auth0 en Etapa 2 sin reescribir código.

El patrón sigue exactamente lo que hace VGM Core:
- Solo el `tenant_id` va en el JWT
- La empresa y sucursal se resuelven desde headers HTTP
- El rol se carga desde base de datos (nunca desde el token)

## Consecuencias

**Positivas:**
- VGM Core Geo funciona sin depender de Auth0 ni de VGM Core
- La migración a Auth0 en Etapa 2 es un cambio de configuración, no de código
- El patrón es idéntico a VGM Core — código de seguridad se copia y adapta directamente

**Trade-offs:**
- En Etapa 1 el equipo mantiene su propia lógica de login y refresh de tokens

**Mitigaciones:**
- Copiar el patrón de seguridad de Mauricio (TenantContextFilter, TenantConnectionPreparer) para no reinventar la rueda
- La aplicación `VGM Core Geo` ya está creada en Auth0 (`vgm-core-dev.us.auth0.com`) para cuando llegue Etapa 2

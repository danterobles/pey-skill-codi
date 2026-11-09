# pey-codi-integration · Skill para agentes de IA

Repositorio: https://github.com/danterobles/pey-skill-codi

Skill en formato abierto [Agent Skills](https://agentskills.io) que le enseña a un agente de código (Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot y otros) a integrar la **API CoDi® de [pey](https://pey.mx/api-codi)** en una plataforma SaaS, multitenant o no, con implementaciones de referencia para **PHP/Laravel** y **Node.js/TypeScript**:

- Cobros CoDi por **QR** y por **notificación push** al celular
- **Links de pago** hospedados (CoDi, tarjeta, Apple Pay y Google Pay)
- Registro de **cuentas CLABE** por tenant para que cada uno reciba su dinero
- Consulta y **conciliación** de transacciones
- **Webhooks** firmados con Standard Webhooks: verificación, deduplicación y procesamiento en cola

Contrato documentado: OpenAPI de pey versión `2026-10-20`. El skill le indica al agente releer la spec actual antes de programar.

## Requisitos

### 1. Cuenta y credenciales en pey

- **Cuenta en el [portal de pey](https://portal.mipey.mx)** con una cuenta de pago de tipo `codi`. Agrega `stripe` si también vas a cobrar con tarjeta, Apple Pay o Google Pay en links de pago.
- **API key** creada en la sección Desarrollador del portal. Solo existe el ambiente de producción, así que usa una key de producción desde el inicio, una por ambiente (staging y producción).
- **Cuenta bancaria activa.** El registro con Banxico tarda hasta 5 días hábiles; mientras tanto, pey ofrece una cuenta de prueba que se activa de inmediato.

### 2. Webhook

- **URL pública con HTTPS** registrada en el portal (engrane de la API key → URL → "Crear Webhook").
- **Secreto del webhook**, que se muestra al crearlo y se puede rotar. Con él se verifica la firma de cada evento.
- **Un webhook por API key.** Separa las keys de staging y producción.
- **Túnel para desarrollo local** (ngrok o Cloudflare Tunnel) para que pey alcance tu máquina.

### 3. Si tu SaaS es multitenant

- **Modelo A (una key para la plataforma):** de cada cliente necesitas su CLABE de 18 dígitos, el nombre legal del titular y si es empresa o persona física. El agente registra las cuentas por API; cada una debe estar activa antes de cobrar.
- **Modelo B (una key por cliente):** cada cliente abre su cuenta en pey y captura su API key y su secreto en tu SaaS, donde se guardan cifrados.

### 4. Infraestructura del proyecto

- Backend en **Laravel 11+** o **Node.js 18+** (en otros lenguajes el agente traduce los mismos componentes).
- **Base de datos** para cuentas, cobros, transacciones y eventos de webhook.
- **Cola** (colas de Laravel, o Redis con BullMQ en Node) y **programador de tareas** (cron o scheduler) para la conciliación.
- **HTTPS con certificado válido**, que también exigen las URLs de regreso de los links de pago.
- Variables de entorno `PEY_API_KEY` y `PEY_WEBHOOK_SECRET`.

### 5. Para trabajar con el agente

- **Claude Code** u otro agente compatible con el skill instalado.
- **Decisiones listas antes de pedirle nada:** modelo de tenencia (A o B), productos (QR, push, links de pago), cuánto tiempo vale un cobro antes de expirar (mínimo 6 minutos) y cómo se liga cada pago con su orden.

### 6. Para probar

- **Una app bancaria con CoDi**, para pagar de verdad con montos de $0.01 MXN.
- **Acceso a "Webhook Logs"** en el portal, para revisar qué eventos llegaron y cada intento de entrega.

## Costos y comisiones

Revisado el 9 de noviembre de 2026 en el sitio y los [términos del API](https://pey.mx/legal/terminos-api) de pey. Confírmalo con pey antes de salir a producción.

| Concepto | Qué dice pey |
|---|---|
| Cobros CoDi | **0% de comisión.** Las operaciones CoDi no generan comisiones; el negocio recibe el monto completo. |
| Uso de la API | **Gratis por ahora.** Los términos lo declaran gratuito hasta que pey notifique lo contrario. |
| Tarifas futuras | pey puede introducirlas con **30 días naturales de aviso** y debe obtener tu aceptación expresa del primer cargo. Se cobrarían por uso del software, **nunca como porcentaje de tus cobros**, con factura mensual más IVA. |
| Tarjeta, Apple Pay y Google Pay (links de pago) | La API reporta un `fee` cobrado al pagador y un `applicationFee` por transacción, pero **pey no publica las tarifas**: pídelas a pey. |
| Comisiones de tu banco | Van aparte de cualquier tarifa de pey. |
| Devoluciones | pey no revierte transferencias liquidadas: **las devoluciones al pagador las hace el integrador**, y las aclaraciones se tramitan con el banco. |

## Instalación

### Claude Code (como plugin)

```
/plugin marketplace add danterobles/pey-skill-codi
/plugin install pey-codi@doit-skills
```

El skill queda disponible como `/pey-codi:pey-codi-integration` y Claude también lo carga solo cuando le pides algo relacionado con CoDi o pey.

### Cualquier agente compatible con Agent Skills

```
npx skills add danterobles/pey-skill-codi        # en el proyecto actual
npx skills add danterobles/pey-skill-codi -g     # para todos tus proyectos
npx skills add danterobles/pey-skill-codi -a claude-code -a cursor
```

### Manual

```
git clone https://github.com/danterobles/pey-skill-codi.git
cp -r pey-skill-codi/skills/pey-codi-integration ~/.claude/skills/
```

O copia la carpeta `skills/pey-codi-integration/` a:

| Agente | Por usuario | Por proyecto |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Otros | Consulta la carpeta de skills de tu agente | |

En agentes que no leen skills, agrega `SKILL.md` como contexto del proyecto.

## Uso

Pídelo en lenguaje natural:

```
> Integra cobros CoDi por QR en el checkout. Somos multitenant: cada restaurante recibe en su propia CLABE.
> Agrega el webhook de pey con verificación de firma y pruebas para firma inválida y eventos duplicados.
> Revisa mi integración actual de pey contra las trampas conocidas y el definition of done.
```

El agente detecta tu stack (Laravel o Node.js/TypeScript; en otros lenguajes traduce los mismos componentes), decide contigo el modelo de tenencia (una API key con CLABE por tenant, o una API key por tenant), implementa el cliente, los webhooks y la conciliación, y cierra con pruebas y una checklist.

## Qué incluye

```
skills/pey-codi-integration/
├── SKILL.md                 # Contrato de la API, decisiones, flujos, trampas, pruebas y checklist
└── references/
    ├── laravel.md           # PHP / Laravel 11+: cliente, verificador, controller, job y conciliación
    └── node-typescript.md   # Node.js 18+ / TypeScript: cliente, verificador, Express, Next.js, NestJS, Fastify, BullMQ
```

## Antes de usarlo en producción

- pey **no tiene sandbox**: las pruebas reales se hacen en producción con montos de $0.01 MXN.
- La API **no tiene llave de idempotencia**: el skill evita reintentar a ciegas los `POST`.
- La activación de una CLABE con Banxico tarda **hasta 5 días hábiles**.
- Revisa siempre lo que entrega el agente contra la sección *Definition of done* del skill.

## Validación

```
claude plugin validate .
```

## Contribuir

La API de pey es nueva y cambia. Si encuentras una diferencia entre el skill y la [documentación oficial](https://docs.pey.mx) o un comportamiento en producción que no esté documentado, abre un [issue](https://github.com/danterobles/pey-skill-codi/issues) o un pull request.

## Aviso

Proyecto independiente, sin afiliación con pey. CoDi® es la plataforma de cobros digitales de Banco de México. Este skill no sustituye la documentación oficial: https://docs.pey.mx

## Licencia

[MIT](LICENSE) © 2026 Dante Robles · DoIT Consulting

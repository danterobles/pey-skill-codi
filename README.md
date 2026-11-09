# pey-codi-integration · Skill para agentes de IA

Repositorio: https://github.com/danterobles/pey-skill-codi

Skill en formato abierto [Agent Skills](https://agentskills.io) que le enseña a un agente de código (Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot y otros) a integrar la **API CoDi® de [pey](https://pey.mx/api-codi)** en una plataforma SaaS, multitenant o no:

- Cobros CoDi por **QR** y por **notificación push** al celular
- **Links de pago** hospedados (CoDi, tarjeta, Apple Pay y Google Pay)
- Registro de **cuentas CLABE** por tenant para que cada uno reciba su dinero
- Consulta y **conciliación** de transacciones
- **Webhooks** firmados con Standard Webhooks: verificación, deduplicación y procesamiento en cola

Contrato documentado: OpenAPI de pey versión `2026-10-20`. El skill le indica al agente releer la spec actual antes de programar.

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

El agente detecta tu stack, decide contigo el modelo de tenencia (una API key con CLABE por tenant, o una API key por tenant), implementa el cliente, los webhooks y la conciliación, y cierra con pruebas y una checklist.

## Qué incluye

```
skills/pey-codi-integration/
├── SKILL.md                 # Contrato de la API, decisiones, flujos, trampas, pruebas y checklist
└── references/
    └── laravel.md           # Implementación de referencia en Laravel (cliente, verificador, webhook, job)
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

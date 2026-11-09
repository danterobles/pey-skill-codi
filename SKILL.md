---
name: "pey-codi-integration"
description: "Integra la API de pey (CoDi QR, CoDi push, links de pago, cuentas bancarias, transacciones y webhooks) en una plataforma SaaS, multitenant o no. Usar al cobrar con CoDi o pey.mx."
---

# Integración de pey (API CoDi®) en una plataforma SaaS

Este skill guía a un agente de código para integrar la API de **pey** (https://pey.mx/api-codi) en un SaaS: cobros CoDi® por QR, cobros CoDi por notificación push al celular, links de pago hospedados (CoDi + tarjeta + Apple Pay + Google Pay), registro de cuentas CLABE por tenant, consulta de transacciones y recepción de webhooks firmados.

Snapshot del contrato: **OpenAPI "CoDi API" versión `2026-10-20`**. La API es joven y cambia: antes de escribir código, vuelve a leer `https://docs.pey.mx/openapi.json` (WebFetch, por partes; mide ~130 KB) y el changelog `https://docs.pey.mx/docs/category/changelog`. Si algo difiere de este documento, **manda la spec**.

## 0. Antes de empezar: detectar y decidir

1. Detecta el stack del proyecto (composer.json / package.json / pyproject). Hay implementación de referencia en Laravel (sección 8); para otros stacks traduce los mismos componentes.
2. Busca si ya existe una capa de pagos (interfaz `PaymentGateway`, tablas `payments`/`orders`, otros proveedores como Openpay o Stripe). Integra pey como **un driver más** detrás de esa abstracción; no la dupliques.
3. **Decide el modelo de tenencia con el usuario** (pregunta si no está claro; cambia todo el diseño):

| Modelo | Cómo funciona | Cuándo usarlo |
|---|---|---|
| **A. Plataforma única + cuenta CLABE por tenant** (recomendado para SaaS) | El SaaS tiene UNA API key. Cada tenant registra su CLABE vía `POST /bank-account`; cada QR/push se crea con `bankAccountId` del tenant, así el dinero cae directo en la cuenta del tenant. Conciliación por tenant con `GET /transaction/bank-account/{id}`. | Marketplace / SaaS donde el tenant cobra a sus clientes y tú no quieres que los tenants creen cuenta en pey. |
| **B. API key por tenant** | Cada tenant crea su cuenta pey y su API key en el portal y la captura en tu SaaS. Cada key tiene su propio webhook y secreto. | El tenant ya es cliente de pey, o necesitas links de pago liquidados a su nombre (los links NO aceptan `bankAccountId`). |

Limitación clave: `bankAccountId` solo existe en **CoDi QR y CoDi push**. Los **links de pago** liquidan a la cuenta de pago ligada a la API key. Si el tenant necesita links de pago a su nombre, usa el modelo B (o híbrido).

## 1. Fundamentos

- **Base URL:** `https://api.pey.mx` (solo producción; el sandbox está "próximamente").
- **Auth:** `Authorization: Bearer <API_KEY>`. Enviar también `Content-Type: application/json` y `Accept: application/json`.
- **API keys:** se crean en `https://portal.mipey.mx/developers` ("Crear API Key"), ligadas a una cuenta de pago de tipo `codi` y/o `stripe` (al menos una). El alta bancaria con Banxico tarda ~5 días hábiles.
- **Moneda:** MXN. Los montos son **números decimales en pesos** (no centavos). Guarda internamente en centavos enteros o `DECIMAL(12,2)` y convierte al enviar.
- **IDs:** transacciones, cuentas y links usan **ULID** (26 chars). El QR CoDi usa un **IDC** de Banxico (string corto, ej. `35a13c183a`). El push devuelve `codiId` = folio Banxico de 20 dígitos.
- **Fechas:** Unix timestamp en **segundos** (`dateCreated`, `datePaid`).
- **Costo:** actualmente sin costo por uso según pey; aun así `amountBreakdown.fee` y `applicationFee` existen en las respuestas: modélalos.
- **Sin idempotency key** en la API y sin rate limits documentados (ver sección 6 para reintentos seguros).

### Sobre de error (todas las rutas)
```json
{ "error": { "message": "clabe is not valid", "type": "INVALID_CLABE", "requestId": "abc123-..." } }
```
Mapea `error.type` a excepciones de dominio y **loguea siempre `requestId`** (es lo que pide soporte de pey).

## 2. Endpoints

### CoDi

**`POST /codi/qr`** — Crear QR CoDi → `201`
| Campo | Tipo | Regla |
|---|---|---|
| `amount` | number, **requerido** | ≥ 0. `0` = el pagador captura el monto (QR de monto libre). |
| `concept` | string | Texto que ve el pagador. Sin maxLength documentado; mantenlo corto (≤ 40 recomendable). |
| `expiration` | integer | Segundos. Default `0` = **no expira**. Si se fija, **mínimo 360**. |
| `numRef` | string | Regex `^[0-9]{1,7}$`, default `"0"`. Siempre string. |
| `generateImage` | boolean | Default `false`. Si `true`, la respuesta trae `qrImage` (PNG base64). |
| `bankAccountId` | ULID | Cuenta destino; debe estar `codiStatus: ACTIVE`. |
| `metadata` | objeto | ≤ 50 pares, llaves ≤ 40 chars, valores escalares ≤ 500 chars. |

Respuesta: `{ id (IDC), qrString, qrImage (base64 o null) }`. Errores 400: `INVALID_REQUEST`, `PEY_CONSISTENCY_ERROR`, `INVALID_STATE`, `INVALID_AMOUNT`, `INVALID_PARAMETER`. 503: `SERVICE_UNAVAILABLE`, `FLAGS_UNAVAILABLE`. 500: `INTERNAL_SERVER_ERROR`.

Render: codifica `qrString` tal cual (es un JSON con llaves como `TYP`, `v`, `ic`, `CRY`; **no lo parsees ni lo reformatees**). pey recomienda corrección de errores **M**, versión automática, negro sobre blanco, borde 4. Alternativa: `generateImage: true` y mostrar `data:image/png;base64,{qrImage}`.

**`POST /codi/pn`** — Cobro por notificación push al celular del pagador (vía Banxico) → `200`
| Campo | Tipo | Regla |
|---|---|---|
| `amount` | number, **requerido** | > 0 y ≤ 1,000,000 |
| `phoneNumber` | string, **requerido** | `^[0-9]{10}$` (10 dígitos, sin +52) |
| `concept`, `numRef`, `bankAccountId`, `metadata` | | Igual que QR |
| `expiration` | integer | Default **360**, mínimo 360 |

Respuesta: `{ transactionId (ULID), codiId (folio Banxico 20 dígitos) }`. Errores 400 adicionales: `INCOMPLETE_PARAMETERS`, `BAD_REQUEST`.

### Cuentas bancarias (CLABE para recibir CoDi)

**`POST /bank-account`** → `200 { id, codiStatus }`
- Body (todos requeridos): `alias` (1–255), `name` (titular legal, 1–255), `clabe` (`^[0-9]{18}$` con dígito verificador válido), `entity` (`BUSINESS` | `PERSON`).
- Nace en `PENDING`; pasa por `PROCESSING` → `ACTIVE` (hasta 5 días hábiles) o `ERROR`.
- Error específico: `INVALID_CLABE`. Valida el dígito verificador en tu formulario antes de llamar.
- `GET` devuelve `taxId` (RFC) pero el `POST` **no** lo acepta en la spec actual: no lo envíes salvo que la spec cambie.

**`GET /bank-account/{id}`** → `{ id, alias, name, clabe, entity, taxId?, codiStatus, errorMessage? (solo si ERROR), dateCreated }`. 404: `BANK_ACCOUNT_NOT_FOUND`.

### Links de pago (checkout hospedado: CoDi, tarjeta, Apple Pay, Google Pay)

**`POST /payment-link/create`** → `200 { id, url }`
- `amount` (requerido, > 0), `successUrl` y `backUrl` (requeridos, **HTTPS**), `hasTip` (default false), `clientReferenceId` (≤ 255; pon tu ID de orden), `metadata`.
- Errores 400: `AUTH_ERROR`, `PEY_CONSISTENCY_ERROR`, `REQUEST_PARSING_ERROR`, `INCOMPLETE_PARAMETERS`, `INVALID_URL`, `INVALID_AMOUNT`, `INVALID_HAS_TIP`, `INVALID_CLIENT_REFERENCE_ID`.

**`POST /payment-link/update`** → `200 { id, amount }`
- Body estricto (`additionalProperties: false`): `paymentLinkId`, `amount`. Solo cambia el monto, solo si el link está `pending`. Redondeo a 2 decimales half-up.
- 403 `FORBIDDEN`, 404 `NOT_FOUND`, 409 `PAYMENT_LINK_NOT_PENDING` o `CONCURRENT_UPDATE` (reintenta tras re-leer el link).

**`GET /payment-link/{id}`** → `{ id, amount, status (pending|processing|paid|partial), successUrl, backUrl, hasTip, paymentMethods[], dateCreated, clientReferenceId?, metadata?, transactions[] }`
- Cada transacción: `transactionId, paymentMethod, datePaid, amount, amountBreakdown { base, tip?, fee? }, applicationFee (o null), metadata`.

### Transacciones

| Endpoint | Uso | Paginación |
|---|---|---|
| `GET /transaction/{id}` | Una transacción (ULID). | — |
| `GET /transaction/api-key?nextToken=` | Todas las de la API key. | 80 por página, más recientes primero |
| `GET /transaction/bank-account/{id}?nextToken=` | **Solo PAID** de una cuenta (QR + push). Ideal para conciliar por tenant en modelo A. | 80 por página |
| `GET /transaction/codi/qr/{idc}?nextToken=` | Transacciones de un QR (PENDING o PAID). Un QR puede tener **varios pagos**. | 20 por página |

Objeto transacción: `id, status (PENDING|PAID), amount, amountBreakdown { base, tip, fee }, currency, paymentMethod (CARD|CODI), payableType (PAYMENT_LINK|API_CODI_QR|API_CODI_PN), payableId, description, applicationFee, dateCreated, datePaid?, metadata`.

Paginación: repite con `nextToken` hasta que sea `null`. Omítelo en la primera página.

### `metadata` en respuestas
- Solo CoDi: `clabe`, `bankCode` (Banxico, 5 dígitos), `accountType` (2 dígitos), `description`, `trackingKey` (clave de rastreo Banxico; guárdala, sirve para aclaraciones).
- `externalMetadata`: lo que tú mandaste en `metadata` al crear. **Todos los valores regresan como string** (`true` → `"true"`, `123` → `"123"`).

## 3. Webhooks

- **Uno por API key**. Se configura en el portal (engrane de la key → URL → "Crear Webhook"). El secreto se muestra una vez y se puede rotar. Historial y payloads en "Webhook Logs".
- Firmados con **Standard Webhooks**: headers `webhook-id` (clave de idempotencia), `webhook-timestamp` (Unix s), `webhook-signature` (lista separada por espacios de `v1,<base64>`). Firma = base64(HMAC-SHA256(`{id}.{timestamp}.{rawBody}`, base64decode(secreto sin prefijo `whsec_`))).
- Librerías oficiales: npm `standardwebhooks`, pip `standardwebhooks`, composer `standard-webhooks/standard-webhooks`. O el verificador nativo de la sección 8 (probado contra la librería oficial de Python).
- **Verifica sobre el cuerpo crudo** (bytes exactos), nunca sobre JSON re-serializado. Tolerancia de timestamp: 5 min.
- La política de reintentos de pey no está documentada: responde **2xx rápido** (encola el procesamiento) y haz el handler idempotente.

| `type` | Cuándo | Campos |
|---|---|---|
| `codi.qr.payment_received` | Se pagó un QR | `amount, apiKeyId, bankAccountId?, codiQrId, concept, datePaid, metadata, numRef, transactionId, type` |
| `codi.pn.payment_received` | Se pagó un push | `amount, apiKeyId, bankAccountId?, concept, datePaid, metadata, numRef, transactionId, type` (sin `codiQrId`) |
| `payment_link.status_updated` | Cambia el status del link | `apiKeyId, type, paymentLinkId, status (paid|pending|processing), clientReferenceId, transactions[]` |
| `bank.account.updated` | Cuenta pasa a `ACTIVE` (solo esa transición) | `type, bankAccountId, name, ownerType (API_KEY), ownerId, taxId?, codiStatus (ACTIVE), clabe, entity` |

`bankAccountId` en los webhooks CoDi solo viene si lo enviaste al crear el cobro. `bank.account.updated` NO avisa de `ERROR`: para detectar rechazos haz polling de `GET /bank-account/{id}` (p. ej. diario) mientras esté `PENDING`/`PROCESSING`.

### Pipeline de procesamiento (obligatorio)
1. Verificar firma (secreto según modelo: global en A; por tenant en B, resuelto por la URL `/webhooks/pey/{tenant}` o por `apiKeyId`). Firma inválida → 401/400, no procesar.
2. Dedupe: insertar `webhook-id` en `pey_webhook_events` con índice único; si ya existe → 200 sin reprocesar.
3. Encolar job y responder 200.
4. En el job: **confirmar con la API** (`GET /transaction/{transactionId}` o `GET /payment-link/{id}`) antes de marcar pagado. El webhook es la señal; la API es la fuente de verdad.
5. Resolver la orden: `metadata.externalMetadata.orderId` (string) o `clientReferenceId` o el `payable` guardado (IDC del QR, `transactionId` del push, `id` del link). Validar que el `apiKeyId`/`bankAccountId` pertenece al tenant esperado.
6. Aplicar transición de estado de forma idempotente (dedupe también por `transactionId`, único en `pey_transactions`). Emitir evento de dominio (`PaymentReceived`).

## 4. Modelo de datos sugerido

- `pey_accounts` (modelo B): `tenant_id`, `api_key` (cifrada), `webhook_secret` (cifrado), `api_key_id` (el `apiKeyId` de los webhooks, para enrutar).
- `pey_bank_accounts` (modelo A): `tenant_id`, `pey_id` (ULID, único), `alias`, `name`, `clabe_last4` (no guardes la CLABE completa en texto si no es necesario; o cífrala), `entity`, `codi_status`, `error_message`, `activated_at`.
- `pey_charges`: `tenant_id`, `order_id`, `kind` (`qr|pn|link`), `pey_id` (IDC, transactionId o linkId), `amount`, `status` (`pending|paid|expired|failed|partial`), `expires_at`, `qr_string`, `link_url`, `num_ref`, `request_id` del último error.
- `pey_transactions`: `pey_transaction_id` (único), `charge_id`, `status`, `amount`, `base`, `tip`, `fee`, `application_fee`, `payment_method`, `tracking_key`, `paid_at`, `raw` (JSON).
- `pey_webhook_events`: `webhook_id` (único), `type`, `payload` (JSON), `processed_at`, `error`.

## 5. Flujos

**Checkout CoDi QR (pantalla/kiosco/POS):** crear orden → `POST /codi/qr` con `amount` exacto, `expiration` ≥ 360 (p. ej. 900), `numRef` derivado de la orden (≤ 7 dígitos), `metadata.orderId`, `bankAccountId` del tenant (modelo A) → mostrar QR + contador de expiración → esperar webhook; en UI, polling ligero a TU backend (no a pey) cada 3–5 s, o websockets/broadcast → al confirmar, pantalla de éxito. Al expirar, ofrece "Generar nuevo QR" (crea uno nuevo; no reutilices).

**Cobro push:** capturar celular a 10 dígitos → `POST /codi/pn` → guarda `transactionId` y `codiId` → el pagador aprueba en su app bancaria dentro de `expiration` → webhook `codi.pn.payment_received`. Si vence sin webhook, confirma con `GET /transaction/{transactionId}` antes de marcar `expired`.

**Link de pago (e-commerce, cobros remotos, tarjeta):** `POST /payment-link/create` con `clientReferenceId = order id` y URLs HTTPS → redirige al `url` → al volver a `successUrl` **NO marques pagado** (el redirect no es prueba de pago): muestra "procesando" y espera `payment_link.status_updated` o consulta `GET /payment-link/{id}`.

**Onboarding de tenant (modelo A):** formulario CLABE con validación de dígito verificador → `POST /bank-account` → estado "En validación con Banxico (hasta 5 días hábiles)" → webhook `bank.account.updated` activa la cuenta → habilitar cobros CoDi del tenant. Bloquea la creación de QR/push mientras la cuenta no sea `ACTIVE` (si no, `INVALID_STATE`).

## 6. Errores, reintentos y concurrencia

- **No hay idempotency key.** No reintentes ciegamente un `POST` que pudo haberse ejecutado (timeout de red): en push podrías mandar dos solicitudes de cobro al celular. Para creates: reintenta solo ante 503 o error de conexión **antes de enviar**; ante timeout de lectura, marca el cargo como `unknown` y concilia con `GET /transaction/api-key` buscando tu `metadata.orderId`.
- `GET`: reintento con backoff exponencial ante 5xx/503 (`SERVICE_UNAVAILABLE`, `FLAGS_UNAVAILABLE`), máx. 3.
- 4xx de validación: no reintentar; mapear a mensajes de usuario (`INVALID_AMOUNT`, `INVALID_CLABE`, `INVALID_URL`, `INVALID_STATE` = cuenta no activa, etc.).
- 409 `CONCURRENT_UPDATE` en link: re-leer y reintentar una vez. `PAYMENT_LINK_NOT_PENDING`: crear un link nuevo.
- Timeouts HTTP: connect 5 s, total 15–20 s.
- Lock por orden (p. ej. `Cache::lock("order:{id}:pay")`) para no crear dos cobros simultáneos para la misma orden.

## 7. Trampas conocidas (revisar en code review)

1. `expiration: 0` en QR = **no expira** y el QR puede recibir **múltiples pagos**. Para órdenes, usa siempre expiración ≥ 360 y trata pagos extra como sobrepago a reembolsar/aclarar.
2. QR con `amount: 0` = monto libre: el monto real es el del webhook/transacción; nunca asumas el de la orden. Úsalo solo para donativos/propinas/cobro abierto.
3. `expiration` entre 1 y 359 es inválido; en push el mínimo y default es 360.
4. `numRef` es **string numérico de 1–7 dígitos**; no mandes enteros ni IDs alfanuméricos. Para trazabilidad usa `metadata`.
5. En links, `amount` pagado = `base + tip + fee`. Para conciliar contra la orden usa `amountBreakdown.base`; registra `tip` y `fee` aparte. `applicationFee` puede ser `null`.
6. `status: partial` solo aparece en `GET /payment-link`; el webhook documenta `paid|pending|processing`. Maneja `partial` igual.
7. `externalMetadata` regresa todo como string; compara con `(string)`.
8. Máx. 50 llaves de metadata; no metas PII innecesaria (email, nombre) si no la ocupas.
9. Un solo webhook por API key: si el SaaS tiene varios entornos (staging/prod) que comparten key, solo uno recibe eventos. Usa keys distintas por entorno.
10. No hay sandbox: los tests automatizados usan HTTP fake; las pruebas reales se hacen en producción con **$0.01 MXN** (pey ofrece una CLABE de prueba en la guía "Crear una API key"; los pagos a ella no se devuelven).
11. Nunca registres el API key, el secreto ni la CLABE completa en logs. Cifra secretos por tenant en BD.
12. Los timestamps son segundos, no milisegundos. Zona horaria del negocio: `America/Mexico_City` (o la del tenant) al mostrar.

## 8. Implementación de referencia (Laravel 11+/12)

Adáptala a la estructura del proyecto (Actions, Services, Spatie Data, etc.).

`config/services.php`:
```php
'pey' => [
    'base_url' => env('PEY_BASE_URL', 'https://api.pey.mx'),
    'api_key' => env('PEY_API_KEY'),            // modelo A (o default)
    'webhook_secret' => env('PEY_WEBHOOK_SECRET'),
    'default_qr_expiration' => (int) env('PEY_QR_EXPIRATION', 900),
],
```

Cliente:
```php
namespace App\Payments\Pey;

use Illuminate\Http\Client\PendingRequest;
use Illuminate\Http\Client\Response;
use Illuminate\Support\Facades\Http;

final class PeyClient
{
    public function __construct(private string $apiKey, private string $baseUrl = 'https://api.pey.mx') {}

    private function http(): PendingRequest
    {
        return Http::baseUrl($this->baseUrl)
            ->withToken($this->apiKey)
            ->acceptJson()->asJson()
            ->connectTimeout(5)->timeout(20);
    }

    private function get(string $path, array $query = []): array
    {
        $res = $this->http()
            ->retry(3, fn (int $attempt) => 200 * (2 ** $attempt), fn ($e) => $e instanceof \Illuminate\Http\Client\ConnectionException
                || ($e instanceof \Illuminate\Http\Client\RequestException && $e->response->serverError()), throw: false)
            ->get($path, $query);
        return $this->unwrap($res);
    }

    private function post(string $path, array $body): array
    {
        // Sin reintento automático: la API no tiene idempotency key.
        return $this->unwrap($this->http()->post($path, $body));
    }

    private function unwrap(Response $res): array
    {
        if ($res->successful()) return $res->json() ?? [];
        $err = $res->json('error') ?? [];
        throw new PeyException(
            message: $err['message'] ?? 'pey request failed',
            type: $err['type'] ?? 'UNKNOWN',
            requestId: $err['requestId'] ?? null,
            status: $res->status(),
        );
    }

    public function createQr(float $amount, string $concept, int $expiration, string $numRef = '0', ?string $bankAccountId = null, array $metadata = [], bool $generateImage = false): array
    {
        return $this->post('/codi/qr', array_filter([
            'amount' => round($amount, 2),
            'concept' => $concept,
            'expiration' => $expiration,
            'numRef' => $numRef,
            'generateImage' => $generateImage,
            'bankAccountId' => $bankAccountId,
            'metadata' => $metadata ?: null,
        ], fn ($v) => $v !== null));
    }

    public function createPushNotification(float $amount, string $phone10, string $concept, int $expiration = 360, string $numRef = '0', ?string $bankAccountId = null, array $metadata = []): array
    {
        return $this->post('/codi/pn', array_filter([
            'amount' => round($amount, 2), 'phoneNumber' => $phone10, 'concept' => $concept,
            'expiration' => max(360, $expiration), 'numRef' => $numRef,
            'bankAccountId' => $bankAccountId, 'metadata' => $metadata ?: null,
        ], fn ($v) => $v !== null));
    }

    public function registerBankAccount(string $alias, string $name, string $clabe, string $entity): array
    { return $this->post('/bank-account', compact('alias', 'name', 'clabe', 'entity')); }

    public function getBankAccount(string $id): array { return $this->get("/bank-account/{$id}"); }

    public function createPaymentLink(float $amount, string $successUrl, string $backUrl, ?string $clientReferenceId = null, bool $hasTip = false, array $metadata = []): array
    {
        return $this->post('/payment-link/create', array_filter([
            'amount' => round($amount, 2), 'successUrl' => $successUrl, 'backUrl' => $backUrl,
            'hasTip' => $hasTip, 'clientReferenceId' => $clientReferenceId, 'metadata' => $metadata ?: null,
        ], fn ($v) => $v !== null));
    }

    public function updatePaymentLinkAmount(string $paymentLinkId, float $amount): array
    { return $this->post('/payment-link/update', ['paymentLinkId' => $paymentLinkId, 'amount' => round($amount, 2)]); }

    public function getPaymentLink(string $id): array { return $this->get("/payment-link/{$id}"); }
    public function getTransaction(string $id): array { return $this->get("/transaction/{$id}"); }

    /** @return \Generator<array> */
    public function paginate(string $path): \Generator
    {
        $token = null;
        do {
            $page = $this->get($path, array_filter(['nextToken' => $token]));
            yield from $page['items'] ?? [];
            $token = $page['nextToken'] ?? null;
        } while ($token);
    }
    // Uso: paginate('/transaction/api-key'), paginate("/transaction/bank-account/{$id}"), paginate("/transaction/codi/qr/{$idc}")
}
```

`PeyException` extiende `RuntimeException` con propiedades `type`, `requestId`, `status`. Registra `PeyClient` en un service provider; en modelo B, resuélvelo con una factory `PeyClientFactory::forTenant($tenant)` que lee la key cifrada.

Verificador de webhooks (nativo, equivalente a la librería oficial):
```php
namespace App\Payments\Pey;

final class PeyWebhookVerifier
{
    public function __construct(private int $toleranceSeconds = 300) {}

    public function verify(string $payload, string $id, string $timestamp, string $signatureHeader, string $secret): array
    {
        if ($id === '' || $timestamp === '' || $signatureHeader === '') throw new InvalidWebhook('Missing headers');
        if (!ctype_digit($timestamp) || abs(time() - (int) $timestamp) > $this->toleranceSeconds) throw new InvalidWebhook('Stale timestamp');

        $raw = str_starts_with($secret, 'whsec_') ? substr($secret, 6) : $secret;
        $key = base64_decode($raw, true);
        if ($key === false) throw new InvalidWebhook('Secret is not base64');

        $expected = base64_encode(hash_hmac('sha256', "{$id}.{$timestamp}.{$payload}", $key, true));
        foreach (explode(' ', trim($signatureHeader)) as $part) {
            [$version, $sig] = array_pad(explode(',', $part, 2), 2, '');
            if ($version === 'v1' && hash_equals($expected, $sig)) {
                return json_decode($payload, true, 512, JSON_THROW_ON_ERROR);
            }
        }
        throw new InvalidWebhook('Invalid signature');
    }
}
```
Si el secreto que entrega el portal no fuera base64 con prefijo `whsec_`, confirma el formato con un evento real de "Webhook Logs" o usa la librería oficial `standard-webhooks/standard-webhooks`.

Controller (excluir de CSRF; ruta `POST /webhooks/pey` o `/webhooks/pey/{tenant}`):
```php
public function __invoke(Request $request, PeyWebhookVerifier $verifier)
{
    $secret = /* modelo A: config('services.pey.webhook_secret'); modelo B: secreto cifrado del tenant */;
    try {
        $event = $verifier->verify(
            $request->getContent(),
            (string) $request->header('webhook-id'),
            (string) $request->header('webhook-timestamp'),
            (string) $request->header('webhook-signature'),
            $secret,
        );
    } catch (InvalidWebhook $e) {
        Log::warning('pey webhook rejected', ['reason' => $e->getMessage()]);
        return response()->noContent(401);
    }

    $stored = PeyWebhookEvent::query()->insertOrIgnore([
        'webhook_id' => $request->header('webhook-id'),
        'type' => $event['type'] ?? 'unknown',
        'payload' => json_encode($event),
        'created_at' => now(), 'updated_at' => now(),
    ]);
    if ($stored) ProcessPeyWebhook::dispatch($request->header('webhook-id'));

    return response()->noContent(200);
}
```

Job `ProcessPeyWebhook` (`ShouldQueue`, `tries=5`, backoff): carga el evento, hace `match ($event['type'])` sobre los 4 tipos, confirma con `PeyClient::getTransaction()` / `getPaymentLink()` / `getBankAccount()`, actualiza `pey_charges`/`pey_transactions`/`pey_bank_accounts` en transacción DB con `lockForUpdate`, marca `processed_at` y dispara eventos de dominio. Tipos desconocidos: guardar y marcar procesado (no fallar).

Comando de conciliación `pey:reconcile` (scheduler cada 15 min): para cargos `pending|unknown` vencidos, consulta la API y corrige estado; para cuentas `PENDING|PROCESSING`, `GET /bank-account/{id}` para detectar `ERROR`.

Filament (si el proyecto lo usa): recurso `PeyCharge` de solo lectura con filtros por tenant/estado, acción "Conciliar con pey", y en el panel del tenant una página de "Cuenta CLABE para CoDi" con badge de `codiStatus`.

## 9. Pruebas

- Unit/feature con `Http::fake([...])` usando los ejemplos de esta guía, incluyendo cada `error.type` relevante y el sobre de error.
- Webhooks: genera firmas en el test con la misma fórmula (o la librería oficial) y prueba: firma válida, cuerpo alterado, timestamp viejo, múltiples firmas en el header, `webhook-id` duplicado (procesa una sola vez), evento de otro tenant/apiKeyId (rechazar o ignorar).
- Flujos: QR expirado, pago doble sobre el mismo QR, push vencido, link `partial`, `CONCURRENT_UPDATE`, cuenta en `ERROR`.
- E2E manual en producción: API key real, monto $0.01, webhook apuntando a un túnel (ngrok/Cloudflare Tunnel) o a staging con key propia; revisar "Webhook Logs" en el portal.

## 10. Definition of done

- [ ] Modelo de tenencia (A/B) decidido y documentado.
- [ ] Cliente con manejo del sobre de error, `requestId` en logs y sin reintentos ciegos en POST.
- [ ] Webhook verificado sobre cuerpo crudo, deduplicado por `webhook-id`, procesado en cola y confirmado contra la API.
- [ ] Órdenes nunca se marcan pagadas por redirect ni por monto supuesto.
- [ ] Expiraciones ≥ 360 en cobros ligados a órdenes; manejo de sobrepago/pagos múltiples.
- [ ] Conciliación programada y detección de cuentas en `ERROR`.
- [ ] Secretos cifrados, nada sensible en logs, keys separadas por entorno.
- [ ] Tests verdes para todos los casos de la sección 9.

## Fuentes
- Producto: https://pey.mx/api-codi
- Docs: https://docs.pey.mx · OpenAPI: https://docs.pey.mx/openapi.json
- API key: https://docs.pey.mx/docs/tutorials/getting-started/create-an-api-key
- Webhooks: https://docs.pey.mx/docs/webhooks/introduction · Standard Webhooks: https://www.standardwebhooks.com
- Portal: https://portal.mipey.mx
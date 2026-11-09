# Implementación de referencia en Laravel (11+/12)

Referencia de la sección 8 de `SKILL.md`. Léela solo cuando el proyecto use Laravel; en otros stacks traduce los mismos componentes (cliente HTTP, verificador de firmas, controller de webhook, job en cola y comando de conciliación).

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

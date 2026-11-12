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
    public function __construct(
        private string $apiKey,
        private string $baseUrl = 'https://api.pey.mx',
        private int $defaultQrExpiration = 900,   // s; se usa si createQr no recibe expiración
    ) {}

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

    public function createQr(float $amount, string $concept, ?int $expiration = null, string $numRef = '0', ?string $bankAccountId = null, array $metadata = [], bool $generateImage = false): array
    {
        // Sin expiración pey crea un QR que NUNCA expira y acepta varios pagos: aplica el default.
        // 0 solo se permite si lo pides explícitamente; 1-359 es inválido para pey.
        $expiration ??= $this->defaultQrExpiration;
        if ($expiration !== 0 && $expiration < 360) {
            throw new \InvalidArgumentException("expiration must be 0 (never expires) or >= 360, got {$expiration}");
        }

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
        if ($expiration < 360) {
            throw new \InvalidArgumentException("push expiration must be >= 360, got {$expiration}");
        }

        return $this->post('/codi/pn', array_filter([
            'amount' => round($amount, 2), 'phoneNumber' => $phone10, 'concept' => $concept,
            'expiration' => $expiration, 'numRef' => $numRef,
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
    $webhookId = (string) $request->header('webhook-id');

    try {
        $event = $verifier->verify(
            $request->getContent(),                       // cuerpo crudo
            $webhookId,
            (string) $request->header('webhook-timestamp'),
            (string) $request->header('webhook-signature'),
            $this->secretFor($request),
        );
    } catch (InvalidWebhook $e) {
        Log::warning('pey webhook rejected', ['reason' => $e->getMessage()]);
        return response()->noContent(401);
    }

    PeyWebhookEvent::query()->insertOrIgnore([
        'webhook_id' => $webhookId,
        'type' => $event['type'] ?? 'unknown',
        'payload' => json_encode($event),
        'created_at' => now(), 'updated_at' => now(),
    ]);

    // Decide por "¿ya se procesó?", no por "¿ya existía?": si un intento anterior guardó el evento pero falló
    // al encolar, el reintento de pey debe volver a encolarlo (si no, el evento se pierde).
    $pending = PeyWebhookEvent::query()->where('webhook_id', $webhookId)->whereNull('processed_at')->exists();
    if ($pending) {
        ProcessPeyWebhook::dispatch($webhookId);   // si falla, la excepción devuelve 500 y pey reintenta
    }

    return response()->noContent(200);
}

private function secretFor(Request $request): string
{
    // Modelo A: un secreto global. Modelo B: resuelve el tenant por la ruta o por apiKeyId y descifra su secreto.
    return (string) config('services.pey.webhook_secret');
}
```

Migración de eventos:
```php
Schema::create('pey_webhook_events', function (Blueprint $t) {
    $t->id();
    $t->string('webhook_id')->unique();   // clave de idempotencia de pey
    $t->string('type');
    $t->json('payload');
    $t->timestamp('processed_at')->nullable();
    $t->text('error')->nullable();
    $t->timestamps();
});
```
El modelo `PeyWebhookEvent` castea `payload` a `array` y `processed_at` a `datetime`.

Registro idempotente del pago (compartido por el job y la conciliación):
```php
namespace App\Payments\Pey;

use Illuminate\Support\Facades\DB;

final class PeyPaymentRecorder
{
    /** @param array $tx objeto transacción de la API (GET /transaction/{id}) */
    public function record(array $tx, ?string $payableId = null): void
    {
        DB::transaction(function () use ($tx, $payableId) {
            // Bloquea el cargo para que dos procesos no marquen la misma orden a la vez.
            $charge = PeyCharge::query()->where('pey_id', $payableId ?? ($tx['payableId'] ?? ''))->lockForUpdate()->first();

            $record = PeyTransaction::query()->firstOrCreate(      // índice único en pey_transaction_id
                ['pey_transaction_id' => $tx['id']],
                [
                    'charge_id' => $charge?->id,
                    'status' => $tx['status'],
                    'amount' => $tx['amount'],
                    'base' => $tx['amountBreakdown']['base'] ?? $tx['amount'],   // concilia contra base, no contra amount
                    'tip' => $tx['amountBreakdown']['tip'] ?? 0,
                    'fee' => $tx['amountBreakdown']['fee'] ?? 0,
                    'application_fee' => $tx['applicationFee'] ?? null,
                    'payment_method' => $tx['paymentMethod'] ?? null,
                    'tracking_key' => $tx['metadata']['trackingKey'] ?? null,
                    'paid_at' => isset($tx['datePaid']) ? now()->setTimestamp($tx['datePaid']) : null,
                    'raw' => $tx,
                ],
            );

            if ($record->wasRecentlyCreated && $charge) {      // solo la primera vez
                $charge->update(['status' => 'paid']);
                event(new PaymentReceived($charge, $record));
            }
        });
    }
}
```

Job `ProcessPeyWebhook`:
```php
namespace App\Payments\Pey;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;

final class ProcessPeyWebhook implements ShouldQueue, ShouldBeUnique
{
    use Dispatchable, InteractsWithQueue, Queueable;

    public int $tries = 5;

    public function __construct(public string $webhookId) {}

    public function uniqueId(): string { return $this->webhookId; }          // no encolar dos veces el mismo evento
    public function backoff(): array { return [10, 60, 300, 900]; }

    public function handle(PeyClient $client, PeyPaymentRecorder $recorder): void
    {
        $row = PeyWebhookEvent::query()->where('webhook_id', $this->webhookId)->first();
        if (! $row || $row->processed_at) return;
        $event = $row->payload;

        match ($event['type'] ?? null) {
            'codi.qr.payment_received', 'codi.pn.payment_received' => $this->payment($client, $recorder, $event),
            'payment_link.status_updated' => $this->link($client, $recorder, $event),
            'bank.account.updated' => $this->bankAccount($client, $event),
            default => null,                                                  // tipo desconocido: guardar y marcar procesado
        };

        $row->update(['processed_at' => now(), 'error' => null]);
    }

    public function failed(\Throwable $e): void
    {
        PeyWebhookEvent::query()->where('webhook_id', $this->webhookId)->update(['error' => $e->getMessage()]);
    }

    private function payment(PeyClient $client, PeyPaymentRecorder $recorder, array $event): void
    {
        $tx = $client->getTransaction($event['transactionId']);               // la API es la fuente de verdad
        if (($tx['status'] ?? null) !== 'PAID') {
            throw new \RuntimeException("Transaction {$tx['id']} not PAID yet");   // el job reintenta
        }
        $payableId = $event['type'] === 'codi.qr.payment_received' ? $event['codiQrId'] : $event['transactionId'];
        $recorder->record($tx, $payableId);
    }

    private function link(PeyClient $client, PeyPaymentRecorder $recorder, array $event): void
    {
        $link = $client->getPaymentLink($event['paymentLinkId']);
        PeyCharge::query()->where('pey_id', $link['id'])->update(['status' => match ($link['status']) {
            'paid' => 'paid', 'partial' => 'partial', default => 'pending',   // processing se queda pendiente
        }]);
        foreach ($link['transactions'] as $t) {
            $recorder->record($t + ['id' => $t['transactionId'], 'status' => 'PAID'], $link['id']);
        }
    }

    private function bankAccount(PeyClient $client, array $event): void
    {
        $acc = $client->getBankAccount($event['bankAccountId']);
        if ($acc['codiStatus'] === 'ACTIVE') {
            PeyBankAccount::query()->where('pey_id', $acc['id'])->update(['codi_status' => 'ACTIVE', 'activated_at' => now()]);
        }
    }
}
```

Comando de conciliación `pey:reconcile` (programado cada 15 min en `routes/console.php`: `Schedule::command('pey:reconcile')->everyFifteenMinutes();`):
```php
namespace App\Console\Commands;

use App\Payments\Pey\{PeyBankAccount, PeyCharge, PeyClient, PeyPaymentRecorder};
use Illuminate\Console\Command;

final class PeyReconcile extends Command
{
    protected $signature = 'pey:reconcile';
    protected $description = 'Concilia cargos pendientes y cuentas CLABE con pey';

    public function handle(PeyClient $client, PeyPaymentRecorder $recorder): int
    {
        PeyCharge::query()->whereIn('status', ['pending', 'unknown'])->where('expires_at', '<', now())
            ->each(function (PeyCharge $ch) use ($client, $recorder) {
                if ($ch->kind === 'qr') {
                    foreach ($client->paginate("/transaction/codi/qr/{$ch->pey_id}") as $tx) {
                        if ($tx['status'] === 'PAID') { $recorder->record($tx, $ch->pey_id); return; }
                    }
                    $ch->update(['status' => 'expired']);
                } elseif ($ch->kind === 'pn') {
                    $tx = $client->getTransaction($ch->pey_id);
                    $tx['status'] === 'PAID' ? $recorder->record($tx, $ch->pey_id) : $ch->update(['status' => 'expired']);
                } else {
                    $link = $client->getPaymentLink($ch->pey_id);
                    foreach ($link['transactions'] as $t) {
                        $recorder->record($t + ['id' => $t['transactionId'], 'status' => 'PAID'], $link['id']);
                    }
                }
            });

        // bank.account.updated no avisa de ERROR: consulta las cuentas pendientes.
        PeyBankAccount::query()->whereIn('codi_status', ['PENDING', 'PROCESSING'])->each(function (PeyBankAccount $acc) use ($client) {
            $a = $client->getBankAccount($acc->pey_id);
            $acc->update(['codi_status' => $a['codiStatus'], 'error_message' => $a['errorMessage'] ?? null]);
        });

        return self::SUCCESS;
    }
}
```

Filament (si el proyecto lo usa): recurso `PeyCharge` de solo lectura con filtros por tenant/estado, acción "Conciliar con pey", y en el panel del tenant una página de "Cuenta CLABE para CoDi" con badge de `codiStatus`.

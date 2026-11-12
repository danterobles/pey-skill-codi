# Implementación de referencia en Node.js / TypeScript

Referencia de la sección 8 de `SKILL.md` para proyectos en Node.js (Express, Next.js, NestJS, Fastify, workers). Léela solo cuando el proyecto use Node.js o TypeScript. Los componentes son los mismos que en `references/laravel.md`: cliente HTTP, verificador de firmas, endpoint de webhook, job en cola y conciliación programada.

Requisitos: **Node.js 18 o superior** (usa `fetch` y `AbortSignal.timeout` globales; recomendado 20+). El cliente y el verificador no tienen dependencias. Código ESM con TypeScript estricto. Los archivos completos de esta guía (tipos, cliente, verificador, Express, Next.js, worker y conciliación) se verificaron con `tsc --strict` y con pruebas de `node:test`, incluida la verificación de firmas generadas por la librería oficial de Standard Webhooks. Los ejemplos de NestJS y Fastify son fragmentos para adaptar.

**JavaScript sin TypeScript:** el mismo código funciona quitando las anotaciones de tipo (o pasándolas a JSDoc). No uses `parameter properties` ni `enum`: el código está escrito para que Node pueda quitar los tipos sin compilar.

Ubicación sugerida: `src/payments/pey/` (adáptala a la estructura del proyecto).

## 1. Configuración

```bash
PEY_API_KEY=...            # modelo A (o default). En modelo B, la key cifrada vive por tenant en BD
PEY_WEBHOOK_SECRET=...     # secreto del webhook de esa key
PEY_BASE_URL=https://api.pey.mx
PEY_QR_EXPIRATION=900      # segundos, >= 360
```

Valida las variables al arrancar (zod, envalid o similar). Nunca las registres en logs.

## 2. Tipos del contrato (`types.ts`)

```ts
// Tipos del contrato de pey (OpenAPI 2026-10-20). Montos en pesos MXN (decimales), fechas en Unix segundos.

export type Metadata = Record<string, string | number | boolean | null>;

export interface CreateQrInput {
  amount: number;            // >= 0; 0 = monto libre
  concept?: string;
  expiration?: number;       // 0 = no expira (explícito); si se fija, >= 360. Omitido: el cliente usa su default (900)
  numRef?: string;           // ^[0-9]{1,7}$
  generateImage?: boolean;
  bankAccountId?: string;    // ULID de cuenta ACTIVE
  metadata?: Metadata;       // <= 50 pares
}
export interface CreateQrResponse { id: string; qrString: string; qrImage: string | null }

export interface CreatePushInput {
  amount: number;            // > 0 y <= 1,000,000
  phoneNumber: string;       // ^[0-9]{10}$
  concept?: string;
  expiration?: number;       // >= 360, default 360
  numRef?: string;
  bankAccountId?: string;
  metadata?: Metadata;
}
export interface CreatePushResponse { transactionId: string; codiId: string }

export type CodiStatus = 'PENDING' | 'PROCESSING' | 'ACTIVE' | 'ERROR';
export interface RegisterBankAccountInput { alias: string; name: string; clabe: string; entity: 'BUSINESS' | 'PERSON' }
export interface BankAccount {
  id: string; alias: string; name: string; clabe: string; entity: 'BUSINESS' | 'PERSON';
  taxId?: string; codiStatus: CodiStatus; errorMessage?: string; dateCreated: number;
}

export interface CreatePaymentLinkInput {
  amount: number; successUrl: string; backUrl: string;   // URLs HTTPS
  hasTip?: boolean; clientReferenceId?: string; metadata?: Metadata;
}
export interface AmountBreakdown { base: number; tip?: number; fee?: number }
export interface PaymentLinkTransaction {
  transactionId: string; paymentMethod: string; datePaid: number; amount: number;
  amountBreakdown: AmountBreakdown; applicationFee: number | null; metadata?: Record<string, unknown>;
}
export interface PaymentLink {
  id: string; amount: number; status: 'pending' | 'processing' | 'paid' | 'partial';
  successUrl: string; backUrl: string; hasTip: boolean; paymentMethods: string[];
  dateCreated: number; clientReferenceId?: string; metadata?: Record<string, string>;
  transactions: PaymentLinkTransaction[];
}

export interface TransactionMetadata {
  clabe?: string; bankCode?: number; accountType?: number; description?: string; trackingKey?: string;
  externalMetadata?: Record<string, string>; // los valores regresan SIEMPRE como string
}
export interface Transaction {
  id: string; status: 'PENDING' | 'PAID'; amount: number; amountBreakdown: AmountBreakdown;
  currency: string; paymentMethod: 'CARD' | 'CODI';
  payableType?: 'PAYMENT_LINK' | 'API_CODI_QR' | 'API_CODI_PN'; payableId?: string;
  description?: string; applicationFee?: number | null; dateCreated: number; datePaid?: number;
  metadata?: TransactionMetadata;
}
export interface Page<T> { items: T[]; nextToken: string | null }

// Webhooks: unión discriminada por `type`
interface CodiPaymentBase {
  amount: number; apiKeyId: string; bankAccountId?: string; concept: string;
  datePaid: number; metadata: TransactionMetadata; numRef: string; transactionId: string;
}
export type PeyWebhookEvent =
  | (CodiPaymentBase & { type: 'codi.qr.payment_received'; codiQrId: string })
  | (CodiPaymentBase & { type: 'codi.pn.payment_received' })
  | {
      type: 'payment_link.status_updated'; apiKeyId: string; paymentLinkId: string;
      status: 'paid' | 'pending' | 'processing'; clientReferenceId?: string;
      transactions: PaymentLinkTransaction[];
    }
  | {
      type: 'bank.account.updated'; bankAccountId: string; name: string; ownerType: string;
      ownerId: string; taxId?: string; codiStatus: 'ACTIVE'; clabe: string; entity: 'BUSINESS' | 'PERSON';
    };
```

## 3. Cliente (`client.ts`)

Reintenta solo los `GET` (5xx o error de red). Los `POST` nunca se reintentan solos: la API no tiene idempotency key.

```ts
import type {
  BankAccount, CreatePaymentLinkInput, CreatePushInput, CreatePushResponse, CreateQrInput,
  CreateQrResponse, Page, PaymentLink, RegisterBankAccountInput, Transaction,
} from './types.ts';

export class PeyError extends Error {
  readonly type: string;
  readonly requestId: string | null;
  readonly status: number;
  constructor(message: string, type: string, requestId: string | null, status: number) {
    super(message);
    this.name = 'PeyError';
    this.type = type;
    this.requestId = requestId;
    this.status = status;
  }
}

export interface PeyClientOptions {
  apiKey: string;
  baseUrl?: string;          // default https://api.pey.mx
  timeoutMs?: number;        // default 20000
  fetch?: typeof fetch;      // inyectable para pruebas
  defaultQrExpiration?: number; // default 900 s. Se usa cuando createQr no recibe `expiration` (omitirlo crearía un QR que nunca expira)
}

// pey exige >= 360 s cuando se fija una expiración. 0 significa "no expira" y solo se permite si lo pides explícitamente.
const MIN_EXPIRATION = 360;
function assertExpiration(seconds: number): void {
  if (!Number.isInteger(seconds) || seconds < 0 || (seconds > 0 && seconds < MIN_EXPIRATION)) {
    throw new RangeError(`expiration must be 0 (never expires) or an integer >= ${MIN_EXPIRATION}, got ${seconds}`);
  }
}

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

export class PeyClient {
  private readonly apiKey: string;
  private readonly baseUrl: string;
  private readonly timeoutMs: number;
  private readonly fetchImpl: typeof fetch;
  private readonly defaultQrExpiration: number;

  constructor(opts: PeyClientOptions) {
    if (!opts.apiKey) throw new Error('PEY_API_KEY is required');
    this.defaultQrExpiration = opts.defaultQrExpiration ?? 900;
    assertExpiration(this.defaultQrExpiration);
    this.apiKey = opts.apiKey;
    this.baseUrl = (opts.baseUrl ?? 'https://api.pey.mx').replace(/\/$/, '');
    this.timeoutMs = opts.timeoutMs ?? 20_000;
    this.fetchImpl = opts.fetch ?? fetch;
  }

  private async send<T>(method: 'GET' | 'POST', path: string, body?: unknown, query?: Record<string, string | undefined>): Promise<T> {
    const url = new URL(this.baseUrl + path);
    for (const [k, v] of Object.entries(query ?? {})) if (v) url.searchParams.set(k, v);
    const res = await this.fetchImpl(url, {
      method,
      headers: {
        Authorization: `Bearer ${this.apiKey}`,
        Accept: 'application/json',
        ...(body !== undefined ? { 'Content-Type': 'application/json' } : {}),
      },
      body: body !== undefined ? JSON.stringify(body) : undefined,
      signal: AbortSignal.timeout(this.timeoutMs),
    });
    const text = await res.text();
    // Un 502/504 de un gateway puede traer HTML: no lo parseamos a ciegas para no perder el status ni el sobre de error.
    let json: Record<string, any> = {};
    try {
      json = text ? JSON.parse(text) : {};
    } catch {
      if (res.ok) throw new PeyError('pey returned a non-JSON response', 'INVALID_RESPONSE', null, res.status);
    }
    if (res.ok) return json as T;
    const err = json?.error ?? {};
    throw new PeyError(err.message ?? `pey request failed (${res.status})`, err.type ?? 'UNKNOWN', err.requestId ?? null, res.status);
  }

  // GET: reintenta ante 5xx o error de red, con backoff exponencial.
  private async get<T>(path: string, query?: Record<string, string | undefined>, attempts = 3): Promise<T> {
    for (let i = 1; ; i++) {
      try {
        return await this.send<T>('GET', path, undefined, query);
      } catch (e) {
        const retryable = !(e instanceof PeyError) || e.status >= 500;
        if (!retryable || i >= attempts) throw e;
        await sleep(200 * 2 ** i);
      }
    }
  }

  // POST: SIN reintento automático. La API no tiene idempotency key.
  private post<T>(path: string, body: unknown): Promise<T> {
    return this.send<T>('POST', path, body);
  }

  async createQr(input: CreateQrInput) {
    // Sin `expiration` pey crea un QR que NUNCA expira y acepta varios pagos: aplicamos el default del cliente.
    const expiration = input.expiration ?? this.defaultQrExpiration;
    assertExpiration(expiration);
    return this.post<CreateQrResponse>('/codi/qr', { ...input, amount: round2(input.amount), expiration });
  }
  async createPushNotification(input: CreatePushInput) {
    const expiration = input.expiration ?? MIN_EXPIRATION;
    if (!Number.isInteger(expiration) || expiration < MIN_EXPIRATION) {
      throw new RangeError(`push expiration must be an integer >= ${MIN_EXPIRATION}, got ${expiration}`);
    }
    return this.post<CreatePushResponse>('/codi/pn', { ...input, amount: round2(input.amount), expiration });
  }
  registerBankAccount(input: RegisterBankAccountInput) {
    return this.post<{ id: string; codiStatus: BankAccount['codiStatus'] }>('/bank-account', input);
  }
  getBankAccount(id: string) {
    return this.get<BankAccount>(`/bank-account/${encodeURIComponent(id)}`);
  }
  createPaymentLink(input: CreatePaymentLinkInput) {
    return this.post<{ id: string; url: string }>('/payment-link/create', { ...input, amount: round2(input.amount) });
  }
  updatePaymentLinkAmount(paymentLinkId: string, amount: number) {
    return this.post<{ id: string; amount: number }>('/payment-link/update', { paymentLinkId, amount: round2(amount) });
  }
  getPaymentLink(id: string) {
    return this.get<PaymentLink>(`/payment-link/${encodeURIComponent(id)}`);
  }
  getTransaction(id: string) {
    return this.get<Transaction>(`/transaction/${encodeURIComponent(id)}`);
  }

  // Uso: for await (const tx of client.paginate('/transaction/api-key')) { ... }
  // Rutas: /transaction/api-key, /transaction/bank-account/{id}, /transaction/codi/qr/{idc}
  async *paginate(path: string): AsyncGenerator<Transaction> {
    let nextToken: string | null = null;
    do {
      const page: Page<Transaction> = await this.get<Page<Transaction>>(path, { nextToken: nextToken ?? undefined });
      yield* page.items;
      nextToken = page.nextToken;
    } while (nextToken);
  }
}

// La API recibe pesos con 2 decimales. Guarda centavos enteros y convierte al enviar.
export const round2 = (n: number) => Math.round((n + Number.EPSILON) * 100) / 100;
export const centsToPesos = (cents: number) => round2(cents / 100);
export const pesosToCents = (pesos: number) => Math.round(pesos * 100);
```

En modelo B crea una instancia por tenant (`new PeyClient({ apiKey: await keys.decryptFor(tenantId) })`) y no la guardes en un singleton global.

## 4. Verificador de webhooks (`webhook.ts`)

Sin dependencias, equivalente a la librería oficial. Alternativa: el paquete npm `standardwebhooks` (`new Webhook(secret).verify(rawBody, headers)`).

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';
import type { PeyWebhookEvent } from './types.ts';

export class InvalidWebhookError extends Error {}

export interface WebhookHeaders {
  id: string | null | undefined;          // webhook-id
  timestamp: string | null | undefined;   // webhook-timestamp
  signature: string | null | undefined;   // webhook-signature
}

// Verificación Standard Webhooks sin dependencias. `rawBody` son los bytes EXACTOS recibidos.
export function verifyPeyWebhook(
  rawBody: string | Buffer,
  headers: WebhookHeaders,
  secret: string,
  toleranceSeconds = 300,
): PeyWebhookEvent {
  const { id, timestamp, signature } = headers;
  if (!id || !timestamp || !signature) throw new InvalidWebhookError('Missing webhook headers');
  if (!/^\d+$/.test(timestamp) || Math.abs(Date.now() / 1000 - Number(timestamp)) > toleranceSeconds) {
    throw new InvalidWebhookError('Timestamp outside tolerance');
  }

  const key = Buffer.from(secret.startsWith('whsec_') ? secret.slice(6) : secret, 'base64');
  if (key.length === 0) throw new InvalidWebhookError('Secret is not valid base64');

  const payload = typeof rawBody === 'string' ? rawBody : rawBody.toString('utf8');
  const expected = createHmac('sha256', key).update(`${id}.${timestamp}.${payload}`).digest();

  const valid = signature.trim().split(' ').some((part) => {
    const [version, sig] = part.split(',', 2);
    if (version !== 'v1' || !sig) return false;
    const given = Buffer.from(sig, 'base64');
    return given.length === expected.length && timingSafeEqual(given, expected);
  });
  if (!valid) throw new InvalidWebhookError('Invalid signature');

  return JSON.parse(payload) as PeyWebhookEvent;
}

// Lee los tres headers de un objeto Headers (fetch/Next.js) o de un record (Express/Nest).
export function readWebhookHeaders(h: Headers | Record<string, string | string[] | undefined>): WebhookHeaders {
  const get = (name: string) => {
    if (typeof (h as Headers).get === 'function') return (h as Headers).get(name);
    const v = (h as Record<string, string | string[] | undefined>)[name];
    return Array.isArray(v) ? v[0] : v;
  };
  return { id: get('webhook-id'), timestamp: get('webhook-timestamp'), signature: get('webhook-signature') };
}
```

Regla clave: verifica sobre los **bytes crudos** del cuerpo. Si un middleware ya parseó el JSON, la firma no coincidirá.

## 5. Endpoint del webhook por framework

Todos siguen el mismo pipeline de la sección 3 de `SKILL.md`: verificar, deduplicar por `webhook-id`, encolar y responder 200 rápido. `store` (persistencia) y `enqueue` son piezas de tu proyecto.

### Express + BullMQ

```ts
import express from 'express';
import { Queue } from 'bullmq';
import { InvalidWebhookError, readWebhookHeaders, verifyPeyWebhook } from './webhook.ts';
import type { PeyStore } from './process.ts';

declare const store: PeyStore;
const connection = { host: process.env.REDIS_HOST ?? '127.0.0.1', port: 6379 };
export const peyQueue = new Queue('pey-webhooks', { connection });

export const app = express();

// La ruta del webhook va ANTES de express.json(), con el cuerpo crudo.
app.post('/webhooks/pey', express.raw({ type: 'application/json' }), async (req, res) => {
  const headers = readWebhookHeaders(req.headers);
  try {
    const event = verifyPeyWebhook(req.body as Buffer, headers, process.env.PEY_WEBHOOK_SECRET!);
    if (await store.saveWebhookEvent(headers.id!, event.type, event)) {
      await peyQueue.add('process', { webhookId: headers.id }, {
        jobId: headers.id!, attempts: 5, backoff: { type: 'exponential', delay: 2000 },
      });
    }
    res.sendStatus(200);
  } catch (e) {
    if (e instanceof InvalidWebhookError) { res.sendStatus(401); return; }
    // Express 4 no captura rechazos de handlers async: responde 5xx explícito para que pey reintente.
    console.error('pey webhook failed', e);
    res.sendStatus(500);
  }
});

app.use(express.json()); // el resto de la app
```

### Next.js (App Router)

```ts
// Next.js App Router: app/api/webhooks/pey/route.ts
import { InvalidWebhookError, readWebhookHeaders, verifyPeyWebhook } from './webhook.ts';
import type { PeyStore } from './process.ts';

declare const store: PeyStore;
declare function enqueue(webhookId: string): Promise<void>;

export const runtime = 'nodejs'; // node:crypto no existe en Edge

export async function POST(req: Request): Promise<Response> {
  const raw = await req.text(); // cuerpo crudo: NO uses req.json() antes de verificar
  const headers = readWebhookHeaders(req.headers);
  try {
    const event = verifyPeyWebhook(raw, headers, process.env.PEY_WEBHOOK_SECRET!);
    const isNew = await store.saveWebhookEvent(headers.id!, event.type, event);
    if (isNew) await enqueue(headers.id!);
    return new Response(null, { status: 200 });
  } catch (e) {
    if (e instanceof InvalidWebhookError) return new Response(null, { status: 401 });
    throw e;
  }
}
```

En Next.js sin Redis, encola con lo que ya use el proyecto (Inngest, Trigger.dev, QStash, una tabla de jobs) o procesa en la misma petición solo si el procesamiento es corto y sigue siendo idempotente.

### NestJS

Activa el cuerpo crudo al crear la app y léelo en el controller:

```ts
// main.ts
const app = await NestFactory.create(AppModule, { rawBody: true });

// pey-webhook.controller.ts
@Post('webhooks/pey')
@HttpCode(200)
async handle(@Req() req: RawBodyRequest<Request>) {
  const headers = readWebhookHeaders(req.headers as Record<string, string>);
  try {
    const event = verifyPeyWebhook(req.rawBody!, headers, this.config.getOrThrow('PEY_WEBHOOK_SECRET'));
    if (await this.store.saveWebhookEvent(headers.id!, event.type, event)) {
      await this.queue.add('process', { webhookId: headers.id }, { jobId: headers.id! });
    }
  } catch (e) {
    if (e instanceof InvalidWebhookError) throw new UnauthorizedException();
    throw e;
  }
}
```

### Fastify

Registra la ruta dentro de un plugin con un parser que entregue el cuerpo como string, para no afectar al resto de la app:

```ts
fastify.register(async (scope) => {
  scope.addContentTypeParser('application/json', { parseAs: 'string' }, (_req, body, done) => done(null, body));
  scope.post('/webhooks/pey', async (req, reply) => {
    const headers = readWebhookHeaders(req.headers as Record<string, string>);
    // verifyPeyWebhook(req.body as string, headers, secret) → saveWebhookEvent → enqueue → reply.code(200)
  });
});
```

## 6. Procesamiento en el worker (`process.ts`)

Framework-agnóstico. Define la interfaz de persistencia que tu ORM implementa y confirma cada evento contra la API antes de cambiar estado.

```ts
import type { PeyClient } from './client.ts';
import type { PeyWebhookEvent } from './types.ts';

// Persistencia que tu proyecto implementa (Prisma, Drizzle, TypeORM, Knex, SQL directo...).
export interface PeyStore {
  /**
   * Inserta el evento con índice único en webhook_id y devuelve si HAY QUE ENCOLARLO:
   * true si es nuevo o si ya existía pero sigue sin procesar (processed_at nulo; p. ej. falló el encolado antes);
   * false solo si ya se procesó. Decidir por "¿ya existía?" perdería eventos cuando falla el encolado.
   */
  saveWebhookEvent(webhookId: string, type: string, payload: unknown): Promise<boolean>;
  loadWebhookEvent(webhookId: string): Promise<PeyWebhookEvent | null>;
  markWebhookProcessed(webhookId: string, error?: string): Promise<void>;
  /** Registra el pago de forma idempotente (transactionId único) y marca la orden. Corre en transacción DB. */
  recordPayment(p: {
    transactionId: string; orderId: string | null; payableId: string | null;
    amount: number; base: number; tip: number; fee: number;
    trackingKey?: string; paidAt: number; raw: unknown;
  }): Promise<void>;
  updatePaymentLinkStatus(paymentLinkId: string, status: string): Promise<void>;
  activateBankAccount(bankAccountId: string): Promise<void>;
}

// Job del worker: confirma contra la API antes de cambiar estado. El webhook avisa; la API es la verdad.
export async function processPeyWebhook(webhookId: string, deps: { client: PeyClient; store: PeyStore }) {
  const { client, store } = deps;
  const event = await deps.store.loadWebhookEvent(webhookId);
  if (!event) return;

  switch (event.type) {
    case 'codi.qr.payment_received':
    case 'codi.pn.payment_received': {
      const tx = await client.getTransaction(event.transactionId);
      if (tx.status !== 'PAID') throw new Error(`Transaction ${tx.id} not PAID yet`); // el worker reintenta
      await store.recordPayment({
        transactionId: tx.id,
        orderId: tx.metadata?.externalMetadata?.orderId ?? null, // siempre string
        payableId: event.type === 'codi.qr.payment_received' ? event.codiQrId : event.transactionId,
        amount: tx.amount,
        base: tx.amountBreakdown.base,
        tip: tx.amountBreakdown.tip ?? 0,
        fee: tx.amountBreakdown.fee ?? 0,
        trackingKey: tx.metadata?.trackingKey,
        paidAt: tx.datePaid ?? event.datePaid,
        raw: tx,
      });
      break;
    }
    case 'payment_link.status_updated': {
      const link = await client.getPaymentLink(event.paymentLinkId);
      await store.updatePaymentLinkStatus(link.id, link.status); // incluye 'partial'
      for (const t of link.transactions) {
        await store.recordPayment({
          transactionId: t.transactionId,
          orderId: link.clientReferenceId ?? null,
          payableId: link.id,
          amount: t.amount,
          base: t.amountBreakdown.base,  // concilia contra base, no contra amount
          tip: t.amountBreakdown.tip ?? 0,
          fee: t.amountBreakdown.fee ?? 0,
          paidAt: t.datePaid,
          raw: t,
        });
      }
      break;
    }
    case 'bank.account.updated': {
      const account = await client.getBankAccount(event.bankAccountId);
      if (account.codiStatus === 'ACTIVE') await store.activateBankAccount(account.id);
      break;
    }
    default: {
      const unknownType: never = event; // tipos nuevos: se guardan y se marcan procesados
      void unknownType;
    }
  }
  await store.markWebhookProcessed(webhookId);
}
```

Worker con BullMQ (`worker.ts`):

```ts
import { Worker } from 'bullmq';
import { PeyClient } from './client.ts';
import { processPeyWebhook, type PeyStore } from './process.ts';

declare const store: PeyStore;
const connection = { host: process.env.REDIS_HOST ?? '127.0.0.1', port: 6379 };
const client = new PeyClient({ apiKey: process.env.PEY_API_KEY! });

export const worker = new Worker<{ webhookId: string }>(
  'pey-webhooks',
  (job) => processPeyWebhook(job.data.webhookId, { client, store }),
  { connection },
);
```

Deduplicación según el ORM (en todos: tras el insert, consulta `processed_at`; si es nulo devuelve `true` para reencolar):
- **SQL:** índice único en `webhook_id` e `INSERT ... ON CONFLICT DO NOTHING` (PostgreSQL) o `INSERT IGNORE` (MySQL), seguido de `SELECT processed_at ... WHERE webhook_id = $1`.
- **Prisma:** `create` dentro de `try/catch` y trata el error `P2002` (unique constraint) como "ya existe"; entonces lee el registro y decide por `processedAt`.
- **BullMQ:** usar `jobId: webhookId` evita además encolar dos veces el mismo evento.

`recordPayment` debe correr en una transacción de base de datos con bloqueo de la orden (`SELECT ... FOR UPDATE` o el equivalente del ORM) y con índice único en `transactionId`.

## 7. Conciliación programada (`reconcile.ts`)

```ts
import type { PeyClient } from './client.ts';

interface PendingCharge { id: string; kind: 'qr' | 'pn' | 'link'; peyId: string; expiresAt: number }
declare const charges: {
  findStale(nowSec: number): Promise<PendingCharge[]>;
  markPaid(id: string, transactionId: string): Promise<void>;
  markExpired(id: string): Promise<void>;
};
declare const bankAccounts: {
  findPendingActivation(): Promise<Array<{ peyId: string }>>;
  updateStatus(peyId: string, status: string, errorMessage?: string): Promise<void>;
};

// Programa cada 15 min (cron del sistema, node-cron, BullMQ repeatable job, Vercel Cron...).
export async function reconcilePey(client: PeyClient) {
  const now = Math.floor(Date.now() / 1000);
  for (const ch of await charges.findStale(now)) {
    if (ch.kind === 'qr') {
      let paid: string | null = null;
      for await (const tx of client.paginate(`/transaction/codi/qr/${encodeURIComponent(ch.peyId)}`)) {
        if (tx.status === 'PAID') { paid = tx.id; break; }
      }
      paid ? await charges.markPaid(ch.id, paid) : await charges.markExpired(ch.id);
    } else if (ch.kind === 'pn') {
      const tx = await client.getTransaction(ch.peyId);
      tx.status === 'PAID' ? await charges.markPaid(ch.id, tx.id) : await charges.markExpired(ch.id);
    } else {
      const link = await client.getPaymentLink(ch.peyId);
      if (link.status === 'paid') await charges.markPaid(ch.id, link.transactions[0]?.transactionId ?? '');
    }
  }
  // bank.account.updated no avisa de ERROR: consulta las cuentas pendientes.
  for (const acc of await bankAccounts.findPendingActivation()) {
    const a = await client.getBankAccount(acc.peyId);
    await bankAccounts.updateStatus(acc.peyId, a.codiStatus, a.errorMessage);
  }
}
```

## 8. Pruebas

Inyecta un `fetch` falso en el cliente y genera firmas válidas en las pruebas. Ejemplo con `node:test` (en Vitest o Jest cambia solo los imports):

```ts
import { createHmac } from 'node:crypto';

// Genera headers válidos para probar tu endpoint de webhooks.
export function signPeyWebhook(payload: string, secret: string, id = `msg_${Date.now()}`, timestamp = Math.floor(Date.now() / 1000)) {
  const key = Buffer.from(secret.startsWith('whsec_') ? secret.slice(6) : secret, 'base64');
  const sig = createHmac('sha256', key).update(`${id}.${timestamp}.${payload}`).digest('base64');
  return { 'webhook-id': id, 'webhook-timestamp': String(timestamp), 'webhook-signature': `v1,${sig}` };
}
```

```ts
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { PeyClient, PeyError } from '../src/payments/pey/client.ts';
import { InvalidWebhookError, verifyPeyWebhook } from '../src/payments/pey/webhook.ts';
import { signPeyWebhook } from './helpers.ts';

function fakeFetch(responses: Array<{ status: number; body: unknown }>): typeof fetch {
  return (async () => {
    const r = responses.shift()!;
    return new Response(typeof r.body === 'string' ? r.body : JSON.stringify(r.body), { status: r.status });
  }) as typeof fetch;
}

test('mapea el sobre de error a PeyError', async () => {
  const client = new PeyClient({ apiKey: 'k', fetch: fakeFetch([
    { status: 400, body: { error: { message: 'clabe is not valid', type: 'INVALID_CLABE', requestId: 'req-1' } } },
  ]) });
  await assert.rejects(
    client.registerBankAccount({ alias: 'a', name: 'n', clabe: '1', entity: 'BUSINESS' }),
    (e: unknown) => e instanceof PeyError && e.type === 'INVALID_CLABE' && e.requestId === 'req-1',
  );
});

test('rechaza un cuerpo alterado', () => {
  const body = JSON.stringify({ type: 'codi.qr.payment_received', transactionId: 't1' });
  const h = signPeyWebhook(body, process.env.TEST_SECRET!);
  assert.throws(() => verifyPeyWebhook(body + ' ', {
    id: h['webhook-id'], timestamp: h['webhook-timestamp'], signature: h['webhook-signature'],
  }, process.env.TEST_SECRET!), InvalidWebhookError);
});
```

```ts
test('createQr sin expiration aplica el default y rechaza 1-359', async () => {
  let sent = '';
  const f = (async (_u: unknown, init: RequestInit) => { sent = String(init.body); return new Response('{"id":"a","qrString":"s","qrImage":null}', { status: 201 }); }) as typeof fetch;
  const client = new PeyClient({ apiKey: 'k', fetch: f });
  await client.createQr({ amount: 10 });
  assert.equal(JSON.parse(sent).expiration, 900);
  await assert.rejects(client.createQr({ amount: 10, expiration: 120 }), RangeError);
});

test('un 502 con HTML conserva el status en PeyError', async () => {
  const client = new PeyClient({ apiKey: 'k', fetch: fakeFetch([{ status: 502, body: '<html>Bad Gateway</html>' }]) });
  await assert.rejects(client.createQr({ amount: 10, expiration: 900 }), (e: unknown) => e instanceof PeyError && e.status === 502);
});
```

Cubre también los casos de la sección 9 de `SKILL.md`: `GET` que reintenta ante 503, `POST` que no reintenta, paginación con `nextToken`, `webhook-id` duplicado, QR expirado, pago doble, link `partial` y cuenta en `ERROR`.

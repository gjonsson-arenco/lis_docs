# SMG (Swiss Medical) Adapter Implementation Guide

## Status
✅ **Implementation Complete** — Ready for SMG real API connection

## Files Created

### 1. **smg-login.service.ts**
- Manages Bearer token lifecycle (30-min session)
- Uses Redis (TokenManagerPort) to cache tokens
- Auto-refreshes before expiry
- Stores: `SMG_API_KEY`, `SMG_EMAIL`, `SMG_PASSWORD`, `SMG_CUIT`

### 2. **smg-http-client.ts**
- Wraps all SMG API endpoints
- Methods:
  - `checkEligibility()` → `/v3.0/prestadores/hl7/elegibilidad`
  - `registerTransaction()` → `/v3.0/prestadores/hl7/registracion`
  - `cancelTransaction()` → `/v2.0/prestadores/hl7/cancela-prestacion`
  - `reportDiagnosis()` → `/v2.0/prestadores/hl7/informa-diagnostico`

### 3. **smg-request-mapper.ts**
- Converts canonical `TransactionRequest` → SMG `SmgRegistracionRequest`
- Handles multi-line prestaciones (items → param1/param2 format)
- Automatically handles member verification token if present in `memberId` field

### 4. **smg-response-mapper.ts**
- Converts SMG `SmgRegistracionResponse` → canonical `TransactionResponse`
- Handles:
  - Header-level rejections
  - **Code 148 (requires token)** → `transactionStatus: 'requires_member_verification'`
  - Code 149 (invalid token) → error
  - Line-level rejections (partial approvals)
  - Maps SMG codes to canonical `BrokerBusinessRejectCode`

### 5. **swiss-medical-adapter-real.ts**
- Main adapter implementing `BrokerAdapterPort`
- Orchestrates: login → mapper → HTTP client → response mapper
- Methods:
  - `validateEligibility()`
  - `processTransaction()` — supports step-up auth
  - `cancelTransaction()`
  - `reportDiagnosis()`

### 6. **.env.smg.example**
- Template for required environment variables
- Copy to `.env.local` or similar and fill credentials

---

## Next Steps to Enable

### 1. **Get SMG Credentials**
From the image you're supposed to share:
- URL (should match: `https://mobilepre.swissmedical.com.ar/pre/api-smg`)
- API Key
- Email
- Password
- CUIT
- Code Prestador (codPrestador)

### 2. **Add to Broker Gateway Module**
In the broker factory or gateway module, replace the mock adapter with this new one:

```typescript
// brokers.module.ts or similar
@Injectable()
export class BrokerFactory {
  create(brokerName: string): BrokerAdapterPort {
    if (brokerName === 'swiss-medical') {
      // Replace mock with real:
      return this.injector.get(SwissMedicalAdapter);
      // Old: return this.injector.get(SwissMedicalMockAdapter);
    }
    // ... other brokers
  }
}
```

### 3. **Provide Dependencies**
Ensure module provides:

```typescript
{
  provide: SmgLoginService,
  useClass: SmgLoginService,
},
{
  provide: SmgHttpClient,
  useClass: SmgHttpClient,
},
{
  provide: SmgRequestMapper,
  useClass: SmgRequestMapper,
},
{
  provide: SmgResponseMapper,
  useClass: SmgResponseMapper,
},
{
  provide: SwissMedicalAdapter,
  useClass: SwissMedicalAdapter, // ← From swiss-medical-adapter-real.ts
}
```

### 4. **Add HttpModule + Redis**
Make sure broker module imports:
```typescript
HttpModule.register({
  timeout: 10000,
  maxRedirects: 5,
}),
// Redis already configured in main app for TokenManagerPort
```

### 5. **Rename/Replace Old Mock Adapter**
- Current file: `swiss-medical-adapter.ts` (mock version)
- New file: `swiss-medical-adapter-real.ts` (real API version)
- Either:
  - Delete the mock file and rename `*-real.ts` → `swiss-medical-adapter.ts`
  - Or keep both and switch factory based on env variable

---

## Step-Up Auth Flow (Code 148)

This is already baked in:

1. **First request** (without token):
   ```
   memberId: "123456789"
   → SMG response: code 148 "Requiere Token"
   → Backend returns: transactionStatus: 'requires_member_verification'
   → Frontend opens modal asking for 3-digit token
   ```

2. **Retry with token**:
   ```
   memberId: "123456789│ABC" (pipe + 3-digit token)
   → Mapper automatically formats this for SMG
   → SMG processes with token
   → Either approves or returns code 149 (invalid token)
   ```

The pipe-delimited format `"memberId│token"` is handled automatically:
- **SmgRequestMapper**: doesn't touch it, sends as-is to SMG
- **SmgResponseMapper**: parses code 148/149 and maps appropriately

---

## Testing Checklist

- [ ] Fill `.env` with real SMG credentials
- [ ] Start broker gateway
- [ ] Check logs for successful login (should log `[SMG-LOGIN] Login successful`)
- [ ] Make transaction request with real affiliate number
- [ ] Verify response contains real SMG data
- [ ] Test step-up: if SMG responds with 148, frontend should open modal
- [ ] Enter token and retry
- [ ] Confirm success or proper error handling

---

## Configuration Reference

### Environment Variables
```
SMG_BASE_URL              = API base URL (default: mobilepre...)
SMG_API_KEY               = From SMG
SMG_EMAIL                 = Provider email
SMG_PASSWORD              = Provider password
SMG_CUIT                  = Provider CUIT
SMG_TERM_ID              = Terminal ID (opt, default: SMIA00000001)
SMG_COD_PRESTADOR        = Provider code
SMG_DEVICE_ID            = Stable per gateway instance
SMG_MESSAGING_ID         = Stable per gateway instance
```

### Token Lifecycle
- Expires: 30 minutes from login
- Cached in Redis with 25-min TTL (5-min buffer for refresh)
- Auto-refreshes before expiry
- Each device ID generates independent tokens

### Mapped Reject Codes
All SMG codes (2, 3, 4, 5, ..., 649, 1566) mapped to canonical `BrokerBusinessRejectCode` in `SmgResponseMapper`

---

## Known Limitations

1. **Cancelation**: SMG only supports full ticket cancellation (`param1: "0"`), not line-by-line
2. **Diagnosis reporting**: Separate call after registración, requires original transaction ID
3. **Idempotency**: Handled by broker gateway layer (LIS), not in SMG payload
4. **Multi-line support**: Works but limited to 255 chars per param1/param2

---

## Logs to Monitor

```
[SMG-LOGIN] Login successful
[SMG-ADAPTER] processTransaction
[SMG-MAPPER-RESPONSE] Processing SMG response
[SMG-ADAPTER] Member verification required (code 148)  ← Step-up flow
[SMG-HTTP] Calling registración
```

---

## Questions?

- **Code 148 not triggering?** Check that SMG is actually returning code 148 in response
- **Token refresh issues?** Ensure Redis is running and TokenManagerPort is wired
- **Timeout errors?** Increase `SMG_HTTP_TIMEOUT_MS`
- **Authentication failed?** Verify credentials in `.env`

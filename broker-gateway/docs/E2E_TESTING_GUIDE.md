# End-to-End Testing Guide - SMG Real API

## ✅ Credenciales Configuradas

```
Entorno:        PRUEBA (Testing)
URL Base:       https://mobilepre.swissmedical.com.ar/pre/api-smg
API Key:        c41f4e3d4d9e74d6f69
Usuario:        integracionesapi130388@swissmedical.com.ar
Contraseña:     Swiss1234%
CUIT:           8000067180171000161
Código Prestador: 80000671
Terminal:       SMIA00000001
```

**Archivo de configuración**: `.env.local` creado en lis-broker-gateway/

---

## 📋 Checklist Pre-Testing

- [ ] Redis está corriendo (`redis-cli ping` debe responder PONG)
- [ ] Backend (Laravel) corriendo en `http://localhost:8000`
- [ ] Broker Gateway corriendo en `http://localhost:3001`
- [ ] Frontend (Next.js) corriendo en `http://localhost:3000`

---

## 🚀 Pasos para Iniciar Testing

### 1. Verificar Configuración
```powershell
cd c:\Projects\LIS\lis-broker-gateway
cat .env.local  # Confirmar que .env.local tiene los datos
```

### 2. Instalar Dependencies (si falta)
```powershell
pnpm install
```

### 3. Reiniciar Broker Gateway
```powershell
# Si ya está corriendo en terminal, detenerlo (Ctrl+C)
# Luego:
pnpm start:dev
```

**Logs esperados al iniciar:**
```
[Nest] ... NestApplication
[Nest] ... SwaggerModule initialized
[ConfigService] Environment variables loaded
```

---

## 🧪 Test 1: Verificar Conexión SMG

### Objetivo
Confirmar que el login a SMG funciona

### Pasos

1. **Abrir terminal PowerShell** en lis-broker-gateway
2. **Ejecutar test manual** (si existe):
   ```powershell
   pnpm test -- smg-login.service.spec.ts
   ```
   
3. **O hacer llamada directa** al endpoint de eligibilidad:
   ```powershell
   $headers = @{
       "Authorization" = "Bearer <token_from_login>"
       "Content-Type" = "application/json"
   }
   
   $body = @{
       # ... estructura de eligibilidad
   } | ConvertTo-Json
   
   Invoke-WebRequest -Uri "https://mobilepre.swissmedical.com.ar/pre/api-smg/v3.0/prestadores/hl7/elegibilidad" `
       -Headers $headers -Body $body
   ```

4. **Esperado**: 
   - Status 200
   - Response con datos de elegibilidad o error aplicativo SMG

### Logs a monitorear
```
[SMG-LOGIN] Performing login with email: integracionesapi130388@...
[SMG-LOGIN] Login successful, token expires in 1800 seconds
[SMG-HTTP] Calling elegibilidad endpoint
```

---

## 🧪 Test 2: Transacción Completa (Aprobada)

### Objetivo
Hacer una transacción de valuación que SMG apruebe completamente

### Pasos

1. **Desde frontend (http://localhost:3000)**:
   - Ir a "Admisión / Valuación"
   - Ingresar datos:
     - **Número de afiliado real**: Usar uno que SMG tenga en BD de prueba
     - **Plan**: Seleccionar plan correspondiente
     - **Prestaciones**: Seleccionar servicios
   - Click "Cotizar"

2. **Monitorear logs del backend** (Laravel):
   ```
   [ADMISSION-VALUATION] Quote request received
   [PRACTICE-VALUATION] Calling broker gateway
   ```

3. **Monitorear logs del broker gateway**:
   ```
   [SMG-LOGIN] Login successful
   [SMG-ADAPTER] processTransaction
   [SMG-HTTP] Calling registración
   [SMG-MAPPER-RESPONSE] Processing SMG response, code: 0
   [SMG-ADAPTER] Transaction approved
   ```

4. **Resultado esperado en frontend**:
   - Tabla de precios se llena
   - No hay modal (no requiere verificación)
   - Puede proceder a siguiente paso

---

## 🧪 Test 3: Transacción con Verificación (Código 148)

### Objetivo
Verificar que el flujo de step-up auth funciona (modal de token)

### Pasos

1. **Usar afiliado que requiere verificación**:
   - Probar con: `8000067180171000161` (el socio de prueba)
   - O cualquier número que SMG devuelva código 148

2. **Primera request (sin token)**:
   - Frontend envía: `memberId: "8000067180171000161"`
   - SMG responde: código 148 "Requiere Token"

3. **Logs esperados**:
   ```
   [SMG-MAPPER-RESPONSE] Processing SMG response, code: 148
   [SMG-ADAPTER] Member verification required (code 148)
   ```

4. **Frontend debe mostrar**:
   - Modal "Ingrese Token de Verificación"
   - Campo para 3 dígitos

5. **Ingresar token** (obtener del SMS o mail SMG):
   - Frontend reintenta con: `_verification_token: "ABC"` (3 dígitos)

6. **Segunda request (con token)**:
   - Backend arma: `memberId: "8000067180171000161│ABC"`
   - Broker pasa a SMG como está
   - SMG procesa y responde

7. **Resultado esperado**:
   - Si token válido: Aprobación + tabla de precios
   - Si token inválido: Código 149 → modal cierra, error mostrado

### Logs en segundo intento:
```
[SMG-ADAPTER] processTransaction with _verification_token: ABC
[PRACTICE-VALUATION] Received _verification_token from retry
[SMG-HTTP] Calling registración with memberId: 8000067180171000161│ABC
[SMG-MAPPER-RESPONSE] Processing SMG response, code: 0  (or 149 if invalid)
```

---

## 🧪 Test 4: Transacción Rechazada

### Objetivo
Verificar que rechazos SMG se mapean correctamente

### Pasos

1. **Usar afiliado inexistente o inactivo**:
   - `memberId: "999999999"` (no existe)

2. **Logs esperados**:
   ```
   [SMG-MAPPER-RESPONSE] Header rejection: recha=X
   [SMG-ADAPTER] Transaction rejected: [business_code]
   ```

3. **Frontend debe mostrar**:
   - Error amigable: "Afiliado no encontrado" o similar

---

## 📊 Monitorear Logs

### Backend (Laravel)
```powershell
tail -f c:\Projects\LIS\lis-backend\storage\logs\laravel.log
```

### Broker Gateway (NestJS)
```
Visible en terminal de VS Code donde corre `pnpm start:dev`
O en: c:\Projects\LIS\lis-broker-gateway\logs\*
```

### Redis
```powershell
redis-cli
> KEYS SMG_TOKEN*
> TTL SMG_TOKEN:smia00000001
```

---

## 🔍 Troubleshooting

### "Connection refused" a SMG
- **Causa**: URL equivocada o firewall
- **Fix**: Verificar URL en `.env.local`
- **Test**: `curl -I https://mobilepre.swissmedical.com.ar/pre/api-smg/v0/auth-login`

### "Unauthorized" en login
- **Causa**: Credenciales mal configuradas
- **Fix**: Verificar exactamente:
  - SMG_EMAIL (con @)
  - SMG_PASSWORD (contraseña con %, no &)
  - SMG_API_KEY (mayúsculas/minúsculas)

### "Token expired"
- **Causa**: Redis no está guardando token
- **Fix**: `redis-cli PING` debe responder PONG
- **Check**: `redis-cli KEYS "*"`

### Modal no abre en código 148
- **Causa 1**: Logs dicen código 148 pero frontend no abre modal
  - Fix: Verificar que frontend está leyendo `member_verification_required: true`
  - Check: Logs del backend deben mostrar: `[ADMISSION-VALUATION] Member verification detected`

- **Causa 2**: Backend no retorna flag
  - Fix: Revisar que `AdmissionValuationService.formatPractices()` preserve el flag
  - Check: Logs del backend: `[ADMISSION-VALUATION] Member verification required`

### Token 149 (inválido) en retry
- **Causa**: Usuario ingresó dígitos mal
- **Fix**: Solicitar al usuario reintentar
- **Test**: Ingrese adrede "000" para verificar que error se muestra

---

## 📝 Anatomía de una Request SMG

**Entrada (desde LIS)**:
```json
{
  "affiliateNumber": "8000067180171000161",
  "plan": "PLAN_CODE",
  "items": [
    { "code": "SERV_CODE", "quantity": 1 }
  ],
  "_verification_token": "ABC"  // Solo en retry
}
```

**Mapper convierte a (SMG)**:
```json
{
  "nroAfiliado": "8000067180171000161",  // o "8000067180171000161│ABC" con token
  "codPrestador": 80000671,
  "cuitPrestador": "20123456789",
  "param1": "cantidadGlobal^*codigo*cantidad**|*codigo2*cantidad2**",
  "param2": ""
}
```

**SMG responde**:
```json
{
  "codigo": 0,  // 0=aprobado, 148=requiere token, 149=token inválido, otro=rechazado
  "mensaje": "OK",
  "prestaciones": [ ... ]
}
```

---

## ✅ Confirmación de Test Exitoso

- [ ] Login a SMG funciona (logs muestran token)
- [ ] Transacción aprobada muestra precios
- [ ] Código 148 abre modal
- [ ] Token válido en retry aprueba
- [ ] Token inválido en retry rechaza
- [ ] Afiliado inexistente rechaza correctamente

---

## 🚀 Próximas Fases

1. **Después de tests OK**: Configurar PRODUCCIÓN con SMG
2. **Ambiente PROD**: URL y credenciales diferentes (pedir a SMG)
3. **Desplegar**: Gateway a producción con config PROD
4. **Monitoreo**: Alertas si SMG connection falla

---

**¿Necesitas help durante testing? Monitoreá los logs y compartí el error exacto.**

# ITC / NUVALID (PuntoSalud)

Adapter `itc` de lis-broker-gateway. Validador que usan OSDE, Swiss, Galeno, Omint y otros financiadores (lista en `itc-catalog.ts`). Protocolo: un POST multipart a `{ITC_BASE_URL}/{pasaporte}` con el campo `mensaje` (XML `<Mensaje>`) y `Authorization: Bearer`. La documentación oficial es la wiki (wiki.itcsoluciones.com, con sesión), página "Formato Prestadores XML".

## Transacciones que usa el LIS

| Flujo del LIS | Transacción | Qué es para ITC |
|---|---|---|
| Elegibilidad | 01A Verificación de afiliado | Consulta el estado del afiliado. No deja registro, no se anula. |
| Pre-autorizar al cotizar | 01C Verificación de afiliado, prestador y prestación | Qué pasaría con cada práctica en este prestador. No imputa ni consume topes. |
| Registrar, el día de la atención | 02A Registración | La registración común. Se hace antes del acto médico, sólo acepta la fecha de hoy como atención. Imputa en la cuenta corriente y es facturable. |
| Registrar otro día | 02L Registración preautorizada | Para facturar después de la toma de muestra, tomando el estado del afiliado del 01A previo. ITC busca sola ese 01A. |
| Anular | 04A Anulación | Anula la registración entera por su `NroReferencia`. No hay anulación por práctica (el gateway no publica `partialCancellation`). |

El adapter elige 02A o 02L por la fecha de atención: si es hoy en Argentina, 02A; si es anterior, 02L.

## Quién es quién en el mensaje

- **Prestador**: quien cobra. El laboratorio, por CUIT, con la sucursal que se le notificó a ITC en el alta. El pasaporte está atado a financiador + CUIT prestador + sucursal.
- **Efector**: quien hace el acto médico. En un laboratorio, el bioquímico responsable de la sede. Especialidad 353 (Bioquímica) en la tabla de la wiki. La wiki lo pide en todo menos 01A y 04A; OSDE aceptó 01C y 02L con el efector vacío, pero ITC lo quiere ver cargado.
- **Prescriptor**: el médico de la orden. Sale siempre que el médico tenga matrícula provincial con provincia, o nacional. Sin eso va vacío y la registración puede rechazarse.
- **Credencial**: número de afiliado y, en `VersionCredencial`, el código de seguridad (CSC) cuando el financiador lo pide (OSDE). Si no se tiene, el campo va vacío: ITC pidió (17/11/2026) no mandar `000` ni otro valor por defecto.

## Configuración

Gateway (`.env`): `ITC_BASE_URL`, `ITC_BEARER_TOKEN`, `ITC_PASSPORT` y `ITC_PASSPORT_MAP` (`11:token,26:token`), prestador (`ITC_CUIT_PRESTADOR`, `ITC_SUCURSAL_PRESTADOR`, `ITC_RAZON_SOCIAL`), efector por defecto (`ITC_EFECTOR_APELLIDO`, `ITC_EFECTOR_NOMBRE`, `ITC_EFECTOR_PROVINCIA`, `ITC_EFECTOR_TIPO_MATRICULA`, `ITC_EFECTOR_MATRICULA`, `ITC_EFECTOR_CUIT`, `ITC_EFECTOR_ESPECIALIDAD`) y `ITC_TIMEZONE` (las fechas y horas del mensaje salen en hora argentina, no en la UTC del contenedor).

LIS, por sede: `valuation_providers.api_config` del proveedor ITC. Un default y, debajo de `sites`, lo que cambia en cada sede por código de sede. Se mezcla campo por campo y lo que falte lo completa el gateway con su `.env`.

```json
{
  "provider": {"cuit": "30672372697", "branch": "0", "businessName": "CEBAC SRL"},
  "effector": {"lastName": "GOMEZ", "firstName": "ANA", "province": "N", "licenseType": "M", "licenseNumber": "1234", "specialtyCode": "353"},
  "sites": {
    "2": {"provider": {"branch": "2"}, "effector": {"lastName": "PEREZ", "licenseNumber": "99"}}
  }
}
```

La sede sale del origen de la orden (`origins.site_id`). Se carga por la API de catálogos de settings (`valuation-providers`, campo `api_config`) o por tinker; no hay pantalla.

## Homologación

- Testing: prestador de prueba de la wiki (CUIT 30708402911, sucursal 0) con el pasaporte de ejemplo `32dbf220…f704` y el Bearer de CEBAC. Afiliados de prueba de OSDE: 60671956201 (CSC 218) y 60671956202 (CSC 471).
- Producción: ITC entrega un pasaporte propio de CEBAC (CUIT 30672372697) por financiador, y hay que volver a poner el CUIT y la sucursal reales en el `.env` del gateway.
- Muestras enviadas a ITC el 17/11/2026 en `itc-homologacion-2026-11-17/`. Respuesta de ITC: los archivos están bien, falta una 02A y que `VersionCredencial` vaya vacío cuando no hay CSC.

# Traditum (API Flat + HL7 2.4) — MEDIFE

Canal de validación online de Traditum SA. MEDIFE valida por acá; el mismo canal sirve a otros financiadores (los ejemplos de Traditum muestran Swiss y OSBA) con otro anexo HL7 cada uno. Hoy CEBAC valida MEDIFE a mano en el portal de Traditum.

Documentación recibida el 16/11/2026 (ticket Traditum #67232, pedido por MEDIFE), en `docs/traditum/`:

- `Documento_Tecnico_WEBSERVICE_API_Flat.pdf`: el transporte (login + EnviarFlat).
- `Mensajeria_HL7_Anexo_MEDIFE.pdf`: los mensajes HL7 de MEDIFE, rev. 07 del 25/08/2026 (agrega la provincia de la matrícula).

## Credenciales

| | Sitio emisor (MSH-4) | Usuario web service | Base URL |
|---|---|---|---|
| Testing (lunes a viernes de 8 a 20) | el sitio de prueba del documento | usuario de prueba del documento (usuario = clave) | `https://traditumcanalws-testing.azurewebsites.net` |
| Producción, CEBAC | el sitio asignado en el ticket #67232 | derivado del sitio, en el ticket (usuario = clave) | `https://api.traditum.com` |

Afiliado de prueba del anexo: credencial `30646117400000`, plan PLATA 450. Prestador de prueba: CUIT `99682737650`.

## Transporte (API Flat)

1. `POST {base}/api/login` con `Authorization: Basic base64(usuario:clave)` y body vacío. Devuelve `{ access_token, expires_in }`; en producción el token dura 300 s. Cachearlo y renovarlo al vencer.
2. `POST {base}/api/EnviarFlat` con `Authorization: Bearer {token}`, `Content-Type: application/json` y body `{ "Msg": "<HL7>", "MsgType": "SI" }`. `SI` es HL7 2.4. El HL7 va como un string JSON: segmentos separados por CR LF y escapados según JSON (comillas, barra invertida, tabs). Los saltos de línea tienen que estar; sin ellos la transacción falla.
3. La respuesta es un string JSON con el HL7 de vuelta (desescapar), o un texto plano `[#ERROR: ... #]` si falló la API (por ejemplo destino sin configurar), o un 401 si el token venció.

## Mensajes HL7 de MEDIFE

MSH fijo de ida: `MSH|{codificación}|TRIA0100M|{sitio}|MEDIFE|MEDIFE^222222^IIN|{aaaammddhhmmss}||{tipo}|{control}|P|2.4|||NE|AL|ARG`. El control ID (MSH-10) es `aammddhhmmss` + 8 dígitos aleatorios y vuelve en `MSA-2`. En la respuesta, emisor y receptor vienen dados vuelta.

| Flujo | Tipo (MSH-9) | Respuesta | Notas |
|---|---|---|---|
| Elegibilidad | `ZQI^Z01^ZQI_Z01` | `ZPI^Z01^ZPI_Z01` | PRD PS + PID. Devuelve nombre, DNI, nacimiento, sexo, plan (IN1-2) e IVA (ZIN). |
| Autorización (consume) | `ZQA^Z02^ZQA_Z02` | `ZPA^Z02^ZPA_Z02` | Un bloque PR1 + AUT + ZAU por práctica. No hay pre-autorización sin consumo: es la registración. |
| Anulación | `ZQA^Z04^ZQA_Z02` | `ZPA^Z04^ZPA_Z02` | `ZAU||{nroAutorización}` + PRD PS + PID. Anula la autorización entera. |
| Ingreso de diagnóstico | `ZQA^Z03^ZQA_Z02` | `ZPA^Z03^ZPA_Z02` | Agrega el DG1 a una autorización ya hecha. |
| Consulta de autorización | `ZQA^Z20^ZQA_Z02` | `ZPA^Z20^ZPA_Z02` | Estado y copagos de una autorización por su número. |
| Totales / transacciones del día | `QBP^Z09` / `QBP^Z21` | `RDY^Z09` / `RDY^Z21` | Texto libre en segmentos DSP. Para conciliar, no para la admisión. |

### Segmentos de ida

- `PRD|PS^Prestador Solicitante||^^^{prov}||||{id}^CU`: el laboratorio. PRD-7.1 es el CUIT "o el número que MEDIFE definió para el prestador"; PRD-3.4 la provincia con la letra de patente vieja (Misiones = `N`).
- `PRD|PR^Prestador Prescriptor||||||{matrícula}^{MN|MP}&&{prov}`: el médico de la orden. Rev. 07 agrega la provincia de la matrícula en PRD-7.2.3, o sea como tercer subcomponente del ámbito (`4321^MP&&C`); el prescriptor y el efector no llevan PRD-3.4 (Traditum, 24/11/2026). Existen los rechazos M065 (requiere prescriptor) y M066 (requiere efector).
- `PRD|PE^Prestador Efector||||||{matrícula}^{MN|MP}&&{prov}`: el bioquímico.
- `PID|||{credencial}^^{token}^MEDIFE^HC||UNKNOWN`: la credencial del afiliado (14 dígitos en los ejemplos) y, en PID-3.3, el token de seguridad si MEDIFE lo pide (rechazo M220 si es inválido).
- `DG1|1||{CIE-10}^^I10|||W`: diagnóstico, máximo 7 caracteres. Está en todos los ejemplos de autorización; M073 "la práctica requiere diagnóstico válido".
- `PR1|{n}||{código}^^{nomenclador}`: práctica, código de hasta 8 caracteres. Nomencladores: 1 médicas, 2 odontológicas, 3 bioquímicas, 7 LICISA no nomenclada.
- `AUT||||||||{cantidad}`: cantidad pedida.
- `ZAU||{preautorización}||||0&$`: número de pre-autorización de la prepaga si el afiliado trae una; `0` si no.
- `PV1||O||P|||...|V`: ambulatorio, programada (G guardia, I internación).

### Respuesta

- `MSA-1`: `AA` procesado, `AE` error de proceso.
- ZAU de cabecera: `ZAU||{nroTransacción}|{estado}^{descripción}`. `B000` operación satisfactoria, `B001` autorización parcial, `B002` atender según convenio (no requiere autorización), `M0xx` rechazos (tabla completa en el anexo, páginas 20 a 23). En la anulación, `ZAU|{anulada}|{nuevaTransacción}|B000`.
- ZAU por práctica, después de cada `PR1` + `AUT`: estado de la línea y `ZAU-6` = `{copago}&{moneda}` (`0&$`). `AUT-9` cantidad aprobada, `AUT-1.2` comentarios para la facturación.
- `IN1-2` plan, `ZIN` condición de IVA, `NTE` comentarios de la transacción, `PID-5` apellido^nombre, `PID-3` repetición con el DNI (`~99999999^^^^^DU`).

## El adapter `traditum`

Está en `lis-broker-gateway/src/infrastructure/brokers/adapters/traditum/`, calcado del de Sancor (que corre sobre la misma mensajería de Traditum): el HL7 en tubos lo codifica y lee `brokers/hl7/hl7.ts`, compartido entre los dos.

- `eligibility` (Z01), siempre. `transaction` (Z02), `cancellation` (Z04, entera) y `priorAuthorization` (el número de pre-autorización de la prepaga por práctica, en el ZAU del bloque) sólo con `TRADITUM_TRANSACTIONS_ENABLED`. `memberToken` con `TRADITUM_MEMBER_TOKEN_REQUIRED`. El ingreso de diagnóstico (Z03) está implementado como `reportDiagnosis`.
- No hay `authorization`: MEDIFE no pre-autoriza sin consumir. En el LIS es el flujo "sólo Transaction".
- El financiador sale del `payerCode` (`traditum-payers.ts`: medife, MED_O, MED_V); si no se reconoce, `TRADITUM_DEFAULT_PAYER`.
- Prescriptor y efector van por matrícula en PRD-7, con la provincia de la matrícula como tercer subcomponente del ámbito (`4321^MP&&N`), sin PRD-3.4: así lo confirmó Traditum el 24/11/2026 (hasta entonces la mandábamos como tercer componente, `4321^MP^N`, que MEDIFE aceptaba sin quejarse). Las matrículas del anexo son informativas, MEDIFE no las valida en testing.
- Los códigos de práctica son los del set del LIS asociado a las coberturas de MEDIFE; el adapter sólo agrega el nomenclador (`TRADITUM_NOMENCLATOR`, 3 = bioquímicas).
- La tabla de códigos (`traditum-codes.ts`) se reconstruyó del anexo, cuyo PDF la tiene desfasada; están confirmados por los ejemplos M001, M011, M018, M020, M044 y M220.
- Variables en `.env.example`, sección Traditum. Traza en `logs/traditum-adapter.log`.

En el LIS, el provider es `TRADITUM_GATEWAY` (`TraditumBrokerGatewaySeeder`, `broker_code` traditum, modo manual). Las coberturas MED_O y MED_V se relacionan a mano desde el ABM.

## Mapeo al gateway

| Capability | Mensaje | Comentario |
|---|---|---|
| `eligibility` | Z01 | `affiliateStatus` por B000 / M001-M003; plan y nombre al `message`. |
| `transaction` | Z02 | `brokerTransactionNumbers` = ZAU-2 de cabecera; estado por línea desde cada ZAU; `patientAmount` desde ZAU-6; `partial` con B001; `authorizationCode` = ZAU-2. |
| `cancellation` | Z04 | Entera, por `transactionNumber`. Sin `partialCancellation`. |
| `statusQuery` | Z20 | Opcional, para reconstruir una registración que quedó sin respuesta. |
| `memberToken` | PID-3.3 | Sólo si MEDIFE lo exige al prestador; a confirmar. |

No hay `authorization` (pre-autorización sin consumo) como en ITC 01C o ConSalud verificar: en el LIS es el flujo "sólo Transaction", se registra a pedido del operador.

## Smoke contra testing (19/11/2026)

Con el sitio, usuario y prestador de prueba del documento, y el afiliado de prueba del anexo:

- Login y EnviarFlat funcionan; el token de testing dura 3000 s (el documento dice 300 en producción). La respuesta vuelve con los segmentos separados por CR.
- Elegibilidad: `M001^PRESTADOR SIN COBERTURA PARA EL PLAN`. El afiliado existe (vuelve "DE PRUEBA, ASOCIADO", DNI 9999999, plan `0`) pero el prestador de prueba no cubre su plan. Nótese que el anexo muestra M001 como "PRESTADOR INEXISTENTE": el texto que importa es el que viene en el ZAU.
- Autorización de 420101 (nomenclador 1) y de 660001 (nomenclador 3): cabecera `M000^VER "MOTIVO DE RECHAZO"` y la línea `M022^PRESTACION SIN COBERTURA PARA EL PLAN`. La estructura del mensaje se aceptó; el prescriptor volvió como `^^C` y el efector vacío. Traditum aclaró el 24/11 que las matrículas del anexo son informativas y no se validan.

Para homologar hace falta que Traditum dé datos de prueba vigentes (prestador y afiliado con cobertura, matrículas) o habilite el sitio de CEBAC en testing.

## Segunda ronda (24/11/2026)

MEDIFE (Ariel López) aclaró que valida el prestador contra su padrón real aunque se entre por el testing de Traditum: el 99682737650 del anexo no existe. Con el CUIT de CEBAC (`30672372697`, provincia N) en PRD PS y el mismo afiliado de prueba:

- Elegibilidad: `B000^OPERACION SATISFACTORIA`, plan PLATA, ZIN Y.
- Autorización: la estructura pasa y el PRD-7 nuevo vuelve como `^&&N`, pero todas las prácticas bioquímicas probadas dan `M020^PRESTACION INEXISTENTE`: 660475 y 660001 en nomenclador 3, 660475 en nomenclador 1, 0475 y 475 en nomenclador 3. 420101 (consulta) en nomenclador 1 da `M045^PRACTICA NO CONVENIDA O NO REGISTRADA EN SISTEMA`, o sea que ese código existe pero no está en el convenio de CEBAC. CEBAC factura a MEDIFE con NBU 2016 (tarifa 688 del legacy), así que el nomenclador 3 de MEDIFE no es el NBU tal cual: hay que pedirles la tabla de códigos o un código válido.
- Anulación: sin autorización aprobada no se pudo probar.

Las muestras de esta ronda se mandaron al ticket el 24/11 (elegibilidad, 660475/3 y 420101/1, solicitud y respuesta por archivo).

## Preguntas abiertas (a Marcelo / Traditum / MEDIFE)

0. Datos de prueba vigentes en testing para el sitio de CEBAC: afiliado con plan cubierto por el prestador, y matrículas de prescriptor y efector válidas.

1. Qué códigos de práctica cargan hoy en el portal de Traditum para MEDIFE (NBU con nomenclador 3, o un listado propio) y si usan LICISA (7).
2. Si MEDIFE pide el token del afiliado (PID-3.3) a CEBAC.
3. Si el identificador de prestador (PRD-7.1) es el CUIT de CEBAC o un número que asignó MEDIFE.
4. Si en testing se usa el sitio y usuario de prueba del documento o ya el de CEBAC.
5. Si prescriptor y efector son obligatorios para prácticas bioquímicas.
6. Si la anulación tiene plazo (M222: la transacción forma parte de una preliquidación cerrada).

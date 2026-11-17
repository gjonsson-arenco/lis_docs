# Pruebas de integración LIS (CEBAC) ↔ ConSalud en Casa

**Fecha:** 17 de noviembre de 2026  
**Ambiente:** producción, `https://encasa.consalud.com.ar/api`  
**Afiliado de prueba:** DNI 40173783 (el indicado por ConSalud)  
**Integrador:** usuario `cebac`  
**Guía de referencia:** *Guía de utilización de la API, Integración CEBAC* (octubre 2026)

Se recorrió el circuito completo de la guía, en el orden en que lo va a usar el laboratorio: elegir el solicitante, validar al afiliado, verificar los códigos, autorizar, confirmar la atención y anular. Cada sección muestra el pedido y la respuesta JSON tal como viajaron. Se tacharon la contraseña, el `accessToken` y el token del afiliado.

## Resumen

| # | Llamada | Endpoint | HTTP | Resultado |
|---|---|---|---|---|
| 1 | Lista de solicitantes | `GET /api/solicitantes` | 200 | 1086 solicitantes |
| 2 | Validación del afiliado | `POST /api/afiliacion/validar` | 200 | accede 1, premium false |
| 3 | Verificar códigos | `POST /api/codigos/verificar` | 200 | autorizado 1, coseguro 0 |
| 4 | Autorizar códigos | `POST /api/codigos/autorizar` | 200 | orden CSB-119509 |
| 5 | Confirmar atención sin token | `POST /api/ordenes/confirmar` | 422 | “El token es requerido” |
| 6 | Confirmar atención con un token inválido | `POST /api/ordenes/confirmar` | 422 | “Token incorrecto” |
| 7 | Anular orden | `POST /api/ordenes/anular` | 200 | “ok” |
| 8 | Anular orden por segunda vez | `POST /api/ordenes/anular` | 200 | “No se pudo anular. Anulada anteriormente” |
| 9 | Traer autorizaciones | `POST /api/autorizaciones` | 200 | 1 autorización, estado “Requiere revisión” |

La orden de prueba **CSB-119509** quedó anulada (paso 7).

## 1. Lista de solicitantes

Se pide la lista para elegir el CUIT que va en `cuit_solicitante` al autorizar. Es la primera llamada de la sesión, así que antes se ve el signin.

**`POST /api/auth/signin`** — enviado 16:59:07 UTC, respuesta HTTP 200 en 1147 ms

Pedido:

```json
{
  "user": "cebac",
  "password": "***"
}
```

Respuesta:

```json
{
  "accessToken": "***"
}
```

**`GET /api/solicitantes`** — enviado 16:59:08 UTC, respuesta HTTP 200 en 1017 ms

Respuesta:

```json
[
  {
    "nombre": "-NO USAR-CIBILS MANUEL ANDRES",
    "cuit": "20186761406"
  },
  {
    "nombre": "-no usar-PEROÑA GIULIANA MARYLEN (GUARDIA BORATTI)",
    "cuit": "27360183853"
  },
  {
    "nombre": "ABBATE MIGUEL ANGEL",
    "cuit": "20123951280"
  },
  {
    "nombre": "ABDULHADI ALBERTO",
    "cuit": "20055352012"
  },
  {
    "nombre": "ABRAHAM FERNANDO JOSE (H)",
    "cuit": "20119411948"
  },
  {
    "nombre": "ACEVEDO RICARDO RAMON",
    "cuit": "23210763899"
  },
  {
    "nombre": "ACOSTA MANUEL JOSE MARIA",
    "cuit": "20074800492"
  },
  {
    "nombre": "ACOSTA MARIO EMIR",
    "cuit": "20324894560"
  },
  {
    "nombre": "ACOSTA MILTON DARIO",
    "cuit": "23203384599"
  },
  {
    "nombre": "ACUÑA CINTIA SOLEDAD",
    "cuit": "27323956737"
  }
]
```

*lista recortada a los primeros 10 de 1086 solicitantes para que el archivo sea legible.*

**Lectura del LIS:** El LIS toma el CUIT de esta lista para el médico solicitante del pedido.

## 2. Validación del afiliado

Afiliado de prueba. Se lee `premium` para saber si al confirmar hay que pedirle el token.

**`POST /api/afiliacion/validar`** — enviado 16:59:09 UTC, respuesta HTTP 200 en 853 ms

Pedido:

```json
{
  "tipoDocumento": "DNI",
  "numeroDocumento": "40173783"
}
```

Respuesta:

```json
[
  {
    "accede": "1",
    "documento": "DNI - 40173783",
    "numero_afiliado": "40173783",
    "secuencia": "0",
    "nombre_completo": "CARUSO ALEJANDRO RENE         ",
    "fecha_nacimiento": "1997-01-29",
    "convenio_vigente": "30   - Prestadores Sanatoriales S.A.",
    "plan_vigente": "0125 - PSSA 0125                     ",
    "premium": false
  }
]
```

**Lectura del LIS:** Elegible, afiliado activo. Como `premium` es `false`, el LIS va a pedir el token al confirmar.

## 3. Verificar códigos

Pre-chequeo del pedido antes de autorizarlo. No crea nada.

**`POST /api/codigos/verificar`** — enviado 16:59:10 UTC, respuesta HTTP 200 en 1587 ms

Pedido:

```json
{
  "dni": "40173783",
  "codigos": [
    {
      "codigo": "660711",
      "cantidad": 1
    },
    {
      "codigo": "660412",
      "cantidad": 1
    }
  ]
}
```

Respuesta:

```json
{
  "autorizado": 1,
  "respuesta": "ok",
  "coseguro": 0,
  "codigos": [
    {
      "codigo": "660001",
      "bloqueado": 0,
      "prohibido": 0,
      "auditable": 0
    },
    {
      "codigo": "660412",
      "bloqueado": 0,
      "prohibido": 0,
      "auditable": 0
    },
    {
      "codigo": "660711",
      "bloqueado": 0,
      "prohibido": 0,
      "auditable": 0
    }
  ]
}
```

**Lectura del LIS:** Pedido autorizado en su totalidad, coseguro 0. Las dos prácticas quedan aprobadas.

## 4. Autorizar códigos

Mismo pedido, con el CUIT del solicitante y el diagnóstico en `observaciones`. Crea la orden de prueba, que se anula en el paso 7.

**`POST /api/codigos/autorizar`** — enviado 16:59:12 UTC, respuesta HTTP 200 en 3822 ms

Pedido:

```json
{
  "dni": "40173783",
  "codigos": [
    {
      "codigo": "660711",
      "cantidad": 1
    },
    {
      "codigo": "660412",
      "cantidad": 1
    }
  ],
  "cuit_solicitante": "20186761406",
  "observaciones": "Prueba de integración LIS CEBAC. Orden de prueba, se anula a continuación."
}
```

Respuesta:

```json
{
  "mensaje": "ok",
  "prefijo": "CSB",
  "numero_orden": 119509
}
```

**Lectura del LIS:** Orden creada: el LIS la guarda como `CSB-119509` (prefijo-número) y es lo que después manda a anular o confirmar.

## 5. Confirmar atención sin token

El afiliado de prueba no es premium: se espera el 422 de token requerido.

**`POST /api/ordenes/confirmar`** — enviado 16:59:16 UTC, respuesta HTTP 422 en 1228 ms

Pedido:

```json
{
  "prefijo": "CSB",
  "numero_orden": "119509"
}
```

Respuesta:

```json
{
  "mensaje": "El token es requerido"
}
```

**Lectura del LIS:** El LIS lo lee como “falta el token del afiliado” y se lo pide al operador.

## 6. Confirmar atención con un token inválido

Token `00000000`, inválido a propósito, para ver la respuesta. No tenemos el token real del afiliado.

**`POST /api/ordenes/confirmar`** — enviado 16:59:17 UTC, respuesta HTTP 422 en 1204 ms

Pedido:

```json
{
  "prefijo": "CSB",
  "numero_orden": "119509",
  "token": "***"
}
```

Respuesta:

```json
{
  "mensaje": "Token incorrecto"
}
```

**Lectura del LIS:** El LIS lo lee como “token incorrecto” y vuelve a pedirlo.

## 7. Anular orden

Anulación de la orden creada en el paso 4.

**`POST /api/ordenes/anular`** — enviado 16:59:18 UTC, respuesta HTTP 200 en 1495 ms

Pedido:

```json
{
  "prefijo": "CSB",
  "numero_orden": "119509"
}
```

Respuesta:

```json
[
  {
    "resultado": "ok"
  }
]
```

**Lectura del LIS:** Anulación aceptada.

## 8. Anular orden por segunda vez

La misma orden otra vez, para ver el mensaje de una orden ya anulada.

**`POST /api/ordenes/anular`** — enviado 16:59:20 UTC, respuesta HTTP 200 en 895 ms

Pedido:

```json
{
  "prefijo": "CSB",
  "numero_orden": "119509"
}
```

Respuesta:

```json
[
  {
    "resultado": "No se pudo anular. Anulada anteriormente"
  }
]
```

**Lectura del LIS:** Anulación rechazada, con el texto de ConSalud para el operador.

## 9. Traer autorizaciones

Estado final de las autorizaciones del afiliado de prueba. Sólo lectura.

**`POST /api/autorizaciones`** — enviado 16:59:20 UTC, respuesta HTTP 200 en 889 ms

Pedido:

```json
{
  "dni": "40173783"
}
```

Respuesta:

```json
[
  {
    "prefijo": "VC",
    "numero_orden": null,
    "numero_autorizacion": 350,
    "estado": "Requiere revisión",
    "codigos": [
      {
        "codigo": "660006",
        "descripcion": "ACTH - HORMONA ADRENOCORTICOTROFINA."
      },
      {
        "codigo": "660002",
        "descripcion": "ACETONURIA."
      },
      {
        "codigo": "660001",
        "descripcion": "ACTO BIOQUÍMICO"
      }
    ]
  }
]
```

**Lectura del LIS:** Sólo aparece la autorización VC-350 en “Requiere revisión”. La orden CSB-119509, creada y anulada por API, no figura.

## Observaciones y consultas

1. **Prefijo de la orden.** En producción la orden salió con prefijo `CSB`; la guía ejemplifica con `VC`. El LIS guarda prefijo y número tal como vienen, así que no afecta, pero lo dejamos anotado.
2. **Solicitante “-NO USAR-”.** El primer registro de la lista de solicitantes se llama `-NO USAR-CIBILS MANUEL ANDRES` (y el segundo `-no usar-PEROÑA ...`). Autorizar aceptó ese CUIT. ¿Pueden no incluirlos en la lista, o rechazarlos al autorizar? Si el marcador es sólo el nombre, nosotros tendríamos que filtrar por texto.
3. **Código 660001.** Verificar devuelve en `codigos` el 660001 (ACTO BIOQUÍMICO) aunque no se lo mandemos. ¿Lo agrega ConSalud a toda orden bioquímica? ¿Al autorizar también queda cargado en la orden?
4. **Confirmación con token.** Sin token responde 422 `El token es requerido`; con un token inválido, 422 `Token incorrecto`. La confirmación exitosa quedó sin probar porque no tenemos el token real del afiliado. ¿De dónde lo obtiene el afiliado (app ConSalud) y qué formato tiene?
5. **Orden anulada en traer autorizaciones.** La orden CSB-119509 no aparece en `/api/autorizaciones` después de anulada; sólo figura la autorización VC-350 en “Requiere revisión”. ¿Las órdenes creadas por API no se listan ahí, o dejan de listarse al anularse?
6. **Médico solicitante.** La guía pide el CUIT sólo al autorizar, no al verificar. Si el control de que el médico es prestador de ConSalud se hace recién al autorizar, el operador se entera tarde. Además, nuestro catálogo de médicos tiene matrícula provincial y nacional pero no siempre CUIT: ¿pueden aceptar la matrícula, o devolverla en la lista de solicitantes, para evitar el mapeo manual?

## Cómo lo usa el LIS

- **Validar** se llama al cargar la cobertura del paciente en admisión. Un 403 se muestra al operador con el texto de `mensaje`.
- **Verificar** se llama al cotizar: define el coseguro a cobrar y si el pedido se autoriza solo. Un código `bloqueado` o `prohibido` se marca como rechazado; un `auditable` queda pendiente de auditoría (por ahora el laboratorio no sigue ese flujo).
- **Autorizar** se llama al registrar el pedido, con el CUIT del médico solicitante y el diagnóstico en `observaciones`.
- **Confirmar** se dispara según la configuración del laboratorio: al guardar la orden, en caja o en la toma de muestra. Lleva el token del afiliado salvo que `premium` sea `true`.
- **Anular** sólo sobre órdenes creadas por el LIS (prefijo y número guardados al autorizar).

# MiCAM Salud: análisis para un adapter del broker gateway

Fecha del relevamiento: 2026-10-06. Portal: `https://micamsalud.com.ar/autorizador1.0.5.5/index_autorizador.php`.

Relevamiento de sólo lectura con el usuario del laboratorio (CEBAC SRL, idprestador `13773`): login,
pantallas, historial de solicitudes y una consulta de afiliado. No se creó, anuló, imprimió ni validó
nada. Las credenciales están en la planilla de coberturas y van al `.env` del gateway
(`MICAM_USER` / `MICAM_PASSWORD`), nunca al repo.

## Resumen

- Portal PHP 5.2 / Apache 2.2 sin API: el adapter maneja las pantallas, como `biosoft`.
- **No autoriza en línea**: toda solicitud entra a una cola de **auditoría humana** y el resultado llega
  horas o días después. Es el punto que cambia el diseño respecto de los adapters actuales.
- El usuario CEBAC tiene habilitado un solo financiador: **`357` Admed - Luz y Fuerza - Misiones**
  (coberturas legacy `184` LUZ Y FUERZA INTERNADOS y `499` LUZ Y FUERZA). En el backend ya existe la
  indicación de facturación `PORTAL_MICAM` (`CebacBillingIndicationsSeeder`).
- Usa el código NBU de 6 dígitos (`660412`), no hace falta traducir códigos.

## Particularidades técnicas

- **Base URL**: `https://micamsalud.com.ar/autorizador1.0.5.5/`. La versión está en la ruta: si
  publican otra versión la URL cambia, así que va en config (`MICAM_BASE_URL`).
- **TLS**: la cadena termina en *SSL.com TLS RSA Root CA 2022*. El curl de Git Bash no la tiene y falla;
  el curl de Windows (store del sistema) anda. Verificar que el Node del contenedor la tenga en su
  bundle antes de ir a producción.
- **Sesión**: cookie `PHPSESSID`. Dos logins simultáneos del mismo usuario conviven (el segundo no
  echa al primero), así que una sesión por operación es viable.
- **Sesión vencida o ausente**: responde **200** con un cuerpo tipo `<a href="login.php">Login</a>`, no
  un redirect. Hay que detectarlo por contenido.
- **Encoding**: mezcla páginas `iso-8859-1` y `utf-8`; decodificar como latin1 y des-escapar entidades.
- **Frames**: `index_autorizador.php` es un shell con un iframe `main`; las pantallas reales son las URLs
  internas, se pueden pedir directo.
- Casi todo es `GET` con todos los campos del formulario en el query string (`ajax_ejecutar_php`), y las
  respuestas son fragmentos HTML que el portal inyecta en un `div`. Algunas acciones nuevas son JSON
  (`ventana_delete_ajax.php`, `token_validar.php`, `cobro_coseguro.php`).

## Flujo y endpoints

Rutas relativas a la base URL.

### Login

`POST login.php` (form-urlencoded) con `user`, `pass`. Antes conviene un `GET index_autorizador.php`
para obtener la cookie.

- OK: la respuesta incluye `parent.document.getElementById('form_index').submit();` (recarga el shell).
  `home_prestador2.php` muestra la razón social del prestador.
- Rechazo: vuelve el formulario de login sin ese script.
- Logout: `GET cerrar_sesion.php`.

### Elegibilidad (sin efectos)

`GET buscadores/afiliado_buscador_valida_convenio.php?lst_mutual=357&txt_dni=<dni>&txt_beneficiario=<cred>`

Se manda uno de los dos (DNI o credencial). **La credencial tiene que ir con su formato
`99-99999-9/99`**; sólo dígitos no la encuentra.

Respuesta: fragmento HTML con inputs ocultos/de lectura:

| Campo | Ejemplo | Uso |
|---|---|---|
| `txt_idpersona` | id interno | necesario para validar prácticas y guardar |
| `txt_idplan_mutual` | `857` | necesario para validar prácticas |
| `txt_plan_mutual` | `ESPECIAL` | plan |
| `txt_estado` | `Activo` / `Inactivo` | → `affiliateStatus` |
| `txt_condicion` | `Obligatorio` | tipo de afiliado |
| `txt_af_convenido` | `1` | plan convenido con el prestador |
| `txt_bloquear_transaccion` | `0` | `1` = el portal no deja cargar solicitudes |
| `txt_idcondicion_afiliado`, `txt_idgrupo`, `txt_tipo_documento`, `txt_dni`, `txt_apellido_nombre` | | datos del afiliado |

No encontrado: `<div id="contenedor_af">No se encontr&oacute; el Afiliado.</div>` y los campos vacíos.

La pantalla "Comprobar Elegibilidad" tiene además un botón Guardar (`elegibilidad/afiliados_reg.php`) que
**registra** la elegibilidad y genera una boleta; el adapter no lo necesita.

### Profesional prescriptor (sin efectos)

`GET buscadores/profesional_buscador.php?txt_matricula=<mat>&=&idplan_mutual=&lst_mutual=357`

Encontrado: `txt_idmedico_prescribe` (id interno), `txt_profesional` (nombre). No encontrado: sin
`txt_idmedico_prescribe`.

### Práctica (sin efectos)

`GET buscadores/nomenclador_buscador.php?txt_cod_nomenclador=<nbu>&lst_mutual=357&idplan_mutual=<plan>&lst_prestador=&idpersona=<idpersona>&idmedico_prescribe=<idmedico>&txt_solic_odonto=0`

- Convenida: `txt_idnomenclador`, `txt_cod_nomenclador`, `txt_descrip_nomenclador`.
- No convenida o inexistente: texto `Código no convenido.`
- **Sin `idplan_mutual`/`idpersona` todo da "no convenido"**: hay que consultar el afiliado antes.
- Códigos probados con el plan `857`: `660412`, `660475`, `660005`, `661001` convenidos; `475` (formato
  PAMI) y `999999` no.
- Si hay varias coincidencias el fragmento contiene `abrir_buscador` (el portal abre un popup).

### Cargar solicitud ambulatoria (escritura, NO probado)

Pantalla: `solicitudes_ambulatorio/solicitud_carga_amb.php?accion=agregar`.

1. Por cada práctica: `GET solicitudes_ambulatorio/carga_array.php?txt_idnomenclador=…&txt_cod_nomenclador=…&txt_nomenclador_homologado=&txt_idnomenclador_homologacion=&txt_descrip_nomenclador=…&txt_cantidad=…&txt_idestado_item=1&txt_aclaracion_practica=&txt_iscodigo_cama_nomen=0&mostrar_alerta_consulta_prescriptor=0&lst_mutual=357&txt_iscodigo_fichado=&txt_solic_odonto=0&txt_exceptua_adjuntos=0`.
   Las prácticas quedan en un **array de la sesión PHP** (igual que la grilla de Biosoft): dos
   solicitudes en paralelo sobre la misma sesión se mezclarían. Devuelve la grilla y
   `txt_cantidad_items`.
2. Guardar: `POST solicitudes_ambulatorio/solicitud_reg.php` con todo `frm_edit`. Campos relevantes:
   `accion=agregar`, `lst_tipo=1`, `ambiente=ambulatorio`, `lst_mutual=357`, `txt_fecha_solic`
   (dd/mm/aaaa), datos del afiliado (los del buscador), `lst_novedades`, `txt_idmedico_prescribe`,
   `txt_matricula`, `txt_fecha_presc`, `txt_fecha_realizacion`, `txt_observacion` (diagnóstico en texto,
   obligatorio), `txt_cantidad_items`, `idprestador_agrup=13773`.

Validaciones del front (`solicitud_js.js`, `validar()`): mutual, tipo, afiliado, `lst_novedades`,
prescriptor, fecha de prescripción (no más de un mes antes de la solicitud), al menos un ítem, y adjuntos
si la mutual los exige (`solicitudes_amb_ajax.php` `action=verificar_si_adjunta`). Si
`txt_bloquear_transaccion=1` no deja guardar.

### Buscar solicitudes / estado (sin efectos)

`GET solicitudes_ambulatorio/grilla_buscador_solicitudes.php?lst_mutual=357&txt_idsolicitud_autorizacion=<nro>&txt_fecha_desde=dd/mm/aaaa&txt_fecha_hasta=dd/mm/aaaa&lst_estados=<1|2|vacío>&busqueda=1`

`lst_estados`: `1` pendientes, `2` auditados, vacío todos. Una fila por solicitud con: número
(8 dígitos, `idsolicitud_autorizacion`), fecha, estado, afiliado, seguimiento (mensajes con el
auditor) y acciones.

Estados de solicitud vistos (2026, financiador 357):

| Estado | Cantidad |
|---|---|
| Pendiente de Auditoria | 8 (en curso) |
| Parcialmente Auditado | 8 (en curso) |
| Autorizado | 1428 |
| Autorizado Parcialmente | 1213 |
| Rechazado | 802 |

Hubo solicitudes auditadas el mismo día y otras pendientes desde hace 4 días.

### Comprobante / detalle por práctica (sin efectos)

`GET solicitudes_ambulatorio/solicitud_impresion_bono.php?idsolicitud_autorizacion=<nro>`

Contiene: número de autorización, fechas de solicitud/prescripción/impresión, hora, **estado**, prestador,
afiliado (credencial, plan, condición, DNI), observaciones, prescriptor, diagnóstico y el **detalle de
prestaciones** con `Cod. Nom.` / descripción / cantidad / **estado por práctica** (`Autorizado` /
`Rechazado`). Además:

- Observación del auditor, por ejemplo `ESTA SOLICITUD NO TIENE VALIDEZ HASTA NO VALIDAR EL TOKEN CON EL PRESTADOR`.
- Coseguro total, por ejemplo `COSEGURO EN PRESTADOR` / `COSEGURO: $26000`.
- `txt_dias_validez`: vigencia en días ("SOLICITUD NO VALIDA" si se pasa).

Leer el comprobante **no** lo marca como impreso. Imprimir es otra llamada:
`GET solicitud_reg.php?accion=imprimir_bono&idsolicitud_autorizacion=…` (deja "IMPRESO POR SISTEMA -
VÁLIDO"); el portal advierte "SOLICITUD SIN VALIDEZ SI ES IMPRESA FUERA DEL SISTEMA".

### Token del afiliado (escritura, NO probado)

`POST clases/ajax/token_validar.php` con `idempresa`, `nro_doc`, `num_afiliado`, `token`, `idusuario_reg`,
`idsolicitud_autorizacion` (los valores están en inputs ocultos del comprobante). Respuesta JSON
`[{ token_valido: 1|0, mensaje }]`. Se pide **después** de auditada la solicitud.

### Cobro de coseguro (escritura, NO probado)

`POST clases/ajax/cobro_coseguro.php` con `action=registrar_cobro_coseguro`, `idsolicitud_autorizacion`,
`cobra_financiador` (`1` financiador / `0` prestador), `idusuario`. JSON `{ status, error }`. Sólo se
puede cambiar el mismo día en que se eligió.

### Anular (escritura, NO probado)

1. `GET ventana_delete/ventana_delete_ajax.php?action=verificar_eliminacion&idsolicitud=<nro>&amb=1&idmutual=357`
   → JSON `data: { eliminar: 1|0, req_justificativo, mensaje }`.
2. `POST ventana_delete/ventana_delete_ajax.php` con `action=eliminar_solicitud`, `amb=1`, `idsolicitud`,
   `idmutual=357`, `trabaja_con_nro_ext=0`, `idempresa=`, `nro_solicitud_externo=0`, `motivo`
   → JSON `{ status: 1 }` o `data.mensaje`.

Probablemente sólo se puede mientras está pendiente (a confirmar).

## Propuesta: adapter `micam`

Carpeta `src/infrastructure/brokers/adapters/micam/` siguiendo el patrón de Biosoft:
`micam-config.service.ts`, `micam-http-client.ts` (sesión, cookies, detección de login),
`micam-page-parser.ts` (fragmentos → datos), `micam-adapter.ts`, fixtures HTML anonimizados en
`__fixtures__/` y specs.

| Capability | Implementación |
|---|---|
| `eligibility` | Buscador de afiliado. `Activo` → active/eligible; `Inactivo` → inactive; no encontrado → not_found; `txt_bloquear_transaccion=1` → no elegible con mensaje. Sin efectos, reintentable. |
| `authorization` | Afiliado → prescriptor → prácticas (las "no convenidas" se rechazan por línea con `PRACTICE_NOT_COVERED` sin mandarlas) → `carga_array` → `solicitud_reg`. Respuesta **siempre `pending_audit`** con el número de solicitud. |
| `statusQuery` | **Nuevo en el puerto**: el enum `BrokerCapability.StatusQuery` existe pero no hay método en `BrokerAdapterPort` ni caso de uso. Leería el comprobante y mapearía cada práctica a approved/rejected, más `patientAmountTotal` (coseguro) y la observación del auditor como alerta. |
| `cancellation` | `ventana_delete_ajax` (verificar + eliminar). Solicitud entera, no por práctica. |
| `receipt` | HTML del comprobante (`#impresion` o equivalente), como Biosoft. |
| `memberToken` | `token_validar.php`. Distinto de ITC/SMG: se valida **después** de la auditoría, no al pedir. |

Fases sugeridas:

1. **Elegibilidad sola** (como `visitar`): bajo riesgo, se puede probar contra producción sin efectos.
   Seeder `MicamBrokerGatewaySeeder` en modo manual.
2. **Autorización + statusQuery + receipt + cancelación**: requiere definir antes cómo el LIS maneja
   autorizaciones diferidas (polling del backend, re-consulta a pedido del operador, o un job), porque
   hoy el flujo asume respuesta sincrónica.

## Pendientes a confirmar

1. **Respuesta de `solicitud_reg.php`** al guardar (cómo devuelve el número, cómo informa errores). Hace
   falta una prueba supervisada con un paciente real, anulándola después mientras siga pendiente.
2. **`lst_novedades` (Cond. Afil.)**: el front lo exige, pero para el afiliado consultado sólo tenía una
   opción vacía. Ver qué manda el navegador.
3. **Token**: quién lo provee (¿app de MiCAM del afiliado?) y en qué momento se valida en el mostrador.
4. **Tiempos de auditoría**: hay casos auditados el mismo día y pendientes de varios días; define la
   frecuencia de re-consulta.
5. **Anulación**: confirmar que sólo se puede con la solicitud pendiente y si exige motivo
   (`req_justificativo`).
6. **Adjuntos**: si la mutual 357 exige adjuntar la orden (`verificar_si_adjunta`), el adapter tiene que
   subir archivos (`archivos.php`).
7. **Internación**: existen pantallas `solicitudes_internado/*` (cobertura 184 es "internados"); no se
   relevaron.

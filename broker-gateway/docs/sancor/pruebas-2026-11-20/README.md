# Sancor Salud · primeras pruebas del autorizador HL7 v2.4 (2026-11-20)

Corridas desde el gateway local (`adapters/sancor/`) contra la URL de
producción `https://servicios.sancorsalud.com.ar/Autorizador/HL7v24` con `D` en
MSH-11, como pide Sancor. Prestador `30672372697^CU`; efector el mismo CUIT
(`SANCOR_EFECTOR_CUIT`); prescriptor `86359^MN` (el del documento técnico);
asociado de prueba `121297/00`; token comodín `9999`.

| # | Método | MSH-10 | Prácticas | Respuesta | Nº Sancor |
| --- | --- | --- | --- | --- | --- |
| 1 | Elegibilidad ZQI^Z01 | 26112011280954306693 | — | B000 AUTORIZADA, plan SANCOR 1500 | 306001355 |
| 2 | Validación ZQA^Z21 | 26112011280951616247 | 660475, 660001 | B000 las dos | 306001370 |
| 3 | Autorización ZQA^Z02 | 26112011281270867545 | 660475, 660001 | B000 las dos | 139222934 |
| 4 | Formulario 4 (AUT-9 = 100034432, el del PDF) | 26112011281405010948 | 660001 | M000 / M032 PREAUTORIZACION INEXISTENTE | 306001400 |
| 5 | Anulación ZQA^Z04 de 139222934 | 26112011281601166444 | — | M050 LA AUTORIZACION FUE ANULADA | 139222934 |
| 6 | Anulación repetida | 26112011281728993229 | — | M035 AUTORIZACION YA ANULADA | 139222934 |

Archivos:

- `mensajes-hl7.txt`: cada mensaje enviado y su respuesta, legibles.
- `sancor-adapter.log`: la traza cruda del adapter.
- `N-*.json`: lo que devolvió el gateway al LIS en cada paso.

Lo que se vio:

- Sancor nos devuelve como `164230^PR` "C.E.B.A.C SRL" aunque mandemos el CUIT.
- Los códigos NBU van tal cual, sin nomenclador (`PR1|1||660475^^`), y vuelven
  con su descripción (HEMOGRAMA, ACTO BIOQUIMICO).
- El efector por CUIT lo aceptan; en la validación lo devolvieron como
  `null^CU` y en la autorización como `30672372697^CU`.
- Los segmentos de la respuesta vienen separados por CR, y en la primera
  llamada del día por la entidad `&#xD;` dentro del XML. Nosotros mandamos LF.
- El DG1 `Z71^CONSULTA` lo resuelven con la descripción completa del CIE-10.
- No vino copago (ZAU-6 vacío) para el asociado de prueba.

Pendiente con Sancor: un formulario 4 de prueba, si el efector puede ir por CUIT
en producción, el comodín 9999 durante el arranque y la fecha de pase a `P`.

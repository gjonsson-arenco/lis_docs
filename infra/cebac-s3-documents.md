# CEBAC: documentos en S3, compartidos con Healthbook

Hoy el LIS de CEBAC (on-premise) guarda los documentos en el disco del
servidor, en el volumen `backend-storage`. Esos documentos son las recetas,
los comprobantes y lo que se sube a la ficha. Este runbook los pasa a un bucket
de S3 de la cuenta de AWS de CEBAC, el mismo que usa el Healthbook de CEBAC.

Hay tres motivos:

- **Las órdenes médicas se sincronizan sin mover archivos.** Viaja la key y el
  otro lado copia dentro de AWS. Ver
  [../integrations/medical-orders-sync.md](../integrations/medical-orders-sync.md).
- **Los documentos salen del disco del servidor.** Dejan de depender de sus
  backups y de su espacio.
- **Textract lee directo del bucket**, sin subir los bytes en cada análisis.

## Lo que hay que decidir antes

- **Región.** Healthbook está en `sa-east-1` (São Paulo). Es transferencia
  internacional de datos de salud: **validar con legales (Ley 25.326) antes de
  mover las historias clínicas**.
- **Dependencia de internet.** Con S3, si se corta el enlace de CEBAC,
  admisión no ve las recetas ni sube nuevas.
  - Las pantallas del LIS muestran los documentos con URLs firmadas, de 60
    minutos, que el navegador baja directo de S3. Los puestos necesitan salida
    a `*.s3.sa-east-1.amazonaws.com`.
  - Textract, Bedrock y Healthbook ya dependen de internet, pero esto suma el
    día a día de admisión.

## 1. Bucket

Un bucket para CEBAC: `<bucket-cebac>`, en la misma región que Healthbook.

| Configuración | Valor |
| --- | --- |
| Block Public Access | Las cuatro opciones activadas |
| Cifrado | SSE-KMS con una clave propia (`alias/cebac-documents`), o SSE-S3 si no se quiere administrar KMS |
| Versionado | Activado, más una regla de lifecycle que borre versiones no vigentes a los 90 días |
| CORS | No hace falta: el navegador sólo hace `GET` de URLs firmadas |
| Object Ownership | Bucket owner enforced (sin ACLs) |

Prefijos (los pone cada aplicación con su `AWS_ROOT`):

```
lis/documents/<uuid>.<ext>        lo escribe el LIS
hb/medical-orders/<uuid>.<ext>    lo escribe Healthbook
hb/…                              el resto de Healthbook
```

## 2. Permisos

**LIS on-premise.** No tiene rol de instancia, así que usa un usuario IAM con
access keys (`cebac-lis-onprem`), o IAM Roles Anywhere si se quiere evitar
claves de larga vida. Puede ser el mismo usuario que ya usan Textract y Bedrock,
sumándole esta política:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "LisReadWriteOwnPrefix", "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::<bucket-cebac>/lis/*" },
    { "Sid": "LisReadHealthbookPrefix", "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::<bucket-cebac>/hb/*" },
    { "Sid": "LisList", "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::<bucket-cebac>",
      "Condition": { "StringLike": { "s3:prefix": ["lis/*", "hb/*"] } } },
    { "Sid": "LisKms", "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:sa-east-1:<cuenta>:key/<id>" }
  ]
}
```

Tres notas sobre esta política:

- **Copia desde Healthbook.** Para copiar una receta de Healthbook al LIS
  (`CopyObject`), el LIS necesita `GetObject` sobre `hb/*` y `PutObject` sobre
  `lis/*`. Las dos están.
- **Textract** lee el objeto con las credenciales de quien lo llama, así que
  alcanza con `lis/*`.
- **Endurecer el acceso** (opcional): agregar una condición `aws:SourceIp` con
  la IP pública de salida de CEBAC.

**Healthbook.** Su rol (ECS, EC2 o Lambda) necesita lo simétrico: lectura y
escritura sobre `hb/*`, lectura sobre `lis/*`, `ListBucket` con los dos
prefijos, y la misma clave KMS.

## 3. Variables del backend

Van en `/opt/lis/lis-backend/.env.prod`, que también lee el contenedor
`scheduler`. **No van en el compose**: ahí pisarían lo de `.env.prod`.

```
AWS_ACCESS_KEY_ID=…
AWS_SECRET_ACCESS_KEY=…
AWS_DEFAULT_REGION=sa-east-1
AWS_BUCKET=<bucket-cebac>
AWS_ROOT=lis
DOCUMENTS_DISK=s3        # recién en el paso 4.4
```

Si Textract o Bedrock ya usaban otra región en `AWS_DEFAULT_REGION`, fijarlas
aparte con `AWS_TEXTRACT_REGION` y `AWS_BEDROCK_REGION`.

## 4. Pasaje de local a S3

Se hace sin cortar el servicio. Nada se borra del disco local.

1. **Credenciales y bucket, todavía con el disco local.** Cargar en
   `.env.prod` todo lo del paso 3 menos `DOCUMENTS_DISK`, y recrear el
   backend:

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d backend scheduler
   ```

2. **Ver qué falta copiar:**

   ```bash
   docker compose … exec backend php artisan documents:copy-to-disk s3 --from=local --dry-run
   ```

   Si aparece "faltan en el origen", son documentos cuyo archivo ya no está en
   el disco. Revisarlos antes de seguir.

3. **Copiar el grueso:**

   ```bash
   docker compose … exec backend php artisan documents:copy-to-disk s3 --from=local
   ```

   No pisa lo que ya está en el bucket: se puede cortar y volver a correr.

4. **Cambiar el disco.** Poner `DOCUMENTS_DISK=s3` en `.env.prod` y volver a
   hacer `up -d backend scheduler`. Desde acá, todo lo nuevo va a S3.

5. **Copiar lo que entró en el medio.** El comando mira el disco origen
   explícitamente, así que sigue funcionando con el disco ya cambiado:

   ```bash
   docker compose … exec backend php artisan documents:copy-to-disk s3 --from=local
   ```

6. **Probar.**
   - Abrir una receta vieja y una nueva desde el LIS: la URL tiene que ser de
     S3.
   - Correr un análisis OCR sobre un documento.
   - Mirar en la consola que las keys estén bajo `lis/documents/`.

7. **El disco local queda como respaldo.** Borrar
   `storage/app/private/documents` del volumen recién cuando se esté seguro
   (semanas, no días).

**Vuelta atrás.** `DOCUMENTS_DISK=local` y `up -d`. Lo que se subió mientras
estuvo en S3 se trae con:

```bash
docker compose … exec backend php artisan documents:copy-to-disk local --from=s3
```

## 5. Órdenes médicas desde y hacia Healthbook

Las órdenes médicas se prenden aparte del pasaje a S3, cuando Healthbook tenga
los endpoints del
[brief](../integrations/healthbook-medical-orders-brief.md):

- **LIS → Healthbook.** En Admin > Integraciones, activar
  `send-medical-order-to-healthbook` y `update-medical-order-in-healthbook`.
  Sin S3 viajan los datos, pero no los archivos.
- **Healthbook → LIS.** En el `.env` de lis-infra, poner
  `MEDICAL_ORDER_PULL_ENABLED=true` y hacer `up -d scheduler`. El estado se ve
  en `integration_inbound_cursors`, con `last_success_at` y `last_error`, y en
  `docker logs lis-scheduler`.
- **El cliente de integración** del LIS en Healthbook necesita los scopes
  `medical_orders.read` y `medical_orders.write`.

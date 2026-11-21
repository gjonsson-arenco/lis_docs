# Respaldos del LIS on-premise (CEBAC)

Cómo se respaldan la base y los archivos del LIS, dónde quedan las copias,
cómo se administra desde el panel y cómo se restaura. Vale para QA
(`qa.cebac.ankra-lis.com`) y para la producción que se instala en la VM nueva.

## 1. Qué se respalda y qué no

| Qué | Cómo | Dónde vive hoy |
| --- | --- | --- |
| `lis_production`, `lis_chat`, `lis_messaging` | un `mysqldump --single-transaction` por base, gzip | contenedor `lis-mysql`, volumen `mysql-data` |
| Documentos y adjuntos (recetas, comprobantes, ficha) | `files.tar.gz` de `storage/app/private` y `storage/app/public` | volumen `backend-storage` |
| `manifest.json` | id del respaldo, fecha, clases de retención, SHA-256 de cada archivo | en la misma carpeta |

No entran: Redis (cache y locks; la cola es `database`), `messaging-storage`
(adjuntos salientes ya enviados) y los `.env` reales del server, que siguen
pendientes de un respaldo cifrado aparte (guía de deploy §11). Copiar el
volumen `mysql-data` en caliente **no** es un respaldo: queda inconsistente.

## 2. Dónde quedan las copias

Dos destinos con la misma estructura `<raíz>/<año>/<fecha_hora>/`:

| Destino | Host | Contenedor | Para qué |
| --- | --- | --- | --- |
| Local | `/opt/lis/backups` | `/backups/local` | la copia del día, pocos días (`Días en el disco local`, 7 por defecto) |
| Storage de respaldos | `/mnt/lis-backups` | `/backups/secondary` | la copia que vale: otro storage físico, retención larga |

El storage de respaldos es el share de ~100 GB que se le pidió a CEBAC (SMB o
NFS, accesible desde las dos VMs). Se monta en el host por `fstab`, con el
usuario del contenedor (uid 82, `www-data` de alpine) como dueño, por ejemplo:

```
//<nas>/lis-backups  /mnt/lis-backups  cifs  credentials=/root/.smb-lis-backups,uid=82,gid=82,file_mode=0660,dir_mode=0770,_netdev,nofail  0  0
```

y se marca con un archivo vacío en la raíz:

```bash
sudo touch /mnt/lis-backups/.lis-backups
```

**Sin ese marcador el LIS no copia nada ahí.** Un punto de montaje sin montar
es una carpeta vacía del disco del server, y "copiar al storage" terminaría
siendo una segunda copia local sin que nadie lo note. El panel lo muestra como
"no montado".

El destino local lo tiene que poder escribir el mismo uid:

```bash
sudo mkdir -p /opt/lis/backups && sudo chown 82:82 /opt/lis/backups
```

(`/opt/lis/backups/cebac-users`, el JSON de `export-users.sh`, convive ahí sin
problema: el LIS sólo toca las carpetas por año.)

### Retención

Abuelo-padre-hijo sobre el storage de respaldos, editable en el panel:

- cada respaldo cuenta como **diario** (14 por defecto);
- el primero de cada semana cuenta además como **semanal** (8);
- el primero de cada mes cuenta además como **mensual** (12).

Un respaldo se conserva mientras esté entre los N más nuevos de alguna de sus
clases. La retención se aplica al final de cada respaldo **exitoso**; un
respaldo fallido no borra nada. En el disco local sólo se mira la antigüedad,
y nunca se borra el último completo (si la copia al storage falló, es la única).

### Tercer destino opcional: S3

Además de los dos anteriores, cada respaldo se puede subir a un bucket de S3
(o compatible, por ejemplo MinIO con `BACKUP_S3_ENDPOINT`). Se activa con
`BACKUP_S3_BUCKET` en `lis-infra/.env` y se pausa desde el panel ("Subir a
S3") sin tocar el server. Misma estructura, debajo de `BACKUP_S3_PREFIX`:
`s3://<bucket>/lis-backups/<año>/<fecha_hora>/`. Entra en la misma retención
abuelo-padre-hijo, y si no queda copia local ni en el storage, la
verificación baja el respaldo del bucket a una carpeta temporal y lo prueba
desde ahí.

Lo que necesita del lado de AWS: un bucket con Block Public Access,
versionado opcional, cifrado (los objetos se suben con SSE-S3 `AES256` o con
`aws:kms` más `BACKUP_S3_KMS_KEY_ID`) y un usuario IAM con
`s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:ListBucket` y
`s3:HeadBucket` sobre ese bucket y prefijo. Las credenciales van en
`BACKUP_AWS_ACCESS_KEY_ID` / `BACKUP_AWS_SECRET_ACCESS_KEY`; vacías, el SDK
usa la cadena por defecto. Clase de almacenamiento `STANDARD_IA` por
defecto; para Glacier, mejor una regla de lifecycle en el bucket que
`BACKUP_S3_STORAGE_CLASS`, porque la verificación necesita poder bajar el
objeto en el momento.

**Son historias clínicas**: antes de prenderlo en producción, validar con
legales la región (Ley 25.326), igual que para los documentos en
[cebac-s3-documents.md](./cebac-s3-documents.md). En QA se puede probar con
un bucket de la cuenta de CEBAC en `sa-east-1`.

## 3. Quién lo ejecuta

| Pieza | Dónde corre | Qué hace |
| --- | --- | --- |
| tarea `backups.run` | `lis-scheduler` (`schedule:work`) | dispara `backups:run` a la hora y los días configurados, en un proceso aparte |
| tarea `backups.verify` | `lis-scheduler` | dispara `backups:verify` (domingos 04:00 por defecto) |
| "Respaldar ahora" / "Verificar" | `lis-worker` (cola `database`) | los jobs `RunScheduledTask` y `VerifyBackup`, con timeout de una hora |
| panel | `lis-backend` | estado, retención, historial, borrar copias |

Los tres contenedores comparten las variables `BACKUP_*` (ancla
`x-lis-backup-env` del compose) y los dos bind mounts. El worker lleva
`DB_QUEUE_RETRY_AFTER=3900` para que el driver `database` no re-encole un
respaldo largo a mitad de camino.

La planificación vive en la base (`scheduled_tasks`), no en el `.env`: como
`schedule:work` relanza `schedule:run` cada minuto y ese proceso vuelve a
cargar `routes/console.php`, un cambio desde el panel entra al minuto
siguiente. El entorno sólo manda si dice apagado: `BACKUP_ENABLED=false`
fuerza los respaldos apagados; vacío o `true`, decide el panel. Lo mismo vale
para `MEDICAL_ORDER_PULL_ENABLED` y `VIRTUAL_QUEUE_SYNC_ENABLED` desde esta
versión.

**Si el stack está caído no hay respaldo.** Lo cubre la alerta de "respaldo
vencido" del panel (`Horas sin respaldo para alertar`, 36 por defecto).

## 4. Alta en una instancia

1. Asegurarse de que la imagen del backend sea posterior al 2026-11-21: trae
   `mariadb-client` (mysqldump y mysql, compatibles con MySQL 8).
2. Carpetas y montaje (§2): `/opt/lis/backups` con dueño 82:82, el share en
   `/mnt/lis-backups` con su marcador.
3. En `lis-infra/.env`:

   ```
   BACKUP_DB_USERNAME=lis_backup
   BACKUP_DB_PASSWORD=<openssl rand -base64 24>
   BACKUP_VERIFY_DATABASE=lis_restore_check
   ```

4. Crear el usuario de respaldos (sólo lectura + dueño de la base de prueba):

   ```bash
   /opt/lis/lis-infra/scripts/create-backup-user.sh
   ```

5. Recrear los contenedores para que tomen variables y montajes:

   ```bash
   cd /opt/lis/lis-infra && docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d backend scheduler worker
   ```

6. Permisos en el LIS: `CebacSeeder` ya corre `MaintenancePermissionSeeder`
   (`backups.manage`, `scheduled_tasks.manage`, los dos al rol `admin`). En una
   instancia que no se re-seedea:

   ```bash
   docker compose … exec -T backend php artisan db:seed --class=MaintenancePermissionSeeder --force
   docker compose … exec -T backend php artisan db:seed --class=ApplicationSettingsSeeder --force
   ```

   (el segundo crea los parámetros de retención; el panel los crea solo
   igual la primera vez que se guardan).

7. En Administración > Respaldos: "Respaldar ahora", esperar el resultado,
   "Verificar". Recién después prender el respaldo programado y la
   verificación automática y elegir la hora.

Hasta que CEBAC entregue el share, se puede desplegar y dejar prendido
igual: queda la copia local en `/opt/lis/backups` y el panel avisa en
amarillo que no hay storage de respaldos. Cuando llegue el share, alcanza con
montarlo en `/mnt/lis-backups`, crear el marcador y esperar al próximo
respaldo: no hace falta redeploy, porque el bind mount del punto de montaje
ya está y va con propagación `rslave`, así que el mount nuevo del host se ve
adentro. Comprobarlo con
`docker compose … exec -T worker ls -la /backups/secondary/`; si no aparece
el marcador, `up -d backend scheduler worker` lo resuelve. Si en el medio se quiere una copia fuera
del edificio, el bucket de S3 (§2) se agrega con las variables
`BACKUP_S3_*` y `up -d backend scheduler worker`.

## 5. Verificación

"Verificar" (a mano o la tarea de los domingos) toma el último respaldo
completo y, desde la copia que siga disponible:

- compara el SHA-256 de cada archivo con el manifiesto;
- restaura cada dump entero en `lis_restore_check` (la crea y la borra;
  las cláusulas `DEFINER` se quitan al vuelo) y compara la cantidad de tablas
  con la que había al momento del dump;
- lista el `files.tar.gz` para saber que se puede leer.

El resultado queda en el respaldo (`verification_status`, detalle por
archivo) y en la tarjeta "Último verificado". Una verificación que falla no
borra nada; es la señal de revisar antes de que haga falta restaurar.

## 6. Restaurar

El restore a producción **no** está en el panel a propósito: cuando hace
falta, el LIS está caído o a medio reemplazar. Es `scripts/db-restore.sh`,
en el server:

```bash
cd /opt/lis/lis-infra
docker compose -f docker-compose.yml -f docker-compose.prod.yml stop backend scheduler worker reverb
./scripts/db-restore.sh /mnt/lis-backups/2026/2026-11-21_020000 lis_production
# idem lis_chat y lis_messaging si hace falta
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d backend scheduler worker reverb
docker compose -f docker-compose.yml -f docker-compose.prod.yml exec -T backend php artisan migrate --force
docker compose -f docker-compose.yml -f docker-compose.prod.yml exec -T backend php artisan cache:clear
./scripts/reload-rules-cache.sh
```

El script verifica el checksum contra el manifiesto, pide escribir el nombre
de la base para confirmar y dropea y recrea la base destino. Con
`--into=<otra_base>` restaura al costado, para mirar sin pisar nada.

Los archivos se restauran al volumen `backend-storage`:

```bash
docker run --rm -v lis-infra_backend-storage:/var/www/storage -v /mnt/lis-backups/2026/2026-11-21_020000:/b:ro alpine \
  sh -c 'cd /var/www/storage/app && tar -xzf /b/files.tar.gz'
```

(el nombre real del volumen sale de `docker volume ls | grep backend-storage`).

## 7. Qué mirar cuando algo falla

| Síntoma en el panel | Causa habitual | Qué hacer |
| --- | --- | --- |
| "Respaldo vencido" y el programado prendido | el scheduler o el worker no corren, o el respaldo falla | `docker compose … logs scheduler worker`, abrir el último respaldo y leer el log |
| Storage "no montado" | el share se cayó o falta el marcador | `mount | grep lis-backups`, `ls -la /mnt/lis-backups/.lis-backups` |
| Base "falló" con `Access denied` | el usuario de respaldos no existe o cambió la clave | volver a correr `create-backup-user.sh` y recrear los contenedores |
| Verificación: "restaurado con N tablas, se esperaban M" | dump truncado o copia dañada | verificar la otra copia (local vs storage); si las dos fallan, respaldar de nuevo |
| Respaldo "en curso" eternamente | el worker murió con el job a medias | el próximo disparo lo cierra como caído; `docker compose … restart worker` |

## 8. Lo que falta

- Aviso por mail o WhatsApp cuando el respaldo vence: hoy es sólo el panel.
  Pide una comunicación nueva en `lis-messaging-service`.
- Binlogs de MySQL copiados cada hora para recuperación a un minuto exacto
  (MySQL 8 los tiene prendidos por defecto).
- Respaldo cifrado de los `.env` reales del server.
- Un cron del host como red de seguridad si el stack está caído: se decidió
  no duplicar la lógica por ahora; la alerta del panel lo cubre.

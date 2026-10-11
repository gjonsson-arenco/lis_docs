# Fila virtual: entrar a la fila desde la app de Healthbook

El paciente, desde la app móvil de Healthbook, **entra a la fila de atención
de una sede como si estuviera en el tótem**, y si tiene un turno agendado se
**anuncia solo cuando llega**, sin pasar por el tótem. Es una función
exclusiva de la app: el web de Healthbook no la tiene a propósito.

Partes del documento:

- **Diseño**: las reglas y por qué.
- **Qué está hecho**: en cada repositorio.
- **Puesta en marcha** y **qué falta**.

Documentos relacionados:

- El circuito de pull sobre el que se apoya: [medical-orders-sync.md](./medical-orders-sync.md)
  ("Restricción de partida: el LIS es on-premise").
- El módulo de filas del LIS: la fila es la entidad; el ticket nace en el tótem
  y recorre admisión → caja → toma.

## Qué ve el paciente

1. **Entrar a la fila ("Sacar turno").** En la app, la pantalla "Fila virtual"
   lista las sedes con fila, con la **espera estimada**, cuántos esperan y a
   qué distancia está cada una. Si puede, toca "Sacar turno" y en segundos ve
   su código (A-023), cuántas personas tiene adelante y los minutos
   estimados. Cuando lo llaman, la pantalla se pone verde con el puesto
   ("Box 2"), igual que el display de la sala.
2. **Anunciarse con turno ("Ya llegué").** En el turno agendado, sobre la
   hora, aparece el botón "Ya llegué, anunciarme". Al tocarlo entra a la fila
   con una nota para el puesto ("Se anunció desde la app con turno de las
   10:30") y sigue el mismo recorrido.
3. **Cancelar.** Hasta que lo atiendan, desde la misma pantalla.

## Los criterios (configurables por sede)

El usuario los fijó así: *si el tiempo de espera es bajo tiene que estar en
las inmediaciones del lugar; hay que usar la localización*. Se configuran en
el LIS, en **Configuración > Sistema > Fila virtual**, por sede:

| Criterio | Qué hace | Default |
| --- | --- | --- |
| Coordenadas y dirección de la sede | Sin coordenadas no se puede medir la distancia y la fila virtual no se puede prender. | — |
| `enabled` | La fila virtual de la sede. | apagada |
| `queue_id` | En qué fila nace el turno virtual. | la de admisión de la sede |
| `short_wait_minutes` | Con una espera estimada **menor** a esto, el paciente tiene que estar en las inmediaciones. | 15 |
| `nearby_radius_meters` | Las inmediaciones. | 300 m |
| `max_radius_meters` | Con más espera, hasta esta distancia. Vacío = desde cualquier lado. | 5 km |
| `max_remote_waiting` | Tope de turnos virtuales esperando a la vez. Vacío = sin tope. | sin tope |
| `opens_at` / `closes_at` | Horario en que se reciben pedidos, en la zona del laboratorio. Vacío = todo el día hábil (`laboratory_calendar.working_weekdays` y feriados). | todo el día |
| `default_service_minutes` | Minutos por paciente mientras no hay ritmo del día. | 6 |
| `checkin_enabled` | El paciente con turno puede anunciarse desde la app. | sí |
| `checkin_before_minutes` / `checkin_after_minutes` | La ventana para anunciarse alrededor de la hora del turno. | 15 antes, 10 después |
| `checkin_radius_meters` | Distancia máxima para anunciarse. Vacío = sin exigir ubicación. | 300 m |

La regla (`VirtualQueueEligibility` en el LIS, `QueueEligibility` en
Healthbook, **la misma función**):

- **Entrar a la fila**: prendida → la sede está recibiendo (día hábil, horario,
  fila activa) → no se pasó el tope → si la espera es corta, a menos de
  `nearby_radius_meters`; si no, a menos de `max_radius_meters` (o sin
  ubicación si no hay tope). Sin ubicación cuando hace falta:
  `location_required`; lejos: `too_far`.
- **Anunciarse**: prendida y `checkin_enabled` → la sede está recibiendo →
  dentro de la ventana → a menos de `checkin_radius_meters` si está puesto.

**La ubicación la manda el teléfono** (`expo-location`, en primer plano, al
entrar a la pantalla y al tocar el botón; nunca en segundo plano). Healthbook
la guarda con el pedido (`latitude`, `longitude`, `accuracy_m`) y el LIS la
vuelve a evaluar al emitir. No hay verificación contra suplantación de GPS: el
criterio es de conveniencia operativa, no de seguridad, y el puesto ve la
nota "entró desde la app" para decidir.

## Arquitectura: el LIS manda y el LIS lee

La fila vive en el LIS, que es on-premise y no recibe conexiones desde la nube
(ver medical-orders-sync.md). Por eso **los dos sentidos los abre el LIS**,
por la cara sincrónica de `lis-adapter-healthbook`, en un ciclo cada 15 s
(`integrations:sync-virtual-queue`, en el contenedor `scheduler`):

| Sentido | Qué viaja | Cómo |
| --- | --- | --- |
| Healthbook → LIS | Los pedidos del paciente: `requested` (entrar o anunciarse) y `cancelled`. | El LIS lee el feed `GET /integration/queue/requests/changes` (outbox de Healthbook, cursor en `integration_inbound_cursors`, stream `virtual_queue_requests`). |
| LIS → Healthbook | **La foto de cada sede**: coordenadas, criterios, estado de la fila (abierta, espera estimada, cuántos esperan) y el estado de cada pedido reciente (el ticket con código, puesto, cuántos adelante y minutos; o `rejected` con motivo). | `PUT /integration/queue/sites/{code}`, en cada vuelta del ciclo y **en el acto** cuando un ticket virtual cambia (se lo llama, atiende, cierra): `VirtualQueueTicketWatcher` despacha `PushVirtualQueueSiteSnapshot` después de la respuesta (`dispatchAfterResponse`), porque en producción no hay worker de colas y el puesto no tiene por qué esperar a Healthbook. |

Decisiones:

- **Una foto, no eventos por ticket.** Healthbook no necesita el historial: le
  alcanza con el estado actual de la sede y de cada pedido. Una foto perdida
  la repone la siguiente; los pedidos cerrados siguen viajando 20 minutos
  (`VIRTUAL_QUEUE_SYNC_RECENT_MINUTES`) por si se perdió la del cierre.
- **Healthbook evalúa primero, el LIS decide.** Con la última foto, Healthbook
  contesta en el acto (`409 TooFar`, `LocationRequired`, `SiteClosed`…), así
  el paciente no espera 15 s para enterarse de que está lejos. Si las dos
  evaluaciones difieren (la foto estaba vieja), el pedido vuelve `rejected`
  con `reject_reason`. Una foto más vieja que 5 minutos
  (`VIRTUAL_QUEUE_STATUS_STALE_MINUTES`) cierra la sede en Healthbook: si el
  LIS está caído, nadie entra a una fila que nadie atiende.
- **Un pedido vigente por paciente.** Como en el tótem.
- **El paciente puede no tener ficha.** El LIS lo cruza como en las órdenes
  médicas (`lis_patient_id`, enlace externo, documento + nacimiento). Si no
  lo encuentra, **igual emite el ticket**, con el documento y la nota "Dice
  ser Ana Pérez; sin ficha en el LIS": admisión lo vincula al atenderlo, como
  con un ticket anónimo del tótem.
- **La cancelación del paciente no revive.** Si cancela en la app y la foto
  siguiente del LIS todavía lo trae `waiting`, Healthbook lo deja cancelado;
  el LIS cancela el ticket en la vuelta siguiente.
- **Las sedes se cruzan por código.** `sites.lis_site_code` en Healthbook es
  el `sites.code` del LIS. Una sede que el LIS manda por primera vez se crea
  en Healthbook **deshabilitada para turnos**; si ya había una con el mismo
  `code`, se enlaza. En dev, la "Sede Central" de Healthbook es la `0` del LIS.
- **Un ticket sabe por dónde entró.** `tickets.source`: `kiosk`, `staff` o
  `healthbook`. Los puestos ven la insignia "Desde la app".

### La espera estimada

No existía. `QueueWaitEstimator` la calcula igual para el tótem, la app y el
administrador: el **ritmo real de la última hora** en esa fila (atenciones
terminadas por minuto, `stage_completed`) aplicado a los que están adelante;
mientras no haya al menos 3 atenciones (`VIRTUAL_QUEUE_MIN_COMPLETIONS_FOR_RATE`),
los minutos por paciente de la sede repartidos entre los puestos que llamaron
en los últimos 30 minutos. Es una estimación: la app la muestra como "~18 min".

### Latencia

| Paso | Tiempo |
| --- | --- |
| El paciente toca "Sacar turno" → Healthbook lo acepta | inmediato |
| El LIS lee el pedido y emite el ticket | ≤ 15 s |
| La app ve el código | ≤ 15 s + el polling de la app (7 s mientras el pedido está vigente) |
| Lo llaman → la app lo muestra | segundos (foto en el acto + polling) |

## Qué está hecho (2026-10-11)

**lis-backend**

- Migración `2026_10_11_100000_create_virtual_queue_tables`: `sites.address/latitude/longitude`,
  `virtual_queue_site_settings`, `virtual_queue_requests`, `tickets.source`.
- `App\Services\Queueing\QueueWaitEstimator` y `App\Services\Queueing\VirtualQueue\*`
  (elegibilidad, estado de sede, importador del feed, publicador de fotos,
  ciclo, watcher), el job `PushVirtualQueueSiteSnapshot` y el comando
  `integrations:sync-virtual-queue` programado cada 15 s.
- `config/virtual_queue.php`; `VIRTUAL_QUEUE_SYNC_ENABLED` apagado por defecto.
- Admin: `GET/PUT /api/v1/admin/virtual-queue/sites/{site}` (`can:users.manage`).
- Tests unitarios de la regla y del estimador; smoke contra la base de dev con
  el adapter simulado (pedido cercano → ticket; lejano → `too_far`;
  cancelación → ticket cancelado).

**lis-adapter-healthbook**

- `GET /api/v1/virtual-queue/requests/changes` y `PUT /api/v1/virtual-queue/sites/{code}`;
  stub con un pedido de Pérez para la sede `0`.

**lis-healthbook-back**

- Migración `2026_10_11_000001_create_virtual_queue_tables` (`sites.*`,
  `queue_requests`, `integration_outbox`), scopes `queue.read` / `queue.write`,
  setting `virtualQueueEnabled` (`features.virtual_queue` en `/me`).
- Portal: `GET /queue/sites`, `GET /sites/{site}/queue`, `GET|POST /patients/{patient}/queue-requests`,
  `GET /queue-requests/{id}`, `POST /queue-requests/{id}/cancel`;
  `AppointmentResource.check_in`.
- Integración: `PUT /integration/queue/sites/{code}`, `GET /integration/queue/requests/changes`.
- Tests de feature del portal y de la integración.

**lis-healthbook-front**

- `@healthbook/types` (`queue.ts`), `@healthbook/core` (`useQueueSites`,
  `useSiteQueue`, `useActiveQueueRequest`, `useQueueRequest`, `useQueueMutations`).
- App móvil: pantallas `queue/index` (sedes y "Sacar turno") y `queue/[id]` (el
  turno en vivo), botón "Ya llegué" en los turnos y en el inicio, tarjeta "Tu
  turno en la fila" en el inicio; `expo-location` (hace falta un dev build nuevo).

**lis-front-monorepo (LIS)**

- Configuración > Sistema > **Fila virtual** (`SettingsVirtualQueueClient`).
- Insignia "Desde la app" en el ticket en curso y en el puesto de atención.

**lis-infra (deploy/cebac)**

- `VIRTUAL_QUEUE_SYNC_ENABLED` / `VIRTUAL_QUEUE_SYNC_EVERY_SECONDS` en el
  contenedor `scheduler`, apagados.

## Puesta en marcha

1. **Healthbook**: migrar; darle `queue.read` y `queue.write` al cliente de
   integración del LIS (recrearlo con `integration:client-create` o editar
   `integration_clients.scopes`); poner `virtualQueueEnabled` en `true`
   (Administración > Parámetros).
2. **LIS**: migrar; en Configuración > Sistema > Fila virtual cargar la
   dirección y las coordenadas de la sede, elegir la fila y prender la fila
   virtual; `VIRTUAL_QUEUE_SYNC_ENABLED=true` en el servidor (el `scheduler`
   ya corre). La primera foto crea o enlaza la sede en Healthbook.
3. **App**: dev build nuevo con `expo-location` (`expo run:android`); en
   Expo Go ya viene.

## Qué falta

- **Aviso cuando lo llaman con la app cerrada.** Hoy el llamado se ve si la
  app está abierta (polling). Lo natural es un push (`expo-notifications`),
  que es un pendiente general de la app (endpoints de dispositivos en
  Healthbook). Mientras tanto, la pantalla dice que hay que estar en la sede
  y el display sigue anunciando.
- **El tótem no muestra la espera estimada** todavía (sigue con "N personas
  antes que vos"); el estimador ya está para usarlo en `KioskController`.
- **Prioridad del que se anuncia con turno.** Entra con prioridad normal en la
  fila elegida; si CEBAC quiere que el turno agendado pase antes, es un
  `attention_type` o una fila propia para turnos, configurable en la sede.
- **El turno agendado vive en Healthbook**, no en el LIS: el ticket lleva la
  hora en la nota pero no un `appointment_id` del LIS. Si algún día los
  turnos se sincronizan, el pedido ya trae `appointment.external_id`.
- **Smoke punta a punta** con la app en el emulador contra el LIS y Healthbook
  locales (el circuito se probó por partes: LIS con adapter simulado,
  Healthbook con tests de feature, app con chequeo de tipos).

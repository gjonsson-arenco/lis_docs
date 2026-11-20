# Impresión en el LIS: documentos, tickets y rótulos

Qué imprime cada pantalla, por dónde sale y qué hay que configurar en una
sede nueva. Estado al 2026-11-20.

## Dos caminos

| Qué | Quién lo arma | Cómo llega al papel |
| --- | --- | --- |
| Factura, comprobante de atención, presupuesto, ticket del tótem | `lis-reporting-service` (HTML/PDF desde una plantilla Nunjucks + Chromium) | El navegador: iframe oculto y diálogo de impresión (`printHtmlDocument`) |
| Indicaciones de los estudios | PDF almacenado por requisito (`storage/app/requirements`, bind mount en el server) | Se abre en otra pestaña y se imprime desde ahí |
| Comprobante del financiador | El broker (HTML o PDF que devuelve el portal) | Igual que los documentos |
| Rótulos de muestras | El backend compila el diseño de la etiqueta a ZPL o EPL | Directo a la impresora por TCP 9100, desde el backend |

No hay servicio de colas de impresión. El backend on-premise está en la misma
red que las impresoras; cuando un backend fuera de la red tenga que imprimir
en el laboratorio se retoma `lis-print-engine`, que hoy es un esqueleto sin
uso.

## Rótulos: cómo se configura una impresora de etiquetas

1. **Dispositivo.** Configuración → Sistema → Dispositivos → nuevo, tipo
   *Impresora*: lenguaje `zpl` (Zebra ZD/ZT/GK y compatibles) o `epl` (Zebra
   LP/TLP 2844 y Eltron), IP de la impresora y puerto (vacío = 9100).
2. **Estación.** Configuración → Estaciones → la estación → *Etiquetas*:
   elegir esa impresora. Una misma impresora puede estar en varias
   estaciones, y una estación puede usar la misma para etiquetas y
   comprobantes.
3. **Etiqueta.** Configuración → Etiquetas: hay una por defecto de 50x25 mm
   a 203 dpi; cada tipo de muestra puede tener la suya. El editor muestra la
   etiqueta tal como sale, con los valores de una muestra real por localizador.

Con eso, "Imprimir rótulos" en admisión y en toma de muestras manda la orden
entera (o un tubo, desde la toma) sin ningún diálogo. El backend contesta:

- `422` si la estación no tiene impresora de etiquetas, si la impresora habla
  otro lenguaje o si no hay etiqueta diseñada;
- `502` si la impresora no responde (apagada, otra IP, puerto cerrado).

Sin impresora en red (una Zebra por USB con driver de Windows, o papel común
para probar) el mismo botón abre un diálogo con los rótulos dibujados a
tamaño real, y los imprime con el diálogo del navegador en páginas del tamaño
de la etiqueta. Hay que poner esa impresora como predeterminada y el tamaño
de papel de la etiqueta en el driver.

### Probar un diseño sin impresora

`GET /api/v1/orders/{id}/labels?language=zpl` descarga el ZPL de la orden.
Pegado en <https://labelary.com/viewer.html> (8 dpmm, etiqueta de
1.9685 x 0.9843 pulgadas para la de 50x25) se ve exactamente lo que va a
imprimir la Zebra. Con `language=epl` sale el EPL en Windows-1252.

### Puertos y red

- La impresora tiene que ser alcanzable **desde el contenedor del backend**,
  no desde la PC del operador: `docker exec lis-backend sh -c 'nc -vz IP 9100'`
  (o `php -r` con `fsockopen`) desde el server.
- Timeouts: `PRINTING_CONNECT_TIMEOUT_SECONDS` (5) y
  `PRINTING_WRITE_TIMEOUT_SECONDS` (10) en el `.env` del backend.
- Conviene IP fija o reserva DHCP para cada impresora: el LIS guarda la IP.

## Tótem

El ticket de 80 mm lo dibuja `lis-reporting-service` y el tótem lo imprime
con el diálogo del navegador. Para que salga solo:

- Chrome en modo kiosco con `--kiosk --kiosk-printing`.
- La tickeadora como impresora predeterminada de Windows, papel de 80 mm y
  márgenes en cero en el driver.

Lo mismo sirve en una caja si el diálogo de impresión molesta: el comprobante
sale en la predeterminada sin preguntar.

## Lo que falta

- Informe de resultados de CEBAC: sólo existe la plantilla de HUA. Mientras
  los resultados los emita Labcore no bloquea.
- Impresoras que no hablen ZPL ni EPL (por ejemplo una térmica de 80 mm para
  rótulos): se agrega un compilador en `App\Services\Labels`, no un transporte.

<a name="inicio"></a>
Tango Software - API REST de Tango eCommerce
=======

API REST para integrar una tienda de comercio electrónico con **Tango Gestión** o **Tango Punto de Venta**: envío de órdenes y consulta de datos (artículos, clientes, precios, stock, comprobantes, etc.).

- [Índice de recursos](#endpoints)
- [Instalación](#instalacion)
  - [Versiones soportadas](#versiones)
  - [Requisitos previos](#requisitos)
  - [Generalidades](#generalidades)
- [Acceso a la API](#acceso)
  - [Credenciales](#credenciales)
  - [Verificar la conexión (Dummy)](#dummy)
- [Cómo interpretar la respuesta](#respuesta)
  - [Formato de respuesta y paginación](#envelope)
  - [Códigos de estado HTTP](#status)
- [Recepción de órdenes](#ordenes)
  - [Notificaciones (webhooks)](#notificaciones)
  - [Órdenes por lote](#lote)
  - [Preguntas frecuentes](#faqs)
  - [Novedades del JSON de la orden](#novedades)
  - [Datos del JSON](#djson)
  - [Tablas de referencia](#tablas)
  - [Ejemplos de JSON](#ejemplos)
- [Consulta de datos](#consulta)
  - [Recursos de consulta](#recursos)
- [Migración desde la API de Tango Tiendas](#migracion)
- [Consideraciones](#consideraciones)

<a name="endpoints"></a>

## Índice de recursos

Todas las rutas parten de la base `https://{llave}.connect.axoft.com/api/eCommerce`. Todos los recursos requieren los headers [`ApiAuthorization` y `Company`](#credenciales).

| Método | Ruta | Descripción |
| ----- | ---- | ----------- |
| POST | [`/Dummy`](#dummy) | Verificación de credenciales (prueba de autenticación). |
| POST | [`/Order`](#ordenes) | Recibe una orden de la tienda para generar el pedido en Tango. Requiere [`?Cuenta=`](#requisitos). |
| POST | [`/Order/Batch`](#lote) | Recibe órdenes en lote (hasta 25) para generar los pedidos en Tango. Requiere [`?Cuenta=`](#requisitos). |
| GET | [`/ArtPorSaldoStock`](#recartsaldo) | Artículos filtrados por saldo de stock. |
| GET | [`/CounterFoil`](#reccounterfoil) | Talonarios. |
| GET | [`/Currency`](#reccurrency) | Monedas. |
| GET | [`/CurrencyExchangeRate`](#reccurrencyrate) | Cotizaciones de moneda extranjera. |
| GET | [`/Customer`](#reccustomer) | Clientes. |
| GET | [`/CustomersFolder`](#reccustomersfolder) | Clasificador de clientes: clientes en carpetas. |
| GET | [`/CustomersFolderClassifier`](#reccustomersclassifier) | Clasificador de clientes: carpetas. |
| GET | [`/CustomersRelation`](#reccustomersrelation) | Clasificador de clientes: relaciones. |
| GET | [`/DiscountByCustomer`](#recdiscount) | Descuentos por cliente. |
| GET | [`/Invoices`](#recinvoices) | Relación entre cada orden y el comprobante electrónico de su pedido facturado (con enlace al PDF). |
| GET | [`/Measure`](#recmeasure) | Unidades de medida. |
| GET | [`/Order`](#recorder) | Obtener información de las órdenes recibidas. |
| GET | [`/Price`](#recprice) | Precios de artículos. |
| GET | [`/PriceByCustomer`](#recpricebycustomer) | Precios por cliente. |
| GET | [`/PriceList`](#recpricelist) | Listas de precios. |
| GET | [`/Product`](#recproduct) | Artículos (composición, comentarios y escalas). |
| GET | [`/ProductsFolder`](#recproductsfolder) | Clasificador de artículos: artículos en carpetas. |
| GET | [`/ProductsFolderClassifier`](#recproductsclassifier) | Clasificador de artículos: carpetas. |
| GET | [`/ProductsRelation`](#recproductsrelation) | Clasificador de artículos: relaciones. |
| GET | [`/Publications`](#recpublications) | Publicaciones (relación artículo de la tienda ↔ artículo de Tango). |
| GET | [`/SaleCondition`](#recsalecondition) | Condiciones de venta. |
| GET | [`/Scale`](#recscale) | Escalas. |
| GET | [`/Seller`](#recseller) | Vendedores. |
| GET | [`/Stock`](#recstock) | Saldos de stock por artículo/depósito/sucursal. |
| GET | [`/StockGroupByProduct`](#recstockgroup) | Saldos de stock agrupados por artículo. |
| GET | [`/Store`](#recstore) | Sucursales. |
| GET | [`/Transport`](#rectransport) | Transportes. |
| GET | [`/ValueScale`](#recvaluescale) | Valores de escala. |
| GET | [`/Warehouse`](#recwarehouse) | Depósitos (incluye depósitos de stock centralizado). |

<a name="instalacion"></a>

## Instalación

<a name="versiones"></a>

### Versiones soportadas

[<sub>Volver al índice</sub>](#inicio)

La API de Tango eCommerce está disponible **a partir de la versión Delta 6** de **Tango Gestión** o **Tango Punto de Venta Argentina**.

**Importante:** el requerimiento mínimo de TLS es la versión **1.2**. No se da soporte a TLS 1.0 ni TLS 1.1. Esto aplica tanto a las llamadas a la API como al servidor del webhook de notificaciones.

<a name="requisitos"></a>

### Requisitos previos

[<sub>Volver al índice</sub>](#inicio)

Para poder consumir la API, la empresa debe tener completada la puesta en marcha de eCommerce en Tango:

- El producto **eCommerce** activo en **Tango Billing**.
- La empresa vinculada con **Tango Connect** y con **eCommerce**.
- Una **Cuenta de Integración API** creada. Su nombre es el valor que se envía en el parámetro `?Cuenta=` de los `POST` (y opcionalmente como filtro en los GET [`Order`](#recorder), [`Invoices`](#recinvoices) y [`Publications`](#recpublications)).

<a name="generalidades"></a>

#### Generalidades

[<sub>Volver al índice</sub>](#inicio)

- La API admite órdenes únicamente en **moneda corriente**.
- Los datos de importes y precios se expresan con hasta **2 decimales**, usando el punto (`.`) como separador decimal. Informar una cantidad mayor de decimales no provoca el rechazo de la orden, pero el tratamiento difiere según el dato:
  - Los **importes de los pagos** (`Payments.Total` y `CashPayments.PaymentTotal`) se **redondean a 2 decimales** antes de registrarse.
  - El **resto de los datos** (precios, cantidades, totales, descuentos y recargos) se registra **tal como se informa**.
  - Las [fórmulas de validación](#formulas) de `Total` y `PaidTotal` comparan los valores redondeados a 2 decimales.

  El criterio de redondeo funciona igual que el redondeo tradicional, salvo cuando el valor queda exactamente a mitad de camino (la parte descartada es exactamente `5`): en ese caso, en lugar de redondear siempre hacia arriba, el segundo decimal queda en el valor **par** más cercano. Ejemplos: `10.126` → `10.13` y `10.124` → `10.12` (igual que el redondeo tradicional); `10.125` → `10.12` (el 2 ya es par) y `10.135` → `10.14` (el 3 sube al par 4).
- A la API se accede a través de **Tango Connect**: la URL base tiene el formato `https://{llave}.connect.axoft.com/api/eCommerce`, donde `{llave}` es el número de llave del sistema Tango reemplazando la barra `/` por un guion `-` (por ejemplo, la llave `000000/000` corresponde a `https://000000-000.connect.axoft.com/api/eCommerce`).

<a name="acceso"></a>

## Acceso a la API

[<sub>Volver al índice</sub>](#inicio)

Para consumir la API sólo necesita las credenciales de acceso. Lo que debe estar configurado en Tango se lista en [Requisitos previos](#requisitos).

<a name="credenciales"></a>

### Credenciales

Todas las llamadas a la API se autentican con **dos headers**:

| Header | Contenido | Dónde obtenerlo |
| ------ | --------- | --------------- |
| `ApiAuthorization` | **Token de desarrollador** de Tango. | Menú de usuario › **Desarrollador** › botón **Generar**. Copie el "Token de desarrollador". |
| `Company` | **IdEmpresa**: identificador interno de la empresa vinculada en Tango. | En la columna IDEmpresa de la tabla Empresa de la base de datos de diccionario |

Ejemplo de headers:

```
ApiAuthorization: D3D5444E-B8DE-455A-91DE-4960B7110FC9
Company: 1
```

**Ambos headers son obligatorios en todos los recursos** (incluidos los GET). En ausencia de cualquiera de los dos, o en caso de que su valor sea vacío, obtendrá como respuesta un código de error **401**.

<a name="dummy"></a>

### Verificar la conexión (Dummy)

Para validar las credenciales, realice un **POST** (sin cuerpo) al recurso `Dummy`:

```
POST https://{llave}.connect.axoft.com/api/eCommerce/Dummy
```

Respuesta cuando el acceso es válido:

```json
{
  "Status": 0,
  "Message": "Valid ApiAuthorization",
  "Data": null,
  "isOk": true
}
```

<a name="respuesta"></a>

## Cómo interpretar la respuesta

[<sub>Volver al índice</sub>](#inicio)

<a name="envelope"></a>

### Formato de respuesta y paginación

Salvo `Dummy`, **todas las respuestas GET usan el mismo formato**. Los datos se entregan de forma **paginada**:

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": true
  },
  "Data": [ /* registros del recurso */ ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```

| Campo | Descripción |
| ----- | ----------- |
| `Paging.PageNumber` | Número de página actual. |
| `Paging.PageSize` | Tamaño de página solicitado. |
| `Paging.MoreData` | Indica si existe una página siguiente con datos. |
| `Data` | Lista con los registros del recurso. Presente en toda respuesta con `succeeded: true` (un GET sin resultados devuelve `[]`). En las respuestas de error con `succeeded: false` la clave puede **no emitirse**; consulte `succeeded` antes de leerla. |
| `PagingError` | Detalle de error de paginación; `null` cuando la paginación es válida. |
| `OrderError` | Detalle de error específico de órdenes (ver [GET Order](#recorder)); `null` en el resto de los casos. |
| `succeeded` | `true` si la operación fue exitosa; es el campo a consultar para conocer el resultado de la operación. |
| `Message` | Mensaje de la operación (string único). `null` cuando no hay mensaje; en un GET sin resultados contiene `"No se encontraron valores"`. |

Un GET sin resultados responde **200** con la lista vacía:

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": "No se encontraron valores"
}
```

> **Importante — fin de la paginación:** la última página se detecta con `Paging.MoreData: false` (o `Data` vacío). Si migra desde la API de Tango Tiendas, el terminador del bucle de sincronización cambió: consúltelo en [Migración](#migracion).

**Parámetros de paginación** (comunes a todos los GET):

| Parámetro | Obligatorio | Descripción |
| --------- | ----------- | ----------- |
| `PageNumber` | No | Página solicitada. Por defecto **1**. |
| `PageSize` | No | Cantidad de ítems por página. Por defecto **50**, máximo **5000**. |

Ejemplo:

```
https://{llave}.connect.axoft.com/api/eCommerce/Currency?PageNumber=1&PageSize=500
```

Si `PageNumber` o `PageSize` son menores o iguales a 0, la respuesta reemplaza `Paging`/`Data` por `PagingError`:

```json
{
  "Message": null,
  "OrderError": null,
  "PagingError": {
    "ErrorNumber": "Invalid page number",
    "ErrorSize": "Invalid page size"
  },
  "succeeded": false
}
```

<a name="status"></a>

### Códigos de estado HTTP

| Situación | HTTP | Cómo se informa |
| --------- | ---- | --------------- |
| GET/POST exitoso | **200** | `succeeded: true` |
| GET sin resultados | **200** | `succeeded: true`, `Message: "No se encontraron valores"` |
| Datos inválidos o con formato incorrecto (GET o POST) | **400** | Cuerpo JSON con el detalle por campo en la propiedad `errors` (ver [ejemplo](#ejemplo400)). Incluye un tipo inválido en la query de un GET, p. ej. `PageSize=abc` (no numérico) |
| Falta un header de autenticación (o viene vacío), o el token es inválido | **401** | Sin cuerpo (respuesta vacía); el código de estado es el único indicador |
| Ruta inexistente bajo `Api/eCommerce` | **404** | `Message: "No HTTP resource was found that matches the request URI '...'."` (única clave del cuerpo) |
| Método HTTP no soportado por el recurso (por ejemplo, DELETE a cualquier recurso, o GET a `/Dummy`, que es POST) | **405** | Sin cuerpo (respuesta vacía); el código de estado es el único indicador |

<a name="ejemplo400"></a>

Ejemplo del cuerpo de un **400** por datos inválidos (orden enviada sin `Customer.Email`):

```json
{
  "errors": {
    "Customer.Email": [
      "The Email field is required."
    ]
  }
}
```

> **Alcance del 404:** el cuerpo JSON del 404 se devuelve sólo para rutas bajo el prefijo exacto `Api/eCommerce`.

> **Dónde leer el detalle del error de negocio:** el mensaje funcional viene en `Message` (string único, p. ej. `"ApertureValidationException: Order total doesn't add up."`). Para los errores de órdenes puntuales, vea `OrderError`.

<a name="ordenes"></a>

## Recepción de órdenes

[<sub>Volver al índice</sub>](#inicio)

**URL del servicio para enviar una orden (POST):**

```
https://{llave}.connect.axoft.com/api/eCommerce/Order?Cuenta={NombreCuenta}
```

El parámetro **`Cuenta`** es **obligatorio** y corresponde al nombre de la [Cuenta de Integración API](#requisitos).

La orden se envía en el cuerpo en formato JSON. Consulte:

- [Notificaciones (webhooks)](#notificaciones)
- [Órdenes por lote](#lote)
- [Preguntas frecuentes](#faqs)
- [Novedades del JSON de la orden](#novedades)
- [Datos del JSON](#djson)
- [Ejemplos de JSON](#ejemplos)

Al reenviar una orden con un `OrderID` ya existente, se interpreta como **modificación** (idempotencia por `OrderID`). La orden sólo puede modificarse mientras esté en estado **RECIBIDA** (o EN PROCESO/FINALIZADA/FINALIZADA* si se trata de una cancelación y únicamente actualiza el estado a CANCELADA). El `CustomerID` no puede cambiar entre modificaciones.

La respuesta usa la forma `Status`/`Message`/`Data`/`isOk` (la misma que devuelve [`Dummy`](#dummy)), más los campos `succeeded` y `OrderError`. En caso de éxito (`Status: 0`, `isOk: true`; `Data` viene siempre y es `null` en la orden individual):

```json
{
  "Status": 0,
  "Message": "Order ORD-1 in process",
  "Data": null,
  "isOk": true,
  "OrderError": null,
  "succeeded": true
}
```

Ante un error de negocio, la respuesta indica `Status: 1`, `isOk: false`, `succeeded: false` y el texto real de la validación en `Message`. Si el nombre enviado en `Cuenta` no corresponde a ninguna [Cuenta de Integración API](#requisitos), la respuesta usa esa misma forma con `Message: "<cuenta> is not registered."`. Ante un error de formato o un campo obligatorio ausente (por ejemplo, falta `Customer.Email`), la respuesta es **400**.

<a name="notificaciones"></a>

### Notificaciones (webhooks)

[<sub>Volver al índice</sub>](#inicio)

Las notificaciones se habilitan durante la [puesta en marcha](#requisitos) de la Cuenta de Integración API, indicando una **URL de notificaciones** y los eventos a recibir. El receptor debe exponer un endpoint **POST** de una web API accesible por **HTTPS** (TLS 1.2 o superior).

Se notifican los siguientes eventos:

| Evento | Tópico | Cuándo se envía |
| ------ | ------ | --------------- |
| Orden **observada** | `OrderObserved` | La orden fue recibida pero quedó observada (por ejemplo, una lista de precios inexistente). |
| Orden **procesada** | `OrderProcessed` | Se generó el pedido a partir de la orden. |
| Orden **rechazada** | `OrderRejected` | La orden fue rechazada. |
| Orden **facturada** | `OrderBilled` | Se facturó el pedido generado. |
| Orden **cancelada** | `OrderCancelled` | La orden fue cancelada. |
| **Factura disponible** | `InvoiceFile` | Hay un comprobante electrónico en PDF disponible. |
| Actualización de **precios** | `PriceProductUpdate` | Cambió el precio de un artículo. |
| Actualización de **saldos de stock** | `StockProductUpdate` | Cambió el saldo de stock de un artículo (incluye saldos centralizados). |

El cuerpo de la notificación se envía como JSON (`Content-Type: application/json`) con los datos de la orden o novedad notificada.

**Formato de JSON de notificación:**

El cuerpo de cada notificación es un objeto JSON con tres propiedades: `Topic` (el tópico del evento, ver tabla anterior), `Resource` (el identificador del recurso afectado) y `Message` (un texto descriptivo, que puede venir vacío). Ejemplos según el tipo de evento:

```json
{
  "Topic": "OrderObserved",
  "Resource": "ORD-1",
  "Message": "Lista de precios inexistente"
}
```

```json
{
  "Topic": "InvoiceFile",
  "Resource": "ORD-1",
  "Message": ""
}
```

```json
{
  "Topic": "PriceProductUpdate",
  "Resource": "1",
  "Message": "Actualización de Precio"
}
```

```json
{
  "Topic": "StockProductUpdate",
  "Resource": "1",
  "Message": "Actualización de Saldo"
}
```

> **Importante — sensibilidad a mayúsculas:** los nombres de las propiedades (`Topic`, `Resource`, `Message`) y los valores de `Topic` distinguen mayúsculas de minúsculas; respételos tal como figuran.

**Aclaración — significado de `Resource` según el tópico:**

- En los tópicos de orden (`OrderObserved`, `OrderProcessed`, `OrderRejected`, `OrderBilled`, `OrderCancelled`, `InvoiceFile`), `Resource` es el identificador de la orden (`OrderID`).
- En `PriceProductUpdate`, `Resource` es el Id del registro de precio modificado.
- En `StockProductUpdate`, `Resource` es el Id del registro de saldo modificado.

<a name="lote"></a>

### Órdenes por lote

[<sub>Volver al índice</sub>](#inicio)

**URL del servicio para enviar órdenes en lote (POST):**

```
https://{llave}.connect.axoft.com/api/eCommerce/Order/Batch?Cuenta={NombreCuenta}
```

El cuerpo contiene una lista de órdenes (con el mismo formato que el [POST Order](#djson)):

```json
{
  "OrderBatch": [
    {
      "OrderID": "1",
      "OrderNumber": "1",
      "...": "..."
    },
    {
      "OrderID": "2",
      "OrderNumber": "2",
      "...": "..."
    }
  ]
}
```

El número **máximo de órdenes por lote es 25**. Un lote inválido se rechaza con `Status: 1`, `isOk: false`, `succeeded: false` y el `Message` correspondiente: `"OrderBatch must contain at least one order"` (lote vacío o cuerpo sin la clave `OrderBatch`), `"The number of orders exceeds the batch limit of 25"` (lote que supera el máximo permitido) o `"<cuenta> is not registered"` (la Cuenta de Integración API se valida **una sola vez, antes del lote**: si no existe, el lote entero se rechaza sin procesar ninguna orden). Superados esos controles, el lote se procesa **orden por orden**: si una falla, el resto continúa (`Status: 0` e `isOk: true`, y el resultado por orden viaja en `Data`):

```json
{
  "Status": 0,
  "Message": "batch processed",
  "isOk": true,
  "OrderError": null,
  "succeeded": true,
  "Data": [
    {
      "OrderID": "1,3",
      "Processed": true,
      "ValidationException": null
    },
    {
      "OrderID": "2",
      "Processed": false,
      "ValidationException": "ApertureValidationException: ..."
    }
  ]
}
```

- Las órdenes procesadas correctamente se consolidan en **un único ítem** con `Processed: true` y los `OrderID` separados por coma. Ese ítem consolidado es **siempre el primer elemento** de `Data`, incluso cuando ninguna orden se procesa (en ese caso viaja con `OrderID: ""`).
- Cada orden con error genera su propio ítem con `Processed: false` y el detalle en `ValidationException`.

<a name="faqs"></a>

### Preguntas frecuentes

[<sub>Volver al índice</sub>](#inicio)

- **¿Cómo autentico las llamadas?**

  Con los headers `ApiAuthorization` (Token de desarrollador) y `Company` (IdEmpresa) en **todas** las llamadas. Para los POST de órdenes, además el parámetro `Cuenta` en la URL. Ver [Credenciales](#credenciales).

- **¿Dónde se envían los datos de facturación para las órdenes?**

  El nombre del receptor se toma de la Razón Social (`BusinessName`) informada en la orden; si no se informa, se toma de los campos que corresponden al apellido y nombre (`LastName` / `FirstName`). Cuando la orden trae el número de CUIL / CUIT o DNI, se consulta AFIP para completar los datos fiscales del receptor (condición de IVA, tipo y número de documento); la razón social registrada en AFIP sólo se utiliza cuando en la orden no se informó ningún nombre.

<a name="novedades"></a>

### Novedades del JSON de la orden

[<sub>Volver al índice</sub>](#inicio)

#### Doble unidad de medida (DUM)

Un artículo con doble unidad de medida maneja dos unidades de stock:

- **Stock 1 (precios y costos):** unidad en la que se expresan los precios, los costos y los saldos.
- **Stock 2:** segunda unidad en la que también se expresa el saldo.
- **UM de control de stock:** determina cuál de las dos unidades controla la disponibilidad y compromete el stock.
- **Equivalencia:** cuántas unidades de stock 1 equivalen a una unidad de stock 2.

> Ejemplo: se venden hormas de queso con Unidad de stock 1 = Kilos, Unidad de stock 2 = Horma y equivalencia = 2 (una horma equivale a 2 kilos).

Para un artículo con doble unidad de medida, el campo `SelectMeasureUnit` de cada ítem indica en qué unidad se expresan la cantidad y el precio:

| Valor | Unidad |
| ----- | ------ |
| `V` | Ventas (valor por defecto si no se informa). |
| `P` | Stock 1 (precios y costos). |
| `S` | Stock 2. |

Para un artículo **simple** se puede indicar `P` (Stock 1). El campo `MeasureCode` indica el código de medida con el que se generará el pedido.

<details>
<summary>Ejemplo: DUM informando unidad de Ventas</summary>

Artículo con Unidad de stock 1 = KILOGRAMOS, Unidad de stock 2 = UNI, equivalencia = 3 kilos por unidad.

```json
{
  "OrderItems": [
    {
      "ProductCode": "1000",
      "SKUCode": "ART_DOBLEUNIDAD",
      "Description": "Artículo de doble unidad de medida",
      "Quantity": 1.0,
      "UnitPrice": 30.0,
      "MeasureCode": "UNI",
      "SelectMeasureUnit": "V"
    }
  ]
}
```

En este caso se informa la unidad de **Ventas** (`SelectMeasureUnit: "V"`), por lo que `UnitPrice` corresponde al precio de venta; al generar el pedido se expresará en la unidad de stock 1.
</details>

<details>
<summary>Ejemplo: DUM informando unidad de Stock 1</summary>

Mismo artículo: Unidad de stock 1 = KILOGRAMOS, Unidad de stock 2 = UNI, equivalencia = 3 kilos por unidad.

```json
{
  "OrderItems": [
    {
      "ProductCode": "1000",
      "SKUCode": "ART_DOBLEUNIDAD",
      "Description": "Artículo de doble unidad de medida",
      "Quantity": 1.0,
      "UnitPrice": 10.0,
      "MeasureCode": "KILOGRAMOS",
      "SelectMeasureUnit": "P"
    }
  ]
}
```

En este caso se informa la unidad de **Stock 1** (`SelectMeasureUnit: "P"`), por lo que `UnitPrice` corresponde al precio por unidad de stock 1 (kilogramo).
</details>

#### Condición de venta, transporte y pagos

- **[Condición de venta](#recsalecondition)** (`SaleConditionCode`): si es distinta de *Contado*, es posible que al valor de la factura se le apliquen cargos propios de la condición (por ejemplo, intereses a 30/60/90 días).
- **[Transporte](#rectransport)**: si la condición de venta es *Contado*, se valida que el transporte informado **no tenga recargo** (`SurchargePercentage = 0`).
- **Pagos**: sólo pueden informarse `CashPayments` / `Payments` cuando la condición de venta es *Contado* y la orden **no** fue acordada con el vendedor. Si la orden fue acordada con el vendedor (`AgreedWithSeller: true`), la condición de venta debe ser *Contado* y no debe informarse ningún pago.

<a name="djson"></a>

### Datos del JSON

[<sub>Volver al índice</sub>](#inicio)

Los importes numéricos se expresan con hasta 2 decimales, usando el punto como separador. Informar más de 2 decimales no hace que la orden se rechace: los importes de los pagos (`Payments.Total` y `CashPayments.PaymentTotal`) se redondean a 2 decimales, y todos los demás datos se registran con los decimales informados. El criterio de redondeo se describe en [Generalidades](#generalidades).

**Tópico principal** (obligatorio)

| Campo | Requerido | Tipo | Descripción / Validación |
| ----- | --------- | ---- | ------------------------ |
| `OrderID` | Sí | Alfanumérico (≤200) | Identificador único de la orden. No puede ser vacío. Reenviar el mismo `OrderID` modifica la orden. |
| `OrderNumber` | Sí | Alfanumérico (≤200) | Número con el que se identifica la orden. No puede ser vacío. |
| `Date` | Sí | Datetime | Fecha de la orden. No puede ser anterior a 30 días de la fecha actual ni a la fecha de inicio configurada en la cuenta. |
| `Total` | Sí | Numérico (≥0) | Importe total de la orden. |
| `TotalDiscount` | No | Numérico (≥0) | Descuento total de la operación. |
| `PaidTotal` | No | Numérico (≥0) | Importe total pagado. |
| `FinancialSurcharge` | No | Numérico (≥0) | Recargo financiero. |
| `WarehouseCode` | No | Alfanumérico (≤10) | Código de [depósito](#recwarehouse). |
| `SellerCode` | No | Alfanumérico (≤10) | Código de [vendedor](#recseller). |
| `TransportCode` | No | Alfanumérico (≤10) | Código de [transporte](#rectransport). |
| `SaleConditionCode` | No | Entero (0–99) | [Condición de venta](#recsalecondition). |
| `PriceListNumber` | No | Entero | Número de [lista de precios](#recpricelist). |
| `InvoiceCounterfoil` | No | Entero (0–9999) | [Talonario](#reccounterfoil) de facturación. |
| `OrderCounterfoil` | No | Entero (0–9999) | [Talonario](#reccounterfoil) de pedido. |
| `AgreedWithSeller` | No | Lógico | Indica si el pago se acuerda con el vendedor. |
| `ValidateTotalWithItems` | No | Lógico | Si es `true`, valida `Total` según la [fórmula](#formulas). |
| `ValidateTotalWithPaidTotal` | No | Lógico | Si es `true`, valida además que `PaidTotal` no exceda `Total` (ver [fórmulas](#formulas)). |
| `Comment` | No | Alfanumérico (≤280) | Comentario del comprador. |
| `CancelOrder` | No | Lógico | Indica que la orden está cancelada. |
| `CancelReason` | No | Alfanumérico | Motivo de cancelación (sólo si `CancelOrder = true`). |
| `CancelDate` | Condicional | Datetime | Requerido si `CancelOrder = true`; no puede ser anterior a `Date`. |
| `Customer` | Sí | Tópico | Datos del cliente. |
| `OrderItems` | Sí | Lista | Al menos un ítem. |
| `Shipping` | No | Tópico | Datos de envío. |
| `CashPayments` | No | Lista | Pagos en efectivo/transferencia/pasarelas. |
| `Payments` | No | Lista | Pagos con tarjeta. |

<a name="formulas"></a>
**Fórmulas de validación**

```
Total     = ∑[ (UnitPrice − (UnitPrice / 100 × DiscountPercentage)) × Quantity ]
            + Shipping.ShippingCost + FinancialSurcharge − TotalDiscount

PaidTotal = ∑(Payments.Total) + ∑(CashPayments.PaymentTotal)
```

`Total` se valida contra su fórmula sólo si `ValidateTotalWithItems = true`. La igualdad de `PaidTotal` con la suma de los pagos (`∑Payments.Total + ∑CashPayments.PaymentTotal`) se valida **siempre**; en particular, informar un `PaidTotal` distinto de 0 sin detallar pagos hace que la orden se rechace. `ValidateTotalWithPaidTotal` agrega además el control de que `PaidTotal` no supere `Total`.

**Tópico `Customer`** (obligatorio)

| Campo | Requerido | Tipo | Descripción |
| ----- | --------- | ---- | ----------- |
| `CustomerID` | Sí | Numérico (>0) | Identificador del cliente. No puede cambiar en modificaciones. |
| `Code` | No | Alfanumérico | Código del cliente en Tango. |
| `DocumentType` | Sí | Alfanumérico | Código de tipo de documento. Ver [Tipo de Documento](#tipodoc). |
| `DocumentNumber` | No | Alfanumérico | Número de documento sin símbolos. |
| `IVACategoryCode` | Sí | Alfanumérico | Categoría de IVA. Ver [Condición Fiscal](#cfiscal). |
| `User` | Sí | Alfanumérico (≤200) | Usuario de la tienda. |
| `Email` | Sí | Alfanumérico (≤255) | Correo electrónico. |
| `ProvinceCode` | Sí | Alfanumérico (≤4) | Código AFIP de la provincia. Ver [Provincias](#provincias). |
| `FirstName` | No | Alfanumérico (≤200) | Nombre. |
| `LastName` | No | Alfanumérico | Apellido. |
| `BusinessName` | No | Alfanumérico | Razón social. |
| `Street` / `HouseNumber` / `Floor` / `Apartment` / `City` / `PostalCode` | No | Alfanumérico | Domicilio (el sistema lo compone en un único domicilio). |
| `Address` | No | Alfanumérico | Domicilio ya compuesto. Si se informa, se usa tal cual como domicilio del comprador en lugar de componerlo desde `Street`/`HouseNumber`/etc. |
| `PhoneNumber1` / `PhoneNumber2` / `MobilePhoneNumber` | No | Alfanumérico | Teléfonos. |
| `BusinessAddress` | No | Alfanumérico | Domicilio comercial. |
| `NumberListPrice` | No | Entero | Número de [lista de precios](#recpricelist) del cliente. |
| `PayInternalTax` | No | Lógico | Indica si se liquidan impuestos internos al comprador. |

<a name="clientehabitual"></a>
**Cómo se relaciona con el cliente habitual**

Si en el proceso **Parámetros de eCommerce** está activada la opción **Busca cliente habitual**, la orden se asocia a un cliente habitual de Tango según el primer criterio que arroje coincidencia, en este orden:

1. **ABM Clientes — solapa principal:** código de cliente (`Code`), luego tipo y número de documento, luego correo electrónico.
2. **ABM Clientes — solapa contactos** (si la solapa principal no arrojó coincidencias): código de cliente, tipo y número de documento, correo electrónico y, por último, usuario de la tienda (`User`).

Para la comparación de documento, CUIT y CUIL se aceptan indistintamente. Si ningún criterio coincide, el pedido se genera con el cliente ocasional.

**Tópico `OrderItems`** (obligatorio, al menos un registro)

| Campo | Requerido | Tipo | Descripción |
| ----- | --------- | ---- | ----------- |
| `ProductCode` | Sí | Alfanumérico (≤200) | Código del artículo de la publicación. |
| `SKUCode` | No | Alfanumérico (≤40) | Código de [artículo](#recproduct), sinónimo o código de barras de Tango. Establece/actualiza la relación de la publicación (ProductCode) con el artículo de Tango. |
| `VariantCode` | No | Alfanumérico (≤200) | Código de la combinación ([escala](#recscale)). |
| `Description` | Sí | Alfanumérico (≤400) | Descripción del artículo. |
| `VariantDescription` | No | Alfanumérico (≤400) | Descripción de la variante. |
| `Quantity` | Sí | Numérico (>0) | Cantidad. |
| `UnitPrice` | Sí | Numérico (>0) | Precio unitario. |
| `DiscountPercentage` | No | Numérico (0–99,99) | Porcentaje de descuento. |
| `SelectMeasureUnit` | No | Alfanumérico (1) | Unidad de medida seleccionada: `V`, `P` o `S` (por defecto `V`). Ver [DUM](#novedades). |
| `MeasureCode` | No | Alfanumérico | Código de [medida](#recmeasure) con el que se genera el pedido. |
| `WarehouseCode` | No | Alfanumérico (≤10) | Código de [depósito](#recwarehouse) del renglón. Si se informa, el renglón del pedido se genera con ese depósito. |

> **Cálculo del precio por renglón:** `(UnitPrice − (UnitPrice / 100 × DiscountPercentage)) × Quantity`.

**Tópico `Shipping`** (opcional)

Si se informa este tópico, `ShippingID` pasa a ser obligatorio (numérico y mayor a 0) y `ProvinceCode` es obligatorio salvo que se informe `ShippingCode` (una dirección de entrega existente del cliente).

| Campo | Requerido | Tipo | Descripción |
| ----- | --------- | ---- | ----------- |
| `ShippingID` | Condicional | Numérico (>0) | Identificador del envío. Requerido si se informa `Shipping`. |
| `ShippingCode` | No | Alfanumérico (≤40) | Código de la dirección de entrega. |
| `ShippingCost` | No | Numérico | Costo de envío. |
| `Street` / `HouseNumber` / `Floor` / `Apartment` / `City` / `PostalCode` | No | Alfanumérico | Domicilio de entrega. |
| `ProvinceCode` | Condicional | Alfanumérico (≤4) | Código AFIP de la provincia. Requerido si se informa `Shipping` sin `ShippingCode`. |
| `PhoneNumber1` / `PhoneNumber2` | No | Alfanumérico (≤30) | Teléfonos. |
| `DeliversMonday` … `DeliversSunday` | No | Alfanumérico (1) | Días de entrega: `S` / `N` (por defecto `N`). |
| `DeliveryHours` | No | Alfanumérico (≤100) | Hora de entrega. |
| `DeliveryDate` | No | Datetime | Fecha de entrega. No puede ser anterior a `Date`. |

**Tópico `CashPayments`** (opcional) — pagos en efectivo/transferencia/pasarelas

| Campo | Requerido | Tipo | Descripción |
| ----- | --------- | ---- | ----------- |
| `PaymentID` | Sí | Numérico (>0, hasta 50 dígitos) | Identificador del pago. Único en toda la orden. |
| `PaymentMethod` | Sí | Alfanumérico (≤3) | Código de forma de cobro. Ver [Formas de Pago](#fpago). |
| `PaymentTotal` | Sí | Numérico (≥0) | Total del pago. Debe cuadrar con `PaidTotal` y la suma de pagos (ver [fórmulas](#formulas)). |

**Tópico `Payments`** (opcional) — pagos con tarjeta

| Campo | Requerido | Tipo | Descripción |
| ----- | --------- | ---- | ----------- |
| `PaymentId` | Sí | Numérico (>0) | Identificador del pago. Único en toda la orden. |
| `TransactionDate` | Sí | Datetime | Fecha del pago. No anterior a `Date`. |
| `Installments` | Sí | Numérico (≥1) | Cantidad de cuotas. |
| `InstallmentAmount` | Sí | Numérico (>0) | Importe por cuota. |
| `Total` | Sí | Numérico (>0) | Total del pago. |
| `CardCode` | Sí | Alfanumérico | Código de la tarjeta de crédito (módulo Tesorería). |
| `CardPlanCode` | Sí | Alfanumérico | Plan de la tarjeta. |
| `VoucherNo` | Sí | Numérico (0–99999999) | Número de cupón. |
| `AuthorizationCode` | No | Alfanumérico | Código de autorización. |
| `TransactionNumber` | No | Alfanumérico | Número de transacción. |
| `CardPromotionCode` | No | Alfanumérico | Código de promoción. |

<a name="tablas"></a>

### Tablas de referencia

[<sub>Volver al índice</sub>](#inicio)

<a name="tipodoc"></a>

<details>
<summary><b>Tipo de Documento</b></summary>

| Código | Descripción |
| ------ | ----------- |
| 80 | CUIT |
| 86 | CUIL |
| 91 | CI extranjera |
| 94 | Pasaporte |
| 96 | DNI |
| 99 | Sin identificar / venta global diaria |

Son los tipos de documento habilitados para el ingreso de comprobantes en Tango (`IVA_TIPO_DOCUMENTO.HABILITADO_PARA_INGRESO = 'S'`). Un código no habilitado (por ejemplo, 89 - LE) se rechaza con `Incorrect value - Field: DocumentType.`
</details>

<a name="provincias"></a>

<details>
<summary><b>Provincias</b></summary>

| Código | Descripción |
| ------ | ----------- |
| 0 | CIUDAD AUTONOMA BUENOS AIRES |
| 1 | BUENOS AIRES |
| 2 | CATAMARCA |
| 3 | CORDOBA |
| 4 | CORRIENTES |
| 5 | ENTRE RIOS |
| 6 | JUJUY |
| 7 | MENDOZA |
| 8 | LA RIOJA |
| 9 | SALTA |
| 10 | SAN JUAN |
| 11 | SAN LUIS |
| 12 | SANTA FE |
| 13 | SANTIAGO DEL ESTERO |
| 14 | TUCUMAN |
| 16 | CHACO |
| 17 | CHUBUT |
| 18 | FORMOSA |
| 19 | MISIONES |
| 20 | NEUQUEN |
| 21 | LA PAMPA |
| 22 | RIO NEGRO |
| 23 | SANTA CRUZ |
| 24 | TIERRA DEL FUEGO |
</details>

<a name="cfiscal"></a>

<details>
<summary><b>Condición Fiscal</b></summary>

| Código | Descripción |
| ------ | ----------- |
| CF | CONSUMIDOR FINAL |
| EX | EXENTO |
| EXE | EXENTO OPERACIÓN EXPORTACIÓN |
| INR | NO RESPONSABLE |
| RI | RESPONSABLE INSCRIPTO |
| RS | RESPONSABLE MONOTRIBUTISTA |
| RSS | RESPONSABLE MONOTRIBUTISTA SOCIAL |
| PCE | PEQUEÑO CONTRIBUYENTE EVENTUAL |
| PCS | PEQUEÑO CONTRIBUYENTE EVENTUAL SOCIAL |
| SNC | SUJETO NO CATEGORIZADO |
| INA | IVA NO ALCANZADO |
</details>

<a name="fpago"></a>

<details>
<summary><b>Formas de Pago</b></summary>

Los códigos disponibles dependen de la configuración de formas de cobro de Tango:

| Código | Descripción |
| ------ | ----------- |
| A01 … A10 | Formas de cobro Web API 01 a 10 |
| MPA | MercadoPago Argentina |
| PPA | PayPal Argentina |
| PUA | PayU Argentina |
| TPA | Todo Pago Argentina |
</details>

<a name="ejemplos"></a>

### Ejemplos de JSON

[<sub>Volver al índice</sub>](#inicio)

<details>
<summary><b>Orden con condición de venta Contado</b></summary>

```json
{
  "Date": "2026-02-14T00:00:00",
  "Total": 8523.0,
  "TotalDiscount": 77.0,
  "PaidTotal": 8523.0,
  "FinancialSurcharge": 200.0,
  "WarehouseCode": "2",
  "SellerCode": "2",
  "TransportCode": "01",
  "SaleConditionCode": 1,
  "InvoiceCounterfoil": 1,
  "OrderID": "75906",
  "OrderNumber": "75906",
  "ValidateTotalWithPaidTotal": true,
  "Customer": {
    "CustomerID": 227060905,
    "DocumentType": "80",
    "DocumentNumber": "11111111111",
    "IVACategoryCode": "CF",
    "User": "ADMIN",
    "Email": "api@axoft.com",
    "FirstName": "Carlos",
    "LastName": "Perez",
    "BusinessName": "Empresa",
    "Street": "Cerrito",
    "HouseNumber": "1186",
    "City": "CABA",
    "ProvinceCode": "0",
    "PostalCode": "1122"
  },
  "OrderItems": [
    {
      "ProductCode": "203",
      "SKUCode": "0100200659",
      "Description": "LAVARROPAS AUTOM. MOD.BLUE",
      "Quantity": 1.0,
      "UnitPrice": 7700.0,
      "DiscountPercentage": 0.0
    }
  ],
  "Shipping": {
    "ShippingID": 71906,
    "Street": "9 de Julio",
    "HouseNumber": "1186",
    "City": "CABA",
    "ProvinceCode": "0",
    "PostalCode": "1122",
    "ShippingCost": 400.0,
    "DeliversMonday": "S"
  },
  "CashPayments": [
    {
      "PaymentID": 38566912,
      "PaymentMethod": "A02",
      "PaymentTotal": 423.0
    }
  ],
  "Payments": [
    {
      "PaymentId": 38566913,
      "TransactionDate": "2026-02-14T00:00:00",
      "Installments": 1,
      "InstallmentAmount": 8100.0,
      "Total": 8100.0,
      "CardCode": "DI",
      "CardPlanCode": "1",
      "VoucherNo": 48
    }
  ]
}
```
</details>

<details>
<summary><b>Orden con condición de venta Cuenta corriente</b></summary>

```json
{
  "Date": "2026-05-28T00:00:00",
  "Total": 8400.0,
  "PaidTotal": 0.0,
  "WarehouseCode": "2",
  "SellerCode": "2",
  "TransportCode": "02",
  "SaleConditionCode": 3,
  "InvoiceCounterfoil": 2,
  "OrderID": "75907",
  "OrderNumber": "75907",
  "ValidateTotalWithPaidTotal": false,
  "Customer": {
    "CustomerID": 227060905,
    "Code": "010010",
    "DocumentType": "80",
    "DocumentNumber": "11111111111",
    "IVACategoryCode": "CF",
    "User": "ADMIN",
    "Email": "api@axoft.com",
    "FirstName": "Carlos",
    "LastName": "Perez",
    "ProvinceCode": "0"
  },
  "OrderItems": [
    {
      "ProductCode": "203",
      "SKUCode": "0100200659",
      "Description": "LAVARROPAS AUTOM. MOD.BLUE",
      "Quantity": 1.0,
      "UnitPrice": 8400.0
    }
  ],
  "CashPayments": [],
  "Payments": []
}
```
</details>

<details>
<summary><b>Orden con código de dirección de entrega</b></summary>

```json
{
  "Date": "2026-05-28T00:00:00",
  "Total": 8800.0,
  "PaidTotal": 0.0,
  "WarehouseCode": "2",
  "SellerCode": "2",
  "TransportCode": "02",
  "SaleConditionCode": 3,
  "OrderID": "75908",
  "OrderNumber": "75908",
  "ValidateTotalWithPaidTotal": false,
  "Customer": {
    "CustomerID": 227060905,
    "Code": "010010",
    "DocumentType": "80",
    "DocumentNumber": "11111111111",
    "IVACategoryCode": "CF",
    "User": "ADMIN",
    "Email": "api@axoft.com",
    "ProvinceCode": "0"
  },
  "OrderItems": [
    {
      "ProductCode": "203",
      "SKUCode": "0100200659",
      "Description": "LAVARROPAS AUTOM. MOD.BLUE",
      "Quantity": 1.0,
      "UnitPrice": 8400.0
    }
  ],
  "Shipping": {
    "ShippingID": 71906,
    "ShippingCode": "PRINCIPAL",
    "ShippingCost": 400.0
  },
  "CashPayments": [],
  "Payments": []
}
```

En este caso se informa `Shipping.ShippingCode` y no hace falta completar el resto del domicilio: si el código existe entre las direcciones de entrega del cliente habitual, el pedido se genera con esa dirección; si no existe, se utiliza la dirección de entrega habitual del cliente. Las direcciones de entrega se consultan con [`GET /Customer`](#reccustomer).
</details>

<details>
<summary><b>Orden con cliente habitual y lista de precios</b></summary>

```json
{
  "Date": "2026-02-14T00:00:00",
  "Total": 7700.0,
  "PaidTotal": 7700.0,
  "WarehouseCode": "2",
  "SellerCode": "2",
  "TransportCode": "01",
  "SaleConditionCode": 1,
  "PriceListNumber": 2,
  "OrderID": "75909",
  "OrderNumber": "75909",
  "ValidateTotalWithPaidTotal": true,
  "Customer": {
    "CustomerID": 227060905,
    "Code": "010010",
    "DocumentType": "80",
    "DocumentNumber": "11111111111",
    "IVACategoryCode": "CF",
    "PayInternalTax": false,
    "User": "ADMIN",
    "Email": "api@axoft.com",
    "ProvinceCode": "0"
  },
  "OrderItems": [
    {
      "ProductCode": "203",
      "SKUCode": "0100200659",
      "Description": "LAVARROPAS AUTOM. MOD.BLUE",
      "Quantity": 1.0,
      "UnitPrice": 7700.0
    }
  ],
  "CashPayments": [
    {
      "PaymentID": 38566912,
      "PaymentMethod": "A02",
      "PaymentTotal": 7700.0
    }
  ],
  "Payments": []
}
```

En este caso se informa `Customer.Code` para enlazar la orden con el cliente habitual de Tango (ver [cómo se relaciona con el cliente habitual](#clientehabitual)) y `PriceListNumber`; el desglose impositivo del pedido se define según la configuración de esa lista de precios en Tango.
</details>

<a name="consulta"></a>

## Consulta de datos

[<sub>Volver al índice</sub>](#inicio)

Los recursos GET permiten consultar datos de **Tango Gestión** o **Tango Punto de Venta**, devolviendo el resultado paginado en el [formato de respuesta](#envelope) descripto arriba. Todos se consumen sobre la misma base; por ejemplo:

```
GET https://{llave}.connect.axoft.com/api/eCommerce/Product?PageNumber=1&PageSize=100
```

**Sincronización incremental:** varios recursos aceptan un parámetro de fecha (`UpdatedDate`, `DatePrice`, `LastUpdate`) para obtener sólo los registros **creados o modificados** desde esa fecha. En `Customer` y `Product`, la comparación contra la fecha de alta es a nivel **día**: un corte con hora incluye también los registros dados de alta ese mismo día (la fecha de alta se registra sin hora). Los registros **sin fecha de alta ni de modificación** (datos históricos de Tango) no se devuelven al filtrar por `UpdatedDate`, cualquiera sea el corte; para obtenerlos, consulte el recurso sin ese parámetro. Envíe la fecha **sin designador de zona** (sin `Z` ni desplazamiento): la API convierte a UTC los valores con zona y el corte deja de coincidir con la hora local registrada.

**Filtro `filter`:** todos los GET aceptan el parámetro `filter`. **Prefiera los filtros nombrados de cada recurso**; el operador de cada filtro se indica en la tabla de parámetros del recurso cuando no es la búsqueda exacta.

Su comportamiento depende del recurso:

- **Búsqueda parcial** (contiene la cadena indicada) en los recursos cuya clave es un código: Currency, Measure, Scale, Warehouse, ValueScale, Seller, Customer, Product, ArtPorSaldoStock, PriceByCustomer, StockGroupByProduct.
- **Parcial o exacta según `UseEqual`** en Stock y DiscountByCustomer: parcial por defecto, búsqueda exacta con `UseEqual=true`.
- **Búsqueda exacta** en los recursos cuya clave es numérica o un identificador: PriceList, Price, Store, Transport, SaleCondition, CounterFoil, los clasificadores (`*Folder`/`*Relation`/`*FolderClassifier`) y CurrencyExchangeRate.
- En **Order, Invoices y Publications** filtra con búsqueda parcial sobre el identificador de orden / código de artículo de la tienda.

> **Nota sobre stock centralizado:** si la empresa centraliza stock de varias sucursales y desea informar saldos centralizados, envíe `Centraliza=true` en [`Stock`](#recstock) / [`StockGroupByProduct`](#recstockgroup). De lo contrario, la consulta devuelve los saldos de la sucursal local (ver [Consideraciones](#consideraciones)).

<a name="recursos"></a>

### Recursos de consulta

[<sub>Volver al índice</sub>](#inicio)

- [Estado de órdenes (`Order`)](#recorder)
- [Comprobantes de facturación (`Invoices`)](#recinvoices)
- [Publicaciones (`Publications`)](#recpublications)
- [Sucursales (`Store`)](#recstore)
- [Depósitos (`Warehouse`)](#recwarehouse)
- [Unidades de medida (`Measure`)](#recmeasure)
- [Escalas (`Scale`)](#recscale) · [Valores de escala (`ValueScale`)](#recvaluescale)
- [Artículos (`Product`)](#recproduct) · [Artículos por saldo de stock (`ArtPorSaldoStock`)](#recartsaldo)
- [Saldos de stock (`Stock`)](#recstock) · [Saldos agrupados (`StockGroupByProduct`)](#recstockgroup)
- [Listas de precios (`PriceList`)](#recpricelist) · [Precios (`Price`)](#recprice)
- [Precios por cliente (`PriceByCustomer`)](#recpricebycustomer) · [Descuentos por cliente (`DiscountByCustomer`)](#recdiscount)
- [Clientes (`Customer`)](#reccustomer) · [Vendedores (`Seller`)](#recseller)
- [Transportes (`Transport`)](#rectransport) · [Condiciones de venta (`SaleCondition`)](#recsalecondition)
- [Monedas (`Currency`)](#reccurrency) · [Cotizaciones (`CurrencyExchangeRate`)](#reccurrencyrate)
- [Talonarios (`CounterFoil`)](#reccounterfoil)
- [Clasificador de artículos](#recproductsfolder): [`ProductsFolder`](#recproductsfolder) · [`ProductsRelation`](#recproductsrelation) · [`ProductsFolderClassifier`](#recproductsclassifier)
- [Clasificador de clientes](#reccustomersfolder): [`CustomersFolder`](#reccustomersfolder) · [`CustomersRelation`](#reccustomersrelation) · [`CustomersFolderClassifier`](#reccustomersclassifier)

---

<a name="recorder"></a>

#### Estado de órdenes — `GET /Order`

[<sub>Volver a recursos</sub>](#recursos)

Devuelve el estado de las órdenes recibidas. Si se superan **50 identificadores** por consulta, la consulta entera se rechaza (`succeeded: false`) y el detalle vuelve en `OrderError` (ver más abajo).

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `OrderId` | string | Filtra por identificador de orden. |
| `FromDate` / `ToDate` | datetime | Rango de fechas. |
| `Status` | string | Estado(s) de la orden. Admite varios valores separados por coma. |
| `IncludeInvoices` | bool | Incluye los datos de facturación. Por defecto **true**. |
| `Cuenta` | string | Cuenta de Integración API. |
| `filter` | string | Filtra por identificador de orden (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

Con `IncludeInvoices=true`, cada orden incluye los datos del comprobante:

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "OrderId": "75906",
      "Date": "2026-02-14T00:00:00",
      "Status": "FACTURADA",
      "IncludeInvoices": true,
      "InvoiceType": "FAC",
      "InvoiceNumber": "0001-00001234",
      "InvoiceFileUrl": "https://.../factura.pdf",
      "InvoiceFileExpiration": "2026-03-14T00:00:00"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```

Con `IncludeInvoices=false`, cada orden trae sólo `OrderId`, `Date`, `Status` e `IncludeInvoices`. `InvoiceFileExpiration` es `null` cuando la orden no tiene comprobante.

Si alguno de los `OrderId` consultados no existe, el detalle vuelve en `OrderError` con los identificadores afectados, junto con las órdenes encontradas (`succeeded` queda `true`: es un resultado parcial):

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [ /* órdenes encontradas */ ],
  "PagingError": null,
  "Message": null,
  "OrderError": {
    "Orders": "75910,75911",
    "Message": "Unexisting Order ID"
  },
  "succeeded": true
}
```

Si se supera el límite de **50** identificadores por consulta, la consulta entera se rechaza sin ejecutarse. La respuesta conserva el formato de respuesta paginado — `Paging` con la paginación pedida, `Data: []`, `PagingError: null` — con `succeeded: false` y el detalle en `OrderError`: los identificadores enviados en `Orders` y `Message: "The number of orders exceeds the limit of 50"`.
</details>

<a name="recinvoices"></a>

#### Comprobantes de facturación — `GET /Invoices`

[<sub>Volver a recursos</sub>](#recursos)

Devuelve la relación entre cada orden y el comprobante electrónico asociado al pedido facturado de esa orden, incluyendo la URL de descarga del archivo PDF. Sólo se incluyen comprobantes electrónicos.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `OrderId` | string | Filtra por identificador de orden. |
| `OrderNumber` | string | Filtra por número de orden. |
| `FromDate` / `ToDate` | datetime | Rango de fechas. |
| `Cuenta` | string | Cuenta de Integración API. |
| `filter` | string | Filtra por identificador de orden (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "OrderID": "75906",
      "OrderNumber": "75906",
      "OrderDate": "2026-02-14T00:00:00",
      "InvoiceType": "FAC",
      "InvoiceNumber": "0001-00001234",
      "InvoiceFileUrl": "https://.../factura.pdf",
      "InvoiceFileExpiration": "2026-03-14T00:00:00"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recpublications"></a>

#### Publicaciones — `GET /Publications`

[<sub>Volver a recursos</sub>](#recursos)

Relación entre el artículo de la tienda y el artículo de Tango.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `ProductCode` | string | Código de artículo de la tienda. |
| `SkuCode` | string | Código de artículo de Tango. |
| `VariantCode` | string | Código de la variante (escala). |
| `Cuenta` | string | Cuenta de Integración API. |
| `filter` | string | Filtra por código de artículo de la tienda (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "ProductCode": "203",
      "Description": "LAVARROPAS AUTOM. MOD.BLUE",
      "VariantCode": "",
      "VariantDescription": "",
      "SkuCode": "0100200659"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recstore"></a>

#### Sucursales — `GET /Store`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterStoreNumber` | string | Filtra por número de sucursal. |

_Recuerde:_ `ProvinceCode` es el código AFIP de la provincia (ver [Provincias](#provincias)).

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 1,
      "StoreNumber": 1,
      "Description": "CASA CENTRAL",
      "Street": "",
      "Number": "",
      "Floor": "",
      "Apartment": "",
      "Tower": "",
      "Block": "",
      "City": "",
      "PostalCode": "",
      "ProvinceCode": "0",
      "Email": "",
      "WebPage": "",
      "Contact": "PRUEBA",
      "PhoneNumber1": "(011)3333-3333",
      "PhoneNumber2": ""
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recwarehouse"></a>

#### Depósitos — `GET /Warehouse`

[<sub>Volver a recursos</sub>](#recursos)

Devuelve tanto las sucursales (`CentralizedStock: false`) como los depósitos de **stock centralizado** (`CentralizedStock: true`).

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de depósito. |

> El campo `Id` puede repetirse entre depósitos de distinto tipo; la clave estable es la combinación `Code` + `CentralizedStock`.

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 1,
      "Code": "1",
      "Description": "DEPOSITO CASA CENTRAL",
      "Disabled": false,
      "CentralizedStock": false
    },
    {
      "Id": 1,
      "Code": "10",
      "Description": "DEPOSITO CENTRALIZADO",
      "Disabled": false,
      "CentralizedStock": true
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recmeasure"></a>

#### Unidades de medida — `GET /Measure`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de medida. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 2,
      "Code": "UNI",
      "Description": "Unidades",
      "Initials": "UN",
      "Quantity": 0,
      "UnitType": "",
      "AfipCode": "07",
      "AfipEquivalence": 1.0,
      "TransportCode": "",
      "TransportEquivalence": 0.0
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recscale"></a>

#### Escalas — `GET /Scale`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de escala. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 1,
      "Code": "01",
      "DescriptionScale": "COLOR",
      "NumberScale": 1
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recvaluescale"></a>

#### Valores de escala — `GET /ValueScale`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de escala. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 1,
      "Code": "01",
      "CodeValue": "BL",
      "DescriptionValue": "BLANCO"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recproduct"></a>

#### Artículos — `GET /Product`

[<sub>Volver a recursos</sub>](#recursos)

Devuelve los artículos con su composición, comentarios y valores de escala. Sólo se muestran artículos que en Tango cumplan: perfil de Venta, Compra-Venta o Inhabilitado; tipo Simple, Fórmula o Kit fijo; y que no sean artículos Base.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterProductCode` | string | Filtra por código de artículo (búsqueda exacta). |
| `AlternativeCode` | string | Filtra por sinónimo (búsqueda parcial). |
| `BarCode` | string | Filtra por código de barras (búsqueda parcial). |
| `OnlyKit` | bool | Sólo artículos Kit. |
| `OnlyEnabled` | bool | Sólo artículos habilitados. |
| `UpdatedDate` | datetime | Sólo artículos creados o modificados desde esa fecha. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 100,
      "SKUCode": "KIT100",
      "Description": "KIT AUDIO COMPLETO",
      "AdditionalDescription": "",
      "AlternativeCode": "",
      "BarCode": "",
      "Commission": 6.0,
      "Discount": 0.0,
      "MeasureUnitCode": "UNI",
      "SecondMeasureUnitCode": "",
      "StockEquivalence": 0.0,
      "StockControlUnit": "P",
      "SalesMeasureUnitCode": "UNI",
      "SalesEquivalence": 1.0,
      "MaximumStock": 0.0,
      "MinimumStock": 0.0,
      "RestockPoint": 0.0,
      "Observations": "",
      "Kit": true,
      "KitValidityDateSince": null,
      "KitValidityDateUntil": null,
      "LastUpdateUtc": "2026-02-14T20:16:30.043",
      "UseScale": "N",
      "Scale1": "",
      "Scale2": "",
      "BaseArticle": "",
      "ScaleValue1": "",
      "ScaleValue2": "",
      "DescriptionScale1": "",
      "DescriptionScale2": "",
      "DescriptionValueScale1": "",
      "DescriptionValueScale2": "",
      "Disabled": false,
      "ProductComposition": [
        {
          "ComponentSKUCode": "0100100150",
          "Quantity": 1
        },
        {
          "ComponentSKUCode": "0100100151",
          "Quantity": 2
        }
      ],
      "ProductComments": [
        {
          "Line": 1,
          "Text": "Comentario para la impresión"
        }
      ]
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recartsaldo"></a>

#### Artículos por saldo de stock — `GET /ArtPorSaldoStock`

[<sub>Volver a recursos</sub>](#recursos)

Devuelve artículos (con la misma forma y las mismas restricciones de perfil y tipo que [`Product`](#recproduct)) filtrados por saldo de stock.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `WarehouseCode` | string | Código de depósito. |
| `StoreNumber` | string | Número de sucursal. |
| `MinStock` | decimal | Saldo de stock mínimo (mayor estricto). |
| `Centraliza` | bool | Incluir stock centralizado. |
| `filter` | string | Filtra por código de artículo (búsqueda parcial). |

La respuesta tiene la misma estructura que [`Product`](#recproduct).

<a name="recstock"></a>

#### Saldos de stock — `GET /Stock`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterProductCode` | string | Filtra por código de artículo (el operador depende de `UseEqual`: parcial por defecto). |
| `filter` | string | Filtra por código de artículo (filtro adicional; el operador depende de `UseEqual`). |
| `StoreNumber` | string | Número de sucursal (valor único; el operador depende de `UseEqual`). Para consultar varias sucursales a la vez, use [`StockGroupByProduct`](#recstockgroup). |
| `WarehouseCode` | string | Código de depósito (valor único; el operador depende de `UseEqual`). |
| `DiscountPendingOrders` | bool | Descuenta las órdenes pendientes. |
| `Id` | string | Filtra por identificador de saldo. |
| `UseEqual` | bool | Búsqueda exacta (`true`) o parcial (`false`, valor por defecto). Aplica a `FilterProductCode`, `StoreNumber`, `WarehouseCode` y `filter`. |
| `Centraliza` | bool | Incluir stock centralizado. |
| `UpdatedDate` | datetime | Sólo saldos actualizados desde esa fecha (hora local). Las filas adicionales de `DiscountPendingOrders` no tienen fecha: se devuelven con `LastUpdate` en `null`, también al filtrar. |
| `groupByProduct` | bool | Si es `true`, la consulta se resuelve como [`StockGroupByProduct`](#recstockgroup) (saldo agrupado por artículo). Por defecto **false**. |
| `stockGroupByProduct` | bool | Alias de `groupByProduct`: si es `true`, la consulta se resuelve como [`StockGroupByProduct`](#recstockgroup). Por defecto **false**. |

Los saldos sin movimientos desde la instalación de este cambio no tienen fecha de actualización: `LastUpdate` vuelve `null` al consultar sin `UpdatedDate` y el filtro los excluye. Lo mismo aplica en [`StockGroupByProduct`](#recstockgroup) cuando ninguna fila del artículo tiene fecha.

Ejemplo:

```
https://{llave}.connect.axoft.com/api/eCommerce/Stock?UpdatedDate=2026-02-14T20:16:30
```

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "SKUCode": "0100200659",
      "MeasureCode": "UNI",
      "SecondMeasureCode": "",
      "WarehouseCode": "1",
      "StoreNumber": 1,
      "Quantity": 120.0,
      "SecondQuantity": 0.0,
      "PendingQuantity": 5.0,
      "SecondPendingQuantity": 0.0,
      "EngagedQuantity": 2.0,
      "SecondEngagedQuantity": 0.0,
      "LastUpdate": "2026-02-14T20:16:30.043"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recstockgroup"></a>

#### Saldos de stock agrupados por artículo — `GET /StockGroupByProduct`

[<sub>Volver a recursos</sub>](#recursos)

Devuelve el saldo agrupado por artículo (sin discriminar sucursal ni depósito: `StoreNumber` vuelve `0` y `WarehouseCode` `null`). El campo `LastUpdate` del grupo es la mayor fecha de actualización de saldo entre las filas agrupadas que cumplen los demás filtros; `null` si ninguna tiene fecha.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterProductCode` | string | Filtra por código de artículo (búsqueda exacta). |
| `filter` | string | Filtra por código de artículo (búsqueda parcial). |
| `WarehouseCode` | string | Código(s) de depósito (lista separada por coma). |
| `StoreNumber` | string | Número(s) de sucursal (lista separada por coma). |
| `DiscountPendingOrders` | bool | Descuenta las órdenes pendientes. |
| `Centraliza` | bool | Incluir stock centralizado. |
| `UpdatedDate` | datetime | Sólo artículos con algún saldo actualizado desde esa fecha (hora local). Con `DiscountPendingOrders=true`, los artículos con comprometido de órdenes pendientes se devuelven aunque no cumplan la fecha. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "SKUCode": "0100200659",
      "MeasureCode": "UNI",
      "SecondMeasureCode": "",
      "StoreNumber": 0,
      "WarehouseCode": null,
      "Quantity": 340.0,
      "SecondQuantity": 0.0,
      "PendingQuantity": 5.0,
      "SecondPendingQuantity": 0.0,
      "EngagedQuantity": 2.0,
      "SecondEngagedQuantity": 0.0,
      "LastUpdate": "2026-02-14T20:16:30.043"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recpricelist"></a>

#### Listas de precios — `GET /PriceList`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterListNumber` | string | Filtra por número de lista. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 10,
      "PriceListNumber": 2,
      "Description": "LISTA MAYORISTA",
      "CommonCurrency": true,
      "IvaIncluded": true,
      "InternalTaxIncluded": false,
      "ValidityDateSince": null,
      "ValidityDateUntil": null,
      "Disabled": false,
      "Display": "2 - LISTA MAYORISTA"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recprice"></a>

#### Precios — `GET /Price`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterListNumber` | string | Filtra por número de lista. |
| `SkuCode` | string | Filtra por código de artículo (búsqueda parcial). |
| `Id` | int | Filtra por identificador. |
| `DatePrice` | datetime | Sólo precios actualizados desde esa fecha. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 5001,
      "PriceListNumber": 2,
      "SKUCode": "0100200659",
      "Price": 7700.0,
      "DatePrice": "2026-02-14T20:16:30.043",
      "ValidityDateSince": null,
      "ValidityDateUntil": null
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recpricebycustomer"></a>

#### Precios por cliente — `GET /PriceByCustomer`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCustomerCode` | string | Filtra por código de cliente. |
| `SkuCode` | string | Filtra por código de artículo (búsqueda parcial). |
| `PriceListNumber` | string | Filtra por número de lista (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "SKUCode": "0100200659",
      "CustomerCode": "010010",
      "Price": 7500.0,
      "PriceListNumber": 2
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recdiscount"></a>

#### Descuentos por cliente — `GET /DiscountByCustomer`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCustomerCode` | string | Filtra por código de cliente (el operador depende de `UseEqual`: parcial por defecto). |
| `filter` | string | Filtra por código de cliente (filtro adicional; el operador depende de `UseEqual`). |
| `UseEqual` | bool | Búsqueda exacta (`true`) o parcial (`false`, valor por defecto). Aplica a `FilterCustomerCode`, `SkuCode` y `filter`. |
| `SkuCode` | string | Filtra por código de artículo (el operador depende de `UseEqual`: parcial por defecto). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "SKUCode": "0100200659",
      "CustomerCode": "010010",
      "Discount": 10.0
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="reccustomer"></a>

#### Clientes — `GET /Customer`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de cliente. |
| `IvaCategoryCode` | string | Filtra por categoría de IVA. |
| `DocumentType` | string | Filtra por tipo de documento. |
| `DocumentNumber` | string | Filtra por número de documento. |
| `UpdatedDate` | datetime | Sólo clientes creados o modificados desde esa fecha. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Code": "010010",
      "BusinessName": "EMPRESA S.A.",
      "TradeName": "EMPRESA",
      "Address": "CERRITO 1186",
      "PostalCode": "1122",
      "City": "CABA",
      "ProvinceCode": "0",
      "TradeAddress": "",
      "PhoneNumbers": "011-3333-3333",
      "Email": "cliente@mail.com",
      "MobilePhoneNumber": "",
      "WebPage": "",
      "IvaCategoryCode": "RI",
      "DocumentType": "80",
      "DocumentNumber": "30111111118",
      "PriceListNumber": 2,
      "Discount": 0.0,
      "Observations": "",
      "DisabledDate": null,
      "UpdateDatetime": "2026-01-10T00:00:00",
      "LastUpdateUtc": "2026-02-14T00:00:00",
      "ShippingAddresses": [
        {
          "Code": "PRINCIPAL",
          "Address": "9 DE JULIO 1186",
          "ProvinceCode": "0",
          "City": "CABA",
          "PostalCode": "1122",
          "PhoneNumber1": "",
          "PhoneNumber2": "",
          "DefaultAddress": "S",
          "Enabled": "S",
          "DeliveryHours": "",
          "DeliversMonday": "S",
          "DeliversTuesday": "S",
          "DeliversWednesday": "S",
          "DeliversThursday": "S",
          "DeliversFriday": "S",
          "DeliversSaturday": "N",
          "DeliversSunday": "N"
        }
      ],
      "CustomerComments": [
        {
          "Line": 1,
          "Text": "Cliente preferencial"
        }
      ],
      "SellerCode": "2",
      "CreditQuota": 500000.0,
      "LocalAccountBalance": 12000.0,
      "ForeignAccountBalance": 0.0,
      "ForeignCurrencyClause": false,
      "CreditQuotaCurrencyCode": "PES",
      "SaleConditionCode": 3,
      "TransportCode": "01"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recseller"></a>

#### Vendedores — `GET /Seller`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de vendedor. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Code": "2",
      "Name": "JUAN PEREZ",
      "CommissionPercentage": 6.0,
      "Disabled": false
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="rectransport"></a>

#### Transportes — `GET /Transport`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de transporte. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 1,
      "Code": "01",
      "Name": "TRANSPORTE OESTE",
      "IVACategory": "RI",
      "Cuit": "30111111118",
      "SurchargePercentage": 0.0,
      "Address": "AV. SIEMPREVIVA 100",
      "PostalCode": "1122",
      "City": "CABA",
      "ProvinceCode": "0",
      "PhoneNumbers": "011-4444-4444",
      "Email": "",
      "WebPage": "",
      "Comments": ""
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recsalecondition"></a>

#### Condiciones de venta — `GET /SaleCondition`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de condición. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Code": 1,
      "Description": "CONTADO",
      "Cash": true,
      "GenerateAlternativeDate": false,
      "GenerateDebitLatePayment": false
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="reccurrency"></a>

#### Monedas — `GET /Currency`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de moneda. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "Id": 1,
      "CurrencyCode": "01",
      "Description": "PESOS",
      "Symbol": "$",
      "Type": "C",
      "RG1547Code": "PES",
      "AFIPCode": ""
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="reccurrencyrate"></a>

#### Cotizaciones de moneda extranjera — `GET /CurrencyExchangeRate`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterIdRenglon` | string | Filtra por identificador de renglón. |
| `LastUpdate` | datetime | Sólo cotizaciones desde esa fecha. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "IdRenglon": 1001,
      "Value": 1050.75,
      "DateTime": "2026-06-30T15:00:00"
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="reccounterfoil"></a>

#### Talonarios — `GET /CounterFoil`

[<sub>Volver a recursos</sub>](#recursos)

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterCode` | string | Filtra por código de talonario. |
| `Voucher` | string | Filtra por comprobante. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "CounterfoilCode": 1,
      "Description": "FACTURAS A",
      "CounterfoilType": "FAC",
      "Voucher": "FAC",
      "CounterfoilExpiration": null
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recproductsfolder"></a>

#### Clasificador de artículos — carpetas y contenido

[<sub>Volver a recursos</sub>](#recursos)

**`GET /ProductsFolder`** — artículos dentro de las carpetas del clasificador.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterIdFolder` | string | Filtra por identificador de carpeta (búsqueda exacta). |
| `SkuCode` | string | Filtra por código de artículo (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "SkuCode": "0100200659",
      "IdFolder": 12
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recproductsrelation"></a>

**`GET /ProductsRelation`** — relaciones del clasificador de artículos.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterIdFolder` | string | Filtra por identificador de carpeta (búsqueda exacta). |
| `SkuCode` | string | Filtra por código de artículo (búsqueda parcial). |
| `RelationName` | string | Filtra por nombre de relación (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "SkuCode": "0100200659",
      "IdFolder": 12,
      "RelationName": "RELACIONADOS",
      "ShowRelation": true
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="recproductsclassifier"></a>

**`GET /ProductsFolderClassifier`** — carpetas del clasificador de artículos.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterIdFolder` | string | Filtra por identificador de carpeta. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "IdFolder": 12,
      "Name": "ELECTRODOMÉSTICOS",
      "IdParent": null
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="reccustomersfolder"></a>

#### Clasificador de clientes — carpetas y contenido

[<sub>Volver a recursos</sub>](#recursos)

**`GET /CustomersFolder`** — clientes dentro de las carpetas del clasificador.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterId` | string | Filtra por identificador de carpeta (búsqueda exacta). |
| `CustomerCode` | string | Filtra por código de cliente (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "CustomerCode": "010010",
      "IdFolder": 5
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="reccustomersrelation"></a>

**`GET /CustomersRelation`** — relaciones del clasificador de clientes.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterId` | string | Filtra por identificador de carpeta (búsqueda exacta). |
| `CustomerCode` | string | Filtra por código de cliente (búsqueda parcial). |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "CustomerCode": "010010",
      "IdFolder": 5,
      "RelationName": "MAYORISTAS",
      "ShowRelation": true
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="reccustomersclassifier"></a>

**`GET /CustomersFolderClassifier`** — carpetas del clasificador de clientes.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `FilterIdFolder` | string | Filtra por identificador de carpeta. |

<details>
<summary>Respuesta</summary>

```json
{
  "Paging": {
    "PageNumber": 1,
    "PageSize": 50,
    "MoreData": false
  },
  "Data": [
    {
      "IdFolder": 5,
      "Name": "CLIENTES MAYORISTAS",
      "IdParent": null
    }
  ],
  "PagingError": null,
  "OrderError": null,
  "succeeded": true,
  "Message": null
}
```
</details>

<a name="migracion"></a>

## Migración desde la API de Tango Tiendas

[<sub>Volver al índice</sub>](#inicio)

Si hoy integra con la **API de Tango Tiendas** (`api/Aperture`) y migra a la API de eCommerce, tenga en cuenta los siguientes cambios de contrato.

<a name="migplataforma"></a>
**Plataforma y autenticación**

| Tema | Tango Tiendas | eCommerce |
| ---- | ------------- | --------- |
| Prefijo de ruta | `api/Aperture/*` | `Api/eCommerce/*` |
| Host | `https://tiendas.axoft.com` | `https://{llave}.connect.axoft.com` |
| Autenticación | header `AccessToken` | headers `ApiAuthorization` + `Company` |
| Selección de tienda en POST | embebida en el token | parámetro [`?Cuenta=`](#requisitos) |
| Prueba de autenticación | `POST Dummy` → *Valid AccessToken* | `POST Dummy` → *Valid ApiAuthorization* |

**Formato de respuesta y estados**

- Una cuenta de integración inexistente en los POST responde `"<cuenta> is not registered."` (en Tiendas: `"Invalid AccessToken"`).
- **Códigos de estado:** el resultado de la operación se consulta en el campo `succeeded` (ver [Códigos de estado HTTP](#status)). Cambios puntuales respecto de Tiendas:
  - Un GET sin resultados devuelve **200** con `Data: []` (en Tiendas: **400**); detecte el fin de la paginación con `Paging.MoreData: false` o `Data` vacío.
  - Un token inválido devuelve **401**.
  - Una ruta inexistente bajo el prefijo `Api/eCommerce` devuelve **404** con un cuerpo JSON que indica el error en `Message`, en cualquier verbo.
  - Los datos inválidos en el `POST Order` responden **400** con el detalle por campo en la propiedad `errors` (en Tiendas: 200 con `isOk: false`).
- En [`StockGroupByProduct`](#recstockgroup) y en los alias de agrupación de [`Stock`](#recstock), el comprometido de las órdenes pendientes (`DiscountPendingOrders=true`) se suma una sola vez por artículo. `EngagedQuantity` puede resultar menor que el informado por Tiendas en artículos con saldo en más de un depósito.

**Recursos nuevos**

- [`GET ArtPorSaldoStock`](#recartsaldo) y [`GET StockGroupByProduct`](#recstockgroup) son rutas nuevas de eCommerce; la capacidad ya existía en Tiendas (respectivamente, `POST DataBy/ArtPorSaldoStock` y `Stock` con `GroupByProduct=true`).
- `POST Dummy` **no** es un recurso nuevo: ya existía en Tiendas y sólo cambia el mensaje de la prueba de autenticación (ver la [tabla de plataforma](#migplataforma) más arriba).

**Parámetros**

- Los recursos GET agregan filtros nombrados propios (`FilterCode`, `FilterListNumber`, `FilterProductCode`, etc.), de búsqueda exacta — salvo en `Stock` y `DiscountByCustomer`, donde el operador depende de `UseEqual` (parcial por defecto).
- Los parámetros `sort` y `validatePageSize` de Tiendas no están disponibles en eCommerce; `lastUpdate` se conserva sólo en `CurrencyExchangeRate` (ver [el recurso](#reccurrencyrate)).
- El filtro por fecha de [`Stock`](#recstock) y [`StockGroupByProduct`](#recstockgroup) es `UpdatedDate`, que compara en hora local contra la fecha de actualización del saldo.

<a name="consideraciones"></a>

## Consideraciones

[<sub>Volver al índice</sub>](#inicio)

- **Consulte siempre `succeeded`** (y `Message` / `OrderError` para el detalle) para determinar el resultado de cada operación. Ver [Códigos de estado HTTP](#status).
- **Stock centralizado:** tenga en cuenta que si no envía `Centraliza=true` en [`Stock`](#recstock) / [`StockGroupByProduct`](#recstockgroup), la consulta devuelve los saldos de la sucursal local. Si la empresa centraliza el stock de varias sucursales y desea obtener los saldos centralizados, envíe `Centraliza=true`.
- Los límites operativos están documentados en la sección de cada recurso: [tamaño y fin de página](#envelope), [órdenes por lote](#lote), [identificadores por consulta en GET Order](#recorder).

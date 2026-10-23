# Análisis de la Droguería Vida Sana

## 1. Flujo integrado

1. **Detección del faltante.** En el dashboard, quien tiene el permiso `crear-compras` ve el indicador «Stock bajo (≤ 5)» y la lista «Sugerencia de reabastecimiento» con los productos de menor stock.
2. **Orden de compra.** El administrador (único rol del seeder con `crear-compras`) registra la orden con `PurchaseService::register`, validada por `StorePurchaseRequest`. Se genera el número `OC-xxxxx`, queda en estado `pendiente` y se guarda quién la creó (`user_id`). El inventario **no cambia**: pedir no es recibir.
3. **Recepción.** Cuando llega la mercancía, el regente o el auxiliar de bodega (roles con `recibir-compras`) usan la ruta `purchases/{purchase}/receive` y `PurchaseService::receive`. En cada entrega indican la cantidad que llegó por línea. La orden pasa a `parcial` o `recibida`.
4. **Inventario.** Dentro de una transacción (`DB::transaction`, con `lockForUpdate`), la recepción aumenta `received_quantity` de la línea y el `stock` del producto. El inventario por lote está en la tabla `lots` (producto, número de lote, vencimiento y cantidad); un mismo producto puede tener varios lotes con vencimientos distintos.
5. **Venta FEFO.** El cajero (permiso `crear-ventas`) registra la venta. `DispensingService::dispense` ignora los lotes vencidos y vacíos, valida que alcance el stock no vencido, descuenta primero del lote que vence antes y descuenta también el stock total del producto, todo dentro de una transacción.
6. **Documento.** La venta tiene su número de factura `FV-xxxxx` y se puede generar el PDF (`sales.pdf`).
7. **Indicador.** El dashboard actualiza sus indicadores (stock bajo, órdenes por recibir, ventas y compras del mes). Con ellos el dueño decide la siguiente compra y el ciclo vuelve a empezar.

## 2. Reglas de negocio

| # | Regla | Módulo | Dónde se implementa |
|---|---|---|---|
| 1 | Nunca se vende un lote vencido. | Ventas / Inventario | **Servicio** `DispensingService`: solo toma lotes con `expiration_date >= hoy` y `quantity > 0`. |
| 2 | Se vende primero el lote que vence antes (FEFO). | Ventas / Inventario | **Servicio** `DispensingService`: `orderBy('expiration_date')`. |
| 3 | Si no alcanza el stock no vencido, la venta falla y no cambia nada. | Ventas | **Servicio** `DispensingService`: lanza `ValidationException` dentro de `DB::transaction` (rollback). |
| 4 | Quien crea la orden de compra no la recibe. | Compras | **Middleware** `permission:crear-compras` y `permission:recibir-compras` en `PurchaseController`, y **seeder** `DrogueriaRolesSeeder`: el admin tiene `crear-compras` pero no `recibir-compras`. |
| 5 | Solo se reciben órdenes `pendiente` o `parcial`. | Compras | **Servicio** `PurchaseService::receive` (`canBeReceived`). |
| 6 | No se puede recibir más de lo pedido. | Compras | **Servicio** `PurchaseService::receive` (compara con la cantidad pendiente de la línea). |
| 7 | Solo se anula una orden que no ha recibido mercancía (`pendiente`). | Compras | **Servicio** `PurchaseService::cancel`. |
| 8 | El mismo producto no repite número de lote. | Inventario | **Base de datos**: índice único `(product_id, lot_number)` en `lots`. |
| 9 | La orden de compra exige proveedor activo, productos sin repetir, cantidad ≥ 1 y costo > 0. | Compras | **Form Request** `StorePurchaseRequest`. |

## 3. Estados

**Orden de compra**

| Estado | Qué pasa con el inventario |
|---|---|
| `pendiente` | Nada: registrar la orden no mueve el stock. Se puede anular. |
| `parcial` | El stock del producto aumenta solo con lo que ya llegó (`received_quantity`). Todavía admite más recepciones. |
| `recibida` | Todo lo pedido ya llegó y está sumado al stock. Se guarda `received_at`. |
| `anulada` | Nada: solo se anula desde `pendiente`, por lo que no hay stock que devolver. |

**Venta**

| Estado | Qué pasa con el inventario |
|---|---|
| `pendiente` | Estado definido en la tabla `sales`; `SaleService` registra las ventas directamente como `pagada`. |
| `pagada` | Se descuenta el stock de los productos vendidos. |
| `anulada` | Se devuelve el stock a los productos. |

## 4. Indicadores (KPIs)

1. **Órdenes por recibir:** órdenes de compra en estado `pendiente` o `parcial`.
2. **Stock bajo (≤ 5):** productos con 5 unidades o menos; alimenta la «Sugerencia de reabastecimiento».
3. **Ventas del mes:** cantidad de ventas y valor total del mes en curso.
4. **Ventas − Compras del mes:** lo vendido menos lo comprado en el mes.
5. **Top 5 productos más vendidos del mes.**

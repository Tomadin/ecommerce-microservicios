# 01: Órdenes inmutables con snapshot de líneas y total

- **Estado**: Aceptado
- **Fecha**: 2026-08-06
- **Autores**: Brian Tomadin

## Contexto

Una orden es un registro histórico: representa lo que efectivamente se compró y se cobró en un
momento dado. El `orders-service` la genera a partir de un carrito. El problema es cómo modelar
los datos de las líneas (código, nombre, precio unitario, cantidad) y el total dentro de la orden.

El precio y el nombre de un producto viven en el `products-service` y pueden cambiar con el tiempo.
Si mañana sube el precio de un electrodoméstico, la orden de ayer no puede alterarse: tiene que
seguir reflejando lo que se cobró. Además, cada servicio tiene su propia base (database-per-service),
así que la orden no puede depender de tablas de otros servicios para reconstruirse.

Distinción para no confundir el anti-patrón: armar el snapshot implica una llamada
sincrónica de una sola vez al carrito, en el momento de confirmar (write). Eso es aceptable. Lo que
se debe evitar es re-leer la cadena Orden -> Carrito -> Catálogo en cada lectura (GET) de una orden
ya creada.

## Opciones Evaluadas

- **Opción 1**: Referencia viva. La orden guarda IDs al carrito y a los productos, y recalcula o
  re-lee contra catálogo cada vez que se lee la orden.
  - **Pros**: sin duplicación de datos; una sola fuente de verdad para nombre y precio.
  - **Contras**: la orden deja de ser inmutable (un cambio de precio altera el histórico); obliga a
    la cadena sincrónica Orden -> Carrito -> Catálogo en cada lectura, con acoplamiento, latencia
    acumulada y un punto de fallo extra; si el carrito se vacía o el producto se borra, la orden
    queda inconsistente.
- **Opción 2**: Snapshot. Al confirmar, se copian las líneas (código, nombre, precio unitario,
  cantidad) y el total dentro de la orden como valores congelados (`OrderLine` dentro del agregado
  `Order`).
  - **Pros**: la orden es un registro histórico inmutable; se lee 100% de sus propios datos, sin
    llamar a otros servicios; resiliente a cambios de precio y a la vida del carrito y del catálogo;
    respeta database-per-service.
  - **Contras**: duplica datos (nombre y precio del producto viven también en la orden); si un dato
    maestro cambia legítimamente, la orden no se entera (que acá es lo deseado, pero hay que tenerlo
    claro); algo más de trabajo al confirmar.

## Decisión

Se elige la **Opción 2**: snapshot inmutable. Al confirmar una orden se copian las líneas y el total
como valores congelados dentro del agregado `Order`, y las lecturas posteriores se resuelven solo con
esos datos, sin re-leer carrito ni catálogo.

El argumento decisivo es la naturaleza del dominio: una orden es un hecho consumado. Lo que se cobró
es lo que se cobró, independientemente de cómo evolucionen los precios. La referencia viva (Opción 1)
rompe esa inmutabilidad y arrastra el anti-patrón de la cadena sincrónica en cada lectura.

## Consecuencias

- **Positivas**:
  - Inmutabilidad garantizada: los cambios de precio en catálogo no afectan órdenes pasadas.
  - Lecturas de orden autocontenidas: sin fan-out sincrónico, sin latencia acumulada, sin acoplamiento.
  - Resiliencia: una caída de `products-service` o `cart-service` no impide leer órdenes existentes.
  - Consistente con database-per-service: la orden no depende de tablas de otros servicios.
- **Negativas / Riesgos**:
  - Duplicación deliberada de datos (nombre y precio también en la orden). Es desnormalización a
    conciencia, no normalización rota.
  - El snapshot se arma con una llamada sincrónica de una sola vez al carrito, al confirmar: un
    acoplamiento puntual en escritura. Mitigación: es de una sola vez y en el write, no en cada lectura.
  - Si el modelo de producto gana campos que la orden debería mostrar, hay que decidir si sumarlos al
    snapshot; el histórico no se actualiza solo.

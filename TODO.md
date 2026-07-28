# Pendiente: sincronización por bloque completo (riesgo de dinero)

## El problema

`camara`, `bodega`, `procesos`, `pedidos` y `notas` se siguen guardando en
Supabase como **un solo bloque JSON por lista** (ver `persist()` en
index.html). Cada vez que alguien guarda algo, sube TODA la lista con lo
que tenga en memoria en ese momento — no solo el registro que cambió.

Si dos personas tienen la app abierta y una de ellas guarda con una copia
desactualizada (su pantalla no alcanzó a recibir por el polling de 8s el
cambio que hizo la otra persona segundos antes), su guardado **pisa y
borra en silencio** lo que la otra persona acababa de hacer.

Ya pasó y ya se corrigió una vez, en el mismo patrón:

- **2026-07-23** (`c0acbf7`): el catálogo (productos/proveedores/clientes/
  tipos de proceso) compartía una sola bandera "sucio" — cualquier edición
  subía las 5 listas completas y borraba productos agregados por otro
  dispositivo. Se arregló separando la bandera por lista y, después
  (`bccd16b`), rediseñando el catálogo a **una fila por item en Supabase**
  en vez de un array completo — ese es el patrón correcto a seguir.

- **2026-07-24** (lote P-0053, Corona): Luis y Janet recibieron el mismo
  pedido casi al mismo tiempo. El segundo guardado (de todo el bloque de
  `bodega`) borró el precio que había puesto Kateryn y las 3 ventas que
  ya había hecho Janet (folios #48/#49/#50) — el lote quedó completo y sin
  precio, como si nunca se hubiera tocado. Se corrigió el dato a mano y se
  agregó un candado puntual en `gRecibirPedido` (verifica contra Supabase
  antes de crear el lote, no solo contra la copia local) — pero eso **solo
  cubre la recepción de pedidos**, no el resto de ediciones a Cámara/Bodega.

- **2026-07-24** (`c5bc18d`): notas #48 y #50 se guardaron con partidas a
  $0 (Papa, Sargo ch.) porque `gNota()` nunca validaba que el precio fuera
  mayor a 0. Se corrigió agregando esa validación — pero es un síntoma
  relacionado, no la causa de fondo.

## Lo que falta (la solución real)

Aplicar a `camara`, `bodega`, `procesos`, `pedidos` y `notas` el mismo
rediseño que ya se hizo para el catálogo: una fila por item en Supabase
(con su propio id como clave), no un array completo por módulo. Así, dos
personas editando cosas distintas nunca se pisan entre sí — cada quien
sube solo lo que tocó.

Mientras eso no se haga, el riesgo sigue abierto en cualquier pantalla que
no sea "Recibir pedido" (por ejemplo: dos personas editando Cámara/Bodega,
o capturando notas, casi al mismo tiempo).

## Variante del mismo problema: colisión de ids dentro del catálogo

- **2026-07-28**: aunque el catálogo ya vive en filas individuales
  (`producto_<id>`) desde `bccd16b`, `siguienteIdProducto()` seguía
  calculando el siguiente id como `max(ids en memoria en ESE dispositivo) +
  1`. Un dispositivo con una copia local incompleta del catálogo (caché
  parcialmente vaciada, o que agregó un producto antes de terminar de
  sincronizar con Supabase) podía calcular un id bajo que YA pertenecía a
  un producto real — y como cada producto vive en su propia fila, guardar
  el nuevo **pisó en silencio la fila del viejo**. Así desaparecieron 14
  productos con inventario activo en Bodega (Ajo, Canela, Consomé Nork,
  Tomillo, Calamar picado, Arrachera, Aceite de oliva, Agua, Café, y las 5
  cervezas: Corona, Cristal, Modelo lata, Negra, Victoria) — sustituidos
  por productos con nombres distintos que reusaron sus mismos ids.
  Se restauraron a mano en Supabase (ids 200-213) y se corrigió
  `siguienteIdProducto()` para usar un piso persistido (`next_producto_id`)
  que nunca baja entre sesiones, en vez de confiar solo en la copia local.

- Esto es la misma familia de bug que el resto de este documento (una
  copia local desactualizada pisando Supabase), pero con un mecanismo
  distinto: no es "subir el bloque completo", es "reusar un id ya
  ocupado". El fix de `next_producto_id` reduce mucho el riesgo pero no lo
  elimina del todo: un dispositivo completamente nuevo (sin caché local)
  que agregue un producto en la ventana antes de que responda el primer
  fetch a Supabase todavía podría, en teoría, calcular un id ya usado si
  esa ventana coincide con la de otro dispositivo haciendo lo mismo. La
  solución completa sería verificar contra Supabase (no solo el piso
  local) antes de confirmar cada id nuevo — pendiente si se quiere cerrar
  el hueco al 100%.

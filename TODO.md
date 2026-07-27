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

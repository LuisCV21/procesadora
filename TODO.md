# 2026-09-24: guardados simultáneos y punto de partida del inventario

**Guardado con versión.** Cada registro de bodega_data que es "una fila por
registro" (`lotecam_`, `lotebod_`, `nota_`, `pedido_`, `proceso_`, `vale_`,
`gasto_`, `comp_`, `producto_`, `corte_`) lleva `_v` dentro de su value y se
guarda con la función SQL `bd_guardar(p_key, p_value, p_base_v, p_kg_delta)`
en vez del upsert ciego:

- versión igual a la que vio el dispositivo → se guarda y `_v` sube;
- versión distinta en un **lote** → se aplica solo la diferencia de kg de este
  dispositivo sobre lo que hay ahora (dos ventas simultáneas del mismo lote
  se restan las dos); si quedaría en negativo se rechaza;
- versión distinta en cualquier otra cosa → **no se sobrescribe**: se avisa
  en pantalla, queda en la bitácora ("Cambio rechazado por guardado
  simultáneo") y se recarga la versión actual;
- un registro que otra persona borró/movió no se revive.

`SRV` (en index.html) guarda por key la versión y kg vistos del servidor; los
guardados de una misma key van en cola (`enCola`) y la cola de pendientes
guarda la base con la que se intentó. `next_id`/`next_folio`/`next_pedido_num`
ya no pueden bajar, y cada dispositivo reserva bloques de 50 ids con
`reservar_ids(n)` para que dos computadoras no generen el mismo id.

También se corrigió `sincronizarRemoto`: el aviso de cambio que llegaba en
los 5 s posteriores a un guardado propio se descartaba (la pantalla se
quedaba con datos viejos hasta 3 min); ahora se difiere.

**Pendiente conocido:** una nota nueva que descuenta varios lotes no es una
transacción única — si un lote se rechaza (ej. quedaría en negativo), la
nota sí se guarda y el aviso lo indica; hay que corregir ese lote a mano.
El folio de nota todavía se toma del contador local (dos notas simultáneas
pueden salir con el mismo folio, aunque ya no se pisan entre sí).

**Punto de partida del inventario** (`inv_base`): ver Caja → Verificar
inventario. Después de un conteo físico se fija, y la verificación solo
muestra diferencias nuevas desde entonces (sin borrar historial).

# RESUELTO (2026-09-10): sincronización por bloque completo (riesgo de dinero)

## Estado actual

**Cerrado.** `camara`, `bodega`, `procesos`, `pedidos`, `notas`, `vales`,
`gastos`, `compensaciones` y `cortes_cerrados` ya viven como **una fila por
registro** en Supabase (`lotecam_<id>`, `lotebod_<id>`, `proceso_<id>`,
`pedido_<id>`, `nota_<id>`, `vale_<id>`, `gasto_<id>`, `comp_<id>`,
`corte_<mes>`), exactamente el mismo esquema que ya usaba el catálogo desde
julio. `persist()` ya no sube ningún arreglo completo — cada alta/edición/
baja guarda solo su propia fila (`guardarFila`/`eliminarFila`/`guardarFilas`/
`eliminarFilas`, junto a `kFila`/`reconstruirColeccionDesdeMapa`, la
generalización del mecanismo del catálogo). La migración de los blobs
viejos a filas es automática y de una sola vez (mismo patrón que ya usaba
el catálogo desde julio), y `aplicarDatos` reconstruye cada colección desde
sus filas frescas en vez de fusionar arreglos completos.

De paso se resolvió la inconsistencia que ya tenía `cortes_cerrados` (antes
`cerrarPeriodo` guardaba el blob completo directo y `extraerUtilidadPeriodo`
hacía doble escritura — ahora ambos son un solo upsert por fila) y las dos
operaciones de renombrado masivo (`renombrarProductoEnTodo`,
`actualizarCaducidades`), que siguen siendo N upserts sin candado (riesgo
aceptado: son acciones raras hechas por un admin, no algo que ocurra
decenas de veces al día como capturar una nota).

Antes de esto, ese mismo día (2026-09-09/10) ya se había agregado un primer
parche: `mergeArrayPorId` empezó a desempatar por `_mod` (marca de tiempo
de la edición real) en vez de "gana local a ciegas", y `doLogout()` pasó a
esperar los guardados en curso antes de recargar (antes la recarga
cancelaba peticiones a medias sin dejar rastro). Ese parche redujo el riesgo
pero no lo eliminaba — la migración a fila-por-registro de este documento
es la solución de fondo que ya lo reemplaza para las 9 colecciones.

**Riesgo residual** (documentado, no crítico): si dos dispositivos editan
literalmente el MISMO registro en la ventana de milisegundos entre leer y
guardar, sigue ganando el último en llegar — igual que el catálogo. Ya no
alcanza con tocar cualquier registro de la colección para pisar a alguien
más, hace falta tocar el mismo registro exacto.

## El problema (histórico, ya resuelto — se deja como referencia)

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

## Lo que se hizo (ver "Estado actual" arriba)

Se aplicó a `camara`, `bodega`, `procesos`, `pedidos`, `notas`, `vales`,
`gastos`, `compensaciones` y `cortes_cerrados` el mismo rediseño que ya
tenía el catálogo: una fila por item en Supabase (con su propio id como
clave), no un array completo por módulo. Dos personas editando cosas
distintas ya no se pisan entre sí — cada quien sube solo lo que tocó.

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

- **2026-08-05: hueco cerrado.** El botón "Agregar producto" (único punto
  donde una persona da de alta un producto de forma interactiva) ya no usa
  `siguienteIdProducto()`/`next_producto_id` — usa `idProductoUnico()`
  (timestamp en ms + 3 dígitos random). No depende de ningún estado
  compartido (ni copia local ni piso de Supabase), así que no hay cálculo
  posible que coincida entre dos dispositivos, sin importar qué tan
  desincronizados estén. `siguienteIdProducto()`/`next_producto_id` se
  dejaron intactos para los demás usos (seeds del catálogo al sincronizar,
  migración de ids viejos) porque esos corren justo después de un fetch
  fresco a Supabase y no por acción directa de una persona — riesgo mucho
  menor, no valía la pena tocarlos.
  Verificado contra Supabase el mismo día: los 84 productos base del
  código y las reglas de proceso de Sargo estaban completos — no había
  nada que restaurar en ese momento.

# Investigación de widgets de Flutter

- **Estudiante:** Julian Londoño Cardona
- **Grupo:** 60X  <!-- cambiar por 601 o 602 -->
- **Fecha:** XX de noviembre de 2026
- **Asignatura:** Programación Móvil (IF2004) – IUE 2026-02

## Mis widgets

1. `Dismissible`
2. `Badge`
3. `ExpansionTile`

> Todos los ejemplos son programas completos: se copian tal cual en [DartPad](https://dartpad.dev) y se ejecutan.

---

## 1. Dismissible

### Qué es y para qué sirve

`Dismissible` envuelve otro widget (por lo general una fila de una lista) y deja **deslizarlo hacia un lado para descartarlo**. Es el gesto de "swipe para borrar" que se ve en apps de correo o de compras. El widget se encarga de la animación de arrastre y de que la fila desaparezca encogiéndose. Lo que **no** hace es borrar el dato: eso me toca a mí en el callback `onDismissed`.

Si no existiera, tendría que armar el gesto a mano con `GestureDetector`, calcular el desplazamiento, animar la salida y colapsar el espacio. Es bastante código para algo que es muy común.

### Cuándo usarlo y cuándo no

- **Usarlo:** para quitar o archivar elementos de una lista, como productos del carrito, notificaciones o tareas.
- **No usarlo:**
  - Si la acción es grave y no se puede deshacer, o si hay que confirmarla siempre. En ese caso es mejor un botón con `AlertDialog`, o por lo menos `confirmDismiss`.
  - Si el usuario no tiene cómo saber que puede deslizar. El gesto queda oculto, así que conviene tener también un botón visible.
  - Si lo que necesito es reordenar los elementos. Para eso está `ReorderableListView`.

### Ejemplo mínimo

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text('Dismissible mínimo')),
        body: const ListaCarrito(),
      ),
    );
  }
}

class ListaCarrito extends StatefulWidget {
  const ListaCarrito({super.key});

  @override
  State<ListaCarrito> createState() => _ListaCarritoState();
}

class _ListaCarritoState extends State<ListaCarrito> {
  final List<String> productos = ['Papa', 'Tomate', 'Cebolla', 'Aguacate'];

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: productos.length,
      itemBuilder: (context, index) {
        final producto = productos[index];
        return Dismissible(
          key: ValueKey(producto),
          onDismissed: (direction) {
            setState(() => productos.removeAt(index));
          },
          child: ListTile(title: Text(producto)),
        );
      },
    );
  }
}
```

### Explicación del ejemplo

- La lista `productos` vive en un `StatefulWidget` porque cambia cuando se borra algo.
- Cada `ListTile` va envuelto en un `Dismissible`.
- `key: ValueKey(producto)` es **obligatorio**. Flutter usa la llave para saber qué fila se descartó. Uso el nombre del producto y no el `index`, porque los índices se corren cuando se borra un elemento.
- Al terminar el deslizamiento se ejecuta `onDismissed`. Ahí saco el producto de la lista y llamo a `setState` para redibujar.

**Lo que pasó cuando lo rompí a propósito:**
- Si quito el `setState(...removeAt...)` y dejo `onDismissed` vacío, Flutter lanza el error *"A dismissed Dismissible widget is still part of the tree"*. La fila se descartó visualmente pero el dato sigue en la lista.
- Si uso `ValueKey(index)` como llave, después de borrar uno las animaciones y los elementos se confunden, porque otra fila hereda esa llave.

### Propiedades importantes

| Propiedad | Qué hace | Valor de ejemplo |
|---|---|---|
| `key` | Identifica de forma única la fila. Es obligatoria. | `ValueKey(producto.id)` |
| `onDismissed` | Se ejecuta cuando la fila ya se descartó. Aquí se borra el dato. | `(dir) => setState(() => lista.removeAt(i))` |
| `direction` | Hacia dónde se permite deslizar. | `DismissDirection.endToStart` |
| `background` | Lo que se ve detrás mientras se desliza (o al deslizar hacia la derecha si hay `secondaryBackground`). | `Container(color: Colors.red)` |
| `secondaryBackground` | Fondo cuando se desliza en la dirección contraria. | `Container(color: Colors.green)` |
| `confirmDismiss` | Función asíncrona que decide si de verdad se descarta. Si devuelve `false`, la fila regresa a su lugar. | `(dir) async => await mostrarDialogo()` |
| `dismissThresholds` | Qué tanto hay que deslizar (0 a 1) para que cuente. | `{DismissDirection.endToStart: 0.6}` |

### Ejemplo personalizado

Carrito de Mercado Campesino. Deslizar a la izquierda **elimina** el producto (pide confirmación y permite deshacer). Deslizar a la derecha lo **guarda para después** y no lo borra.

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(colorSchemeSeed: Colors.green),
      home: Scaffold(
        appBar: AppBar(title: const Text('Mi carrito')),
        body: const CarritoPersonalizado(),
      ),
    );
  }
}

class CarritoPersonalizado extends StatefulWidget {
  const CarritoPersonalizado({super.key});

  @override
  State<CarritoPersonalizado> createState() => _CarritoPersonalizadoState();
}

class _CarritoPersonalizadoState extends State<CarritoPersonalizado> {
  final List<String> productos = [
    'Papa criolla x 1 kg',
    'Tomate chonto x 500 g',
    'Mora x 1 lb',
    'Queso campesino',
  ];

  Future<bool> _confirmarBorrado(String producto) async {
    final respuesta = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('¿Quitar del carrito?'),
        content: Text(producto),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Cancelar'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Quitar'),
          ),
        ],
      ),
    );
    return respuesta ?? false;
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: productos.length,
      itemBuilder: (context, index) {
        final producto = productos[index];
        return Dismissible(
          key: ValueKey(producto),
          // Derecha: guardar para después (verde)
          background: Container(
            color: Colors.green,
            alignment: Alignment.centerLeft,
            padding: const EdgeInsets.only(left: 20),
            child: const Icon(Icons.bookmark, color: Colors.white),
          ),
          // Izquierda: eliminar (rojo)
          secondaryBackground: Container(
            color: Colors.red,
            alignment: Alignment.centerRight,
            padding: const EdgeInsets.only(right: 20),
            child: const Icon(Icons.delete, color: Colors.white),
          ),
          dismissThresholds: const {DismissDirection.endToStart: 0.5},
          confirmDismiss: (direction) async {
            if (direction == DismissDirection.startToEnd) {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(content: Text('$producto guardado para después')),
              );
              return false; // no se borra, la fila regresa
            }
            return _confirmarBorrado(producto);
          },
          onDismissed: (direction) {
            setState(() => productos.removeAt(index));
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(
                content: Text('$producto eliminado'),
                action: SnackBarAction(
                  label: 'Deshacer',
                  onPressed: () {
                    setState(() => productos.insert(index, producto));
                  },
                ),
              ),
            );
          },
          child: ListTile(
            leading: const Icon(Icons.shopping_basket),
            title: Text(producto),
          ),
        );
      },
    );
  }
}
```

**Qué cambié respecto al mínimo:**
- Puse dos fondos con colores e íconos distintos (`background` y `secondaryBackground`), así el usuario sabe qué va a pasar antes de soltar.
- Agregué `confirmDismiss`. Hacia la derecha devuelve `false`, entonces la fila **vuelve** a su sitio y solo sale un aviso. Hacia la izquierda abre un `AlertDialog`.
- Agregué un `SnackBar` con "Deshacer" que vuelve a meter el producto en la misma posición.
- Con `dismissThresholds` hay que deslizar la mitad de la fila para borrar. Así se evitan borrados por accidente.

### Dónde lo usaría en Mercado Campesino

- En el **carrito de compras**, para quitar productos deslizando.
- En la **lista de notificaciones** (pedido confirmado, productor cercano), para descartarlas.
- En la **lista de favoritos** del comprador.

### Fuentes y ayudas usadas

- Documentación oficial: https://api.flutter.dev/flutter/widgets/Dismissible-class.html
- Receta oficial "Implement swipe to dismiss": https://docs.flutter.dev/cookbook/gestures/dismissible
- Apoyo de IA (Claude, de Anthropic) para organizar el documento y armar un primer borrador de los ejemplos. Ejecuté, modifiqué y rompí los ejemplos en DartPad para verificar cómo se comportan.

---

## 2. Badge

### Qué es y para qué sirve

`Badge` pone una **etiqueta pequeña encima de otro widget**, normalmente un ícono, para avisar que hay algo pendiente. Puede ser un número (3 productos en el carrito) o solo un punto (hay mensajes nuevos). Es parte de Material 3.

Si no existiera, tendría que montar un `Stack` con un `Positioned` y un `Container` redondo, y ajustar a mano el tamaño, el color y la posición para que no tape el ícono.

### Cuándo usarlo y cuándo no

- **Usarlo:** para contadores o avisos cortos sobre íconos: carrito, notificaciones, mensajes, destinos de una `NavigationBar`.
- **No usarlo:**
  - Para mostrar texto largo o etiquetas de categoría ("Orgánico", "Oferta"). Para eso sirven `Chip` o un `Container` con texto.
  - Como botón: `Badge` no responde a toques. Si necesito una acción, el `Badge` va dentro de un `IconButton`, no al revés.
  - **Error típico:** pensar que `Badge` cuenta solo. Es solo decoración: el número lo tengo que manejar yo con el estado y volver a dibujar.

### Ejemplo mínimo

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text('Badge mínimo')),
        body: const Center(
          child: Badge(
            label: Text('3'),
            child: Icon(Icons.shopping_cart, size: 40),
          ),
        ),
      ),
    );
  }
}
```

### Explicación del ejemplo

- `child` es el widget que se decora, en este caso el ícono del carrito.
- `label` es lo que va dentro del círculo rojo, aquí el texto `'3'`.
- Por defecto el `Badge` queda en la esquina superior derecha del `child`, con fondo rojo (el color de error del tema) y texto blanco.

**Lo que pasó cuando lo rompí a propósito:**
- Sin `label` se dibuja solo un **punto pequeño**. Eso sirve para decir "hay algo nuevo" sin dar un número.
- Con `label: Text('Productos en carrito')` el círculo se estira y queda feo y montado sobre el ícono. Así confirmé que es para textos muy cortos.

### Propiedades importantes

| Propiedad | Qué hace | Valor de ejemplo |
|---|---|---|
| `label` | Contenido de la etiqueta. Si es `null` se dibuja un punto. | `Text('5')` |
| `isLabelVisible` | Muestra u oculta todo el badge sin quitarlo del árbol. | `cantidad > 0` |
| `backgroundColor` | Color del fondo de la etiqueta. | `Colors.green` |
| `textColor` | Color del texto de la etiqueta. | `Colors.white` |
| `smallSize` | Diámetro del punto cuando no hay `label`. | `10` |
| `largeSize` | Alto de la etiqueta cuando tiene `label`. | `20` |
| `offset` | Mueve la etiqueta respecto a su posición normal. | `Offset(6, -6)` |
| `Badge.count(count: n)` | Constructor que convierte un número en el texto de la etiqueta. | `Badge.count(count: carrito.length)` |

### Ejemplo personalizado

Contador del carrito que **cambia en vivo**. Se oculta cuando llega a 0, usa los colores de la app y además hay un punto en las notificaciones.

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(colorSchemeSeed: Colors.green),
      home: const TiendaPage(),
    );
  }
}

class TiendaPage extends StatefulWidget {
  const TiendaPage({super.key});

  @override
  State<TiendaPage> createState() => _TiendaPageState();
}

class _TiendaPageState extends State<TiendaPage> {
  int cantidad = 0;
  bool hayNotificaciones = true;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Mercado Campesino'),
        actions: [
          IconButton(
            icon: Badge(
              isLabelVisible: hayNotificaciones,
              smallSize: 10,
              backgroundColor: Colors.orange,
              child: const Icon(Icons.notifications),
            ),
            onPressed: () => setState(() => hayNotificaciones = false),
          ),
          IconButton(
            icon: Badge.count(
              count: cantidad,
              isLabelVisible: cantidad > 0,
              backgroundColor: Colors.green.shade800,
              textColor: Colors.white,
              largeSize: 20,
              offset: const Offset(8, -6),
              child: const Icon(Icons.shopping_cart),
            ),
            onPressed: () {},
          ),
        ],
      ),
      body: Center(
        child: Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            IconButton(
              icon: const Icon(Icons.remove_circle, size: 40),
              onPressed: cantidad == 0 ? null : () => setState(() => cantidad--),
            ),
            Text('$cantidad', style: const TextStyle(fontSize: 32)),
            IconButton(
              icon: const Icon(Icons.add_circle, size: 40),
              onPressed: () => setState(() => cantidad++),
            ),
          ],
        ),
      ),
    );
  }
}
```

**Qué cambié respecto al mínimo:**
- Usé `Badge.count` para pasarle el número directamente, sin armar el `Text`.
- Con `isLabelVisible: cantidad > 0` el badge desaparece cuando el carrito está vacío. No se ve un "0".
- Cambié colores, tamaño y posición con `backgroundColor`, `textColor`, `largeSize` y `offset` para que combine con el verde de la app.
- La campana tiene un **punto naranja** sin `label` que se apaga al tocarla.
- El `Badge` está **dentro** del `IconButton`, así el botón sigue recibiendo los toques.

### Dónde lo usaría en Mercado Campesino

- En el ícono del **carrito**, con la cantidad de productos.
- En la **campana de notificaciones**, como punto de "hay algo nuevo".
- En la `NavigationBar` inferior, sobre la pestaña de **pedidos**, con los pedidos pendientes de un productor.

### Fuentes y ayudas usadas

- Documentación oficial: https://api.flutter.dev/flutter/material/Badge-class.html
- Guía de Material 3 sobre badges: https://m3.material.io/components/badges/overview
- Apoyo de IA (Claude, de Anthropic) para organizar el documento y armar un primer borrador de los ejemplos. Ejecuté, modifiqué y rompí los ejemplos en DartPad para verificar cómo se comportan.

---

## 3. ExpansionTile

### Qué es y para qué sirve

`ExpansionTile` es una fila con título (parecida a un `ListTile`) que al tocarla **se despliega y muestra más contenido** debajo. Al tocarla otra vez se pliega. Trae la flecha que gira y la animación de abrir y cerrar.

Sirve para no llenar la pantalla: el usuario ve primero los títulos y solo abre lo que le interesa. Si no existiera, tendría que manejar un `bool` de abierto o cerrado, animar la altura y girar la flecha a mano.

### Cuándo usarlo y cuándo no

- **Usarlo:** para preguntas frecuentes, detalles opcionales de un elemento o listas agrupadas por categoría.
- **No usarlo:**
  - Si la información es importante y el usuario siempre la debe ver. Escondida en un desplegable, se la puede saltar.
  - Si el contenido de adentro es muy largo o es otra pantalla completa. Es mejor navegar a una pantalla de detalle.
  - **Errores típicos:**
    - Ponerlo dentro de un `Column` sin scroll: al abrir varios se desborda (franja amarilla y negra).
    - Esperar que `initiallyExpanded` lo abra o cierre cuando cambio una variable. Solo se tiene en cuenta la primera vez que se construye.

### Ejemplo mínimo

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text('ExpansionTile mínimo')),
        body: const ExpansionTile(
          title: Text('¿Cómo pago mi pedido?'),
          children: [
            Padding(
              padding: EdgeInsets.all(16),
              child: Text('Puedes pagar en efectivo al recibir o por transferencia.'),
            ),
          ],
        ),
      ),
    );
  }
}
```

### Explicación del ejemplo

- `title` es lo que se ve siempre, la pregunta.
- `children` es la lista de widgets que aparecen al desplegar. Aquí es un solo texto con `Padding`.
- La flecha de la derecha y la animación las pone el widget solo. Yo no manejo ningún estado.

**Lo que pasó cuando lo rompí a propósito:**
- Si pongo `children: []`, la fila se ve igual y la flecha gira, pero no se abre nada. Funciona, aunque no tiene sentido.
- Si pongo 10 `ExpansionTile` dentro de un `Column` y los abro, aparece el error de *overflow*. Se arregla cambiando el `Column` por un `ListView`.

### Propiedades importantes

| Propiedad | Qué hace | Valor de ejemplo |
|---|---|---|
| `title` | Widget principal de la fila, siempre visible. | `Text('Finca La Esperanza')` |
| `subtitle` | Texto secundario debajo del título. | `Text('Rionegro · 4 productos')` |
| `leading` | Widget a la izquierda del título. | `Icon(Icons.agriculture)` |
| `trailing` | Reemplaza la flecha por otro widget. | `Icon(Icons.add)` |
| `children` | Lo que se muestra al desplegar. | `[ListTile(...), ListTile(...)]` |
| `initiallyExpanded` | Si arranca abierto. Solo aplica en la primera construcción. | `true` |
| `onExpansionChanged` | Se ejecuta al abrir o cerrar y recibe `true` o `false`. | `(abierto) => print(abierto)` |
| `backgroundColor` / `collapsedBackgroundColor` | Color de fondo cuando está abierto / cerrado. | `Colors.green.shade50` |
| `childrenPadding` | Margen interno del contenido desplegado. | `EdgeInsets.symmetric(horizontal: 16)` |

### Ejemplo personalizado

Lista de **productores cercanos**. Cada uno se despliega y muestra sus productos. El primero viene abierto y el ícono cambia según si la fila está abierta o cerrada.

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(colorSchemeSeed: Colors.green),
      home: Scaffold(
        appBar: AppBar(title: const Text('Productores cercanos')),
        body: const ListaProductores(),
      ),
    );
  }
}

class Productor {
  final String nombre;
  final String municipio;
  final List<String> productos;
  const Productor(this.nombre, this.municipio, this.productos);
}

const productores = [
  Productor('Finca La Esperanza', 'Rionegro', ['Papa criolla', 'Arveja', 'Mora']),
  Productor('Huerta Doña Rosa', 'Marinilla', ['Lechuga', 'Cilantro']),
  Productor('Lácteos El Ordeño', 'San Pedro', ['Queso campesino', 'Kumis', 'Cuajada']),
];

class ListaProductores extends StatefulWidget {
  const ListaProductores({super.key});

  @override
  State<ListaProductores> createState() => _ListaProductoresState();
}

class _ListaProductoresState extends State<ListaProductores> {
  final Set<int> abiertos = {0};

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: productores.length,
      itemBuilder: (context, i) {
        final p = productores[i];
        final abierto = abiertos.contains(i);
        return ExpansionTile(
          initiallyExpanded: i == 0,
          leading: Icon(abierto ? Icons.storefront : Icons.agriculture),
          title: Text(p.nombre, style: const TextStyle(fontWeight: FontWeight.bold)),
          subtitle: Text('${p.municipio} · ${p.productos.length} productos'),
          backgroundColor: Colors.green.shade50,
          collapsedBackgroundColor: Colors.white,
          childrenPadding: const EdgeInsets.only(left: 24, bottom: 8),
          expandedCrossAxisAlignment: CrossAxisAlignment.start,
          onExpansionChanged: (estaAbierto) {
            setState(() {
              estaAbierto ? abiertos.add(i) : abiertos.remove(i);
            });
          },
          children: [
            for (final producto in p.productos)
              Padding(
                padding: const EdgeInsets.symmetric(vertical: 4),
                child: Text('• $producto'),
              ),
          ],
        );
      },
    );
  }
}
```

**Qué cambié respecto al mínimo:**
- Los datos salen de una lista de objetos `Productor` y no están escritos a mano, como sería en la app real.
- Uso `ListView.builder` para que no haya desbordamiento aunque abra todos.
- Con `initiallyExpanded: i == 0` el primer productor arranca abierto.
- Con `onExpansionChanged` guardo qué filas están abiertas y cambio el `leading` (finca o tienda).
- Los colores cambian según el estado, con `backgroundColor` y `collapsedBackgroundColor`, y el contenido va alineado a la izquierda con sangría (`childrenPadding`, `expandedCrossAxisAlignment`).

### Dónde lo usaría en Mercado Campesino

- En la **lista de productores cercanos**, para ver sus productos sin entrar a otra pantalla.
- En la sección de **preguntas frecuentes** (pagos, envíos, devoluciones).
- En el **detalle de un pedido**, para desplegar la lista de productos de cada productor.

### Fuentes y ayudas usadas

- Documentación oficial: https://api.flutter.dev/flutter/material/ExpansionTile-class.html

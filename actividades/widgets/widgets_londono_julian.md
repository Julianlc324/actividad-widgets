# Investigación de widgets de Flutter

- **Estudiante:** Julian Londoño Cardona
- **Grupo:** 601
- **Fecha:** 22 de noviembre de 2026
- **Asignatura:** Programación Móvil (IF2004) – IUE 2026-02

## Mis widgets

1. `Dismissible`
2. `RefreshIndicator`
3. `Stepper`

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

## 2. RefreshIndicator

### Qué es y para qué sirve

`RefreshIndicator` agrega el gesto de **"halar hacia abajo para recargar"** (*pull to refresh*) a una lista o cualquier contenido con scroll. Cuando el usuario arrastra desde arriba, aparece un círculo de carga que gira mientras se ejecuta la función `onRefresh`. Cuando esa función termina, el círculo desaparece solo.

Si no existiera, tendría que detectar el arrastre más allá del inicio de la lista, dibujar y animar el indicador, y saber cuándo esconderlo. Así que lo normal sería poner un botón de "Recargar", que es menos natural en el celular.

### Cuándo usarlo y cuándo no

- **Usarlo:** en listas cuyos datos cambian y se traen de un servidor, como productos disponibles, pedidos o notificaciones.
- **No usarlo:**
  - Si los datos no cambian (una lista fija de categorías o una pantalla de ayuda). El gesto no hace nada útil.
  - Si los datos se actualizan solos en tiempo real (por ejemplo con un `StreamBuilder`). Ahí no hace falta recargar a mano.
  - Para cargar **más** elementos al llegar al final de la lista. Eso es scroll infinito y se hace de otra forma.
  - **Errores típicos:**
    - Poner como `child` algo que **no tiene scroll** (un `Column` o un `Center`). El gesto no se detecta y no pasa nada.
    - Si el contenido con scroll tiene un `physics` que no deja desplazar cuando la lista es corta, tampoco se puede halar. Se asegura con `physics: AlwaysScrollableScrollPhysics()`.
    - `onRefresh` debe devolver un `Future`. Si la función no espera (`await`) la carga, el círculo desaparece antes de que lleguen los datos.

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
        appBar: AppBar(title: const Text('RefreshIndicator mínimo')),
        body: const ListaProductos(),
      ),
    );
  }
}

class ListaProductos extends StatefulWidget {
  const ListaProductos({super.key});

  @override
  State<ListaProductos> createState() => _ListaProductosState();
}

class _ListaProductosState extends State<ListaProductos> {
  final List<String> productos = ['Papa', 'Tomate', 'Cebolla'];

  Future<void> _recargar() async {
    await Future.delayed(const Duration(seconds: 2)); // simula el servidor
    setState(() => productos.insert(0, 'Producto nuevo ${productos.length + 1}'));
  }

  @override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: _recargar,
      child: ListView.builder(
        itemCount: productos.length,
        itemBuilder: (context, index) => ListTile(title: Text(productos[index])),
      ),
    );
  }
}
```

### Explicación del ejemplo

- El `RefreshIndicator` envuelve al `ListView`, que es el que tiene scroll.
- `onRefresh` recibe la función `_recargar`, que es `async` y devuelve `Future<void>`.
- `Future.delayed` de 2 segundos hace de cuenta que estamos pidiendo datos a un servidor. Mientras ese `Future` no termina, el círculo sigue girando.
- Al terminar, agrego un producto al inicio con `setState` para que se vea el cambio.

**Lo que pasó cuando lo rompí a propósito:**
- Si cambio el `ListView` por un `Column` con los textos, al halar hacia abajo no aparece nada. El `RefreshIndicator` necesita un hijo con scroll.
- Si quito el `await` del `Future.delayed`, el círculo aparece y desaparece de una vez, y el producto se agrega antes de los 2 segundos.
- Si le pongo al `ListView` `physics: const NeverScrollableScrollPhysics()`, la lista ya no se mueve y el gesto de recargar tampoco funciona.

### Propiedades importantes

| Propiedad | Qué hace | Valor de ejemplo |
|---|---|---|
| `onRefresh` | Función que se ejecuta al soltar. Debe devolver un `Future`; el círculo gira hasta que termina. Es obligatoria. | `_recargar` |
| `child` | Contenido con scroll que se puede halar. Es obligatorio. | `ListView.builder(...)` |
| `color` | Color del círculo que gira. | `Colors.green` |
| `backgroundColor` | Color del fondo redondo detrás del círculo. | `Colors.white` |
| `displacement` | Qué tan abajo queda el círculo mientras carga. | `60` |
| `strokeWidth` | Grosor de la línea del círculo. | `3` |
| `triggerMode` | Si el gesto funciona solo desde el borde de arriba o desde cualquier posición. | `RefreshIndicatorTriggerMode.anywhere` |
| `RefreshIndicator.adaptive` | Variante que en iOS usa el indicador estilo Cupertino. Cuenta como el mismo widget. | `RefreshIndicator.adaptive(...)` |

### Ejemplo personalizado

Lista de productos disponibles en Mercado Campesino. Se puede recargar halando hacia abajo **o** con un botón en la barra superior (usando una `GlobalKey`), con colores de la app y un aviso al terminar.

```dart
import 'dart:math';
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(colorSchemeSeed: Colors.green),
      home: const ProductosPage(),
    );
  }
}

class ProductosPage extends StatefulWidget {
  const ProductosPage({super.key});

  @override
  State<ProductosPage> createState() => _ProductosPageState();
}

class _ProductosPageState extends State<ProductosPage> {
  final _refreshKey = GlobalKey<RefreshIndicatorState>();
  final _aleatorio = Random();
  final List<String> catalogo = [
    'Papa criolla', 'Mora', 'Lulo', 'Queso campesino',
    'Arveja', 'Huevos de campo', 'Cilantro', 'Panela',
  ];
  List<String> disponibles = ['Papa criolla', 'Mora'];
  DateTime? ultimaActualizacion;

  Future<void> _recargar() async {
    await Future.delayed(const Duration(seconds: 2)); // simula el servidor
    setState(() {
      disponibles = (catalogo.toList()..shuffle(_aleatorio))
          .take(2 + _aleatorio.nextInt(5))
          .toList();
      ultimaActualizacion = DateTime.now();
    });
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('${disponibles.length} productos disponibles')),
    );
  }

  @override
  Widget build(BuildContext context) {
    final hora = ultimaActualizacion == null
        ? 'Hala hacia abajo para actualizar'
        : 'Actualizado a las ${ultimaActualizacion!.hour}:'
            '${ultimaActualizacion!.minute.toString().padLeft(2, '0')}';

    return Scaffold(
      appBar: AppBar(
        title: const Text('Disponibles hoy'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () => _refreshKey.currentState?.show(),
          ),
        ],
      ),
      body: RefreshIndicator(
        key: _refreshKey,
        onRefresh: _recargar,
        color: Colors.green.shade800,
        backgroundColor: Colors.green.shade50,
        displacement: 60,
        strokeWidth: 3,
        child: ListView.builder(
          physics: const AlwaysScrollableScrollPhysics(),
          itemCount: disponibles.length + 1,
          itemBuilder: (context, index) {
            if (index == 0) {
              return Padding(
                padding: const EdgeInsets.all(12),
                child: Text(hora, style: const TextStyle(color: Colors.grey)),
              );
            }
            return ListTile(
              leading: const Icon(Icons.eco, color: Colors.green),
              title: Text(disponibles[index - 1]),
            );
          },
        ),
      ),
    );
  }
}
```

**Qué cambié respecto al mínimo:**
- Cambié el aspecto del indicador con `color`, `backgroundColor`, `displacement` y `strokeWidth` para que combine con el verde de la app.
- Puse `physics: AlwaysScrollableScrollPhysics()` para que se pueda halar aunque haya solo 2 productos.
- Agregué una `GlobalKey<RefreshIndicatorState>`. Con `_refreshKey.currentState?.show()` el botón de la barra muestra el mismo indicador y ejecuta la misma recarga, sin que el usuario tenga que halar.
- Cada recarga cambia de verdad la lista (escoge productos al azar del catálogo), muestra la hora de la última actualización y un `SnackBar` con cuántos productos hay.
- Antes de mostrar el `SnackBar` reviso `mounted`, porque si el usuario salió de la pantalla durante la espera ya no hay `context` válido.

### Dónde lo usaría en Mercado Campesino

- En la **lista de productos disponibles hoy**, porque los productores van publicando y agotando productos durante el día.
- En **mis pedidos**, para ver si el productor ya confirmó o despachó.
- En la **lista de productores cercanos**, después de que el usuario cambia de ubicación.

### Fuentes y ayudas usadas

- Documentación oficial: https://api.flutter.dev/flutter/material/RefreshIndicator-class.html
- Apoyo de IA (Claude, de Anthropic) para organizar el documento y armar un primer borrador de los ejemplos. Ejecuté, modifiqué y rompí los ejemplos en DartPad para verificar cómo se comportan.

---

## 3. Stepper

### Qué es y para qué sirve

`Stepper` muestra un **proceso dividido en pasos numerados** que se recorren en orden: paso 1, paso 2, paso 3… Cada paso tiene un título, un contenido y un estado (pendiente, en edición, completo o con error). Trae botones para avanzar y retroceder.

Sirve para procesos largos que asustarían en una sola pantalla, como un registro o una compra. El usuario ve en qué paso va y cuánto le falta. Si no existiera, tendría que dibujar los círculos numerados, las líneas que los unen, los botones y manejar qué contenido se muestra en cada momento.

### Cuándo usarlo y cuándo no

- **Usarlo:** en procesos de varios pasos con un orden claro: pagar un pedido, registrarse como productor, publicar un producto.
- **No usarlo:**
  - Si son solo 1 o 2 campos. Un formulario normal es más rápido.
  - Si el usuario puede ir a cualquier sección en cualquier orden. Para eso son mejores las pestañas (`TabBar`).
  - Si los pasos son pantallas muy grandes. Puede ser mejor navegar entre pantallas o usar un `PageView`.
  - **Errores típicos:**
    - Creer que `Stepper` avanza solo. No es así: yo guardo `currentStep` en el estado y lo cambio en `onStepContinue` y `onStepCancel`. Si no lo hago, los botones no hacen nada.
    - Sumar 1 a `currentStep` en el último paso. El índice se sale de la lista de pasos y la app falla.
    - En `StepperType.horizontal` con muchos pasos o títulos largos, los títulos no caben y se desbordan.

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
        appBar: AppBar(title: const Text('Stepper mínimo')),
        body: const PasosPedido(),
      ),
    );
  }
}

class PasosPedido extends StatefulWidget {
  const PasosPedido({super.key});

  @override
  State<PasosPedido> createState() => _PasosPedidoState();
}

class _PasosPedidoState extends State<PasosPedido> {
  int pasoActual = 0;

  @override
  Widget build(BuildContext context) {
    return Stepper(
      currentStep: pasoActual,
      onStepContinue: () {
        if (pasoActual < 2) setState(() => pasoActual++);
      },
      onStepCancel: () {
        if (pasoActual > 0) setState(() => pasoActual--);
      },
      steps: const [
        Step(title: Text('Carrito'), content: Text('Revisa tus productos.')),
        Step(title: Text('Dirección'), content: Text('¿Dónde entregamos?')),
        Step(title: Text('Pago'), content: Text('Elige cómo pagar.')),
      ],
    );
  }
}
```

### Explicación del ejemplo

- `steps` es la lista de pasos. Cada `Step` tiene un `title` (lo que se ve siempre) y un `content` (lo que se ve solo cuando ese paso está abierto).
- `currentStep` dice cuál paso está abierto. Lo guardo en `pasoActual` dentro de un `StatefulWidget`.
- Los botones "Continue" y "Cancel" llaman a `onStepContinue` y `onStepCancel`. Ahí subo o bajo `pasoActual` con `setState`.
- Los `if` evitan pasarme del último paso (índice 2) o bajar de 0.

**Lo que pasó cuando lo rompí a propósito:**
- Si quito el `setState` de `onStepContinue`, el botón "Continue" se ve pero no pasa nada. Confirma que el `Stepper` no maneja su propio estado.
- Si quito el `if (pasoActual < 2)` y presiono "Continue" en el último paso, `currentStep` queda en 3 y solo hay 3 pasos (0, 1, 2): sale un error o ya no queda ningún paso abierto.
- Si quito `onStepContinue` completo, el botón "Continue" queda deshabilitado (gris).

### Propiedades importantes

| Propiedad | Qué hace | Valor de ejemplo |
|---|---|---|
| `steps` | Lista de pasos (`Step`). Es obligatoria. | `[Step(...), Step(...)]` |
| `currentStep` | Índice del paso abierto. | `pasoActual` |
| `onStepContinue` | Se ejecuta al presionar continuar. Aquí avanzo el paso. | `() => setState(() => pasoActual++)` |
| `onStepCancel` | Se ejecuta al presionar cancelar o atrás. | `() => setState(() => pasoActual--)` |
| `onStepTapped` | Se ejecuta al tocar el título de un paso; recibe su índice. | `(i) => setState(() => pasoActual = i)` |
| `type` | Pasos en vertical (por defecto) u horizontal. | `StepperType.horizontal` |
| `controlsBuilder` | Reemplaza los botones por defecto por los míos. | `(context, details) => Row(...)` |
| `Step.state` | Estado del paso: cambia el ícono del círculo. | `StepState.complete` |
| `Step.isActive` | Si el círculo del paso se pinta con el color principal. | `pasoActual >= 1` |

### Ejemplo personalizado

Proceso de compra en Mercado Campesino, **horizontal**, con botones propios en español, pasos que se marcan como completos, validación de la dirección y un botón de "Confirmar pedido" en el último paso.

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
        appBar: AppBar(title: const Text('Finalizar compra')),
        body: const CompraStepper(),
      ),
    );
  }
}

class CompraStepper extends StatefulWidget {
  const CompraStepper({super.key});

  @override
  State<CompraStepper> createState() => _CompraStepperState();
}

class _CompraStepperState extends State<CompraStepper> {
  int paso = 0;
  final direccionCtrl = TextEditingController();
  bool direccionConError = false;
  String metodoPago = 'Efectivo';

  StepState _estado(int i) {
    if (i == 1 && direccionConError) return StepState.error;
    if (paso > i) return StepState.complete;
    if (paso == i) return StepState.editing;
    return StepState.indexed;
  }

  void _continuar() {
    if (paso == 1 && direccionCtrl.text.trim().isEmpty) {
      setState(() => direccionConError = true);
      return;
    }
    if (paso < 2) {
      setState(() {
        direccionConError = false;
        paso++;
      });
    } else {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Pedido confirmado. Pago: $metodoPago')),
      );
    }
  }

  Widget _opcionPago(String metodo, IconData icono) {
    final elegido = metodoPago == metodo;
    return ListTile(
      leading: Icon(icono),
      title: Text(metodo),
      trailing: Icon(elegido ? Icons.check_circle : Icons.circle_outlined,
          color: elegido ? Colors.green : Colors.grey),
      onTap: () => setState(() => metodoPago = metodo),
    );
  }

  @override
  void dispose() {
    direccionCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Stepper(
      type: StepperType.horizontal,
      currentStep: paso,
      onStepContinue: _continuar,
      onStepCancel: paso == 0 ? null : () => setState(() => paso--),
      onStepTapped: (i) {
        if (i < paso) setState(() => paso = i); // solo se puede volver atrás
      },
      controlsBuilder: (context, details) {
        final esUltimo = details.stepIndex == 2;
        return Padding(
          padding: const EdgeInsets.only(top: 16),
          child: Row(
            children: [
              FilledButton(
                onPressed: details.onStepContinue,
                child: Text(esUltimo ? 'Confirmar pedido' : 'Siguiente'),
              ),
              const SizedBox(width: 12),
              if (details.stepIndex > 0)
                TextButton(
                  onPressed: details.onStepCancel,
                  child: const Text('Atrás'),
                ),
            ],
          ),
        );
      },
      steps: [
        Step(
          title: const Text('Carrito'),
          isActive: paso >= 0,
          state: _estado(0),
          content: const Column(
            children: [
              ListTile(title: Text('Papa criolla x 1 kg'), trailing: Text('\$4.500')),
              ListTile(title: Text('Mora x 1 lb'), trailing: Text('\$6.000')),
            ],
          ),
        ),
        Step(
          title: const Text('Dirección'),
          isActive: paso >= 1,
          state: _estado(1),
          content: TextField(
            controller: direccionCtrl,
            decoration: InputDecoration(
              labelText: 'Dirección de entrega',
              errorText: direccionConError ? 'Escribe una dirección' : null,
            ),
          ),
        ),
        Step(
          title: const Text('Pago'),
          isActive: paso >= 2,
          state: _estado(2),
          content: Column(
            children: [
              _opcionPago('Efectivo', Icons.payments),
              _opcionPago('Transferencia', Icons.account_balance),
            ],
          ),
        ),
      ],
    );
  }
}
```

**Qué cambié respecto al mínimo:**
- Con `type: StepperType.horizontal` los pasos quedan en una fila arriba, como en las apps de compras. Usé títulos cortos para que quepan.
- Con `controlsBuilder` cambié los botones "Continue/Cancel" por "Siguiente/Atrás" en español. En el último paso el botón dice "Confirmar pedido" y en el primero no aparece "Atrás".
- Cada `Step` tiene `state` calculado: los pasos ya hechos salen con chulo (`complete`), el actual con lápiz (`editing`) y la dirección vacía en rojo (`error`). `isActive` pinta de verde los círculos hasta el paso actual.
- En `_continuar` valido que la dirección no esté vacía antes de avanzar.
- Con `onStepTapped` se puede volver a un paso anterior tocando su título, pero no saltarse pasos hacia adelante.
- `onStepCancel` es `null` en el primer paso, así no se puede retroceder desde el inicio.

### Dónde lo usaría en Mercado Campesino

- En el **proceso de compra**: carrito → dirección → pago → confirmación.
- En el **registro de un productor**: datos personales → datos de la finca → productos que vende.
- Al **publicar un producto nuevo**: foto → precio y cantidad → disponibilidad.

### Fuentes y ayudas usadas

- Documentación oficial: https://api.flutter.dev/flutter/material/Stepper-class.html
- Documentación de `Step`: https://api.flutter.dev/flutter/material/Step-class.html
- Apoyo de IA (Claude, de Anthropic) para organizar el documento y armar un primer borrador de los ejemplos. Ejecuté, modifiqué y rompí los ejemplos en DartPad para verificar cómo se comportan.

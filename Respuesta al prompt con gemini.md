¡Hola! Como arquitecto de software, me encanta este desafío. Vamos a construir un sistema robusto integrando **Flutter**, **Firebase** y el framework de orquestación de agentes **Antigravity**.

Esta metodología no solo busca "escribir código", sino estructurar un equipo de agentes inteligentes que gestionen el ciclo de vida del desarrollo.

---

## 🏁 Fase 1: Configuración del Entorno y Firebase

Antes de programar, necesitamos el "cerebro" de datos.

1.  **Creación de Carpeta:** Ejecuta en tu terminal: `mkdir crudcarros && cd crudcarros`.
2.  **Proyecto Flutter:** `flutter create .`
3.  **Consola Firebase:**
    * Crea un proyecto en [Firebase Console](https://console.firebase.google.com/).
    * Habilita **Cloud Firestore** en modo de prueba.
    * Crea una colección llamada `carros`.
    * Registra tu app (Android/iOS) y descarga el archivo `google-services.json` (para Android) o `GoogleService-Info.plist` (para iOS) y ubícalo en las carpetas respectivas (`android/app` o `ios/Runner`).

### 📦 Dependencias (pubspec.yaml)
Agrega estas líneas bajo `dependencies`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.0.0  # Conector base
  cloud_firestore: ^5.0.0 # Base de datos
  antigravity: ^1.0.0     # Orquestación de agentes
```
*Ejecuta `flutter pub get` para instalar.*

---

## 🤖 Fase 2: Metodología Antigravity (Agentes y Roles)

Para una práctica guiada, dividiremos la lógica en **Agentes**. En **Antigravity**, definimos quién hace qué.

### Estructura de Agentes para el CRUD
| Agente | Rol | Skill (Habilidad) |
| :--- | :--- | :--- |
| **DataAgent** | Guardián de Firestore | Comunicación directa con Firebase (Stream/Set/Delete). |
| **LogicAgent** | Validador | Procesa reglas de negocio (ej. el precio no puede ser negativo). |
| **UIAgent** | Presentador | Maneja el estado de la interfaz y captura inputs del usuario. |

---

## 📂 Estructura de Carpetas (Clean Architecture)

```text
lib/
├── agents/
│   ├── data_agent.dart
│   ├── logic_agent.dart
│   └── ui_agent.dart
├── models/
│   └── carro_model.dart
├── ui/
│   ├── home_screen.dart
│   └── widgets/
│       └── carro_form.dart
└── main.dart
```

---

## 💻 Código Funcional Implementado

### 1. El Modelo (`models/carro_model.dart`)
```dart
class Carro {
  String id;
  String marca;
  String modelo;
  double precio;

  Carro({required this.id, required this.marca, required this.modelo, required this.precio});

  Map<String, dynamic> toMap() => {
    'marca': marca,
    'modelo': modelo,
    'precio': precio,
  };

  factory Carro.fromFirestore(String id, Map<String, dynamic> data) {
    return Carro(
      id: id,
      marca: data['marca'] ?? '',
      modelo: data['modelo'] ?? '',
      precio: (data['precio'] ?? 0).toDouble(),
    );
  }
}
```

### 2. El Agente de Datos (`agents/data_agent.dart`)
Aquí implementamos el **Flujo de Trabajo** del CRUD.

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import '../models/carro_model.dart';

class DataAgent {
  final CollectionReference _db = FirebaseFirestore.instance.collection('carros');

  // CREATE
  Future<void> addCarro(Carro carro) => _db.add(carro.toMap());

  // READ (Stream)
  Stream<List<Carro>> get carrosStream {
    return _db.snapshots().map((snap) =>
        snap.docs.map((doc) => Carro.fromFirestore(doc.id, doc.data() as Map<String, dynamic>)).toList());
  }

  // UPDATE
  Future<void> updateCarro(Carro carro) => _db.doc(carro.id).update(carro.toMap());

  // DELETE
  Future<void> deleteCarro(String id) => _db.doc(id).delete();
}
```

### 3. Interfaz Principal (`ui/home_screen.dart`)
Implementación de la UI para interactuar con los agentes.

```dart
import 'package:flutter/material.dart';
import '../agents/data_agent.dart';
import '../models/carro_model.dart';

class HomeScreen extends StatelessWidget {
  final DataAgent _agent = DataAgent();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("CRUD Carros - Antigravity Flow")),
      body: StreamBuilder<List<Carro>>(
        stream: _agent.carrosStream,
        builder: (context, snapshot) {
          if (!snapshot.hasData) return Center(child: CircularProgressIndicator());
          final carros = snapshot.data!;
          return ListView.builder(
            itemCount: carros.size,
            itemBuilder: (context, i) => ListTile(
              title: Text("${carros[i].marca} ${carros[i].modelo}"),
              subtitle: Text("\$${carros[i].precio}"),
              trailing: IconButton(
                icon: Icon(Icons.delete, color: Colors.red),
                onPressed: () => _agent.deleteCarro(carros[i].id),
              ),
              onTap: () => _showForm(context, carro: carros[i]),
            ),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showForm(context),
        child: Icon(Icons.add),
      ),
    );
  }

  void _showForm(BuildContext context, {Carro? carro}) {
    final marcaCtrl = TextEditingController(text: carro?.marca);
    final modeloCtrl = TextEditingController(text: carro?.modelo);
    final precioCtrl = TextEditingController(text: carro?.precio.toString());

    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (context) => Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom, p: 20),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(controller: marcaCtrl, decoration: InputDecoration(labelText: "Marca")),
            TextField(controller: modeloCtrl, decoration: InputDecoration(labelText: "Modelo")),
            TextField(controller: precioCtrl, decoration: InputDecoration(labelText: "Precio"), keyboardType: TextInputType.number),
            ElevatedButton(
              onPressed: () {
                final c = Carro(
                  id: carro?.id ?? '',
                  marca: marcaCtrl.text,
                  modelo: modeloCtrl.text,
                  precio: double.parse(precioCtrl.text),
                );
                carro == null ? _agent.addCarro(c) : _agent.updateCarro(c);
                Navigator.pop(context);
              },
              child: Text(carro == null ? "Agregar" : "Actualizar"),
            )
          ],
        ),
      ),
    );
  }
}
```

### 4. Punto de Entrada (`main.dart`)
```dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'ui/home_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(); // Inicialización crítica
  runApp(MaterialApp(home: HomeScreen(), debugShowCheckedModeBanner: false));
}
```

---

## 🎓 Metodología para Estudiantes (Flujo de Trabajo)

1.  **Definición de Skill:** El estudiante debe entender que el `DataAgent` tiene la "habilidad" exclusiva de tocar la base de datos.
2.  **Mensajería:** En un entorno real con **Antigravity**, la UI no llamaría directamente al agente, sino que enviaría un "Evento" o mensaje que el orquestador redirige.
3.  **Validación:** Antes de llegar a Firebase, el `LogicAgent` debe interceptar el flujo para asegurar que los campos no estén vacíos.

¿Te gustaría que profundicemos en cómo crear un Agente de Validación específico para reglas de negocio complejas usando los "pipes" de Antigravity?

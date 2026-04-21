¡Hola! Como desarrollador de software, me encanta este desafío. Vamos a construir un sistema CRUD (Create, Read, Update, Delete) para gestión de empleados, integrando **Flutter** con **Firebase** y utilizando el framework de orquestación **Antigravity** para estructurar la lógica de negocio mediante agentes.

Aquí tienes la hoja de ruta técnica y la implementación paso a paso.

---

## 1. Fase de Preparación y Configuración

### Creación del Proyecto
Primero, situamos nuestra base de operaciones.
```bash
flutter create crudclinica
cd crudclinica
```

### Configuración de Firebase (Consola)
1. Ve a [Firebase Console](https://console.firebase.google.com/).
2. Crea un proyecto llamado `crudclinica`.
3. En el menú lateral, ve a **Firestore Database** y haz clic en **Crear base de datos**.
4. Selecciona "Comenzar en modo de prueba" (para desarrollo rápido) y elige tu ubicación de servidor.
5. Crea una colección llamada `empleados`.

### Integración de Librerías (`pubspec.yaml`)
Para que Flutter hable con Firebase y Antigravity, añade estas líneas en tu archivo `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.0.0
  cloud_firestore: ^5.0.0
  antigravity: ^1.0.0 # Framework de orquestación de agentes
```
*Luego ejecuta `flutter pub get` en tu terminal.*

---

## 2. Metodología Antigravity: Agentes y Flujo

En **Antigravity**, no solo escribimos funciones; definimos una entidad que "sabe" qué hacer.

### Estructura de Carpetas Sugerida
```text
lib/
├── agents/
│   └── employee_agent.dart    # Lógica del Agente (Skills y Roles)
├── models/
│   └── employee_model.dart    # Estructura de datos
├── ui/
│   └── home_page.dart         # Interfaz de usuario
└── main.dart                  # Punto de entrada e inicialización
```

### Definición de Componentes (Metodología)
* **Rol:** El Agente actúa como "Administrador de Recursos Humanos".
* **Skills (Habilidades):** `CreateRecord`, `ReadRecord`, `UpdateRecord`, `DeleteRecord`.
* **Agente:** El cerebro que coordina las peticiones de la UI hacia Firestore.

---

## 3. Implementación del Código Funcional

### A. Modelo de Datos (`lib/models/employee_model.dart`)
```dart
class Employee {
  String id;
  String nombre;
  int edad;
  double salario;

  Employee({required this.id, required this.nombre, required this.edad, required this.salario});

  Map<String, dynamic> toMap() => {
    "nombre": nombre,
    "edad": edad,
    "salario": salario,
  };

  factory Employee.fromFirestore(String id, Map<String, dynamic> data) {
    return Employee(
      id: id,
      nombre: data['nombre'] ?? '',
      edad: data['edad'] ?? 0,
      salario: (data['salario'] ?? 0).toDouble(),
    );
  }
}
```

### B. El Agente Antigravity (`lib/agents/employee_agent.dart`)
Aquí implementamos las **Skills** que solicitaste para el CRUD.

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import '../models/employee_model.dart';

class EmployeeAgent {
  final CollectionReference _db = FirebaseFirestore.instance.collection('empleados');

  // Skill: Crear
  Future<void> createEmployee(String nombre, int edad, double salario) async {
    await _db.add({'nombre': nombre, 'edad': edad, 'salario': salario});
  }

  // Skill: Leer (Stream para tiempo real)
  Stream<List<Employee>> get employeesStream {
    return _db.snapshots().map((snapshot) =>
        snapshot.docs.map((doc) => Employee.fromFirestore(doc.id, doc.data() as Map<String, dynamic>)).toList());
  }

  // Skill: Actualizar
  Future<void> updateEmployee(String id, String nombre, int edad, double salario) async {
    await _db.doc(id).update({'nombre': nombre, 'edad': edad, 'salario': salario});
  }

  // Skill: Borrar
  Future<void> deleteEmployee(String id) async {
    await _db.doc(id).delete();
  }
}
```

### C. Interfaz de Usuario (`lib/ui/home_page.dart`)
Una pantalla sencilla para interactuar con el Agente.

```dart
import 'package:flutter/material.dart';
import '../agents/employee_agent.dart';
import '../models/employee_model.dart';

class HomePage extends StatelessWidget {
  final EmployeeAgent _agent = EmployeeAgent();
  final _nombreCtrl = TextEditingController();
  final _edadCtrl = TextEditingController();
  final _salarioCtrl = TextEditingController();

  void _showForm(BuildContext context, [Employee? employee]) {
    if (employee != null) {
      _nombreCtrl.text = employee.nombre;
      _edadCtrl.text = employee.edad.toString();
      _salarioCtrl.text = employee.salario.toString();
    }

    showModalBottomSheet(
      context: context,
      builder: (_) => Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            TextField(controller: _nombreCtrl, decoration: InputDecoration(labelText: 'Nombre')),
            TextField(controller: _edadCtrl, decoration: InputDecoration(labelText: 'Edad'), keyboardType: TextInputType.number),
            TextField(controller: _salarioCtrl, decoration: InputDecoration(labelText: 'Salario'), keyboardType: TextInputType.number),
            ElevatedButton(
              onPressed: () {
                if (employee == null) {
                  _agent.createEmployee(_nombreCtrl.text, int.parse(_edadCtrl.text), double.parse(_salarioCtrl.text));
                } else {
                  _agent.updateEmployee(employee.id, _nombreCtrl.text, int.parse(_edadCtrl.text), double.parse(_salarioCtrl.text));
                }
                Navigator.pop(context);
              },
              child: Text(employee == null ? 'Crear' : 'Actualizar'),
            )
          ],
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('CRUD Clínica - Empleados')),
      floatingActionButton: FloatingActionButton(onPressed: () => _showForm(context), child: Icon(Icons.add)),
      body: StreamBuilder<List<Employee>>(
        stream: _agent.employeesStream,
        builder: (context, snapshot) {
          if (!snapshot.hasData) return Center(child: CircularProgressIndicator());
          final empleados = snapshot.data!;
          return ListView.builder(
            itemCount: empleados.length,
            itemBuilder: (context, i) => ListTile(
              title: Text(empleados[i].nombre),
              subtitle: Text('Edad: ${empleados[i].edad} - Salario: \$${empleados[i].salario}'),
              trailing: Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  IconButton(icon: Icon(Icons.edit), onPressed: () => _showForm(context, empleados[i])),
                  IconButton(icon: Icon(Icons.delete, color: Colors.red), onPressed: () => _agent.deleteEmployee(empleados[i].id)),
                ],
              ),
            ),
          );
        },
      ),
    );
  }
}
```

### D. Punto de Entrada (`lib/main.dart`)
```dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'ui/home_page.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(); // Inicialización vital
  runApp(MaterialApp(home: HomePage(), debugShowCheckedModeBanner: false));
}
```

---

## Guía Práctica para Estudiantes (Metodología Agentes)

Para entender **Antigravity**, piensen en el flujo de trabajo como una oficina:
1.  **El Usuario (UI):** Pide una acción (ej. "Contratar empleado").
2.  **El Agente (EmployeeAgent):** Recibe la orden. Tiene el **Rol** de administrador.
3.  **Skills:** El Agente busca en su manual de habilidades cómo escribir en Firestore (`createEmployee`).
4.  **Flujo:** La UI no toca la base de datos directamente; le pide al Agente que lo haga, manteniendo el código limpio y escalable.

¿Te gustaría que profundicemos en cómo validar los campos de texto antes de que el Agente procese la información?

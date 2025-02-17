import 'package:flutter/material.dart';
import 'planet_list_page.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Gerenciador de Planetas',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: PlanetListPage(),
    );
  }
}
class Planet {
  final int? id;
  final String name;
  final double distance;
  final double size;
  final String? nickname;

  Planet({
    this.id,
    required this.name,
    required this.distance,
    required this.size,
    this.nickname,
  });

  Map<String, dynamic> toMap() {
    return {
      'name': name,
      'distance': distance,
      'size': size,
      'nickname': nickname,
    };
  }

  static Planet fromMap(Map<String, dynamic> map) {
    return Planet(
      id: map['id'],
      name: map['name'],
      distance: map['distance'],
      size: map['size'],
      nickname: map['nickname'],
    );
  }
}
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import 'planet.dart';

class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._init();

  static Database? _database;

  DatabaseHelper._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('planets.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);
    return await openDatabase(path, version: 1, onCreate: _onCreate);
  }

  Future _onCreate(Database db, int version) async {
    await db.execute('''
    CREATE TABLE planets (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT,
      distance REAL,
      size REAL,
      nickname TEXT
    )
    ''');
  }

  Future<int> insertPlanet(Planet planet) async {
    final db = await instance.database;
    return await db.insert('planets', planet.toMap());
  }

  Future<List<Planet>> fetchPlanets() async {
    final db = await instance.database;
    final result = await db.query('planets');
    return result.map((json) => Planet.fromMap(json)).toList();
  }

  Future<int> updatePlanet(Planet planet) async {
    final db = await instance.database;
    return await db.update(
      'planets',
      planet.toMap(),
      where: 'id = ?',
      whereArgs: [planet.id],
    );
  }

  Future<int> deletePlanet(int id) async {
    final db = await instance.database;
    return await db.delete(
      'planets',
      where: 'id = ?',
      whereArgs: [id],
    );
  }
}
import 'package:flutter/material.dart';
import 'database_helper.dart';
import 'planet.dart';
import 'planet_form_page.dart';

class PlanetListPage extends StatefulWidget {
  @override
  _PlanetListPageState createState() => _PlanetListPageState();
}

class _PlanetListPageState extends State<PlanetListPage> {
  late Future<List<Planet>> planets;

  @override
  void initState() {
    super.initState();
    planets = DatabaseHelper.instance.fetchPlanets();
  }

  void _refreshPlanets() {
    setState(() {
      planets = DatabaseHelper.instance.fetchPlanets();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Planetas"),
      ),
      body: FutureBuilder<List<Planet>>(
        future: planets,
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return Center(child: CircularProgressIndicator());
          }
          if (snapshot.hasError) {
            return Center(child: Text('Erro: ${snapshot.error}'));
          }
          final planetList = snapshot.data ?? [];
          return ListView.builder(
            itemCount: planetList.length,
            itemBuilder: (context, index) {
              final planet = planetList[index];
              return ListTile(
                title: Text(planet.name),
                subtitle: Text(planet.nickname ?? ''),
                onTap: () => _showPlanetDetails(planet),
                trailing: IconButton(
                  icon: Icon(Icons.delete),
                  onPressed: () async {
                    final confirm = await _showDeleteConfirmDialog();
                    if (confirm == true) {
                      await DatabaseHelper.instance.deletePlanet(planet.id!);
                      _refreshPlanets();
                    }
                  },
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _navigateToPlanetForm(),
        child: Icon(Icons.add),
      ),
    );
  }

  Future<void> _showPlanetDetails(Planet planet) async {
    // Exibir detalhes (não implementado neste exemplo)
  }

  Future<bool?> _showDeleteConfirmDialog() {
    return showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Confirmar Exclusão'),
        content: Text('Deseja excluir este planeta?'),
        actions: <Widget>[
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: Text('Cancelar'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: Text('Excluir'),
          ),
        ],
      ),
    );
  }

  void _navigateToPlanetForm() {
    Navigator.push(
      context,
      MaterialPageRoute(builder: (context) => PlanetFormPage(onSave: _refreshPlanets)),
    );
  }
}
import 'package:flutter/material.dart';
import 'database_helper.dart';
import 'planet.dart';

class PlanetFormPage extends StatefulWidget {
  final Function onSave;

  PlanetFormPage({required this.onSave});

  @override
  _PlanetFormPageState createState() => _PlanetFormPageState();
}

class _PlanetFormPageState extends State<PlanetFormPage> {
  final _formKey = GlobalKey<FormState>();
  late String _name;
  late double _distance;
  late double _size;
  String? _nickname;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Cadastro de Planeta")),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            children: <Widget>[
              TextFormField(
                decoration: InputDecoration(labelText: 'Nome do Planeta'),
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
                onSaved: (value) => _name = value!,
              ),
              TextFormField(
                decoration: InputDecoration(labelText: 'Distância do Sol (UA)'),
                keyboardType: TextInputType.number,
                validator: (value) {
                  if (value!.isEmpty || double.tryParse(value) == null || double.parse(value) <= 0) {
                    return 'Valor inválido';
                  }
                  return null;
                },
                onSaved: (value) => _distance = double.parse(value!),
              ),
              TextFormField(
                decoration: InputDecoration(labelText: 'Tamanho (km)'),
                keyboardType: TextInputType.number,
                validator: (value) {
                  if (value!.isEmpty || double.tryParse(value) == null || double.parse(value) <= 0) {
                    return 'Valor inválido';
                  }
                  return null;
                },
                onSaved: (value) => _size = double.parse(value!),
              ),
              TextFormField(
                decoration: InputDecoration(labelText: 'Apelido (opcional)'),
                onSaved: (value) => _nickname = value,
              ),
              SizedBox(height: 20),
              ElevatedButton(
                onPressed: _submit,
                child: Text('Salvar'),
              ),
            ],
          ),
        ),
      ),
    );
  }

  void _submit() {
    if (_formKey.currentState!.validate()) {
      _formKey.currentState!.save();
      final planet = Planet(
        name: _name,
        distance: _distance,
        size: _size,
        nickname: _nickname,
      );
      DatabaseHelper.instance.insertPlanet(planet);
      widget.onSave();
      Navigator.pop(context);
    }
  }
}

git init
git add .
git commit -m "Iniciando o projeto CRUD de planetas"
git branch -M main
git remote add origin <seu-link-do-repositorio>
git push -u origin main


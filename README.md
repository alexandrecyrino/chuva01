rain_record_app/
name: rain_record_app
description: Aplicativo para registro de dados pluviométricos

version: 1.0.0+1

environment:
  sdk: ">=2.17.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter
  sqflite: ^2.0.0
  path: ^1.8.0
  intl: ^0.17.0
  charts_flutter: ^0.12.0
  pdf: ^3.8.1
  printing: ^5.9.1
  flutter_localizations:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/
├── lib/
import 'package:flutter/material.dart';
import 'screens/home_screen.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Registro de Chuva',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const HomeScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}
│   ├── screens/import 'package:flutter/material.dart';
import 'package:intl/intl.dart';
import '../models/rain_record.dart';
import '../services/database_service.dart';
import 'reports_screen.dart';
import 'statistics_screen.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final DatabaseService _databaseService = DatabaseService.instance;
  final TextEditingController _millimetersController = TextEditingController();
  final TextEditingController _notesController = TextEditingController();
  DateTime _selectedDate = DateTime.now();
  List<RainRecord> _records = [];

  @override
  void initState() {
    super.initState();
    _loadRecords();
  }

  Future<void> _loadRecords() async {
    final records = await _databaseService.getAllRecords();
    setState(() {
      _records = records;
    });
  }

  Future<void> _selectDate(BuildContext context) async {
    final DateTime? picked = await showDatePicker(
      context: context,
      initialDate: _selectedDate,
      firstDate: DateTime(2000),
      lastDate: DateTime.now(),
    );
    if (picked != null && picked != _selectedDate) {
      setState(() {
        _selectedDate = picked;
      });
    }
  }

  Future<void> _saveRecord() async {
    if (_millimetersController.text.isEmpty) return;

    final record = RainRecord(
      date: _selectedDate,
      millimeters: double.parse(_millimetersController.text),
      notes: _notesController.text,
    );

    await _databaseService.insertRecord(record);
    _millimetersController.clear();
    _notesController.clear();
    _loadRecords();

    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Registro salvo com sucesso!')),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Registro de Chuva'),
        actions: [
          IconButton(
            icon: const Icon(Icons.bar_chart),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (context) => const StatisticsScreen()),
            ),
          ),
          IconButton(
            icon: const Icon(Icons.picture_as_pdf),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (context) => const ReportsScreen()),
            ),
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            Row(
              children: [
                Expanded(
                  child: Text(
                    DateFormat('dd/MM/yyyy').format(_selectedDate),
                    style: const TextStyle(fontSize: 18),
                  ),
                ),
                TextButton(
                  onPressed: () => _selectDate(context),
                  child: const Text('Alterar Data'),
                ),
              ],
            ),
            const SizedBox(height: 20),
            TextField(
              controller: _millimetersController,
              keyboardType: TextInputType.number,
              decoration: const InputDecoration(
                labelText: 'Milímetros de chuva',
                border: OutlineInputBorder(),
                suffixText: 'mm',
              ),
            ),
            const SizedBox(height: 20),
            TextField(
              controller: _notesController,
              decoration: const InputDecoration(
                labelText: 'Observações (opcional)',
                border: OutlineInputBorder(),
              ),
              maxLines: 3,
            ),
            const SizedBox(height: 20),
            ElevatedButton(
              onPressed: _saveRecord,
              child: const Text('Salvar Registro'),
              style: ElevatedButton.styleFrom(
                minimumSize: const Size(double.infinity, 50),
              ),
            ),
            const SizedBox(height: 20),
            Expanded(
              child: _records.isEmpty
                  ? const Center(child: Text('Nenhum registro encontrado'))
                  : ListView.builder(
                      itemCount: _records.length,
                      itemBuilder: (context, index) {
                        final record = _records[index];
                        return ListTile(
                          title: Text(DateFormat('dd/MM/yyyy').format(record.date)),
                          subtitle: Text('${record.millimeters} mm'),
                          trailing: Text(record.notes ?? ''),
                        );
                      },
                    ),
            ),
          ],
        ),
      ),
    );
  }
}
import 'package:flutter/material.dart';
import 'package:charts_flutter/flutter.dart' as charts;
import 'package:intl/intl.dart';
import '../models/rain_record.dart';
import '../services/database_service.dart';

class StatisticsScreen extends StatefulWidget {
  const StatisticsScreen({super.key});

  @override
  State<StatisticsScreen> createState() => _StatisticsScreenState();
}

class _StatisticsScreenState extends State<StatisticsScreen> {
  final DatabaseService _databaseService = DatabaseService.instance;
  DateTimeRange _selectedRange = DateTimeRange(
    start: DateTime.now().subtract(const Duration(days: 30)),
    end: DateTime.now(),
  );
  List<RainRecord> _filteredRecords = [];
  String _statsSummary = '';

  @override
  void initState() {
    super.initState();
    _loadRecords();
  }

  Future<void> _loadRecords() async {
    final records = await _databaseService.getRecordsByDateRange(
      _selectedRange.start,
      _selectedRange.end,
    );
    
    _calculateStatistics(records);
    
    setState(() {
      _filteredRecords = records;
    });
  }

  void _calculateStatistics(List<RainRecord> records) {
    if (records.isEmpty) {
      setState(() {
        _statsSummary = 'Nenhum dado disponível para o período selecionado';
      });
      return;
    }

    final total = records.fold(0.0, (sum, record) => sum + record.millimeters);
    final average = total / records.length;
    final maxRainDay = records.reduce((a, b) => a.millimeters > b.millimeters ? a : b);
    
    int maxDryDays = 0;
    int currentDryDays = 0;
    DateTime? lastRainDate;
    
    records.sort((a, b) => a.date.compareTo(b.date));
    
    for (final record in records) {
      if (record.millimeters == 0) {
        currentDryDays++;
        if (currentDryDays > maxDryDays) {
          maxDryDays = currentDryDays;
        }
      } else {
        lastRainDate = record.date;
        currentDryDays = 0;
      }
    }
    
    setState(() {
      _statsSummary = '''
Período: ${DateFormat('dd/MM/yyyy').format(_selectedRange.start)} - ${DateFormat('dd/MM/yyyy').format(_selectedRange.end)}
Total de chuva: ${total.toStringAsFixed(1)} mm
Média diária: ${average.toStringAsFixed(1)} mm
Dia com maior chuva: ${DateFormat('dd/MM/yyyy').format(maxRainDay.date)} (${maxRainDay.millimeters} mm)
Maior período de estiagem: $maxDryDays dias
Última chuva: ${lastRainDate != null ? DateFormat('dd/MM/yyyy').format(lastRainDate) : 'Nenhuma chuva registrada'}
''';
    });
  }

  Future<void> _selectDateRange(BuildContext context) async {
    final DateTimeRange? picked = await showDateRangePicker(
      context: context,
      firstDate: DateTime(2000),
      lastDate: DateTime.now(),
      initialDateRange: _selectedRange,
    );
    if (picked != null && picked != _selectedRange) {
      setState(() {
        _selectedRange = picked;
      });
      _loadRecords();
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Estatísticas de Chuva'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            Row(
              children: [
                Expanded(
                  child: Text(
                    '${DateFormat('dd/MM/yyyy').format(_selectedRange.start)} - ${DateFormat('dd/MM/yyyy').format(_selectedRange.end)}',
                    style: const TextStyle(fontSize: 16),
                  ),
                ),
                TextButton(
                  onPressed: () => _selectDateRange(context),
                  child: const Text('Alterar Período'),
                ),
              ],
            ),
            const SizedBox(height: 20),
            Expanded(
              flex: 2,
              child: _buildRainChart(),
            ),
            const SizedBox(height: 20),
            Expanded(
              child: SingleChildScrollView(
                child: Text(
                  _statsSummary,
                  style: const TextStyle(fontSize: 16),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildRainChart() {
    if (_filteredRecords.isEmpty) {
      return const Center(child: Text('Nenhum dado disponível para exibir o gráfico'));
    }

    _filteredRecords.sort((a, b) => a.date.compareTo(b.date));

    final series = [
      charts.Series<RainRecord, DateTime>(
        id: 'Chuva',
        colorFn: (_, __) => charts.MaterialPalette.blue.shadeDefault,
        domainFn: (RainRecord record, _) => record.date,
        measureFn: (RainRecord record, _) => record.millimeters,
        data: _filteredRecords,
      )
    ];

    return charts.TimeSeriesChart(
      series,
      animate: true,
      dateTimeFactory: const charts.LocalDateTimeFactory(),
      primaryMeasureAxis: const charts.NumericAxisSpec(
        tickProviderSpec: charts.BasicNumericTickProviderSpec(
          desiredMinTickCount: 5,
          desiredMaxTickCount: 10,
        ),
        renderSpec: charts.GridlineRendererSpec(
          labelStyle: charts.TextStyleSpec(
            fontSize: 12,
            color: charts.MaterialPalette.black,
          ),
        ),
      ),
      domainAxis: const charts.DateTimeAxisSpec(
        tickProviderSpec: charts.DayTickProviderSpec(
          increments: [7],
        ),
        renderSpec: charts.SmallTickRendererSpec(
          labelStyle: charts.TextStyleSpec(
            fontSize: 10,
            color: charts.MaterialPalette.black,
          ),
        ),
      ),
    );
  }
}
import 'package:flutter/material.dart';
import 'package:pdf/pdf.dart';
import 'package:pdf/widgets.dart' as pw;
import 'package:printing/printing.dart';
import 'package:intl/intl.dart';
import '../models/rain_record.dart';
import '../services/database_service.dart';

class ReportsScreen extends StatefulWidget {
  const ReportsScreen({super.key});

  @override
  State<ReportsScreen> createState() => _ReportsScreenState();
}

class _ReportsScreenState extends State<ReportsScreen> {
  final DatabaseService _databaseService = DatabaseService.instance;
  DateTimeRange _selectedRange = DateTimeRange(
    start: DateTime.now().subtract(const Duration(days: 30)),
    end: DateTime.now(),
  );
  List<RainRecord> _filteredRecords = [];

  @override
  void initState() {
    super.initState();
    _loadRecords();
  }

  Future<void> _loadRecords() async {
    final records = await _databaseService.getRecordsByDateRange(
      _selectedRange.start,
      _selectedRange.end,
    );
    setState(() {
      _filteredRecords = records;
    });
  }

  Future<void> _selectDateRange(BuildContext context) async {
    final DateTimeRange? picked = await showDateRangePicker(
      context: context,
      firstDate: DateTime(2000),
      lastDate: DateTime.now(),
      initialDateRange: _selectedRange,
    );
    if (picked != null && picked != _selectedRange) {
      setState(() {
        _selectedRange = picked;
      });
      _loadRecords();
    }
  }

  Future<pw.Document> _generatePdf() async {
    final pdf = pw.Document();

    final total = _filteredRecords.fold(0.0, (sum, record) => sum + record.millimeters);
    final average = _filteredRecords.isNotEmpty ? total / _filteredRecords.length : 0;
    final maxRainDay = _filteredRecords.isNotEmpty
        ? _filteredRecords.reduce((a, b) => a.millimeters > b.millimeters ? a : b)
        : null;

    pdf.addPage(
      pw.MultiPage(
        pageFormat: PdfPageFormat.a4,
        build: (context) => [
          pw.Header(
            level: 0,
            child: pw.Text(
              'Relatório Pluviométrico',
              style: pw.TextStyle(fontSize: 24, fontWeight: pw.FontWeight.bold),
            ),
          ),
          pw.Paragraph(
            text:
                'Período: ${DateFormat('dd/MM/yyyy').format(_selectedRange.start)} - ${DateFormat('dd/MM/yyyy').format(_selectedRange.end)}',
          ),
          pw.SizedBox(height: 20),
          pw.Header(
            level: 1,
            child: pw.Text('Resumo Estatístico'),
          ),
          pw.Paragraph(
            text: '''
Total de chuva: ${total.toStringAsFixed(1)} mm
Média diária: ${average.toStringAsFixed(1)} mm
Dia com maior chuva: ${maxRainDay != null ? '${DateFormat('dd/MM/yyyy').format(maxRainDay.date)} (${maxRainDay.millimeters} mm)' : 'Nenhum registro'}
Número de dias com registro: ${_filteredRecords.length}
''',
          ),
          pw.SizedBox(height: 20),
          pw.Header(
            level: 1,
            child: pw.Text('Registros Diários'),
          ),
          pw.Table.fromTextArray(
            context: context,
            data: [
              ['Data', 'Milímetros', 'Observações'],
              ..._filteredRecords.map((record) => [
                    DateFormat('dd/MM/yyyy').format(record.date),
                    record.millimeters.toString(),
                    record.notes ?? '',
                  ]),
            ],
          ),
        ],
      ),
    );

    return pdf;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Gerar Relatório'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            Row(
              children: [
                Expanded(
                  child: Text(
                    '${DateFormat('dd/MM/yyyy').format(_selectedRange.start)} - ${DateFormat('dd/MM/yyyy').format(_selectedRange.end)}',
                    style: const TextStyle(fontSize: 16),
                  ),
                ),
                TextButton(
                  onPressed: () => _selectDateRange(context),
                  child: const Text('Alterar Período'),
                ),
              ],
            ),
            const SizedBox(height: 20),
            Expanded(
              child: _filteredRecords.isEmpty
                  ? const Center(child: Text('Nenhum registro encontrado'))
                  : PdfPreview(
                      build: (format) => _generatePdf(),
                    ),
            ),
          ],
        ),
      ),
    );
  }
}
│   ├── models/
class RainRecord {
  final DateTime date;
  final double millimeters;
  final String? notes;

  RainRecord({
    required this.date,
    required this.millimeters,
    this.notes,
  });

  Map<String, dynamic> toMap() {
    return {
      'date': date.toIso8601String(),
      'millimeters': millimeters,
      'notes': notes,
    };
  }

  factory RainRecord.fromMap(Map<String, dynamic> map) {
    return RainRecord(
      date: DateTime.parse(map['date']),
      millimeters: map['millimeters'],
      notes: map['notes'],
    );
  }
}
│   └── services/
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import '../models/rain_record.dart';

class DatabaseService {
  static final DatabaseService instance = DatabaseService._init();
  static Database? _database;

  DatabaseService._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('rain_records.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);

    return await openDatabase(
      path,
      version: 1,
      onCreate: _createDB,
    );
  }

  Future _createDB(Database db, int version) async {
    await db.execute('''
      CREATE TABLE rain_records (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        date TEXT NOT NULL,
        millimeters REAL NOT NULL,
        notes TEXT
      )
    ''');
  }

  Future<int> insertRecord(RainRecord record) async {
    final db = await instance.database;
    return await db.insert('rain_records', record.toMap());
  }

  Future<List<RainRecord>> getAllRecords() async {
    final db = await instance.database;
    final maps = await db.query('rain_records');
    return maps.map((map) => RainRecord.fromMap(map)).toList();
  }

  Future<List<RainRecord>> getRecordsByDateRange(DateTime start, DateTime end) async {
    final db = await instance.database;
    final maps = await db.query(
      'rain_records',
      where: 'date BETWEEN ? AND ?',
      whereArgs: [start.toIso8601String(), end.toIso8601String()],
    );
    return maps.map((map) => RainRecord.fromMap(map)).toList();
  }

  Future<void> close() async {
    final db = await instance.database;
    db.close();
  }
}
.gitignore
# Flutter
**/android/**/gradle-wrapper.jar
**/android/.gradle
**/android/captures/
**/android/gradlew
**/android/gradlew.bat
**/android/local.properties
**/android/**/GeneratedPluginRegistrant.java
**/ios/**
**/ios/Flutter/App.framework
**/ios/Flutter/Flutter.framework
**/ios/Flutter/Generated.xcconfig
**/ios/Flutter/app.flx
**/ios/Flutter/app.zip
**/ios/Flutter/flutter_assets/
**/ios/Flutter/flutter_export_environment.sh
**/ios/ServiceDefinitions.json
**/ios/Runner/GeneratedPluginRegistrant.*

# Android Studio
.idea/
.gradle/
*.iml
local.properties
.DS_Store

# VS Code
.vscode/

# Files
*.swp
*.swo
*~
*.lock
*.log
*.pyc
*.pub-cache/
*.swiftpm/
*.symlinks/
*.tags*
*.version
*Icon?
*/build/
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/seu-usuario/seu-repositorio.git
git push -u origin main

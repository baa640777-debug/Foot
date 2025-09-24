// Flutter app: Football Scores — Scores Foot (APK-ready) // Fichier: main.dart // But: you need Flutter SDK to build (flutter build apk) and an API key from football-data.org // This single-file example includes: //  - Listing de matches (live / today / finished) //  - Page détail match (buteurs, statut, minute) //  - Mode Démo sans API //  - Design inspiré Sofascore (timeline simplifiée, couleurs, onglets) //  - Instructions pubspec.yaml et build en commentaires en bas

import 'dart:convert'; import 'package:flutter/material.dart'; import 'package:http/http.dart' as http;

void main() => runApp(FootballScoresApp());

class FootballScoresApp extends StatelessWidget { @override Widget build(BuildContext context) { return MaterialApp( title: 'Scores Foot', theme: ThemeData( primarySwatch: Colors.blue, scaffoldBackgroundColor: Colors.grey[100], ), home: ScoresHomePage(), ); } }

class ScoresHomePage extends StatefulWidget { @override _ScoresHomePageState createState() => _ScoresHomePageState(); }

class _ScoresHomePageState extends State<ScoresHomePage> with SingleTickerProviderStateMixin { bool _loading = false; String _error = ''; List<MatchItem> _matches = [];

// ----- CONFIGUREZ ICI VOTRE API ----- // Exemple: football-data.org (https://www.football-data.org/) // Endpoint: https://api.football-data.org/v2/matches // Header: 'X-Auth-Token': 'YOUR_API_KEY' final String API_ENDPOINT = 'https://api.football-data.org/v2/matches'; final String API_KEY = 'YOUR_API_KEY_HERE'; // ------------------------------------

late TabController _tabController;

@override void initState() { super.initState(); _tabController = TabController(length: 3, vsync: this); _fetchScores(); }

Future<void> _fetchScores({String filter = 'TODAY'}) async { setState(() { _loading = true; _error = ''; });

try {
  if (API_ENDPOINT.contains('example.com') || API_KEY.contains('YOUR_API_KEY_HERE')) {
    // If user hasn't configured API, use mock data
    await Future.delayed(Duration(milliseconds: 500));
    _loadMockData();
    return;
  }

  // Build query - football-data.org supports dateFrom/dateTo, competitions, etc.
  Uri uri = Uri.parse(API_ENDPOINT);

  final response = await http.get(uri, headers: {
    'X-Auth-Token': API_KEY,
    'Accept': 'application/json',
  }).timeout(Duration(seconds: 12));

  if (response.statusCode == 200) {
    final data = json.decode(response.body);
    List<MatchItem> loaded = [];
    if (data is Map && data['matches'] != null) {
      for (var m in data['matches']) {
        loaded.add(MatchItem.fromJson(m));
      }
    }

    setState(() {
      _matches = loaded;
      _loading = false;
    });
  } else {
    setState(() {
      _error = 'Erreur serveur: ${response.statusCode}';
      _loading = false;
    });
  }
} catch (e) {
  setState(() {
    _error = 'Erreur: ${e.toString()}';
    _loading = false;
  });
}

}

void _loadMockData() { setState(() { _matches = [ MatchItem( id: 1, utcDate: DateTime.now().subtract(Duration(hours: 2)), homeTeam: 'Paris FC', awayTeam: 'Lyon FC', status: 'FINISHED', homeScore: 2, awayScore: 1, competition: 'Ligue 1', minute: 90, events: [ MatchEvent(minute: 12, type: 'GOAL', team: 'Paris FC', player: 'M. Dupont'), MatchEvent(minute: 55, type: 'GOAL', team: 'Lyon FC', player: 'A. Traore'), MatchEvent(minute: 78, type: 'GOAL', team: 'Paris FC', player: 'K. Mbappé'), ], ), MatchItem( id: 2, utcDate: DateTime.now(), homeTeam: 'Barca', awayTeam: 'Real', status: 'LIVE', homeScore: 1, awayScore: 0, competition: 'La Liga', minute: 63, events: [ MatchEvent(minute: 21, type: 'GOAL', team: 'Barca', player: 'L. Messi'), MatchEvent(minute: 61, type: 'YELLOW_CARD', team: 'Real', player: 'Sergio'), ], ), MatchItem( id: 3, utcDate: DateTime.now().add(Duration(hours: 5)), homeTeam: 'AC Milan', awayTeam: 'Inter', status: 'SCHEDULED', homeScore: 0, awayScore: 0, competition: 'Serie A', minute: 0, events: [], ), ];

_loading = false;
});

}

@override void dispose() { _tabController.dispose(); super.dispose(); }

@override Widget build(BuildContext context) { return Scaffold( appBar: AppBar( title: Text('Résultats Foot'), bottom: TabBar( controller: _tabController, tabs: [ Tab(text: 'En direct'), Tab(text: 'Aujourd'hui'), Tab(text: 'À venir'), ], ), actions: [ IconButton( icon: Icon(Icons.refresh), onPressed: () => _fetchScores(), tooltip: 'Rafraîchir', ), ], ), body: TabBarView( controller: _tabController, children: [ _buildListView(filter: 'LIVE'), _buildListView(filter: 'TODAY'), _buildListView(filter: 'SCHEDULED'), ], ), floatingActionButton: FloatingActionButton.extended( onPressed: () { _loadMockData(); ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Mode Démo activé'))); }, label: Text('Démo'), icon: Icon(Icons.visibility), ), ); }

Widget _buildListView({required String filter}) { List<MatchItem> list; final now = DateTime.now(); if (filter == 'LIVE') { list = _matches.where((m) => m.status.toUpperCase().contains('LIVE') || m.status.toUpperCase().contains('IN_PLAY') ).toList(); } else if (filter == 'SCHEDULED') { list = _matches.where((m) => m.status.toUpperCase().contains('SCHEDULED')).toList(); } else { // TODAY list = _matches.where((m) => m.utcDate.year == now.year && m.utcDate.month == now.month && m.utcDate.day == now.day).toList(); }

return RefreshIndicator(
  onRefresh: () => _fetchScores(),
  child: _loading
      ? Center(child: CircularProgressIndicator())
      : _error.isNotEmpty
          ? ListView(
              physics: AlwaysScrollableScrollPhysics(),
              children: [
                SizedBox(height: 40),
                Icon(Icons.error_outline, size: 48),
                Padding(
                  padding: const EdgeInsets.all(16.0),
                  child: Text(
                    _error,
                    textAlign: TextAlign.center,
                    style: TextStyle(fontSize: 16),
                  ),
                )
              ],
            )
          : list.isEmpty
              ? ListView(
                  physics: AlwaysScrollableScrollPhysics(),
                  children: [
                    SizedBox(height: 80),
                    Icon(Icons.sports_soccer, size: 56),
                    Center(child: Text('Aucun match trouvé pour ce filtre')),
                  ],
                )
              : ListView.builder(
                  itemCount: list.length,
                  itemBuilder: (ctx, i) {
                    final m = list[i];
                    return _matchCard(m);
                  },
                ),
);

}

Widget matchCard(MatchItem m) { Color statusColor = m.status.toUpperCase().contains('LIVE') ? Colors.redAccent : Colors.grey; return Card( margin: EdgeInsets.symmetric(horizontal: 12, vertical: 8), shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)), child: InkWell( borderRadius: BorderRadius.circular(12), onTap: () { Navigator.of(context).push(MaterialPageRoute(builder: () => MatchDetailPage(match: m))); }, child: Padding( padding: const EdgeInsets.symmetric(horizontal: 12.0, vertical: 14), child: Row( children: [ Expanded( flex: 5, child: Column( crossAxisAlignment: CrossAxisAlignment.start, children: [ Text(m.competition, style: TextStyle(fontSize: 12, color: Colors.grey[600])), SizedBox(height: 6), Row( children: [ Expanded(child: Text(m.homeTeam, style: TextStyle(fontWeight: FontWeight.w600))), Text('${m.homeScore}', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)), SizedBox(width: 8), Text('-', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)), SizedBox(width: 8), Text('${m.awayScore}', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)), Expanded(child: Container(),) ], ), SizedBox(height: 6), Row( children: [ Icon(Icons.access_time, size: 14, color: Colors.grey[600]), SizedBox(width: 6), Text(_statusLabel(m), style: TextStyle(color: statusColor)), SizedBox(width: 10), if (m.minute > 0) Chip(label: Text('${m.minute}''), backgroundColor: Colors.grey[200]) ], ) ], ), ), // Simple timeline indicator (Sofascore-like vibe) Container( width: 70, child: Column( children: [ _miniTimeline(m), SizedBox(height: 6), Icon(Icons.chevron_right) ], ), ) ], ), ), ), ); }

Widget _miniTimeline(MatchItem m) { // Very simplified: show small icons for events (goals/cards) return Row( mainAxisAlignment: MainAxisAlignment.center, children: m.events.take(3).map((e) { IconData icon = Icons.sports_soccer; if (e.type == 'YELLOW_CARD') icon = Icons.crop_square; if (e.type == 'RED_CARD') icon = Icons.stop_circle_outlined; return Padding( padding: const EdgeInsets.symmetric(horizontal: 2.0), child: Column( children: [ Icon(icon, size: 14), SizedBox(height: 2), Text('${e.minute}'', style: TextStyle(fontSize: 10)) ], ), ); }).toList(), ); }

String _statusLabel(MatchItem m) { final s = m.status.toUpperCase(); if (s.contains('LIVE') || s.contains('IN_PLAY')) return 'En direct'; if (s.contains('FINISHED') || s.contains('FIN')) return 'Terminé'; if (s.contains('SCHEDULED')) return 'À venir'; return m.status; } }

class MatchDetailPage extends StatelessWidget { final MatchItem match; const MatchDetailPage({Key? key, required this.match}) : super(key: key);

@override Widget build(BuildContext context) { return Scaffold( appBar: AppBar(title: Text('${match.homeTeam} vs ${match.awayTeam}')), body: SingleChildScrollView( padding: EdgeInsets.all(12), child: Column( crossAxisAlignment: CrossAxisAlignment.start, children: [ Card( shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)), child: Padding( padding: const EdgeInsets.all(12.0), child: Row( children: [ Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [ Text(match.homeTeam, style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)), SizedBox(height: 6), Text(match.competition, style: TextStyle(color: Colors.grey[600])) ])), Column(children: [ Text('${match.homeScore} - ${match.awayScore}', style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold)), SizedBox(height: 6), Text(match.status) ]), Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.end, children: [ Text(match.awayTeam, textAlign: TextAlign.right, style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)) ])), ], ), ), ), SizedBox(height: 12), Text('Timeline & Événements', style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600)), SizedBox(height: 8), ...match.events.map((e) => ListTile( leading: Icon(e.type == 'GOAL' ? Icons.sports_soccer : Icons.note, size: 28), title: Text('${e.player} (${e.team})'), subtitle: Text('${e.type} • ${e.minute}''), )), if (match.events.isEmpty) Padding(padding: EdgeInsets.symmetric(vertical: 20), child: Center(child: Text('Aucun événement disponible'))), SizedBox(height: 20), Text('Statistiques (simple)', style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600)), SizedBox(height: 8), Card( child: Padding( padding: EdgeInsets.all(12), child: Row( mainAxisAlignment: MainAxisAlignment.spaceAround, children: [ _statColumn('Tirs', '12', '5'), _statColumn('Poss.', '61%', '39%'), _statColumn('Corners', '6', '1'), ], ), ), ) ], ), ), ); }

Widget _statColumn(String name, String homeVal, String awayVal) { return Column( children: [ Text(name, style: TextStyle(fontSize: 12, color: Colors.grey[700])), SizedBox(height: 8), Row( children: [ Text(homeVal, style: TextStyle(fontWeight: FontWeight.bold)), SizedBox(width: 8), Container(width: 60, height: 8, color: Colors.grey[300]), SizedBox(width: 8), Text(awayVal, style: TextStyle(fontWeight: FontWeight.bold)), ], ) ], ); } }

class MatchItem { final int id; final DateTime utcDate; final String homeTeam; final String awayTeam; final int homeScore; final int awayScore; final String status; final String competition; final int minute; final List<MatchEvent> events;

MatchItem({ required this.id, required this.utcDate, required this.homeTeam, required this.awayTeam, required this.homeScore, required this.awayScore, required this.status, required this.competition, required this.minute, required this.events, });

factory MatchItem.fromJson(Map<String, dynamic> json) { // Adapté pour football-data.org structure v2 int id = json['id'] ?? 0; DateTime date = DateTime.tryParse(json['utcDate'] ?? '') ?? DateTime.now(); String home = json['homeTeam'] is Map ? (json['homeTeam']['name'] ?? '') : (json['homeTeam'] ?? ''); String away = json['awayTeam'] is Map ? (json['awayTeam']['name'] ?? '') : (json['awayTeam'] ?? '');

int homeScore = 0;
int awayScore = 0;
try {
  if (json['score'] != null && json['score']['fullTime'] != null) {
    homeScore = json['score']['fullTime']['home'] ?? 0;
    awayScore = json['score']['fullTime']['away'] ?? 0;
  }
} catch (e) {
  // ignore
}

String status = json['status'] ?? 'SCHEDULED';
String comp = json['competition'] is Map ? (json['competition']['name'] ?? '') : (json['competition'] ?? '');
int minute = json['minute'] ?? 0;

// Events are not included in /matches by default in many APIs; keep empty list unless provider gives them
List<MatchEvent> events = [];
if (json['events'] != null && json['events'] is List) {
  for (var e in json['events']) {
    events.add(MatchEvent.fromJson(e));
  }
}

return MatchItem(
  id: id,
  utcDate: date,
  homeTeam: home,
  awayTeam: away,
  homeScore: homeScore,
  awayScore: awayScore,
  status: status,
  competition: comp,
  minute: minute,
  events: events,
);

} }

class MatchEvent { final int minute; final String type; final String team; final String player;

MatchEvent({required this.minute, required this.type, required this.team, required this.player});

factory MatchEvent.fromJson(Map<String, dynamic> json) { return MatchEvent( minute: json['minute'] ?? 0, type: json['type'] ?? 'EVENT', team: json['team'] ?? '', player: json['player'] ?? '', ); } }

/* PUBSPEC.YAML (ajoute dans ton projet Flutter -> pubspec.yaml):

name: scores_foot description: Application simple pour afficher résultats de foot publish_to: 'none' version: 1.0.0+1

environment: sdk: '>=2.17.0 <3.0.0'

dependencies: flutter: sdk: flutter http: ^0.13.6

dev_dependencies: flutter_test: sdk: flutter

flutter: uses-material-design: true


---

INSTRUCTIONS RAPIDES:

1. Installer Flutter: https://flutter.dev/docs/get-started/install


2. Créer un nouveau projet: flutter create scores_foot


3. Remplacer le contenu de lib/main.dart par ce fichier main.dart


4. Copier le bloc pubspec.yaml ci-dessus dans le fichier pubspec.yaml du projet


5. Exécuter: flutter pub get


6. Ouvrir un émulateur ou brancher ton téléphone (mode développeur)


7. Lancer en debug: flutter run


8. Pour générer l'APK: flutter build apk --release



CONFIGURATION DE L'API:

Si tu veux utiliser football-data.org : inscris-toi pour une clé (ils demandent X-Auth-Token). Remplace API_KEY dans le code.

Si tu veux utiliser API-Football (rapidapi): adapte headers et endpoint (Authorization ou X-RapidAPI-Key). Je peux adapter le code si tu me donnes le nom exact de l'API et la façon d'envoyer la clé.


CE QUE J'AI AJOUTÉ pour toi (option "Tout"):

1. Parsing et structure adaptée à football-data.org (header X-Auth-Token).


2. Page détail match avec timeline et événements.


3. Style et composants inspirés de Sofascore (timeline mini, chips, onglets compétitions/live/today).


4. Pubspec.yaml d'exemple et instructions de build.



SI TU VEUX QUE JE FASSE PLUS:

Je peux adapter précisément au fournisseur que tu possèdes (donne le nom: api-football.com, sportdataapi, rapidapi "api-football", etc.) et je mettrai les headers + parsing exacts.

Ajouter authentification utilisateur, favoris (matches/équipes), notifications push (génère de la complexité, nécessite Firebase), ou un backend pour proxy la clé API.


*/


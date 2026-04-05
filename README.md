import 'dart:convert';
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() => runApp(MrWolferApp());

class MrWolferApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData.dark(),
      home: PlayerListScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}

// ----------------- MODEL -----------------
class PlayerProfile {
  String realName;
  String gameName;
  String uid;
  String role;
  String previousTeam;
  String experience;
  String phone;
  String activeTime;
  String age;
  String zilla;
  String upaz;
  String? imagePath;

  PlayerProfile({
    required this.realName,
    required this.gameName,
    required this.uid,
    required this.role,
    required this.previousTeam,
    required this.experience,
    required this.phone,
    required this.activeTime,
    required this.age,
    required this.zilla,
    required this.upaz,
    this.imagePath,
  });

  Map<String, dynamic> toMap() => {
        'realName': realName,
        'gameName': gameName,
        'uid': uid,
        'role': role,
        'previousTeam': previousTeam,
        'experience': experience,
        'phone': phone,
        'activeTime': activeTime,
        'age': age,
        'zilla': zilla,
        'upaz': upaz,
        'imagePath': imagePath,
      };

  factory PlayerProfile.fromMap(Map<String, dynamic> map) => PlayerProfile(
        realName: map['realName'],
        gameName: map['gameName'],
        uid: map['uid'],
        role: map['role'],
        previousTeam: map['previousTeam'],
        experience: map['experience'],
        phone: map['phone'],
        activeTime: map['activeTime'],
        age: map['age'],
        zilla: map['zilla'],
        upaz: map['upaz'],
        imagePath: map['imagePath'],
      );
}

// ----------------- PLAYER LIST SCREEN -----------------
class PlayerListScreen extends StatefulWidget {
  @override
  _PlayerListScreenState createState() => _PlayerListScreenState();
}

class _PlayerListScreenState extends State<PlayerListScreen> {
  List<PlayerProfile> players = [];
  List<PlayerProfile> filteredPlayers = [];
  TextEditingController _searchController = TextEditingController();
  String _sortOption = "Game Name";

  @override
  void initState() {
    super.initState();
    loadPlayers();
    _searchController.addListener(_filterAndSortPlayers);
  }

  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }

  Future<void> loadPlayers() async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
    List<String>? saved = prefs.getStringList('players');
    if (saved != null) {
      players = saved.map((e) => PlayerProfile.fromMap(json.decode(e))).toList();
    }
    _filterAndSortPlayers();
  }

  void _filterAndSortPlayers() {
    String query = _searchController.text.toLowerCase();
    List<PlayerProfile> tempList = players.where((p) {
      return p.gameName.toLowerCase().contains(query) ||
          p.role.toLowerCase().contains(query) ||
          p.uid.toLowerCase().contains(query);
    }).toList();

    if (_sortOption == "Game Name") {
      tempList.sort((a, b) => a.gameName.toLowerCase().compareTo(b.gameName.toLowerCase()));
    } else if (_sortOption == "Role") {
      tempList.sort((a, b) => a.role.toLowerCase().compareTo(b.role.toLowerCase()));
    } else if (_sortOption == "Experience") {
      tempList.sort((a, b) => _extractYears(b.experience).compareTo(_extractYears(a.experience)));
    }

    setState(() {
      filteredPlayers = tempList;
    });
  }

  int _extractYears(String exp) {
    RegExp reg = RegExp(r'\d+');
    var match = reg.firstMatch(exp);
    if (match != null) return int.parse(match.group(0)!);
    return 0;
  }

  void _onSortChanged(String? newValue) {
    if (newValue != null) {
      setState(() {
        _sortOption = newValue;
      });
      _filterAndSortPlayers();
    }
  }

  void navigateToCreate({PlayerProfile? editPlayer, int? index}) async {
    final result = await Navigator.push(
      context,
      MaterialPageRoute(
        builder: (_) => CreatePlayerScreen(editPlayer: editPlayer),
      ),
    );

    if (result != null && result is PlayerProfile) {
      SharedPreferences prefs = await SharedPreferences.getInstance();

      if (editPlayer != null && index != null) {
        players[index] = result;
      } else {
        players.add(result);
      }

      List<String> saved = players.map((p) => json.encode(p.toMap())).toList();
      await prefs.setStringList('players', saved);
      _filterAndSortPlayers();
    }
  }

  void deletePlayer(int index) async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
    players.removeAt(index);
    List<String> saved = players.map((p) => json.encode(p.toMap())).toList();
    await prefs.setStringList('players', saved);
    _filterAndSortPlayers();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("MR WOLFER PLAYERS"),
        backgroundColor: Colors.orange[900],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: TextField(
              controller: _searchController,
              style: TextStyle(color: Colors.white),
              decoration: InputDecoration(
                hintText: "Search by Name, Role, or UID",
                hintStyle: TextStyle(color: Colors.grey[400]),
                filled: true,
                fillColor: Colors.grey[850],
                prefixIcon: Icon(Icons.search, color: Colors.orange[300]),
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(10),
                  borderSide: BorderSide.none,
                ),
              ),
            ),
          ),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 8.0),
            child: Row(
              children: [
                Text("Sort by: ", style: TextStyle(color: Colors.orange[200])),
                SizedBox(width: 10),
                DropdownButton<String>(
                  value: _sortOption,
                  dropdownColor: Colors.grey[900],
                  items: ["Game Name", "Role", "Experience"]
                      .map((option) => DropdownMenuItem(
                            value: option,
                            child: Text(option, style: TextStyle(color: Colors.white)),
                          ))
                      .toList(),
                  onChanged: _onSortChanged,
                ),
              ],
            ),
          ),
          Expanded(
            child: filteredPlayers.isEmpty
                ? Center(child: Text("No players found. Tap + to add one.", style: TextStyle(color: Colors.orange[200])))
                : ListView.builder(
                    itemCount: filteredPlayers.length,
                    itemBuilder: (context, index) {
                      final p = filteredPlayers[index];
                      return Card(
                        margin: EdgeInsets.all(8),
                        color: Colors.grey[900],
                        child: ListTile(
                          leading: CircleAvatar(
                            radius: 30,
                            backgroundImage: (p.imagePath != null && File(p.imagePath!).existsSync())
                                ? FileImage(File(p.imagePath!))
                                : AssetImage('assets/wolf_logo.png') as ImageProvider,
                          ),
                          title: Text(p.gameName, style: TextStyle(color: Colors.orange[200])),
                          subtitle: Text("${p.role} | ${p.uid}"),
                          trailing: Row(
                            mainAxisSize: MainAxisSize.min,
                            children: [
                              IconButton(
                                icon: Icon(Icons.edit, color: Colors.greenAccent),
                                onPressed: () => navigateToCreate(editPlayer: p, index: players.indexOf(p)),
                              ),
                              IconButton(
                                icon: Icon(Icons.delete, color: Colors.redAccent),
                                onPressed: () => deletePlayer(players.indexOf(p)),
                              ),
                            ],
                          ),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        backgroundColor: Colors.orange[800],
        child: Icon(Icons.add),
        onPressed: () => navigateToCreate(),
      ),
    );
  }
}

// ----------------- CREATE/EDIT PLAYER SCREEN -----------------
class CreatePlayerScreen extends StatefulWidget {
  final PlayerProfile? editPlayer;

  CreatePlayerScreen({this.editPlayer});

  @override
  _CreatePlayerScreenState createState() => _CreatePlayerScreenState();
}

class _CreatePlayerScreenState extends State<CreatePlayerScreen> {
  final _formKey = GlobalKey<FormState>();
  late PlayerProfile player;
  XFile? pickedImage;

  @override
  void initState() {
    super.initState();
    player = widget.editPlayer ??
        PlayerProfile(
          realName: '',
          gameName: '',
          uid: '',
          role: '',
          previousTeam: '',
          experience: '',
          phone: '',
          activeTime: '',
          age: '',
          zilla: '',
          upaz: '',
        );
  }

  Future<void> pickImage() async {
    final ImagePicker picker = ImagePicker();
    XFile? image = await picker.pickImage(source: ImageSource.gallery);
    if (image != null) {
      setState(() {
        pickedImage = image;
        player.imagePath = image.path;
      });
    }
  }

  void savePlayer() {
    if (_formKey.currentState!.validate()) {
      _formKey.currentState!.save();
      Navigator.pop(context, player);
    }
  }

  Widget buildTextField(String label, String initialValue, Function(String?) onSaved) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 6.0),
      child: TextFormField(
        initialValue: initialValue,
        style: TextStyle(color: Colors.white),
        decoration: InputDecoration(
          labelText: label,
          labelStyle: TextStyle(color: Colors.orange[200]),
          filled: true,
          fillColor: Colors.grey[850],
          border: OutlineInputBorder(borderRadius: BorderRadius.circular(10)),
        ),
        validator: (val) => val == null || val.isEmpty ? 'Required' : null,
        onSaved: onSaved,
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.editPlayer != null ? "Edit Player" : "Add Player"),
        backgroundColor: Colors.orange[900],
      ),
      body: SingleChildScrollView(
        padding: EdgeInsets.all(12),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              GestureDetector(
                onTap: pickImage,
                child: CircleAvatar(
                  radius: 50,
                  backgroundImage: (pickedImage != null)
                      ? FileImage(File(pickedImage!.path))
                      : (player.imagePath != null && File(player.imagePath!).existsSync())
                          ? FileImage(File(player.imagePath!))
                          : AssetImage('assets/wolf_logo.png') as ImageProvider,
                ),
              ),
              SizedBox(height: 12),
              buildTextField("Real Name", player.realName, (v) => player.realName = v!),
              buildTextField("Game Name", player.gameName, (v) => player.gameName = v!),
              buildTextField("UID", player.uid, (v) => player.uid = v!),
              buildTextField("Role", player.role, (v) => player.role = v!),
              buildTextField("Previous Team", player.previousTeam, (v) => player.previousTeam = v!),
              buildTextField("Experience", player.experience, (v) => player.experience = v!),
              buildTextField("Phone", player.phone, (v) => player.phone = v!),
              buildTextField("Active Time", player.activeTime, (v) => player.activeTime = v!),
              buildTextField("Age", player.age, (v) => player.age = v!),
              buildTextField("Zilla", player.zilla, (v) => player.zilla = v!),
              buildTextField("Upazila", player.upaz, (v) => player.upaz = v!),
              SizedBox(height: 12),
              ElevatedButton(
                onPressed: savePlayer,
                style: ElevatedButton.styleFrom(backgroundColor: Colors.orange[800]),
                child: Text("Save", style: TextStyle(color: Colors.white)),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

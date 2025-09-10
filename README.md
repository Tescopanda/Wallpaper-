# Wallpaper-
Motivative Wallpaper 
// main.dart
// Flutter prototype: Motivational Wallpaper App
// Single-file prototype. Shows random AI-style wallpapers with quote overlays,
// allows custom quotes, categories, save/share, and (Android) set-as-wallpaper.
// Notes:
// - This prototype uses placeholder network images. Replace image URLs with
//   AI-generated images from your preferred API (Stable Diffusion, DALL·E, etc.).
// - To set wallpaper on Android, add and configure a package like
//   wallpaper_manager_flutter (Android-only). Setting wallpaper on iOS
//   requires platform-specific code and is intentionally left as a download option.
// - Daily automatic wallpaper refresh is stubbed using shared_preferences. For
//   background scheduling, integrate WorkManager / android_alarm_manager / BackgroundFetch.


import 'dart:io';
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:http/http.dart' as http;
import 'package:share_plus/share_plus.dart';


void main() {
runApp(const MotivationApp());
}


class MotivationApp extends StatelessWidget {
const MotivationApp({Key? key}) : super(key: key);


@override
Widget build(BuildContext context) {
return MaterialApp(
title: 'Daily Motivation',
theme: ThemeData.dark(),
home: const HomePage(),
);
}
}


class WallpaperItem {
final String imageUrl;
final String quote;
final String author;
final String category;


WallpaperItem({
required this.imageUrl,
required this.quote,
this.author = '',
this.category = 'General',
});
}


class HomePage extends StatefulWidget {
const HomePage({Key? key}) : super(key: key);


@override
State createState() => _HomePageState();
}


class _HomePageState extends State {
final Random _rand = Random();
late List _items;
late WallpaperItem _current;
final TextEditingController _customQuoteController = TextEditingController();
String _selectedCategory = 'All';
bool _dailyAuto = false;


final List _categories = [
'All',
'Calm',
'Focus',
'Luxury',
'Gym',
'Abstract',
];


@override
void initState() {
super.initState();
_items = _buildPlaceholderItems();
_current = _items[0];
_loadPrefs();
}


List _buildPlaceholderItems() {
// Replace these placeholder image URLs with AI-generated images or your own CDN.
return [
WallpaperItem(
imageUrl: 'https://images.unsplash.com/photo-1507525428034-b723cf961d3e?w=1600',
quote: 'Discipline is the bridge between goals and accomplishment.',
author: 'Jim Rohn',
category: 'Focus',
),
WallpaperItem(
imageUrl: 'https://images.unsplash.com/photo-1503264116251-35a269479413?w=1600',
quote: 'You want to be the person who keeps showing up.',
author: 'Steve Harvey',
category: 'Calm',
),
WallpaperItem(
imageUrl: 'https://images.unsplash.com/photo-1496307042754-b4aa456c4a2d?w=1600',
quote: 'Small daily improvements are the key to staggering long-term results.',
author: 'Unknown',
category: 'Gym',
),
WallpaperItem(
imageUrl: 'https://images.unsplash.com/photo-1496307042754-b4aa456c4a2d?w=1600',
quote: 'Focus on being productive instead of busy.',
author: '',
category: 'Focus',
),
WallpaperItem(
imageUrl: 'https://images.unsplash.com/photo-1432821596592-e2c18b78144f?w=1600',
quote: 'Dream big. Start small. Act now.',
author: '',
category: 'Luxury',
),
];
}


Future _loadPrefs() async {
final prefs = await SharedPreferences.getInstance();
setState(() {
_dailyAuto = prefs.getBool('dailyAuto') ?? false;
final savedIndex = prefs.getInt('lastIndex') ?? 0;
if (savedIndex >= 0 && savedIndex < _items.length) {
_current = _items[savedIndex];
}
});
}


Future _savePrefs() async {
final prefs = await SharedPreferences.getInstance();
final idx = _items.indexOf(_current);
await prefs.setBool('dailyAuto', _dailyAuto);
await prefs.setInt('lastIndex', idx);
}


void _randomize() {
final pool = (_selectedCategory == 'All')
? _items
: _items.where((i) => i.category == _selectedCategory).toList();
if (pool.isEmpty) return;
setState(() {
_current = pool[_rand.nextInt(pool.length)];
});
_savePrefs();
}


void _useCustomQuote() {
final text = _customQuoteController.text.trim();
if (text.isEmpty) return;
setState(() {
_current = WallpaperItem(
imageUrl: _current.imageUrl,
quote: text,
author: 'You',
category: _current.category,
);
});
_customQuoteController.clear();
_savePrefs();
}


Future _downloadAndShare() async {
try {
if (await _requestStoragePermission() == false) return;
final bytes = await _fetchImageBytes(_current.imageUrl);
final file = await saveBytesToFile(bytes, 'motivation${DateTime.now().millisecondsSinceEpoch}.jpg');
await Share.shareFiles([file.path], text: '${_current.quote} — ${_current.author}');
} catch (e) {
_showSnack('Error sharing image: $e');
}
}


Future _downloadToGallery() async {
try {
if (await _requestStoragePermission() == false) return;
final bytes = await _fetchImageBytes(_current.imageUrl);
final file = await saveBytesToFile(bytes, 'motivation${DateTime.now().millisecondsSinceEpoch}.jpg');
_showSnack('Downloaded to: ${file.path}');
} catch (e) {
_showSnack('Error downloading image: $e');
}
}


Future _requestStoragePermission() async {
if (Platform.isAndroid) {
final status = await Permission.storage.request();
return status.isGranted;
}
return true; // iOS: handled via platform UI when needed
}


Future<List> _fetchImageBytes(String url) async {
final res = await http.get(Uri.parse(url));
if (res.statusCode != 200) throw Exception('Failed to fetch image');
return res.bodyBytes;
}


Future _saveBytesToFile(List bytes, String filename) async {
final dir = await getTemporaryDirectory();
final file = File('${dir.path}/$filename');
await file.writeAsBytes(bytes);
return file;
}


void _showSnack(String message) {
ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(message)));
}


Widget _buildTopBar() {
return Row(
children: [
Expanded(
child: DropdownButtonFormField(
value: _selectedCategory,
items: _categories.map((c) => DropdownMenuItem(value: c, child: Text(c))).toList(),
onChanged: (v) {
if (v == null) return;
setState(() {
_selectedCategory = v;
});
},
decoration: const InputDecoration(border: OutlineInputBorder(), labelText: 'Category'),
),
),
const SizedBox(width: 8),
IconButton(
icon: const Icon(Icons.shuffle),
onPressed: _randomize,
tooltip: 'Randomize',
),
],
);
}


@override
Widget build(BuildContext context) {
return Scaffold(
body: SafeArea(
child: Column(
children: [
Padding(
padding: const EdgeInsets.all(12.0),
child: _buildTopBar(),
),
Expanded(
child: Stack(
fit: StackFit.expand,
children: [
// Background image
Positioned.fill(
child: _NetworkImageWithPlaceholder(url: _current.imageUrl),
),


              // Dark overlay for readability
              Positioned.fill(
                child: Container(
                  color: Colors.black.withOpacity(0.45),
                ),
              ),

              // Quote text
              Positioned(
                left: 24,
                right: 24,
                bottom: 120,
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      '"' + _current.quote + '"',
                      style: const TextStyle(fontSize: 26, fontWeight: FontWeight.bold),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      _current.author.isEmpty ? '' : '- ' + _current.author,
                      style: const TextStyle(fontSize: 16, fontStyle: FontStyle.italic),
                    ),
                  ],
                ),
              ),

              // Bottom controls
              Positioned(
                left: 12,
                right: 12,
                bottom: 12,
                child: Row(
                  children: [
                    ElevatedButton.icon(
                      onPressed: _downloadToGallery,
                      icon: const Icon(Icons.download),
                      label: const Text('Download'),
                    ),
                    const SizedBox(width: 8),
                    ElevatedButton.icon(
                      onPressed: _downloadAndShare,
                      icon: const Icon(Icons.share),
                      label: const Text('Share'),
                    ),
                    const SizedBox(width: 8),
                    Expanded(
                      child: ElevatedButton.icon(
                        onPressed: () {
                          _showSnack('Set wallpaper: Android-only action. See code comments.');
                        },
                        icon: const Icon(Icons.wallpaper),
                        label: const Text('Set Wallpaper'),
                      ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),

        // Custom quote input + toggle
        Padding(
          padding: const EdgeInsets.all(12.0),
          child: Column(
            children: [
              Row(
                children: [
                  Expanded(
                    child: TextField(
                      controller: _customQuoteController,
                      decoration: const InputDecoration(hintText: 'Type your own motivational message'),
                    ),
                  ),
                  const SizedBox(width: 8),
                  ElevatedButton(onPressed: _useCustomQuote, child: const Text('Use')),
                ],
              ),
              const SizedBox(height: 8),
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Row(
                    children: [
                      const Text('Daily auto-refresh'),
                      Switch(
                        value: _dailyAuto,
                        onChanged: (v) async {
                          setState(() => _dailyAuto = v);
                          await _savePrefs();
                          _showSnack(v ? 'Daily auto-refresh ON (setup scheduler to run in background)' : 'Daily auto-refresh OFF');
                        },
                      ),
                    ],
                  ),
                  Text('Preview: ' + _current.category),
                ],
              )
            ],
          ),
        ),
      ],
    ),
  ),
);



}
}


class _NetworkImageWithPlaceholder extends StatelessWidget {
final String url;
const _NetworkImageWithPlaceholder({Key? key, required this.url}) : super(key: key);


@override
Widget build(BuildContext context) {
return Image.network(
url,
fit: BoxFit.cover,
loadingBuilder: (context, child, progress) {
if (progress == null) return child;
return Container(
color: Colors.black12,
child: const Center(child: CircularProgressIndicator()),
);
},
errorBuilder: (context, error, stack) {
return Container(color: Colors.grey[900], child: const Center(child: Icon(Icons.broken_image, size: 80)));
},
);
}
}


/*
HOW TO EXTEND:




AI IMAGE GENERATION






Create a server endpoint that calls an image-generation API (Stable Diffusion / DALL·E / Midjourney via your workflow).


Endpoint should accept "prompt" and "style/category" and return a hosted image URL or image bytes.


In the app, call your endpoint to fetch a new image when user selects a category or taps Randomize.






SETTING WALLPAPER (Android)






Add wallpaper_manager_flutter or equivalent package.


Example (Android only):




import 'package:wallpaper_manager_flutter/wallpaper_manager_flutter.dart';


Future setAndroidWallpaper(File file) async {
final location = WallpaperManagerFlutter.HOME_SCREEN; // or LOCK_SCREEN
await WallpaperManagerFlutter().setwallpaperfromFile(file.path, location);
}




Call this after downloadToGallery and inform the user.






DAILY SCHEDULER






Use workmanager or android_alarm_manager_plus to schedule a background task that fetches a new image each day and sets it.


On iOS, background tasks are limited—consider using silent push notifications or letting the user open the app to refresh.






IMPROVE DESIGN






Add typography customization (font family, size, alignment).


Add templates (big-word minimal, two-line quotes, lower caption style).


Let users choose between static and animated backgrounds (Lottie or short looping videos).






STORAGE & CLOUD






Use Firebase Storage / Supabase to store generated images and a Firestore collection for quotes.


Allow users to like/save wallpapers to a personal gallery in the cloud.






LEGAL / QUOTE ATTRIBUTION






Ensure you store author/source when using famous quotes (Steve Harvey, Les Brown). Check licensing for quote collections.




DEPENDENCIES (add to pubspec.yaml):


dependencies:
flutter:
sdk: flutter
shared_preferences: ^2.0.15
http: ^0.13.5
path_provider: ^2.0.11
permission_handler: ^10.2.0
share_plus: ^6.3.0


dev_dependencies:
flutter_test:
sdk: flutter


*/


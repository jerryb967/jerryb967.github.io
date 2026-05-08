# Flutter Continuous Photo Strip + Slide-Out Text Guide

This guide outlines one practical way to build:

- **Vertical scrolling**: an endless/continuous strip of photos.
- **Horizontal drag** on each photo row: reveal text for that photo.
- **External flat-file content source**: photo URLs + text loaded from JSON/CSV.

## 1) Data format (flat file)

Use JSON first (easiest in Flutter):

```json
[
  {
    "id": "1",
    "imageUrl": "https://example.com/images/one.jpg",
    "title": "Sunrise Ridge",
    "description": "Shot at 6:10 AM from the east trail overlook."
  },
  {
    "id": "2",
    "imageUrl": "https://example.com/images/two.jpg",
    "title": "City Rain",
    "description": "Long exposure with reflections across 5th Avenue."
  }
]
```

You can host this as:

- a static file in your Flutter `assets/` folder, or
- a remote URL (CDN, S3, GitHub raw, etc.) so you can update content without app redeploy.

---

## 2) Core UI pattern

Use a `CustomScrollView` + `SliverList` for smooth long scrolling.

Each row is a **swipe-reveal card**:

- Foreground: image.
- Background: text panel.
- Horizontal drag moves foreground to expose text.

For this, `flutter_slidable` is the fastest path.

### Recommended packages

- `cached_network_image` (network images + caching)
- `flutter_slidable` (left/right reveal interaction)
- `http` (if loading remote JSON)

---

## 3) Data model

```dart
class PhotoItem {
  final String id;
  final String imageUrl;
  final String title;
  final String description;

  PhotoItem({
    required this.id,
    required this.imageUrl,
    required this.title,
    required this.description,
  });

  factory PhotoItem.fromJson(Map<String, dynamic> json) {
    return PhotoItem(
      id: json['id'] as String,
      imageUrl: json['imageUrl'] as String,
      title: json['title'] as String,
      description: json['description'] as String,
    );
  }
}
```

---

## 4) Load from external file

### A) From bundled asset

```dart
import 'dart:convert';
import 'package:flutter/services.dart' show rootBundle;

Future<List<PhotoItem>> loadFromAsset() async {
  final raw = await rootBundle.loadString('assets/photos.json');
  final List data = jsonDecode(raw) as List;
  return data.map((e) => PhotoItem.fromJson(e)).toList();
}
```

### B) From remote URL

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

Future<List<PhotoItem>> loadFromUrl(String url) async {
  final res = await http.get(Uri.parse(url));
  if (res.statusCode != 200) throw Exception('Failed to load feed');
  final List data = jsonDecode(res.body) as List;
  return data.map((e) => PhotoItem.fromJson(e)).toList();
}
```

---

## 5) Screen structure

```dart
class PhotoStripScreen extends StatefulWidget {
  const PhotoStripScreen({super.key});

  @override
  State<PhotoStripScreen> createState() => _PhotoStripScreenState();
}

class _PhotoStripScreenState extends State<PhotoStripScreen> {
  late Future<List<PhotoItem>> _future;

  @override
  void initState() {
    super.initState();
    _future = loadFromUrl('https://example.com/photos.json');
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: FutureBuilder<List<PhotoItem>>(
        future: _future,
        builder: (context, snapshot) {
          if (snapshot.connectionState != ConnectionState.done) {
            return const Center(child: CircularProgressIndicator());
          }
          if (snapshot.hasError) {
            return Center(child: Text('Error: ${snapshot.error}'));
          }

          final items = snapshot.data ?? [];

          return CustomScrollView(
            slivers: [
              SliverList.builder(
                itemCount: items.length,
                itemBuilder: (context, index) {
                  return PhotoSlidableRow(item: items[index]);
                },
              ),
            ],
          );
        },
      ),
    );
  }
}
```

---

## 6) Swipe-left to reveal text

```dart
import 'package:flutter_slidable/flutter_slidable.dart';
import 'package:cached_network_image/cached_network_image.dart';

class PhotoSlidableRow extends StatelessWidget {
  final PhotoItem item;
  const PhotoSlidableRow({super.key, required this.item});

  @override
  Widget build(BuildContext context) {
    return Slidable(
      key: ValueKey(item.id),
      endActionPane: ActionPane(
        motion: const DrawerMotion(),
        extentRatio: 0.7,
        children: [
          Expanded(
            child: Container(
              color: const Color(0xFF1F2937),
              padding: const EdgeInsets.all(12),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(item.title,
                      style: const TextStyle(
                          color: Colors.white,
                          fontWeight: FontWeight.bold,
                          fontSize: 16)),
                  const SizedBox(height: 8),
                  Text(item.description,
                      maxLines: 5,
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(color: Colors.white70)),
                ],
              ),
            ),
          ),
        ],
      ),
      child: AspectRatio(
        aspectRatio: 16 / 10,
        child: CachedNetworkImage(
          imageUrl: item.imageUrl,
          fit: BoxFit.cover,
          placeholder: (_, __) => const ColoredBox(color: Colors.black12),
          errorWidget: (_, __, ___) => const Center(child: Icon(Icons.error)),
        ),
      ),
    );
  }
}
```

---

## 7) Make the strip feel continuous

For truly endless feed behavior:

- Keep a `ScrollController`.
- When user nears bottom, fetch next page from your flat-file API (or another JSON file chunk).
- Append to list state.

If your source is static, simulate looping by repeating data:

- build from `items[index % items.length]` with a very large `itemCount`.
- preserve keys carefully if you do this.

---

## 8) Production tips

- Pre-cache first N images for instant startup experience.
- Keep row height fixed for smoother scrolling.
- Add shimmer/skeleton placeholders.
- Add retry + offline fallback (cached JSON + cached images).
- Validate all URLs in your content pipeline.

---

## 9) CSV option (if you must)

CSV works, but nested/long text is more fragile. JSON is usually better for rich descriptions.

If using CSV, keep columns simple:

`id,imageUrl,title,description`

And use a parser package (`csv`) to convert rows to `PhotoItem`.

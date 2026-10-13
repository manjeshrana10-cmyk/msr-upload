// ============================================================
// MSR Upload v2 - single-file Flutter app (lib/main.dart)
// pubspec.yaml -> dependencies:
//   http: ^1.2.2
//   image_picker: ^1.1.2
//   google_sign_in: ^6.2.1
//   flutter_facebook_auth: ^7.1.1
//   shared_preferences: ^2.3.2
//   video_thumbnail: ^0.5.3
//   video_player: ^2.9.1
//   path_provider: ^2.1.4
//   image: ^4.2.0
//   ffmpeg_kit_flutter_new: ^3.1.0   (pub.dev par latest version check karein)
// ============================================================
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'dart:ui';
import 'package:ffmpeg_kit_flutter_new/ffmpeg_kit.dart';
import 'package:ffmpeg_kit_flutter_new/return_code.dart';
import 'package:flutter/material.dart';
import 'package:flutter_facebook_auth/flutter_facebook_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:http/http.dart' as http;
import 'package:image/image.dart' as img;
import 'package:image_picker/image_picker.dart';
import 'package:path_provider/path_provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:video_player/video_player.dart';
import 'package:video_thumbnail/video_thumbnail.dart';

const fbGraph = 'https://graph.facebook.com/v19.0';
final navIndex = ValueNotifier<int>(0);
final pendingVideo = ValueNotifier<String?>(null); // editor -> home
final isDark = ValueNotifier<bool>(false); // light/dark (moon button)
const kSub = Color(0xFF8A8FA8);
const kHint = Color(0xFFA3A8BE);
const kLine = Color(0x33888FA8);
const kPurple = Color(0xFF7B61FF);
const kAmber = Color(0xFFE59100);
Color sheetBg() => isDark.value ? const Color(0xF20B1026) : const Color(0xF2F4F1FF);

final platIcons = {
  'yt': Icons.play_circle_fill,
  'fb': Icons.facebook,
  'ig': Icons.camera_alt
};
final platColors = {
  'yt': const Color(0xFFFF3B30),
  'fb': const Color(0xFF1877F2),
  'ig': const Color(0xFFE1306C)
};
final platNames = {'yt': 'YouTube', 'fb': 'Facebook', 'ig': 'Instagram'};

String fmt(DateTime d) =>
    '${d.day}/${d.month}/${d.year}  ${d.hour.toString().padLeft(2, '0')}:${d.minute.toString().padLeft(2, '0')}';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Store.load();
  isDark.value = Store.sp.getBool('dark') ?? false;
  runApp(const MsrApp());
}

class MsrApp extends StatelessWidget {
  const MsrApp({super.key});
  @override
  Widget build(BuildContext context) => ValueListenableBuilder<bool>(
        valueListenable: isDark,
        builder: (_, d, __) => MaterialApp(
          title: 'MSR Upload',
          debugShowCheckedModeBanner: false,
          theme: ThemeData(
              useMaterial3: true,
              colorSchemeSeed: kPurple,
              brightness: d ? Brightness.dark : Brightness.light),
          home: const SplashScreen(),
        ),
      );
}

// ================= Data: Post + Store =================
class Post {
  String id, path, title, cap, privacy, note;
  String? thumb;
  bool custom, short;
  int? sched;
  int created;
  Map<String, String> st; // yt/fb/ig: off | run | done | sched | pending | fail
  Post(
      {required this.id,
      required this.path,
      required this.title,
      required this.cap,
      required this.privacy,
      required this.short,
      required this.created,
      required this.st,
      this.thumb,
      this.custom = false,
      this.sched,
      this.note = ''});
  Map<String, dynamic> toJson() => {
        'id': id, 'path': path, 'title': title, 'cap': cap, 'privacy': privacy,
        'note': note, 'thumb': thumb, 'custom': custom, 'short': short,
        'sched': sched, 'created': created, 'st': st,
      };
  factory Post.fromJson(Map<String, dynamic> j) => Post(
        id: j['id'], path: j['path'], title: j['title'], cap: j['cap'],
        privacy: j['privacy'], note: j['note'] ?? '', thumb: j['thumb'],
        custom: j['custom'] ?? false, short: j['short'] ?? true,
        sched: j['sched'], created: j['created'],
        st: Map<String, String>.from(j['st']),
      );
  // schedule ka time nikal gaya ho to 'sched' ko 'done' maanein
  String eff(String k) {
    final s = st[k] ?? 'off';
    if (s == 'sched' && sched != null && sched! <= DateTime.now().millisecondsSinceEpoch) return 'done';
    return s;
  }
}

class Store {
  static late SharedPreferences sp;
  static final posts = ValueNotifier<List<Post>>([]);

  static Future<void> load() async {
    sp = await SharedPreferences.getInstance();
    posts.value = (sp.getStringList('posts') ?? [])
        .map((s) => Post.fromJson(jsonDecode(s)))
        .toList();
  }

  static Future<void> save() async {
    await sp.setStringList(
        'posts', posts.value.map((p) => jsonEncode(p.toJson())).toList());
    posts.value = [...posts.value];
  }

  static Map<String, dynamic>? get profile {
    final s = sp.getString('profile');
    return s == null ? null : jsonDecode(s);
  }

  static Future<String?> autoThumb(String video) async {
    try {
      final d = await getApplicationDocumentsDirectory();
      return await VideoThumbnail.thumbnailFile(
          video: video,
          thumbnailPath: d.path,
          imageFormat: ImageFormat.JPEG,
          maxWidth: 720,
          quality: 80,
          timeMs: 1000);
    } catch (_) {
      return null;
    }
  }

  // Har 30 sec: pending Instagram schedule post karo
  static bool _ticking = false;
  static Future<void> tick() async {
    if (_ticking) return;
    _ticking = true;
    try {
      final now = DateTime.now().millisecondsSinceEpoch;
      var changed = false;
      for (final p in posts.value) {
        if (p.st['ig'] == 'pending' && p.sched != null && p.sched! <= now) {
          if (!Session.igOk) continue;
          p.st['ig'] = 'run';
          changed = true;
          try {
            await Api.instagram(File(p.path), p.title, p.cap, (_) {});
            p.st['ig'] = 'done';
          } catch (e) {
            p.st['ig'] = 'fail';
            p.note += 'Instagram: ${e.toString().split('\n').first}\n';
          }
        }
      }
      if (changed) await save();
    } finally {
      _ticking = false;
    }
  }
}

// ================= Session: accounts =================
class Session {
  static final gsi = GoogleSignIn(scopes: [
    'https://www.googleapis.com/auth/youtube.upload',
    'https://www.googleapis.com/auth/youtube.readonly',
  ]);
  static GoogleSignInAccount? yt;
  static String? pageId, pageToken, pageName, igId, igName;
  static String? fbEmail, fbName, ytPhoto, fbPhoto, igPhoto;
  static bool get ytOk => yt != null;
  static bool get fbOk => pageId != null;
  static bool get igOk => igId != null;

  static Future<void> restore() async {
    try {
      yt = await gsi.signInSilently();
      if (yt != null) await _ytProfile();
    } catch (_) {}
    try {
      final t = await FacebookAuth.instance.accessToken;
      if (t != null) await _loadPage(t.tokenString);
    } catch (_) {}
  }

  static Future<void> logout() async {
    try { await gsi.signOut(); } catch (_) {}
    try { await FacebookAuth.instance.logOut(); } catch (_) {}
    yt = null; pageId = pageToken = pageName = igId = igName = null;
    fbEmail = fbName = ytPhoto = fbPhoto = igPhoto = null;
  }

  static Future<void> _ytProfile() async {
    try {
      final h = await yt!.authHeaders;
      final r = await http.get(
          Uri.parse('https://www.googleapis.com/youtube/v3/channels?part=snippet&mine=true'),
          headers: h);
      final items = jsonDecode(r.body)['items'] as List?;
      final th = items?.first['snippet']['thumbnails'];
      ytPhoto = (th['high'] ?? th['medium'] ?? th['default'])['url'];
    } catch (_) {
      ytPhoto = null;
    }
  }

  static Future<String?> connectYT() async {
    try {
      yt = await gsi.signIn();
      if (yt == null) return 'Cancel ho gaya';
      await _ytProfile();
      if (ytPhoto == null) return 'Is Google account par YouTube channel nahi mila';
      return null;
    } catch (e) {
      return 'YouTube error: $e';
    }
  }

  static Future<String?> connectFB() async {
    try {
      final r = await FacebookAuth.instance.login(permissions: [
        'public_profile', 'email', 'pages_show_list', 'pages_read_engagement',
        'pages_manage_posts', 'publish_video', 'instagram_basic',
        'instagram_content_publish',
      ]);
      if (r.status != LoginStatus.success || r.accessToken == null) {
        return r.message ?? 'Facebook login nahi hua';
      }
      return await _loadPage(r.accessToken!.tokenString);
    } catch (e) {
      return 'Facebook error: $e';
    }
  }

  static Future<String?> _loadPage(String token) async {
    try {
      final me = jsonDecode((await http.get(Uri.https('graph.facebook.com',
              '/v19.0/me', {'fields': 'name,email', 'access_token': token})))
          .body);
      fbEmail = me['email'];
      fbName = me['name'];
    } catch (_) {}
    final res = await http.get(Uri.https('graph.facebook.com', '/v19.0/me/accounts', {
      'fields': 'id,name,access_token,instagram_business_account{id,username,profile_picture_url}',
      'access_token': token
    }));
    final list = (jsonDecode(res.body)['data'] as List?) ?? [];
    if (list.isEmpty) return 'Koi Facebook Page nahi mila (Page chahiye)';
    final p = list.first;
    pageId = p['id'];
    pageToken = p['access_token'];
    pageName = p['name'];
    final ig = p['instagram_business_account'];
    igId = ig?['id'];
    igName = ig?['username'];
    igPhoto = ig?['profile_picture_url'];
    try {
      final pic = jsonDecode((await http.get(Uri.https('graph.facebook.com',
              '/v19.0/$pageId/picture', {
        'redirect': 'false', 'type': 'large', 'access_token': pageToken!
      })))
          .body);
      fbPhoto = pic['data']['url'];
    } catch (_) {
      fbPhoto = null;
    }
    return null;
  }

  // 8x8 average-hash: photo milti-julti hai ya nahi
  static Future<List<bool>?> _hash(String? url) async {
    if (url == null) return null;
    try {
      final r = await http.get(Uri.parse(url));
      final im = img.decodeImage(r.bodyBytes);
      if (im == null) return null;
      final s = img.copyResize(im, width: 8, height: 8, interpolation: img.Interpolation.average);
      final g = <double>[];
      for (var y = 0; y < 8; y++) {
        for (var x = 0; x < 8; x++) {
          final p = s.getPixel(x, y);
          g.add(0.299 * p.r + 0.587 * p.g + 0.114 * p.b);
        }
      }
      final m = g.reduce((a, b) => a + b) / g.length;
      return g.map((v) => v > m).toList();
    } catch (_) {
      return null;
    }
  }

  static Future<String?> photosMatch() async {
    final h = await Future.wait([_hash(ytPhoto), _hash(fbPhoto), _hash(igPhoto)]);
    final names = ['YouTube', 'Facebook', 'Instagram'];
    for (var i = 0; i < 3; i++) {
      if (h[i] == null) return '${names[i]} par profile photo nahi mili - profile photo lagao';
    }
    int dist(List<bool> a, List<bool> b) {
      var d = 0;
      for (var i = 0; i < 64; i++) { if (a[i] != b[i]) d++; }
      return d;
    }
    if (dist(h[0]!, h[1]!) > 12 || dist(h[0]!, h[2]!) > 12 || dist(h[1]!, h[2]!) > 12) {
      return 'Teeno jagah profile photo alag-alag hai. Profile photo lagao - YouTube, Facebook aur Instagram par ek hi photo rakhein.';
    }
    return null;
  }
}

// ================= Api: real uploads =================
class Api {
  static Future<http.Response> _stream(String method, Uri u,
      Map<String, String> h, File f, void Function(double) onP) async {
    final len = await f.length();
    final req = http.StreamedRequest(method, u)
      ..headers.addAll(h)
      ..contentLength = len;
    final fut = req.send();
    int sent = 0;
    await req.sink.addStream(f.openRead().map((c) {
      sent += c.length;
      onP(sent / len);
      return c;
    }));
    await req.sink.close();
    return http.Response.fromStream(await fut);
  }

  // YouTube: schedule = publishAt (private -> time par public). thumb = custom thumbnail
  static Future<String?> youtube(File f, String title, String cap, String privacy,
      bool short, int? sched, String? thumb, void Function(double) onP) async {
    final acc = Session.yt ?? await Session.gsi.signInSilently();
    if (acc == null) throw 'YouTube connected nahi hai';
    final auth = await acc.authHeaders;
    final len = await f.length();
    final status = <String, dynamic>{
      'privacyStatus': sched != null ? 'private' : privacy,
      if (sched != null)
        'publishAt': DateTime.fromMillisecondsSinceEpoch(sched).toUtc().toIso8601String(),
    };
    final init = await http.post(
      Uri.parse('https://www.googleapis.com/upload/youtube/v3/videos?uploadType=resumable&part=snippet,status'),
      headers: {
        ...auth,
        'Content-Type': 'application/json; charset=UTF-8',
        'X-Upload-Content-Length': '$len',
        'X-Upload-Content-Type': 'video/*',
      },
      body: jsonEncode({
        'snippet': {'title': title, 'description': short ? '$cap\n#Shorts' : cap, 'categoryId': '22'},
        'status': status
      }),
    );
    if (init.statusCode != 200) throw 'YT start ${init.statusCode}: ${init.body}';
    final loc = init.headers['location'];
    if (loc == null) throw 'YT upload URL nahi mila';
    final res = await _stream('PUT', Uri.parse(loc), {...auth, 'Content-Type': 'video/*'}, f, onP);
    if (res.statusCode != 200 && res.statusCode != 201) throw 'YT ${res.statusCode}: ${res.body}';
    if (thumb != null) {
      final id = jsonDecode(res.body)['id'];
      final png = thumb.toLowerCase().endsWith('.png');
      final r = await http.post(
          Uri.parse('https://www.googleapis.com/upload/youtube/v3/thumbnails/set?videoId=$id&uploadType=media'),
          headers: {...auth, 'Content-Type': png ? 'image/png' : 'image/jpeg'},
          body: await File(thumb).readAsBytes());
      if (r.statusCode != 200) {
        return 'YouTube: video upload ho gaya, par thumbnail set nahi hua (channel verify karein)';
      }
    }
    return null;
  }

  // Facebook: schedule = scheduled_publish_time, thumb = custom thumbnail
  static Future<String?> facebook(File f, String title, String cap, int? sched,
      String? thumb, void Function(double) onP) async {
    onP(-1);
    final req = http.MultipartRequest('POST',
        Uri.parse('https://graph-video.facebook.com/v19.0/${Session.pageId}/videos'));
    req.fields['access_token'] = Session.pageToken!;
    req.fields['title'] = title;
    req.fields['description'] = cap;
    if (sched != null) {
      req.fields['published'] = 'false';
      req.fields['scheduled_publish_time'] = '${sched ~/ 1000}';
    }
    req.files.add(await http.MultipartFile.fromPath('source', f.path));
    if (thumb != null) req.files.add(await http.MultipartFile.fromPath('thumb', thumb));
    final res = await http.Response.fromStream(await req.send());
    if (res.statusCode != 200) throw 'FB ${res.statusCode}: ${res.body}';
    return null;
  }

  // Instagram Reels (abhi turant post; schedule app-side hota hai)
  static Future<String?> instagram(File f, String title, String cap, void Function(double) onP) async {
    final igId = Session.igId, token = Session.pageToken;
    if (igId == null || token == null) throw 'Instagram connected nahi hai';
    final len = await f.length();
    final c = await http.post(Uri.parse('$fbGraph/$igId/media'), body: {
      'media_type': 'REELS',
      'upload_type': 'resumable',
      'caption': '$title\n$cap',
      'access_token': token,
    });
    final cj = jsonDecode(c.body);
    if (cj['id'] == null || cj['uri'] == null) throw 'IG container: ${c.body}';
    final up = await _stream('POST', Uri.parse(cj['uri']),
        {'Authorization': 'OAuth $token', 'offset': '0', 'file_size': '$len'}, f, onP);
    if (up.statusCode != 200) throw 'IG upload ${up.statusCode}: ${up.body}';
    for (var i = 0; i < 60; i++) {
      await Future.delayed(const Duration(seconds: 5));
      final s = await http.get(Uri.parse('$fbGraph/${cj['id']}?fields=status_code&access_token=$token'));
      final code = jsonDecode(s.body)['status_code'];
      if (code == 'FINISHED') break;
      if (code == 'ERROR' || code == 'EXPIRED') throw 'IG processing: $code';
      if (i == 59) throw 'IG processing timeout';
    }
    final pub = await http.post(Uri.parse('$fbGraph/$igId/media_publish'),
        body: {'creation_id': cj['id'], 'access_token': token});
    if (pub.statusCode != 200) throw 'IG publish: ${pub.body}';
    return null;
  }
}

// ================= Glass UI =================
class Bg extends StatelessWidget {
  final Widget child;
  const Bg({super.key, required this.child});
  Widget blob(Color c, double s) => ImageFiltered(
      imageFilter: ImageFilter.blur(sigmaX: 60, sigmaY: 60),
      child: Container(width: s, height: s, decoration: BoxDecoration(color: c, shape: BoxShape.circle)));
  @override
  Widget build(BuildContext context) => ValueListenableBuilder<bool>(
        valueListenable: isDark,
        builder: (_, d, __) => Scaffold(
          body: Container(
            decoration: BoxDecoration(
                gradient: LinearGradient(
                    begin: Alignment.topLeft,
                    end: Alignment.bottomRight,
                    colors: d
                        ? const [Color(0xFF0B1026), Color(0xFF1B1146), Color(0xFF0E2A47)]
                        : const [Color(0xFFF3F0FF), Color(0xFFEAF3FF), Color(0xFFFFF0F6)])),
            child: Stack(children: [
              Positioned(top: -70, left: -90, child: blob(d ? const Color(0xFF7C4DFF) : const Color(0xFFC9B8FF), 300)),
              Positioned(top: 240, right: -110, child: blob(d ? const Color(0xFF00E5FF) : const Color(0xFFA8E6F5), 280)),
              Positioned(bottom: -90, left: -50, child: blob(d ? const Color(0xFFFF4081) : const Color(0xFFFFC2D9), 260)),
              SafeArea(child: child),
            ]),
          ),
        ),
      );
}

// Liquid glass: moti kinara + rainbow refraction + naram shadow
class Glass extends StatelessWidget {
  final Widget child;
  final EdgeInsets padding;
  final double radius;
  const Glass({super.key, required this.child, this.padding = const EdgeInsets.all(16), this.radius = 24});
  @override
  Widget build(BuildContext context) {
    final d = Theme.of(context).brightness == Brightness.dark;
    return Container(
      padding: const EdgeInsets.all(1.6),
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(radius),
        boxShadow: [
          BoxShadow(
              color: (d ? Colors.black : kPurple).withOpacity(d ? .35 : .14),
              blurRadius: 26,
              offset: const Offset(0, 10))
        ],
        gradient: LinearGradient(
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
            colors: d
                ? [Colors.white.withOpacity(.45), const Color(0xFF8FE9FF).withOpacity(.35), const Color(0xFFB69CFF).withOpacity(.35), Colors.white.withOpacity(.2)]
                : [Colors.white, const Color(0xFFBFEFFF), const Color(0xFFD9C8FF), const Color(0xFFFFD3E6), Colors.white]),
      ),
      child: ClipRRect(
        borderRadius: BorderRadius.circular(radius > 2 ? radius - 1.6 : radius),
        child: BackdropFilter(
          filter: ImageFilter.blur(sigmaX: 22, sigmaY: 22),
          child: Container(
            padding: padding,
            decoration: BoxDecoration(
              gradient: LinearGradient(
                  begin: Alignment.topLeft,
                  end: Alignment.bottomRight,
                  colors: d
                      ? [Colors.white.withOpacity(.14), Colors.white.withOpacity(.04)]
                      : [Colors.white.withOpacity(.78), Colors.white.withOpacity(.42)]),
            ),
            child: child,
          ),
        ),
      ),
    );
  }
}

// Glossy purple pill button (arrow circle ke saath)
class GlowButton extends StatelessWidget {
  final String text;
  final VoidCallback? onTap;
  final bool busy;
  const GlowButton(this.text, this.onTap, {super.key, this.busy = false});
  @override
  Widget build(BuildContext context) => GestureDetector(
        onTap: onTap,
        child: AnimatedOpacity(
          duration: const Duration(milliseconds: 200),
          opacity: onTap == null ? .5 : 1,
          child: Container(
            height: 58,
            decoration: BoxDecoration(
              borderRadius: BorderRadius.circular(29),
              gradient: const LinearGradient(
                  begin: Alignment.topCenter,
                  end: Alignment.bottomCenter,
                  colors: [Color(0xFF9B87FF), Color(0xFF6A4DFF)]),
              border: Border.all(color: Colors.white.withOpacity(.55), width: 1.4),
              boxShadow: [BoxShadow(color: kPurple.withOpacity(.45), blurRadius: 24, offset: const Offset(0, 10))],
            ),
            child: Stack(alignment: Alignment.center, children: [
              Positioned(
                  top: 3, left: 18, right: 18,
                  child: Container(
                      height: 20,
                      decoration: BoxDecoration(
                          borderRadius: BorderRadius.circular(20),
                          gradient: LinearGradient(
                              begin: Alignment.topCenter,
                              end: Alignment.bottomCenter,
                              colors: [Colors.white.withOpacity(.45), Colors.white.withOpacity(0)])))),
              if (busy)
                const SizedBox(width: 24, height: 24, child: CircularProgressIndicator(strokeWidth: 2.5, color: Colors.white))
              else
                Text(text, style: const TextStyle(color: Colors.white, fontSize: 17, fontWeight: FontWeight.w800, letterSpacing: 1.2)),
              if (!busy)
                Positioned(
                    right: 8,
                    child: Container(
                        width: 42, height: 42,
                        decoration: BoxDecoration(
                            shape: BoxShape.circle,
                            color: Colors.white.withOpacity(.22),
                            border: Border.all(color: Colors.white.withOpacity(.5))),
                        child: const Icon(Icons.arrow_forward_rounded, color: Colors.white, size: 22))),
            ]),
          ),
        ),
      );
}

// Moon button: light <-> dark mode
class MoonButton extends StatelessWidget {
  const MoonButton({super.key});
  @override
  Widget build(BuildContext context) {
    final d = Theme.of(context).brightness == Brightness.dark;
    return GestureDetector(
      onTap: () async {
        isDark.value = !isDark.value;
        await Store.sp.setBool('dark', isDark.value);
      },
      child: Glass(
          radius: 18,
          padding: const EdgeInsets.all(10),
          child: Icon(d ? Icons.light_mode_rounded : Icons.dark_mode_rounded,
              size: 24, color: d ? kAmber : const Color(0xFF1B1F4B))),
    );
  }
}

InputDecoration dec(String h, {String? err}) => InputDecoration(
    hintText: h, errorText: err, border: InputBorder.none,
    hintStyle: const TextStyle(color: kHint));

void snack(BuildContext c, String m) => ScaffoldMessenger.of(c)
    .showSnackBar(SnackBar(content: Text(m), behavior: SnackBarBehavior.floating));

class PlatBadges extends StatelessWidget {
  final Post p;
  final double size;
  const PlatBadges(this.p, {super.key, this.size = 26});
  @override
  Widget build(BuildContext context) => Row(mainAxisSize: MainAxisSize.min, children: [
        for (final k in ['yt', 'fb', 'ig'])
          if (p.st[k] != null && p.st[k] != 'off')
            Builder(builder: (_) {
              final s = p.eff(k);
              final c = s == 'done' ? platColors[k]! : s == 'fail' ? Colors.redAccent : kAmber;
              return Container(
                margin: const EdgeInsets.only(right: 5),
                width: size, height: size,
                decoration: BoxDecoration(
                    color: s == 'done' ? Colors.white : Colors.black54,
                    shape: BoxShape.circle,
                    border: Border.all(color: c, width: 2)),
                child: Icon(platIcons[k], size: size * .62, color: c),
              );
            }),
      ]);
}

Widget thumbBox(String? path, {double? h}) => path != null && File(path).existsSync()
    ? Image.file(File(path), fit: BoxFit.cover, height: h, width: double.infinity)
    : Container(height: h, color: Colors.black26, child: const Center(child: Icon(Icons.movie, size: 36, color: kHint)));

class AccountTile extends StatelessWidget {
  final String name, sub;
  final IconData icon;
  final Color color;
  final bool ok, loading;
  final VoidCallback onConnect;
  const AccountTile(this.name, this.sub, this.icon, this.color, this.ok, this.loading, this.onConnect, {super.key});
  @override
  Widget build(BuildContext context) => Padding(
        padding: const EdgeInsets.only(bottom: 12),
        child: Glass(
          padding: const EdgeInsets.all(12),
          child: Row(children: [
            Container(
                width: 46, height: 46,
                decoration: BoxDecoration(color: color.withOpacity(.25), borderRadius: BorderRadius.circular(14)),
                child: Icon(icon, color: color, size: 26)),
            const SizedBox(width: 12),
            Expanded(
                child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Text(name, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.w700)),
              Text(sub, style: const TextStyle(color: kSub, fontSize: 12)),
            ])),
            ok
                ? const Icon(Icons.check_circle, color: Color(0xFF00E676), size: 28)
                : TextButton(
                    onPressed: loading ? null : onConnect,
                    style: TextButton.styleFrom(
                        backgroundColor: kPurple.withOpacity(.12),
                        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(14))),
                    child: const Text('Connect')),
          ]),
        ),
      );
}

// ================= Splash =================
class SplashScreen extends StatefulWidget {
  const SplashScreen({super.key});
  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen> with SingleTickerProviderStateMixin {
  late final AnimationController _c =
      AnimationController(vsync: this, duration: const Duration(milliseconds: 1400))..forward();
  @override
  void initState() {
    super.initState();
    _go();
  }

  Future<void> _go() async {
    final hasProfile = Store.profile != null;
    await Future.wait([
      Future.delayed(const Duration(milliseconds: 2600)),
      if (hasProfile) Session.restore(),
    ]);
    if (!mounted) return;
    Navigator.pushReplacement(
        context,
        PageRouteBuilder(
            transitionDuration: const Duration(milliseconds: 600),
            pageBuilder: (_, __, ___) => hasProfile ? const Shell() : const SignupScreen(),
            transitionsBuilder: (_, a, __, w) => FadeTransition(opacity: a, child: w)));
  }

  @override
  void dispose() {
    _c.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Bg(
        child: Center(
          child: FadeTransition(
            opacity: _c,
            child: ScaleTransition(
              scale: CurvedAnimation(parent: _c, curve: Curves.elasticOut),
              child: Glass(
                radius: 36,
                padding: const EdgeInsets.symmetric(horizontal: 44, vertical: 36),
                child: Column(mainAxisSize: MainAxisSize.min, children: const [
                  Icon(Icons.cloud_upload_rounded, size: 72, color: kPurple),
                  SizedBox(height: 14),
                  Text('MSR Upload', style: TextStyle(fontSize: 32, fontWeight: FontWeight.w900, letterSpacing: 1.2)),
                  SizedBox(height: 6),
                  Text('Ek baar upload, teeno jagah', style: TextStyle(color: kSub)),
                ]),
              ),
            ),
          ),
        ),
      );
}

// ================= Signup (strict checks) =================
class SignupScreen extends StatefulWidget {
  const SignupScreen({super.key});
  @override
  State<SignupScreen> createState() => _SignupScreenState();
}

class _SignupScreenState extends State<SignupScreen> {
  final name = TextEditingController(), email = TextEditingController(), mobile = TextEditingController();
  Map<String, String> errs = {};
  String? banner;
  bool busy = false;

  Future<void> _connect(String k) async {
    setState(() => busy = true);
    final err = k == 'yt' ? await Session.connectYT() : await Session.connectFB();
    if (!mounted) return;
    setState(() => busy = false);
    if (err != null) snack(context, err);
    else if (k != 'yt' && !Session.igOk) snack(context, 'Instagram Business account is Facebook Page se linked nahi hai');
  }

  Future<void> _submit() async {
    setState(() { busy = true; errs = {}; banner = null; });
    final e = <String, String>{};
    String? b;
    final n = name.text.trim();
    if (!RegExp(r"^[A-Za-z\u0900-\u097F .'-]{2,40}$").hasMatch(n)) e['name'] = 'Sahi naam likhein (sirf akshar, 2-40)';
    final em = email.text.trim().toLowerCase();
    if (!RegExp(r'^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$').hasMatch(em)) e['email'] = 'Sahi email likhein';
    var m = mobile.text.replaceAll(RegExp(r'[\s\-]'), '');
    if (m.startsWith('+91')) m = m.substring(3);
    else if (m.length == 12 && m.startsWith('91')) m = m.substring(2);
    if (!RegExp(r'^[6-9]\d{9}$').hasMatch(m)) e['mobile'] = 'Sahi 10 digit mobile number likhein';

    if (!(Session.ytOk && Session.fbOk && Session.igOk)) {
      b = 'Pehle YouTube, Facebook aur Instagram teeno connect karein';
    } else if (e.isEmpty) {
      if (Session.yt!.email.toLowerCase() != em) {
        e['email'] = 'Ye email YouTube wale Google account (${Session.yt!.email}) se match nahi karta';
      } else if (Session.fbEmail == null) {
        b = 'Facebook account par email nahi mila. Facebook mein wahi email jodein jo YouTube par hai.';
      } else if (Session.fbEmail!.toLowerCase() != em) {
        b = 'Facebook par alag email (${Session.fbEmail}) hai. Teeno jagah ek hi Gmail ID chahiye.';
      } else {
        b = await Session.photosMatch();
      }
    }
    if (e.isNotEmpty || b != null) {
      if (!mounted) return;
      setState(() { errs = e; banner = b; busy = false; });
      return;
    }
    String? b64;
    try {
      b64 = base64Encode((await http.get(Uri.parse(Session.ytPhoto!))).bodyBytes);
    } catch (_) {}
    await Store.sp.setString('profile', jsonEncode({'name': n, 'email': em, 'mobile': m, 'photo': b64}));
    if (!mounted) return;
    Navigator.pushReplacement(context, MaterialPageRoute(builder: (_) => const Shell()));
  }

  Widget _avatar(String label, String? url) => Column(children: [
        CircleAvatar(
            radius: 28,
            backgroundColor: kLine,
            backgroundImage: url != null ? NetworkImage(url) : null,
            child: url == null ? const Icon(Icons.person, color: kHint) : null),
        const SizedBox(height: 4),
        Text(label, style: const TextStyle(fontSize: 11, color: kSub)),
      ]);

  @override
  Widget build(BuildContext context) => Bg(
        child: ListView(padding: const EdgeInsets.all(20), children: [
          const SizedBox(height: 8),
          const Text('Sign up', style: TextStyle(fontSize: 30, fontWeight: FontWeight.w900)),
          const Text('Teeno accounts par ek hi Gmail ID aur ek hi profile photo honi chahiye',
              style: TextStyle(color: kSub)),
          const SizedBox(height: 18),
          Glass(child: TextField(controller: name, decoration: dec('Naam', err: errs['name']))),
          const SizedBox(height: 10),
          Glass(
              child: TextField(
                  controller: email,
                  keyboardType: TextInputType.emailAddress,
                  decoration: dec('Gmail ID (jo YouTube/Facebook/Instagram par hai)', err: errs['email']))),
          const SizedBox(height: 10),
          Glass(
              child: TextField(
                  controller: mobile,
                  keyboardType: TextInputType.phone,
                  decoration: dec('Mobile number', err: errs['mobile']))),
          const SizedBox(height: 16),
          AccountTile('YouTube', Session.yt?.email ?? 'Google account se login', Icons.play_circle_fill,
              platColors['yt']!, Session.ytOk, busy, () => _connect('yt')),
          AccountTile('Facebook Page', Session.fbName ?? 'Facebook login (Page chahiye)', Icons.facebook,
              platColors['fb']!, Session.fbOk, busy, () => _connect('fb')),
          AccountTile('Instagram', Session.igName != null ? '@${Session.igName}' : 'Facebook Page se linked account',
              Icons.camera_alt, platColors['ig']!, Session.igOk, busy, () => _connect('fb')),
          if (Session.ytOk || Session.fbOk || Session.igOk)
            Padding(
              padding: const EdgeInsets.symmetric(vertical: 8),
              child: Row(mainAxisAlignment: MainAxisAlignment.spaceEvenly, children: [
                _avatar('YouTube', Session.ytPhoto),
                _avatar('Facebook', Session.fbPhoto),
                _avatar('Instagram', Session.igPhoto),
              ]),
            ),
          if (banner != null)
            Padding(
              padding: const EdgeInsets.only(bottom: 12),
              child: Glass(
                  padding: const EdgeInsets.all(12),
                  child: Row(children: [
                    const Icon(Icons.error_outline, color: Colors.redAccent),
                    const SizedBox(width: 10),
                    Expanded(child: Text(banner!, style: const TextStyle(color: Colors.redAccent))),
                  ])),
            ),
          GlowButton('ACCOUNT BANAYEIN', busy ? null : _submit, busy: busy),
          const SizedBox(height: 20),
        ]),
      );
}

// ================= Shell: 5 buttons =================
class Shell extends StatefulWidget {
  const Shell({super.key});
  @override
  State<Shell> createState() => _ShellState();
}

class _ShellState extends State<Shell> {
  Timer? _t;
  void _nav() => setState(() {});
  @override
  void initState() {
    super.initState();
    navIndex.addListener(_nav);
    Store.tick();
    _t = Timer.periodic(const Duration(seconds: 30), (_) => Store.tick());
  }

  @override
  void dispose() {
    navIndex.removeListener(_nav);
    _t?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    const items = [
      [Icons.home_rounded, 'Home'],
      [Icons.schedule_rounded, 'Schedule'],
      [Icons.video_library_rounded, 'Uploaded'],
      [Icons.content_cut_rounded, 'Editor'],
      [Icons.person_rounded, 'Profile'],
    ];
    return Bg(
      child: Column(children: [
        Expanded(
            child: IndexedStack(index: navIndex.value, children: const [
          HomeScreen(), ScheduleScreen(), UploadedScreen(), EditorScreen(), ProfileScreen(),
        ])),
        Padding(
          padding: const EdgeInsets.fromLTRB(14, 0, 14, 10),
          child: Glass(
            radius: 30,
            padding: const EdgeInsets.symmetric(vertical: 8),
            child: Row(mainAxisAlignment: MainAxisAlignment.spaceAround, children: [
              for (var i = 0; i < 5; i++)
                GestureDetector(
                  behavior: HitTestBehavior.opaque,
                  onTap: () => navIndex.value = i,
                  child: Column(mainAxisSize: MainAxisSize.min, children: [
                    AnimatedContainer(
                        duration: const Duration(milliseconds: 250),
                        padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 4),
                        decoration: BoxDecoration(
                            borderRadius: BorderRadius.circular(16),
                            color: navIndex.value == i ? kPurple.withOpacity(.16) : Colors.transparent),
                        child: Icon(items[i][0] as IconData,
                            color: navIndex.value == i ? kPurple : kSub, size: 26)),
                    Text(items[i][1] as String,
                        style: TextStyle(
                            fontSize: 10.5,
                            color: navIndex.value == i ? kPurple : kSub)),
                  ]),
                ),
            ]),
          ),
        ),
      ]),
    );
  }
}

// ================= 1) Home: upload + thumbnail + schedule =================
class Plat {
  final String key;
  bool on = true, busy = false, done = false, failed = false;
  double? progress = 0;
  String status = 'Ready';
  Plat(this.key);
}

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});
  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  String? video, autoThumb, customThumb;
  final title = TextEditingController(), cap = TextEditingController();
  String privacy = 'public';
  bool short = true, running = false, schedOn = false;
  DateTime? when;
  Post? cur;
  final plats = [Plat('yt'), Plat('fb'), Plat('ig')];

  bool _ok(Plat p) => p.key == 'yt' ? Session.ytOk : p.key == 'fb' ? Session.fbOk : Session.igOk;

  @override
  void initState() {
    super.initState();
    pendingVideo.addListener(_fromEditor);
  }

  @override
  void dispose() {
    pendingVideo.removeListener(_fromEditor);
    super.dispose();
  }

  void _fromEditor() {
    final p = pendingVideo.value;
    if (p != null) {
      pendingVideo.value = null;
      _setVideo(p);
    }
  }

  Future<void> _setVideo(String path) async {
    setState(() {
      video = path;
      customThumb = null;
      autoThumb = null;
      cur = null;
      for (final p in plats) { p.busy = p.done = p.failed = false; p.progress = 0; p.status = 'Ready'; }
    });
    final t = await Store.autoThumb(path);
    if (mounted) setState(() => autoThumb = t);
  }

  Future<void> _pick() async {
    final v = await ImagePicker().pickVideo(source: ImageSource.gallery);
    if (v != null) await _setVideo(v.path);
  }

  Future<void> _pickThumb() async {
    final i = await ImagePicker().pickImage(source: ImageSource.gallery, maxWidth: 1280);
    if (i != null) setState(() => customThumb = i.path);
  }

  Future<void> _pickWhen() async {
    final now = DateTime.now();
    final d = await showDatePicker(
        context: context,
        initialDate: now.add(const Duration(days: 1)),
        firstDate: now,
        lastDate: now.add(const Duration(days: 28)));
    if (d == null || !mounted) return;
    final t = await showTimePicker(context: context, initialTime: TimeOfDay.fromDateTime(now.add(const Duration(hours: 1))));
    if (t == null) return;
    setState(() => when = DateTime(d.year, d.month, d.day, t.hour, t.minute));
  }

  Future<void> _run(Plat p, Post post) async {
    void upd(VoidCallback f) { if (mounted) setState(f); }
    upd(() { p.busy = true; p.done = false; p.failed = false; p.progress = 0; p.status = 'Shuru ho raha hai...'; });
    void prog(double v) => upd(() {
          p.progress = v < 0 ? null : v;
          p.status = v < 0 ? 'Upload ho raha hai...' : 'Uploading ${(v * 100).toInt()}%';
        });
    post.st[p.key] = 'run';
    try {
      final f = File(post.path);
      final thumb = post.custom ? post.thumb : null;
      String? note;
      if (p.key == 'yt') {
        note = await Api.youtube(f, post.title, post.cap, post.privacy, post.short, post.sched, thumb, prog);
      } else if (p.key == 'fb') {
        note = await Api.facebook(f, post.title, post.cap, post.sched, thumb, prog);
      } else {
        note = await Api.instagram(f, post.title, post.cap, prog);
      }
      if (note != null) post.note += '$note\n';
      post.st[p.key] = (post.sched != null && p.key != 'ig') ? 'sched' : 'done';
      upd(() {
        p.busy = false; p.done = true; p.progress = 1;
        p.status = (post.sched != null && p.key != 'ig') ? 'Schedule ho gaya ✓' : 'Upload ho gaya ✓';
      });
    } catch (e) {
      final s = e.toString();
      post.st[p.key] = 'fail';
      post.note += '${platNames[p.key]}: ${s.length > 150 ? s.substring(0, 150) : s}\n';
      upd(() {
        p.busy = false; p.failed = true; p.progress = 0;
        p.status = 'Fail: ${s.length > 110 ? s.substring(0, 110) : s}';
      });
    }
    await Store.save();
  }

  Future<void> _start() async {
    if (video == null) return snack(context, 'Pehle video chunein');
    if (title.text.trim().isEmpty) return snack(context, 'Title likhein');
    final sel = plats.where((p) => p.on && _ok(p)).toList();
    if (sel.isEmpty) return snack(context, 'Kam se kam ek platform ON karein');
    if (schedOn) {
      if (when == null) return snack(context, 'Schedule ka time chunein');
      if (when!.isBefore(DateTime.now().add(const Duration(minutes: 15)))) {
        return snack(context, 'Schedule kam se kam 15 minute baad ka rakhein');
      }
    }
    setState(() => running = true);
    final igSel = sel.any((p) => p.key == 'ig');
    var path = video!;
    if (schedOn && igSel) {
      // Instagram baad mein post hoga, isliye video ki copy surakshit rakho
      final d = await getApplicationDocumentsDirectory();
      final dst = '${d.path}/sched_${DateTime.now().millisecondsSinceEpoch}.mp4';
      await File(path).copy(dst);
      path = dst;
    }
    final post = Post(
      id: '${DateTime.now().millisecondsSinceEpoch}',
      path: path,
      title: title.text.trim(),
      cap: cap.text.trim(),
      privacy: privacy,
      short: short,
      created: DateTime.now().millisecondsSinceEpoch,
      thumb: customThumb ?? autoThumb,
      custom: customThumb != null,
      sched: schedOn ? when!.millisecondsSinceEpoch : null,
      st: {for (final p in plats) p.key: sel.contains(p) ? 'run' : 'off'},
    );
    for (final p in plats) {
      if (!sel.contains(p)) { p.status = _ok(p) ? 'Off' : 'Connected nahi'; }
    }
    final runNow = <Plat>[];
    for (final p in sel) {
      if (p.key == 'ig' && schedOn) {
        post.st['ig'] = 'pending';
        setState(() { p.done = true; p.progress = 1; p.status = 'Schedule: ${fmt(when!)} (app khula hona chahiye)'; });
      } else {
        runNow.add(p);
      }
    }
    Store.posts.value = [post, ...Store.posts.value];
    await Store.save();
    cur = post;
    await Future.wait(runNow.map((p) => _run(p, post)));
    if (mounted) setState(() => running = false);
    if (mounted && post.note.isNotEmpty) snack(context, post.note.trim().split('\n').first);
  }

  Widget _platRow(Plat p) {
    final ok = _ok(p);
    final c = platColors[p.key]!;
    return Padding(
      padding: const EdgeInsets.only(bottom: 10),
      child: Glass(
        padding: const EdgeInsets.fromLTRB(14, 10, 8, 10),
        radius: 20,
        child: Column(children: [
          Row(children: [
            Icon(platIcons[p.key], color: c, size: 28),
            const SizedBox(width: 12),
            Expanded(
                child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Text(platNames[p.key]!, style: const TextStyle(fontWeight: FontWeight.w700, fontSize: 16)),
              Text(ok ? p.status : 'Connected nahi (Profile se connect karein)',
                  style: TextStyle(
                      fontSize: 12,
                      color: p.failed ? Colors.redAccent : p.done ? const Color(0xFF00E676) : kSub)),
            ])),
            if (p.failed && !running && cur != null)
              TextButton(onPressed: () => _run(p, cur!), child: const Text('Retry'))
            else
              Switch(value: p.on && ok, onChanged: (ok && !running) ? (v) => setState(() => p.on = v) : null),
          ]),
          if (p.busy || p.done)
            Padding(
              padding: const EdgeInsets.only(top: 6, right: 8),
              child: ClipRRect(
                  borderRadius: BorderRadius.circular(8),
                  child: LinearProgressIndicator(value: p.progress, minHeight: 6, color: c, backgroundColor: kLine)),
            ),
        ]),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final thumb = customThumb ?? autoThumb;
    return ListView(padding: const EdgeInsets.all(20), children: [
      Row(children: const [
        Expanded(child: Text('MSR Upload', style: TextStyle(fontSize: 28, fontWeight: FontWeight.w900))),
        MoonButton(),
      ]),
      const SizedBox(height: 14),
      GestureDetector(
        onTap: running ? null : _pick,
        child: Glass(
          padding: const EdgeInsets.symmetric(vertical: 24),
          child: Column(children: [
            Icon(video == null ? Icons.video_library_rounded : Icons.check_circle,
                size: 42, color: video == null ? kSub : const Color(0xFF00E676)),
            const SizedBox(height: 8),
            Text(video == null ? 'Video chunein' : video!.split('/').last, maxLines: 1, overflow: TextOverflow.ellipsis),
          ]),
        ),
      ),
      const SizedBox(height: 12),
      Glass(child: TextField(controller: title, decoration: dec('Title'), enabled: !running)),
      const SizedBox(height: 10),
      Glass(child: TextField(controller: cap, maxLines: 3, decoration: dec('Caption / hashtags'), enabled: !running)),
      const SizedBox(height: 12),
      // ---- Thumbnail ----
      Glass(
        child: Row(children: [
          ClipRRect(
              borderRadius: BorderRadius.circular(12),
              child: SizedBox(width: 90, height: 120, child: thumbBox(thumb, h: 120))),
          const SizedBox(width: 14),
          Expanded(
              child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
            const Text('Thumbnail', style: TextStyle(fontWeight: FontWeight.w800, fontSize: 16)),
            const SizedBox(height: 2),
            Text(customThumb != null ? 'Custom thumbnail laga hai' : 'Video se automatic frame',
                style: const TextStyle(color: kSub, fontSize: 12)),
            const SizedBox(height: 8),
            Wrap(spacing: 8, children: [
              TextButton.icon(onPressed: running ? null : _pickThumb, icon: const Icon(Icons.image, size: 18), label: const Text('Chunein')),
              if (customThumb != null)
                TextButton(onPressed: running ? null : () => setState(() => customThumb = null), child: const Text('Hatayein')),
            ]),
          ])),
        ]),
      ),
      const SizedBox(height: 12),
      // ---- Schedule ----
      Glass(
        child: Column(children: [
          Row(children: [
            const Icon(Icons.schedule_rounded),
            const SizedBox(width: 10),
            const Expanded(child: Text('Schedule karein', style: TextStyle(fontWeight: FontWeight.w800, fontSize: 16))),
            Switch(value: schedOn, onChanged: running ? null : (v) => setState(() => schedOn = v)),
          ]),
          if (schedOn)
            Row(children: [
              Expanded(child: Text(when == null ? 'Date aur time chunein' : fmt(when!), style: const TextStyle(color: kSub))),
              TextButton(onPressed: running ? null : _pickWhen, child: const Text('Time chunein')),
            ]),
        ]),
      ),
      const SizedBox(height: 12),
      Wrap(spacing: 8, children: [
        for (final v in ['public', 'unlisted', 'private'])
          ChoiceChip(label: Text(v), selected: privacy == v, onSelected: running ? null : (_) => setState(() => privacy = v)),
      ]),
      SwitchListTile(
          contentPadding: EdgeInsets.zero,
          title: const Text('YouTube Short (#Shorts)'),
          value: short,
          onChanged: running ? null : (v) => setState(() => short = v)),
      const SizedBox(height: 6),
      ...plats.map(_platRow),
      const SizedBox(height: 10),
      GlowButton(schedOn ? 'SCHEDULE KAREIN' : 'START', running ? null : _start, busy: running),
      const SizedBox(height: 16),
    ]);
  }
}

// ================= 2) Schedule / queue screen =================
class ScheduleScreen extends StatelessWidget {
  const ScheduleScreen({super.key});
  @override
  Widget build(BuildContext context) => ValueListenableBuilder<List<Post>>(
        valueListenable: Store.posts,
        builder: (_, all, __) {
          final now = DateTime.now().millisecondsSinceEpoch;
          final list = all.where((p) {
            final sched = p.sched != null && p.sched! > now && p.st.values.any((s) => s == 'sched' || s == 'pending');
            return sched || p.st.values.contains('run') || p.st.values.contains('pending');
          }).toList()
            ..sort((a, b) => (a.sched ?? 0).compareTo(b.sched ?? 0));
          return Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
            const Padding(
                padding: EdgeInsets.fromLTRB(20, 16, 20, 4),
                child: Text('Schedule / Queue', style: TextStyle(fontSize: 26, fontWeight: FontWeight.w900))),
            Expanded(
              child: list.isEmpty
                  ? const Center(child: Text('Abhi koi video schedule ya upload mein nahi hai', style: TextStyle(color: kSub)))
                  : ListView(padding: const EdgeInsets.all(16), children: [
                      for (final p in list)
                        Padding(
                          padding: const EdgeInsets.only(bottom: 12),
                          child: Glass(
                            padding: const EdgeInsets.all(12),
                            child: Row(children: [
                              ClipRRect(
                                  borderRadius: BorderRadius.circular(12),
                                  child: SizedBox(width: 70, height: 92, child: thumbBox(p.thumb, h: 92))),
                              const SizedBox(width: 12),
                              Expanded(
                                  child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                                Text(p.title, maxLines: 2, overflow: TextOverflow.ellipsis, style: const TextStyle(fontWeight: FontWeight.w700)),
                                const SizedBox(height: 4),
                                Text(p.sched != null ? 'Post hoga: ${fmt(DateTime.fromMillisecondsSinceEpoch(p.sched!))}' : 'Upload ho raha hai...',
                                    style: const TextStyle(color: kAmber, fontSize: 12)),
                                const SizedBox(height: 8),
                                PlatBadges(p),
                              ])),
                              if (!p.st.values.contains('run'))
                                IconButton(
                                  icon: const Icon(Icons.delete_outline),
                                  onPressed: () async {
                                    final ok = await showDialog<bool>(
                                        context: context,
                                        builder: (_) => AlertDialog(
                                              title: const Text('App se hatayein?'),
                                              content: const Text(
                                                  'YouTube/Facebook par jo schedule pehle se ho chuka hai wo wahin rahega; use wahan se alag se hatana hoga. Instagram wala schedule cancel ho jayega.'),
                                              actions: [
                                                TextButton(onPressed: () => Navigator.pop(context, false), child: const Text('Nahi')),
                                                TextButton(onPressed: () => Navigator.pop(context, true), child: const Text('Haan')),
                                              ],
                                            ));
                                    if (ok == true) {
                                      Store.posts.value = Store.posts.value.where((x) => x.id != p.id).toList();
                                      await Store.save();
                                    }
                                  },
                                ),
                            ]),
                          ),
                        ),
                    ]),
            ),
          ]);
        },
      );
}

// ================= 3) Uploaded videos + platform icons =================
class UploadedScreen extends StatelessWidget {
  const UploadedScreen({super.key});

  void _detail(BuildContext context, Post p) {
    showModalBottomSheet(
        context: context,
        backgroundColor: sheetBg(),
        builder: (_) => Padding(
              padding: const EdgeInsets.all(20),
              child: Column(mainAxisSize: MainAxisSize.min, crossAxisAlignment: CrossAxisAlignment.start, children: [
                Text(p.title, style: const TextStyle(fontSize: 18, fontWeight: FontWeight.w800)),
                const SizedBox(height: 4),
                Text(fmt(DateTime.fromMillisecondsSinceEpoch(p.created)), style: const TextStyle(color: kSub)),
                const SizedBox(height: 12),
                for (final k in ['yt', 'fb', 'ig'])
                  if (p.st[k] != null && p.st[k] != 'off')
                    ListTile(
                      dense: true,
                      leading: Icon(platIcons[k], color: platColors[k]),
                      title: Text(platNames[k]!),
                      trailing: Text(
                          {'done': 'Upload ho gaya', 'fail': 'Fail', 'sched': 'Scheduled', 'pending': 'Pending', 'run': 'Chal raha hai'}[p.eff(k)] ?? '',
                          style: TextStyle(color: p.eff(k) == 'done' ? const Color(0xFF00E676) : p.eff(k) == 'fail' ? Colors.redAccent : kAmber)),
                    ),
                if (p.note.trim().isNotEmpty) ...[
                  const SizedBox(height: 8),
                  Text(p.note.trim(), style: const TextStyle(color: kSub, fontSize: 12)),
                ],
                const SizedBox(height: 10),
                TextButton.icon(
                    onPressed: () async {
                      Store.posts.value = Store.posts.value.where((x) => x.id != p.id).toList();
                      await Store.save();
                      if (context.mounted) Navigator.pop(context);
                    },
                    icon: const Icon(Icons.delete_outline),
                    label: const Text('List se hatayein')),
              ]),
            ));
  }

  @override
  Widget build(BuildContext context) => ValueListenableBuilder<List<Post>>(
        valueListenable: Store.posts,
        builder: (_, all, __) {
          final list = all.where((p) => ['yt', 'fb', 'ig'].any((k) => ['done', 'fail'].contains(p.eff(k)))).toList()
            ..sort((a, b) => b.created.compareTo(a.created));
          return Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
            const Padding(
                padding: EdgeInsets.fromLTRB(20, 16, 20, 4),
                child: Text('Uploaded videos', style: TextStyle(fontSize: 26, fontWeight: FontWeight.w900))),
            Expanded(
              child: list.isEmpty
                  ? const Center(child: Text('Abhi koi video upload nahi hui', style: TextStyle(color: kSub)))
                  : GridView.builder(
                      padding: const EdgeInsets.all(16),
                      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                          crossAxisCount: 2, mainAxisSpacing: 12, crossAxisSpacing: 12, childAspectRatio: .66),
                      itemCount: list.length,
                      itemBuilder: (_, i) {
                        final p = list[i];
                        return GestureDetector(
                          onTap: () => _detail(context, p),
                          child: ClipRRect(
                            borderRadius: BorderRadius.circular(22),
                            child: Stack(fit: StackFit.expand, children: [
                              thumbBox(p.thumb),
                              Positioned.fill(
                                  child: DecoratedBox(
                                      decoration: BoxDecoration(
                                          gradient: LinearGradient(
                                              begin: Alignment.topCenter,
                                              end: Alignment.bottomCenter,
                                              colors: [Colors.black54, Colors.transparent, Colors.black87])))),
                              Positioned(top: 8, left: 8, child: PlatBadges(p, size: 28)),
                              Positioned(
                                  left: 10, right: 10, bottom: 10,
                                  child: Column(crossAxisAlignment: CrossAxisAlignment.start, mainAxisSize: MainAxisSize.min, children: [
                                    Text(p.title, maxLines: 2, overflow: TextOverflow.ellipsis, style: const TextStyle(fontWeight: FontWeight.w700, color: Colors.white)),
                                    Text(fmt(DateTime.fromMillisecondsSinceEpoch(p.created)), style: const TextStyle(fontSize: 11, color: Colors.white70)),
                                  ])),
                            ]),
                          ),
                        );
                      },
                    ),
            ),
          ]);
        },
      );
}

// ================= 4) Editor (CapCut jaisa chhota) =================
class FilterDef {
  final String name, ff;
  final List<double> m;
  const FilterDef(this.name, this.m, this.ff);
}

List<double> _sat(double s) {
  const r = .2126, g = .7152, b = .0722;
  return [
    r * (1 - s) + s, g * (1 - s), b * (1 - s), 0, 0,
    r * (1 - s), g * (1 - s) + s, b * (1 - s), 0, 0,
    r * (1 - s), g * (1 - s), b * (1 - s) + s, 0, 0,
    0, 0, 0, 1, 0,
  ];
}

final filters = <FilterDef>[
  FilterDef('Original', [1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0], ''),
  FilterDef('Vivid', _sat(1.5), 'eq=saturation=1.5:contrast=1.1'),
  FilterDef('Warm', [1.1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, .9, 0, 0, 0, 0, 0, 1, 0], 'colorbalance=rs=.15:bs=-.15'),
  FilterDef('Cool', [.9, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1.1, 0, 0, 0, 0, 0, 1, 0], 'colorbalance=rs=-.15:bs=.15'),
  FilterDef('B&W', _sat(0), 'hue=s=0'),
  FilterDef('Vintage', [.393, .769, .189, 0, 0, .349, .686, .168, 0, 0, .272, .534, .131, 0, 0, 0, 0, 0, 1, 0],
      'colorchannelmixer=.393:.769:.189:0:.349:.686:.168:0:.272:.534:.131'),
  FilterDef('Cinematic', [1.15, 0, 0, 0, -10, 0, 1.1, 0, 0, -5, 0, 0, 1.1, 0, 5, 0, 0, 0, 1, 0],
      'eq=contrast=1.2:saturation=1.1,colorbalance=rs=.08:bs=-.08'),
];

class EditorScreen extends StatefulWidget {
  const EditorScreen({super.key});
  @override
  State<EditorScreen> createState() => _EditorScreenState();
}

class _EditorScreenState extends State<EditorScreen> {
  String? path, out;
  VideoPlayerController? vc;
  int fi = 0;
  double bright = 0, speed = 1, dur = 1;
  bool mute = false, busy = false;
  RangeValues range = const RangeValues(0, 1);

  @override
  void dispose() {
    vc?.dispose();
    super.dispose();
  }

  Future<void> _pick() async {
    final v = await ImagePicker().pickVideo(source: ImageSource.gallery);
    if (v == null) return;
    await vc?.dispose();
    final c = VideoPlayerController.file(File(v.path));
    await c.initialize();
    c.setLooping(true);
    c.play();
    final d = c.value.duration.inMilliseconds / 1000;
    setState(() {
      vc = c; path = v.path; dur = d < 1 ? 1 : d; range = RangeValues(0, dur);
      fi = 0; bright = 0; speed = 1; mute = false; out = null;
    });
  }

  List<double> _matrix() {
    final m = [...filters[fi].m];
    final o = bright * 255;
    m[4] += o; m[9] += o; m[14] += o;
    return m;
  }

  Future<void> _export() async {
    if (path == null) return;
    setState(() { busy = true; out = null; });
    try {
      final dir = await getApplicationDocumentsDirectory();
      final o = '${dir.path}/msr_${DateTime.now().millisecondsSinceEpoch}.mp4';
      final vf = [
        if (filters[fi].ff.isNotEmpty) filters[fi].ff,
        if (bright != 0) 'eq=brightness=${bright.toStringAsFixed(2)}',
        if (speed != 1) 'setpts=PTS/$speed',
      ].join(',');
      final audio = mute ? '-an' : (speed != 1 ? '-af "atempo=$speed" -c:a aac' : '-c:a aac');
      final cmd = '-y -ss ${range.start.toStringAsFixed(2)} -i "$path" -t ${(range.end - range.start).toStringAsFixed(2)} '
          '${vf.isNotEmpty ? '-vf "$vf"' : ''} -c:v libx264 -preset veryfast -crf 23 -pix_fmt yuv420p $audio "$o"';
      final s = await FFmpegKit.execute(cmd);
      final rc = await s.getReturnCode();
      if (ReturnCode.isSuccess(rc)) {
        if (mounted) setState(() => out = o);
      } else {
        if (mounted) snack(context, 'Export fail ho gaya');
      }
    } catch (e) {
      if (mounted) snack(context, 'Export error: $e');
    }
    if (mounted) setState(() => busy = false);
  }

  Widget _label(String t) => Padding(
      padding: const EdgeInsets.only(top: 14, bottom: 4),
      child: Text(t, style: const TextStyle(fontWeight: FontWeight.w800)));

  @override
  Widget build(BuildContext context) => ListView(padding: const EdgeInsets.all(20), children: [
        const Text('Video Editor', style: TextStyle(fontSize: 26, fontWeight: FontWeight.w900)),
        const SizedBox(height: 12),
        if (vc == null)
          GestureDetector(
            onTap: _pick,
            child: const Glass(
                padding: EdgeInsets.symmetric(vertical: 50),
                child: Center(child: Column(children: [
                  Icon(Icons.movie_creation_rounded, size: 44, color: kSub),
                  SizedBox(height: 8),
                  Text('Edit karne ke liye video chunein'),
                ]))),
          )
        else ...[
          Center(
            child: ConstrainedBox(
              constraints: const BoxConstraints(maxHeight: 340),
              child: GestureDetector(
                onTap: () => setState(() => vc!.value.isPlaying ? vc!.pause() : vc!.play()),
                child: ClipRRect(
                  borderRadius: BorderRadius.circular(20),
                  child: AspectRatio(
                    aspectRatio: vc!.value.aspectRatio,
                    child: ColorFiltered(colorFilter: ColorFilter.matrix(_matrix()), child: VideoPlayer(vc!)),
                  ),
                ),
              ),
            ),
          ),
          _label('Color filter'),
          SizedBox(
            height: 44,
            child: ListView.separated(
              scrollDirection: Axis.horizontal,
              itemCount: filters.length,
              separatorBuilder: (_, __) => const SizedBox(width: 8),
              itemBuilder: (_, i) => ChoiceChip(
                  label: Text(filters[i].name), selected: fi == i, onSelected: (_) => setState(() => fi = i)),
            ),
          ),
          _label('Brightness'),
          Slider(value: bright, min: -.3, max: .3, onChanged: (v) => setState(() => bright = v)),
          _label('Trim  (${range.start.toStringAsFixed(1)}s - ${range.end.toStringAsFixed(1)}s)'),
          RangeSlider(values: range, min: 0, max: dur, onChanged: (v) {
            setState(() => range = v);
            vc!.seekTo(Duration(milliseconds: (v.start * 1000).toInt()));
          }),
          _label('Speed'),
          Wrap(spacing: 8, children: [
            for (final s in [.5, 1.0, 1.5, 2.0])
              ChoiceChip(label: Text('${s}x'), selected: speed == s, onSelected: (_) => setState(() => speed = s)),
          ]),
          SwitchListTile(
              contentPadding: EdgeInsets.zero,
              title: const Text('Awaaz band (mute)'),
              value: mute,
              onChanged: (v) => setState(() => mute = v)),
          const SizedBox(height: 8),
          GlowButton('EXPORT KAREIN', busy ? null : _export, busy: busy),
          if (out != null) ...[
            const SizedBox(height: 12),
            Glass(
                child: Row(children: [
              const Icon(Icons.check_circle, color: Color(0xFF00E676)),
              const SizedBox(width: 10),
              const Expanded(child: Text('Edit hui video taiyar hai')),
              TextButton(
                  onPressed: () {
                    pendingVideo.value = out;
                    navIndex.value = 0;
                  },
                  child: const Text('Upload ke liye bhejein')),
            ])),
          ],
          TextButton.icon(onPressed: busy ? null : _pick, icon: const Icon(Icons.swap_horiz), label: const Text('Dusri video chunein')),
        ],
      ]);
}

// ================= 5) Profile =================
class ProfileScreen extends StatefulWidget {
  const ProfileScreen({super.key});
  @override
  State<ProfileScreen> createState() => _ProfileScreenState();
}

class _ProfileScreenState extends State<ProfileScreen> {
  bool loading = false;
  Future<void> _connect(String k) async {
    setState(() => loading = true);
    final err = k == 'yt' ? await Session.connectYT() : await Session.connectFB();
    if (!mounted) return;
    setState(() => loading = false);
    if (err != null) snack(context, err);
  }

  Widget _row(IconData i, String label, String v) => Padding(
        padding: const EdgeInsets.only(bottom: 10),
        child: Glass(
          padding: const EdgeInsets.all(14),
          child: Row(children: [
            Icon(i, color: kSub),
            const SizedBox(width: 12),
            Expanded(
                child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Text(label, style: const TextStyle(color: kSub, fontSize: 12)),
              Text(v, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.w600)),
            ])),
          ]),
        ),
      );

  @override
  Widget build(BuildContext context) {
    final pr = Store.profile ?? {};
    final b64 = pr['photo'] as String?;
    return ListView(padding: const EdgeInsets.all(20), children: [
      const Text('Profile', style: TextStyle(fontSize: 26, fontWeight: FontWeight.w900)),
      const SizedBox(height: 16),
      Center(
        child: Container(
          padding: const EdgeInsets.all(4),
          decoration: const BoxDecoration(shape: BoxShape.circle, gradient: LinearGradient(colors: [Color(0xFF7C4DFF), Color(0xFF00E5FF)])),
          child: CircleAvatar(
              radius: 56,
              backgroundColor: Colors.black38,
              backgroundImage: b64 != null ? MemoryImage(base64Decode(b64)) : null,
              child: b64 == null ? const Icon(Icons.person, size: 48) : null),
        ),
      ),
      const SizedBox(height: 18),
      _row(Icons.badge_outlined, 'Naam', pr['name'] ?? '-'),
      _row(Icons.email_outlined, 'Email', pr['email'] ?? '-'),
      _row(Icons.phone_outlined, 'Mobile', pr['mobile'] ?? '-'),
      const SizedBox(height: 8),
      AccountTile('YouTube', Session.yt?.email ?? 'Connected nahi', Icons.play_circle_fill, platColors['yt']!, Session.ytOk, loading, () => _connect('yt')),
      AccountTile('Facebook Page', Session.pageName ?? 'Connected nahi', Icons.facebook, platColors['fb']!, Session.fbOk, loading, () => _connect('fb')),
      AccountTile('Instagram', Session.igName != null ? '@${Session.igName}' : 'Connected nahi', Icons.camera_alt, platColors['ig']!, Session.igOk, loading, () => _connect('fb')),
      const SizedBox(height: 8),
      TextButton.icon(
        onPressed: () async {
          await Store.sp.remove('profile');
          await Session.logout();
          if (!context.mounted) return;
          Navigator.pushAndRemoveUntil(context, MaterialPageRoute(builder: (_) => const SignupScreen()), (_) => false);
        },
        icon: const Icon(Icons.logout, color: Colors.redAccent),
        label: const Text('Logout', style: TextStyle(color: Colors.redAccent)),
      ),
    ]);
  }
}

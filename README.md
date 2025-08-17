import 'dart:async';
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart' show rootBundle;

void main() {
  runApp(const MkhmkhLocalApp());
}

class MkhmkhLocalApp extends StatelessWidget {
  const MkhmkhLocalApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Mkhmkh Local',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(useMaterial3: true, brightness: Brightness.dark),
      home: const LoadingScreen(),
    );
  }
}

/* =========================
   Models
   ========================= */
class Category {
  final String id;
  final String nameAr;
  Category({required this.id, required this.nameAr});
  factory Category.fromJson(Map<String, dynamic> j) =>
      Category(id: j['id'], nameAr: j['name_ar']);
}

class Question {
  final String categoryId;
  final String textAr;
  final List<String> options;
  final int correctIndex;
  Question({
    required this.categoryId,
    required this.textAr,
    required this.options,
    required this.correctIndex,
  });
  factory Question.fromJson(Map<String, dynamic> j) => Question(
        categoryId: j['category_id'],
        textAr: j['text_ar'],
        options: (j['options'] as List).map((e) => e.toString()).toList(),
        correctIndex: j['correct_index'],
      );
}

/* =========================
   Loading & Data
   ========================= */
class LoadingScreen extends StatefulWidget {
  const LoadingScreen({super.key});
  @override
  State<LoadingScreen> createState() => _LoadingScreenState();
}

class _LoadingScreenState extends State<LoadingScreen> {
  Future<(List<Category>, List<Question>)> _load() async {
    final jsonStr = await rootBundle.loadString('assets/questions.json');
    final data = json.decode(jsonStr) as Map<String, dynamic>;
    final categories = (data['categories'] as List)
        .map((e) => Category.fromJson(e))
        .toList();
    final questions = (data['questions'] as List)
        .map((e) => Question.fromJson(e))
        .toList();
    return (categories, questions);
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<(List<Category>, List<Question>)>(
      future: _load(),
      builder: (context, snap) {
        if (!snap.hasData) {
          return Scaffold(
            body: Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: const [
                  CircularProgressIndicator(),
                  SizedBox(height: 12),
                  Text('جاري تحميل الأسئلة...')
                ],
              ),
            ),
          );
        }
        final (cats, qs) = snap.data!;
        return CategorySelectScreen(categories: cats, allQuestions: qs);
      },
    );
  }
}

/* =========================
   Category Selection Screen
   ========================= */
class CategorySelectScreen extends StatefulWidget {
  final List<Category> categories;
  final List<Question> allQuestions;
  const CategorySelectScreen(
      {super.key, required this.categories, required this.allQuestions});

  @override
  State<CategorySelectScreen> createState() => _CategorySelectScreenState();
}

class _CategorySelectScreenState extends State<CategorySelectScreen> {
  final Set<String> _selected = {};

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('اختيار الفئات')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            const Text(
              'اختر الفئات اللي تبغى تلعب فيها (تقدر تختار أكثر من فئة):',
              style: TextStyle(fontSize: 18),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 16),
            Wrap(
              spacing: 10,
              runSpacing: 10,
              children: widget.categories.map((c) {
                final isOn = _selected.contains(c.id);
                return FilterChip(
                  label: Text(c.nameAr),
                  selected: isOn,
                  onSelected: (_) {
                    setState(() {
                      if (isOn) {
                        _selected.remove(c.id);
                      } else {
                        _selected.add(c.id);
                      }
                    });
                  },
                );
              }).toList(),
            ),
            const Spacer(),
            FilledButton.icon(
              icon: const Icon(Icons.play_arrow),
              label: const Text('ابدأ اللعب'),
              onPressed: _selected.isEmpty
                  ? null
                  : () {
                      // جهّز بنك أسئلة عشوائي من الفئات المحددة
                      final filtered = widget.allQuestions
                          .where((q) => _selected.contains(q.categoryId))
                          .toList();
                      filtered.shuffle();
                      // خذ أول 10 (أو كلّها لو أقل)
                      final gameQs = filtered.take(10).toList();
                      Navigator.push(
                        context,
                        MaterialPageRoute(
                          builder: (_) => GameScreen(questions: gameQs),
                        ),
                      );
                    },
            )
          ],
        ),
      ),
    );
  }
}

/* =========================
   Game Screen (Timer + Score)
   ========================= */
class GameScreen extends StatefulWidget {
  final List<Question> questions;
  const GameScreen({super.key, required this.questions});

  @override
  State<GameScreen> createState() => _GameScreenState();
}

class _GameScreenState extends State<GameScreen> {
  static const int secondsPerQuestion = 30;

  late int _current;
  int _score = 0;
  int? _selected;
  bool _answered = false;

  late int _remaining;
  Timer? _timer;

  void _startTimer() {
    _timer?.cancel();
    _remaining = secondsPerQuestion;
    _timer = Timer.periodic(const Duration(seconds: 1), (t) {
      if (_remaining <= 1) {
        t.cancel();
        setState(() {
          _answered = true; // انتهى الوقت
        });
      } else {
        setState(() {
          _remaining -= 1;
        });
      }
    });
  }

  void _pick(int idx) {
    if (_answered) return;
    setState(() {
      _selected = idx;
      _answered = true;
      if (idx == widget.questions[_current].correctIndex) {
        _score += 1;
      }
    });
    _timer?.cancel();
  }

  void _next() {
    if (_current + 1 >= widget.questions.length) {
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(
          builder: (_) => ResultScreen(score: _score, total: widget.questions.length),
        ),
      );
      return;
    }
    setState(() {
      _selected = null;
      _answered = false;
      _current += 1;
    });
    _startTimer();
  }

  @override
  void initState() {
    super.initState();
    widget.questions.shuffle(); // مزيد من العشوائية
    _current = 0;
    _startTimer();
  }

  @override
  void dispose() {
    _timer?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final q = widget.questions[_current];
    return Scaffold(
      appBar: AppBar(
        title: const Text('Mkhmkh (Local)'),
        actions: [
          Center(
              child: Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Text('النقاط: $_score'),
          ))
        ],
      ),
      body: Center(
        child: ConstrainedBox(
          constraints: const BoxConstraints(maxWidth: 900),
          child: Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              children: [
                // العداد الزمني
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text('سؤال ${_current + 1} / ${widget.questions.length}',
                        style: const TextStyle(fontSize: 16)),
                    Chip(
                      label: Text('$_remaining ثانية'),
                    ),
                  ],
                ),
                const SizedBox(height: 16),
                // نص السؤال
                Expanded(
                  child: Center(
                    child: Text(
                      q.textAr,
                      textAlign: TextAlign.center,
                      style: const TextStyle(
                        fontSize: 34,
                        fontWeight: FontWeight.w700,
                      ),
                    ),
                  ),
                ),
                const SizedBox(height: 8),
                // الخيارات
                ...List.generate(q.options.length, (i) {
                  final isSelected = _selected == i;
                  final isCorrect = q.correctIndex == i;
                  Color? bg;
                  if (_answered && isSelected) {
                    bg = isCorrect ? Colors.green : Colors.red;
                  } else if (_answered && isCorrect) {
                    bg = Colors.green.withOpacity(0.7);
                  }
                  return Padding(
                    padding: const EdgeInsets.symmetric(vertical: 6.0),
                    child: SizedBox(
                      width: double.infinity,
                      child: ElevatedButton(
                        onPressed: () => _pick(i),
                        style: ElevatedButton.styleFrom(
                          padding: const EdgeInsets.all(18),
                          backgroundColor: bg,
                        ),
                        child: Text(
                          q.options[i],
                          style: const TextStyle(
                              fontSize: 20, fontWeight: FontWeight.w600),
                        ),
                      ),
                    ),
                  );
                }),
                const SizedBox(height: 12),
                // التالي / إنهاء
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text('الإجابة الصحيحة تُعرض بالأخضر بعد اختيارك أو انتهاء الوقت.'),
                    FilledButton(
                      onPressed: _next,
                      child: Text(_current + 1 >= widget.questions.length
                          ? 'إنهاء'
                          : 'السؤال التالي'),
                    ),
                  ],
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}

/* =========================
   Result Screen
   ========================= */
class ResultScreen extends StatelessWidget {
  final int score;
  final int total;
  const ResultScreen({super.key, required this.score, required this.total});

  @override
  Widget build(BuildContext context) {
    final pct = (score / total * 100).toStringAsFixed(0);
    return Scaffold(
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.emoji_events, size: 72),
              const SizedBox(height: 12),
              Text('مبروك! نتيجتك: $score / $total ($pct%)',
                  style: const TextStyle(fontSize: 26, fontWeight: FontWeight.bold)),
              const SizedBox(height: 24),
              FilledButton.icon(
                icon: const Icon(Icons.replay),
                label: const Text('العودة للاختيار'),
                onPressed: () {
                  Navigator.pushAndRemoveUntil(
                    context,
                    MaterialPageRoute(builder: (_) => const LoadingScreen()),
                    (route) => false,
                  );
                },
              )
            ],
          ),
        ),
      ),
    );
  }
}
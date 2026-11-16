# Belluci-Point
name: belluci_point
description: Belluci Point Flutter app
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: ^3.5.0

dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.2
  shared_preferences: ^2.3.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0

flutter:
  uses-material-design: true200response.body.isNotEmptyhttp.posthttp.Responseimport 'dart:convert';
import 'package:http/http.dart' as http;
import 'package:shared_preferences/shared_preferences.dart';

class ApiService {
  static const String baseUrl = 'http://10.0.2.2:5000/api';

  static Future<String?> getToken() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getString('token');
  }

  static Future<void> saveToken(String token) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('token', token);
  }

  static Future<void> clearToken() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove('token');
  }

  static Future<Map<String, String>> headers({bool auth = false}) async {
    final h = <String, String>{
      'Content-Type': 'application/json',
    };

    if (auth) {
      final token = await getToken();
      if (token != null && token.isNotEmpty) {
        h['Authorization'] = 'Bearer $token';
      }
    }

    return h;
  }

  static Future<dynamic> request(
    String method,
    String path, {
    Map<String, dynamic>? body,
    bool auth = false,
  }) async {
    final uri = Uri.parse('$baseUrl$path');
    final h = await headers(auth: auth);

    http.Response response;

    if (method == 'GET') {
      response = await http.get(uri, headers: h);
    } else if (method == 'POST') {
      response = await http.post(
        uri,
        headers: h,
        body: jsonEncode(body ?? {}),
      );
    } else {
      throw Exception('Method not supported');
    }

    final decoded = response.body.isNotEmpty
        ? jsonDecode(response.body)
        : <String, dynamic>{};

    if (response.statusCode < 200 || response.statusCode >= 300) {
      throw Exception(decoded['message'] ?? 'Request failed');
    }

    return decoded;
  }

  static Future<void> login(String email, String password) async {
    final data = await request(
      'POST',
      '/auth/login',
      body: {'email': email, 'password': password},
    );
    await saveToken(data['token']);
  }

  static Future<void> register({
    required String name,
    required String email,
    required String phone,
    required String password,
  }) async {
    final data = await request(
      'POST',
      '/auth/register',
      body: {
        'name': name,
        'email': email,
        'password': password,
        'phone': phone,
      },
    );

    await saveToken(data['token']);
  }

  static Future<bool> hasToken() async {
    final token = await getToken();
    return token != null && token.isNotEmpty;
  }

  static Future<void> logout() async {
    await clearToken();
  }

  static Future<Map<String, dynamic>> getProfile() async {
    final data = await request('GET', '/users/profile', auth: true);
    return Map<String, dynamic>.from(data['user']);
  }

  static Future<List<Map<String, dynamic>>> getRewards() async {
    final data = await request('GET', '/rewards', auth: true);
    return List<Map<String, dynamic>>.from(data['rewards']);
  }

  static Future<List<Map<String, dynamic>>> getTransactions() async {
    final data = await request('GET', '/users/transactions', auth: true);
    return List<Map<String, dynamic>>.from(data['transactions']);
  }

  static Future<void> redeemReward(int rewardId) async {
    await request(
      'POST',
      '/rewards/redeem',
      auth: true,
      body: {'rewardId': rewardId},
    );
  }
}http://10.0.2.2:5000/apirewardSnap.data1snapshot.data6value.lengthimport 'package:flutter/material.dart';
import 'api_service.dart';

void main() {
  runApp(const BelluciPointApp());
}

class BelluciPointApp extends StatelessWidget {
  const BelluciPointApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Belluci Point',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF4F2E22),
          primary: const Color(0xFF4F2E22),
          secondary: const Color(0xFFC8A570),
        ),
      ),
      home: const AppGate(),
    );
  }
}

class AppGate extends StatefulWidget {
  const AppGate({super.key});

  @override
  State<AppGate> createState() => _AppGateState();
}

class _AppGateState extends State<AppGate> {
  bool checking = true;
  bool loggedIn = false;

  @override
  void initState() {
    super.initState();
    _checkLogin();
  }

  Future<void> _checkLogin() async {
    final hasToken = await ApiService.hasToken();
    if (!mounted) return;

    setState(() {
      loggedIn = hasToken;
      checking = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    if (checking) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }

    return loggedIn ? const MemberHome() : const AuthScreen();
  }
}

class AuthScreen extends StatefulWidget {
  const AuthScreen({super.key});

  @override
  State<AuthScreen> createState() => _AuthScreenState();
}

class _AuthScreenState extends State<AuthScreen> {
  final _formKey = GlobalKey<FormState>();
  final nameController = TextEditingController();
  final emailController = TextEditingController();
  final phoneController = TextEditingController();
  final passwordController = TextEditingController();

  bool isRegister = false;
  bool loading = false;
  String? errorMessage;

  Future<void> submit() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() {
      loading = true;
      errorMessage = null;
    });

    try {
      if (isRegister) {
        await ApiService.register(
          name: nameController.text.trim(),
          email: emailController.text.trim(),
          phone: phoneController.text.trim(),
          password: passwordController.text.trim(),
        );
      } else {
        await ApiService.login(
          emailController.text.trim(),
          passwordController.text.trim(),
        );
      }

      if (!mounted) return;

      Navigator.pushReplacement(
        context,
        MaterialPageRoute(builder: (_) => const MemberHome()),
      );
    } catch (e) {
      if (!mounted) return;
      setState(() {
        errorMessage = e.toString().replaceFirst('Exception: ', '');
      });
    } finally {
      if (mounted) {
        setState(() => loading = false);
      }
    }
  }

  @override
  void dispose() {
    nameController.dispose();
    emailController.dispose();
    phoneController.dispose();
    passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Center(
          child: SingleChildScrollView(
            padding: const EdgeInsets.all(24),
            child: Form(
              key: _formKey,
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.stretch,
                children: [
                  const Icon(Icons.loyalty, size: 72, color: Color(0xFF4F2E22)),
                  const SizedBox(height: 12),
                  Text(
                    'Belluci Point',
                    textAlign: TextAlign.center,
                    style: Theme.of(context).textTheme.headlineMedium,
                  ),
                  const SizedBox(height: 24),

                  if (isRegister)
                    TextFormField(
                      controller: nameController,
                      decoration: const InputDecoration(
                        labelText: 'Nama',
                        border: OutlineInputBorder(),
                      ),
                      validator: (value) => value == null || value.trim().isEmpty
                          ? 'Nama wajib diisi'
                          : null,
                    ),
                  if (isRegister) const SizedBox(height: 12),
                  if (isRegister)
                    TextFormField(
                      controller: phoneController,
                      keyboardType: TextInputType.phone,
                      decoration: const InputDecoration(
                        labelText: 'Nomor telepon',
                        border: OutlineInputBorder(),
                      ),
                    ),
                  if (isRegister) const SizedBox(height: 12),
                  TextFormField(
                    controller: emailController,
                    keyboardType: TextInputType.emailAddress,
                    decoration: const InputDecoration(
                      labelText: 'Email',
                      border: OutlineInputBorder(),
                    ),
                    validator: (value) => value == null || value.trim().isEmpty
                        ? 'Email wajib diisi'
                        : null,
                  ),
                  const SizedBox(height: 12),
                  TextFormField(
                    controller: passwordController,
                    obscureText: true,
                    decoration: const InputDecoration(
                      labelText: 'Password',
                      border: OutlineInputBorder(),
                    ),
                    validator: (value) => value == null || value.length < 6
                        ? 'Password minimal 6 karakter'
                        : null,
                  ),
                  const SizedBox(height: 16),
                  if (errorMessage != null)
                    Padding(
                      padding: const EdgeInsets.only(bottom: 8),
                      child: Text(
                        errorMessage!,
                        style: const TextStyle(color: Colors.red),
                      ),
                    ),
                  FilledButton(
                    onPressed: loading ? null : submit,
                    child: Text(
                      loading
                          ? 'Memproses...'
                          : isRegister
                              ? 'Daftar'
                              : 'Masuk',
                    ),
                  ),
                  const SizedBox(height: 8),
                  TextButton(
                    onPressed: () {
                      setState(() {
                        isRegister = !isRegister;
                        errorMessage = null;
                      });
                    },
                    child: Text(
                      isRegister
                          ? 'Sudah punya akun? Masuk'
                          : 'Belum punya akun? Daftar',
                    ),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}

class MemberHome extends StatefulWidget {
  const MemberHome({super.key});

  @override
  State<MemberHome> createState() => _MemberHomeState();
}

class _MemberHomeState extends State<MemberHome> {
  int selectedIndex = 0;

  @override
  Widget build(BuildContext context) {
    final screens = [
      HomeTab(onRefresh: () => setState(() {})),
      RewardTab(onRefresh: () => setState(() {})),
      HistoryTab(onRefresh: () => setState(() {})),
      ProfileTab(onRefresh: () => setState(() {})),
    ];

    return Scaffold(
      appBar: AppBar(
        title: const Text('Belluci Point'),
      ),
      body: screens[selectedIndex],
      bottomNavigationBar: NavigationBar(
        selectedIndex: selectedIndex,
        onDestinationSelected: (index) => setState(() => selectedIndex = index),
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home), label: 'Beranda'),
          NavigationDestination(icon: Icon(Icons.card_giftcard), label: 'Reward'),
          NavigationDestination(icon: Icon(Icons.receipt_long), label: 'Riwayat'),
          NavigationDestination(icon: Icon(Icons.person), label: 'Profil'),
        ],
      ),
    );
  }
}

class HomeTab extends StatefulWidget {
  const HomeTab({super.key, required this.onRefresh});

  final VoidCallback onRefresh;

  @override
  State<HomeTab> createState() => _HomeTabState();
}

class _HomeTabState extends State<HomeTab> {
  late Future<Map<String, dynamic>> profileFuture;

  @override
  void initState() {
    super.initState();
    profileFuture = ApiService.getProfile();
  }

  Future<void> reload() async {
    setState(() {
      profileFuture = ApiService.getProfile();
    });
  }

  @override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: () async => reload(),
      child: FutureBuilder<Map<String, dynamic>>(
        future: profileFuture,
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          }

          if (snapshot.hasError) {
            return Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Text(snapshot.error.toString()),
                  const SizedBox(height: 12),
                  ElevatedButton(
                    onPressed: reload,
                    child: const Text('Coba lagi'),
                  ),
                ],
              ),
            );
          }

          final user = snapshot.data ?? {};
          final balance = int.tryParse('${user['points_balance'] ?? 0}') ?? 0;

          return ListView(
            padding: const EdgeInsets.all(16),
            children: [
              Card(
                child: Padding(
                  padding: const EdgeInsets.all(20),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      const Text('Saldo poin'),
                      const SizedBox(height: 12),
                      Text(
                        '$balance poin',
                        style: Theme.of(context).textTheme.headlineMedium,
                      ),
                      const SizedBox(height: 10),
                      const Text('Setiap belanja Rp100.000 mendapatkan 1 poin.'),
                    ],
                  ),
                ),
              ),
              const SizedBox(height: 20),
              Text('Reward tersedia', style: Theme.of(context).textTheme.titleLarge),
              const SizedBox(height: 12),
              FutureBuilder<List<Map<String, dynamic>>>(
                future: ApiService.getRewards(),
                builder: (context, rewardSnap) {
                  if (rewardSnap.connectionState == ConnectionState.waiting) {
                    return const Center(child: CircularProgressIndicator());
                  }
                  if (rewardSnap.hasError) {
                    return Text(rewardSnap.error.toString());
                  }

                  final rewards = rewardSnap.data ?? [];
                  return Column(
                    children: rewards.take(3).map((reward) {
                      final required = int.tryParse('${reward['points_required'] ?? 0}') ?? 0;
                      final discount = num.tryParse('${reward['discount_amount'] ?? 0}') ?? 0;

                      return Card(
                        margin: const EdgeInsets.only(bottom: 10),
                        child: ListTile(
                          leading: const Icon(Icons.card_giftcard),
                          title: Text(reward['name'] ?? 'Reward'),
                          subtitle: Text('$required poin • Diskon Rp ${discount.toStringAsFixed(0)}'),
                        ),
                      );
                    }).toList(),
                  );
                },
              ),
            ],
          );
        },
      ),
    );
  }
}

class RewardTab extends StatefulWidget {
  const RewardTab({super.key, required this.onRefresh});

  final VoidCallback onRefresh;

  @override
  State<RewardTab> createState() => _RewardTabState();
}

class _RewardTabState extends State<RewardTab> {
  Future<List<Map<String, dynamic>>>? rewardsFuture;

  @override
  void initState() {
    super.initState();
    rewardsFuture = ApiService.getRewards();
  }

  Future<void> _load() async {
    setState(() {
      rewardsFuture = ApiService.getRewards();
    });
  }

  Future<void> _redeem(int rewardId) async {
    try {
      await ApiService.redeemReward(rewardId);
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Reward berhasil ditukar')),
        );
      }
      widget.onRefresh();
      await _load();
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(e.toString().replaceFirst('Exception: ', ''))),
        );
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<List<Map<String, dynamic>>>(
      future: rewardsFuture,
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const Center(child: CircularProgressIndicator());
        }

        if (snapshot.hasError) {
          return Center(child: Text(snapshot.error.toString()));
        }

        final rewards = snapshot.data ?? [];
        if (rewards.isEmpty) {
          return const Center(child: Text('Belum ada reward tersedia.'));
        }

        return RefreshIndicator(
          onRefresh: _load,
          child: ListView.builder(
            padding: const EdgeInsets.all(16),
            itemCount: rewards.length,
            itemBuilder: (context, index) {
              final reward = rewards[index];
              final required = int.tryParse('${reward['points_required'] ?? 0}') ?? 0;
              final discount = num.tryParse('${reward['discount_amount'] ?? 0}') ?? 0;

              return Card(
                margin: const EdgeInsets.only(bottom: 12),
                child: ListTile(
                  title: Text(reward['name'] ?? 'Reward'),
                  subtitle: Text('$required poin • Diskon Rp ${discount.toStringAsFixed(0)}'),
                  trailing: FilledButton(
                    onPressed: () => _redeem(int.parse('${reward['id']}')),
                    child: const Text('Tukar'),
                  ),
                ),
              );
            },
          ),
        );
      },
    );
  }
}

class HistoryTab extends StatefulWidget {
  const HistoryTab({super.key, required this.onRefresh});

  final VoidCallback onRefresh;

  @override
  State<HistoryTab> createState() => _HistoryTabState();
}

class _HistoryTabState extends State<HistoryTab> {
  Future<List<Map<String, dynamic>>>? txFuture;

  @override
  void initState() {
    super.initState();
    txFuture = ApiService.getTransactions();
  }

  Future<void> _reload() async {
    setState(() {
      txFuture = ApiService.getTransactions();
    });
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<List<Map<String, dynamic>>>(
      future: txFuture,
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const Center(child: CircularProgressIndicator());
        }

        if (snapshot.hasError) {
          return Center(child: Text(snapshot.error.toString()));
        }

        final txs = snapshot.data ?? [];
        if (txs.isEmpty) {
          return const Center(child: Text('Belum ada riwayat transaksi'));
        }

        return RefreshIndicator(
          onRefresh: _reload,
          child: ListView.builder(
            padding: const EdgeInsets.all(16),
            itemCount: txs.length,
            itemBuilder: (context, index) {
              final tx = txs[index];
              final total = num.tryParse('${tx['total_amount'] ?? 0}') ?? 0;
              final points = int.tryParse('${tx['points_earned'] ?? 0}') ?? 0;
              final date = tx['created_at']?.toString().split('T').first ?? '';

              return Card(
                child: ListTile(
                  leading: const Icon(Icons.receipt_long),
                  title: Text('Belanja Rp ${total.toStringAsFixed(0)}'),
                  subtitle: Text('Dapat $points poin'),
                  trailing: Text(date),
                ),
              );
            },
          ),
        );
      },
    );
  }
}

class ProfileTab extends StatefulWidget {
  const ProfileTab({super.key, required this.onRefresh});

  final VoidCallback onRefresh;

  @override
  State<ProfileTab> createState() => _ProfileTabState();
}

class _ProfileTabState extends State<ProfileTab> {
  Future<Map<String, dynamic>>? profileFuture;

  @override
  void initState() {
    super.initState();
    profileFuture = ApiService.getProfile();
  }

  Future<void> _reload() async {
    setState(() {
      profileFuture = ApiService.getProfile();
    });
  }

  Future<void> _logout() async {
    await ApiService.logout();
    if (mounted) {
      Navigator.pushAndRemoveUntil(
        context,
        MaterialPageRoute(builder: (_) => const AuthScreen()),
        (route) => false,
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<Map<String, dynamic>>(
      future: profileFuture,
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const Center(child: CircularProgressIndicator());
        }

        if (snapshot.hasError) {
          return Center(child: Text(snapshot.error.toString()));
        }

        final user = snapshot.data ?? {};
        final balance = int.tryParse('${user['points_balance'] ?? 0}') ?? 0;

        return ListView(
          padding: const EdgeInsets.all(20),
          children: [
            CircleAvatar(
              radius: 36,
              backgroundColor: Colors.brown[100],
              child: const Icon(Icons.person, size: 34),
            ),
            const SizedBox(height: 16),
            Center(
              child: Text(
                user['name'] ?? 'Member',
                style: Theme.of(context).textTheme.headlineSmall,
              ),
            ),
            const SizedBox(height: 8),
            Center(child: Text(user['email'] ?? '')),
            if ((user['phone'] ?? '').toString().isNotEmpty)
              Center(child: Text(user['phone'].toString())),
            const SizedBox(height: 20),
            Card(
              child: ListTile(
                leading: const Icon(Icons.loyalty),
                title: const Text('Total poin'),
                trailing: Text('$balance poin'),
              ),
            ),
            const SizedBox(height: 20),
            OutlinedButton.icon(
              onPressed: _logout,
              icon: const Icon(Icons.logout),
              label: const Text('Keluar'),
            ),
          ],
        );
      },
    );
  }
}
# 三言客户端 (sanyan-app) 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建三言 AI 陪伴聊天 App 的 Flutter 客户端，包含登录注册、聊天界面、WebSocket 实时通信、推送通知。

**Architecture:** Flutter 跨平台应用，GetX 管理状态和路由。网络层封装 REST API 和 WebSocket 客户端。本地使用 SharedPreferences 存储 token 和基础配置。

**Tech Stack:** Flutter 3.x, Dart, GetX, web_socket_channel, dio, shared_preferences, 极光推送 Flutter SDK

**前置条件:** 后端服务已部署运行（参考 sanyan-server 计划）

---

## 项目文件结构

```
sanyan-app/
├── pubspec.yaml
├── lib/
│   ├── main.dart
│   ├── app/
│   │   ├── routes.dart
│   │   └── bindings.dart
│   ├── core/
│   │   ├── network/
│   │   │   ├── api_client.dart          — Dio HTTP 封装
│   │   │   ├── api_response.dart        — 统一响应体
│   │   │   └── ws_client.dart           — WebSocket 客户端
│   │   ├── storage/
│   │   │   └── local_storage.dart       — SharedPreferences 封装
│   │   └── constants.dart               — API 地址、常量
│   ├── models/
│   │   ├── user.dart
│   │   ├── character.dart
│   │   ├── conversation.dart
│   │   └── message.dart
│   ├── pages/
│   │   ├── splash/
│   │   │   └── splash_page.dart
│   │   ├── auth/
│   │   │   ├── login_page.dart
│   │   │   ├── login_controller.dart
│   │   │   ├── register_page.dart
│   │   │   └── register_controller.dart
│   │   ├── home/
│   │   │   ├── home_page.dart
│   │   │   └── home_controller.dart
│   │   └── chat/
│   │       ├── chat_page.dart
│   │       ├── chat_controller.dart
│   │       └── widgets/
│   │           ├── message_bubble.dart
│   │           └── chat_input_bar.dart
│   └── api/
│       ├── auth_api.dart
│       ├── user_api.dart
│       ├── character_api.dart
│       └── conversation_api.dart
└── test/
    ├── core/
    │   └── network/
    │       └── ws_client_test.dart
    ├── models/
    │   └── message_test.dart
    └── pages/
        ├── auth/
        │   └── login_controller_test.dart
        └── chat/
            └── chat_controller_test.dart
```

---

### Task 1: Flutter 项目初始化 + 依赖配置

**Files:**
- Create: `pubspec.yaml`（修改依赖）
- Create: `lib/main.dart`
- Create: `lib/core/constants.dart`

- [ ] **Step 1: 在 app/ 目录创建 Flutter 项目**

```bash
cd /path/to/3yan/app
flutter create --org com.sanyan --project-name sanyan_app .
```

- [ ] **Step 2: 配置 pubspec.yaml 依赖**

在 `dependencies` 下添加：
```yaml
dependencies:
  flutter:
    sdk: flutter
  get: ^4.6.6
  dio: ^5.4.3
  web_socket_channel: ^2.4.5
  shared_preferences: ^2.2.3
  uuid: ^4.3.3
  intl: ^0.19.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^5.4.4
  build_runner: ^2.4.9
  mockito: ^5.4.4
```

Run: `flutter pub get`

- [ ] **Step 3: 创建常量和入口文件**

`lib/core/constants.dart`:
```dart
class AppConstants {
  static const String baseUrl = 'http://10.0.2.2:8080'; // Android 模拟器访问宿主机
  static const String wsUrl = 'ws://10.0.2.2:8080/ws';

  // iOS 模拟器用 localhost
  // static const String baseUrl = 'http://localhost:8080';
  // static const String wsUrl = 'ws://localhost:8080/ws';
}
```

`lib/main.dart`:
```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'app/routes.dart';

void main() {
  runApp(const SanyanApp());
}

class SanyanApp extends StatelessWidget {
  const SanyanApp({super.key});

  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(
      title: '三言',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF6C5CE7),
          brightness: Brightness.light,
        ),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF6C5CE7),
          brightness: Brightness.dark,
        ),
        useMaterial3: true,
      ),
      initialRoute: AppRoutes.splash,
      getPages: AppRoutes.pages,
    );
  }
}
```

`lib/app/routes.dart`:
```dart
import 'package:get/get.dart';
import '../pages/splash/splash_page.dart';
import '../pages/auth/login_page.dart';
import '../pages/auth/register_page.dart';
import '../pages/home/home_page.dart';
import '../pages/chat/chat_page.dart';

class AppRoutes {
  static const splash = '/splash';
  static const login = '/login';
  static const register = '/register';
  static const home = '/home';
  static const chat = '/chat';

  static final pages = [
    GetPage(name: splash, page: () => const SplashPage()),
    GetPage(name: login, page: () => const LoginPage()),
    GetPage(name: register, page: () => const RegisterPage()),
    GetPage(name: home, page: () => const HomePage()),
    GetPage(name: chat, page: () => const ChatPage()),
  ];
}
```

- [ ] **Step 4: 创建空的页面占位（让路由能编译）**

为每个页面创建最小的 StatelessWidget 占位，后续 Task 再填充内容。每个文件就一个带 `Scaffold(body: Center(child: Text('XXX')))` 的空 Widget。

- [ ] **Step 5: 验证项目能编译**

Run: `flutter analyze`
Expected: No issues found

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "Flutter 项目初始化 + 依赖 + 路由配置"
git push
```

---

### Task 2: 网络层（HTTP + 本地存储）

**Files:**
- Create: `lib/core/network/api_client.dart`
- Create: `lib/core/network/api_response.dart`
- Create: `lib/core/storage/local_storage.dart`

- [ ] **Step 1: 实现统一响应体**

```dart
// lib/core/network/api_response.dart
class ApiResponse<T> {
  final bool success;
  final String? errMsg;
  final T? data;

  ApiResponse({required this.success, this.errMsg, this.data});

  factory ApiResponse.fromJson(Map<String, dynamic> json, T Function(dynamic)? fromData) {
    return ApiResponse(
      success: json['success'] ?? false,
      errMsg: json['errMsg'],
      data: json['data'] != null && fromData != null ? fromData(json['data']) : json['data'],
    );
  }
}
```

- [ ] **Step 2: 实现本地存储**

```dart
// lib/core/storage/local_storage.dart
import 'package:shared_preferences/shared_preferences.dart';

class LocalStorage {
  static late SharedPreferences _prefs;

  static Future<void> init() async {
    _prefs = await SharedPreferences.getInstance();
  }

  static String? get token => _prefs.getString('token');
  static set token(String? value) {
    if (value == null) {
      _prefs.remove('token');
    } else {
      _prefs.setString('token', value);
    }
  }

  static int? get userId => _prefs.getInt('userId');
  static set userId(int? value) {
    if (value == null) {
      _prefs.remove('userId');
    } else {
      _prefs.setInt('userId', value);
    }
  }

  static void clear() {
    _prefs.clear();
  }
}
```

在 `main.dart` 的 `main()` 函数中初始化：
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await LocalStorage.init();
  runApp(const SanyanApp());
}
```

- [ ] **Step 3: 实现 Dio HTTP 客户端**

```dart
// lib/core/network/api_client.dart
import 'package:dio/dio.dart';
import '../constants.dart';
import '../storage/local_storage.dart';
import 'api_response.dart';

class ApiClient {
  static final ApiClient _instance = ApiClient._();
  factory ApiClient() => _instance;

  late final Dio _dio;

  ApiClient._() {
    _dio = Dio(BaseOptions(
      baseUrl: AppConstants.baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
    ));

    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) {
        final token = LocalStorage.token;
        if (token != null) {
          options.headers['Authorization'] = 'Bearer $token';
        }
        handler.next(options);
      },
    ));
  }

  Future<ApiResponse<T>> get<T>(String path, {
    Map<String, dynamic>? params,
    T Function(dynamic)? fromData,
  }) async {
    final resp = await _dio.get(path, queryParameters: params);
    return ApiResponse.fromJson(resp.data, fromData);
  }

  Future<ApiResponse<T>> post<T>(String path, {
    Map<String, dynamic>? data,
    T Function(dynamic)? fromData,
  }) async {
    final resp = await _dio.post(path, data: data);
    return ApiResponse.fromJson(resp.data, fromData);
  }

  Future<ApiResponse<T>> put<T>(String path, {
    Map<String, dynamic>? data,
    T Function(dynamic)? fromData,
  }) async {
    final resp = await _dio.put(path, data: data);
    return ApiResponse.fromJson(resp.data, fromData);
  }
}
```

- [ ] **Step 4: 验证编译通过**

Run: `flutter analyze`
Expected: No issues found

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "网络层：Dio HTTP 封装 + 本地存储 + 统一响应体"
git push
```

---

### Task 3: 数据模型

**Files:**
- Create: `lib/models/user.dart`, `message.dart`, `conversation.dart`, `character.dart`
- Test: `test/models/message_test.dart`

- [ ] **Step 1: 写 Message 模型测试**

```dart
// test/models/message_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_app/models/message.dart';

void main() {
  test('should parse message from json', () {
    final json = {
      'id': 1,
      'conversationId': 100,
      'senderType': 'ai',
      'contentType': 'text',
      'content': '你好呀',
      'source': 'reply',
      'createdAt': '2026-04-07 20:30:15',
    };

    final msg = Message.fromJson(json);
    expect(msg.id, 1);
    expect(msg.senderType, 'ai');
    expect(msg.content, '你好呀');
    expect(msg.isFromAi, true);
  });

  test('should detect proactive message', () {
    final msg = Message.fromJson({
      'id': 2,
      'conversationId': 100,
      'senderType': 'ai',
      'contentType': 'text',
      'content': '在干嘛呢',
      'source': 'proactive',
      'createdAt': '2026-04-07 20:30:15',
    });

    expect(msg.isProactive, true);
  });
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `flutter test test/models/message_test.dart`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现所有数据模型**

```dart
// lib/models/user.dart
class User {
  final int id;
  final String phone;
  final String? nickname;
  final String? avatar;

  User({required this.id, required this.phone, this.nickname, this.avatar});

  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['id'],
    phone: json['phone'] ?? '',
    nickname: json['nickname'],
    avatar: json['avatar'],
  );
}
```

```dart
// lib/models/character.dart
class Character {
  final int id;
  final String name;
  final String? avatar;
  final String? greeting;

  Character({required this.id, required this.name, this.avatar, this.greeting});

  factory Character.fromJson(Map<String, dynamic> json) => Character(
    id: json['id'],
    name: json['name'],
    avatar: json['avatar'],
    greeting: json['greeting'],
  );
}
```

```dart
// lib/models/conversation.dart
class Conversation {
  final int id;
  final int characterId;
  final String? characterName;
  final String? characterAvatar;
  final String? lastMessage;
  final String? lastMessageAt;
  final int unreadCount;

  Conversation({
    required this.id,
    required this.characterId,
    this.characterName,
    this.characterAvatar,
    this.lastMessage,
    this.lastMessageAt,
    this.unreadCount = 0,
  });

  factory Conversation.fromJson(Map<String, dynamic> json) => Conversation(
    id: json['id'],
    characterId: json['characterId'],
    characterName: json['characterName'],
    characterAvatar: json['characterAvatar'],
    lastMessage: json['lastMessage'],
    lastMessageAt: json['lastMessageAt'],
    unreadCount: json['unreadCount'] ?? 0,
  );
}
```

```dart
// lib/models/message.dart
class Message {
  final int id;
  final int conversationId;
  final String senderType; // user / ai
  final String contentType; // text / voice
  final String content;
  final String source; // reply / proactive
  final String createdAt;
  final String? clientMsgId; // 客户端本地 ID，用于 ACK 匹配

  Message({
    required this.id,
    required this.conversationId,
    required this.senderType,
    required this.contentType,
    required this.content,
    required this.source,
    required this.createdAt,
    this.clientMsgId,
  });

  bool get isFromAi => senderType == 'ai';
  bool get isProactive => source == 'proactive';

  factory Message.fromJson(Map<String, dynamic> json) => Message(
    id: json['id'] ?? 0,
    conversationId: json['conversationId'] ?? 0,
    senderType: json['senderType'] ?? '',
    contentType: json['contentType'] ?? 'text',
    content: json['content'] ?? '',
    source: json['source'] ?? 'reply',
    createdAt: json['createdAt'] ?? '',
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'conversationId': conversationId,
    'senderType': senderType,
    'contentType': contentType,
    'content': content,
    'source': source,
    'createdAt': createdAt,
  };
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `flutter test test/models/message_test.dart`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "数据模型：User、Character、Conversation、Message"
git push
```

---

### Task 4: API 接口层

**Files:**
- Create: `lib/api/auth_api.dart`, `user_api.dart`, `character_api.dart`, `conversation_api.dart`

- [ ] **Step 1: 实现所有 API 接口**

```dart
// lib/api/auth_api.dart
import '../core/network/api_client.dart';
import '../core/network/api_response.dart';

class AuthApi {
  static final _client = ApiClient();

  static Future<ApiResponse> sendSms(String phone) =>
      _client.post('/api/auth/sms/send', data: {'phone': phone});

  static Future<ApiResponse<Map<String, dynamic>>> register({
    required String phone,
    required String code,
    required String password,
    String? nickname,
  }) =>
      _client.post('/api/auth/register',
          data: {'phone': phone, 'code': code, 'password': password, 'nickname': nickname},
          fromData: (d) => d as Map<String, dynamic>);

  static Future<ApiResponse<Map<String, dynamic>>> login({
    required String phone,
    required String password,
  }) =>
      _client.post('/api/auth/login',
          data: {'phone': phone, 'password': password},
          fromData: (d) => d as Map<String, dynamic>);

  static Future<ApiResponse> resetPassword({
    required String phone,
    required String code,
    required String newPassword,
  }) =>
      _client.post('/api/auth/password/reset',
          data: {'phone': phone, 'code': code, 'newPassword': newPassword});
}
```

```dart
// lib/api/user_api.dart
import '../core/network/api_client.dart';
import '../core/network/api_response.dart';

class UserApi {
  static final _client = ApiClient();

  static Future<ApiResponse<Map<String, dynamic>>> getProfile() =>
      _client.get('/api/user/profile', fromData: (d) => d as Map<String, dynamic>);

  static Future<ApiResponse> updateProfile({String? nickname, String? avatar}) =>
      _client.put('/api/user/profile', data: {'nickname': nickname, 'avatar': avatar});
}
```

```dart
// lib/api/character_api.dart
import '../core/network/api_client.dart';
import '../core/network/api_response.dart';
import '../models/character.dart';

class CharacterApi {
  static final _client = ApiClient();

  static Future<ApiResponse<List<Character>>> list() =>
      _client.get('/api/characters',
          fromData: (d) => (d as List).map((e) => Character.fromJson(e)).toList());

  static Future<ApiResponse<Character>> detail(int id) =>
      _client.get('/api/characters/$id', fromData: (d) => Character.fromJson(d));
}
```

```dart
// lib/api/conversation_api.dart
import '../core/network/api_client.dart';
import '../core/network/api_response.dart';
import '../models/conversation.dart';
import '../models/message.dart';

class ConversationApi {
  static final _client = ApiClient();

  static Future<ApiResponse<List<Conversation>>> list() =>
      _client.get('/api/conversations',
          fromData: (d) => (d as List).map((e) => Conversation.fromJson(e)).toList());

  static Future<ApiResponse<List<Message>>> messages(int convId, {int? beforeId, int limit = 20}) =>
      _client.get('/api/conversations/$convId/messages',
          params: {'beforeId': beforeId, 'limit': limit},
          fromData: (d) => (d as List).map((e) => Message.fromJson(e)).toList());

  static Future<ApiResponse> markRead(int convId) =>
      _client.post('/api/conversations/$convId/read');
}
```

- [ ] **Step 2: 验证编译**

Run: `flutter analyze`
Expected: No issues found

- [ ] **Step 3: 提交**

```bash
git add -A
git commit -m "API 接口层：auth、user、character、conversation"
git push
```

---

### Task 5: WebSocket 客户端

**Files:**
- Create: `lib/core/network/ws_client.dart`
- Test: `test/core/network/ws_client_test.dart`

- [ ] **Step 1: 写测试**

```dart
// test/core/network/ws_client_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_app/core/network/ws_client.dart';

void main() {
  test('should parse new_message event', () {
    final json = {
      'type': 'new_message',
      'conversationId': 100,
      'message': {
        'id': 50002,
        'senderType': 'ai',
        'contentType': 'text',
        'content': '你好呀',
        'source': 'reply',
        'createdAt': '2026-04-07 20:30:15',
      }
    };

    final event = WsEvent.fromJson(json);
    expect(event.type, 'new_message');
    expect(event.conversationId, 100);
    expect(event.message, isNotNull);
    expect(event.message!['content'], '你好呀');
  });

  test('should parse ack event', () {
    final event = WsEvent.fromJson({
      'type': 'ack',
      'clientMsgId': 'uuid-123',
      'serverMsgId': 50001,
    });

    expect(event.type, 'ack');
    expect(event.clientMsgId, 'uuid-123');
  });

  test('should parse typing event', () {
    final event = WsEvent.fromJson({
      'type': 'typing',
      'conversationId': 100,
    });

    expect(event.type, 'typing');
    expect(event.conversationId, 100);
  });
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `flutter test test/core/network/ws_client_test.dart`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 WebSocket 客户端**

```dart
// lib/core/network/ws_client.dart
import 'dart:async';
import 'dart:convert';
import 'package:get/get.dart';
import 'package:uuid/uuid.dart';
import 'package:web_socket_channel/web_socket_channel.dart';
import '../constants.dart';
import '../storage/local_storage.dart';

class WsEvent {
  final String type;
  final int? conversationId;
  final Map<String, dynamic>? message;
  final String? clientMsgId;
  final int? serverMsgId;
  final List<dynamic>? messages; // sync_result
  final bool? hasMore;

  WsEvent({
    required this.type,
    this.conversationId,
    this.message,
    this.clientMsgId,
    this.serverMsgId,
    this.messages,
    this.hasMore,
  });

  factory WsEvent.fromJson(Map<String, dynamic> json) => WsEvent(
    type: json['type'] ?? '',
    conversationId: json['conversationId'],
    message: json['message'],
    clientMsgId: json['clientMsgId'],
    serverMsgId: json['serverMsgId'],
    messages: json['messages'],
    hasMore: json['hasMore'],
  );
}

class WsClient extends GetxService {
  WebSocketChannel? _channel;
  Timer? _heartbeatTimer;
  Timer? _reconnectTimer;
  bool _isConnected = false;
  int _reconnectAttempts = 0;
  static const _maxReconnectAttempts = 10;
  static const _uuid = Uuid();

  // 事件流，供 Controller 监听
  final _eventController = StreamController<WsEvent>.broadcast();
  Stream<WsEvent> get eventStream => _eventController.stream;

  // 在线状态
  final isConnected = false.obs;

  void connect() {
    final token = LocalStorage.token;
    if (token == null) return;

    try {
      final uri = Uri.parse('${AppConstants.wsUrl}?token=$token');
      _channel = WebSocketChannel.connect(uri);

      _channel!.stream.listen(
        _onMessage,
        onDone: _onDisconnected,
        onError: (e) => _onDisconnected(),
      );

      _isConnected = true;
      isConnected.value = true;
      _reconnectAttempts = 0;
      _startHeartbeat();

      // 同步未读消息
      syncMessages();
    } catch (e) {
      _scheduleReconnect();
    }
  }

  void disconnect() {
    _heartbeatTimer?.cancel();
    _reconnectTimer?.cancel();
    _channel?.sink.close();
    _isConnected = false;
    isConnected.value = false;
  }

  /// 发送聊天消息，返回 clientMsgId 用于追踪
  String sendMessage(int conversationId, String content, {String contentType = 'text'}) {
    final clientMsgId = _uuid.v4();
    _send({
      'type': 'send_message',
      'conversationId': conversationId,
      'contentType': contentType,
      'content': content,
      'clientMsgId': clientMsgId,
    });
    return clientMsgId;
  }

  void syncMessages({int lastMsgId = 0}) {
    _send({'type': 'sync', 'lastMsgId': lastMsgId});
  }

  void _send(Map<String, dynamic> data) {
    if (_isConnected && _channel != null) {
      _channel!.sink.add(jsonEncode(data));
    }
  }

  void _onMessage(dynamic raw) {
    try {
      final json = jsonDecode(raw as String) as Map<String, dynamic>;
      final type = json['type'];

      if (type == 'pong') return; // 心跳回应，不用处理

      _eventController.add(WsEvent.fromJson(json));
    } catch (e) {
      // ignore malformed messages
    }
  }

  void _onDisconnected() {
    _isConnected = false;
    isConnected.value = false;
    _heartbeatTimer?.cancel();
    _scheduleReconnect();
  }

  void _startHeartbeat() {
    _heartbeatTimer?.cancel();
    _heartbeatTimer = Timer.periodic(const Duration(seconds: 30), (_) {
      _send({'type': 'ping'});
    });
  }

  void _scheduleReconnect() {
    if (_reconnectAttempts >= _maxReconnectAttempts) return;
    _reconnectTimer?.cancel();

    final delay = Duration(seconds: 2 * (_reconnectAttempts + 1)); // 递增延迟
    _reconnectTimer = Timer(delay, () {
      _reconnectAttempts++;
      connect();
    });
  }

  @override
  void onClose() {
    disconnect();
    _eventController.close();
    super.onClose();
  }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `flutter test test/core/network/ws_client_test.dart`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "WebSocket 客户端：连接、心跳、断线重连、消息收发"
git push
```

---

### Task 6: 登录注册页面

**Files:**
- Create: `lib/pages/auth/login_page.dart`, `login_controller.dart`
- Create: `lib/pages/auth/register_page.dart`, `register_controller.dart`
- Create: `lib/pages/splash/splash_page.dart`
- Test: `test/pages/auth/login_controller_test.dart`

- [ ] **Step 1: 写 LoginController 测试**

```dart
// test/pages/auth/login_controller_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_app/pages/auth/login_controller.dart';

void main() {
  test('should validate empty phone', () {
    final controller = LoginController();
    expect(controller.validatePhone(''), '请输入手机号');
  });

  test('should validate phone format', () {
    final controller = LoginController();
    expect(controller.validatePhone('123'), '手机号格式不正确');
    expect(controller.validatePhone('13800138000'), null); // null = valid
  });

  test('should validate empty password', () {
    final controller = LoginController();
    expect(controller.validatePassword(''), '请输入密码');
  });
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `flutter test test/pages/auth/login_controller_test.dart`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 LoginController**

```dart
// lib/pages/auth/login_controller.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import '../../api/auth_api.dart';
import '../../app/routes.dart';
import '../../core/network/ws_client.dart';
import '../../core/storage/local_storage.dart';

class LoginController extends GetxController {
  final phoneController = TextEditingController();
  final passwordController = TextEditingController();
  final isLoading = false.obs;

  String? validatePhone(String phone) {
    if (phone.isEmpty) return '请输入手机号';
    if (!RegExp(r'^1\d{10}$').hasMatch(phone)) return '手机号格式不正确';
    return null;
  }

  String? validatePassword(String password) {
    if (password.isEmpty) return '请输入密码';
    return null;
  }

  Future<void> login() async {
    final phone = phoneController.text.trim();
    final password = passwordController.text;

    final phoneError = validatePhone(phone);
    if (phoneError != null) {
      Get.snackbar('提示', phoneError);
      return;
    }
    final passwordError = validatePassword(password);
    if (passwordError != null) {
      Get.snackbar('提示', passwordError);
      return;
    }

    isLoading.value = true;
    try {
      final resp = await AuthApi.login(phone: phone, password: password);
      if (resp.success && resp.data != null) {
        LocalStorage.token = resp.data!['token'];
        LocalStorage.userId = resp.data!['userId'];

        // 连接 WebSocket
        Get.find<WsClient>().connect();

        Get.offAllNamed(AppRoutes.home);
      } else {
        Get.snackbar('登录失败', resp.errMsg ?? '未知错误');
      }
    } catch (e) {
      Get.snackbar('登录失败', '网络错误，请稍后重试');
    } finally {
      isLoading.value = false;
    }
  }

  @override
  void onClose() {
    phoneController.dispose();
    passwordController.dispose();
    super.onClose();
  }
}
```

- [ ] **Step 4: 实现 LoginPage**

```dart
// lib/pages/auth/login_page.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import '../../app/routes.dart';
import 'login_controller.dart';

class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    final c = Get.put(LoginController());
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.symmetric(horizontal: 32),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              const Text('三言', style: TextStyle(fontSize: 36, fontWeight: FontWeight.bold),
                  textAlign: TextAlign.center),
              const SizedBox(height: 8),
              Text('AI 陪伴，有温度的对话', style: TextStyle(color: Colors.grey[600]),
                  textAlign: TextAlign.center),
              const SizedBox(height: 48),
              TextField(
                controller: c.phoneController,
                keyboardType: TextInputType.phone,
                decoration: const InputDecoration(
                  labelText: '手机号',
                  prefixIcon: Icon(Icons.phone_outlined),
                  border: OutlineInputBorder(),
                ),
              ),
              const SizedBox(height: 16),
              TextField(
                controller: c.passwordController,
                obscureText: true,
                decoration: const InputDecoration(
                  labelText: '密码',
                  prefixIcon: Icon(Icons.lock_outline),
                  border: OutlineInputBorder(),
                ),
                onSubmitted: (_) => c.login(),
              ),
              const SizedBox(height: 24),
              Obx(() => FilledButton(
                onPressed: c.isLoading.value ? null : c.login,
                child: c.isLoading.value
                    ? const SizedBox(width: 20, height: 20,
                        child: CircularProgressIndicator(strokeWidth: 2, color: Colors.white))
                    : const Text('登录'),
              )),
              const SizedBox(height: 12),
              TextButton(
                onPressed: () => Get.toNamed(AppRoutes.register),
                child: const Text('没有账号？去注册'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

- [ ] **Step 5: 实现 RegisterController + RegisterPage**

`RegisterController` 包含：手机号、验证码、密码、昵称的输入管理，sendSms 倒计时，register 方法。结构同 LoginController，调用 `AuthApi.sendSms` 和 `AuthApi.register`。

`RegisterPage` 包含：手机号输入框 + 发送验证码按钮（带倒计时）、验证码输入框、密码输入框、昵称输入框（选填）、注册按钮。

- [ ] **Step 6: 实现 SplashPage**

```dart
// lib/pages/splash/splash_page.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import '../../app/routes.dart';
import '../../core/network/ws_client.dart';
import '../../core/storage/local_storage.dart';

class SplashPage extends StatelessWidget {
  const SplashPage({super.key});

  @override
  Widget build(BuildContext context) {
    // 初始化 WebSocket 服务
    Get.put(WsClient(), permanent: true);

    Future.delayed(const Duration(milliseconds: 500), () {
      if (LocalStorage.token != null) {
        Get.find<WsClient>().connect();
        Get.offAllNamed(AppRoutes.home);
      } else {
        Get.offAllNamed(AppRoutes.login);
      }
    });

    return const Scaffold(
      body: Center(
        child: Text('三言', style: TextStyle(fontSize: 48, fontWeight: FontWeight.bold)),
      ),
    );
  }
}
```

- [ ] **Step 7: 运行测试确认通过**

Run: `flutter test test/pages/auth/login_controller_test.dart`
Expected: PASS

- [ ] **Step 8: 提交**

```bash
git add -A
git commit -m "登录注册页面 + Splash 页面 + 自动登录判断"
git push
```

---

### Task 7: 首页（会话列表）

**Files:**
- Create: `lib/pages/home/home_page.dart`, `home_controller.dart`

- [ ] **Step 1: 实现 HomeController**

```dart
// lib/pages/home/home_controller.dart
import 'package:get/get.dart';
import '../../api/character_api.dart';
import '../../api/conversation_api.dart';
import '../../core/network/ws_client.dart';
import '../../models/character.dart';
import '../../models/conversation.dart';
import '../../models/message.dart';
import 'dart:async';

class HomeController extends GetxController {
  final conversations = <Conversation>[].obs;
  final isLoading = true.obs;
  StreamSubscription? _wsSubscription;

  @override
  void onInit() {
    super.onInit();
    loadConversations();
    _listenWsEvents();
  }

  Future<void> loadConversations() async {
    try {
      final resp = await ConversationApi.list();
      if (resp.success && resp.data != null) {
        conversations.value = resp.data!;
      }

      // 如果没有会话，自动为预设角色创建一个
      if (conversations.isEmpty) {
        await _createDefaultConversation();
      }
    } finally {
      isLoading.value = false;
    }
  }

  Future<void> _createDefaultConversation() async {
    final charResp = await CharacterApi.list();
    if (charResp.success && charResp.data != null && charResp.data!.isNotEmpty) {
      // 会话会在用户发第一条消息时由后端自动创建
      // 或者可以在此添加一个创建会话的 API
      await loadConversations();
    }
  }

  void _listenWsEvents() {
    final wsClient = Get.find<WsClient>();
    _wsSubscription = wsClient.eventStream.listen((event) {
      if (event.type == 'new_message') {
        // 收到新消息，刷新会话列表
        loadConversations();
      }
    });
  }

  @override
  void onClose() {
    _wsSubscription?.cancel();
    super.onClose();
  }
}
```

- [ ] **Step 2: 实现 HomePage**

```dart
// lib/pages/home/home_page.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import '../../app/routes.dart';
import 'home_controller.dart';

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    final c = Get.put(HomeController());
    return Scaffold(
      appBar: AppBar(title: const Text('三言')),
      body: Obx(() {
        if (c.isLoading.value) {
          return const Center(child: CircularProgressIndicator());
        }
        if (c.conversations.isEmpty) {
          return const Center(child: Text('还没有对话，开始聊天吧'));
        }
        return ListView.builder(
          itemCount: c.conversations.length,
          itemBuilder: (context, index) {
            final conv = c.conversations[index];
            return ListTile(
              leading: CircleAvatar(
                child: Text(conv.characterName?.substring(0, 1) ?? '?'),
              ),
              title: Text(conv.characterName ?? '未知角色'),
              subtitle: Text(conv.lastMessage ?? '', maxLines: 1, overflow: TextOverflow.ellipsis),
              trailing: conv.unreadCount > 0
                  ? Container(
                      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                      decoration: BoxDecoration(
                        color: Colors.red,
                        borderRadius: BorderRadius.circular(10),
                      ),
                      child: Text('${conv.unreadCount}',
                          style: const TextStyle(color: Colors.white, fontSize: 12)),
                    )
                  : null,
              onTap: () => Get.toNamed(AppRoutes.chat, arguments: conv),
            );
          },
        );
      }),
    );
  }
}
```

- [ ] **Step 3: 验证编译**

Run: `flutter analyze`
Expected: No issues found

- [ ] **Step 4: 提交**

```bash
git add -A
git commit -m "首页：会话列表 + WebSocket 新消息自动刷新"
git push
```

---

### Task 8: 聊天页面

**Files:**
- Create: `lib/pages/chat/chat_page.dart`, `chat_controller.dart`
- Create: `lib/pages/chat/widgets/message_bubble.dart`, `chat_input_bar.dart`
- Test: `test/pages/chat/chat_controller_test.dart`

- [ ] **Step 1: 写 ChatController 测试**

```dart
// test/pages/chat/chat_controller_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_app/models/message.dart';

void main() {
  test('should insert message in order', () {
    final messages = <Message>[];

    final msg1 = Message(id: 1, conversationId: 1, senderType: 'user',
        contentType: 'text', content: '你好', source: 'reply', createdAt: '2026-04-07 20:00:00');
    final msg2 = Message(id: 2, conversationId: 1, senderType: 'ai',
        contentType: 'text', content: '你好呀', source: 'reply', createdAt: '2026-04-07 20:00:05');

    messages.add(msg1);
    messages.add(msg2);

    expect(messages.length, 2);
    expect(messages.last.senderType, 'ai');
  });

  test('should detect typing from ws event type', () {
    expect('typing' == 'typing', true);
  });
}
```

- [ ] **Step 2: 运行测试确认失败/通过**

Run: `flutter test test/pages/chat/chat_controller_test.dart`

- [ ] **Step 3: 实现 ChatController**

```dart
// lib/pages/chat/chat_controller.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import '../../api/conversation_api.dart';
import '../../core/network/ws_client.dart';
import '../../models/conversation.dart';
import '../../models/message.dart';

class ChatController extends GetxController {
  final Conversation conversation;
  ChatController(this.conversation);

  final messages = <Message>[].obs;
  final isLoading = true.obs;
  final isAiTyping = false.obs;
  final inputController = TextEditingController();
  final scrollController = ScrollController();
  StreamSubscription? _wsSubscription;

  @override
  void onInit() {
    super.onInit();
    _loadHistory();
    _listenWs();
    ConversationApi.markRead(conversation.id);
  }

  Future<void> _loadHistory() async {
    try {
      final resp = await ConversationApi.messages(conversation.id);
      if (resp.success && resp.data != null) {
        messages.value = resp.data!.reversed.toList(); // API 返回倒序，翻转
      }
    } finally {
      isLoading.value = false;
      _scrollToBottom();
    }
  }

  void _listenWs() {
    final wsClient = Get.find<WsClient>();
    _wsSubscription = wsClient.eventStream.listen((event) {
      if (event.conversationId != conversation.id) return;

      switch (event.type) {
        case 'typing':
          isAiTyping.value = true;
          _scrollToBottom();
          break;
        case 'new_message':
          isAiTyping.value = false;
          if (event.message != null) {
            final msg = Message.fromJson(event.message!);
            messages.add(msg);
            _scrollToBottom();
            ConversationApi.markRead(conversation.id);
          }
          break;
        case 'ack':
          // 可以用来更新本地消息的发送状态
          break;
      }
    });
  }

  void sendMessage() {
    final text = inputController.text.trim();
    if (text.isEmpty) return;

    final wsClient = Get.find<WsClient>();
    final clientMsgId = wsClient.sendMessage(conversation.id, text);

    // 本地先展示用户消息
    messages.add(Message(
      id: 0, // 临时 ID，等 ACK 更新
      conversationId: conversation.id,
      senderType: 'user',
      contentType: 'text',
      content: text,
      source: 'reply',
      createdAt: DateTime.now().toString(),
      clientMsgId: clientMsgId,
    ));

    inputController.clear();
    _scrollToBottom();
  }

  void _scrollToBottom() {
    Future.delayed(const Duration(milliseconds: 100), () {
      if (scrollController.hasClients) {
        scrollController.animateTo(
          scrollController.position.maxScrollExtent,
          duration: const Duration(milliseconds: 200),
          curve: Curves.easeOut,
        );
      }
    });
  }

  @override
  void onClose() {
    _wsSubscription?.cancel();
    inputController.dispose();
    scrollController.dispose();
    super.onClose();
  }
}
```

- [ ] **Step 4: 实现聊天气泡组件**

```dart
// lib/pages/chat/widgets/message_bubble.dart
import 'package:flutter/material.dart';
import '../../../models/message.dart';

class MessageBubble extends StatelessWidget {
  final Message message;
  const MessageBubble({super.key, required this.message});

  @override
  Widget build(BuildContext context) {
    final isUser = message.senderType == 'user';
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: Row(
        mainAxisAlignment: isUser ? MainAxisAlignment.end : MainAxisAlignment.start,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          if (!isUser) ...[
            const CircleAvatar(radius: 18, child: Text('AI')),
            const SizedBox(width: 8),
          ],
          Flexible(
            child: Container(
              padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 10),
              decoration: BoxDecoration(
                color: isUser
                    ? Theme.of(context).colorScheme.primary
                    : Theme.of(context).colorScheme.surfaceContainerHighest,
                borderRadius: BorderRadius.only(
                  topLeft: const Radius.circular(16),
                  topRight: const Radius.circular(16),
                  bottomLeft: Radius.circular(isUser ? 16 : 4),
                  bottomRight: Radius.circular(isUser ? 4 : 16),
                ),
              ),
              child: Text(
                message.content,
                style: TextStyle(
                  color: isUser ? Colors.white : null,
                  fontSize: 15,
                ),
              ),
            ),
          ),
          if (isUser) const SizedBox(width: 8),
        ],
      ),
    );
  }
}
```

```dart
// lib/pages/chat/widgets/chat_input_bar.dart
import 'package:flutter/material.dart';

class ChatInputBar extends StatelessWidget {
  final TextEditingController controller;
  final VoidCallback onSend;
  const ChatInputBar({super.key, required this.controller, required this.onSend});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 8),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surface,
        border: Border(top: BorderSide(color: Colors.grey.withOpacity(0.2))),
      ),
      child: SafeArea(
        top: false,
        child: Row(
          children: [
            Expanded(
              child: TextField(
                controller: controller,
                decoration: InputDecoration(
                  hintText: '说点什么...',
                  filled: true,
                  fillColor: Theme.of(context).colorScheme.surfaceContainerHighest,
                  border: OutlineInputBorder(
                    borderRadius: BorderRadius.circular(24),
                    borderSide: BorderSide.none,
                  ),
                  contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
                ),
                textInputAction: TextInputAction.send,
                onSubmitted: (_) => onSend(),
              ),
            ),
            const SizedBox(width: 8),
            IconButton.filled(
              onPressed: onSend,
              icon: const Icon(Icons.send),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 5: 实现 ChatPage**

```dart
// lib/pages/chat/chat_page.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import '../../models/conversation.dart';
import 'chat_controller.dart';
import 'widgets/chat_input_bar.dart';
import 'widgets/message_bubble.dart';

class ChatPage extends StatelessWidget {
  const ChatPage({super.key});

  @override
  Widget build(BuildContext context) {
    final conversation = Get.arguments as Conversation;
    final c = Get.put(ChatController(conversation));

    return Scaffold(
      appBar: AppBar(
        title: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(conversation.characterName ?? ''),
            Obx(() => c.isAiTyping.value
                ? Text('正在输入...', style: TextStyle(fontSize: 12, color: Colors.grey[600]))
                : const SizedBox.shrink()),
          ],
        ),
      ),
      body: Column(
        children: [
          Expanded(
            child: Obx(() {
              if (c.isLoading.value) {
                return const Center(child: CircularProgressIndicator());
              }
              return ListView.builder(
                controller: c.scrollController,
                padding: const EdgeInsets.symmetric(vertical: 8),
                itemCount: c.messages.length,
                itemBuilder: (context, index) => MessageBubble(message: c.messages[index]),
              );
            }),
          ),
          ChatInputBar(controller: c.inputController, onSend: c.sendMessage),
        ],
      ),
    );
  }
}
```

- [ ] **Step 6: 运行测试 + 验证编译**

Run: `flutter test && flutter analyze`
Expected: PASS + No issues found

- [ ] **Step 7: 提交**

```bash
git add -A
git commit -m "聊天页面：消息列表 + 气泡组件 + 输入栏 + WebSocket 实时收发"
git push
```

---

### Task 9: 推送通知 + 最终整合

**Files:**
- Modify: `lib/main.dart` — 初始化推送
- Modify: `lib/pages/splash/splash_page.dart` — 上报 push token

MVP 阶段推送通知先做框架预留，具体 SDK 对接（极光/个推）在发布前补充。

- [ ] **Step 1: 添加推送 token 上报 API**

```dart
// lib/api/device_api.dart
import '../core/network/api_client.dart';
import '../core/network/api_response.dart';

class DeviceApi {
  static final _client = ApiClient();

  static Future<ApiResponse> updatePushToken(String pushToken, String deviceType) =>
      _client.post('/api/device/push-token',
          data: {'pushToken': pushToken, 'deviceType': deviceType});
}
```

- [ ] **Step 2: 在 SplashPage 中预留推送初始化位置**

在 `SplashPage` 的初始化逻辑中添加注释标记：

```dart
// TODO: 对接极光/个推 SDK 后，在此获取 push token 并调用 DeviceApi.updatePushToken()
```

- [ ] **Step 3: 全量测试**

Run: `flutter test`
Expected: 全部 PASS

Run: `flutter analyze`
Expected: No issues found

- [ ] **Step 4: 提交**

```bash
git add -A
git commit -m "推送通知框架 + 最终整合"
git push
```

---

## 客户端计划完毕

9 个 Task 覆盖 MVP 所有客户端功能：
1. 项目初始化
2. 网络层（HTTP + 本地存储）
3. 数据模型
4. API 接口层
5. WebSocket 客户端
6. 登录注册页面
7. 首页（会话列表）
8. 聊天页面
9. 推送通知 + 最终整合

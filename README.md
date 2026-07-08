# Flutter Clean Architecture 首席架构设计

## Context

绿地新项目(工作目录 `D:\ai\tinode` 下无现存 Flutter 代码)。UI 已用 Stitch 完成设计,需转 Flutter。开发在 Antigravity IDE(Agent 全自动执行 + MCP)。目标是产出一套**稳固、可扩展、类型极严**的整体架构骨架 —— 不含具体 UI Dart 代码,只给结构、流向、转换策略与 Agent 执行清单。

**落地环境说明(重要)**:
- 本机(`D:\ai\tinode`)**未安装 Flutter/Dart SDK** —— 已核查 PATH、`where.exe` 及常见安装路径,均无。因此 plan 中所有 `flutter create` / `dart run build_runner` / `flutter analyze` 命令**须在已装 Flutter 的 Antigravity Agent 环境执行**,不在本机运行。
- 项目落地目录:**`D:\ai\tinode\tinode-flutter\`**(与现存 `tinode-rn` React Native 工程并列,互不干扰;第四节 Step 0 的 `flutter create` 在此目录内执行)。
- 服务端已部署:`ws://149.28.153.26:6060/v0/channels`(Tinode wire 协议,JSON over WebSocket)。

**已确认技术决策**:
- 状态管理:Riverpod + **AsyncNotifier**(响应式异步流)
- 代码生成:**riverpod_generator + freezed**(注解生成 Provider 与不可变实体/状态,`build_runner` 驱动)
- 架构:Clean Architecture 三层严格分离(Data / Domain / Presentation)
- 依赖注入:纯 Riverpod Provider,**拒绝硬编码单例**
- 路由:**go_router**(声明式 + 守卫 + 深链接)

---

## 一、目录树规划

Clean Architecture + Feature-First 混合(顶层按 feature 切,feature 内部按层切)。这是大型可扩展项目最抗腐化的组织方式:横向切层保证依赖方向单一,纵向切 feature 保证模块可独立演进/删除。

```
lib/
├── main.dart                      # 入口: runApp(ProviderScope(...))
├── app.dart                       # MaterialApp.router + 全局主题注入
│
├── core/                          # 跨 feature 共享的基础设施(不含业务)
│   ├── error/
│   │   ├── failures.dart          # freezed sealed Failure(Server/Cache/Network/Auth...)
│   │   └── exceptions.dart        # DataSource 层抛出的原始异常
│   ├── network/
│   │   ├── dio_client.dart        # Dio 实例封装 + 拦截器
│   │   └── network_info.dart      # 连通性检测抽象
│   ├── router/
│   │   ├── app_router.dart        # go_router 配置(@riverpod 提供)
│   │   └── routes.dart            # 路由路径常量 + name 枚举
│   ├── di/
│   │   └── providers.dart         # 跨层基础 Provider(dio/storage/networkInfo)
│   ├── usecase/
│   │   └── usecase.dart           # abstract UseCase<Type, Params> 基类 + NoParams
│   ├── constants/                 # 环境常量、API base url、超时
│   ├── theme/                     # 由 Stitch 设计令牌转换而来(见第三节)
│   │   ├── app_colors.dart        # 颜色令牌
│   │   ├── app_typography.dart    # 字体令牌
│   │   ├── app_spacing.dart       # 间距/圆角令牌
│   │   └── app_theme.dart         # ThemeData 组装
│   └── utils/                     # logger、extensions、formatters
│
├── shared/                        # 跨 feature 复用的 UI 组件(原子/分子)
│   └── widgets/
│       ├── atoms/                 # AppButton / AppTextField / AppAvatar ...
│       └── molecules/             # SearchBar / ListTileCard / FormField ...
│
└── features/
    ├── auth/                      # 示例 feature(登录)
    │   ├── data/
    │   │   ├── models/            # freezed + json_serializable DTO
    │   │   │   └── user_model.dart          # UserModel extends/maps to User
    │   │   ├── datasources/
    │   │   │   ├── auth_remote_datasource.dart   # abstract + impl(调 dio)
    │   │   │   └── auth_local_datasource.dart    # abstract + impl(token 缓存)
    │   │   └── repositories/
    │   │       └── auth_repository_impl.dart     # 实现 domain 接口, try/catch→Either
    │   ├── domain/
    │   │   ├── entities/
    │   │   │   └── user.dart       # freezed 纯业务实体(无 json 依赖)
    │   │   ├── repositories/
    │   │   │   └── auth_repository.dart          # abstract 接口(Domain 定义,Data 实现)
    │   │   └── usecases/
    │   │       ├── login_usecase.dart            # call(LoginParams)→Future<Either<Failure,User>>
    │   │       └── logout_usecase.dart
    │   └── presentation/
    │       ├── controllers/
    │       │   └── auth_controller.dart          # @riverpod AsyncNotifier<AuthState>
    │       ├── state/
    │       │   └── auth_state.dart               # freezed 状态(可选,或直接用 AsyncValue)
    │       ├── pages/
    │       │   └── login_page.dart               # Page 级(ConsumerWidget)
    │       └── widgets/                          # 该 feature 私有的 organisms
    │
    ├── chat/                      # 后续 feature 同构展开
    └── contacts/

test/                              # 镜像 lib 结构: 单测按 usecase/repository/controller
```

**依赖方向铁律**:`Presentation → Domain ← Data`。Domain 层零外部依赖(不 import Flutter/Dio/freezed-json)。Data 层的 Model 映射到 Domain 的 Entity。

---

## 二、数据与状态流向图(以"点击登录"为例)

```
        ┌─────────────────────────── 正向:用户操作驱动 ───────────────────────────┐

  [LoginPage]  用户点击登录按钮
      │  ref.read(authControllerProvider.notifier).login(email, pwd)
      ▼
  [AuthController]  (@riverpod AsyncNotifier<AuthState>)
      │  state = const AsyncLoading();               ← 立即进入 loading
      │  final result = await ref.read(loginUseCaseProvider).call(LoginParams(...));
      ▼
  [LoginUseCase]  call(params) → Future<Either<Failure, User>>
      │  return repository.login(params);            ← 只编排,不含 IO
      ▼
  [AuthRepository] (domain 接口) → [AuthRepositoryImpl] (data 实现)
      │  if(!networkInfo.isConnected) return Left(NetworkFailure());
      │  try { final dto = await remote.login(...); local.cacheToken(dto.token);
      │        return Right(dto.toEntity()); }
      │  catch(ServerException) { return Left(ServerFailure()); }
      ▼
  [AuthRemoteDataSource]  dio.post('/login') → UserModel.fromJson(...)
  [AuthLocalDataSource]   secureStorage.write(token)          (副作用缓存)

        └───────────────────────────────────────────────────────────────────────┘

        ┌────────────────────── 逆向:状态通知回 UI(响应式) ──────────────────────┐

  Either<Failure,User> 冒泡回 [AuthController]
      │  result.fold(
      │    (f) => state = AsyncError(f, StackTrace.current),   ← 失败态
      │    (u) => state = AsyncData(AuthState.authenticated(u)) ← 成功态
      │  );
      ▼
  Riverpod 侦测 state 变更 → 通知所有 watch 该 Provider 的 Widget
      ▼
  [LoginPage]  final authState = ref.watch(authControllerProvider);
      │  authState.when(
      │    loading: () => 显示 Spinner,
      │    error:   (e,_) => 显示 SnackBar/错误文案,
      │    data:    (s) => 触发 go_router 重定向到主页
      │  );

        └───────────────────────────────────────────────────────────────────────┘
```

**要点**:
- 错误用 `Either<Failure, T>`(dartz/fpdart)在 Data↔Domain 间传递,**不抛异常穿层**;异常只存在于 DataSource 内部,Repository 边界捕获转 `Failure`。
- Presentation 层用 `AsyncValue<T>` 天然表达 loading/data/error 三态,无需手写布尔 flag。
- 路由跳转不写在 Controller 里,而由 Page 监听状态或 go_router 的 `redirect` 守卫读 `authControllerProvider` 完成 —— 保持 Controller 无 UI 依赖。
- **DI 全程 Provider 注入**:`loginUseCaseProvider` 依赖 `authRepositoryProvider` 依赖 `authRemoteDataSourceProvider` 依赖 `dioProvider`,层层 `ref.watch`,无一处 `Singleton()` / 全局变量。

---

## 三、Stitch UI 转换策略(原子设计)

Stitch 导出通常是 HTML/CSS 或扁平 Flutter widget 树。转换分两步:**先抽设计令牌,再拆组件层级**。

### 3.1 先固化设计令牌(Design Tokens)→ `core/theme/`
把 Stitch 里散落的颜色/字号/间距/圆角**提取为常量**,而非在组件里写魔法值:
- 颜色 → `app_colors.dart`(`AppColors.primary`, `.surface`, `.error`...)
- 字体 → `app_typography.dart`(映射到 `TextTheme`)
- 间距/圆角 → `app_spacing.dart`(`AppSpacing.md = 16`, `AppRadius.card = 12`)
- 组装进 `app_theme.dart` 的 `ThemeData`,全局 `Theme.of(context)` 取用。

这样 Stitch 改版时只改令牌,组件零改动。

### 3.2 按原子设计三级拆解

| 层级 | 定义 | 放置位置 | Stitch 映射示例 |
|---|---|---|---|
| **Atoms(原子)** | 不可再拆的最小 UI 单元,无业务逻辑,纯 props 驱动 | `shared/widgets/atoms/` | 按钮、输入框、头像、图标、标签、单行文本 |
| **Molecules(分子)** | 几个原子组合成的功能单元,仍无业务/状态 | `shared/widgets/molecules/` | 搜索栏(输入框+图标)、表单项(label+输入框+错误文案)、列表卡片 |
| **Organisms(组织体)** | 分子+原子组成的、带特定 feature 语义的区块,可 `ref.watch` 状态 | `features/<f>/presentation/widgets/` | 登录表单整块、聊天消息气泡列表、联系人列表区 |
| **Pages(页面)** | 一屏路由目标,组装 organisms + 接 Controller | `features/<f>/presentation/pages/` | LoginPage、ChatPage、ContactsPage |

**拆解手法**:
1. 拿到 Stitch 一个页面,自顶向下识别**重复出现**的视觉单元 → 越重复越该下沉为 atom/molecule。
2. Atoms/Molecules 一律**无状态、无 Riverpod、无 context 业务依赖**,只接受参数与回调(`onPressed`),保证可跨 feature 复用、可独立预览(可选接 `widgetbook`)。
3. 业务状态只在 **Organisms 和 Pages** 层用 `ConsumerWidget` / `ref.watch` 注入,把"长得像"和"做什么"彻底解耦。
4. Stitch 生成的内联样式 → 全部替换为第 3.1 节的令牌引用。

---

## 四、Antigravity Agent 自动化执行清单

> 按顺序交给 Antigravity Agent 逐条执行。每步做完让 Agent 运行 `flutter analyze` 确认零错误再进下一步。

### ☐ Step 0 — 初始化项目
- [ ] `flutter create --org com.yourorg --project-name app .`(或指定项目名)
- [ ] 删除默认 `test/widget_test.dart` 与 `lib/main.dart` 样板内容
- [ ] 确认 Flutter 为最新 stable:`flutter --version`

### ☐ Step 1 — 安装依赖(写入 pubspec.yaml)
运行时依赖:
- [ ] `flutter pub add flutter_riverpod riverpod_annotation`
- [ ] `flutter pub add go_router`
- [ ] `flutter pub add freezed_annotation json_annotation`
- [ ] `flutter pub add fpdart`(Either/Failure 函数式错误处理;或用 `dartz`)
- [ ] `flutter pub add dio`(网络)
- [ ] `flutter pub add flutter_secure_storage`(token 安全存储)
- [ ] `flutter pub add equatable`(可选,若某处不用 freezed)

开发依赖(dev_dependencies):
- [ ] `flutter pub add -d build_runner`
- [ ] `flutter pub add -d riverpod_generator`
- [ ] `flutter pub add -d freezed`
- [ ] `flutter pub add -d json_serializable`
- [ ] `flutter pub add -d custom_lint riverpod_lint`(Riverpod 静态检查)
- [ ] `flutter pub add -d flutter_lints`

### ☐ Step 2 — 配置严格 lint 与 codegen
- [ ] 在 `analysis_options.yaml` 启用 `custom_lint` 插件 + `flutter_lints`,开启严格 analyzer(`strict-casts`, `strict-raw-types`)
- [ ] 建立 codegen 常用命令备忘:`dart run build_runner watch -d`(开发期常驻)

### ☐ Step 3 — 搭建 core 骨架
- [ ] 创建 `core/error/failures.dart`(freezed sealed `Failure`)与 `exceptions.dart`
- [ ] 创建 `core/usecase/usecase.dart`(`abstract class UseCase<T, P>` + `NoParams`)
- [ ] 创建 `core/network/dio_client.dart` + `network_info.dart`
- [ ] 创建 `core/di/providers.dart`(`@riverpod dioProvider` / `secureStorageProvider` / `networkInfoProvider`)
- [ ] 创建 `core/theme/`(先放空令牌文件,待第五步从 Stitch 填充)
- [ ] 创建 `core/router/app_router.dart`(`@riverpod GoRouter appRouter(ref)`)+ `routes.dart`

### ☐ Step 4 — 搭建首个 feature(auth)作为模板
- [ ] `domain/entities/user.dart`(freezed 实体)
- [ ] `domain/repositories/auth_repository.dart`(abstract 接口)
- [ ] `domain/usecases/login_usecase.dart`(依赖抽象 repository)
- [ ] `data/models/user_model.dart`(freezed + json_serializable + `toEntity()`)
- [ ] `data/datasources/auth_remote_datasource.dart` / `auth_local_datasource.dart`(abstract + impl)
- [ ] `data/repositories/auth_repository_impl.dart`(实现接口,try/catch→Either)
- [ ] `presentation/controllers/auth_controller.dart`(`@riverpod class AuthController extends _$AuthController` → `AsyncNotifier`)
- [ ] 为以上每个可注入类补 `@riverpod` Provider(datasource→repository→usecase 链)

### ☐ Step 5 — Stitch 转换
- [ ] 从 Stitch 设计提取设计令牌填入 `core/theme/*`
- [ ] 识别并生成 `shared/widgets/atoms/` 原子组件(无状态)
- [ ] 组合出 `shared/widgets/molecules/` 分子组件
- [ ] 在 `features/auth/presentation/pages/login_page.dart` 用 `ConsumerWidget` 组装,`ref.watch(authControllerProvider)` 接状态

### ☐ Step 6 — 组装入口与路由
- [ ] `main.dart`:`runApp(ProviderScope(child: MyApp()))`
- [ ] `app.dart`:`MaterialApp.router(routerConfig: ref.watch(appRouterProvider), theme: AppTheme.light)`
- [ ] go_router `redirect` 读 `authControllerProvider` 做登录守卫

### ☐ Step 7 — 生成代码 + 验证(见下节)
- [ ] `dart run build_runner build --delete-conflicting-outputs`
- [ ] `flutter analyze` 零错误
- [ ] `flutter test` 通过

---

## 五、验证方法

1. **代码生成通过**:`dart run build_runner build --delete-conflicting-outputs` 无冲突、无报错,生成全部 `*.g.dart` / `*.freezed.dart`。
2. **静态零错误**:`flutter analyze`(含 riverpod_lint)输出 0 issues —— 验证依赖方向、Provider 用法合规。
3. **依赖注入无硬编码**:全局搜索确认无 `= Singleton()` / 全局可变实例,所有依赖经 `ref.watch/read` 注入。
4. **分层单测**(镜像 `test/` 结构):
   - UseCase 测试:mock `AuthRepository`,验证 `call` 编排正确。
   - Repository 测试:mock DataSource,验证异常→`Failure` 的 `Either` 转换。
   - Controller 测试:用 `ProviderContainer` override `loginUseCaseProvider`,断言 `state` 依次经历 `AsyncLoading → AsyncData/AsyncError`。
5. **运行冒烟**:`flutter run` 启动到 LoginPage,点击登录观察 loading→结果三态切换与路由跳转。
6. **架构守护(可选)**:引入 `dart_code_metrics` 或自定义 import 规则,CI 中禁止 `domain/` import `flutter`/`dio`/`data/`。

---

## 交付说明

本方案只交付**架构骨架、流向与执行清单**,不含具体 UI 界面 Dart 代码(按你要求省 Token)。落地时 Antigravity Agent 按第四节清单顺序执行;auth feature 作为模板,chat/contacts 等后续 feature 完全同构复制展开即可。

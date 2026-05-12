# Model 生成指南

本项目使用 [`json_serializable`](https://pub.dev/packages/json_serializable) 自动生成 JSON 序列化代码，消除手写 `fromJson` / `toJson` 时容易引入的字段遗漏、类型错误等问题。

---

## 技术栈

| 包 | 角色 |
|----|------|
| `json_annotation` | 提供 `@JsonSerializable`、`@JsonKey`、`@JsonEnum` 注解 |
| `json_serializable` | 读取注解，生成 `*.g.dart` 文件（dev 依赖） |
| `build_runner` | 代码生成运行器（dev 依赖） |

---

## 一键生成

### macOS / Linux

```bash
# 普通构建（推荐日常使用）
bash scripts/gen_models.sh

# 监听模式（开发时自动重新生成）
bash scripts/gen_models.sh watch

# 清理旧文件后全量重建
bash scripts/gen_models.sh clean
```

> 首次运行需赋予执行权限：`chmod +x scripts/gen_models.sh`

### Windows

```bat
rem 普通构建
scripts\gen_models.bat

rem 监听模式
scripts\gen_models.bat watch

rem 清理后重建
scripts\gen_models.bat clean
```

### 手动命令（任意平台）

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs
```

---

## 文件说明

生成器会在每个 Model 文件同级目录下产生对应的 `*.g.dart` 文件：

```
lib/models/
├── user_model.dart           ← 手写源文件（含注解）
├── user_model.g.dart         ← 自动生成，不要手改
├── device_model.dart
└── device_model.g.dart
```

> `*.g.dart` 已在 `.gitignore` 中忽略，**不要提交到版本控制**。  
> 每次拉取代码后执行一次生成脚本即可。

---

## 创建新 Model

### 步骤 1 — 从 JSON 生成初版

1. 打开 [quicktype.io](https://app.quicktype.io)
2. 粘贴接口返回的 JSON 示例，语言选 **Dart**
3. 复制生成的类，粘贴到 `lib/models/your_model.dart`

或使用 VS Code 插件 **Dart Data Class Generator**：右键 JSON → `Generate Dart Data Class`。

### 步骤 2 — 添加注解

```dart
import 'package:json_annotation/json_annotation.dart';

part 'your_model.g.dart';  // ← 必须声明

@JsonSerializable()
class YourModel {
  final String id;

  // 字段名与 JSON key 不同时，用 @JsonKey 映射
  @JsonKey(name: 'created_at')
  final DateTime createdAt;

  // 为可空字段设置默认值
  @JsonKey(defaultValue: 0)
  final int count;

  const YourModel({
    required this.id,
    required this.createdAt,
    this.count = 0,
  });

  factory YourModel.fromJson(Map<String, dynamic> json) =>
      _$YourModelFromJson(json);

  Map<String, dynamic> toJson() => _$YourModelToJson(this);
}
```

### 步骤 3 — 生成代码

```bash
bash scripts/gen_models.sh
```

---

## 常用注解速查

| 注解 | 说明 | 示例 |
|------|------|------|
| `@JsonSerializable()` | 标记一个类需要生成序列化代码 | 类级别 |
| `@JsonKey(name: 'snake_key')` | 映射不同的 JSON 字段名 | `@JsonKey(name: 'user_id')` |
| `@JsonKey(defaultValue: value)` | 字段缺失时的默认值 | `@JsonKey(defaultValue: 0)` |
| `@JsonKey(ignore: true)` | 序列化时忽略该字段 | `@JsonKey(ignore: true)` |
| `@JsonKey(includeIfNull: false)` | `null` 时不写入 JSON | `@JsonKey(includeIfNull: false)` |
| `@JsonEnum()` | 枚举类型序列化为字符串 | 枚举类级别 |
| `@JsonSerializable(explicitToJson: true)` | 嵌套对象也调用 `toJson` | 含嵌套 Model 时使用 |

### 嵌套 Model 示例

```dart
@JsonSerializable(explicitToJson: true)  // ← 嵌套时必须加这个
class OrderModel {
  final String orderId;
  final UserModel user;   // 嵌套另一个 @JsonSerializable 类

  factory OrderModel.fromJson(Map<String, dynamic> json) =>
      _$OrderModelFromJson(json);
  Map<String, dynamic> toJson() => _$OrderModelToJson(this);
}
```

---

## 已有 Model 一览

| 文件 | 类 | 说明 |
|------|----|------|
| `user_model.dart` | `UserModel` | 账号信息（id、email、nickname、avatar） |
| `device_model.dart` | `DeviceModel` | BLE 设备信息（状态、强度、模式、电量等） |

---

## 常见问题

**Q: 为什么修改了字段但运行时报错 `_$XxxFromJson` 未定义？**  
A: 修改 Model 后需要重新运行生成脚本。

**Q: `*.g.dart` 能提交到 Git 吗？**  
A: 推荐加入 `.gitignore` 不提交，团队成员拉取代码后各自生成。如果 CI/CD 环境需要，可以提交，但必须保证每次修改 Model 后都重新生成并一起提交。

**Q: build_runner 报冲突如何解决？**  
A: 使用 clean 模式：`bash scripts/gen_models.sh clean`

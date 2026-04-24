# TODO

## 待补充规则（按需决定，不急）

- [ ] `~/.claude/rules/flutter-business-layer.md`：补"业务模块间依赖约束必须由工具守护"
  - CI 脚本扫 pubspec 白名单：业务包 dependencies 只能是 `flutter` / `get` / `foundation_packages/*`
  - 配 `custom_lint` 规则：business_packages 下的 `.dart` 文件禁止 `import 'package:<proj>_<另一业务包>/...'`
  - 对齐后端的 Maven Enforcer（`banned-dependencies` 规则）+ ArchUnit 机制
  - 触发条件：下次整理 CI / 工具链时一起做

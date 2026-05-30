# Bulk Stonecutting — Fabric 1.21.11 移植版

原模组 `bulk_stonecutting-master` 从 **Minecraft 1.21.4 → 1.21.11** 的移植版本。

## 功能

在切石机（Stonecutter）界面添加"Mass Crafting"复选框，勾选后 Shift+左键 点击输出槽位，会自动从背包中取出一组相同材料放入输入槽，实现批量切石。

## 环境要求

| 组件 | 版本 |
|---|---|
| Minecraft | **1.21.11** |
| Fabric Loader | **≥ 0.19.2** |
| Fabric API | **≥ 0.141.4** |
| Java | **21** |

## 快速构建

```bash
cd new_bulk_stonecutting
export JAVA_HOME="/path/to/graalvm-jdk-21"
./gradlew build
```

产物位于 `build/libs/bulk_stonecutting-1.21.11.jar`。

---

## 移植修改说明

### 1. 构建配置变更

#### `gradle.properties` — 版本号更新

| 属性 | 1.21.4（原版） | 1.21.11（新版） | 原因 |
|---|---|---|---|
| `minecraft_version` | `1.21.4` | `1.21.11` | 目标 Minecraft 版本 |
| `yarn_mappings` | `1.21.4+build.8` | `1.21.11+build.6` | 匹配新版本的 Yarn 反混淆映射 |
| `loader_version` | `0.16.10` | `0.19.2` | Fabric Loader 最新稳定版 |
| `fabric_version` | `0.118.0+1.21.4` | `0.141.4+1.21.11` | Fabric API 最新稳定版 |
| `mod_version` | `1.21.4` | `1.21.11` | 模组版本号跟随 MC 版本 |

#### `build.gradle` — 插件版本升级

| 项目 | 旧版 | 新版 | 原因 |
|---|---|---|---|
| `fabric-loom` | `1.9.2` | **`1.14.10`** | 1.21.11 使用了新版 unpick 格式，旧版 Loom 不支持（报错 `Unsupported unpick version`） |
| Gradle | `8.12` | **`9.2.0`** | Loom 1.14.10 需要 Gradle 9.2+ 的插件 API |

---

### 2. 源代码修改

#### 2.1 `StonecutterMouseMixin.java` — 鼠标事件方法签名变更

**修改原因：** Minecraft 1.21.11 重构了鼠标输入，将原来的 `int button` + `int mods` 两个参数合并为新的 `MouseInput` 类。

```diff
- import net.minecraft.client.Mouse;
+ import net.minecraft.client.input.MouseInput;

  @Inject(method = "onMouseButton", at = @At("HEAD"), cancellable = true)
- private void onMouseButton(long window, int button, int action, int mods, CallbackInfo ci) {
+ private void onMouseButton(long window, MouseInput input, int action, CallbackInfo ci) {
      ...
-     if(button != 0) return;
+     if(input.button() != 0) return;
```

`MouseInput`（`net.minecraft.client.input.MouseInput`）提供了：
- `button()` → 获取鼠标按键编号
- `modifiers()` → 获取修饰键

#### 2.2 `StonecutterMouseMixin.java` — PlayerInventory 访问方式变更

**修改原因：** Minecraft 1.21.11 将 `PlayerInventory.main` 字段可见性从 `public` 改为 `private`，需要改用新增的 getter 方法 `getMainStacks()`。

```diff
  int invSlot = -1;
- for(int i = 0; i < player.getInventory().main.size(); i++) {
-     ItemStack stack = player.getInventory().main.get(i);
+ for(int i = 0; i < player.getInventory().getMainStacks().size(); i++) {
+     ItemStack stack = player.getInventory().getMainStacks().get(i);

  private boolean isInventoryFull(PlayerInventory inventory) {
-     for(ItemStack stack : inventory.main) {
+     for(ItemStack stack : inventory.getMainStacks()) {
```

#### 2.3 `StonecutterScreenMixin.java` — 渲染层级 API 重构

**修改原因：** Minecraft 1.21.11 将渲染管线从 `RenderLayer` + `Function<Identifier, RenderLayer>` 重构为独立的 `RenderPipeline` 类，预定义的所有渲染管线常量移至新类 `RenderPipelines`（而非直接在 `RenderPipeline` 上）。

```diff
- import net.minecraft.client.render.RenderLayer;
+ import net.minecraft.client.gl.RenderPipelines;

- context.drawGuiTexture(RenderLayer::getGuiTextured, identifier, ...);
+ context.drawGuiTexture(RenderPipelines.GUI_TEXTURED, identifier, ...);
```

> **说明：** 此修改将原基于静态工厂方法 + 方法引用的写法，改为直接引用 `RenderPipelines` 类中的 `GUI_TEXTURED` 静态常量（类型为 `RenderPipeline`）。

**`StonecutterScreenMixin.java` 额外修改：** `renderWidget()` 方法在 1.21.11 中可见性变为 `protected`，改用 `render()` 公共方法。

```diff
- massCraftCheckbox.renderWidget(context, mouseX, mouseY, delta);
+ // render() 为父类 ClickableWidget 的公共方法，内部调用 renderWidget()
+ // addDrawableChild 已自动渲染，无需手动调用，直接移除此行
```

---

### 3. 移除的文件

| 文件 | 原因 |
|---|---|
| `CheckboxWidgetResizableMixin.java` | `CheckboxWidget` 在 1.21.11 中不再重写 `renderWidget()`，该方法仅存在于父类 `ClickableWidget`。Mixin refmap 无法跨层级映射未重写的方法，导致运行时 `MixinApplyError`。移除自定义尺寸渲染，改用标准 `CheckboxWidget`。 |
| `ResizableCheckbox.java` | 上述 Mixin 所依赖的接口，一同移除。 |

---

### 4. 未修改的文件（无 API 变更）

以下文件从原版直接复制，无需修改：

| 文件 | 说明 |
|---|---|
| `Bulk_stonecutting.java` | Mod 入口（仅实现 `ModInitializer`，无逻辑） |
| `Bulk_stonecuttingClient.java` | Client 入口（仅实现 `ClientModInitializer`，无逻辑） |
| `ModConfig.java` | 存储 checkbox 勾选状态 |
| `HandledScreenAccessor.java` | 访问 `HandledScreen.focusedSlot`（API 未变） |
| `fabric.mod.json` | 使用 Gradle 变量替换，无需手动修改 |
| `settings.gradle` | 仓库配置不变 |

---

## 文件结构

```
new_bulk_stonecutting/
├── build.gradle                          # 构建脚本（Loom 1.14.10）
├── gradle.properties                     # 版本配置（1.21.11）
├── settings.gradle                       # 仓库配置
├── gradlew / gradlew.bat                 # Gradle Wrapper
├── gradle/wrapper/
│   ├── gradle-wrapper.jar
│   └── gradle-wrapper.properties         # Gradle 9.2.0
└── src/
    ├── main/
    │   ├── java/org/vee/bulk_stonecutting/
    │   │   └── Bulk_stonecutting.java
    │   └── resources/
    │       ├── fabric.mod.json
    │       └── bulk_stonecutting.mixins.json
    └── client/
        ├── java/org/vee/bulk_stonecutting/
        │   ├── client/
        │   │   ├── Bulk_stonecuttingClient.java
        │   │   └── ModConfig.java
        │   └── mixin/client/
        │       ├── HandledScreenAccessor.java
        │       ├── StonecutterMouseMixin.java
        │       └── StonecutterScreenMixin.java
        └── resources/
            └── bulk_stonecutting.client.mixins.json
```

## 原模组

- 来源：`bulk_stonecutting-master`
- 原始版本：Minecraft 1.21.4 / Fabric Loader 0.16.10 / Fabric API 0.118.0

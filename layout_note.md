# ZMK Layout 整理笔记

本文基于以下资料整理：
- https://zmk.dev/docs/development/hardware-integration/physical-layouts
- https://zmk.dev/docs/config/layout#physical-layout
- https://zmk.dev/docs/features/studio
- https://github.com/nickcoutsos/keymap-editor/wiki/Defining-keyboard-layouts
- https://zmk-physical-layout-converter.streamlit.app/
- https://nickcoutsos.github.io/keymap-layout-tools/
- https://zmk-layout-helper.netlify.app/

目标：说明 ZMK 里的 layout 相关概念、文件结构、工具转换流程，以及在 keymap-editor / ZMK Studio 场景下怎么调整 layout。

---

## 1. 先区分三个核心概念

### 1.1 Matrix Transform（逻辑按键顺序映射）
- `compatible = "zmk,matrix-transform"`
- 作用：把 keymap 的逻辑顺序映射到扫描矩阵位置（`RC(r,c)`）
- 你在 `.keymap` 里写的 `bindings` 顺序，本质上遵循 transform 的逻辑位置顺序

### 1.2 Physical Layout（物理布局定义）
- `compatible = "zmk,physical-layout"`
- 作用：把一个可选布局方案描述成一个“布局实体”
- 典型包含：
  - `display-name`
  - `transform`（指向 matrix transform）
  - `kscan`（可选，若不同布局使用不同 kscan）
  - `keys`（可选，但 **ZMK Studio 需要**）

### 1.3 Position Map（多布局之间的映射）
- `compatible = "zmk,physical-layout-position-map"`
- 作用：在多个 physical layout 间切换时，告诉系统“旧布局第 i 个键”应该对应“新布局哪个键”
- 没有 position map 时，Studio 会尝试根据物理属性自动推断；复杂布局常不够精确

---

## 2. `keys` 字段是什么，为什么重要

`keys` 是 physical layout 中每个按键的物理属性数组。每项格式：

`<&key_physical_attrs w h x y r rx ry>`

含义（ZMK 文档单位）：
- `w`/`h`：按键宽高（centi-keyunit）
- `x`/`y`：左上角坐标（centi-keyunit）
- `r`：旋转角（centi-degree，正值顺时针）
- `rx`/`ry`：旋转中心

要点：
- **ZMK Studio 支持前提之一**：physical layout 必须有 `keys`
- 同一布局下，`keys` 顺序要与 keymap/transform 的逻辑位置顺序一致
- 负数在 devicetree 里常写成 `(-3000)` 这种括号形式

---

## 3. keymap-editor 的 layout 是什么格式

keymap-editor wiki 的关键信息：
- keymap-editor 采用“QMK info.json 风格”的布局描述（但不是完整 QMK 编译格式）
- 顶层核心是 `layouts`
- 每个 layout 内关键是 `layout` 数组，元素常见字段：`row`、`col`、`x`、`y`、`w`、`h`、`r`、`rx`、`ry`
- 其中 `row/col` 在这里用于文本渲染分组，不等同于硬件 GPIO 矩阵

自动生成能力：
- 若你的 zmk-config 没有 `info.json`，可在 keymap-editor 打开自动布局解析（Automatic Layout Parsing）
- 它会从 matrix transform 尝试推断一个初版布局（可用但通常不够精细）

结论：
- keymap-editor 主要消费/输出 JSON 布局描述
- ZMK Studio 最终需要的是 devicetree physical layout（尤其 `keys`）
- 所以通常需要“JSON <-> DTS”的转换流程

---

## 4. 两个在线工具怎么分工

### 4.1 ZMK Physical Layout Converter
地址：https://zmk-physical-layout-converter.streamlit.app/

用途（从其源码可确认）：
- 在 **QMK-like JSON** 与 **ZMK physical layout DTS** 间双向转换
- 可视化预览布局
- 可从“正交参数”快速初始化布局
- 可载入 ZMK 内置共享布局做起点

常见操作路径：
1. 贴入 JSON（或先用参数生成）
2. 在可视化里检查按键位置/旋转
3. 生成 ZMK DTS 片段（含 `keys`）
4. 贴回你的 `.dtsi/.overlay` 中并接上 `transform`/`kscan`

说明：该站有时会遇到 Streamlit 侧重定向或访问限制；打不开时可改用本地运行其仓库代码，或先使用 keymap-layout-tools 编辑 JSON，再手工写入 DTS。

### 4.2 Keymap Layout Tools Helper
地址：https://nickcoutsos.github.io/keymap-layout-tools/

用途：
- 动态渲染 layout JSON
- 便于微调 `x/y/w/h/r/rx/ry`
- 支持导入来源（项目说明提到 ZMK DTS、KLE JSON、KiCad）
- 支持生成更易读的文本布局辅助结果

推荐定位：
- 把它当“布局几何编辑器/检查器”
- 再把结果送进 converter 产出 ZMK DTS

### 4.3 ZMK Layout Helper（Position Map）
地址：https://zmk-layout-helper.netlify.app/

用途：
- 辅助编写 `zmk,physical-layout-position-map`
- 尤其适用于同一键盘多个物理方案切换时保持按位映射

---

## 5. 面向你的使用场景：怎么调整 layout

下面按你提到的两类使用方式拆解。

### 5.1 场景 A：主要使用 keymap-editor

目标：让编辑器显示更接近真实键盘，并保证导出的布局可用于 ZMK Studio。

建议流程：
1. 在 keymap-editor 先打开仓库，尝试自动布局解析拿到初稿
2. 若布局误差大，用 keymap-layout-tools 调整几何（`x/y/w/h/r/rx/ry`）
3. 用 zmk-physical-layout-converter 把 JSON 转成 ZMK DTS physical layout
4. 把 DTS 放入你的布局文件（通常建议独立 `<keyboard>-layouts.dtsi`）
5. 给每个 physical layout 指定 `transform`（及需要时的 `kscan`）
6. 如果有多个 physical layout，再补 position map
7. 编译并在 Studio 里验证切换布局时的按位迁移是否正确

实操建议：
- 先保证“键数量和顺序”正确，再微调坐标
- 能复用 ZMK in-tree 共享布局时，优先 `#include`，减少维护成本

### 5.2 场景 B：主要使用 ZMK Studio

目标：在 Studio 中可切换布局、在线改键位且迁移稳定。

必须条件：
1. keyboard 定义里有 `zmk,physical-layout` 节点
2. 该节点带 `keys` 属性
3. 多布局时有合理 position map（推荐）
4. 构建时启用 Studio 支持（snippet + Kconfig）

Studio 相关注意点（官方文档关键）：
- Studio 可以“选择已预定义 physical layout”，但不能在 UI 里新建 layout
- 一旦进入 Studio 管理 keymap，后续直接改 `.keymap` 文件不会自动生效，通常要通过 Studio 的恢复/同步流程处理
- Studio 固件占用 RAM 更高，小内存 MCU 可能需要额外裁剪
- Split 键盘通常只在中枢侧启用 Studio RPC 相关构建项

建议流程：
1. 先把 physical layouts + keys 定义好
2. 再补齐 position map（特别是可拆列、不同拇指区方案）
3. 启用 Studio 构建参数并刷写
4. 在 Studio 内切换不同 layout，验证映射
5. 若映射错误，优先修 position map，而不是仅靠自动推断

---

## 6. 最小示例骨架（便于你对照）

```dts
#include <physical_layouts.dtsi>

/ {
    physical_layout0: physical_layout_0 {
        compatible = "zmk,physical-layout";
        display-name = "Default";
        transform = <&default_transform>;
        kscan = <&kscan0>;
        keys =
            <&key_physical_attrs 100 100   0   0    0   0   0>
          , <&key_physical_attrs 100 100 100   0    0   0   0>
          , <&key_physical_attrs 100 100   0 100    0   0   0>
          , <&key_physical_attrs 100 100 100 100    0   0   0>
        ;
    };

    alt_layout: physical_layout_1 {
        compatible = "zmk,physical-layout";
        display-name = "Alt";
        transform = <&alt_transform>;
        kscan = <&kscan0>;
        keys =
            <&key_physical_attrs 100 100   0   0    0   0   0>
          , <&key_physical_attrs 100 100 100   0    0   0   0>
          , <&key_physical_attrs 100 100   0 100    0   0   0>
        ;
    };

    my_position_map {
        compatible = "zmk,physical-layout-position-map";
        complete;

        default_map: default_map {
            physical-layout = <&physical_layout0>;
            positions = <0 1 2 3>;
        };

        alt_map: alt_map {
            physical-layout = <&alt_layout>;
            positions = <0 1 2 3>; // 示例：按你的真实映射调整
        };
    };
};
```

---

## 7. 常见坑与排查

1. Studio 看不到布局切换
- 检查是否定义了 `zmk,physical-layout`，以及是否启用了 Studio 构建

2. Studio 切布局后键位错位
- 先检查 `keys` 顺序是否与 bindings 逻辑顺序一致
- 再检查 position map 的 `positions` 是否一一对应

3. keymap-editor 显示和真实键盘差很多
- 自动解析只是起点，必须手工调整 `x/y/w/h/r/rx/ry`
- 使用 keymap-layout-tools 或 converter 的可视化反复校准

4. 负旋转写法报错
- 在 DTS 里通常使用括号负数，如 `(-1500)`

5. 多布局映射策略混乱
- 选“键最多”的布局做参考布局，通常更容易保证信息不丢

---

## 8. 推荐的稳定工作流（简版）

1. 先确定 transform 的逻辑顺序
2. 用 keymap-layout-tools / converter 调整 geometry
3. 产出 `keys` 并写入 physical layout
4. 多布局时写 position map
5. 本地构建 + Studio 实机切换验证
6. 最后再做细节美化（旋转中心、特殊键尺寸）

如果你愿意，我可以下一步直接按你这个仓库当前 shield（比如 shiduo3400）的现有 dtsi/overlay 结构，给出一版“可直接粘贴”的 layout 与 position-map 初稿。
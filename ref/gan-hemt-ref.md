# GaN HEMT 器件结构绘制参考

基于 p-GaN HEMT 器件的实践经验总结，涵盖 SDE 脚本编写中的坐标系、几何绘制、掺杂、网格细化等关键环节的最佳实践与 API 细节。

> 本参考配合 `@tongleen/tcad-sde` 使用。region 名一律不写 `.region` 后缀（API 会自动追加）。

---

## 1. 坐标系约定

### y_down 模式

```typescript
draw.setCoordMode("y_down")
```

- **y 向下递增**：器件表面在顶部（y 值小），衬底在底部（y 值大）
- **2DEG 设为 y=0**：以 AlGaN/GaN 异质结界面为原点，便于各层厚度计算
  - 2DEG 以上（势垒层、p-GaN、金属等）：**负值**
  - 2DEG 以下（沟道、Buffer、衬底）：**正值**

### 层叠参考坐标

```
         ↑ 负值方向（器件表面）
         |
  y=-0.115  ── 高掺杂 p-GaN 顶部
  y=-0.045  ── 常规 p-GaN 顶部
  y=-0.015  ── AlGaN 表面
  y=0       ── 2DEG（AlGaN/GaN 界面）
  y=0.300   ── GaN 沟道底部
  y=1.300   ── GaN Buffer 底部
  y=4.300   ── Al₂O₃ 衬底底部
         ↓ 正值方向（衬底方向）
```

---

## 2. 常用材料与掺杂剂

### Material 类型

```typescript
type Material = "GaN" | "AlGaN" | "Al2O3" | "AlN"
               | "Si3N4" | "PI" | "Metal" | "Contact"
```

| 材料 | 用途 |
|------|------|
| GaN | 沟道层、Buffer 层、p-GaN（与沟道同材料，不同 region） |
| AlGaN | 势垒层 |
| Al₂O₃ | 衬底（蓝宝石） |
| AlN | 插入层/成核层 |
| Si₃N₄ | 钝化层 |
| PI | 绝缘胶（层间介质） |
| Metal | 电极几何绘制 |
| Contact | 电极 remove 后 SDE 自动赋予的材料名，网格引用时使用 |

### Dopant 类型

```typescript
type Dopant = "xMoleFraction" | "pMagnesiumActiveConcentration"
```

| 掺杂剂 | 用途 |
|--------|------|
| xMoleFraction | AlGaN 中 Al 组分 |
| pMagnesiumActiveConcentration | p-GaN 中 Mg 受主掺杂 |

---

## 3. 几何绘制最佳实践

### 3.1 重叠行为

```typescript
draw.setOverlapBehavior("replace")   // 默认，新覆盖旧
draw.setOverlapBehavior("keep")      // 保留旧，新填空白
```

- **半导体层 + 金属层** → `"replace"`
- **钝化层覆盖** → 先 `"replace"` 画完所有器件结构，再切 `"keep"` 画钝化层

```typescript
// 1. 画器件结构（replace）
draw.setOverlapBehavior("replace")
draw.rectangle({ name: "GaN_Channel", material: "GaN", ... })
draw.rectangle({ name: "Source_Ohmic", material: "Metal", ... })

// 2. 画钝化层（keep，自动填充空白，不覆盖已有结构）
draw.setOverlapBehavior("keep")
draw.rectangle({ name: "Si3N4_Passivation", material: "Si3N4", ... })
draw.setOverlapBehavior("replace")   // 恢复
```

### 3.2 矩形与圆角

```typescript
draw.rectangle({
    name: "Ohmic",
    material: "Metal",
    p0: position(x0, y0),       // 左下角
    p1: position(x1, y1),       // 右上角
    round: [[position(x_corner, y_corner), radius_in_um]],  // 可选
})
```

- `round` 类型：`[Position, number][]`，每个元素 = `[顶点位置, 圆角半径(μm)]`
- 仅需要圆角的位置才加 `round`，不需要时省略

### 3.3 Region 命名

- API **自动给 region 名加 `.region` 后缀**
- **用户代码中所有地方都不加 `.region`**（包括 `dop`、`mesh` 的 `region` 参数，以及 `MaxLenInt` 的 `interface` 参数）

```typescript
// ○ 正确：draw 命名 + 后续引用
draw.rectangle({ name: "pGaN", material: "GaN", ... })
dop.constant({ region: "pGaN", ... })           // → "pGaN.region"
mesh.refine({ region: "pGaN", kind: "region", ... })

// × 错误：手动加 .region
region: "pGaN.region"
```

> 例外：`mesh.offset` 使用 `kind: "region"` 时，`targets[].target` 不会被自动追加后缀，需传入完整 region 名（含 `.region`）。建议优先使用 `kind: "material"`。

### 3.4 Unite 合并

```typescript
draw.unite({
    positions: [
        position(15, -0.20),   // region A 内（唯一一个点）
        position(15, -0.35),   // region B 内（唯一一个点）
        position(15, -0.50),   // region C 内（唯一一个点）
    ],
    name: "Merged_Name",
})
```

- **每个参与合并的 region 选且只选一个点**
- 点要选在各自 region 的**独有区域**（不在重叠边界上）
- 合并后的 region 名由 `name` 指定，API 自动加 `.region`

### 3.5 电极定义（body + remove）

```typescript
// Step 1: 画出金属体区
draw.rectangle({ name: "Source_Ohmic", material: "Metal", ... })

// Step 2: 在体区内定义接触，移除体
contact.add({
    name: "Source",
    contacts: [{ position: position(5, -0.34), shape: "body", remove: true }],
})
```

**重要**：
- 几何绘制时材料用 `"Metal"`
- Remove 后 SDE 将该电极材料标记为 **`"Contact"`**
- **网格语句中引用该材料时用 `"Contact"`**，不是 `"Metal"`
- 与电极相邻的半导体侧网格加密，电极本身不需要

---

## 4. 掺杂最佳实践

### 4.1 kind 选择

| kind | 适用场景 |
|------|----------|
| `kind: 'region'` | 整个 region 均匀掺杂或组分 |
| `kind: 'position'` | 同一 region 内局部/分级掺杂 |
| `kind: 'material'` | 同材料名所有区域 |

> 注意：`kind: "region"` 时 `decay` / `replace` 不会生效（API 不透传），需要用替换语义时请改用 `kind: "position"`。

### 4.2 分级掺杂（同一个 region 内）

```typescript
// 底层：NoReplace（默认）
dop.constant({
    name: "Base",
    dopant: "pMagnesiumActiveConcentration",
    concentration: 6e17,
    kind: "position",
    rectangle: [position(x0, y0), position(x1, y1)],
})

// 上层：Replace 覆盖
dop.constant({
    name: "Upper",
    dopant: "pMagnesiumActiveConcentration",
    concentration: 5e19,
    replace: "Replace",
    kind: "position",
    rectangle: [position(x2, y2), position(x3, y3)],
})
```

- 下层用默认 `NoReplace`，上层用 `Replace` 覆盖重叠区
- 命名必须唯一（`name` 在上下文中全局唯一）

### 4.3 摩尔组分

```typescript
dop.constant({
    name: "AlGaN_Mole",
    dopant: "xMoleFraction",
    concentration: 0.18,       // Al 组分 18%
    kind: "region",
    region: "AlGaN_Barrier",
})
```

---

## 5. 网格细化最佳实践

### 5.1 核心原则

| 原则 | 说明 |
|------|------|
| **优先用 kind 而非 position** | `kind: 'region'` 或 `kind: 'material'`；`kind: 'position'` 仅用于跨 region/material 的大范围 |
| **界面加密用 MaxLenInt** | 不要用 position rectangle 框一块区域，用 `func: [{ func: "MaxLenInt", ... }]` |
| **offset 仅用于圆角** | 几何有圆角的局部位置才用 `mesh.offset` |
| **电极移除后引用 "Contact"** | 所有 interface/target 中的 `"Metal"` 改为 `"Contact"` |
| **绝缘体粗体内 + 界面** | 体内用粗 dx/dy，仅在与其他材料的界面加 MaxLenInt 10nm |

### 5.2 决策树

```
需要加密的位置是什么？
├── 一个独立的 material       → kind: "material", material: "..."
├── 一个独立的 region         → kind: "region", region: "..."
└── 跨多个 material/region   → kind: "position"（最后选择）

加密方式？
├── 界面需要加密             → func: [{ func: "MaxLenInt", ... }]
├── 掺杂梯度需要加密         → func: [{ func: "MaxTransDiff", value }]
├── 几何圆角处               → mesh.offset
└── 大面积体区               → 仅 refine dx/dy，不加 func

加密施加在哪个材料侧？
├── 半导体侧                 → 关注的重点
├── 金属/电极侧              → 不加密（remove 后无意义）
└── 绝缘体侧                 → 仅界面附近需要
```

### 5.3 MaxLenInt 用法

```typescript
mesh.refine({
    name: "Interface_Refine",
    dx: [min, max],               // 区域内整体网格步长
    dy: [min, max],
    kind: "material",             // 或 "region"
    material: "GaN",              // 或 region: "pGaN"
    func: [{
        func: "MaxLenInt",
        interface: ["GaN", "AlGaN"],           // 界面两侧的材料或 region 名
        value: 0.001,                           // 界面处网格步长（μm）
        factor: 1.4,                            // 向外扩展因子
        double_side: true,                      // 界面两侧均加密
        use_region_names: true,                 // 同种材料不同 region 时需要
    }],
})
```

**关于 interface 参数**：
- 不同材料间：直接写材料名，如 `["GaN", "AlGaN"]`
- 同种材料（如 GaN 沟道 vs pGaN）：需要 `use_region_names: true`，interface 中用 region 名
- 与电极相邻：用 `"Contact"`（而非 `"Metal"`）

### 5.4 offset 用法（仅圆角）

```typescript
mesh.offset({
    maxlevel: 5,
    kind: "material",
    material: "GaN",           // 加密施加在 GaN 侧
    targets: [{
        target: "Contact",     // 边界伙伴材料
        value: 0.001,          // 界面步长（μm）
        factor: 1.5,           // 扩展因子
    }],
})
```

- 与 MaxLenInt 不同，offset 在有几何曲率（圆角、尖角）的位置效果更好
- `material` = 加密要施加在哪一侧的材料
- `targets` = 边界伙伴材料列表

### 5.5 各类材料的网格策略

| 材料类别 | 网格策略 |
|----------|----------|
| **有源半导体**（GaN 沟道、AlGaN） | 精细，界面 MaxLenInt 1nm |
| **p-GaN** | 精细，region 内 MaxTransDiff=1 |
| **电极（Contact）** | 不加密，加密其周围半导体界面即可 |
| **钝化/绝缘体**（Si₃N₄、PI） | 粗体内（dx~0.1, dy~0.05~0.1）+ 界面 MaxLenInt 10nm |
| **衬底**（Al₂O₃） | 粗体内（dx~0.1, dy~0.1）+ 与 GaN 界面 MaxLenInt 10nm |

### 5.6 2DEG 特定优化

```typescript
mesh.refine({
    name: "HeteroInterface",
    dx: [0.001, 0.1],
    dy: [0.001, 0.1],
    kind: "material",
    material: "GaN",
    func: [{
        func: "MaxLenInt",
        interface: ["GaN", "AlGaN"],
        value: 0.001,        // 1nm 界面步长
        factor: 1.4,
        double_side: true,
    }],
})
```

- 2DEG 在 AlGaN/GaN 界面，是器件最关键区域
- 通常用 1nm 步长 + factor=1.4
- GaN 体区（沟道+Buffer）可用粗网格（从 y=0.05 起），2DEG 由独立的 MaxLenInt 控制

---

## 6. 典型 p-GaN HEMT 结构参考

### 6.1 层叠

| 层 | 材料 | 厚度 | Y 范围（2DEG=0） |
|----|------|------|-------------------|
| 高掺杂 p-GaN | GaN (p++) | 70nm | -0.115 ~ -0.045 |
| 常规 p-GaN | GaN (p) | 30nm | -0.045 ~ -0.015 |
| AlGaN 势垒 | AlGaN (18%) | 15nm | -0.015 ~ 0 |
| 2DEG | — | — | 0 |
| 非掺杂 GaN 沟道 | GaN (i) | 300nm | 0 ~ 0.300 |
| GaN Buffer (C Trap) | GaN | 1μm | 0.300 ~ 1.300 |
| Al₂O₃ 衬底 | Al₂O₃ | 3μm | 1.300 ~ 4.300 |

### 6.2 X 方向布局

| 区域 | X 范围 | 宽度 |
|------|--------|------|
| 源极欧姆金属 | 0 ~ 10 | 10μm |
| 源极侧间隙 | 10 ~ 12 | 2μm |
| p-GaN 有源区 | 12 ~ 45 | 33μm |
| （高掺杂 p-GaN） | 13 ~ 17 | 4μm |
| 漏极侧间隙 | 45 ~ 47 | 2μm |
| 漏极欧姆金属 | 47 ~ 57 | 10μm |

### 6.3 金属互联

| 层 | Y 范围 | 说明 |
|----|--------|------|
| M1 + 场板 | -6.500 ~ -3.000 | 3.5μm 厚，距 2DEG 3μm |
| 通孔 | -3.000 ~ -0.695 | 6μm 宽（x 方向），连接 M1 与欧姆 |
| 欧姆金属 | -0.695 ~ 0.015 | 底部延伸到 2DEG 下 15nm |
| 栅极金属 | -0.315 ~ -0.115 | 2μm 长，居中 |
| 栅极场板 | -0.675 ~ -0.415 | 距 AlGaN 400nm，厚 260nm |
| PI 绝缘 | -6.500 ~ -0.415 | 与 M1 顶部对齐 |
| Si₃N₄ 钝化 | -0.415 ~ -0.015 | 400nm |

---

## 7. 常见错误

| 错误 | 后果 | 修正 |
|------|------|------|
| region 名手动加 `.region` | API 重复添加导致名称错误 | 全都不加 `.region` |
| Unite 中同一个 region 选多个点 | 浪费，可能引起歧义 | 每个 region 只选一个点 |
| 点选在 region 重叠边界上 | unite 可能找不到正确 region | 选在各自独有区域内 |
| 网格用 `"Metal"` 而不是 `"Contact"` | body+remove 后找不到材料 | 一律用 `"Contact"` |
| 用 position rectangle 框界面区域 | 网格不贴合界面，效率低 | 改用 `MaxLenInt` |
| 绝缘体加精细网格 | 网格膨胀，仿真慢 | 粗体内 + 仅界面 10nm |
| 2DEG 界面网格太粗 | 沟道电荷计算不准 | 1nm 步长 + factor=1.4 |

---

## 8. 完整脚本结构模板

```typescript
import { useSde, position } from "@tongleen/tcad-sde"

type Material = "GaN" | "AlGaN" | "Al2O3" | "Si3N4" | "PI" | "Metal" | "Contact"
type Dopant = "xMoleFraction" | "pMagnesiumActiveConcentration"

const { draw, contact, dop, mesh, save, runAndExit } = useSde<Material, Dopant>()

draw.setCoordMode("y_down")
draw.setOverlapBehavior("replace")

// ===== 几何参数 =====
// ... 定义常量

// ===== 1. 连续半导体层 =====
draw.rectangle({ name: "...", material: "GaN", ... })

// ===== 2. p-GaN / 其他特殊结构 =====
draw.rectangle({ name: "...", material: "GaN", ... })
draw.unite({ positions: [...], name: "..." })

// ===== 3. 金属（欧姆、通孔、M1、栅极） =====
draw.rectangle({ name: "...", material: "Metal", ... })
draw.rectangle({ name: "...", material: "Metal", ... })
draw.unite({ positions: [...], name: "Source_Metal" })

// ===== 4. 钝化层（keep 模式） =====
draw.setOverlapBehavior("keep")
draw.rectangle({ name: "Si3N4", material: "Si3N4", ... })
draw.rectangle({ name: "PI", material: "PI", ... })
draw.setOverlapBehavior("replace")

// ===== 5. 电极定义（body + remove） =====
contact.add({ name: "Source", contacts: [{ position: pos, shape: "body", remove: true }] })
contact.add({ name: "Drain",  contacts: [{ position: pos, shape: "body", remove: true }] })
contact.add({ name: "Gate",   contacts: [{ position: pos, shape: "body", remove: true }] })

// ===== 6. 掺杂 =====
dop.constant({ name: "AlGaN_Mole", dopant: "xMoleFraction", concentration: 0.18, kind: "region", region: "AlGaN_Barrier" })
// ... 分级掺杂

// ===== 7. 网格细化 =====
mesh.refine({ name: "Global", dx: [0.01, 0.5], dy: [0.001, 0.5], kind: "position", rectangle: [...] })

mesh.refine({ name: "pGaN_Mesh", dx: [0.001, 0.02], dy: [0.0005, 0.005], kind: "region", region: "pGaN",
    func: [{ func: "MaxTransDiff", value: 1 }] })

mesh.refine({ name: "2DEG", dx: [0.001, 0.1], dy: [0.001, 0.1], kind: "material", material: "GaN",
    func: [{ func: "MaxLenInt", interface: ["GaN", "AlGaN"], value: 0.001, factor: 1.4, double_side: true }] })

mesh.offset({ maxlevel: 5, kind: "material", material: "GaN",
    targets: [{ target: "Contact", value: 0.001, factor: 1.5 }] })

mesh.refine({ name: "Insulator", dx: [0.1, 0.5], dy: [0.05, 0.2], kind: "material", material: "Si3N4",
    func: [{ func: "MaxLenInt", interface: ["Si3N4", "AlGaN"], value: 0.01, factor: 1.4, double_side: true }] })

// ===== 8. 保存 =====
draw.saveModel("n@node@")
mesh.buildMesh("n@node@")
save("n@node@_cod.cmd")
runAndExit()
```

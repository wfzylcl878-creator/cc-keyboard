# CC-Keyboard — 6 键蓝牙小键盘

方向键 + 回车 + ESC 的紧凑蓝牙小键盘。布局（两行 + 右侧竖跨 2u 回车）：

```
┌───┐      ┌───┐
│ESC│  ↑   │ ↵ │
└───┘      │   │
┌───┬───┬───┐   │
│ ← │ ↓ │ → │   │
└───┴───┴───┤   │
            └───┘
```

主控 SuperMini nRF52840（排针插接可拆卸），ZMK 固件，热插拔轴座。

## 目录结构

```
cc-keyboard/
├── ergogen/config.yaml      # PCB 生成配置（改布局/接线就改它）
├── output/                  # 生成产物（可随时重新生成）
│   ├── pcbs/main.kicad_pcb  #   PCB（嘉立创打样直接上传）
│   ├── outlines/plate.dxf   #   定位板（1.5mm 激光切割/CNC）
│   ├── outlines/plate.svg   #   定位板 SVG（切片软件 3D 打印用，导入后厚度设 1.5mm）
│   └── outlines/board.dxf   #   外壳参考轮廓
├── zmk-config/              # ZMK 固件配置（push 到 GitHub 自动编译）
│   ├── build.yaml           #   编译矩阵（板卡 + shield）
│   ├── west.yml             #   west manifest（CI 用，须在此层级）
│   ├── cc_keyboard.keymap   #   键位（可加层）
│   └── boards/shields/cc_keyboard/   # shield（zephyr 标准目录）
│       ├── cc_keyboard.overlay       #   6 键直连扫描 + 引脚
│       └── cc_keyboard.conf
└── .github/workflows/build.yml  # 固件 CI（Actions 自动编译）
```

## 重新生成 PCB

```bash
cd cc-keyboard
npx ergogen ergogen/config.yaml -o output --svg
# makerjs 输出的 SVG 是线框风格（无填充带描边），切片软件需要实心填充、无描边：
sed -i '' 's/fill="none"/fill="#000"/; s/fill:none/fill:#000/; s/ stroke="#000"//; s/ stroke-width="0.25mm"//; s/ stroke-linecap="round"//; s/stroke:#000;//; s/stroke-width:0.25mm;//' output/outlines/*.svg
```

## 制造 / BOM（约 ¥80）

| 部件 | 规格 | 数量 |
|---|---|---|
| PCB | `output/pcbs/main_clean.kicad_pcb` 导入嘉立创 EDA 布线后下单（见下） | 1 |
| 定位板 | `output/outlines/plate.dxf`，1.5mm | 1 |
| SuperMini nRF52840 | 带 UF2 bootloader | 1 |
| 排针/排母 | 2.54mm，1×9 | 各 1 |
| MX 热插拔座 | 已在 PCB footprint 内 | 6（随 PCB） |
| 轴体 | 任意 MX | 6 |
| 键帽 | 1u×5 + 竖 2u 回车键帽 | 6 |
| 电池 | PH2.0 插头，401230 (150mAh) | 1 |
| 电源开关 | MSK-12C02 拨动（或不装、锡桥短接） | 1 |

## PCB 布线与下单

**Ergogen 只做布局、不布线**——`main.kicad_pcb` 里没有任何铜走线（EDA 里看到的连线全是飞线），直接打样回来不能工作。流程：

1. 嘉立创 EDA（pro.lceda.cn）→ 文件 → 导入 → KiCad PCB → 选 `output/pcbs/main_clean.kicad_pcb`（数字规整版，直传下单页可能解析失败）
2. 布线（全部才 13 个连接，一把过）：
   - 信号线宽 0.25mm，电源线（`BAT_*`）0.5mm
   - GND 用整板敷铜（放置 → 整块敷铜 → 网络 GND），省 6 条线
   - 懒人直接用 EDA 的自动布线，完成后肉眼检查
3. 设计规则检查（DRC）通过后一键下单

## 接线表（PCB 排针 → SuperMini）

| 针 | 网络 | SuperMini 引脚 |
|---|---|---|
| 1 | ESC | P0.06 |
| 2 | ↑ | P0.08 |
| 3 | ← | P0.17 |
| 4 | ↓ | P0.20 |
| 5 | → | P0.22 |
| 6 | ⏎ | P0.24 |
| 7 | GND | GND |
| 8 | B+ | B+（电池正，经开关） |
| 9 | B- | B-（电池负） |

电池接 PCB 的 JST PH 座（免焊）；BAT+ 串电源开关后进 SuperMini。

## 固件（ZMK）

1. push 本仓库到 GitHub（`.github/workflows/build.yml` 已就位，push 即自动编译）
2. Actions 自动编译 → 构建产物页下载 `firmware.uf2`
3. SuperMini 双击 RST → 弹出 USB 磁盘 → 拖入 uf2 → 完成
4. 系统蓝牙配对 "CC-Keyboard" 即可使用

改键位：编辑 `zmk-config/cc_keyboard.keymap`（bindings 顺序对应排针 1-6），push 后重新下载 uf2。

## 焊接顺序

1. 热插拔轴座（贴片，先焊一个脚对位再焊其余）
2. 排母 ×9、JST 座、开关焊盘
3. PCB 上所有焊接完成后插 SuperMini 通电测试
4. 验证：插 USB → CHG 灯亮；按键在 ZMK 下生效（`zmk-kconfig test` 或直接连手机蓝牙试）

## 已知取舍

- 回车是**热插拔轴**竖跨 2u，键帽需 1×2 竖键帽；无卫星轴孔（2u 竖键无卫星轴可接受，边缘按压略晃，介意可后续 rev2 加 stab 孔）
- `where` 过滤器使用 `/regex/` 斜杠格式（Ergogen 的字符串 where 是全等匹配，不是正则）
- Ergogen 稀疏布局：列内 `rows` 不能删行，须对要删的键写 `skip: true`；且行按**声明顺序**自 y=0 向上堆叠，全局 `rows` 必须把 `bottom` 写在 `top` 前，否则行名与实际位置颠倒

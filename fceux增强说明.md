# FCEUX-2.6.6-Yhc 增强说明（Mapper 195 / 外星科技 FS303 完整支持）

> **本文档是给 AI 助手看的自包含操作手册。**
> 使用场景：上游模拟器源码（Yhc-Studio/FCEUX-2.6.6-Yhc，或其它 FCEUX 版本）
> 更新后，需要在不了解历史背景的情况下重新实现 Mapper195 增强。本文不依赖
> 任何历史对话——所有需要的知识（完整最终代码、fceux 框架知识、原理、验证
> 方法、调试手段）都在本文内。
>
> **版本约定**：本文有两个副本，**以 TSZY2 工程根目录这份为主本**，
> `fceux-2.6.6-yhc/fceux增强说明.md` 是随仓库提交的同步副本；修改后两边要
> 同步（改根目录这份 → 复制到仓库 → 提交推送）。
>
> **上游合并后的核对清单（TL;DR）**，共 2 个文件 4 处改动：
> | # | 文件 | 改动 | 一句话 |
> |---|------|------|--------|
> | A | `src/ines.cpp` | `not_power2[]` 名单含 195 | 否则 PRG 按填充尺寸读文件，CHR 数据被吞（画面色块化） |
> | B | `src/boards/mmc3.cpp` `Mapper195_Init` | PRG 大小从 `totalFileSize - CHR` 推导 + `PRGmask8[0] = 0xFF` | `PRGRomSize` 是填充值不可信；非 2 幂 bank 数不能靠位掩码 |
> | C | `src/boards/mmc3.cpp` `M195PW` | bank 号翻译：`0xFE→N-2`、`0xFF→N-1`、其余 `% N` | 让非 2 幂 bank 数正确映射（N = 实际 8KB bank 数） |
> | D | `src/boards/mmc3.cpp` `M195CW`/`M195Power` | CHR 值 ≤3 → 4KB CHR RAM；$5000-$5FFF 独立 4KB XRAM；header 镜像；上电清零 | FS303 板载 RAM 特性 |
>
> **最省事的做法**：把 `src/boards/mmc3.cpp` 里 `// ----- Mapper 195 -----` 到
> `// ----- Mapper 196 -----` 之间的段落**整段替换**为本文第 6 节的完整代码，
> 再做第 4 节的 ines.cpp 一行改动。若上游该区域代码结构变了，按第 3 节框架
> 知识适配语义。

---

## 1. 目标与验证基准

### 1.1 目标 ROM（都在 TSZY2 工程根目录）

| ROM | PRG | CHR | 8KB bank 数 | 说明 |
|---|---|---|---|---|
| `origin.nes` | 512KB | 256KB | 64（2 的幂） | 中文原版，上游原版"碰巧"能跑 |
| `game.nes` | 1280KB | 256KB | **160（非 2 幂）** | hack 改版"传说之翼" |
| `game2.nes` | 1536KB | 256KB | **192（非 2 幂）** | hack 改版"纵横天下" |

三份都是 iNES 1.0：**Mapper 195**（flag6=0x30 flag7=0xC0，mapper=0xC3），
无 trainer，水平镜像（flag6 bit0=0），板载 4KB CHR RAM + $5000-$5FFF 独立
4KB PRG-RAM。它们是《天使之翼2》汉化版及其扩容 hack，ROM 内含 12KB 补丁层
（复位时从 bank $42/$43 拷到 CPU $5000-$7FFF 执行）。

### 1.2 修好后的预期画面（验证基准）

- **origin.nes**：开场为比赛场景+对话框"虎看到食物一般球被断下了！"，
  全彩（蓝天/绿坡/人物/头像框），细节完整。
- **game.nes**：上电直接进开场剪影场景（黑底），下方两行中文字幕
  "期待世界冠军那一天来临的大空翼"（等待按键推进，画面静止是正常的）。
  **合格标准**：画面上部有白云（两朵、有形状）、看台横条纹、球场灰色横纹、
  人物有头发/衣服/足球细节。注意该场景本身是灰度配色，不是全彩。
- **game2.nes**：同 game.nes（banks=192）。
- 游戏中途换场景/比赛画面也应图形完整；fceux 自身存读档（F5/F7）来回自洽。

### 1.3 故障症状 → 根因对照（先看症状再查）

| 症状 | 根因 | 查哪 |
|---|---|---|
| 完全黑屏，游戏不动 | 修改点 B/C 缺失：bank 错位，CPU 卡死在垃圾代码 | 本文 §5、§6 |
| 能进游戏、文字正常，但图形是大色块剪影（无云/看台/细节） | 修改点 A 缺失：VROM 缓冲没读到数据（保持 0xFF 填充） | 本文 §4 |
| 图形错位/花屏但不是空白 | CHR 窗口逻辑或镜像问题 | 本文 §7 |
| 画面正常但音乐/滚动效果错乱 | IRQ 问题（不应发生——用 fceux 标准 MMC3 IRQ 即可） | 本文 §8 |

---

## 2. 硬件背景：FS303 / Mapper 195

外星科技（Waixing FS303）的 MMC3 克隆板，与标准 MMC3 的差异只有三条：

1. **PRG 超出 2 的幂**：160/192 个 8KB bank（fceux 的 bank 映射是
   `V & PRGmask8[]` 位与，只能表达 2 的幂）。
2. **板载 4KB CHR RAM**：CHR 寄存器值 ≤3 的槽映射到 CHR RAM，其余映射
   CHR ROM（对应 VirtuaNESex 的 `SetBank_PPUSUB`）。中文 hack 的字库 tile
   由游戏运行时写进这块 RAM。
3. **CPU $5000-$5FFF 独立 4KB PRG-RAM**（VirtuaNES 叫 XRAM）：必须与
   $6000-$7FFF 的 WRAM、CHR RAM 互不重叠。

**IRQ 与 MMC3 寄存器完全兼容**（$C000 latch/$C001 reload/$E000 disable/
$E001 enable），fceux GenMMC3 自带的 `GameHBIRQHook = MMC3_hb`（每扫描线
一个时钟）实测每帧时钟约 241 次，工作正常——**不要自定义 IRQ**。

**参考实现**（行为有分歧时以它们为准，都实测能跑本组 ROM）：

- `VirtuaNESex_src_191105/NES/Mapper/Mapper195.cpp`（VirtuaNESex 源码目录；
  注意其 MMU：`SetVROM_1K_Bank` 用 `% VROM_1K_SIZE` 取模、`SetCRAM_1K_Bank`
  用 `& 0x1F` 共 32KB CRAM——Mapper195 只用到 CRAM 槽 0-3）
- `web/emu/mappers/mapper195.js`（TSZY2 工程 web 模拟器内核，`% prgCount`
  取模模型；注意其 `chrBankSelect`/`updateChrBanks` 的 $8000 写检测是死代码
  且 CHR RAM 上电预载了 ROM 数据——这两点都不重要：CHR RAM 上电全 0 即可，
  VirtuaNES 也是全 0）

NESdev Wiki 描述的 FS303 "GAL 动态查表"（$80/$82/$88/$C0/$CA 等值决定
CHR RAM 窗口）**不需要实现**——"≤3 → RAM"这条简单规则已被两个参照
模拟器验证足够。

---

## 3. fceux MMC3 框架速查（改代码前先懂这些）

都在 `src/boards/mmc3.cpp`：

- **`GenMMC3_Init(CartInfo*, prg, chr, wram, battery)`**：通用初始化。参数
  prg/chr 只是 KB 数值（512/256 是 MMC3 惯例），内部 `PRGmask8[0] &=
  (prg>>13)-1`——当 prg=512 时 `512>>13=0`，`-1` 下溢成 0xFFFFFFFF，等于
  没改（掩码保持 ines.cpp 设的 uppow2 值 255），这就是为什么要在 Init 里
  自己重新赋值掩码。
- **`pwrap(A, V)` / `cwrap(A, V)`**：PRG/CHR 寄存器写入的 board 钩子。游戏写
  $8001 时按 MMC3 命令分发：R6→`pwrap(0x8000 或 0xC000, V)`、R7→
  `pwrap(0xA000, V)`、R0-R5→`cwrap(addr, V)`（addr 已含 4KB 模式 cbase
  交换）。最终各自调 `setprg8(A, V)` / `setchr1r(chip, A, V)` 落到实际映射。
- **`FixMMC3PRG(cmd)`**：MMC3 模式位（bit6）变化时重放所有 PRG 槽。**固定
  库用特殊值传入**：`pwrap(0xC000, ~1)` / `pwrap(0xE000, ~0)`（即
  0xFE/0xFF）。这就是 M195PW 里 `V==0xFE/0xFF` 特判的来源——真实游戏只写
  小 bank 号（game.nes 实测只写 R6=0x42、R7=0x43），值特判无歧义。
- **`setchr1r(chip, A, V)`**：映射 1KB CHR 槽。chip 0 = CHR ROM 缓冲、
  chip 0x10 = `SetupCartCHRMapping(0x10, ...)` 注册的 CHR RAM；内部对 V 做
  `& CHRmask1[chip]` 并维护 PPU 写入路由（PPUCHRRAM 位）。
- **`setprg4r(chip, A, bank)`**：把 chip 对应缓冲映射到 CPU 地址（4KB 粒度），
  配 `SetReadHandler/SetWriteHandler(a1, a2, CartBR/CartBW)` 让 CPU 读写走
  该映射。注意 `GenMMC3Power` 会在 $5000-$5FFF 装 KT008HackWrite，自定义
  handler 必须在 `GenMMC3Power()` 之后设置才能覆盖它。
- **`GameHBIRQHook = MMC3_hb`**：GenMMC3_Init 默认设置（每扫描线调用
  `ClockMMC3Counter`）。ppu.cpp 里 `(PPU[0]&0x38)!=0x18` 只是选择两种时序
  路径，**两条路径都会调用**，无需干预。
- **`info->totalFileSize`**：iNES 文件大小 - 16 字节头（= PRG + CHR + 杂项）。
- **`info->PRGRomSize`**：**不可信**——iNES 1.0 时是 uppow2 填充值，且
  ines.cpp 的 CRC 段（约 1362 行）会用填充后的 ROM_size 无条件再覆盖一次。
  本 mapper 改用 `totalFileSize - CHR` 推导。
- **`VROM_size`**：全局 int，8KB 单位（game.nes = 32）。
- **状态保存**：`AddExState(指针, 大小, 0, "TAG4字符")` 注册进存档；MMC3
  寄存器/IRQ 状态由 GenMMC3 自带的 `MMC3_StateRegs` 负责，无需额外注册。
- **镜像**：`GenMMC3Power` 默认 `setmirror(1)`（垂直）；按 header 纠正用
  `info->mirror`（0=水平，MI_H 宏），参考 `Mapper4_Init/M4Power` 的做法。
- **电池存档时序**：`FCEU_LoadGameSave`（.sav 填回 WRAM）在 iNESLoad 内部、
  `info->Power()` **之前**执行。Power 里的清零/初始化必须避开 battery 已注册
  （`mmc3opts & 2`）时的 WRAM，否则会把载入的存档清掉。

---

## 4. 修改点 A：ines.cpp 的 not_power2 名单（最隐蔽、最易回归）

`src/ines.cpp` 约 532 行：

```c
static int not_power2[] =
{
	53, 195, 198, 228, 547
};
```

**确认 195 在名单里**（上游原版没有，本 fork 加的）。

机制：195 在名单 → `round=false` → PRG 的 fread 按**实际 PRG 大小**
（`not_round_size = head.ROM_size << 14`）读文件，读完后文件指针正好停在
CHR 段开头，VROM 的 fread 才能读到 CHR 数据。

缺失时的失败链：`round=true` → PRG 按 `rom_size_bytes`（uppow2 填充后的
2MB）读 → game.nes 文件只有 1.5MB，这条 fread 把**整个 CHR 段吞进 PRG
缓冲** → VROM 的 fread 从文件尾读到 0 字节 → VROM 缓冲保持 malloc 后的
0xFF 填充。

**现象**：游戏逻辑完全正常（PRG 前 1.28MB 数据完整，能进开场、字幕正常——
字幕 tile 在 CHR RAM），但**所有 CHR ROM 图形是空白 tile**，画面只剩大色块。
实锤手段：dump `CHRptr[0] + 页偏移` 的内存与 ROM 文件 CHR 段对比（见 §10）。

> ⚠️ **回归警告**：这是上游合并时**最容易丢的一条**（只是一行名单改动），
> 且症状隐蔽（游戏能跑、只是图形空）。历史上就是因为在一次源码回退中丢了
> 这行，导致"黑屏修好了但画面花"又排查了一轮。

---

## 5. 修改点 B/C：PRG 大小推导与 bank 号翻译（黑屏根因）

### 5.1 机制（为什么上游会黑屏）

fceux 对每个 bank 号做 `V & PRGmask8[0]` 位与。非 2 幂的 bank 数（160）掩码
为 159 (0x9F)，游戏写 `R6=0x42`（想切 bank 0x42）时静默落到 `0x42 & 0x9F =
0x02`——**12KB 补丁层从错误 bank 拷贝，CPU 跑进未初始化 RAM 死循环（黑屏）**。

固定库的 `0xFE/0xFF & 0x9F = 0x9E/0x9F` 碰巧正确（bank 158/159），所以复位
向量是对的、初始化链能跑起来——极具迷惑性，让人以为是别的问题。

### 5.2 解法

掩码放开为 0xFF（位与不再起作用），在 pwrap 钩子里做值翻译（与 web 版
`% prgCount` 模型一致）；固定库用值特判（FixMMC3PRG 传 0xFE/0xFF）：

```c
static void M195PW(uint32 A, uint8 V) {
	if (V == 0xFE)
		V = M195_prgbanks - 2;	/* FixMMC3PRG: 固定 $C000 库 */
	else if (V == 0xFF)
		V = M195_prgbanks - 1;	/* FixMMC3PRG: 固定 $E000 库 */
	else
		V %= M195_prgbanks;	/* R6/R7 按真实 bank 数取模 */
	setprg8(A, V);
}
```

`M195_prgbanks` 在 Init 里推导（PRGRomSize 不可信，见 §3）：

```c
int prgbytes = (int)(info->totalFileSize - (uint32)VROM_size * 8192);
if (prgbytes < 512 * 1024 || prgbytes > 4096 * 1024 || (prgbytes & 0x3FFF))
	prgbytes = 512 * 1024;	/* 不合理时回退标准 FS303（64 bank） */
M195_prgbanks = prgbytes >> 13;	/* game.nes=160, game2.nes=192, origin=64 */
```

**现象**：缺失时黑屏死循环。诊断手段：MapIRQHook 里记 256 字节块首访 PC，
看执行流是否卡在 $5000-$7FFF（补丁层区域）的固定地址反复自读（历史案例：
卡在 $54BD）。

---

## 6. 修改点 D 与完整最终代码（整段替换用）

把 `src/boards/mmc3.cpp` 中 `// ----- Mapper 195 -----` 到
`// ----- Mapper 196 -----` 之间**整段替换**为以下代码（这就是本仓库当前的
最终实现，直接复制可用；若上游 API 变化按 §3 适配）：

```c
// ---------------------------- Mapper 195 -------------------------------
// Waixing FS303 / Alien Technology (外星科技): MMC3 clone used by the
// Captain Tsubasa 2 Chinese versions and their expanded hacks. Verified
// against VirtuaNESex's Mapper195 and the working web/emu mapper195.js.
//
// 1. iNES loading: 195 MUST stay in ines.cpp's not_power2 list. Without
//    it fceux rounds the PRG read up to the padded power-of-2 size
//    (2MB for game.nes), so the fread swallows the whole CHR section
//    into the PRG buffer and the VROM buffer stays 0xFF-filled: the game
//    logic runs fine (PRG data is intact) but every CHR ROM tile renders
//    blank. Non-power-of-2 sizes also break fceux's AND-mask bank
//    mapping: game.nes has 160 (game2.nes 192) 8KB banks, and with the
//    leftover mask 159 (0x9F) a write of R6=0x42 silently landed on
//    0x42 & 0x9F = 0x02, so the hacks' 12KB RAM patch layer
//    ($5000-$7FFF) copied from wrong banks and the game died inside
//    uninitialized RAM. We lift PRGmask8 to 0xFF and translate bank
//    numbers in the pwrap hook instead (value % bankCount, like the web
//    port; FixMMC3PRG passes ~1/~0 for the fixed $C000/$E000 banks,
//    which become bankCount-2/-1).
// 2. CHR: register values <= 3 map into a 4KB on-board CHR RAM window
//    (VirtuaNESex SetBank_PPUSUB); everything else is CHR ROM. The hacks
//    draw their Chinese font tiles through this RAM.
// 3. CPU $5000-$5FFF is an independent 4KB PRG-RAM (VirtuaNES XRAM): the
//    hacks copy their patch layer to $5000-$7FFF, so it must not alias
//    WRAM or CHR RAM.
// The scanline IRQ is register-compatible with MMC3; the stock fceux
// GenMMC3 IRQ (GameHBIRQHook = MMC3_hb, one clock per scanline) works,
// so no custom IRQ code here.
static uint8 *M195_XRAM = NULL;		/* 4KB PRG-RAM at $5000-$5FFF */
static uint32 M195_prgbanks = 64;	/* 8KB PRG bank count (any value, not just powers of 2) */
static int M195_mirror = MI_H;		/* header mirroring (GenMMC3Power defaults to vertical) */

static void M195CW(uint32 A, uint8 V) {
	if (V <= 3)
		setchr1r(0x10, A, V);	/* on-board CHR RAM */
	else
		setchr1r(0, A, V);	/* CHR ROM */
}

static void M195PW(uint32 A, uint8 V) {
	if (V == 0xFE)
		V = M195_prgbanks - 2;	/* FixMMC3PRG: fixed $C000 bank */
	else if (V == 0xFF)
		V = M195_prgbanks - 1;	/* FixMMC3PRG: fixed $E000 bank */
	else
		V %= M195_prgbanks;	/* R6/R7 wrap at the real bank count */
	setprg8(A, V);
}

static void M195Power(void) {
	GenMMC3Power();
	setmirror(M195_mirror);
	memset(CHRRAM, 0, CHRRAMSIZE);	/* VirtuaNES zeroes CRAM/XRAM at boot */
	if (!(mmc3opts & 2))		/* don't clobber a battery save already loaded into WRAM */
		memset(WRAM, 0, WRAMSIZE);
	memset(M195_XRAM, 0, 0x1000);
	setprg4r(0x12, 0x5000, 0);
	SetWriteHandler(0x5000, 0x5FFF, CartBW);
	SetReadHandler(0x5000, 0x5FFF, CartBR);
}

static void M195Close(void) {
	if (M195_XRAM) {
		FCEU_gfree(M195_XRAM);
		M195_XRAM = NULL;
	}
	GenMMC3Close();
}

void Mapper195_Init(CartInfo *info) {
	/* CartInfo.PRGRomSize is the power-of-2 padded size for iNES 1.0
	   (ines.cpp overwrites it from the padded ROM_size after board init
	   would run), so derive the real PRG size from totalFileSize (= file
	   size minus the 16-byte header) minus CHR. A trainer breaks the
	   16KB alignment check and falls back below. */
	int prgbytes = (int)(info->totalFileSize - (uint32)VROM_size * 8192);
	if (prgbytes < 512 * 1024 || prgbytes > 4096 * 1024 || (prgbytes & 0x3FFF))
		prgbytes = 512 * 1024;	/* implausible: standard FS303 fallback */
	M195_prgbanks = prgbytes >> 13;
	M195_mirror = info->mirror;

	GenMMC3_Init(info, 512, 256, 16, info->battery);
	/* GenMMC3_Init's mask math cannot express non-power-of-2 sizes; lift
	   the 8KB mask so the modulo bank numbers from M195PW survive
	   setprg8()'s AND. */
	PRGmask8[0] = 0xFF;
	pwrap = M195PW;
	cwrap = M195CW;
	info->Power = M195Power;
	info->Close = M195Close;

	CHRRAMSIZE = 4096;
	CHRRAM = (uint8*)FCEU_gmalloc(CHRRAMSIZE);
	SetupCartCHRMapping(0x10, CHRRAM, CHRRAMSIZE, 1);
	AddExState(CHRRAM, CHRRAMSIZE, 0, "CHRR");

	M195_XRAM = (uint8*)FCEU_gmalloc(0x1000);
	memset(M195_XRAM, 0, 0x1000);
	SetupCartPRGMapping(0x12, M195_XRAM, 0x1000, 1);
	AddExState(M195_XRAM, 0x1000, 0, "M5KX");
}
```

要点复核（自查用）：

- `M195CW`：≤3 → CHR RAM（chip 0x10），否则 CHR ROM（chip 0）。CHR ROM
  256KB 是 2 的幂，fceux 的 CHRmask 机制天然正确，**不需要改 CHRmask**。
- `M195Power`：按 header 设镜像（`M195_mirror = info->mirror` 在 Init 存；
  `GenMMC3Power` 默认垂直必须覆盖）；CHRRAM/WRAM/XRAM 上电清零（VirtuaNES
  行为；CHR RAM 上电内容无关紧要，游戏会重写字库）；$5000-$5FFF 挂 XRAM。
  注意 `SetWriteHandler(0x5000,...)` 必须在 `GenMMC3Power()` 之后设置才能
  覆盖它装的 KT008HackWrite。
- `GenMMC3_Init(info, 512, 256, 16, info->battery)`：wram=16KB（游戏用
  $6000-$7FFF 8KB WRAM）、battery 参数只影响 WRAM 是否进存档。
- **IRQ 不写任何自定义代码**：GenMMC3_Init 默认 `GameHBIRQHook = MMC3_hb`。

---

## 7. 其它注意点

- **镜像**：三份 ROM header 是水平（flag6 bit0=0）。若发现画面布局错乱，
  检查 `setmirror(M195_mirror)` 是否被上游改动破坏。
- **CHR RAM 大小固定 4KB**：不要按 wiki 的"有效 8KB"说法扩到 8KB——两个
  参照模拟器都是 4KB（槽 0-3）且运行正确。
- **存档状态**：CHRRAM（"CHRR"）/XRAM（"M5KX"）已 AddExState；MMC3 寄存器
  与 IRQ 状态由 GenMMC3 自带的 `MMC3_StateRegs` 负责。

---

## 8. 编译与部署

- **CI（推荐）**：push 到 main 分支自动触发 `.github/workflows/windows.yml`
  （windows-2022 + MSBuild v143，工程 `vc/vc14_fceux.vcxproj` Release x64），
  产物 artifact 名 `fceux-win64`，含 `output64/fceux64.exe`、
  `src/drivers/win/lua/x64/lua5.1.dll`、`src/auxlib.lua`。
- **下载 artifact**（无 gh CLI 时）：`git credential fill` 拿 token →
  `GET /repos/qqzhal/FCEUX-2.6.6-Yhc/actions/runs?per_page=1` 查状态 →
  `GET .../runs/{id}/artifacts` 拿 artifact id → `GET
  .../artifacts/{id}/zip` 下载（curl 带 `Authorization: token` 头）。
- **运行**：exe 需要和 `lua5.1.dll`、`palettes/` 等资源同目录；最简单是
  解压 artifact 后从任一旧版 fceux 安装目录拷贝其余资源文件。TSZY2 工程
  的 `fceux-2.6.6-yhc/testrun/`（不进 git）是现成的测试环境样例，
  `fceux-2.6.6-yhc/build-output/fceux-win64.zip` 是最近一次成功构建的产物。

---

## 9. 验证与调试手册

### 9.1 验证流程

1. 打开 `origin.nes` → 对话框画面全彩完整（§1.2）。
2. 打开 `game.nes` → 开场剪影场景，云/看台/球场/人物细节齐全。
3. 打开 `game2.nes` → 同上。
4. （可选）存档/读档来回几次，图形不丢。

### 9.2 症状→根因的精确判据

见 §1.3。补充：

- **怀疑修改点 A 丢了**（图形空白但游戏能跑）：在 Init 里打印
  `info->PRGRomSize` 和 `info->totalFileSize`——若 PRGRomSize 是 2097152
  （uppow2 的 2MB），而 totalFileSize 是 1572864（game.nes）或
  1835008（game2.nes），则名单没生效。
- **怀疑修改点 B/C 丢了**（黑屏）：PRG 寄存器写日志里看 `R6=0x42` 被翻译
  成多少——正确 66，错误 2。也可以在 Power 后检查
  `ROM[prgbytes-6..prgbytes-1]`，game.nes/game2.nes 应为
  `00 C5 60 FA 06 C5`（NMI=$C500、RST=$FA60、IRQ=$C506）。若检查最后
  8 字节，则为 `00 00 00 C5 60 FA 06 C5`。

### 9.3 诊断手段（本次调试中验证有效，可复刻）

- **寄存器写日志**：M195PW/M195CW 里记录 `(A, V, X.PC, scanline)`，前 300
  条全记、之后按 1/50 采样（否则日志爆炸）。文件名用相对路径（写到 exe 的
  工作目录）。
- **执行流跟踪**：`MapIRQHook` 里维护 256 字节块首访位图，输出 `K PC=xxxx`
  行——判断 CPU 卡死位置的利器（黑屏时看最后一个块和之后反复出现的地址）。
- **帧级状态 dump**：每帧（`scanline==0 && lastSl>0` 检测新帧）输出 CHR 槽
  表 + PPU[0]/PPU[1] + CHR RAM 校验和——判断"图形数据有没有被游戏写入"。
- **CHR 内存 dump**：`fwrite(CHRptr[0] + 页*1024)` 4KB 与 ROM 文件 CHR 段
  对比（实锤 not_power2 问题）；CHRRAM 单独 dump 渲染成 PNG（字库确认）。
- **截图验证**：fceux 用 OpenGL 渲染，PrintWindow 抓不到帧缓冲（抓到的黑图
  不一定是真黑屏）；要用 `SetWindowPos(HWND_TOPMOST)` 置顶后全屏
  `CopyFromScreen`。

### 9.4 已踩过的坑（别再踩）

1. **诊断日志被 taskkill /F 丢掉**：fopen 的 stdio 缓冲在强杀时不刷盘。
   要么每条日志后 fflush，要么优雅关闭（CloseMainWindow 发 WM_CLOSE，CRT
   exit 会 flush）后再取日志。
2. **不要相信 PRGRomSize**（§3），两轮根因都跟它有关。
3. **不要加自定义 IRQ**：历史上几轮"VirtuaNES HSync 移植"全是被黑屏根因
   误导的无用功；标准 MMC3_hb 工作正常（实测每帧时钟 241 次）。
4. **PrintWindow 截图对 OpenGL 窗口无效**（黑图不一定是真黑屏，先置顶+
   屏幕截取确认）。
5. **怀疑编译产物不对时**：用 python 在 exe 二进制里 `find` 特征字符串
   （如日志文件名、注释常量）确认版本；CI 是全新 checkout，无增量缓存问题。
6. **GAL 查表模型（wiki 版）不需要**：按 §2 的简单规则实现即可，历史上
   曾朝这个方向走弯路。
7. **Power 里的 memset 会清掉已载入的电池存档**：存档在 Power 之前填进
   WRAM（§3 电池存档时序），清 WRAM 前必须用 `!(mmc3opts & 2)` 过滤——
   本组 ROM 无电池没暴露，PR 审查时被发现补上的。

---

## 10. 本仓库相对上游的完整差异（合并前核对）

```
.gitignore                    +1   （忽略本地 testrun 测试目录）
.github/workflows/windows.yml +28  （CI：msbuild fceux64 Release x64 + artifact）
src/boards/mmc3.cpp           ~130 （Mapper195 重写，即本文第 6 节）
src/ines.cpp                  +1   （not_power2 名单加 195）
```

关键提交索引（`git log --oneline` 可查）：

- `Mapper195: modulo PRG bank mapping instead of bitmask (root cause fix)`
  —— 黑屏根因修复
- `Keep 195 in ines not_power2 list; drop Mapper195 diagnostics`
  —— 图形根因修复 + 诊断清理
- `Add Mapper195 enhancement guide for future upstream merges`
  —— 本文档

## 11. 上游更新后的再增强流程（操作步骤）

1. 拉上游新代码，`git diff` 对照本仓库：关注 `src/ines.cpp`（名单行）和
   `src/boards/mmc3.cpp`（Mapper195 段落）。
2. 若上游 Mapper195 段落未大改：整段替换为第 6 节代码 + 补名单行。
3. 若上游结构变化：按 §3 框架知识把 4 处语义（名单 / 掩码放开 / 取模翻译 /
   ≤3→CHR RAM + XRAM + header 镜像）重新落在新代码上。
4. push → CI 编译 → 下载 artifact → 按 §9.1 验证三份 ROM。
5. 不通过时按 §1.3 症状鉴别 → §9.2 精确判据 → §9.3 手段定位。
6. 验证通过后：更新根目录本文档（如有新知识点），同步仓库副本，提交推送。

---

## 12. NES.emu（Android 端）移植记录（2026-09 完成）

仓库 `qqzhal/emu-ex-plus-alpha`（fork 自 Rakashazi/emu-ex-plus-alpha），
分支 `mapper195-fs303-1584`。NES.emu 内嵌一份裁剪版 fceux
（`NES.emu/src/fceu/`），上述 4 处修改的语义直接适用，最终提交为
"NES.emu 1.5.84: add FS303 mapper 195 enhancements"。**改动前先通读本节
与 §3，NES.emu 与 PC 版有大量结构差异，不要假设 API 一致。**

### 12.1 平台差异适配清单（deepseek 首次移植遗漏/需注意的）

- `CartInfo` 没有 `PRGRomSize` 字段 → 用自加的 `totalFileSize`
  （uint64，"文件大小 - 16 字节头"，在 `iNESLoad` 里 `filesize -= 16`
  之后赋值）+ `VROM_size` 推导 PRG 大小，即 §6 的 Init 写法可直接用。
- `CartInfo::Power/Reset/Close` 是**无参**函数指针
  （PC 版是 `void(*)(CartInfo*)`）。
- PC 版 ines.cpp 的"CRC 段覆盖 PRGRomSize"在 NES.emu 不存在，无需担心。
- `not_power2` 名单、`GenMMC3_*`、`setprg8r/setchr1r/PPUCHRRAM/VPageR`
  机制与 PC 版语义一致（`VPageR` 就是 `VPage[8]` 的别名）。
- 顺手修复：上游 `Mapper198_Init` 复用 `M195Power`（会映射 4KB XRAM 到
  $5000），已拆出独立的 `M198Power`。
- CI：`.github/workflows/build.yml` 增加 `workflow_dispatch`；产物为
  `NesEmu-<commit前缀>.apk`（包名 `com.explusalpha.NesEmu`，artifact 名
  `NES.emu`）。

### 12.2 核心验证：headless 测试（NES.emu/headless-test/）

`bash NES.emu/headless-test/build.sh [真实ROM...]`，CI 自动跑
（`.github/workflows/headless-test.yml`）。它把 fceu 核心（ines/cart/
file/x6502/ppu/全部 boards）与一个 stub 前端链接成 Linux 程序：

- 合成 ROM（与 game.nes 同 header：160/192/64 bank + 256KB CHR）验证
  §9.2 全部判据 + 真实 CPU 跑 240 帧的全链路（XRAM 写、bank $42 读回、
  CHR RAM 字库、开显示、帧缓冲有像素）。
- 附加命令行参数视为真实 ROM：跑 600 帧输出 jam/XBuf/CHR RAM 统计并
  dump PGM 帧。**该模式仅限本地运行**——游戏 ROM 严禁放进仓库
  （git/release/artifact 都不许；曾临时传过 release 又撤下）。
- ppu.cpp 依赖 `emuframework/EmuApp.hh`，由
  `headless-test/stub-include/emuframework/EmuApp.hh` 提供最小替身；
  `FCEUPPU_FrameReady`/`emulateSound` 在 stubs.cc 置空。

结论（2026-09-10）：**78 项断言 + 3 个执行用例全过**，核心层（加载→
bank 推导→取模映射→CPU→IRQ→CHR RAM→XBuf 像素）对扩容 FS303 完全正常。

### 12.3 真机"黑屏"的最终根因（与 mapper195 无关）

症状：APK（`com.explusalpha.NesEmu`）打开"天使之翼2 传说之翼 定制版
.nes"（**中文文件名**）黑屏不崩溃；改成英文名 `game_hack.nes` 后正常；
game.nes / origin.nes（英文名）/ 冒险岛2 均正常。game.nes 与
game_hack.nes 仅差 4 字节（bank $01/$02 各 2 字节的 LDA 立即数 / NOP
逻辑补丁），与启动无关。

结论：**中文 ROM 文件名在前端加载路径上有 bug**（尚未修复，怀疑
imagine/SAF 打开或文件名相关推导环节，核心层已排除）。规避方式：
ROM 文件用英文/数字命名。若日后修复，从
`NES.emu/src/main/Main.cc loadContent`（`ioStream->size()` 与
`contentFileName()`）和 imagine 的 `IG::MapIO`/URI 打开入手。
另：logcat 抓不到 fceux 加载日志（release 构建 imagine logger 不输出
info），实机诊断不要走这条路。

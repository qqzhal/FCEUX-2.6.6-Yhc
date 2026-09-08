# FCEUX-2.6.6-Yhc 增强说明（Mapper 195 / 外星科技 FS303 支持）

> **本文档的用途**：本仓库是 `Yhc-Studio/FCEUX-2.6.6-Yhc` 的 fork，相对上游只做
> 了"让 Mapper 195 的扩容游戏能正常运行"这一件事。**当上游（或其它 FCEUX 版本）
> 更新后需要再次增强时，AI 助手照本文档逐条把修改点重新套用到新代码上即可**。
> 修改全部集中在 2 个文件、共 4 处，见下文"修改点清单"。
>
> 验证基准：`game.nes`（传说之翼，1280KB PRG）、`game2.nes`（纵横天下，1536KB
> PRG）、`origin.nes`（中文原版，512KB PRG）——三份都是 iNES Mapper 195（0xC3，
> flag6=0x30 flag7=0xC0，无 trainer，CHR 256KB，水平镜像）。上游原版只能跑
> origin.nes（而且 CHR 也可能错），game.nes/game2.nes 黑屏。

## 修改点清单（上游合并时的核对清单）

| # | 文件 | 位置 | 内容 |
|---|------|------|------|
| 1 | `src/ines.cpp` | `not_power2[]` 名单（约 532 行） | 名单里**必须有 195**：`{ 53, 195, 198, 228, 547 }` |
| 2 | `src/boards/mmc3.cpp` | `Mapper195_Init` | PRG 真实大小从 `info->totalFileSize - CHR` 推导（`PRGRomSize` 是 uppow2 填充值）；`PRGmask8[0] = 0xFF` 放开掩码 |
| 3 | `src/boards/mmc3.cpp` | `M195PW`（pwrap） | bank 号翻译：`0xFE→banks-2`、`0xFF→banks-1`、其余 `% banks`（非 2 幂 bank 数不能用位与） |
| 4 | `src/boards/mmc3.cpp` | `M195CW`（cwrap）/ `M195Power` | CHR 值 ≤3 → 4KB 板载 CHR RAM（chip 0x10），否则 CHR ROM；Power 时按 header 设镜像、清零 CHRRAM/WRAM/XRAM；$5000-$5FFF 挂独立 4KB XRAM（chip 0x12） |

详细原理和失败机制见下文，每条都附了"不修会怎样"的现象，方便在新版本上确认
问题是否复现。

---

## 背景：FS303 / Mapper 195 是什么

外星科技（Waixing FS303）的 MMC3 克隆板，ROM 特征：

- iNES 1.0，mapper = `(flag7 & 0xF0) | (flag6 >> 4)` = 0xC3 = 195
- PRG 可达 80×16KB（game.nes）、96×16KB（game2.nes）——**8KB bank 数 160/192，
  不是 2 的幂**（origin.nes 是 64，恰好是 2 的幂，所以上游"碰巧"能跑）
- CHR 256KB + **板载 4KB CHR RAM**（中文 hack 用它画字库 tile）
- CPU $5000-$5FFF 有**独立 4KB PRG-RAM**（VirtuaNES 的 XRAM）：游戏复位时把
  bank $42/$43 的 12KB 补丁层拷到 $5000-$7FFF 执行
- IRQ 与 MMC3 寄存器兼容（$C000/$C001/$E000/$E001），fceux 自带的
  `GameHBIRQHook = MMC3_hb`（每扫描线一个时钟）实测可用，**不需要**自定义 IRQ

参考实现（都实测能跑本组 ROM，行为有分歧时以它们为准）：

- `VirtuaNESex_src_191105/NES/Mapper/Mapper195.cpp`（VirtuaNESex 源码目录）
  （注意它的 MMU：`SetVROM_1K_Bank` 用 `% VROM_1K_SIZE` 取模、`SetCRAM_1K_Bank`
  用 `& 0x1F` 共 32KB CRAM——Mapper195 只用到槽 0-3）
- `web/emu/mappers/mapper195.js`（TSZY2 工程 web 模拟器内核，
  `% prgCount` 取模模型；注意其 `chrBankSelect`/`updateChrBanks` 的 $8000 写
  检测实际是死代码，CHR RAM 上电内容无关紧要——VirtuaNES 与 fceux 都是全 0）

## 修改点原理

### 1. ines.cpp：not_power2 名单必须含 195（否则 CHR 全丢）

fceux 加载 iNES 1.0 时：

```c
FCEU_fread(ROM, 1, (round) ? rom_size_bytes : not_round_size, fp);
if (vrom_size_bytes)
    FCEU_fread(VROM, 1, vrom_size_bytes, fp);
```

mapper 不在 `not_power2` 名单 → `round=true` → PRG 按 **uppow2 填充后的
2MB**（rom_size_bytes）读文件。game.nes 文件总共只有 1.5MB，这条 fread 会把
**整个 CHR 段一起吞进 PRG 缓冲**，随后 VROM 的 fread 从文件尾读到 0 字节，
VROM 缓冲保持 malloc 后的 0xFF 填充。

**现象**：游戏逻辑完全正常（PRG 前 1.28MB 数据是完整的）、能进开场、字幕正常
（字幕 tile 在 CHR RAM 里，是游戏运行时写的），但**所有 CHR ROM 图形是空
tile**——画面只剩大色块剪影。诊断手段：把 `CHRptr[0] + 页偏移` dump 出来与
ROM 文件的 CHR 段对比即可实锤。

### 2. PRG 真实 bank 数的推导

`CartInfo.PRGRomSize` 对 iNES 1.0 是 **uppow2 填充值**（ines.cpp 在 CRC 段
还有一次无条件覆盖 `PRGRomSize = ROM_size * 0x4000`，ROM_size 已是填充值），
不能直接用。可靠的推导：

```c
int prgbytes = (int)(info->totalFileSize - (uint32)VROM_size * 8192);
/* totalFileSize = 文件大小 - 16 字节头（有 trainer 会破坏 16KB 对齐，走 fallback） */
if (prgbytes < 512*1024 || prgbytes > 4096*1024 || (prgbytes & 0x3FFF))
    prgbytes = 512 * 1024;          /* 标准 FS303 兜底 */
M195_prgbanks = prgbytes >> 13;     /* game.nes=160, game2.nes=192 */
```

### 3. PRG bank 号翻译（非 2 幂不能用位掩码）

fceux 的 `setprg8r` 内部对 bank 号做 `V & PRGmask8[chip]`——位与只能表达
2 的幂。game.nes 160 个 bank → 掩码 159 (0x9F)，游戏写 `R6=0x42` 会静默落到
`0x42 & 0x9F = 0x02`：**12KB 补丁层从错误的 bank 拷贝，CPU 跑进未初始化 RAM
死循环（黑屏）**。固定库 `0xFE/0xFF & 0x9F = 0x9E/0x9F` 碰巧正确，所以复位
向量是对的，更具迷惑性。

解法：`PRGmask8[0] = 0xFF` 放开掩码，在 pwrap 里做值翻译（与 web 版的
`% prgCount` 模型一致）：

```c
static void M195PW(uint32 A, uint8 V) {
    if (V == 0xFE)      V = M195_prgbanks - 2;  /* FixMMC3PRG 的 ~1 */
    else if (V == 0xFF) V = M195_prgbanks - 1;  /* FixMMC3PRG 的 ~0 */
    else                V %= M195_prgbanks;
    setprg8(A, V);
}
```

### 4. CHR RAM 窗口 / XRAM / 镜像

- **M195CW**：寄存器值 ≤3 → `setchr1r(0x10, A, V)`（4KB 板载 CHR RAM 的槽
  V），否则 `setchr1r(0, A, V)`（CHR ROM）。对应 VirtuaNESex 的
  `SetBank_PPUSUB`。CHR ROM 是 256KB（2 的幂），fceux 的 CHRmask 机制天然
  正确，不用动。
- **$5000-$5FFF**：`setprg4r(0x12, 0x5000, 0)` + `CartBR/CartBW`，XRAM 是
  独立 4KB（chip 0x12），不能和 $6000-$7FFF 的 WRAM 或 CHR RAM 混用——
  游戏会把 12KB 补丁层拷到 $5000-$7FFF。
- **镜像**：`GenMMC3Power` 默认 `setmirror(1)`（垂直），要按 header 纠正
  （三份 ROM 都是水平）：Init 里存 `M195_mirror = info->mirror`，Power 里
  `setmirror(M195_mirror)`（参考 `Mapper4_Init`/`M4Power` 的做法）。
- **上电清零**：`CHRRAM/WRAM/XRAM` memset 0（VirtuaNES 上电行为）。
- **IRQ**：完全用 GenMMC3 自带的（`GameHBIRQHook = MMC3_hb`，每扫描线一个
  时钟，实测每帧 241 次）。不要用 fceux 的 `MapIRQHook`/PPU A12 钩子另搞
  一套——以前几轮调试的"IRQ 移植"全是被 bug #1/#3 误导的无用功。

## 上游更新后的再增强流程（给 AI 的操作步骤）

1. 拉上游新代码，对 `src/ines.cpp` 和 `src/boards/mmc3.cpp` 做三方对照：
   上游新版 vs 本仓库旧版（`git log --oneline -- src/boards/mmc3.cpp` 从
   `Mapper195: modulo PRG bank mapping...` 系列提交取旧实现）。
2. 按上文"修改点清单"把 4 处改动套到新版代码上。API 若有变化，保持语义：
   名单成员 / 掩码放开 / 取模翻译 / ≤3→CHR RAM / XRAM / header 镜像。
3. 编译（GitHub Actions：push 到 main 自动构建 `windows.yml`，产物
   artifact 名 `fceux-win64`，含 `output64/fceux64.exe` + lua dll）。
4. 验证（按顺序）：
   - `origin.nes`：开场"虎看到食物一般球被断下了！"对话框画面，色彩图案完整
   - `game.nes`：开场剪影场景要有**白云、看台条纹、球场横纹、人物细节**
     （只有色块 = 修改点 1 回归了；黑屏死循环 = 修改点 2/3 回归了）
   - `game2.nes`：同 game.nes（banks=192）
5. 调试手段（曾经用过、可复刻）：
   - MMC3 层日志：M195PW/M195CW 里记录 `(A, V, X.PC, scanline)`；MapIRQHook
     里记录 256 字节块首访 PC（判断执行流卡死位置的利器）
   - CHR 内存 dump：`CHRptr[0] + 页*1024` 与 ROM 文件 CHR 段对比（实锤
     加载问题）；CHRRAM 渲染成 PNG（实锤字库写入）
   - 注意 fopen 的 stdio 缓冲：进程被 taskkill /F 强杀会丢日志，要优雅关
     闭（WM_CLOSE）或定期 fflush

## 本仓库相对上游的完整差异（合并前核对）

```
.gitignore                    +1   （忽略本地 testrun 测试目录）
.github/workflows/windows.yml +28  （CI：msbuild fceux64 Release x64 + artifact）
src/boards/mmc3.cpp           ~130 （Mapper195 重写，即本文档）
src/ines.cpp                  +1   （not_power2 名单加 195）
```

## 已知未处理项（上游更新时可选做）

- UNIF 格式的 FS303 dump 未处理（只支持 iNES 195）
- NES 2.0 格式的 mapper 195 未专门验证（本组 ROM 都是 iNES 1.0）
- save state 兼容性未与其它模拟器交叉验证（fceux 自身存读档自洽）

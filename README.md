# Badlands Coupling Fix: Strata 可选化 + 耦合输出修复

本仓库合并了 Badlands 在与 Underworld2 耦合 (`udw=1`) 时的多项修复，涵盖 strata 文件生成可选化、沉积/侵蚀输出为零、后续步输出丢失、以及重启时缺少 strata 文件等问题。

This repository merges multiple fixes for Badlands when coupled with Underworld2 (`udw=1`), including optional strata generation, zero cumdiff fix, missing output fix, and restart strata bypass.

---

## 修改文件列表 / Modified Files

| 文件 / File | 安装目标路径 / Install Path | 说明 / Description |
|---|---|---|
| `model.py` | `badlands/model.py` | strata 可选化（XML 检测）+ cumdiff 修复 + 输出条件修复 |
| `buildMesh.py` | `badlands/simulation/buildMesh.py` | `laytime=0` 时跳过 stratigraphic TIN 初始化 |
| `buildFlux.py` | `badlands/simulation/buildFlux.py` | `CFLtime` 在 minDT/maxDT 钳制后追加 `tEnd-tNow` 封顶 |
| `strataMesh.py` | `badlands/underland/strataMesh.py` | restart 缺少 `sed.time*.hdf5` 时发出警告并继续（bypass） |

详细修改说明请参见 [Badlands_Strata_Optional_Changes.md](./Badlands_Strata_Optional_Changes.md)。

For detailed modification documentation, see [Badlands_Strata_Optional_Changes.md](./Badlands_Strata_Optional_Changes.md).

---

## 修复内容概览 / Fix Overview

### 1. Strata 可选化 / Optional Strata (model.py + buildMesh.py)

当 Badlands 被 Underworld 调用 (`udw=1`) 时：
- **XML 中有 `<strata>`**：正常启用 strata，生成对应输出文件。
- **XML 中无 `<strata>`**：不生成 strata 输出文件，不影响耦合计算，restart 时不需要 `sed.time*.hdf5`。

实现方式：在 `load_xml()` 中检测 XML 是否包含 `<strata>` 节点，若无则强制 `laytime=0`、`stratdx=0`，从源头关闭 strata 构建。所有 strata 相关代码路径均加 `self.strata is not None` 守卫。

### 2. cumdiff 始终为 0 的修复 / Zero Cumdiff Fix (model.py)

Underworld 耦合时 `surfaceProcesses.py` 设 `next_display = run_until = tEnd`，导致 `tStop = tEnd`。原代码 `if tStop < tEnd:` 走 else 分支跳过 `sediment_flux`，cumdiff 恒为 0。

修复：改为 `tStop <= tEnd`（等价写法：`fluxTarget = tStop if tStop < tEnd else tEnd` 并无条件调用 `sediment_flux`）。

### 3. 后续步输出丢失的修复 / Missing Output Fix (model.py + buildFlux.py)

修复一后，`outbdls/h5/` 仍只有 `time0`，后续步无输出。根因：
- `buildFlux.py` 中 `CFLtime = min(CFLtime, tEnd - tNow)` 被 minDT 钳制（默认 1.0 年）重新抬升，导致 `tNow` 越过 `tEnd` 约 1 年。
- 主循环后输出条件用 `==` 精确比较，越界后不满足。

修复（3 处）：
1. `model.py`：主循环结束后 `if self.tNow >= tEnd: self.tNow = tEnd`（snap 回收）。
2. `model.py`：输出条件 `==` → `>=`。
3. `buildFlux.py`：在 minDT/maxDT 钳制**之后**追加 `CFLtime = min(CFLtime, tEnd - tNow)`。

### 4. 重启缺少 strata 文件的 bypass / Restart Strata Bypass (strataMesh.py)

当 strata 已启用但重启文件 `sed.time*.hdf5` 缺失时，原代码抛出 `ValueError` 终止运行。此修改将其改为发出警告并将 `rstep` 重置为 0，允许模拟继续。

> 注：修复 1（strata 可选化）从源头避免了 strataMesh 的创建，此 bypass 作为额外安全网保留。
> Note: Fix 1 (optional strata) prevents strataMesh creation at the source; this bypass is kept as an additional safety net.

---

## 安装方法 / Installation

### 1. 备份原文件 / Backup Original Files

```bash
BADLANDS_DIR=/path/to/site-packages/badlands

cp "$BADLANDS_DIR/model.py" "$BADLANDS_DIR/model.py.bak"
cp "$BADLANDS_DIR/simulation/buildMesh.py" "$BADLANDS_DIR/simulation/buildMesh.py.bak"
cp "$BADLANDS_DIR/simulation/buildFlux.py" "$BADLANDS_DIR/simulation/buildFlux.py.bak"
cp "$BADLANDS_DIR/underland/strataMesh.py" "$BADLANDS_DIR/underland/strataMesh.py.bak"
```

### 2. 替换文件 / Replace Files

```bash
git clone https://github.com/HonghaoXiong/Badlands_coupling_fixedStrata.git
cd Badlands_coupling_fixedStrata

cp model.py      "$BADLANDS_DIR/model.py"
cp buildMesh.py  "$BADLANDS_DIR/simulation/buildMesh.py"
cp buildFlux.py  "$BADLANDS_DIR/simulation/buildFlux.py"
cp strataMesh.py "$BADLANDS_DIR/underland/strataMesh.py"
```

### 3. 验证 / Verify

```bash
python -m py_compile "$BADLANDS_DIR/model.py"
python -m py_compile "$BADLANDS_DIR/simulation/buildMesh.py"
python -m py_compile "$BADLANDS_DIR/simulation/buildFlux.py"
python -m py_compile "$BADLANDS_DIR/underland/strataMesh.py"
```

---

## 跨机器检查清单 / Cross-Machine Checklist

1. `model.py` 主循环：`tStop == tEnd` 时也调用 `sediment_flux`（`<=` 或 fluxTarget 写法）。
2. `buildFlux.py`：`flow.compute_sedflux` 调用前，`CFLtime = min(CFLtime, tEnd - tNow)` 封顶位于 minDT/maxDT 钳制**之后**。
3. `model.py` 主循环后：输出条件用 `>=` 而非 `==`；循环结束后 snap `tNow` 到 `tEnd`。
4. 短步长耦合测试：确认 `tin.time1.hdf5` 及后续步的 `cumdiff` 非零，且每步都有输出。

---

## 合并说明 / Merge Note

本仓库合并了以下两个仓库的内容 / This repository merges content from:

- ~~`HonghaoXiong/Badlands-restart`~~ — strataMesh.py restart bypass（已合并到本仓库）
- `HonghaoXiong/Badlands_coupling_fixedStrata` — model.py strata 可选化（本仓库为更新版）

---

## 免责声明 / Disclaimer

These patches are provided "as is". Please ensure you understand the implications of these modifications for your specific simulation needs.

本补丁"按原样"提供。请确保您了解这些修改对特定模拟需求可能产生的影响。

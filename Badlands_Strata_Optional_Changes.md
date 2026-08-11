# Badlands strata 可选化（Underworld 耦合 udw=1）修改汇总

生成时间：2026-02-06

## 目标需求

当 Badlands 被 Underworld 调用（`udw=1`）时：

1. **如果 XML 中显式包含 `<strata>...</strata>`**：
   - Badlands 读取并启用 strata（按 XML 中的 `stratdx/laytime/...` 执行），生成对应 strata 输出文件。
2. **如果 XML 中不包含 `<strata>`**：
   - Badlands **不生成 strata 输出文件**；
   - **不影响** Badlands 与 Underworld 的耦合与计算；
   - Badlands restart 时 **不需要**读取/传入 `sed.time*.hdf5`（因为 strata 模块根本不启用）。

## 功能行为矩阵

| 场景 | XML 中 `<grid><udw>1</udw></grid>` | XML 中是否有 `<strata>` | strata 是否启用 | 是否会读取 `sed.time*.hdf5`（restart） |
|---|---:|---:|---:|---:|
| A | 1 | 有 | 是 | 是（仅在 `restartStep>0` 且 strata 启用时） |
| B | 1 | 无 | 否 | 否 |

## 修改文件列表

- Badlands 侧主要修改：
  - `/home/user/Downloads/badlands-coupling-update/badlands/badlands/model.py`
  - `/home/user/Downloads/badlands-coupling-update/badlands/badlands/simulation/buildMesh.py`
- restart 读取 `sed.time*.hdf5` 的位置（**未修改**，但现在会在 strata 未启用时绕开）：
  - `/home/user/Downloads/badlands-coupling-update/badlands/badlands/underland/strataMesh.py`
- Underworld 侧（仅核查，**未修改**）：
  - `/home/user/Downloads/underworld2-2.17.x/.pixi/envs/default/lib/python3.11/site-packages/underworld/UWGeodynamics/surfaceProcesses.py`

---

# 1) model.py：检测 XML 是否包含 `<strata>` 并动态关闭 strata

文件：`/home/user/Downloads/badlands-coupling-update/badlands/badlands/model.py`

## 1.1 在 `load_xml()` 中检测 `<strata>`（当前行号 95-111）

### 修改后（当前代码）

> 位置：model.py 第 95-111 行

```python
95  # Only the first node should create a unique output dir
96  self.input = xmlParser.xmlParser(filename, makeUniqueOutputDir=True)
97  import xml.etree.ElementTree as etree
98  try:
99      _root = etree.parse(filename).getroot()
100     _strata = _root.find("strata")
101     self.input.strata_enabled = _strata is not None
102 except Exception:
103     self.input.strata_enabled = True
104
105 if not getattr(self.input, "strata_enabled", True):
106     self.input.laytime = 0.0
107     self.input.stratdx = 0.0
108     if hasattr(self.input, "laststrat"):
109         self.input.laststrat = False
110
111 self.tNow = self.input.tStart
```

### 修改前（原始代码片段）

原始逻辑只做 XML 解析，不判断是否存在 `<strata>`：

```python
# Only the first node should create a unique output dir
self.input = xmlParser.xmlParser(filename, makeUniqueOutputDir=True)
self.tNow = self.input.tStart
```

### 作用说明

- 当 XML **没有 `<strata>`** 时：
  - `strata_enabled=False`
  - 强制 `laytime=0`、`stratdx=0`（从源头关闭 strata 构建与输出）
- 当 XML **有 `<strata>`** 时：保持原参数不变。

---

## 1.2 `_rebuild_mesh()` 中 strata 更新加守卫（当前行号 274-276）

### 修改后（当前代码）

> 位置：model.py 第 274-276 行

```python
274  # Update stratigraphic mesh
275  if self.strata is not None and self.input.stratdx > 0:
276      self.strata.update_TIN(self.FVmesh.node_coords[:, :2])
```

### 修改前（原始代码片段）

原逻辑只判断 `stratdx`，可能在 strata 未正确初始化时仍尝试调用：

```python
# Update stratigraphic mesh
if self.input.stratdx > 0:
    self.strata.update_TIN(self.FVmesh.node_coords[:, :2])
```

---

## 1.3 主循环中 strata 更新加守卫（当前行号 676-697）

### 修改后（当前代码）

> 位置：model.py 第 676-697 行

```python
676  # Update next stratal layer time
677  if self.strata is not None and laytime > 0 and self.tNow >= self.force.next_layer:
678      self.force.next_layer += laytime
679      if self.straTIN is not None:
680          self.straTIN.step += 1
681      if self.tNow == tEnd:
682          self.strat_write = 1
683      else:
684          self.strat_write = 0
685      if self.input.laststrat == False:
686          self.strat_write = 1
687      sub = self.strata.buildStrata(
688          self.elevation,
689          self.cumdiff,
690          self.force.sealevel,
691          self.recGrid.boundsPt,
692          self.strat_write,
693          self.outputStep,
694      )
695      self.elevation += sub
696      self.cumdiff += sub
```

### 修改前（原始代码片段）

原逻辑在 `tNow>=next_layer` 时进入，内部再判断 `if self.strata:`，但没有把 `laytime` 与 `strata is not None` 作为外层硬条件：

```python
# Update next stratal layer time
if self.tNow >= self.force.next_layer:
    self.force.next_layer += laytime
    if self.straTIN is not None:
        self.straTIN.step += 1
    if self.strata:
        ...
        sub = self.strata.buildStrata(...)
```

---

## 1.4 结束阶段 udw 强制 strata 输出加守卫（当前行号 842-873）

### 修改后（当前代码）

> 位置：model.py 第 842-873 行

```python
842  # Update next stratal layer time
843  if self.strata is not None and laytime > 0 and self.tNow >= self.force.next_layer:
844      ...
860  if self.input.udw == 1 and self.strata is not None and laytime > 0:
861      purple = ""
862      endcol = ""
863      print(purple + "Stratal layering output to align with Underworld coupling" + endcol)
864      self.strat_write = 1
865      self.strata.buildStrata(
866          self.elevation,
867          self.cumdiff,
868          self.force.sealevel,
869          self.recGrid.boundsPt,
870          self.strat_write,
871          self.outputStep + 1,
872      )
```

### 修改前（原始代码片段）

原逻辑在 `udw==1` 时**无条件**强制 strata write：

```python
# if Underworld coupling is active, force a strata write at the end
if self.input.udw==1:
    print(...)
    self.strat_write = 1
    sub = self.strata.buildStrata(...)
```

### 作用说明

- 现在只有在 `self.strata` 实际存在且 `laytime>0` 时才会在 udw 模式下强制 strata 输出。
- 当 XML 没有 `<strata>` 时（`laytime=0`、`self.strata=None`），此段代码不会执行，因此不会生成 strata 文件。

---

# 2) buildMesh.py：避免 laytime=0 时仍初始化 stratigraphic TIN

文件：`/home/user/Downloads/badlands-coupling-update/badlands/badlands/simulation/buildMesh.py`

## 2.1 为 `straTIN` 初始化添加 `laytime>0` 守卫（当前行号 210-212）

### 修改后（当前代码）

```python
210  # Stratigraphic TIN initialisation
211  if input.rockNb > 0 and (input.laytime and input.laytime > 0):
212      layNb = int((input.tEnd - input.tStart) / input.laytime) + 2
```

### 修改前（原始代码片段）

```python
# Stratigraphic TIN initialisation
if input.rockNb > 0:
    layNb = int((input.tEnd - input.tStart) / input.laytime) + 2
```

### 作用说明

当 XML 无 `<strata>` 时我们会把 `laytime` 强制置 0，如果这里不加守卫，会出现 `1/0` 或 `inf` 导致的异常（例如 `OverflowError`）。该守卫保证 strata 被关闭时不会进入 stratigraphic TIN 初始化。

---

# 3) strataMesh.py：restart 读取 `sed.time*.hdf5` 的触发条件（未改）

文件：`/home/user/Downloads/badlands-coupling-update/badlands/badlands/underland/strataMesh.py`

此文件本次没有修改，但它说明了为什么“关闭 strata”就能避免 restart 强制读取 `sed.time*.hdf5`：

> 位置：strataMesh.py 第 88-115 行（仅在 strataMesh 被构建且 `rstep>0` 时执行）

```python
88  if rstep > 0:
89      if os.path.exists(rfolder):
90          folder = rfolder + "/h5/"
91          fileCPU = "sed.time%s.hdf5" % (rstep)
...
107     df = h5py.File("%s/h5/sed.time%s.hdf5" % (rfolder, rstep), "r")
108     layDepth = numpy.array((df["/layDepth"]))
...
114     self.step = rstlays
```

现在的策略是：当 XML 不包含 `<strata>` 时，从上游（`load_xml` + `buildMesh`）开始就不创建 strataMesh，因此不会进入上述 restart 读文件逻辑。

---

# 4) 验证要点（已在环境中检查）

- XML **无 `<strata>`**：
  - `Model().load_xml(xml)` 后 `strata_enabled=False`，且 `laytime=0.0`、`stratdx=0.0`
- XML **有 `<strata>`**（例如你的 `badlands.xml`）：
  - `strata_enabled=True`，且 `laytime/stratdx` 保持 XML 配置值
- 语法检查：
  - `python -m py_compile` 对 `model.py` 与 `buildMesh.py` 编译通过

---

# 5) Underworld 耦合时沉积/侵蚀输出为 0 的问题（2026-08-11 新增）

在 Underworld 调用 Badlands（`udw=1`）且 XML 中**没有 `<strata>`** 时，除了上面第 1、2 节的修改外，还发现两个导致 `cumdiff` 始终为 0 的 bug。这两个问题在独立运行 Badlands 时不易出现，但在耦合模式下会同时触发。

## 5.1 现象

- 耦合运行后，`outbdls_Ref/h5/tin.time*.hdf5` 中的 `cumdiff` 全为 0。
- 单独用同样的 `badlands.xml` 跑 Badlands 时，`cumdiff` 可以非零。
- 日志里能看到 `Processing surface with Badlands...Done`，但地表过程没有留下沉积/侵蚀记录。

## 5.2 根因一：`model.py` 中 `tStop < tEnd` 导致最后区间跳过沉积计算

### 位置

`/home/user/Downloads/underworld2-17-3/.pixi/envs/default/lib/python3.11/site-packages/badlands/model.py`

### 问题说明

`Underworld/UWGeodynamics/surfaceProcesses.py` 在每次耦合步会把 `badlands_model.force.next_display` 设为当前 UW 步的结束时间 `run_until`：

```python
run_until = self.time_years + dt_years
bdm.force.next_display = run_until
bdm.run_to_time(run_until)
```

在 `model.py` 主循环里：

```python
tStop = min([
    self.force.next_display,
    self.force.next_layer,
    ...,
    tEnd,
])
```

因为 `next_display == run_until == tEnd`，所以 `tStop == tEnd`。原代码判断：

```python
if tStop < tEnd:
    ... = buildFlux.sediment_flux(...)
else:
    self.tNow = tEnd
```

这就导致 **Badlands 在最后一个时间区间完全不调用 `sediment_flux`**，直接跳到 `tEnd`，所以 `cumdiff` 保持为 0。

### 修改后

```python
if tStop <= tEnd:
    (
        self.tNow,
        self.elevation,
        self.cumdiff,
        self.cumhill,
        self.cumfail,
        self.slopeTIN,
    ) = buildFlux.sediment_flux(...)
else:
    self.tNow = tEnd
```

将 `<` 改为 `<=`，让 `tStop == tEnd` 时仍然执行沉积计算。

## 5.3 根因二：`buildFlux.py` 中 `CFLtime` 越界导致输出丢失

### 位置

`/home/user/Downloads/underworld2-17-3/.pixi/envs/default/lib/python3.11/site-packages/badlands/simulation/buildFlux.py`

### 问题说明

即使沉积计算被调用，`sediment_flux` 内部返回的时间步 `timestep` 可能大于剩余的 `tEnd - tNow`。例如：

```text
tNow=233.0, tEnd=233.685, CFLtime=1.0, timestep=1.0
```

执行后 `tNow` 变成 234.0，**超过了 `tEnd`**。而主循环后的输出判断是：

```python
if (
    self.input.udw == 0
    or self.tNow == tEnd
    or self.tNow == self.force.next_display
):
    checkPoints.write_checkpoints(...)
```

对于 `udw=1`，只有当 `self.tNow == tEnd` 或 `self.tNow == next_display` 时才写输出。`tNow` 越界后这两个条件都不满足，所以**计算出来的沉积结果写不进 h5 文件**，看起来 `cumdiff` 仍然是 0。

### 修改后

在调用 `flow.compute_sedflux` 之前，把 `CFLtime` 限制在剩余时间内：

```python
# Do not let the sediment timestep overshoot the requested end time
CFLtime = min(CFLtime, tEnd - tNow)

timestep, sedchange, erosion, deposition, slopeTIN = flow.compute_sedflux(
    ...
)
```

这样 `tNow` 会精确到达 `tEnd`，输出条件成立，`tin.timeN.hdf5` 能正确写入非零 `cumdiff`。

## 5.4 修改文件列表（新增）

- `/home/user/Downloads/underworld2-17-3/.pixi/envs/default/lib/python3.11/site-packages/badlands/model.py`
- `/home/user/Downloads/underworld2-17-3/.pixi/envs/default/lib/python3.11/site-packages/badlands/simulation/buildFlux.py`

## 5.5 验证结果

- 修复后重新运行 Underworld 耦合模型，第一步结束写出的 `tin.time1.hdf5` 中 `cumdiff` 即非零：
  - `min = -0.065`, `max = 0.787`
  - 正点数 46,162，负点数 78,903
- 作为副作用（正向），现在**每个 UW 步结束都会写出 `tin.timeN.hdf5`**，而不再只按 `tDisplay` 的 10000 年间隔写，输出密度更高。

## 5.6 去其他设备检查的要点

如果其他机器上也按本文档修改了 Badlands，请重点检查：

1. `badlands/model.py` 中主循环的 `if tStop < tEnd:` 是否已改为 `if tStop <= tEnd:`。
2. `badlands/simulation/buildFlux.py` 中 `flow.compute_sedflux` 调用前是否有 `CFLtime = min(CFLtime, tEnd - tNow)`。
3. 跑一个短步长（例如 200–500 年）的耦合测试，确认 `tin.time1.hdf5` 里的 `cumdiff` 非零。


---

# 6) 与 underworld2-17-3 机器排查结论的对照（2026-08-11）

另一台机器（`underworld2-17-3`）的排查文档（本文第 5 节）与本机
（`underworld2-2.17.x`，badlands 2.3.1，与 PyPI 官方源码 diff 仅 model.py /
buildMesh.py 两处改动）对照结论：

## 6.1 根因一（tStop < tEnd 跳过 sediment_flux）：两边是同一个问题

- 第 5.2 节的修法 `<` → `<=` 与本机已应用的修法
  （`fluxTarget = tStop if tStop < tEnd else tEnd` 并无条件调用
  `sediment_flux`）**功能完全等价**：`tStop = min(..., tEnd, ...)` 恒有
  `tStop ≤ tEnd`，`<=` 恒真，else 分支为死代码。
- 本机应用的版本备份：`model.py.bak-cumdiff-fix`。

## 6.2 根因二（CFLtime 越界）：仅适用于旧版 badlands，本机不存在

- 本机 badlands 2.3.1 的 `buildFlux.py` 第 299 行**原本就有**
  `CFLtime = min(CFLtime, tEnd - tNow)`，且后续 minDT/maxDT 钳制的所有分支
  均保持该上限；`flowNetwork.compute_sedflux` 内部还有
  `if newdt > dt: newdt = dt` 硬封顶，保证 `timestep ≤ CFLtime ≤ tEnd-tNow`，
  不会冲过头。
- 实证（仅适用于根因一）：本机坏运行产出的 18 个 tin/flow 输出时间戳与 UW
  步终点精确相等，是因为当时 `sediment_flux` 根本没被调用、`tNow = tEnd` 来自
  else 分支的精确赋值；一旦侵蚀计算真正运行，tNow 就会因 CFL 子步累加而偏离
  tEnd（见第 8 节）。
- 结论（**2026-08-11 下午更正：原结论有误**）：第 299 行的封顶确实存在，
  但它**紧接着被 minDT 钳制重新抬升**（默认 minDT=1.0 年，把小于 1 年的收尾
  子步抬回 1.0 年），所以根因二在 badlands 2.3.1 上**同样存在**；正确的补丁
  位置是在 minDT/maxDT 钳制**之后**再加一道封顶（见 8.3 第 3 条）。

## 6.3 跨机器检查清单（合并版）

1. `model.py` 主循环：`tStop` 判断必须保证 `tStop == tEnd` 时也调用
   `sediment_flux`（`<=` 或 fluxTarget 写法均可）。
2. `buildFlux.py`：确认 `flow.compute_sedflux` 调用前存在
   `CFLtime = min(CFLtime, tEnd - tNow)` 封顶，**且必须位于 minDT/maxDT
   钳制之后**（仅 2.3.1 第 299 行那一道不够，见第 8 节）。
3. `model.py` 主循环后：输出条件用 `>=` 而非 `==`，且建议在循环结束后把
   越界的 `tNow` 回收到 `tEnd`（见第 8 节）。
4. 用短步长耦合测试验证 `tin.time1.hdf5` 的 `cumdiff` 非零，**并确认后续步
   （time2、time3…）持续产出、时间戳与 UW 步终点对齐**。


---

# 7) 当前服务器（Wholeworks/underworld2-17-3）排查与修复（2026-08-11）

对当前服务器 `/home/user/Wholeworks/underworld2-17-3/`（badlands 2.3.1）
做的完整排查，并修复了两个导致 Badlands 输出异常的问题。

涉及文件：
- `/home/user/Wholeworks/underworld2-17-3/.pixi/envs/default/lib/python3.11/site-packages/badlands/model.py`
- `/home/user/Wholeworks/underworld2-17-3/.pixi/envs/default/lib/python3.11/site-packages/badlands/simulation/buildFlux.py`

## 7.1 排查前状态

| 项 | 文档节 | 状态 |
|---|---|---|
| strata 可选化（1.1–1.4） | 第 1–4 节 | ✓ 已应用（有 `model.py.orig` 备份） |
| 根因一（`tStop < tEnd` 跳过 sediment_flux） | 5.2 | ✗ **未修复**（仍是 `<`） |
| 根因二（CFLtime 越界） | 5.3 / 6.2 | ✓ 已存在（2.3.1 第 299 行自带） |

`model.py.orig` 与 `model.py` 的 diff 显示只改了 strata 可选化（5 处），
没有第 5.2 节的 `<=` 修复。

## 7.2 修复一：`tStop < tEnd` → `tStop <= tEnd`（同第 5.2 节）

### 位置

`model.py` 主循环（第 775 行）

### 修改

```python
# 修改前
if tStop < tEnd:
    (...) = buildFlux.sediment_flux(...)
else:
    self.tNow = tEnd

# 修改后
if tStop <= tEnd:
    (...) = buildFlux.sediment_flux(...)
else:
    self.tNow = tEnd
```

### 作用

Underworld 耦合时 `surfaceProcesses.py` 设 `next_display = run_until = tEnd`，
` tStop = min(..., tEnd, next_display, ...) = tEnd`。原代码 `tStop < tEnd` 为 False，
走 else 分支跳过 `sediment_flux`，`cumdiff` 恒为 0。改 `<=` 后正常调用沉积计算。

## 7.3 修复二：主循环后 `==` → `>=`（新发现，第 5、6 节未涵盖）

### 现象

修复一之后重跑模型，`outbdls/h5/` 仍然只有 `tin.time0.hdf5`、`flow.time0.hdf5`，
后续步的 `time1`、`time2`... 都没输出。日志显示后续步的 `Processing surface with Badlands...Done`
正常完成，但没有 `Writing outputs` 行。

### 根因：两个因素叠加

**因素 A：主循环内 `write_checkpoints` 在 `sediment_flux` 之前**

`model.py` 主循环内的执行顺序：
```
while self.tNow < tEnd:
    ...                                     # strata, streamflow 等
    if self.tNow >= self.force.next_display:# 第 714 行 ← 先检查（上一步的 tNow）
        checkPoints.write_checkpoints(...)
    tStop = min(...)                        # 第 761 行
    sediment_flux(...)                      # 第 782 行 ← 后更新 tNow
```

`write_checkpoints` 检查的是**上一步 `sediment_flux` 留下的 `tNow`**，不是当前步的。
Underworld 每次调用前设 `next_display = run_until`，但此时 `tNow` 还是上一步的值
（< `run_until`），`tNow >= next_display` 不满足，主循环内不写。

**因素 B：主循环后用 `==` 精确比较，CFLtime round 导致 `tNow` 略超 `tEnd`**

主循环后（第 876–881 行）的输出条件：
```python
if (
    self.input.udw == 0
    or self.tNow == tEnd                      # ← 精确相等
    or self.tNow == self.force.next_display   # ← 精确相等
):
    checkPoints.write_checkpoints(...)
```

`buildFlux.py` 第 295–296 行把 CFLtime round 到整数：
```python
if CFLtime > 1.0:
    CFLtime = float(round(CFLtime - 0.5, 0))
```

导致 `tNow += timestep` 后 `tNow` 略超 `tEnd`（如 `tNow=339.0` vs `tEnd=338.527929`），
`==` 比较失败，三个条件都不满足，不写。

### 修复

`model.py` 第 879–880 行，把 `==` 改为 `>=`：

```python
# 修改前
or self.tNow == tEnd
or self.tNow == self.force.next_display

# 修改后
or self.tNow >= tEnd
or self.tNow >= self.force.next_display
```

### 验证

修复后重跑（任务 28），日志显示每步都有 `Writing outputs`：
```
Step 1: Writing outputs (tNow=0.0)   → time0（初始状态）
        Writing outputs (tNow=148.0) → time1
Step 2: Writing outputs (tNow=339.0) → time2
Step 3: Writing outputs (tNow=574.0) → time3
```

## 7.4 第一步输出两个文件的现象说明

第一次调用 `run_to_time` 时 `simStarted=False`，触发 `next_display = tStart = 0`
（`model.py` 第 341 行）。主循环内 `tNow(0) >= next_display(0)` 满足 → 写 time0
（初始状态，cumdiff=0）。sediment_flux 执行后主循环退出，`tNow >= tEnd` 满足 → 写 time1。

后续步 `simStarted=True`，`next_display` 由 Underworld 设为 `run_until`，
主循环内 `tNow(上一步值) < next_display` 不满足，只在主循环后写一次。

**数据对应关系**：

| Badlands 文件 | tNow | 含义 | 对应 UW step |
|---|---|---|---|
| time0 | 0.0 | 初始状态（cumdiff=0） | 无（额外快照） |
| time1 | ~148 | 第 1 步沉积后 | UW step 1 |
| time2 | ~339 | 第 2 步沉积后 | UW step 2 |
| timeN | … | 第 N 步沉积后 | UW step N |

因 time0 是额外的初始快照，**UW step N 对应 Badlands time(N+1)**。
分析时如不需要初始状态，跳过 time0 即可。

## 7.5 更新后的跨机器检查清单

1. `model.py` 主循环 `tStop` 判断：`tStop == tEnd` 时也必须调用 `sediment_flux`
   （`<=` 或 fluxTarget 写法均可）。—— 第 5.2 节
2. `buildFlux.py`：`flow.compute_sedflux` 调用前存在 `CFLtime = min(CFLtime, tEnd - tNow)`
   封顶（2.3.1 自带；旧版需手补）。—— 第 5.3 / 6.2 节
3. `model.py` 主循环后的 write_checkpoints 条件：`tNow == tEnd` 必须改为 `tNow >= tEnd`，
   否则 CFLtime round 导致 `tNow` 略超 `tEnd` 时输出丢失。—— 第 7.3 节（新）
4. 短步长耦合测试：确认 `tin.time1.hdf5` 及后续步的 `cumdiff` 非零，且每步都有输出。

---

# 8) udw 耦合下只写 time0、后续输出丢失的问题与修复（2026-08-11 下午，本机第二批修复）

## 8.1 现象

应用第 5.2/7.2 节修复（`tStop <= tEnd`）后，剥蚀/沉积恢复正常，但
`outbdls_Ref/h5/` 只有 `flow.time0.hdf5`、`tin.time0.hdf5`，后续 UW 步正常
完成（日志有 `Processing surface with Badlands...Done`）却没有 time1、time2…
写出。两台服务器（本机与 underworld2-17-3）现象一致。

## 8.2 机制（三个因素叠加）

1. **time0 的来源**：第一次 `run_to_time` 调用时，`simStarted` 初始化块会把
   `next_display` 重置为 `tStart`（=0），主循环第一轮 `tNow(0) >= next_display(0)`
   成立，写出初始状态（cumdiff=0）。这就是"只有第一步"的那一步。
2. **tNow 冲过步终点**：`buildFlux.py sediment_flux` 中，
   `CFLtime = min(CFLtime, tEnd - tNow)`（第 299 行）封顶之后，紧随的
   minDT 钳制（xmlParser 默认 `minDT = 1.0` 年）会把小于 1 年的收尾子步
   重新抬回 1.0 年：`CFLtime = max(input.minDT, CFLtime)`。于是最后一段
   积分超出剩余时间，`tNow` 越过 `tEnd` 最多约 1 年。
   本机 2026-08-11 运行日志证据：
   - 步 2 目标 526.494604 → `tNow = 527.0`
   - 步 3 目标 870.581759 → `tNow = 871.0`
   （第 7.3 节把越界归因于 CFL 整数 round；round 只让 tNow 以整数年逼近、
   落在 tEnd 之前，真正把 tNow 推过 tEnd 的是 minDT 抬升。）
3. **输出条件精确相等**：主循环后的写出条件
   `self.tNow == tEnd or self.tNow == self.force.next_display`
   因越界永不成立 → 不写。

## 8.3 本机修复（3 处，均在 site-packages/badlands/）

1. `model.py` 主循环结束后（`tloop` 打印之前）加 snap：

   ```python
   if self.tNow >= tEnd:
       self.tNow = tEnd
   ```

   消除舍入/越界残留，让 badlands 时钟与 UW 步时间精确同步。
2. `model.py` 主循环后输出条件 `==` → `>=`（与第 7.3 节修法一致，双保险）。
3. `buildFlux.py sediment_flux`：在 minDT/maxDT 钳制块**之后**追加
   `CFLtime = min(CFLtime, tEnd - tNow)`，从源头消除越界。
   （这才是"根因二"在 2.3.1 上需要的补丁形态；第 6.2 节原结论已更正。）

## 8.4 验证（harness 精确模拟 UW 驱动，21 个 UW 步）

- 无 strata 与有 strata 均写出 22 步 ×（tin+flow）= 44 个 xmf；
- 全部输出时间戳与 UW 步终点**精确相等**（最大偏差 0.0）；
- `cumdiff` 非零，含剥蚀（负）与沉积（正）；日志 `tNow = ...` 与每步
  `run_until` 一致，不再出现 527.0/871.0 这类越界值。

## 8.5 给 underworld2-17-3 的建议

第 7.3 节单独把 `==` 改 `>=` 即可恢复写出，但写出的时间戳是 CFL 整数 round +
minDT 越界后的值（如 148.0 / 339.0 / 574.0），与 UW 模型时间最多漂移约 1 年。
如需时间戳精确对齐 UW 步，请补加 8.3 的第 1、3 条（snap + buildFlux 再封顶）。

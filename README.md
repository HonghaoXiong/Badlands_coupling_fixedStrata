# Badlands 代码修改指南：控制 Strata 文件生成

本指南旨在指导如何修改 Badlands 模型代码 (`model.py`)，以增加一个开关，允许在与 Underworld 耦合 (Coupling) 运行时选择是否生成地层 (Strata) 文件。

## 修改目标文件
*   **文件名**: `model.py`
*   **路径**: `/Users/haibinyang/Xiong_test/model.py`

---

## 修改步骤 1：增加控制参数

我们需要在主运行函数 `run_to_time` 中增加一个名为 `generate_udw_strata` 的开关参数。

*   **位置**: 第 **290** 行左右
*   **操作**: 修改函数定义行，并添加参数说明。

**🔴 修改前 (Original Code):**

```python
    def run_to_time(self, tEnd, verbose=False):
        """
        Run the simulation to a specified point in time.

        Args:
            tEnd : (float) time in years up to run the model for...
            verbose : (bool) when :code:`True`, output additional debug information (default: :code:`False`).
```

**🟢 修改后 (Modified Code):**

```python
    def run_to_time(self, tEnd, verbose=False, generate_udw_strata=True):
        """
        Run the simulation to a specified point in time.

        Args:
            tEnd : (float) time in years up to run the model for...
            verbose : (bool) when :code:`True`, output additional debug information (default: :code:`False`).
            generate_udw_strata : (bool) when :code:`True`, generate strata output for Underworld coupling (default: :code:`True`).
```

> **解释**: 我们添加了 `generate_udw_strata=True`。这意味着如果你不改动任何调用代码，它默认还是会生成文件（保持原有习惯，安全第一）。只有当你明确说“不要生成”时，它才会听话。

---

## 修改步骤 2：应用控制开关

接下来，我们需要找到真正执行“强制生成文件”的代码块，加上我们的开关判断。

*   **位置**: 第 **846** 行左右
*   **操作**: 在 `if` 判断语句中增加 `and generate_udw_strata` 条件。

**🔴 修改前 (Original Code):**

```python
        # if Underworld coupling is active, force a strata write at the end
        if self.input.udw==1:
            purple = "\033[0;35m"
            endcol = "\033[00m"
```

**🟢 修改后 (Modified Code):**

```python
        # if Underworld coupling is active, force a strata write at the end
        if self.input.udw==1 and generate_udw_strata:
            purple = "\033[0;35m"
            endcol = "\033[00m"
```

> **解释**: 原来的代码只要 `udw==1`（开启了 Underworld 模式）就会强制写入。现在我们加了一个条件：必须 `udw==1` **并且** 我们允许生成 (`generate_udw_strata` 为真) 时，才执行写入操作。

---

## 如何使用新功能？

修改完成后，你可以通过以下方式来控制是否生成文件：

1.  **正常模式 (生成文件)** - *适用于需要地层数据进行热力学计算的情况*
    ```python
    model.run_to_time(tEnd)
    # 或者显式写出
    model.run_to_time(tEnd, generate_udw_strata=True)
    ```

2.  **忽略模式 (不生成文件)** - *适用于只需要地形演化，想节省时间的情况*
    ```python
    model.run_to_time(tEnd, generate_udw_strata=False)
    ```

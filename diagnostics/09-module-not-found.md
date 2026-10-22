# 09 · 明明装了却 ModuleNotFoundError

## 适用

- 多 Python 版本 / conda / venv 共存的机器
- VS Code / PyCharm / Jupyter 内核选错
- Windows 与 Linux 均常见

## 现象

```python
>>> import cv2
ModuleNotFoundError: No module named 'cv2'
```

但：

```bash
$ pip list | grep opencv
opencv-python    4.10.0.84
```

或者 Jupyter 里能导入，命令行不能；反过来也有。

## 原因

1. **`pip` 和 `python` 不是同一个解释器。** 装了 A 解释器，运行用 B。
2. **虚拟环境没激活。** 终端提示符里没有 `(venv)`。
3. **IDE / Jupyter 内核选错。** 内核列表里有多个同名环境。
4. **包名 ≠ 导入名。**
   - `opencv-python` → `cv2`
   - `pillow` → `PIL`
   - `beautifulsoup4` → `bs4`
   - `pyyaml` → `yaml`
   - `scikit-learn` → `sklearn`
   - `python-dateutil` → `dateutil`
5. **装到了 `--user` 目录，但当前进程不读该目录。**
6. **当前目录有同名文件遮蔽。** 比如本地有个 `random.py`，`import random` 导入的是它。

## 命令

先看「我是谁、装到哪」：

```bash
which -a python python3 pip pip3
python -c "import sys; print(sys.executable); print(sys.version)"
python -m pip -V
python -m pip show -f opencv-python | head -30
python -m site
```

Windows：

```powershell
where.exe python
where.exe pip
python -c "import sys; print(sys.executable)"
python -m pip -V
```

判断标准：`sys.executable` 所在目录，必须和 `pip -V` 里 `from ... (python X.Y)` 的路径一致。

### 统一安装方式

```bash
python -m pip install opencv-python
```

永远用 `python -m pip`，不要裸 `pip`。这一条能消掉一半问题。

### 确认导入名

```bash
python -m pip show opencv-python | Select-String -Pattern "Name|Version|Location"
```

去 [PyPI](https://pypi.org/) 搜包名，页面顶部会写常用 import 名；或看包的 `top_level.txt`：

```bash
python -c "import importlib.metadata as m; print(m.distribution('opencv-python').read_text('top_level.txt'))"
```

### 检查同名文件遮蔽

```bash
python -c "import cv2; print(cv2.__file__)"
```

如果打印的是当前目录下的 `cv2.py`，把它改名。

### Jupyter 内核

```bash
python -m ipykernel install --user --name myenv --display-name "Python (myenv)"
jupyter kernelspec list
```

然后在 notebook 里 Kernel → Change kernel 选对的那个。

### VS Code

`Ctrl+Shift+P` → `Python: Select Interpreter` → 选 `sys.executable` 对应的那个。

## 验证

```bash
python -c "import cv2, sys; print(sys.executable); print(cv2.__version__); print(cv2.__file__)"
```

三条同时满足才算修好：

1. `sys.executable` 是你预期的解释器
2. 版本号能打印
3. `__file__` 落在该解释器的 `site-packages` 下

再跑一次 `python -m pip check`，确认没有依赖冲突。

## 还不行时

贴出：

1. `python -c "import sys; print(sys.executable)"`
2. `python -m pip -V`
3. `python -m pip show <包名>`
4. 完整报错
5. 是在终端、Jupyter 还是 IDE 里运行

如果只在 Jupyter 里出错，八成是内核问题，不是包问题。

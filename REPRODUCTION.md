# Arjun 复现笔记

> 本文件记录我在本地复现 [s0md3v/Arjun](https://github.com/s0md3v/Arjun) 的过程、观察和改动。
> 目的：留下可验证的复现痕迹，而不是"我跑过"这种无法证明的说法。

---

## 一、环境

| 项 | 值 |
|---|---|
| 操作系统 | Windows 11 |
| Python | 3.14.7 |
| Git | 2.56.0.windows.1 |
| Arjun | 2.2.7 |
| 安装方式 | `pip install -e .`（editable，改源码立即生效） |

**依赖版本：**

```
requests            2.34.2
dicttoxml           1.7.16
ratelimit           2.2.1
charset_normalizer  3.5.2
urllib3             2.8.0
idna                3.20
certifi             2026.10.23
```

> ⚠️ Arjun 官方未针对 Python 3.14 测试。本次复现验证了在该版本下可以安装并运行。

---

## 二、复现步骤

```cmd
git clone https://github.com/s0md3v/Arjun.git
cd Arjun
py -m venv venv
venv\Scripts\activate.bat
python -m pip install --upgrade pip
pip install -e .
arjun --help
```

> 注意：本项目没有 `requirements.txt`，只有 `setup.py`，因此用 `pip install -e .` 安装。

---

## 三、测试靶场

真实站点不能随意扫描，因此自建一个**答案已知**的靶场来验证工具行为。

`target.py`（Flask）：

```python
from flask import Flask, request

app = Flask(__name__)

@app.route("/")
def index():
    html = "<h1>Welcome</h1>"
    if "debug" in request.args:
        html += "<p>DEBUG MODE</p>" + "A" * 300
    if "admin" in request.args:
        html += "<p>admin panel</p>"
    if "token" in request.args:
        html += "<p>token accepted</p>" + "B" * 150
    return html

app.run(port=5000)
```

**字典 `my.txt`（6 行，其中 3 个真参数、3 个干扰项）：**

```
debug
admin
token
foo
bar
id
```

**运行：**

```cmd
python target.py                                    :: 终端 1
arjun -u http://127.0.0.1:5000/ -w my.txt          :: 终端 2
```

---

## 四、观察到的行为

### 1. 检出结果

```
[✓] parameter detected: debug, based on: body length
[✓] parameter detected: token, based on: body length
[✓] parameter detected: admin, based on: body length
[+] Parameters found: debug, token, admin
```

- 3 个真参数全部检出
- 3 个干扰项（foo / bar / id）**全部正确排除**
- 判据为 `body length`

### 2. 响应长度对照

| 请求 | 长度 |
|---|---|
| 无参数 | 16 |
| `?admin=1` | 34 |
| `?token=1` | 187 |
| `?debug=1` | 333 |
| `?token=1&admin=1` | 205 |
| 三个一起 | 522 |

**结论**：长度是可加的，因此工具可以靠"一次发多个参数、看长度是否变化"来做批量筛选，再逐步二分收窄。

### 3. 输出顺序不稳定

两次运行，`parameter detected` 的顺序不同（`debug, token, admin` / `debug, admin, token`）。

**原因**：`narrower()` 使用 `as_completed`，谁先完成谁先返回，多线程下顺序不确定。

### 4. 请求头是伪装的

```
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:83.0) Gecko/20100101 Firefox/83.0
```

在 Windows 上运行，却伪装成 Linux 上的 Firefox。目的是降低被 WAF 识别的概率。

---

## 五、遇到的问题

### 问题 1：字典只有 1 行时崩溃

```
ZeroDivisionError: division by zero
  File "arjun/__main__.py", line 167, in initialize
```

**根因**：`__main__.py` 第 90–91 行

```python
if len(wordlist) < mem.var['chunks']:
    mem.var['chunks'] = int(len(wordlist)/2)    # 1 行时 = 0
```

第 167 行用 `mem.var['chunks']` 做除数，于是除零。

**临时绕过**：用命令重建字典文件，保证一行一个参数。
**根治方案**：`mem.var['chunks'] = max(1, int(len(wordlist)/2))`

> 这是 Arjun 的一个边界 bug：无法处理行数小于 2 的字典。

---

### 问题 2：变量名写错，却表现为"跳过目标"

在 `requester.py` 里误用了不存在的变量 `r`：

```
[-] Skipped http://127.0.0.1:5000/ due to errors
```

**完整链路：**

```
NameError（r 不存在）
  → 被 requester.py 的 except Exception 捕获
  → return str(e)，错误变成字符串
  → __main__.py 中 isinstance(response_1, str) 成立
  → 返回 'skipped'
  → main() 打印 "Skipped ... due to errors"
```

**教训**：过宽的 `except Exception` 会把"代码错误"伪装成"业务失败"，导致排查困难。

---

### 问题 3：输出前多了一个 `2`

```python
print('2''%s parameter detected: ...' % (res, name, reason))
```

`'2'` 与 `'%s...'` 是相邻字符串字面量，Python 会自动拼接，结果是 `'2%s...'`。

**教训**：相邻字符串会自动拼接，这是 Python 的特性，也是容易踩的坑。

---

## 六、我的改动

| 文件 | 改动 | 理由 |
|---|---|---|
| `arjun/core/colors.py` | 从平台判断中移除 `'win'` | Windows 10+ 已支持 ANSI 转义码，可以启用彩色输出 |

---

## 七、已知局限（本复现未覆盖）

- 只在自建靶场上验证，未在真实授权目标上测试
- 未验证 `--passive`（被动信息收集）功能，该功能依赖外部公共服务
- 未验证 POST / JSON / XML 三种请求方法
- 未验证 `--stable` 模式和限速行为

---

## 八、说明

本仓库为 [s0md3v/Arjun](https://github.com/s0md3v/Arjun) 的 fork，改动仅用于个人学习与复现验证，未对核心扫描逻辑做修改。

原项目采用 AGPL-3.0 许可证。

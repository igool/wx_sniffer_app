
---

## 一、背景说明

目标：
做一个 **视频图片媒体嗅探 GUI 工具**，要求：

1. 提供图形界面（PySide6），一键启动/停止抓包；
2. 使用 **mitmproxy/mitmdump + 自定义脚本 `wx_sniffer_addon.py`** 抓取图片、视频流等资源；
3. 最终形态是 **Windows 下的单文件 EXE**；
4. 最好 **不要求用户安装 Python / conda / mitmproxy**，即“内置运行时”。

---

## 二、环境准备过程

### 1. 开发环境

* 使用 **conda** 创建开发环境（示例）：

```bash
conda create -n video python=3.11 -y
conda activate video
pip install mitmproxy PySide6 pyinstaller
```

* 在该环境下开发：

    * `wx_sniffer_addon.py`（mitmproxy 脚本，负责写入 output 目录）
    * `wx_sniffer_gui.py`（PySide6 GUI）

### 2. CLI 下验证抓包脚本

在命令行中，使用 mitmproxy/mitmdump 单独验证脚本是否正常：

```bash
# 带交互界面（TUI），需要真正的终端
mitmproxy -s wx_sniffer_addon.py

# 纯命令行模式，适合自动化/日志重定向
mitmdump -s wx_sniffer_addon.py
```

验证点：

* mitmproxy 能正常启动，不报错；
* 配置电脑/手机代理指向 127.0.0.1:8080；
* 访问微信/网页后，`WX_SNIFFER_WORKDIR/output` 下可以看到图片、视频等抓包结果。

---

## 三、第一次 GUI + 内嵌 mitmproxy 方案遇到的问题

一开始尝试使用 **mitmproxy Python API + DumpMaster** 在 GUI 进程内部直接跑（in-process）：

```python
opts = options.Options()
master = DumpMaster(opts, with_termlog=False, with_dumper=False)
master.addons.add(Script("wx_sniffer_addon.py"))
await master.run()
```

### 遇到的问题：

1. **`AddonManagerError: addon already exists`**

    * 因为某些版本会自动加载脚本，再 add 一次就报 “already exists”。

2. **`KeyError: 'Unknown options: scripts'`**

    * 新旧 mitmproxy 版本的 Options 字段不一致，有些版本没有 `scripts` 配置项。

3. **打包后 EXE 起不来 / 直接崩溃**

    * API 在不同版本 mitmproxy 上兼容性较差。
    * GUI 和 mitmproxy 事件循环在同一进程内，调度容易出问题。

**结论**：
在 GUI 里直接跑 mitmproxy 的 API，兼容性太差、调试非常痛苦，不如老老实实用 **子进程方式调用 `mitmdump`**。

---

## 四、改为 GUI 调用系统 mitmdump 子进程时的问题

改成 GUI 启动一个子进程：

```bash
mitmdump -s wx_sniffer_addon.py
```

打包前在 conda 环境里是没问题的。但打包后出现：

### 问题 1：`未找到 mitmdump 命令`

日志：

```text
[CMD] mitmdump -s C:\...\wx_sniffer_addon.py
[ERROR] 未找到 mitmdump 命令，请确认已安装 mitmproxy 并加入 PATH。
```

原因：

* EXE 运行时不会自动使用 conda 环境；
* 系统 PATH 里没有 `mitmdump`；
* 所以子进程调用 `mitmdump` 找不到可执行文件。

### 问题 2：依赖 conda，无法做到“零环境”

即便在 GUI 配置里改成绝对路径：

```text
C:\Users\admin\anaconda3\envs\video\Scripts\mitmdump.exe
```

也还是依赖这台机器上必须有这个 conda 环境，无法作为真正的独立发行包给别人用。

---

## 五、方案 C：内置 python_runtime + mitmdump

为实现真正“无外部依赖”的 EXE，最终采用方案 C：

> **在项目目录中自带一个独立的 Python 运行时（python_runtime），里面已经安装好 mitmproxy，然后 GUI 只调用这个内置的 mitmdump.exe。**

### 1. 用 conda 帮忙准备一个独立的 Python 运行时

只用 conda 做“构建工具”，生成一个可复制的 Python 目录：

```bash
# 创建纯 Python 环境目录（不绑定名字，用路径）
conda create -p C:\wx_sniffer_python python=3.11 -y
conda activate C:\wx_sniffer_python

# 在这个 env 里用 pip 安装 mitmproxy
pip install mitmproxy
```

完成后，`C:\wx_sniffer_python` 下有：

```text
C:\wx_sniffer_python\
  ├ python.exe
  ├ Lib\
  ├ Scripts\
  │   ├ mitmdump.exe
  │   └ mitmproxy.exe
  └ ...
```

### 2. 拷贝运行时到项目目录

把整个目录复制到项目根目录，并改名为 `python_runtime`：

```text
C:\wx_sniffer_app\
  ├ wx_sniffer_gui.py / wx_sniffer_gui.exe
  ├ wx_sniffer_addon.py
  └ python_runtime\
      ├ python.exe
      ├ Lib\
      ├ Scripts\
      │   ├ mitmdump.exe
      │   └ mitmproxy.exe
      └ ...
```

以后发给别人用，只需要保证这三个东西在同一目录下。

---

## 六、GUI 侧的关键改动

### 1. 不再使用 `_MEIPASS` 找 python_runtime

之前用 `_MEIPASS` 导致路径变成临时目录（`AppData\Local\Temp\_MEIxxx`），找不到外置 runtime。

改成只根据 **exe 所在目录** 定位：

```python
def app_base_dir() -> Path:
    if getattr(sys, "frozen", False):
        return Path(sys.executable).resolve().parent  # exe 目录
    return Path(__file__).resolve().parent
```

然后：

```python
def get_runtime_root() -> Path:
    return app_base_dir() / "python_runtime"

def get_runtime_mitmdump_exe() -> Path:
    return get_runtime_root() / "Scripts" / "mitmdump.exe"

def resource_path(rel_path: str) -> Path:
    return app_base_dir() / rel_path  # wx_sniffer_addon.py 在 exe 同目录
```

### 2. 子进程改为调用内置 mitmdump.exe

使用 `QThread + subprocess.Popen` 启动 mitmdump，并把 `WX_SNIFFER_WORKDIR` 环境变量传给脚本：

```python
cmd = [
    str(self.mitmdump_exe),
    "--listen-port", str(self.listen_port),
    "-s", self.addon_script_path,
]

env = os.environ.copy()
env["WX_SNIFFER_WORKDIR"] = self.workdir

self._proc = subprocess.Popen(
    cmd,
    cwd=self.workdir,
    env=env,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True,
    encoding="utf-8",
    errors="replace",
    creationflags=subprocess.CREATE_NO_WINDOW,  # 隐藏黑窗口（Windows）
)
```

并在循环中持续把 mitmdump 的输出写到 GUI 日志窗口：

```python
for line in self._proc.stdout:
    line = line.rstrip("\n")
    if line:
        self.log.emit(line)
```

### 3. 支持自定义端口、解决 8080 被占用问题

之前日志出现：

```text
[Errno 10048] HTTP(S) proxy failed to listen on *:8080 ...
```

说明端口被占用。为解决这个问题：

* GUI 增加“监听端口”输入框，默认 8080，可以改成 8081 / 8082 等；
* 启动前检测端口是否占用；
* 若占用，在 Windows 下通过 `netstat` 查到 PID 提示用户。

示例检测代码：

```python
def port_is_free(host: str, port: int) -> bool:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind((host, port))
        return True
    except OSError:
        return False
    finally:
        s.close()
```

查 PID：

```python
def find_listening_pid_windows(port: int) -> Optional[int]:
    out = subprocess.check_output(
        ["cmd", "/c", f"netstat -ano | findstr :{port}"],
        text=True, encoding="utf-8", errors="ignore"
    )
    for line in out.splitlines():
        if "LISTENING" in line.upper():
            m = re.search(r"\sLISTENING\s+(\d+)\s*$", line, re.IGNORECASE)
            if m:
                return int(m.group(1))
    return None
```

GUI 提示用户换端口或手动关闭。

---

## 七、常见问题与排查记录

1. **mitmdump 命令找不到**

    * 日志：`未找到 mitmdump 命令`
    * 原因：EXE 运行时 PATH 无该命令
    * 解决：改为调用 `python_runtime\Scripts\mitmdump.exe` 的绝对路径

2. **mitmproxy 正常退出 code=0，但没有任何输出**

    * 日志只有 `[EXIT] mitmproxy exited. code=0`
    * 原因：用 `python -m mitmproxy.tools.dump` 的方式在某个版本运行时直接退出，没有真正监听
    * 解决：改用已验证可用的 `mitmdump.exe -s script.py` 方式

3. **端口 8080 被占用（Errno 10048）**

    * 日志中出现 `failed to listen on *:8080` / “只能使用一次”
    * 原因：已有其他代理/服务占用 8080
    * 解决：

        * GUI 支持自定义端口；
        * 启动前检测占用，提示 PID；
        * 建议改用 8081 / 8082 等。

4. **开发环境 vs 运行环境**

    * 打包 EXE 使用的是 conda 环境；
    * 运行 EXE 时不再自动使用 conda；
    * 所以所有依赖（mitmdump、python）必须显式内置。

---

## 八、最终交付形态总结

用户拿到的目录结构示例：

```text
wx-sniffer/
  ├ wx_sniffer_gui.exe          # 图形界面入口
  ├ wx_sniffer_addon.py         # 嗅探脚本
  └ python_runtime/             # 内置 Python + mitmproxy 运行时
       ├ python.exe
       ├ Lib\...
       ├ Scripts\
       │    ├ mitmdump.exe
       │    └ mitmproxy.exe
       └ ...
```

使用步骤：

1. 解压目录；
2. 双击 `wx_sniffer_gui.exe`；
3. 选择“工作目录”（默认 `文档/wx-sniffer`）；
4. 如有必要修改“监听端口”（默认 8080，避开占用可用 8081/8082）；
5. 点击“启动抓包”，看到 mitmdump 启动日志；
6. 把电脑/手机代理指向 `127.0.0.1:端口`；
7. 打开微信/网页，媒体资源会写入 `工作目录/output`。

---

# Personal Downloader Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个 Windows 便携单任务下载器，可解析普通 HTTP/HTTPS 链接和公开视频页面，支持清晰度选择、暂停、重启续传、取消及安全清理。

**Architecture:** Tkinter 只负责界面，`Controller` 管理状态机，`YtDlpEngine` 以无 shell 的子进程调用独立 `yt-dlp.exe`，`StateStore` 原子保存当前任务。下载在目标目录内的任务专属临时目录完成，成功后再移动到最终路径；FFmpeg 作为独立工具随便携包分发。

**Tech Stack:** Python 3.12、标准库 `tkinter`/`subprocess`/`unittest`、PyInstaller 6.21.0、yt-dlp 2026.07.04、yt-dlp FFmpeg Windows GPL 自动构建（2026-07-12）。

---

## 实施约束

当前构建解释器：

```powershell
$PY = 'C:\Users\PC\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe'
& $PY --version
```

预期输出包含 `Python 3.12.13`。所有测试命令都从仓库根目录运行，并设置：

```powershell
$env:PYTHONPATH = 'src'
```

每个功能都遵循 RED → GREEN：先写测试并观察预期失败，再写最小实现。不得用真实视频网站替代单元测试；真实网站只用于最终验收。

## 文件结构

```text
src/personal_downloader/
├── __init__.py          # 包版本
├── models.py            # 数据对象和状态枚举
├── state.py             # 任务持久化和安全清理
├── engine.py            # yt-dlp 命令、解析和进程管理
├── controller.py        # 单任务状态机
├── ui.py                # Tkinter 窗口
└── main.py              # 组合依赖和入口
tests/
├── fake_yt_dlp.py       # 可控的测试子进程
├── test_models.py
├── test_state.py
├── test_engine.py
├── test_controller.py
└── test_ui_smoke.py
scripts/
├── fetch-tools.ps1      # 固定来源、校验哈希、提取工具
└── build.ps1            # 测试、PyInstaller、便携 ZIP
tools.lock.json          # 第三方版本、URL、SHA-256
pyproject.toml
README.md
LICENSE
LICENSES/third-party-notices.txt
```

### Task 1: 项目骨架和领域模型

**Files:**
- Create: `pyproject.toml`
- Create: `src/personal_downloader/__init__.py`
- Create: `src/personal_downloader/models.py`
- Create: `tests/test_models.py`

- [ ] **Step 1: 写模型失败测试**

测试必须覆盖：状态枚举值、URL 仅允许 HTTP/HTTPS、Windows 文件名清理、解析结果清晰度按高度去重并降序。

```python
# tests/test_models.py
import unittest

from personal_downloader.models import (
    FormatOption, MediaInfo, TaskStatus, clean_filename, validate_url,
)


class ModelsTest(unittest.TestCase):
    def test_validate_url_accepts_http_and_https(self):
        self.assertEqual(validate_url("https://example.com/a"), "https://example.com/a")
        self.assertEqual(validate_url("http://example.com/a"), "http://example.com/a")

    def test_validate_url_rejects_other_schemes(self):
        with self.assertRaisesRegex(ValueError, "HTTP/HTTPS"):
            validate_url("file:///C:/secret.txt")

    def test_clean_filename_replaces_windows_reserved_characters(self):
        self.assertEqual(clean_filename('a<b>:c"d/e\\f|g?h*.mp4'), "a_b__c_d_e_f_g_h_.mp4")

    def test_media_info_deduplicates_heights_in_descending_order(self):
        info = MediaInfo(
            url="https://example.com/watch/1", title="demo", site="example",
            duration=10, suggested_filename="demo.mp4", is_video=True,
            formats=(
                FormatOption("720p-a", "a", 720),
                FormatOption("1080p", "b", 1080),
                FormatOption("720p-b", "c", 720),
            ),
        )
        self.assertEqual([item.height for item in info.quality_options()], [1080, 720])

    def test_status_values_are_stable_for_json(self):
        self.assertEqual(TaskStatus.PAUSED.value, "paused")


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: 运行测试并确认因包不存在而失败**

```powershell
& $PY -m unittest tests.test_models -v
```

预期：`ModuleNotFoundError: No module named 'personal_downloader'`。

- [ ] **Step 3: 添加最小模型实现**

`models.py` 使用冻结 dataclass；`DownloadTask` 包含 `task_id/url/save_dir/filename/format_selector/temp_dir/status/media`，并提供 JSON 兼容的 `to_dict()` 与 `from_dict()`。文件名清理还要拒绝空值、`.`、`..` 和 Windows 设备名 `CON/PRN/AUX/NUL/COM1..9/LPT1..9`。

核心公开接口固定为：

```python
class TaskStatus(str, Enum):
    IDLE = "idle"
    ANALYZING = "analyzing"
    READY = "ready"
    DOWNLOADING = "downloading"
    MERGING = "merging"
    PAUSED = "paused"
    FAILED = "failed"
    COMPLETED = "completed"
    CANCELLED = "cancelled"

@dataclass(frozen=True)
class FormatOption:
    label: str
    selector: str
    height: int | None

@dataclass(frozen=True)
class MediaInfo:
    url: str
    title: str
    site: str
    duration: int | None
    suggested_filename: str
    is_video: bool
    filesize: int | None = None
    formats: tuple[FormatOption, ...] = ()

@dataclass(frozen=True)
class ProgressEvent:
    status: str
    downloaded_bytes: int | None = None
    total_bytes: int | None = None
    speed: float | None = None
    eta: int | None = None

@dataclass(frozen=True)
class DownloadTask:
    task_id: str
    url: str
    save_dir: str
    filename: str
    format_selector: str
    temp_dir: str
    status: TaskStatus
    media: MediaInfo
```

- [ ] **Step 4: 运行模型测试**

```powershell
& $PY -m unittest tests.test_models -v
```

预期：5 项测试通过。

- [ ] **Step 5: 提交**

```powershell
git add pyproject.toml src/personal_downloader tests/test_models.py
git commit -m "feat: add downloader domain models"
```

### Task 2: 原子状态存储和安全任务清理

**Files:**
- Create: `src/personal_downloader/state.py`
- Create: `tests/test_state.py`

- [ ] **Step 1: 写状态存储失败测试**

使用 `tempfile.TemporaryDirectory()` 创建真实目录，覆盖保存/载入、损坏 JSON 备份、清除索引，以及拒绝删除任务根目录之外的路径。

```python
# tests/test_state.py
import tempfile
import unittest
from pathlib import Path

from personal_downloader.models import DownloadTask, MediaInfo, TaskStatus
from personal_downloader.state import StateStore, UnsafeTaskPathError


def make_task(root: Path, temp_dir: Path) -> DownloadTask:
    media = MediaInfo("https://example.com/a", "a", "example", None, "a.bin", False)
    return DownloadTask("abc", media.url, str(root), "a.bin", "best", str(temp_dir), TaskStatus.PAUSED, media)


class StateStoreTest(unittest.TestCase):
    def test_round_trip_and_clear(self):
        with tempfile.TemporaryDirectory() as raw:
            root = Path(raw)
            task_dir = root / ".personal-downloader-tasks" / "abc"
            task_dir.mkdir(parents=True)
            store = StateStore(root / "state.json")
            store.save(make_task(root, task_dir))
            self.assertEqual(store.load().task_id, "abc")
            store.clear()
            self.assertIsNone(store.load())

    def test_corrupt_state_is_backed_up(self):
        with tempfile.TemporaryDirectory() as raw:
            path = Path(raw) / "state.json"
            path.write_text("{", encoding="utf-8")
            store = StateStore(path)
            self.assertIsNone(store.load())
            self.assertEqual(len(list(path.parent.glob("state.corrupt-*.json"))), 1)

    def test_cleanup_rejects_path_outside_task_root(self):
        with tempfile.TemporaryDirectory() as raw:
            root = Path(raw)
            outside = root / "keep"
            outside.mkdir()
            store = StateStore(root / "state.json")
            with self.assertRaises(UnsafeTaskPathError):
                store.cleanup_task(make_task(root, outside))
            self.assertTrue(outside.exists())


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: 运行测试并观察缺少模块失败**

```powershell
& $PY -m unittest tests.test_state -v
```

预期：导入 `personal_downloader.state` 失败。

- [ ] **Step 3: 实现 StateStore**

精确规则：`save()` 写入同目录的 `.tmp` 后调用 `Path.replace()`；`load()` 只接受版本 `1`，JSON/字段错误都重命名为 `state.corrupt-YYYYMMDD-HHMMSS.json`；`cleanup_task()` 解析绝对路径后，必须满足 `temp.parent == save_dir/.personal-downloader-tasks` 且 `temp.name == task_id`，再调用 `shutil.rmtree(temp)`。

公开接口：

```python
class UnsafeTaskPathError(RuntimeError):
    pass

class StateStore:
    def __init__(self, path: Path): ...
    def save(self, task: DownloadTask) -> None: ...
    def load(self) -> DownloadTask | None: ...
    def clear(self) -> None: ...
    def cleanup_task(self, task: DownloadTask) -> None: ...
```

- [ ] **Step 4: 运行状态测试及当前全套测试**

```powershell
& $PY -m unittest discover -s tests -v
```

预期：8 项测试通过。

- [ ] **Step 5: 提交**

```powershell
git add src/personal_downloader/state.py tests/test_state.py
git commit -m "feat: persist resumable task state"
```

### Task 3: 资源解析和清晰度映射

**Files:**
- Create: `src/personal_downloader/engine.py`
- Create: `tests/test_engine.py`

- [ ] **Step 1: 写解析失败测试**

测试用固定 JSON，包含重复 720p、纯音频和 1080p。断言解析结果只暴露 1080p/720p，选择器分别为 `bv*[height<=1080]+ba/b[height<=1080]` 和 `bv*[height<=720]+ba/b[height<=720]`。普通直链 JSON 产生 `is_video=False`、文件大小和安全文件名。

```python
class EngineParsingTest(unittest.TestCase):
    def test_parse_video_formats(self):
        raw = json.dumps({
            "webpage_url": "https://video.example/watch/1", "title": "demo", "extractor_key": "Example",
            "duration": 61, "ext": "mp4", "formats": [
                {"format_id": "a", "height": None, "vcodec": "none"},
                {"format_id": "v720a", "height": 720, "vcodec": "avc1"},
                {"format_id": "v720b", "height": 720, "vcodec": "vp9"},
                {"format_id": "v1080", "height": 1080, "vcodec": "avc1"},
            ],
        })
        info = parse_probe_json(raw, "https://video.example/watch/1")
        self.assertTrue(info.is_video)
        self.assertEqual([f.height for f in info.quality_options()], [1080, 720])
        self.assertEqual(info.quality_options()[0].selector, "bv*[height<=1080]+ba/b[height<=1080]")

    def test_probe_arguments_disable_config_and_playlists(self):
        args = build_probe_args(Path("tools/yt-dlp.exe"), "https://example.com/a")
        self.assertEqual(args[-1], "https://example.com/a")
        self.assertIn("--ignore-config", args)
        self.assertIn("--no-playlist", args)
        self.assertIn("--dump-single-json", args)
```

- [ ] **Step 2: 运行指定测试并确认接口缺失**

```powershell
& $PY -m unittest tests.test_engine.EngineParsingTest -v
```

预期：`parse_probe_json` 或 `build_probe_args` 导入失败。

- [ ] **Step 3: 实现纯函数解析层**

`parse_probe_json(raw, requested_url)` 必须拒绝 `_type` 为 `playlist` 的结果，标题回退到 `id`，扩展名回退到 `bin`；视频判定为至少一个格式的 `vcodec` 不是 `none`。`build_probe_args()` 返回字符串列表，不使用命令行拼接。

公开接口：

```python
class EngineError(RuntimeError): ...
class MissingToolError(EngineError): ...
class UnsupportedMediaError(EngineError): ...

@dataclass(frozen=True)
class EnginePaths:
    yt_dlp: Path
    ffmpeg_dir: Path

def build_probe_args(yt_dlp: Path, url: str) -> list[str]: ...
def parse_probe_json(raw: str, requested_url: str) -> MediaInfo: ...
```

- [ ] **Step 4: 运行引擎解析测试及全套测试**

```powershell
& $PY -m unittest discover -s tests -v
```

预期：所有测试通过。

- [ ] **Step 5: 提交**

```powershell
git add src/personal_downloader/engine.py tests/test_engine.py
git commit -m "feat: parse downloadable media metadata"
```

### Task 4: 下载命令、进度和可停止子进程

**Files:**
- Modify: `src/personal_downloader/engine.py`
- Modify: `tests/test_engine.py`
- Create: `tests/fake_yt_dlp.py`

- [ ] **Step 1: 写命令与进度失败测试**

新增断言：下载命令包含 `--continue`、`--no-overwrites`、5 次重试、4 路分片、FFmpeg 路径、任务临时路径、格式选择器、JSON 进度前缀和最终路径前缀。进度解析只接受 `PD_PROGRESS:` 开头的 JSON；其他日志返回 `None`。

假引擎支持三种 URL：`probe` 输出固定资源 JSON；`download` 输出两条进度和 `PD_RESULT:` 后退出 0；`wait` 创建 `.part` 后持续等待，以测试 `stop()` 能终止进程并保留文件。

```python
def test_parse_progress_line(self):
    event = parse_progress_line('PD_PROGRESS:{"status":"downloading","downloaded_bytes":5,"total_bytes":10,"speed":2,"eta":3}')
    self.assertEqual(event.downloaded_bytes, 5)
    self.assertEqual(event.total_bytes, 10)
    self.assertIsNone(parse_progress_line("ordinary log"))

def test_download_args_are_resumable_and_bounded(self):
    args = build_download_args(self.paths, self.task)
    self.assertIn("--continue", args)
    self.assertEqual(args[args.index("--concurrent-fragments") + 1], "4")
    self.assertEqual(args[args.index("-f") + 1], self.task.format_selector)
```

- [ ] **Step 2: 运行新增测试并确认失败**

```powershell
& $PY -m unittest tests.test_engine -v
```

预期：缺少 `parse_progress_line`、`build_download_args` 或 `YtDlpEngine`。

- [ ] **Step 3: 实现进程层**

`YtDlpEngine` 的 `probe()` 和 `download()` 都用 `subprocess.Popen(..., shell=False, stdout=PIPE, stderr=STDOUT, text=True, encoding="utf-8", errors="replace")`。Windows 下添加 `CREATE_NO_WINDOW`。当前进程保存在锁保护字段中；`stop()` 先 `terminate()`，5 秒未退出再 `kill()`。非零退出码转换为精简中文错误，但调用方正在暂停或取消时允许忽略该错误。

下载命令使用：

```text
--ignore-config --no-playlist --continue --no-overwrites
--retries 5 --fragment-retries 5 --concurrent-fragments 4 --newline
--progress-template download:PD_PROGRESS:%(progress)j
--progress-template postprocess:PD_POSTPROCESS:%(progress)j
--print after_move:PD_RESULT:%(filepath)j
--ffmpeg-location <tools目录> -P <任务临时目录>
-o result.%(ext)s -f <选择器> <URL>
```

公开接口：

```python
def build_download_args(paths: EnginePaths, task: DownloadTask) -> list[str]: ...
def parse_progress_line(line: str) -> ProgressEvent | None: ...

class YtDlpEngine:
    def __init__(self, paths: EnginePaths): ...
    def validate_probe_tools(self) -> None: ...
    def validate_download_tools(self) -> None: ...
    def probe(self, url: str) -> MediaInfo: ...
    def download(self, task: DownloadTask, on_progress: Callable[[ProgressEvent], None]) -> Path: ...
    def stop(self) -> None: ...
```

- [ ] **Step 4: 运行引擎和全套测试**

```powershell
& $PY -m unittest tests.test_engine -v
& $PY -m unittest discover -s tests -v
```

预期：全部通过，假引擎留下的 `.part` 文件仍存在。

- [ ] **Step 5: 提交**

```powershell
git add src/personal_downloader/engine.py tests/test_engine.py tests/fake_yt_dlp.py
git commit -m "feat: run resumable yt-dlp downloads"
```

### Task 5: 单任务控制器状态机

**Files:**
- Create: `src/personal_downloader/controller.py`
- Create: `tests/test_controller.py`

- [ ] **Step 1: 写控制器失败测试**

用同步 `run_async=lambda fn: fn()` 和假 `Engine`/`StateStore` 覆盖：无效 URL、解析成功、开始创建专属目录、完成后移动结果、取消安全清理、目标存在拒绝覆盖、保存目录不可写、引擎失败进入 `FAILED`。暂停/继续使用 `DeferredRunner`：`run_async()` 只把函数加入队列，测试显式调用 `run_next()`，从而在下载函数真正执行前观察 `DOWNLOADING` 并暂停。

关键验收测试：

```python
def test_download_moves_completed_file_and_clears_state(self):
    self.controller.analyze("https://example.com/watch/1")
    self.controller.start(self.root, "demo.mp4", "best")
    self.assertEqual(self.controller.status, TaskStatus.COMPLETED)
    self.assertEqual((self.root / "demo.mp4").read_bytes(), b"done")
    self.assertTrue(self.store.cleared)

def test_pause_keeps_task_and_resume_reuses_it(self):
    runner = DeferredRunner()
    controller = self.make_controller(run_async=runner.submit)
    controller.analyze("https://example.com/watch/1")
    runner.run_next()  # 完成解析
    controller.start(self.root, "demo.mp4", "best")
    task_id = controller.task.task_id
    self.assertEqual(controller.status, TaskStatus.DOWNLOADING)
    controller.pause()
    self.assertEqual(controller.status, TaskStatus.PAUSED)
    self.assertEqual(self.store.saved.status, TaskStatus.PAUSED)
    controller.resume()
    self.assertEqual(controller.task.task_id, task_id)
    self.assertEqual(controller.status, TaskStatus.DOWNLOADING)

def test_cancel_only_uses_store_safe_cleanup(self):
    self.controller.cancel()
    self.assertEqual(self.store.cleaned_task_id, self.controller.last_cancelled_task_id)
```

- [ ] **Step 2: 运行测试并确认缺少控制器**

```powershell
& $PY -m unittest tests.test_controller -v
```

预期：导入 `personal_downloader.controller` 失败。

- [ ] **Step 3: 实现 Controller**

构造参数固定为 `engine/store/run_async/on_status/on_media/on_progress/on_error`。默认 `run_async` 创建 daemon 线程；测试注入同步或延迟执行器。所有公开动作先锁定状态再启动后台函数。开始前要求保存目录存在且可写：在任务根目录内创建并删除一个零字节探针文件。下载完成时要求返回路径存在、最终目标不存在，并用 `Path.replace()` 移动；暂停/取消先改变状态再调用 `engine.stop()`，避免进程退出回调覆盖用户状态。

公开接口：

```python
class Controller:
    @property
    def status(self) -> TaskStatus: ...
    @property
    def task(self) -> DownloadTask | None: ...
    def restore(self) -> DownloadTask | None: ...
    def analyze(self, url: str) -> None: ...
    def start(self, save_dir: Path, filename: str, format_selector: str) -> None: ...
    def pause(self) -> None: ...
    def resume(self) -> None: ...
    def cancel(self) -> None: ...
    def close(self) -> None: ...
```

- [ ] **Step 4: 运行控制器测试和全套测试**

```powershell
& $PY -m unittest tests.test_controller -v
& $PY -m unittest discover -s tests -v
```

预期：全部通过。

- [ ] **Step 5: 提交**

```powershell
git add src/personal_downloader/controller.py tests/test_controller.py
git commit -m "feat: coordinate single download lifecycle"
```

### Task 6: Tkinter 单窗口

**Files:**
- Create: `src/personal_downloader/ui.py`
- Create: `tests/test_ui_smoke.py`

- [ ] **Step 1: 写 UI 冒烟失败测试**

测试在 `tkinter.TclError` 时仅跳过无桌面环境；Windows 桌面环境必须真正构造并销毁窗口。断言标题、初始“解析”可用、“开始/暂停/取消”禁用，以及 `READY` 状态使“开始”可用。

```python
class UiSmokeTest(unittest.TestCase):
    def test_initial_and_ready_button_states(self):
        try:
            root = tk.Tk()
        except tk.TclError as exc:
            self.skipTest(str(exc))
        root.withdraw()
        try:
            ui = DownloaderWindow(root, FakeController())
            self.assertEqual(root.title(), "个人下载器")
            self.assertEqual(str(ui.analyze_button["state"]), "normal")
            self.assertEqual(str(ui.start_button["state"]), "disabled")
            ui.apply_status(TaskStatus.READY)
            self.assertEqual(str(ui.start_button["state"]), "normal")
        finally:
            root.destroy()
```

- [ ] **Step 2: 运行测试并确认缺少 UI**

```powershell
& $PY -m unittest tests.test_ui_smoke -v
```

预期：导入 `personal_downloader.ui` 失败。

- [ ] **Step 3: 实现窗口**

窗口只包含规格批准的控件。后台回调不得直接操作 Tkinter：回调把 `(kind, payload)` 放入 `queue.Queue`，窗口每 100ms 用 `root.after` 排空队列。质量下拉的显示值映射到 `FormatOption.selector`；普通文件隐藏质量行。关闭协议调用 `controller.close()` 后销毁窗口。

状态按钮矩阵：

| 状态 | 解析 | 开始 | 暂停/继续 | 取消 |
|---|---:|---:|---:|---:|
| 空闲/完成 | 开 | 关 | 关 | 关 |
| 解析中 | 关 | 关 | 关 | 开 |
| 可下载 | 开 | 开 | 关 | 开 |
| 下载中/合并中 | 关 | 关 | 开（暂停） | 开 |
| 已暂停 | 关 | 关 | 开（继续） | 开 |
| 失败（保留任务） | 关 | 关 | 开（继续） | 开 |

格式化函数分别处理字节、速度、时长，`None` 显示 `--`。错误通过 `messagebox.showerror("下载失败", message)`。

- [ ] **Step 4: 运行 UI 冒烟和全套测试**

```powershell
& $PY -m unittest tests.test_ui_smoke -v
& $PY -m unittest discover -s tests -v
```

预期：桌面环境下 UI 测试通过；无桌面环境时仅该项明确 `skipped`，其余全部通过。

- [ ] **Step 5: 提交**

```powershell
git add src/personal_downloader/ui.py tests/test_ui_smoke.py
git commit -m "feat: add downloader desktop interface"
```

### Task 7: 程序入口、恢复和许可证

**Files:**
- Create: `src/personal_downloader/main.py`
- Create: `LICENSE`
- Create: `LICENSES/third-party-notices.txt`
- Create: `README.md`

- [ ] **Step 1: 写入口组合测试**

在 `test_ui_smoke.py` 增加对 `application_paths(executable_dir, local_app_data)` 的纯函数测试，断言工具位于 `<exe>/tools`，状态位于 `%LOCALAPPDATA%/PersonalDownloader/state.json`。

- [ ] **Step 2: 运行测试并观察入口接口缺失**

```powershell
& $PY -m unittest tests.test_ui_smoke -v
```

预期：`application_paths` 导入失败。

- [ ] **Step 3: 实现入口和文档**

`main()` 创建 `EnginePaths`、`StateStore`、`Controller` 和 `DownloaderWindow`。若存在未完成任务，恢复其 URL、目录、文件名和格式，并显示“已暂停”；不自动开始网络请求。开发模式以 `Path(__file__).parents[2]` 为程序目录，冻结模式以 `Path(sys.executable).parent` 为程序目录。

README 精确说明：支持范围、便携启动、普通链接/视频页面使用方法、暂停与恢复、已知限制、不支持 DRM，以及源码测试命令。MIT `LICENSE` 使用 2026 和仓库所有者 `changbaizhou`。第三方声明列出 yt-dlp Unlicense/第三方许可和 FFmpeg GPL 构建来源，提示重新分发时保留随包许可材料。

- [ ] **Step 4: 运行全套测试**

```powershell
& $PY -m unittest discover -s tests -v
```

预期：全部通过或仅无桌面环境 UI 项跳过。

- [ ] **Step 5: 提交**

```powershell
git add src/personal_downloader/main.py README.md LICENSE LICENSES tests/test_ui_smoke.py
git commit -m "docs: add runnable application entrypoint"
```

### Task 8: 固定并校验第三方工具

**Files:**
- Create: `tools.lock.json`
- Create: `scripts/fetch-tools.ps1`
- Modify: `.gitignore`

- [ ] **Step 1: 添加锁文件数据测试**

在 `tests/test_tools_lock.py` 读取 JSON，断言每个工具都有 HTTPS URL 和 64 位小写 SHA-256，且 yt-dlp 版本为 `2026.07.04`。

锁文件使用以下已核验数据：

```json
{
  "yt_dlp": {
    "version": "2026.07.04",
    "url": "https://github.com/yt-dlp/yt-dlp/releases/download/2026.07.04/yt-dlp.exe",
    "sha256": "52fe3c26dcf71fbdc85b528589020bb0b8e383155cfa81b64dd447bbe35e24b8"
  },
  "ffmpeg": {
    "build": "2026-07-12-win64-gpl",
    "url": "https://github.com/yt-dlp/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-win64-gpl.zip",
    "sha256": "e887d4c7b3ef08bd3734f56b72afc2ff734cc53745be0a655fa5719bd9aac5cf"
  }
}
```

- [ ] **Step 2: 运行锁文件测试并确认失败**

```powershell
& $PY -m unittest tests.test_tools_lock -v
```

预期：`tools.lock.json` 不存在。

- [ ] **Step 3: 实现下载脚本**

`fetch-tools.ps1` 使用 `Invoke-WebRequest` 下载到 `build/cache`，通过 `Get-FileHash -Algorithm SHA256` 与锁文件逐字比较；不匹配立即删除下载并退出 1。FFmpeg ZIP 解压到临时目录，用递归精确文件名查找复制 `ffmpeg.exe`、`ffprobe.exe` 和构建内的许可文件到 `tools/`/`LICENSES/`。脚本不得下载或复制 `ffplay.exe`。`tools/`、`build/`、`dist/` 加入 `.gitignore`。

- [ ] **Step 4: 运行锁文件测试和脚本**

```powershell
& $PY -m unittest tests.test_tools_lock -v
powershell -ExecutionPolicy Bypass -File scripts/fetch-tools.ps1
& tools/yt-dlp.exe --version
& tools/ffmpeg.exe -version
```

预期：测试通过；yt-dlp 输出 `2026.07.04`；FFmpeg 输出版本且退出 0。若 FFmpeg 的移动 `latest` 资产已变化，哈希校验必须失败；此时从官方 `checksums.sha256` 审核新构建日期和哈希后单独更新锁文件，不能跳过校验。

- [ ] **Step 5: 提交**

```powershell
git add tools.lock.json scripts/fetch-tools.ps1 .gitignore tests/test_tools_lock.py
git commit -m "build: pin external downloader tools"
```

### Task 9: 真实直链集成测试和便携构建

**Files:**
- Create: `tests/test_direct_download_integration.py`
- Create: `requirements-build.txt`
- Create: `scripts/build.ps1`

- [ ] **Step 1: 写本地 HTTP 集成测试**

测试启动标准库 `ThreadingHTTPServer`，提供固定 64 KiB 文件和 Range 支持；通过真实 `tools/yt-dlp.exe` 下载，中途停止后再次执行，最终比较 SHA-256。若工具未下载，使用 `skipUnless(Path("tools/yt-dlp.exe").exists(), ...)` 明确跳过。

- [ ] **Step 2: 运行集成测试验证跨进程续传**

```powershell
& $PY -m unittest tests.test_direct_download_integration -v
```

这是已有单元测试行为的端到端验收，不新增生产接口，因此允许首次即通过。预期：真实 `yt-dlp.exe` 在本地 HTTP 服务上完成中断与续传，最终哈希匹配；不得接受“工具不存在导致跳过”作为通过证据，先运行 Task 8 脚本准备工具。若失败，先用 `systematic-debugging` 定位原因，再写一个能复现该缺陷的最小失败测试后修改生产代码。

- [ ] **Step 3: 完成集成夹具并添加构建脚本**

`requirements-build.txt` 固定 `pyinstaller==6.21.0`。`build.ps1` 依次：运行全套测试、确认三个工具存在、安装固定 PyInstaller、以 `--onefile --windowed --name app --paths src` 构建入口、创建 `dist/PersonalDownloader/tools` 和 `LICENSES`、复制工具/README/许可证、压缩为 `dist/PersonalDownloader-windows-x64.zip`。任何命令非零立即退出。

- [ ] **Step 4: 执行新环境验证**

```powershell
& $PY -m unittest discover -s tests -v
powershell -ExecutionPolicy Bypass -File scripts/build.ps1
& dist/PersonalDownloader/app.exe
```

预期：自动化测试零失败；ZIP 存在；`app.exe` 打开“个人下载器”窗口。手动关闭后进程正常退出。

- [ ] **Step 5: 公开视频验收**

使用 yt-dlp 官方当前测试视频或用户有权下载的公开短视频页面：解析标题和清晰度，下载最佳画质，确认 FFmpeg 合并完成；再下载一次并在中途暂停、关闭、重启、继续。记录测试日期、页面类型、yt-dlp/FFmpeg 版本和结果到 `docs/acceptance/2026-07-13.md`。若因网站、地区或登录限制失败，原样记录错误，并换一个公开测试页面验证应用流程。

- [ ] **Step 6: 提交**

```powershell
git add requirements-build.txt scripts/build.ps1 tests/test_direct_download_integration.py docs/acceptance/2026-07-13.md
git commit -m "build: package verified Windows downloader"
```

### Task 10: 最终验证和推送

**Files:**
- Modify only if verification exposes a tested defect.

- [ ] **Step 1: 运行完整验证**

```powershell
& $PY -m unittest discover -s tests -v
git diff --check
git status --short
Get-FileHash dist/PersonalDownloader-windows-x64.zip -Algorithm SHA256
```

正确结果：测试零失败；`git diff --check` 无输出；工作树干净；输出 ZIP 的 SHA-256。

- [ ] **Step 2: 逐项对照设计成功标准**

核对直链续传、公开视频下载、暂停重启继续、安全取消、便携启动五项验收证据。没有证据的项目标记为未验证，不得写“完成”。

- [ ] **Step 3: 推送 main**

```powershell
git push origin main
```

预期：`main -> main`，不使用 `--force`。

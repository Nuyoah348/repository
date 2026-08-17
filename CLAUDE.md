# 纯文本驱动数字人 MVP：Claude Code 开发任务书

> 使用方式：将本文件放到项目根目录，建议命名为 `CLAUDE.md`。  
> 然后在 Claude Code 中先让它阅读本文件，再严格按照阶段顺序开发。  
> 不要一次性完成所有阶段，每完成一个阶段都必须先运行测试并等待确认。

---

# 一、项目目标

开发一款“纯文本驱动数字人 MVP”。

用户在网页中输入一段文字，系统将文字转换成语音，再驱动指定数字人生成口型同步视频，最后在网页中播放和下载 MP4 视频。

## 核心流程

```text
用户输入文字
    ↓
文本预处理
    ↓
TTS 生成语音
    ↓
数字人模型生成口型视频
    ↓
FFmpeg 音视频后处理
    ↓
前端播放和下载 MP4
```

---

# 二、第一版项目边界

第一版只完成最小闭环。

## 第一版必须实现

- 用户输入文字。
- 用户选择数字人。
- 用户选择声音。
- 用户选择语速。
- 用户选择视频分辨率。
- 后端创建生成任务。
- 后台异步执行任务。
- 前端查看任务状态。
- TTS 生成 WAV 音频。
- 数字人模型生成视频。
- FFmpeg 输出浏览器可播放的 MP4。
- 前端播放视频。
- 前端下载视频。
- 失败时显示明确错误原因。
- 保存最近任务记录。
- 支持删除任务。
- 支持失败任务重试。

## 第一版暂时不实现

- 不接入 RAG。
- 不接入大模型问答。
- 不做语音输入。
- 不做实时对话。
- 不做 WebRTC。
- 不做实时流式数字人。
- 不做模型训练。
- 不做多用户高并发。
- 不做复杂情绪控制。
- 不做全身动作生成。
- 不做多人数字人管理后台。

---

# 三、开发与部署策略

## 本地开发环境

本地电脑使用 RTX 4060。

本地主要负责：

- Vue 前端开发。
- FastAPI 后端开发。
- 数据库开发。
- 任务状态管理。
- 文件管理。
- FFmpeg 调用。
- Mock 模型服务。
- 接口联调。
- 单元测试。
- Git 代码管理。

本地开发阶段不要求下载真实大模型。

## AutoDL 算力云

AutoDL 主要负责：

- 安装 CUDA 和 PyTorch 环境。
- 下载 CosyVoice 或 GPT-SoVITS。
- 下载 Wav2Lip。
- 运行真实模型。
- 完成完整流水线联调。
- 生成正式演示视频。
- 后续做模型优化或微调。

## 推荐 GPU

```text
初次调试：RTX 3090 24GB
正式演示：RTX 4090 24GB
```

---

# 四、技术栈

## 前端

- Vue 3
- Vite
- TypeScript
- Element Plus
- Axios

## 后端

- Python 3.10
- FastAPI
- Uvicorn
- Pydantic
- SQLAlchemy
- SQLite
- FastAPI BackgroundTasks

## 后续异步任务升级

第一版先使用：

```text
FastAPI BackgroundTasks
```

后续可以升级为：

```text
Redis + Celery
```

## 模型

TTS：

- 优先适配 CosyVoice。
- 预留 GPT-SoVITS 适配器。
- 本地开发使用 MockTTSAdapter。

数字人：

- 第一版适配 Wav2Lip。
- 本地开发使用 MockAvatarAdapter。

## 音视频处理

- FFmpeg
- 输出 H.264 编码 MP4
- 音频使用 AAC
- 像素格式使用 yuv420p
- 默认帧率 25fps

## 工程工具

- Git
- `.env`
- Python logging
- Docker Compose
- pytest

---

# 五、架构设计原则

## 1. 前后端分离

前端只负责：

- 输入参数。
- 创建任务。
- 查询任务状态。
- 播放和下载视频。

后端负责：

- 参数校验。
- 任务创建。
- 状态管理。
- 模型调用。
- 文件管理。
- 异常处理。

## 2. 模型适配器模式

业务代码不能直接依赖某个模型。

TTS 必须定义统一接口：

```python
generate(
    text: str,
    output_path: str,
    voice_id: str,
    speed: float
) -> str
```

数字人必须定义统一接口：

```python
generate(
    audio_path: str,
    avatar_path: str,
    output_path: str
) -> str
```

本地开发使用：

```text
MockTTSAdapter
MockAvatarAdapter
```

AutoDL 部署使用：

```text
CosyVoiceAdapter
Wav2LipAdapter
```

切换模型时，不允许修改 PipelineService 的业务逻辑，只允许通过配置切换 Adapter。

## 3. 分层设计

后端必须分为：

```text
api
schemas
models
repositories
services
workers
utils
```

## 4. 模型常驻

真实模型必须在服务启动时加载。

禁止每次任务都重新加载模型。

## 5. 路径配置化

禁止在代码中写死绝对路径。

所有模型路径、存储路径和服务配置都必须从 `.env` 读取。

---

# 六、项目目录结构

请严格按照下面的目录结构创建项目。

```text
digital-human-mvp/
├── CLAUDE.md
├── README.md
├── .gitignore
├── .env.example
├── docker-compose.yml
│
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── main.ts
│   │   ├── App.vue
│   │   ├── api/
│   │   │   └── task.ts
│   │   ├── types/
│   │   │   └── task.ts
│   │   ├── views/
│   │   │   └── GenerateView.vue
│   │   ├── components/
│   │   │   ├── TextInputPanel.vue
│   │   │   ├── AvatarSelector.vue
│   │   │   ├── VoiceSelector.vue
│   │   │   ├── TaskProgress.vue
│   │   │   ├── VideoResult.vue
│   │   │   └── RecentTasks.vue
│   │   └── styles/
│   │       └── global.css
│   └── public/
│
├── backend/
│   ├── requirements.txt
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── logging_config.py
│   │   │
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── router.py
│   │   │   ├── task_api.py
│   │   │   ├── voice_api.py
│   │   │   └── avatar_api.py
│   │   │
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   ├── task_schema.py
│   │   │   ├── voice_schema.py
│   │   │   └── avatar_schema.py
│   │   │
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   └── task_model.py
│   │   │
│   │   ├── repositories/
│   │   │   ├── __init__.py
│   │   │   └── task_repository.py
│   │   │
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── text_service.py
│   │   │   ├── tts_service.py
│   │   │   ├── avatar_service.py
│   │   │   ├── ffmpeg_service.py
│   │   │   ├── pipeline_service.py
│   │   │   └── storage_service.py
│   │   │
│   │   ├── workers/
│   │   │   ├── __init__.py
│   │   │   └── task_worker.py
│   │   │
│   │   └── utils/
│   │       ├── __init__.py
│   │       ├── file_utils.py
│   │       └── id_utils.py
│   │
│   └── tests/
│       ├── test_task_api.py
│       ├── test_text_service.py
│       ├── test_pipeline_service.py
│       └── test_storage_service.py
│
├── model_services/
│   ├── tts/
│   │   ├── __init__.py
│   │   ├── adapter.py
│   │   ├── mock_tts.py
│   │   ├── cosyvoice_adapter.py
│   │   └── gpt_sovits_adapter.py
│   │
│   └── avatar/
│       ├── __init__.py
│       ├── adapter.py
│       ├── mock_avatar.py
│       └── wav2lip_adapter.py
│
├── assets/
│   ├── avatars/
│   │   └── avatar_01.png
│   ├── voices/
│   └── demo/
│       ├── demo.wav
│       └── demo.mp4
│
├── storage/
│   ├── tasks/
│   ├── audio/
│   ├── video/
│   └── logs/
│
└── scripts/
    ├── run_backend.py
    ├── run_pipeline.py
    ├── check_ffmpeg.py
    ├── clean_storage.py
    ├── test_cosyvoice.py
    └── test_wav2lip.py
```

---

# 七、数据库设计

第一版使用 SQLite。

任务表名称：

```text
tasks
```

字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | 任务 ID |
| text | text | 输入文字 |
| avatar_id | string | 数字人 ID |
| voice_id | string | 声音 ID |
| speed | float | 语速 |
| resolution | string | 视频分辨率 |
| status | string | 任务状态 |
| progress | integer | 进度 |
| message | string | 当前状态描述 |
| audio_path | string | 音频路径 |
| raw_video_path | string | 原始视频路径 |
| video_path | string | 最终视频路径 |
| error_message | text | 错误信息 |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

---

# 八、任务状态设计

统一使用以下状态：

```python
PENDING = "pending"
PROCESSING_TEXT = "processing_text"
GENERATING_AUDIO = "generating_audio"
GENERATING_VIDEO = "generating_video"
FINALIZING = "finalizing"
SUCCESS = "success"
FAILED = "failed"
```

建议进度值：

```text
pending             0
processing_text    10
generating_audio   30
generating_video   60
finalizing         90
success           100
failed              保留失败时的当前进度
```

---

# 九、任务目录规范

每个任务必须建立独立目录：

```text
storage/tasks/{task_id}/
```

目录内容：

```text
input.json
speech.wav
raw_video.mp4
final.mp4
task.log
```

`input.json` 保存：

```json
{
  "text": "欢迎使用智能数字人系统。",
  "avatar_id": "avatar_01",
  "voice_id": "voice_01",
  "speed": 1.0,
  "resolution": "512x512"
}
```

---

# 十、后端接口设计

## 健康检查

```http
GET /health
```

返回：

```json
{
  "success": true,
  "data": {
    "status": "ok"
  },
  "message": "服务正常"
}
```

## 创建任务

```http
POST /api/tasks
```

请求：

```json
{
  "text": "欢迎使用智能数字人系统。",
  "avatar_id": "avatar_01",
  "voice_id": "voice_01",
  "speed": 1.0,
  "resolution": "512x512"
}
```

返回：

```json
{
  "success": true,
  "data": {
    "task_id": "dh_20260710_001",
    "status": "pending",
    "progress": 0
  },
  "message": "任务创建成功"
}
```

## 获取任务列表

```http
GET /api/tasks
```

支持：

```text
page
page_size
status
```

## 查询任务状态

```http
GET /api/tasks/{task_id}
```

返回：

```json
{
  "success": true,
  "data": {
    "task_id": "dh_20260710_001",
    "status": "generating_video",
    "progress": 70,
    "message": "正在生成数字人视频"
  },
  "message": "查询成功"
}
```

## 获取任务结果

```http
GET /api/tasks/{task_id}/result
```

返回：

```json
{
  "success": true,
  "data": {
    "task_id": "dh_20260710_001",
    "status": "success",
    "audio_url": "/static/tasks/dh_20260710_001/speech.wav",
    "video_url": "/static/tasks/dh_20260710_001/final.mp4"
  },
  "message": "生成成功"
}
```

## 重试任务

```http
POST /api/tasks/{task_id}/retry
```

## 删除任务

```http
DELETE /api/tasks/{task_id}
```

删除：

- 数据库记录。
- 对应任务目录。
- 生成的音频和视频。

## 获取声音列表

```http
GET /api/voices
```

## 获取数字人列表

```http
GET /api/avatars
```

---

# 十一、统一响应格式

所有接口必须返回统一格式：

```json
{
  "success": true,
  "data": {},
  "message": "操作成功"
}
```

异常返回：

```json
{
  "success": false,
  "data": null,
  "message": "具体错误原因"
}
```

---

# 十二、文本预处理要求

TextService 负责：

- 删除首尾空格。
- 删除重复换行。
- 合并多余空格。
- 过滤空文本。
- 限制最大长度。
- 第一版最大长度建议 500 个汉字。
- 根据中文标点切句。
- 处理数字。
- 处理日期。
- 处理百分比。
- 处理英文缩写。
- 返回句子列表。

示例输入：

```text
欢迎使用智能数字人系统。今天是2026年7月10日。
```

示例输出：

```python
[
    "欢迎使用智能数字人系统。",
    "今天是二零二六年七月十日。"
]
```

---

# 十三、TTS 适配器要求

## 抽象接口

```python
from abc import ABC, abstractmethod


class TTSAdapter(ABC):

    @abstractmethod
    def generate(
        self,
        text: str,
        output_path: str,
        voice_id: str,
        speed: float = 1.0,
    ) -> str:
        raise NotImplementedError
```

## MockTTSAdapter

本地开发时：

- 从 `assets/demo/demo.wav` 复制到任务目录。
- 模拟 1 秒延迟。
- 返回输出路径。
- 输入文件不存在时抛出清晰异常。
- 生成后检查文件存在。
- 生成后检查文件大小大于 0。

## CosyVoiceAdapter

AutoDL 阶段实现：

- 通过环境变量 `TTS_PROVIDER=cosyvoice` 启用。
- 模型路径从 `COSYVOICE_MODEL_PATH` 读取。
- 服务启动时加载模型。
- 不允许每次任务重新加载。
- 自动判断 CPU 或 CUDA。
- 生成 WAV。
- 检查音频时长。
- 支持 voice_id。
- 支持 speed。
- 异常必须转换成可读信息。
- 保留 Mock 模式。

## GPTSoVITSAdapter

只预留接口和占位实现。

第一版可以不接入真实模型。

---

# 十四、数字人适配器要求

## 抽象接口

```python
from abc import ABC, abstractmethod


class AvatarAdapter(ABC):

    @abstractmethod
    def generate(
        self,
        audio_path: str,
        avatar_path: str,
        output_path: str,
    ) -> str:
        raise NotImplementedError
```

## MockAvatarAdapter

本地开发时：

- 从 `assets/demo/demo.mp4` 复制到任务目录。
- 模拟 2 秒延迟。
- 返回输出路径。
- 检查输入音频存在。
- 检查人物素材存在。
- 检查输出文件存在且大小大于 0。

## Wav2LipAdapter

AutoDL 阶段实现：

- 通过 `AVATAR_PROVIDER=wav2lip` 启用。
- 模型权重路径从 `.env` 读取。
- Wav2Lip 仓库路径从 `.env` 读取。
- 输入支持 PNG、JPG、MP4。
- 输出必须为 MP4。
- 使用 `subprocess.run` 调用推理脚本。
- 禁止使用 `shell=True`。
- 设置超时时间。
- 捕获 stdout。
- 捕获 stderr。
- 检查输入文件。
- 检查模型权重。
- 检查输出文件。
- 推理失败时返回明确错误信息。
- 保留 Mock 模式。

---

# 十五、FFmpegService 要求

必须实现：

- 检查 FFmpeg 是否安装。
- 音视频合并。
- 转换为 H.264。
- 音频编码 AAC。
- 像素格式 yuv420p。
- 默认帧率 25fps。
- 支持设置输出分辨率。
- 禁止 `shell=True`。
- 设置执行超时。
- 保存 stderr。
- 检查输出文件。
- 失败时给出完整错误信息。

建议输出命令逻辑：

```text
raw_video.mp4 + speech.wav
    ↓
final.mp4
```

---

# 十六、PipelineService 流程

PipelineService 必须严格按照以下步骤运行：

1. 校验任务是否存在。
2. 创建任务目录。
3. 保存 `input.json`。
4. 更新状态为 `processing_text`。
5. 调用 TextService。
6. 更新状态为 `generating_audio`。
7. 调用 TTSAdapter。
8. 生成 `speech.wav`。
9. 更新状态为 `generating_video`。
10. 调用 AvatarAdapter。
11. 生成 `raw_video.mp4`。
12. 更新状态为 `finalizing`。
13. 调用 FFmpegService。
14. 生成 `final.mp4`。
15. 检查最终文件存在。
16. 检查最终文件大小大于 0。
17. 更新状态为 `success`。
18. 更新进度为 100。
19. 发生异常时：
    - 更新状态为 `failed`。
    - 保存 `error_message`。
    - 记录完整错误堆栈。
    - 写入 `task.log`。
20. Pipeline 业务代码不能直接依赖具体模型类。

---

# 十七、前端页面要求

页面采用简洁的深色科技风格。

## 页面区域

- 页面标题。
- 项目说明。
- 文字输入框。
- 字数统计。
- 数字人选择卡片。
- 声音选择下拉框。
- 语速选择。
- 分辨率选择。
- 生成按钮。
- 任务状态。
- 任务进度条。
- 错误提示。
- 视频播放器。
- 下载按钮。
- 重新生成按钮。
- 最近任务列表。
- 删除任务按钮。

## 前端交互要求

- 空文本不能提交。
- 超长文本不能提交。
- 提交期间按钮禁用。
- 每 2 秒轮询任务状态。
- 成功后停止轮询。
- 失败后停止轮询。
- 页面刷新后可以继续查询已有任务。
- 播放器支持 MP4。
- 下载按钮下载 `final.mp4`。
- 错误信息使用中文显示。

---

# 十八、配置文件要求

`.env.example` 至少包含：

```env
APP_NAME=Digital Human MVP
APP_ENV=development
APP_HOST=0.0.0.0
APP_PORT=8000

DATABASE_URL=sqlite:///./digital_human.db

STORAGE_ROOT=./storage
ASSETS_ROOT=./assets

TTS_PROVIDER=mock
AVATAR_PROVIDER=mock

COSYVOICE_MODEL_PATH=
GPT_SOVITS_MODEL_PATH=

WAV2LIP_REPO_PATH=
WAV2LIP_CHECKPOINT_PATH=

FFMPEG_PATH=ffmpeg

MAX_TEXT_LENGTH=500
TASK_TIMEOUT_SECONDS=600
```

---

# 十九、日志要求

必须记录：

- 请求 ID。
- 任务 ID。
- 任务创建时间。
- 每个阶段开始时间。
- 每个阶段结束时间。
- TTS 耗时。
- 数字人生成耗时。
- FFmpeg 耗时。
- 总耗时。
- 错误堆栈。
- 外部命令 stdout。
- 外部命令 stderr。

禁止只打印到控制台。

必须同时写入：

```text
storage/logs/app.log
storage/tasks/{task_id}/task.log
```

---

# 二十、测试要求

## 单元测试

至少包含：

- TextService 空文本测试。
- TextService 超长文本测试。
- TextService 切句测试。
- TaskRepository 创建任务测试。
- TaskRepository 更新状态测试。
- MockTTSAdapter 测试。
- MockAvatarAdapter 测试。
- StorageService 测试。
- Pipeline 成功测试。
- Pipeline 失败测试。
- 任务 API 测试。

## 集成测试

至少验证：

1. 创建任务。
2. 后台执行 Pipeline。
3. 状态从 pending 变化到 success。
4. 生成 speech.wav。
5. 生成 raw_video.mp4。
6. 生成 final.mp4。
7. 结果接口返回 video_url。
8. 删除任务后文件被清理。

## 稳定性测试

Mock 模式下连续执行 20 次任务。

要求：

- 不崩溃。
- 不出现任务状态卡死。
- 每个任务都有独立目录。
- 每个失败任务都有日志。
- 数据库状态和文件状态一致。

---

# 二十一、代码质量要求

1. 所有 Python 函数添加类型注解。
2. 所有 API 使用 Pydantic 校验。
3. 所有模块职责单一。
4. 使用依赖注入。
5. 使用适配器模式。
6. 使用 Repository 模式。
7. 统一异常处理。
8. 统一 JSON 响应。
9. 禁止写死绝对路径。
10. 配置从 `.env` 读取。
11. 关键代码添加中文注释。
12. 禁止使用 `shell=True`。
13. 子进程必须设置 timeout。
14. 文件操作必须检查异常。
15. 模型加载必须单例化。
16. README 必须写清楚启动步骤。
17. 每完成一个阶段必须运行测试。
18. 测试失败时必须先修复，不能继续下一阶段。

---

# 二十二、开发阶段

---

## 阶段 1：项目初始化与后端基础

### 目标

完成：

- 项目目录。
- 后端基础配置。
- `.env.example`。
- SQLite。
- SQLAlchemy。
- 任务模型。
- Repository。
- 健康检查接口。
- 日志基础配置。

### 验收标准

- 后端能启动。
- `/health` 返回成功。
- 数据库自动创建。
- 可以创建任务记录。
- pytest 通过。

### Claude Code 指令

```text
请阅读项目根目录的 CLAUDE.md。

现在只执行“阶段1：项目初始化与后端基础”。

要求：

1. 创建完整项目目录，但只实现阶段1所需文件。
2. 创建 FastAPI 项目。
3. 创建 config.py。
4. 创建 database.py。
5. 创建 logging_config.py。
6. 创建 Task 数据库模型。
7. 创建 TaskRepository。
8. 创建统一响应格式。
9. 创建统一异常处理基础。
10. 实现 GET /health。
11. 创建 .env.example。
12. 创建 backend/requirements.txt。
13. 添加阶段1单元测试。
14. 启动后端并运行测试。
15. 修复所有错误。

完成后只输出：

- 新增和修改的文件列表。
- 每个文件的作用。
- 启动命令。
- 测试命令。
- 测试结果。
- 当前仍存在的问题。

不要执行阶段2。
```

---

## 阶段 2：任务 API 与 Mock Pipeline

### 目标

完成：

- 创建任务接口。
- 查询任务接口。
- 任务列表接口。
- 删除任务。
- 重试任务。
- MockTTSAdapter。
- MockAvatarAdapter。
- StorageService。
- PipelineService。
- BackgroundTasks。

### 验收标准

- 不安装真实模型也能跑完整闭环。
- 输入文字后能复制 demo.wav 和 demo.mp4。
- 最终生成 final.mp4。
- 状态正常更新。
- 失败可追踪。

### Claude Code 指令

```text
请先阅读 CLAUDE.md 和当前项目代码。

现在只执行“阶段2：任务 API 与 Mock Pipeline”。

要求：

1. 实现任务创建、查询、列表、结果、重试和删除接口。
2. 实现 TaskSchema。
3. 实现 TextService。
4. 实现 StorageService。
5. 实现 TTSAdapter 抽象类。
6. 实现 MockTTSAdapter。
7. 实现 AvatarAdapter 抽象类。
8. 实现 MockAvatarAdapter。
9. 实现 FFmpegService 基础版本。
10. 实现 PipelineService。
11. 使用 FastAPI BackgroundTasks 执行任务。
12. 每个任务建立独立目录。
13. 保存 input.json、speech.wav、raw_video.mp4、final.mp4、task.log。
14. 实现完整状态流转。
15. 实现失败状态和错误日志。
16. 如果 demo.wav 或 demo.mp4 不存在，提供清晰提示。
17. 添加单元测试和集成测试。
18. 运行全部后端测试。
19. 修复所有错误。

完成后只输出：

- 新增和修改的文件列表。
- 核心流程说明。
- API 列表。
- 测试结果。
- 一次完整 Mock 任务的运行结果。
- 当前仍存在的问题。

不要执行阶段3。
```

---

## 阶段 3：前端页面

### 目标

完成 Vue 前端。

### 验收标准

- 页面可输入文字。
- 可提交任务。
- 可查看进度。
- 可播放视频。
- 可下载视频。
- 可查看最近任务。
- 可删除任务。

### Claude Code 指令

```text
请先阅读 CLAUDE.md 和当前项目。

现在只执行“阶段3：前端页面”。

要求：

1. 使用 Vue 3 + Vite + TypeScript。
2. 使用 Element Plus。
3. 使用 Axios。
4. 开发 GenerateView。
5. 开发文字输入组件。
6. 开发数字人选择组件。
7. 开发声音选择组件。
8. 开发任务进度组件。
9. 开发视频结果组件。
10. 开发最近任务组件。
11. 实现创建任务。
12. 每2秒轮询任务状态。
13. 成功后停止轮询。
14. 失败后显示 error_message。
15. 实现 MP4 播放。
16. 实现视频下载。
17. 实现任务删除。
18. 页面采用深色科技风格。
19. 添加前端类型定义。
20. 添加前端 API 封装。
21. 配置开发环境代理。
22. 运行 npm build。
23. 修复所有错误。

完成后只输出：

- 新增和修改的文件列表。
- 页面模块说明。
- 本地启动命令。
- 构建结果。
- 前后端联调方法。
- 当前仍存在的问题。

不要执行阶段4。
```

---

## 阶段 4：Mock 全链路联调

### 目标

完成前后端完整联调。

### 验收标准

- 用户在网页输入文字。
- 后端创建任务。
- Mock Pipeline 执行。
- 前端显示进度。
- 最终播放 final.mp4。
- 下载正常。
- 删除正常。

### Claude Code 指令

```text
请阅读 CLAUDE.md 和当前项目。

现在只执行“阶段4：Mock 全链路联调”。

要求：

1. 启动前端和后端。
2. 检查 CORS。
3. 检查静态文件访问。
4. 检查视频 URL。
5. 检查轮询逻辑。
6. 检查状态流转。
7. 检查任务失败提示。
8. 检查视频播放。
9. 检查视频下载。
10. 检查任务删除。
11. 完成一次端到端 Mock 测试。
12. 完成连续10次任务测试。
13. 修复所有问题。
14. 更新 README 的本地开发说明。

完成后只输出：

- 联调结果。
- 已修复的问题。
- 完整启动命令。
- 完整操作步骤。
- 连续任务测试结果。
- 当前仍存在的问题。

不要接入真实模型。
```

---

## 阶段 5：代码上传 Git

### 目标

将本地项目上传 GitHub 或 Gitee。

### Claude Code 指令

```text
请检查当前项目是否适合提交 Git。

只执行以下工作：

1. 完善 .gitignore。
2. 排除模型权重。
3. 排除 storage 生成文件。
4. 排除数据库运行文件。
5. 排除日志。
6. 排除 node_modules。
7. 排除 .env。
8. 保留 .env.example。
9. 检查是否存在密钥。
10. 检查是否存在绝对路径。
11. 更新 README。
12. 给出 Git 初始化命令。
13. 给出首次提交命令。
14. 给出推送 GitHub 或 Gitee 的命令。

不要执行 git push，除非我明确要求。
```

建议 `.gitignore` 包含：

```gitignore
.env
__pycache__/
*.pyc
.pytest_cache/
.venv/
venv/
node_modules/
frontend/dist/
*.db
storage/tasks/*
storage/audio/*
storage/video/*
storage/logs/*
!storage/tasks/.gitkeep
!storage/audio/.gitkeep
!storage/video/.gitkeep
!storage/logs/.gitkeep
pretrained_models/
checkpoints/
*.pth
*.pt
*.ckpt
*.onnx
*.safetensors
```

---

## 阶段 6：AutoDL 环境部署

### 目标

在 AutoDL 拉取代码并完成基础环境安装。

### Claude Code 指令

```text
当前代码已经推送到 Git 仓库。

现在项目运行环境是 AutoDL Linux GPU 服务器。

请只完成“AutoDL 基础部署”，不要接入真实模型。

要求：

1. 检查操作系统。
2. 检查 nvidia-smi。
3. 检查 CUDA。
4. 检查 Python。
5. 创建 Python 3.10 虚拟环境。
6. 安装后端依赖。
7. 安装 FFmpeg。
8. 安装前端依赖。
9. 创建 .env。
10. 保持 TTS_PROVIDER=mock。
11. 保持 AVATAR_PROVIDER=mock。
12. 启动后端。
13. 启动前端。
14. 测试 /health。
15. 测试 Mock 完整任务。
16. 给出 AutoDL 端口映射说明。
17. 给出进程启动和停止命令。
18. 更新 README 的 AutoDL 部署说明。

完成后只输出：

- 环境检查结果。
- 安装命令。
- 启动命令。
- 访问方式。
- Mock 测试结果。
- 当前仍存在的问题。

不要接入 CosyVoice。
不要接入 Wav2Lip。
```

---

## 阶段 7：接入 CosyVoice

### 目标

将 MockTTSAdapter 替换为真实 CosyVoiceAdapter。

### Claude Code 指令

```text
当前项目已经完成 MockTTSAdapter，并已在 AutoDL 上运行。

现在只接入 CosyVoice。

要求：

1. 先阅读当前 TTSAdapter 接口。
2. 不修改 PipelineService 的业务逻辑。
3. 保留 MockTTSAdapter。
4. 实现 CosyVoiceAdapter。
5. 通过 TTS_PROVIDER 切换 mock 和 cosyvoice。
6. 模型路径从 COSYVOICE_MODEL_PATH 读取。
7. 服务启动时加载模型。
8. 禁止每次任务重新加载模型。
9. 自动判断 CPU 或 CUDA。
10. 支持 text。
11. 支持 output_path。
12. 支持 voice_id。
13. 支持 speed。
14. 输出必须为 WAV。
15. 检查输出文件存在。
16. 检查文件大小大于0。
17. 检查音频时长大于0。
18. 异常转换成中文可读错误。
19. 不允许写死 CUDA 设备。
20. 编写 scripts/test_cosyvoice.py。
21. 测试文字：
    欢迎使用智能数字人系统，这是一次语音合成测试。
22. 给出模型下载和安装命令。
23. 给出 .env 配置示例。
24. 运行独立 TTS 测试。
25. 运行 Pipeline 测试。
26. 修复所有错误。

完成后只输出：

- 修改的文件。
- CosyVoice 加载方式。
- 安装命令。
- 配置方法。
- 测试结果。
- 生成音频路径。
- TTS 耗时。
- 当前仍存在的问题。

不要接入 Wav2Lip。
```

---

## 阶段 8：接入 Wav2Lip

### 目标

将 MockAvatarAdapter 替换为真实 Wav2LipAdapter。

### Claude Code 指令

```text
当前项目已经完成 CosyVoiceAdapter，并且可以生成 speech.wav。

现在只接入 Wav2Lip。

要求：

1. 阅读当前 AvatarAdapter 接口。
2. 不修改 PipelineService 的核心逻辑。
3. 保留 MockAvatarAdapter。
4. 实现 Wav2LipAdapter。
5. 通过 AVATAR_PROVIDER 切换 mock 和 wav2lip。
6. Wav2Lip 仓库路径从 WAV2LIP_REPO_PATH 读取。
7. 模型权重路径从 WAV2LIP_CHECKPOINT_PATH 读取。
8. 输入支持 JPG、PNG、MP4。
9. 输出必须为 MP4。
10. 使用 subprocess.run。
11. 禁止 shell=True。
12. 设置执行超时。
13. 捕获 stdout。
14. 捕获 stderr。
15. 检查输入音频。
16. 检查人物素材。
17. 检查模型权重。
18. 检查输出文件。
19. 失败时返回中文错误。
20. 实现 scripts/test_wav2lip.py。
21. 使用 FFmpeg 输出：
    - H.264
    - AAC
    - yuv420p
    - 25fps
22. 给出 Wav2Lip 下载和安装命令。
23. 给出模型权重目录示例。
24. 运行独立 Wav2Lip 测试。
25. 运行完整 Pipeline 测试。
26. 修复所有错误。

完成后只输出：

- 修改的文件。
- Wav2Lip 调用方式。
- 安装命令。
- 配置方法。
- 测试结果。
- 生成视频路径。
- 视频生成耗时。
- 当前仍存在的问题。
```

---

## 阶段 9：真实模型完整联调

### 目标

完成：

```text
文字
→ CosyVoice
→ Wav2Lip
→ FFmpeg
→ final.mp4
→ 前端播放
```

### Claude Code 指令

```text
当前项目已经完成：

- Vue 前端
- FastAPI 后端
- SQLite
- CosyVoiceAdapter
- Wav2LipAdapter
- FFmpegService
- PipelineService

现在只执行真实模型完整联调。

要求：

1. 输入文字后创建任务。
2. CosyVoice 生成 speech.wav。
3. Wav2Lip 生成 raw_video.mp4。
4. FFmpeg 生成 final.mp4。
5. 前端显示真实任务进度。
6. 前端播放 final.mp4。
7. 前端下载 final.mp4。
8. 失败时显示明确原因。
9. 增加任务超时。
10. 增加显存不足处理。
11. 增加模型未找到处理。
12. 增加输入文件不存在处理。
13. 增加 FFmpeg 失败处理。
14. 增加任务日志。
15. 记录各阶段耗时。
16. 连续执行10次真实任务。
17. 检查显存释放情况。
18. 检查磁盘文件。
19. 检查数据库状态。
20. 修复所有错误。
21. 更新 README。

完成后只输出：

- 完整联调结果。
- 一次任务的各阶段耗时。
- 连续任务测试结果。
- GPU 显存使用情况。
- 已修复的问题。
- 当前仍存在的问题。
- 最终启动命令。
```

---

## 阶段 10：稳定性与交付

### 目标

形成可演示、可交付版本。

### Claude Code 指令

```text
当前项目真实模型链路已经跑通。

现在只执行稳定性优化和项目交付。

要求：

1. 检查模型是否只加载一次。
2. 增加启动预热。
3. 增加任务超时。
4. 增加并发限制。
5. 第一版每张 GPU 同时只执行一个视频任务。
6. 增加磁盘空间检查。
7. 增加过期文件清理。
8. 增加失败任务重试。
9. 增加服务健康检查。
10. 增加模型健康检查。
11. 完善日志。
12. 完善错误信息。
13. 完善 Dockerfile。
14. 完善 docker-compose.yml。
15. 完善 README。
16. 编写 AutoDL 部署手册。
17. 编写常见错误排查手册。
18. 编写项目演示操作步骤。
19. 编写最终验收清单。
20. 运行全部测试。
21. 修复所有错误。

完成后只输出：

- 最终项目结构。
- 启动方式。
- 部署方式。
- 测试结果。
- 验收结果。
- 已知限制。
- 后续升级建议。
```

---

# 二十三、Claude Code 总控 Prompt

第一次打开 Claude Code 时，可以先粘贴下面这段：

```text
你现在是一名具有10年以上经验的系统架构师、Python工程师和Vue工程师。

请先完整阅读项目根目录的 CLAUDE.md。

这个项目是“纯文本驱动数字人 MVP”。

核心流程：

文字输入
→ 文本预处理
→ TTS 生成语音
→ 数字人模型生成视频
→ FFmpeg 后处理
→ 前端播放和下载

请严格遵守以下规则：

1. 不要一次性开发整个项目。
2. 必须按照 CLAUDE.md 中的阶段顺序执行。
3. 当前只执行我明确指定的阶段。
4. 每个阶段完成后必须运行测试。
5. 测试失败必须先修复。
6. 不允许跳过阶段。
7. 不允许擅自扩大项目范围。
8. 不接入 RAG。
9. 不接入大模型问答。
10. 不做 WebRTC。
11. 不做实时数字人。
12. 模型必须使用适配器模式。
13. Pipeline 不得直接依赖具体模型。
14. 本地先使用 Mock 模型。
15. AutoDL 再接入真实模型。
16. 禁止在代码中写死绝对路径。
17. 禁止使用 shell=True。
18. 所有配置从 .env 读取。
19. 所有 Python 代码必须有类型注解。
20. 所有关键逻辑必须有中文注释。
21. 每次完成后告诉我：
    - 修改了哪些文件
    - 每个文件的作用
    - 如何启动
    - 如何测试
    - 测试结果
    - 仍存在的问题
22. 完成当前阶段后停止，等待我的确认。

现在先检查 CLAUDE.md 和当前目录，不要修改任何代码。

告诉我：

1. 你理解的项目目标。
2. 你理解的系统架构。
3. 当前应该执行的第一个阶段。
4. 第一个阶段准备创建哪些文件。
5. 第一个阶段的验收标准。
```

---

# 二十四、建议开发顺序

严格按照下面的顺序执行：

```text
阶段1：项目初始化与后端基础
    ↓
阶段2：任务 API 与 Mock Pipeline
    ↓
阶段3：Vue 前端
    ↓
阶段4：Mock 全链路联调
    ↓
阶段5：上传 Git
    ↓
阶段6：AutoDL 基础环境
    ↓
阶段7：接入 CosyVoice
    ↓
阶段8：接入 Wav2Lip
    ↓
阶段9：真实模型完整联调
    ↓
阶段10：稳定性优化与交付
```

---

# 二十五、最终验收标准

项目最终必须满足：

- 网页可以输入文字。
- 可以选择人物。
- 可以选择声音。
- 可以选择语速。
- 可以创建任务。
- 可以查看进度。
- CosyVoice 能生成音频。
- Wav2Lip 能生成口型视频。
- FFmpeg 能输出浏览器可播放 MP4。
- 前端可以播放视频。
- 前端可以下载视频。
- 失败任务有明确原因。
- 每个任务有独立目录。
- 每个任务有完整日志。
- 连续 10 次真实任务不崩溃。
- 模型只加载一次。
- 本地 Mock 模式可运行。
- AutoDL 真实模型模式可运行。
- README 包含完整启动和部署步骤。

---

# 二十六、项目关键原则

```text
先完成稳定闭环
再优化声音
再优化口型
再优化速度
最后考虑微调和并发
```

第一版最重要的不是生成最逼真的数字人，而是确保：

```text
输入文字
→ 稳定生成语音
→ 稳定生成视频
→ 网页正常播放
```

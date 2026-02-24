# Banana Slides Backend

蕉幻（Banana Slides）後端服務 - AI驅動的PPT生成系統

## 技術棧

- **框架**: Flask 3.0
- **數據庫**: SQLite + SQLAlchemy ORM
- **AI服務**: Google Gemini API
- **PPT處理**: python-pptx
- **並發處理**: ThreadPoolExecutor
- **包管理**: uv

## 項目結構

```
backend/
├── app.py                    # Flask應用入口
├── config.py                 # 配置文件
├── models/                   # 數據庫模型
│   ├── __init__.py
│   ├── project.py           # Project模型
│   ├── page.py              # Page模型
│   └── task.py              # Task模型
├── services/                 # 服務層
│   ├── __init__.py
│   ├── ai_service.py        # AI相關服務
│   ├── file_service.py      # 文件管理服務
│   ├── export_service.py    # 導出服務
│   └── task_manager.py      # 異步任務管理
├── controllers/              # 控制器層
│   ├── __init__.py
│   ├── project_controller.py
│   ├── page_controller.py
│   ├── template_controller.py
│   ├── export_controller.py
│   └── file_controller.py
├── utils/                    # 工具函數
│   ├── __init__.py
│   ├── response.py          # 統一響應格式
│   └── validators.py        # 數據驗證
├── instance/                 # 數據庫文件目錄（自動創建）
├── uploads/                  # 文件上傳目錄（自動創建）
├── .env.example             # 環境變量示例
└── README.md                # 本文件
```

## 快速開始

### 1. 安裝依賴

本項目使用 [uv](https://github.com/astral-sh/uv) 管理 Python 依賴。所有依賴定義在項目根目錄的 `pyproject.toml` 文件中。

在項目根目錄下運行：
```bash
uv sync
```

這將自動安裝所有必需的依賴包。

### 2. 配置環境變量

複制 `.env.example` 為 `.env` 並填寫配置：

```bash
cp .env.example .env
```

編輯 `.env` 文件：

```env
GOOGLE_API_KEY=your-google-api-key
GOOGLE_API_BASE=https://generativelanguage.googleapis.com

# 火山引擎配置（可選，用於 Inpainting 圖像消除功能）
VOLCENGINE_ACCESS_KEY=your-volcengine-access-key
VOLCENGINE_SECRET_KEY=your-volcengine-secret-key
VOLCENGINE_INPAINTING_TIMEOUT=60
VOLCENGINE_INPAINTING_MAX_RETRIES=3
```

### 3. 初始化 / 升級數據庫結構（Alembic 遷移）

從當前版本開始，後端使用 Alembic 管理數據庫結構變更。

```bash
cd backend
uv run alembic upgrade head
```

> 注意：
> - 首次運行時會自動創建 `alembic_version` 表並將數據庫遷移到最新結構；
> - 後續新增模型字段時，只需要更新 `models/`，然後使用 `alembic revision --autogenerate` 生成遷移，再執行 `alembic upgrade head`。

### 4. 運行服務

使用 uv 運行：
```bash
cd backend
uv run python app.py
```
服務將在 `http://localhost:5000` 啟動。

## API文件

完整的API文件請参考項目根目錄的 `API設計文件.md`。

### 主要端點

#### 項目管理
- `POST /api/projects` - 創建項目
- `GET /api/projects/{project_id}` - 獲取項目詳情
- `PUT /api/projects/{project_id}` - 更新項目
- `DELETE /api/projects/{project_id}` - 刪除項目

#### 大綱生成
- `POST /api/projects/{project_id}/generate/outline` - 生成大綱

#### 描述生成
- `POST /api/projects/{project_id}/generate/descriptions` - 批量生成描述（異步）
- `POST /api/projects/{project_id}/pages/{page_id}/generate/description` - 單頁生成

#### 圖片生成
- `POST /api/projects/{project_id}/generate/images` - 批量生成圖片（異步）
- `POST /api/projects/{project_id}/pages/{page_id}/generate/image` - 單頁生成
- `POST /api/projects/{project_id}/pages/{page_id}/edit/image` - 編輯圖片

#### 模板管理
- `POST /api/projects/{project_id}/template` - 上傳模板
- `DELETE /api/projects/{project_id}/template` - 刪除模板

#### 導出
- `GET /api/projects/{project_id}/export/pptx` - 導出PPTX
- `GET /api/projects/{project_id}/export/pdf` - 導出PDF

#### 靜態文件
- `GET /files/{project_id}/{type}/{filename}` - 獲取文件

## 核心功能

### 1. AI驅動的內容生成

基於 Google Gemini API，支持：
- 自動生成PPT大綱
- 並行生成頁面描述
- 根據参考模板生成圖片
- 自然語言編輯圖片

### 2. 異步任務處理

使用 `ThreadPoolExecutor` 實現簡單但高效的異步任務處理：
- 並行生成多個頁面描述
- 並行生成多個頁面圖片
- 實時任務進度跟踪

### 3. 文件管理

完整的文件管理系統：
- 項目級文件隔離
- 模板圖片管理
- 生成圖片管理
- 自動清理机制

### 4. Inpainting 圖像消除（可選）

基於火山引擎的 Inpainting 服務，支持：
- 根據邊界框（bbox）精確消除圖像區域
- 自動生成掩碼圖像
- 重新生成背景（保留前景，消除其他區域）
- 支持批量處理和重試机制

使用方法：
```python
from services.inpainting_service import InpaintingService, remove_regions
from PIL import Image

# 方式1：使用服務類
service = InpaintingService()
image = Image.open('original.png')
bboxes = [(100, 100, 200, 200), (300, 150, 400, 250)]  # 要消除的區域
result = service.remove_regions_by_bboxes(image, bboxes)

# 方式2：使用便捷函數
result = remove_regions(image, bboxes, expand_pixels=5)
```

### 5. 數據持久化

使用 SQLite + SQLAlchemy：
- 輕量級，無需額外配置
- 支持關系型數據操作
- 事務保證數據一致性

## 開發說明

### 數據模型

#### Project（項目）
- 項目基本信息
- 模板圖片路徑
- 項目狀態
- 關联的頁面和任務

#### Page（頁面）
- 頁面順序
- 大綱內容（JSON）
- 描述內容（JSON）
- 生成的圖片路徑
- 頁面狀態

#### Task（任務）
- 任務類型（生成描述/生成圖片）
- 任務狀態
- 進度信息（JSON）
- 錯誤信息

### 狀態机

#### 項目狀態
```
DRAFT → OUTLINE_GENERATED → DESCRIPTIONS_GENERATED → GENERATING_IMAGES → COMPLETED
```

#### 頁面狀態
```
DRAFT → DESCRIPTION_GENERATED → GENERATING → COMPLETED | FAILED
```

#### 任務狀態
```
PENDING → PROCESSING → COMPLETED | FAILED
```

### 擴展開發

#### 添加新的AI模型

在 `services/ai_service.py` 中添加新的模型支持：

```python
class AIService:
    def __init__(self, api_key: str, model_type: str = 'gemini'):
        if model_type == 'gemini':
            # Gemini implementation
        elif model_type == 'openai':
            # OpenAI implementation
        # ...
```

#### 自定義提示詞模板

修改 `services/ai_service.py` 中的提示詞生成邏輯：

```python
def generate_image_prompt(self, ...):
    prompt = dedent(f"""
        # 自定義提示詞模板
        ...
    """)
    return prompt
```

#### 添加新的導出格式

在 `services/export_service.py` 中添加新的導出方法：

```python
class ExportService:
    @staticmethod
    def create_custom_format(image_paths, output_file):
        # 實現自定義格式導出
        pass
```


## 測試

### 健康檢查

```bash
curl http://localhost:5000/health
```

### 創建項目

```bash
curl -X POST http://localhost:5000/api/projects \
  -H "Content-Type: application/json" \
  -d '{"creation_type":"idea","idea_prompt":"生成環保主題ppt"}'
```

### 上傳模板

```bash
curl -X POST http://localhost:5000/api/projects/{project_id}/template \
  -F "template_image=@template.png"
```

### 生成大綱

```bash
curl -X POST http://localhost:5000/api/projects/{project_id}/generate/outline \
  -H "Content-Type: application/json" \
  -d '{"idea_prompt":"生成環保主題ppt"}'
```

## 常見問題

### Q: 數據庫文件在哪裡？
A: 在 `backend/instance/database.db`，會自動創建。

### Q: 上傳的文件存在哪裡？
A: 在 `uploads/{project_id}/` 目錄下，按項目隔離。

### Q: 如何修改並發數？
A: 推薦通過前端設置頁修改（會同步到數據庫並覆蓋 `.env` 值）；也可以在 `.env` 文件中修改 `MAX_DESCRIPTION_WORKERS` 和 `MAX_IMAGE_WORKERS` 作為默認值，然後在設置頁點擊“重置為默認值”同步到 DB。

### Q: 如何切換到其他AI模型 / 修改 MinerU 地址？
A: 從當前版本開始，推薦通過前端“系統設置”頁面修改：
- 大模型提供商格式 / API Base / API Key
- 文本模型 (`TEXT_MODEL`) / 圖片模型 (`IMAGE_MODEL`)
- MinerU 地址 (`MINERU_API_BASE`) / 圖片識別模型 (`IMAGE_CAPTION_MODEL`)

這些值會保存到 `settings` 表並覆蓋 `.env` 中對應配置，點擊“重置為默認值”會回到 `.env` 的默認值。

### Q: 支持哪些圖片格式？
A: PNG, JPG, JPEG, GIF, WEBP。在 `config.py` 中的 `ALLOWED_EXTENSIONS` 配置。


## 開源字體說明

本項目包含 **Noto Sans CJK SC**（思源黑體簡體中文）字體文件，用於 PPT 導出時的精確文本測量。

- **字體文件**: `fonts/NotoSansSC-Regular.ttf`
- **來源**: [Google Noto CJK Fonts](https://github.com/googlefonts/noto-cjk)
- **許可證**: [SIL Open Font License 1.1 (OFL)](https://scripts.sil.org/OFL)

OFL 許可證允許自由使用、修改和分發該字體。

## 联系方式

如有問題或建議，請通過 GitHub Issues 反饋。


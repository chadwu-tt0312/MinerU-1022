# MinerU 使用說明

**專案名稱**：MinerU  
**專案網頁**：<https://github.com/opendatalab/MinerU>  
**參考網頁**：<https://deepwiki.com/opendatalab/MinerU>  
**目標讀者**：負責部署與維護的技術人員

---

## 1. 簡介

### 專案功能與架構

MinerU 是一個智能 PDF 解析工具，能夠將 PDF 文件轉換為機器可讀的格式（如 Markdown、JSON）。專案採用模組化架構，支援兩種後端：

- **Pipeline 後端**：傳統流水線方式，模組化設計，支援 CPU/GPU 運算
- **VLM 後端**：使用 MinerU2.5 多模態大模型，端到端處理，需要 GPU

### 核心功能

- 版面分析（識別標題、段落、表格、圖片位置）
- 文字提取（支援文字型 PDF 和掃描版 PDF 的 OCR）
- 公式識別（轉換為 LaTeX 格式）
- 表格識別（轉換為 HTML 格式，支援跨頁表格）
- 閱讀順序排序（支援單欄、多欄、複雜版面）
- 多語言支援（84 種語言）

### 技術架構

```
用戶輸入 (PDF/圖片)
    ↓
文件分類 (文字型/掃描版)
    ↓
版面分析 → 文字提取 → 公式識別 → 表格識別
    ↓
閱讀順序排序 → 段落拼接 → 標題分級 (可選 LLM)
    ↓
輸出生成 (Markdown/JSON)
```

---

## 2. 環境變數詳解 (Environment Variables)

### 核心環境變數

| 變數名稱 | 預設值 | 必填 | 說明 |
|---------|--------|------|------|
| `MINERU_DEVICE_MODE` | `auto` | 否 | 指定運算設備：`cuda`（NVIDIA GPU）、`mps`（Apple Silicon）、`npu`（華為昇騰）、`cpu`（純 CPU） |
| `MINERU_MODEL_SOURCE` | `huggingface` | 否 | 模型來源：`huggingface`、`modelscope`、`local`（本地模型） |
| `MINERU_FORMULA_ENABLE` | `true` | 否 | 是否啟用公式識別：`true`、`false` |
| `MINERU_TABLE_ENABLE` | `true` | 否 | 是否啟用表格識別：`true`、`false` |
| `MINERU_TOOLS_CONFIG_JSON` | `mineru.json` | 否 | 配置檔案名稱或絕對路徑（預設在用戶目錄 `~` 下） |
| `MINERU_MIN_BATCH_INFERENCE_SIZE` | `384` | 否 | 批次推理大小，調大可提升速度但消耗更多記憶體 |
| `MINERU_VIRTUAL_VRAM_SIZE` | 自動偵測 | 否 | 虛擬顯存大小（MB），用於記憶體管理 |
| `MINERU_DONOT_CLEAN_MEM` | 未設定 | 否 | 設為任意值時，不清理記憶體（用於除錯） |
| `VLLM_USE_V1` | `1` | 否 | 是否使用 vLLM v1 API：`1`（是）、`0`（否） |

### LLM 整合環境變數（Azure OpenAI）

> **注意**：MinerU 的 LLM 整合使用 OpenAI 協議，可透過配置檔案設定 Azure OpenAI。環境變數主要用於配置檔案路徑。

| 變數名稱 | 預設值 | 必填 | 說明 |
|---------|--------|------|------|
| `MINERU_TOOLS_CONFIG_JSON` | `mineru.json` | 否 | 配置檔案路徑，LLM 設定在此檔案中 |

**Azure OpenAI 配置方式**：

在 `~/mineru.json` 中設定：

```json
{
  "llm-aided-config": {
    "title_aided": {
      "api_key": "your_azure_openai_api_key",
      "base_url": "https://your-resource-name.openai.azure.com/v1",
      "model": "gpt-4",  // 或您的 Azure OpenAI Deployment Name
      "enable": true
    }
  }
}
```

**Azure OpenAI 標準變數對應**：

| Azure OpenAI 標準變數 | MinerU 配置方式 | 說明 |
|----------------------|----------------|------|
| `AZURE_OPENAI_API_KEY` | `llm-aided-config.title_aided.api_key` | API 金鑰 |
| `AZURE_OPENAI_ENDPOINT` | `llm-aided-config.title_aided.base_url` | 端點 URL（需包含 `/v1`） |
| `AZURE_OPENAI_DEPLOYMENT_NAME` | `llm-aided-config.title_aided.model` | 部署名稱 |
| `OPENAI_API_VERSION` | 不支援 | 目前不支援指定 API 版本，使用預設版本 |

**Azure OpenAI 配置範例**：

```json
{
  "llm-aided-config": {
    "title_aided": {
      "api_key": "1234567890abcdef",
      "base_url": "https://my-resource.openai.azure.com/v1",
      "model": "gpt-4-deployment",
      "enable": true
    }
  }
}
```

### VLM 後端環境變數

| 變數名稱 | 預設值 | 必填 | 說明 |
|---------|--------|------|------|
| `MINERU_VLM_FORMULA_ENABLE` | `true` | 否 | VLM 後端是否啟用公式識別 |
| `MINERU_VLM_TABLE_ENABLE` | `true` | 否 | VLM 後端是否啟用表格識別 |

---

## 3. 安裝與部署 (Installation & Deployment)

### 套件安裝（本機部署）

#### 前置需求

- **Python**：3.10 - 3.13
- **作業系統**：Linux、Windows、macOS
- **記憶體**：最低 16GB，建議 32GB+
- **磁碟空間**：20GB+（用於模型下載）
- **GPU**（可選）：
  - Pipeline 後端：Turing 架構及以後，6GB+ 顯存
  - VLM 後端：Turing 架構及以後，8GB+ 顯存

#### 使用 pip 安裝

```bash
# 升級 pip
pip install --upgrade pip

# 安裝 uv（推薦）
pip install uv

# 安裝 MinerU（核心功能）
uv pip install -U "mineru[core]"
```

#### 使用 uv 安裝（推薦）

```bash
# 安裝 uv
pip install uv

# 安裝 MinerU
uv pip install -U "mineru[core]"
```

#### 從源碼安裝

```bash
# 克隆倉庫
git clone https://github.com/opendatalab/MinerU.git
cd MinerU

# 安裝
uv pip install -e .[core]
```

#### 下載模型

```bash
# 下載所有模型（首次使用必須執行）
mineru-models-download -s huggingface -m all

# 或指定模型來源
export MINERU_MODEL_SOURCE=modelscope
mineru-models-download -s modelscope -m all
```

### Docker 部署

#### 使用 Docker Compose（推薦）

專案提供 `docker/compose.yaml`，支援三種服務：

1. **vLLM Server**：提供 vLLM 加速服務
2. **API Server**：提供 FastAPI 服務
3. **Gradio WebUI**：提供 Web 介面

**啟動服務**：

```bash
# 進入 docker 目錄
cd docker

# 啟動 vLLM Server
docker compose --profile vllm-server up -d

# 啟動 API Server
docker compose --profile api up -d

# 啟動 Gradio WebUI
docker compose --profile gradio up -d
```

**查看日誌**：

```bash
docker compose logs -f mineru-vllm-server
docker compose logs -f mineru-api
docker compose logs -f mineru-gradio
```

#### 使用 Dockerfile 建置

專案提供多個 Dockerfile：

- `docker/global/Dockerfile`：全球版本（使用 HuggingFace）
- `docker/china/Dockerfile`：中國版本（使用 ModelScope）

**建置映像**：

```bash
# 建置全球版本
cd docker/global
docker build -t mineru:latest .

# 建置中國版本
cd docker/china
docker build -t mineru:latest .
```

**運行容器**：

```bash
# 基本運行
docker run --gpus all -v /path/to/input:/input -v /path/to/output:/output mineru:latest \
  mineru -p /input -o /output

# 運行 API 服務
docker run --gpus all -p 8000:8000 mineru:latest mineru-api --host 0.0.0.0 --port 8000
```

### Kubernetes Helm 部署

> **注意**：專案未提供現成的 Helm Chart，以下為根據 `docker-compose.yml` 反推的部署指引。

#### 建立 Helm Chart

**目錄結構**：

```
mineru-helm/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pvc.yaml
│   └── configmap.yaml
```

**Chart.yaml**：

```yaml
apiVersion: v2
name: mineru
description: MinerU PDF Parser
type: application
version: 2.5.4
appVersion: 2.5.4
```

**values.yaml**：

```yaml
replicaCount: 1

image:
  repository: mineru
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 8000
  nodePort: 30080

persistence:
  enabled: true
  storageClass: nfs-client
  accessMode: ReadWriteMany
  size: 100Gi
  nfs:
    server: nfs-server.example.com
    path: /data/mineru

resources:
  limits:
    cpu: 4
    memory: 32Gi
    nvidia.com/gpu: 1
  requests:
    cpu: 2
    memory: 16Gi

env:
  MINERU_DEVICE_MODE: "cuda"
  MINERU_MODEL_SOURCE: "local"
```

**templates/deployment.yaml**：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mineru.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "mineru.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "mineru.name" . }}
    spec:
      containers:
      - name: mineru
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        command: ["mineru-api"]
        args: ["--host", "0.0.0.0", "--port", "8000"]
        ports:
        - containerPort: 8000
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        {{- if .Values.persistence.enabled }}
        persistentVolumeClaim:
          claimName: {{ include "mineru.fullname" . }}-pvc
        {{- end }}
```

**templates/pvc.yaml**：

```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "mineru.fullname" . }}-pvc
spec:
  accessModes:
    - {{ .Values.persistence.accessMode }}
  storageClassName: {{ .Values.persistence.storageClass }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
{{- end }}
```

**templates/service.yaml**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mineru.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 8000
      protocol: TCP
      {{- if eq .Values.service.type "NodePort" }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    app: {{ include "mineru.name" . }}
```

**部署指令**：

```bash
# 安裝 Helm Chart
helm install mineru ./mineru-helm

# 升級
helm upgrade mineru ./mineru-helm

# 卸載
helm uninstall mineru
```

---

## 4. 操作指南 (Operations)

### 基本操作

#### 命令行使用

**基本語法**：

```bash
mineru -p <input_path> -o <output_path>
```

**常用參數**：

```bash
# 指定後端
mineru -p input.pdf -o output/ --backend pipeline
mineru -p input.pdf -o output/ --backend vlm-transformers

# 指定語言
mineru -p input.pdf -o output/ --lang ch  # 中文
mineru -p input.pdf -o output/ --lang en  # 英文

# 指定解析方法
mineru -p input.pdf -o output/ --method auto  # 自動判斷
mineru -p input.pdf -o output/ --method txt   # 文字提取
mineru -p input.pdf -o output/ --method ocr  # OCR

# 指定頁面範圍
mineru -p input.pdf -o output/ --start 0 --end 10  # 處理第 1-11 頁
```

#### 啟動 API 服務

```bash
# 啟動 FastAPI 服務
mineru-api --host 0.0.0.0 --port 8000

# 訪問 API 文檔
# 瀏覽器打開 http://localhost:8000/docs
```

**API 使用範例**：

```bash
# 上傳文件並解析
curl -X POST "http://localhost:8000/api/v1/parse" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@document.pdf"
```

#### 啟動 Web UI

```bash
# 啟動 Gradio WebUI
mineru-gradio --server-name 0.0.0.0 --server-port 7860

# 訪問 Web UI
# 瀏覽器打開 http://localhost:7860
```

#### 啟動 vLLM Server

```bash
# 啟動 vLLM Server
mineru-vllm-server --host 0.0.0.0 --port 30000

# 在另一個終端使用 Client 連接
mineru -p input.pdf -o output/ --backend vlm-http-client -u http://localhost:30000
```

### 進階設定 (LLM Integration)

#### Azure OpenAI 整合設定

**步驟一：建立配置檔案**

在用戶目錄下建立 `~/mineru.json`：

```json
{
  "llm-aided-config": {
    "title_aided": {
      "api_key": "your_azure_openai_api_key",
      "base_url": "https://your-resource-name.openai.azure.com/v1",
      "model": "your-deployment-name",
      "enable": true
    }
  }
}
```

**步驟二：取得 Azure OpenAI 資訊**

1. 登入 Azure Portal
2. 建立 Azure OpenAI 資源
3. 建立部署（Deployment）
4. 取得以下資訊：
   - **API Key**：在「Keys and Endpoint」頁面
   - **Endpoint**：格式為 `https://<resource-name>.openai.azure.com`
   - **Deployment Name**：您建立的部署名稱
   - **API Version**：目前使用預設版本（通常為 `2024-02-15-preview`）

**步驟三：設定 base_url**

`base_url` 格式應為：
```
https://<resource-name>.openai.azure.com/v1
```

**注意**：必須包含 `/v1` 路徑。

**步驟四：設定 model**

`model` 應設定為您的 **Deployment Name**，而非模型名稱（如 `gpt-4`）。

**完整範例**：

```json
{
  "llm-aided-config": {
    "title_aided": {
      "api_key": "a1b2c3d4e5f6g7h8i9j0",
      "base_url": "https://my-mineru-resource.openai.azure.com/v1",
      "model": "gpt-4-deployment",
      "enable": true
    }
  }
}
```

**步驟五：驗證設定**

```bash
# 執行解析，系統會自動使用 LLM 優化標題分級
mineru -p test.pdf -o output/
```

**檢查日誌**：

如果看到以下日誌，表示 LLM 整合成功：
```
llm aided title time: 2.5
```

#### 其他 OpenAI 相容服務整合

MinerU 支援任何符合 OpenAI 協議的服務，只需設定 `base_url` 和 `api_key`：

```json
{
  "llm-aided-config": {
    "title_aided": {
      "api_key": "your_api_key",
      "base_url": "https://your-service.com/v1",
      "model": "your-model-name",
      "enable": true
    }
  }
}
```

### 故障排除 (Troubleshooting)

#### Azure 連線錯誤

**錯誤 401（Unauthorized）**

- **原因**：API Key 錯誤或過期
- **解決方法**：
  1. 檢查 `api_key` 是否正確
  2. 確認 API Key 未過期
  3. 檢查 Azure OpenAI 資源是否啟用

**錯誤 404（Not Found）**

- **原因**：端點 URL 或 Deployment Name 錯誤
- **解決方法**：
  1. 檢查 `base_url` 格式是否正確（必須包含 `/v1`）
  2. 檢查 `model`（Deployment Name）是否存在
  3. 確認 Azure OpenAI 資源區域正確

**錯誤 429（Too Many Requests）**

- **原因**：API 請求過於頻繁
- **解決方法**：
  1. 降低並發請求數
  2. 增加重試間隔
  3. 檢查 Azure OpenAI 配額限制

#### NFS 掛載失敗（K8s 環境）

**錯誤**：`mount.nfs: access denied by server`

- **原因**：NFS 伺服器權限設定不正確
- **解決方法**：
  1. 檢查 NFS 伺服器 export 設定
  2. 確認 K8s Node 可以訪問 NFS 伺服器
  3. 檢查 StorageClass 設定

**錯誤**：`mount.nfs: connection refused`

- **原因**：NFS 伺服器未啟動或網路不通
- **解決方法**：
  1. 檢查 NFS 伺服器狀態
  2. 檢查防火牆設定
  3. 測試網路連線：`telnet <nfs-server> 2049`

#### Python 套件相依性問題

**錯誤**：`ImportError: No module named 'xxx'`

- **原因**：缺少依賴套件
- **解決方法**：
  ```bash
  # 重新安裝
  uv pip install -U "mineru[core]"
  
  # 或安裝特定依賴
  pip install <package-name>
  ```

**錯誤**：`torch` 版本衝突

- **原因**：PyTorch 版本不兼容
- **解決方法**：
  ```bash
  # 檢查 PyTorch 版本
  python -c "import torch; print(torch.__version__)"
  
  # MinerU 要求：torch >= 2.6.0, < 3.0.0
  pip install "torch>=2.6.0,<3.0.0"
  ```

#### GPU 相關問題

**錯誤**：`CUDA out of memory`

- **原因**：GPU 顯存不足
- **解決方法**：
  1. 降低批次大小：設定 `MINERU_MIN_BATCH_INFERENCE_SIZE=256`
  2. 使用 CPU 模式：設定 `MINERU_DEVICE_MODE=cpu`
  3. 關閉部分功能：設定 `MINERU_FORMULA_ENABLE=false`

**錯誤**：`NVIDIA driver version is insufficient`

- **原因**：NVIDIA 驅動版本過舊
- **解決方法**：
  1. 更新 NVIDIA 驅動
  2. 檢查 CUDA 版本兼容性

#### 模型下載問題

**錯誤**：無法從 HuggingFace 下載模型

- **原因**：網路無法訪問 HuggingFace
- **解決方法**：
  ```bash
  # 切換到 ModelScope
  export MINERU_MODEL_SOURCE=modelscope
  mineru-models-download -s modelscope -m all
  ```

**錯誤**：模型下載中斷

- **原因**：網路不穩定
- **解決方法**：
  1. 重新執行下載命令
  2. 使用代理或 VPN
  3. 手動下載模型到本地，設定 `models-dir`

---

## 5. 範例與截圖 (Examples)

### 命令行使用範例

**範例一：基本使用**

```bash
# 解析單一 PDF
mineru -p document.pdf -o output/

# 輸出結果
# output/
#   ├── document.md          # Markdown 格式
#   ├── content_list.json     # JSON 格式
#   ├── middle.json          # 中間格式
#   ├── layout.pdf           # 版面視覺化
#   └── span.pdf             # 文字區塊視覺化
```

**範例二：批量處理**

```bash
# 處理整個資料夾
mineru -p ./pdfs -o ./output

# 處理特定頁面範圍
mineru -p document.pdf -o output/ --start 0 --end 10
```

**範例三：使用 VLM 後端**

```bash
# 使用 VLM 後端（需要 GPU）
mineru -p document.pdf -o output/ --backend vlm-transformers

# 使用 vLLM 加速（需要安裝 vLLM）
mineru -p document.pdf -o output/ --backend vlm-vllm-engine
```

### API 使用範例

**Python 範例**：

```python
from mineru import MinerU

# 初始化
parser = MinerU(backend="pipeline")

# 解析文件
result = parser.parse("document.pdf", output_dir="output/")

# 取得結果
markdown_content = result.get_markdown()
json_content = result.get_json()
```

**cURL 範例**：

```bash
# 上傳並解析
curl -X POST "http://localhost:8000/api/v1/parse" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@document.pdf" \
  -F "backend=pipeline"

# 回應
# {
#   "status": "success",
#   "output_dir": "/path/to/output",
#   "files": ["document.md", "content_list.json"]
# }
```

### 輸出格式範例

**Markdown 輸出範例**：

```markdown
# 第一章 緒論

## 1.1 研究背景

本研究旨在...

## 1.2 研究目的

本研究的目的是...

### 1.2.1 具體目標

- 目標一：...
- 目標二：...

## 公式範例

行內公式：$E=mc^2$

獨立公式：
$$
\sum_{i=1}^{n} x_i = 0
$$

## 表格範例

| 項目 | 數值 | 單位 |
|------|------|------|
| 溫度 | 25   | °C   |
| 壓力 | 1013 | hPa  |

![圖片說明](images/figure1.png)
```

**JSON 輸出範例**：

```json
{
  "pages": [
    {
      "page_idx": 0,
      "blocks": [
        {
          "type": "title",
          "level": 1,
          "content": "第一章 緒論",
          "bbox": [100, 50, 500, 80]
        },
        {
          "type": "text",
          "content": "本研究旨在...",
          "bbox": [100, 100, 500, 200]
        }
      ]
    }
  ]
}
```

### 視覺化結果

> [圖片說明：此處應顯示 layout.pdf 的截圖，展示版面分析結果，標示出標題、段落、表格、圖片的位置]
> [圖片說明：此處應顯示 span.pdf 的截圖，展示文字區塊的識別結果]

### Web UI 使用範例

> [圖片說明：此處應顯示 Gradio WebUI 的截圖，展示上傳文件、選擇後端、查看結果的介面]

**使用步驟**：

1. 打開瀏覽器訪問 `http://localhost:7860`
2. 上傳 PDF 文件
3. 選擇後端（pipeline / vlm-transformers）
4. 點擊「解析」按鈕
5. 查看結果（Markdown 預覽、下載連結）

---

**文件結束**

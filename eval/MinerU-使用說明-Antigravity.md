# MinerU-使用說明.md

## 1. 簡介

MinerU 是一款智能 PDF 解析工具，專注於將複雜文檔（如學術論文、教科書）轉換為機器可讀的 Markdown 與 JSON 格式。它能精準還原多欄排版、提取公式（轉換為 LaTeX）、表格（轉換為 HTML/Markdown）及圖片，是 RAG (Retrieval-Augmented Generation) 系統與 LLM 訓練數據準備的理想前處理工具。

- **專案網頁**：[https://github.com/opendatalab/MinerU](https://github.com/opendatalab/MinerU)
- **參考網頁**：[https://deepwiki.com/opendatalab/MinerU](https://deepwiki.com/opendatalab/MinerU)
- **目標讀者**：負責部署 MinerU 服務與維護的 DevOps 工程師及技術人員。

## 2. 環境變數詳解 (Environment Variables)

以下列出 MinerU 核心配置與 LLM 整合所需的環境變數。請在部署時（`.env` 或 K8s ConfigMap）進行設定。

| 變數名稱 | 預設值 | 必填 | 說明 |
| :--- | :--- | :--- | :--- |
| **MinerU 核心配置** | | | |
| `MINERU_MODEL_SOURCE` | `local` | 否 | 模型來源，設為 `local` 以使用本機下載的模型。 |
| `MINERU_DEVICE_MODE` | `cuda` | 否 | 指定運算裝置 (`cuda`, `cpu`, `mps`, `npu`)。若未設定則自動偵測。 |
| `MINERU_VIRTUAL_VRAM_SIZE` | (自動偵測) | 否 | 虛擬 VRAM 大小 (GB)，用於計算 Batch Size。 |
| `MINERU_MIN_BATCH_INFERENCE_SIZE` | `384` | 否 | 最小批次推論大小，調大可提高吞吐量但消耗更多記憶體。 |
| `MINERU_FORMULA_ENABLE` | `true` | 否 | 是否啟用公式識別。 |
| `MINERU_TABLE_ENABLE` | `true` | 否 | 是否啟用表格識別。 |
| **Azure OpenAI 整合** | | | *(標準 LLM 整合變數)* |
| `AZURE_OPENAI_API_KEY` | - | **是** | Azure OpenAI 的 API Key。 |
| `AZURE_OPENAI_ENDPOINT` | - | **是** | Azure OpenAI 的 Endpoint URL (e.g., `https://my-resource.openai.azure.com/`)。 |
| `OPENAI_API_VERSION` | `2023-05-15` | **是** | API 版本號。 |
| `AZURE_DEPLOYMENT_NAME` | - | **是** | 模型部署名稱 (Deployment Name)。 |

## 3. 安裝與部署 (Installation & Deployment)

### 3.1 本機安裝 (Local Installation)

適用於開發測試或單機執行環境。

**前置需求**：
- Python 3.10 - 3.12
- CUDA (若使用 GPU 加速)

**安裝指令**：
```bash
# 建立虛擬環境 (建議)
conda create -n mineru python=3.10
conda activate mineru

# 安裝核心套件 (包含 VLM 與 Pipeline 功能)
pip install -U "mineru[core]"

# 驗證安裝
mineru --version
```

### 3.2 Kubernetes Helm 部署指引

本專案具備容器化服務特徵（參考 `docker/compose.yaml`），以下提供部署至 Kubernetes 的 Helm Chart 建議配置。

#### 架構設計
- **核心服務**：`mineru-api` (提供 HTTP 解析介面)。
- **服務類型**：`NodePort` (便於內部網路存取)。
- **儲存**：需掛載 NFS Persistent Volume (PV) 以共享模型文件與下載數據。

#### Helm Chart 規格建議

**Deployment (`templates/deployment.yaml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-api
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-api
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-api
    spec:
      containers:
        - name: mineru-api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["mineru-api", "--host", "0.0.0.0", "--port", "8000"]
          ports:
            - containerPort: 8000
          env:
            - name: MINERU_MODEL_SOURCE
              value: "local"
            # 建議透過 Secret 注入 Azure OpenAI 憑證
            - name: AZURE_OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-secrets
                  key: azure-openai-key
          volumeMounts:
            - name: models-storage
              mountPath: /root/.cache/huggingface  # 模型快取路徑
          resources:
            limits:
              nvidia.com/gpu: 1  # 啟用 GPU 支援
      volumes:
        - name: models-storage
          persistentVolumeClaim:
            claimName: {{ .Release.Name }}-pvc
```

**Service (`templates/service.yaml`)**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-api
spec:
  type: NodePort
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30080  # 範例 NodePort
  selector:
    app: {{ .Release.Name }}-api
```

**PVC (`templates/pvc.yaml`)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ .Release.Name }}-pvc
spec:
  accessModes:
    - ReadWriteMany  # 支援多 Pod 共用 (NFS)
  storageClassName: nfs-client
  resources:
    requests:
      storage: 50Gi
```

## 4. 操作指南 (Operations)

### 4.1 基本操作

**啟動 API 服務**：
```bash
mineru-api --host 0.0.0.0 --port 8000
```

**執行單檔解析 (CLI)**：
```bash
# -p 輸入檔案路徑, -o 輸出目錄
mineru -p input/paper.pdf -o output/
```

### 4.2 進階設定：Azure OpenAI 整合

若需啟用 LLM 輔助解析功能，請確保環境變數已正確設定。您也可以透過設定檔 `mineru.json` 進行配置：

```json
{
  "llm-aided-config": {
    "api_key": "您的_AZURE_OPENAI_KEY",
    "api_base": "https://您的_RESOURCE.openai.azure.com/",
    "api_version": "2023-05-15",
    "deployment_name": "gpt-4-32k",
    "model": "gpt-4"
  }
}
```
> [!TIP]
> 建議優先使用環境變數管理敏感資訊 (API Key)，避免將金鑰明文寫入設定檔。

### 4.3 故障排除 (Troubleshooting)

1.  **Azure 連線錯誤 (401 Unauthorized / 404 Not Found)**
    - **檢查點**：確認 `AZURE_OPENAI_ENDPOINT` 是否包含完整的 `https://` 協定頭。
    - **檢查點**：確認 `AZURE_DEPLOYMENT_NAME` 與 Azure Portal 上建立的部署名稱完全一致（注意大小寫）。
    - **檢查點**：確認 `OPENAI_API_VERSION` 是否為該模型支援的版本。

2.  **NFS 掛載失敗 (Kubernetes)**
    - **現象**：Pod 狀態卡在 `ContainerCreating`，Event Log 顯示 `MountVolume.SetUp failed`。
    - **建議**：檢查 NFS Server 是否允許 K8s Node 的 IP 存取；確認 PVC 的 `storageClassName` 是否正確對應叢集內的 StorageClass。

3.  **Python 套件相依性衝突**
    - **現象**：安裝時出現 `conflicting dependencies`。
    - **建議**：使用 `uv pip install` 替代標準 `pip`，或嚴格使用 `conda` 隔離環境。MinerU 對 `torch` 版本較敏感，建議優先參考官方 `requirements.txt` 或 Docker Image 版本。

## 5. 範例與截圖 (Examples)

### 5.1 解析結果範例

**輸入 PDF (雙欄論文)**：
> [圖片說明：此處應顯示原始 PDF 的雙欄學術論文截圖]

**輸出 Markdown**：
```markdown
# 論文標題

## 摘要
這裡顯示被正確提取的摘要內容...

## 1. 介紹
MinerU 能自動將雙欄內容重組為單欄閱讀順序。

$$
E = mc^2
$$
(公式自動轉換為 LaTeX 格式)
```

**輸出 JSON (middle.json)**：
```json
{
  "page_info": {
    "page_no": 0,
    "width": 1240,
    "height": 1754
  },
  "layout_dets": [
    {
      "category_id": 1,
      "category_name": "text",
      "bbox": [100, 100, 500, 200],
      "text": "這是提取出的文字內容..."
    }
  ]
}
```

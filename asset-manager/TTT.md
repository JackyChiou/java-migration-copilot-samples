# 🚀 GitHub Copilot App Modernization for Java — Asset Manager Hands-On Lab

> **實驗目標**：使用 GitHub Copilot App Modernization 將一個基於 Java 8 的 Asset Manager 應用程式，從舊架構（AWS S3 + RabbitMQ + 密碼驗證）遷移至 Azure 原生架構（Azure Blob Storage + Azure Service Bus + Managed Identity），並完成容器化與部署至 Azure。
>
> **預計時間**：約 2 小時

---

## 📋 目錄

- [實驗概覽](#1-實驗概覽)
- [前置準備](#2-前置準備)
- [Step 1：本地運行原始應用](#step-1本地運行原始應用約-5-分鐘)
- [Step 2：安裝 GitHub Copilot App Modernization 擴充套件](#step-2安裝-github-copilot-app-modernization-擴充套件)
- [Step 3：評估 Java 應用程式（Assessment）](#step-3評估-java-應用程式assessment約-5-分鐘)
- [Step 4：升級 Java Runtime 與 Framework](#step-4升級-java-runtime-與-framework約-10-分鐘)
- [Step 5：遷移至 Azure Database for PostgreSQL](#step-5遷移至-azure-database-for-postgresql約-15-分鐘)
- [Step 6：遷移至 Azure Blob Storage](#step-6遷移至-azure-blob-storage約-15-分鐘)
- [Step 7：遷移至 Azure Service Bus](#step-7遷移至-azure-service-bus約-15-分鐘)
- [Step 8：使用 Custom Skills 加入 Health Endpoints](#step-8使用-custom-skills-加入-health-endpoints約-15-分鐘)
- [Step 9：容器化應用程式（Containerize）](#step-9容器化應用程式containerize約-5-分鐘)
- [Step 10：部署至 Azure](#step-10部署至-azure約-40-分鐘)
- [資源清理（Clean Up）](#資源清理clean-up)
- [附錄：備用分支與疑難排解](#附錄備用分支與疑難排解)

---

## 1. 實驗概覽

### 現代化轉換內容

本實驗將把應用程式進行以下現代化轉換：

| 項目 | 遷移前（舊架構） | 遷移後（Azure 原生） |
|------|-------------------|----------------------|
| **Java 版本** | Java 8 | Java 21 |
| **Spring Boot** | 2.x | 3.x |
| **圖片儲存** | AWS S3（密碼驗證） | Azure Blob Storage（Managed Identity） |
| **訊息佇列** | RabbitMQ（密碼驗證） | Azure Service Bus（Managed Identity） |
| **資料庫** | PostgreSQL（密碼驗證） | Azure Database for PostgreSQL（Managed Identity） |
| **健康檢查** | 無 | Spring Boot Actuator Health Endpoints |
| **部署方式** | 本地執行 | 容器化 + Azure Container Apps / AKS |

### 原始架構圖

```
┌──────────┐     ┌─────────────┐     ┌──────────────┐
│   User   │────▶│  Web App    │────▶│   AWS S3     │ (圖片儲存)
└──────────┘     │  (Java 8)   │────▶│  RabbitMQ    │ (訊息佇列)
                 │             │────▶│  PostgreSQL  │ (資料庫)
                 └─────────────┘     └──────────────┘
                        │
                        ▼
                 ┌─────────────┐
                 │   Worker    │ (圖片處理服務)
                 │  (Java 8)   │
                 └─────────────┘
```

> 所有連線皆使用密碼驗證 (password-based authentication)

### 備用分支

如果在任何步驟中遇到問題，可以切換到以下預備分支繼續：

| 分支名稱 | 狀態說明 |
|----------|---------|
| `main` | 原始應用程式狀態 |
| `workshop/java-upgrade` | 完成評估和 Java 升級後的狀態 |
| `workshop/expected` | 完成評估、Java 升級和所有遷移任務後的狀態 |
| `workshop/deployment-expected` | 完成容器化和部署後的狀態 |

---

## 2. 前置準備

### 2.1 GitHub 帳號

- 需要一個啟用 **GitHub Copilot** 的 GitHub 帳號。
- 必須使用 **Pro、Pro+、Business 或 Enterprise** 計畫。

### 2.2 開發環境（二選一）

#### 選項 A：Visual Studio Code（推薦）

| 項目 | 需求 | 下載連結 |
|------|------|----------|
| **VS Code** | 版本 1.101 或更新 | [下載](https://code.visualstudio.com/) |
| **GitHub Copilot** 擴充套件 | 最新版本 | [安裝指南](https://code.visualstudio.com/docs/copilot/setup) |
| **GitHub Copilot App Modernization** 擴充套件 | 最新版本 | [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=vscjava.migrate-java-to-azure) |

#### 選項 B：IntelliJ IDEA

| 項目 | 需求 | 下載連結 |
|------|------|----------|
| **IntelliJ IDEA** | 版本 2023.3 或更新 | [下載](https://www.jetbrains.com/idea/download) |
| **GitHub Copilot** 外掛 | 版本 1.5.59+ | [JetBrains Marketplace](https://plugins.jetbrains.com/plugin/17718-github-copilot) |
| **GitHub Copilot App Modernization** 外掛 | 最新版本 | [JetBrains Marketplace](https://plugins.jetbrains.com/plugin/28791-github-copilot-app-modernization) |

> ⚠️ IntelliJ IDEA 建議額外設定：**Tools** > **GitHub Copilot** > 勾選 **Auto-approve** 和 **Trust MCP Tool Annotations**。
>
> ⚠️ IntelliJ IDEA **不支援** Custom Skills (My Skills) 功能，且目前僅支援 Windows 和 macOS 平台。

### 2.3 開發工具

| 工具 | 用途 | 下載連結 |
|------|------|----------|
| **JDK 8** | 執行原始應用程式 | [Microsoft OpenJDK 8](https://learn.microsoft.com/en-us/java/openjdk/download#openjdk-8) |
| **JDK 21**（或 JDK 25） | 升級後的目標版本 | [Microsoft OpenJDK](https://learn.microsoft.com/en-us/java/openjdk/download) |
| **Maven 3.6.0+** | 建置 Java 專案 | [Maven 下載](https://maven.apache.org/install.html) |
| **Docker** | 本地容器化執行 | [Docker Desktop](https://docs.docker.com/desktop/) |
| **Git** | 版本控制 | [Git 下載](https://git-scm.com/) |

### 2.4 VS Code 設定確認

在 VS Code Settings 中確認以下設定：

```json
{
  "chat.extensionTools.enabled": true
}
```

> 此設定可能受組織策略控制。

### 2.5 GitHub Copilot 設定建議

- **允許匹配公共程式碼的建議**：Copilot 可能會封鎖與公共程式碼相似的生成（如 `pom.xml`），建議開啟此選項。參考：[Enabling suggestions matching public code](https://docs.github.com/en/copilot/managing-copilot/managing-copilot-as-an-individual-subscriber/managing-your-copilot-plan/managing-copilot-policies-as-an-individual-subscriber#enabling-or-disabling-suggestions-matching-public-code)
- **減少確認提示**（可選）：
  - 點擊 **Continue** 旁的下拉箭頭 → 選擇 **Always Allow**
  - 或設定 `chat.tools.autoApprove` 為 `true`
  - 設定 `chat.agent.maxRequests` 為 `128`

---

## Step 1：本地運行原始應用（約 5 分鐘）

### 1.1 Clone 程式碼倉庫

```bash
git clone https://github.com/Azure-Samples/java-migration-copilot-samples.git
cd java-migration-copilot-samples/asset-manager
```

### 1.2 確認環境

```bash
java -version
# 應顯示 1.8.x (Java 8)

mvn -version
# 應顯示 3.6.x 或更高

docker --version
# 確認 Docker 已安裝

# 開啟 Docker Desktop (Startapp.cmd 會下載 image 並使用)
```

### 1.3 啟動應用

啟動腳本會自動使用本地檔案系統取代 S3 來儲存圖片，並透過 Docker 啟動 RabbitMQ 和 PostgreSQL。

**Windows：**
```batch
scripts\startapp.cmd
```

**Linux / macOS：**
```bash
scripts/startapp.sh
```

### 1.4 驗證應用運行

- 等待終端顯示應用啟動成功。
- 在瀏覽器中訪問應用程式，確認 Asset Manager 介面正常運作。
- 嘗試上傳一張圖片，確認基本功能正常。

### 1.5 停止應用

驗證完成後，停止應用程式：

**Windows：**
```batch
scripts\stopapp.cmd
```

**Linux / macOS：**
```bash
scripts/stopapp.sh
```

> ✅ **檢查點**：你已成功在本地運行原始的 Java 8 Asset Manager 應用程式。

---

## Step 2：安裝 GitHub Copilot App Modernization 擴充套件

### 2.1 在 VS Code 中安裝

1. 開啟 VS Code，用 `code .` 開啟 `asset-manager` 目錄。
2. 點擊左側 **Activity Bar** 上的 **Extensions（擴充功能）** 圖示。
3. 在搜尋欄中輸入 `GitHub Copilot app modernization`。
4. 找到擴充套件後，點擊 **Install**。
5. 安裝完成後，VS Code 右下角會顯示成功通知。
6. **重新啟動 VS Code**（重要！）。

### 2.2 確認安裝成功

- 重啟後，在 Activity Bar 中應可看到 **GitHub Copilot App Modernization** 的圖示。
- 點擊該圖示，確認擴充套件面板可以正常開啟。

> ✅ **檢查點**：擴充套件已安裝完成，可以在 Activity Bar 中看到 App Modernization 圖示。

---

## Step 3：評估 Java 應用程式（Assessment）（約 5 分鐘）

評估是現代化流程的第一步，它會分析應用程式的程式碼、配置和相依套件，提供雲端就緒度的深入洞察。

### 3.1 開啟擴充套件面板

1. 確保已在 VS Code 中開啟 `asset-manager` 資料夾。
2. 在左側 **Activity Bar** 中，點擊 **GitHub Copilot App Modernization** 圖示。

### 3.2 啟動評估

1. 在擴充套件面板的 **QUICKSTART** 區塊中，點擊 **Start Assessment** 按鈕。

   > ![📸 參考截圖](doc-media/trigger-assessment.png)

2. 等待評估完成（約需數分鐘）。

### 3.3 查看評估報告

1. 評估完成後，會自動開啟 **Assessment Report** 分頁。
2. 報告提供**分類檢視**的雲端就緒度問題和建議解決方案。
3. 點擊 **Issues** 分頁，查看：
   - 需要修復的問題清單
   - 每個問題的建議解決方案（Solutions）
   - 可直接執行的遷移任務（Run Task 按鈕）

### 3.4 了解評估結果

報告中你應該會看到以下主要問題類別：

| 問題類別 | 說明 |
|----------|------|
| **Java Version Upgrade** | Java 8 需升級至 21 |
| **Database Migration** | PostgreSQL 需遷移至 Azure Database for PostgreSQL |
| **Storage Migration (AWS S3)** | AWS S3 需遷移至 Azure Blob Storage |
| **Messaging Service Migration** | RabbitMQ 需遷移至 Azure Service Bus |

> ✅ **檢查點**：Assessment Report 已生成，可看到各類遷移問題和對應的解決方案。

---

## Step 4：升級 Java Runtime 與 Framework（約 10 分鐘）

將 Java 從 8 升級到 21，同時將 Spring Boot 從 2.x 升級到 3.x。

### 4.1 啟動 Java 升級

1. 在 Assessment Report 的 **Issues** 分頁中，找到底部的 **Java Upgrade** 表格。
2. 在 **Java Version Upgrade** 項目右側，點擊 **Run Task** 按鈕。

   > ![📸 參考截圖](doc-media/java-upgrade.png)

### 4.2 配合 Agent 執行

1. 點擊 Run Task 後，**Copilot Chat** 面板將以 **Agent Mode** 開啟。
2. Agent 會自動：
   - **建立新分支**（check out a new branch）
   - **升級 JDK 版本**（Java 8 → Java 21）
   - **升級 Spring/Spring Boot Framework**（2.x → 3.x）
3. 對於 Agent 的所有請求，點擊 **Allow** 允許執行。

> 💡 **提示**：升級工具也支援升級至 JDK 25（最新 LTS 版本）。如需升級至 25，可在 Copilot Chat 中點擊生成的訊息，將目標 Java 版本修改為 25，然後點擊 **Send**。

### 4.3 等待升級完成

- Agent 會自動執行程式碼修改、建置驗證和測試。
- 如果 Agent 暫停等待確認，在 Chat 中輸入 `continue` 或 `yes`。

### 4.4 確認結果

- 升級完成後，檢視 Agent 生成的變更摘要。
- 確認建置成功且測試通過。

> ✅ **檢查點**：Java 已從 8 升級至 21，Spring Boot 已從 2.x 升級至 3.x。
>
> ⚠️ 若此步驟遇到問題，可切換至 `workshop/java-upgrade` 分支繼續後續步驟。

---

## Step 5：遷移至 Azure Database for PostgreSQL（約 15 分鐘）

將資料庫連線從密碼驗證的 PostgreSQL 遷移至使用 Managed Identity 的 Azure Database for PostgreSQL Flexible Server。

### 5.1 啟動遷移任務

1. 回到 Assessment Report 的 **Issues** 分頁。
2. 在 Solution 清單中，找到 **Migrate to Azure Database for PostgreSQL (Spring)**。
3. 點擊該項目右側的 **Run Task** 按鈕。

   > ![📸 參考截圖](doc-media/confirm-postgresql-solution.png)

### 5.2 Copilot Agent 自動執行

Agent 開啟後會依序執行以下步驟：

1. **分析專案** — 理解現有程式碼結構和資料庫連線模式。
2. **生成計畫** — 自動建立 `plan.md`（遷移計畫）和 `progress.md`（進度追蹤）。
3. **建立新分支** — 檢查版本控制狀態並 check out 新遷移分支。
4. **執行程式碼變更** — 修改資料庫連線配置、依賴套件等。
5. **驗證與修復迴圈** — 自動執行一系列驗證：

   | 驗證項目 | 說明 |
   |----------|------|
   | **CVE Validation** | 偵測目前依賴套件中的已知漏洞並修復 |
   | **Build Validation** | 嘗試解決任何建置錯誤 |
   | **Consistency Validation** | 分析程式碼的功能一致性 |
   | **Test Validation** | 執行單元測試並自動修復失敗項目 |
   | **Completeness Validation** | 捕捉初始遷移中遺漏的項目並修復 |

6. **生成摘要** — 完成後生成 `summary.md` 摘要報告。

### 5.3 操作提示

- 對於 Agent 的所有 tool call 請求，點擊 **Allow** 允許。
- 如果 Agent 暫停等待確認，輸入 `Continue` 繼續。

### 5.4 確認並套用變更

1. 所有驗證完成後，檢視 Agent 提出的程式碼變更。
2. 確認變更內容正確無誤後，點擊 **Keep** 套用變更。

> ✅ **檢查點**：PostgreSQL 連線已遷移至使用 Managed Identity 的 Azure Database for PostgreSQL。

---

## Step 6：遷移至 Azure Blob Storage（約 15 分鐘）

將圖片儲存從 AWS S3 遷移至 Azure Blob Storage。

### 6.1 啟動遷移任務

1. 回到 Assessment Report。
2. 在 **Storage Migration (AWS S3)** 列中，找到 **Migrate from AWS S3 to Azure Blob Storage**。
3. 點擊右側的 **Run Task** 按鈕。

### 6.2 Agent 自動執行

- 操作流程與 Step 5 的 PostgreSQL 遷移相同。
- Agent 會自動：
  - 生成 `plan.md` 和 `progress.md`
  - 建立新遷移分支
  - 將所有 AWS S3 SDK 呼叫替換為 Azure Blob Storage SDK
  - 將 access key/secret key 驗證替換為 Managed Identity
  - 執行驗證與修復迴圈（CVE、Build、Consistency、Test、Completeness）
  - 生成 `summary.md`

### 6.3 確認並套用變更

- 檢視程式碼變更，點擊 **Keep** 套用。

> ✅ **檢查點**：AWS S3 圖片儲存已遷移至 Azure Blob Storage + Managed Identity。

---

## Step 7：遷移至 Azure Service Bus（約 15 分鐘）

將訊息佇列從 RabbitMQ 遷移至 Azure Service Bus。

### 7.1 啟動遷移任務

1. 回到 Assessment Report。
2. 在 **Messaging Service Migration (Spring AMQP RabbitMQ)** 列中，找到 **Migrate from RabbitMQ(AMQP) to Azure Service Bus**。
3. 點擊右側的 **Run Task** 按鈕。

### 7.2 Agent 自動執行

- 操作流程與 Step 5、Step 6 相同。
- Agent 會自動將 RabbitMQ 的 AMQP 互動邏輯替換為 Azure Service Bus 等效實作，保持相同的訊息模式和語意。

### 7.3 確認並套用變更

- 檢視程式碼變更，點擊 **Keep** 套用。

> ✅ **檢查點**：RabbitMQ 訊息佇列已遷移至 Azure Service Bus + Managed Identity。

---

## Step 8：使用 Custom Skills 加入 Health Endpoints（約 15 分鐘）

使用自訂技能（Custom Skills）為應用程式加入 Spring Boot Actuator 健康檢查端點，準備好部署至 Azure Container Apps。

> ⚠️ **注意**：IntelliJ IDEA 外掛不支援 Custom Skills (My Skills) 功能。使用 IntelliJ 的使用者可跳過此步驟。

### 8.1 建立 Custom Skill

1. 在左側 Activity Bar 中，開啟 **GitHub Copilot App Modernization** 擴充套件面板。
2. 將滑鼠移至 **TASKS** 區塊上方，點擊 **Create a Custom Skill** 按鈕。

   > ![📸 參考截圖](doc-media/create-formula-from-source-control.png)

### 8.2 填寫 Skill 表單

在開啟的 **Create a Skill** 表單中，填入以下內容：

| 欄位 | 輸入值 |
|------|--------|
| **Skill Name** | `expose-health-endpoint` |
| **Skill Description** | `This skill helps add Spring Boot Actuator health endpoints for Azure Container Apps deployment readiness.` |
| **Skill Content** | `You are a Spring Boot developer assistant, follow the Spring Boot Actuator documentation to add basic health endpoints for Azure Container Apps deployment.` |

### 8.3 新增參考資源

1. 點擊 **Add Resources**。
2. 貼上 Spring Boot Actuator 官方文件連結, 存成 PDF 後加入 Resource：
   ```
   https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
   ```

   > ![📸 參考截圖](doc-media/health-endpoint-task.png)

### 8.4 儲存並執行

1. 點擊 **Save** 儲存 Skill。
2. 自訂 Skill 會出現在 **TASKS** > **My Skills** 區塊。
3. 點擊 **Run** 執行。

### 8.5 Agent 自動執行

- Copilot Chat 以 Agent Mode 開啟。
- Agent 會自動：
  - 生成遷移計畫
  - 建立新分支
  - 在 `web` 和 `worker` 模組中加入 Spring Boot Actuator 依賴
  - 配置 Health Endpoint
  - 執行驗證與修復迴圈

### 8.6 確認並套用變更

- 檢視程式碼變更，點擊 **Keep** 套用。

> ✅ **檢查點**：Health endpoints 已透過 Spring Boot Actuator 加入，應用程式已準備好進行容器化。
>
> ⚠️ 如果前面的遷移步驟遇到問題，可切換至 `workshop/expected` 分支繼續。

---

## Step 9：容器化應用程式（Containerize）（約 5 分鐘）

將遷移完成的 `web` 和 `worker` 模組容器化，為雲端部署做準備。

### 9.1 啟動容器化任務

1. 在擴充套件面板的 **TASKS** 區塊中，展開 **Common Tasks** > **Containerize Tasks**。
2. 點擊 **Containerize Application** 的 Run 按鈕。

   > ![📸 參考截圖](doc-media/containerization-run-task.png)

### 9.2 Agent 分析並建立計畫

1. Copilot Chat 以 Agent Mode 開啟，預設 prompt 已自動填入。
2. Agent 分析工作區，建立 `containerization-plan.copilotmd` 檔案。

   > ![📸 參考截圖](doc-media/containerization-plan.png)

### 9.3 配合 Agent 執行

1. 查看計畫中的 **Execution Steps**。
2. 點擊 **Continue / Allow** 允許 Agent 執行命令。
3. Agent 會利用 **Container Assist** 的 agentic tools 來：
   - 生成 Dockerfile（web 和 worker 各一個）
   - 建置 Docker 映像檔
   - 自動修復建置錯誤（如有）

### 9.4 確認並套用變更

- 檢視生成的 Dockerfile 和相關配置。
- 點擊 **Keep** 套用。

> ✅ **檢查點**：web 和 worker 模組的 Dockerfile 已生成，Docker 映像檔建置成功。

---

## Step 10：部署至 Azure（約 40 分鐘）

將容器化後的應用程式部署至 Azure，預設目標為 Azure Container Apps。

### 10.1 啟動部署任務

1. 在擴充套件面板的 **TASKS** 區塊中，展開 **Common Tasks** > **Deployment Tasks**。
2. 點擊 **Provision Infrastructure and Deploy to Azure** 的 Run 按鈕。

   > ![📸 參考截圖](doc-media/deployment-run-task.png)

### 10.2 選擇目標服務

- 預設的 Azure 託管服務是 **Azure Container Apps**。
- 如果想改用 **Azure Kubernetes Service (AKS)**：
  - 點擊 Copilot Chat 中的 prompt。
  - 將最後一句修改為 `Hosting service: AKS`。

   > ![📸 參考截圖](doc-media/deployment-prompt.png)

### 10.3 Agent 建立部署計畫

1. 點擊 **Continue / Allow** 允許 Agent 分析專案。
2. Agent 會建立 `plan.copilotmd` 檔案，包含：
   - **Azure 資源架構圖** — 視覺化部署架構
   - **建議的 Azure 資源** — 各服務的配置與安全設定
   - **執行步驟** — 逐步的部署流程

### 10.4 檢視計畫並開始部署

1. 仔細查看架構圖、資源配置和執行步驟。
2. 點擊 **Keep** 儲存計畫。
3. 在 Chat 中輸入 **`Execute the plan`** 開始部署。

   > ![📸 參考截圖](doc-media/deployment-execute.png)

### 10.5 配合 Agent 完成部署

1. 部署過程中，點擊 **Continue / Allow** 或在終端機中輸入 **`y`** / **`yes`**。
2. Agent 會執行：
   - 建立和執行基礎設施佈建腳本
   - 建立和執行部署腳本
   - 修復可能出現的錯誤
   - 完成部署

   > ![📸 參考截圖](doc-media/deployment-progress.png)

3. 可在 `progress.copilotmd` 中查看部署狀態。

> ⚠️ **重要**：佈建或部署腳本執行時 **請勿中斷**（DO NOT interrupt）！

### 10.6 驗證部署成功

- 部署完成後，Agent 會提供應用程式的 URL。
- 在瀏覽器中訪問該 URL，確認 Asset Manager 在 Azure 上正常運作。

> ✅ **檢查點**：應用程式已成功部署至 Azure Container Apps（或 AKS）！
>
> ⚠️ 如果部署遇到問題，可參考 `workshop/deployment-expected` 分支中 `/.azure` 目錄的腳本進行比對和排錯。

---

## 資源清理（Clean Up）

實驗完成後，刪除所有相關 Azure 資源以避免產生費用。

**Windows：**
```batch
scripts\cleanup-azure-resources.cmd -ResourceGroupName <your-resource-group-name>
```

**Linux / macOS：**
```bash
scripts/cleanup-azure-resources.sh -ResourceGroupName <your-resource-group-name>
```

> 如果使用 GitHub Codespaces 進行實驗，也請刪除 Codespace 環境：前往你 Fork 的倉庫 → **Code** > **Codespaces** > **Delete**。

---

## 附錄：備用分支與疑難排解

### A. 備用分支快速切換

```bash
# 跳過 Java 升級步驟，直接進行遷移任務
git checkout workshop/java-upgrade

# 跳過所有遷移任務，直接進行容器化和部署
git checkout workshop/expected

# 查看完成部署後的預期結果
git checkout workshop/deployment-expected
```

### B. 常見問題

| 問題 | 解決方案 |
|------|---------|
| Agent 停在計畫列表後不進行程式碼變更 | 在 Chat 中輸入 `yes` 或 `continue` |
| Agent 頻繁要求確認 | 設定 `chat.tools.autoApprove` 為 `true`，`chat.agent.maxRequests` 為 `128` |
| 看不到 MCP 伺服器工具 | 點擊擴充套件面板的 **Refresh** 按鈕 |
| 程式碼生成不穩定 | 重試即可；AI 偶爾會產生不穩定輸出 |
| `pom.xml` 修改被 Copilot 封鎖 | 開啟「允許匹配公共程式碼的建議」設定 |
| MCP Server 問題排查 | 檢查 Log：`%USERPROFILE%/.ghcp-appmod-java/logs` |
| 部署腳本執行失敗 | 比對 `workshop/deployment-expected` 分支的 `/.azure` 目錄 |

### C. 預定義遷移任務完整清單

| # | 任務名稱 | 說明 |
|---|----------|------|
| 1 | Spring RabbitMQ → Azure Service Bus | 將 Spring AMQP/JMS + RabbitMQ 遷移至 Azure Service Bus |
| 2 | Managed Identities for Database | 將資料庫改為 Managed Identity 驗證 |
| 3 | Managed Identities for Credentials | 將 Event Hubs/Service Bus 改為 Managed Identity |
| 4 | AWS S3 → Azure Blob Storage | 將 AWS S3 操作遷移至 Azure Blob Storage |
| 5 | Logging to Console | 將檔案日誌轉為 Console 日誌 |
| 6 | Local File I/O → Azure Storage File Share | 將本地檔案存取遷移至 Azure Storage File Share |
| 7 | Java Mail → Azure Communication Service | 將 SMTP 遷移至 Azure Communication Services |
| 8 | Secrets → Azure Key Vault | 將密鑰和憑證遷移至 Azure Key Vault |
| 9 | User Auth → Microsoft Entra ID | 將 LDAP 驗證遷移至 Entra ID |
| 10 | SQL Dialect: Oracle → PostgreSQL | 轉換 Oracle SQL 語法至 PostgreSQL |
| 11 | AWS Secret Manager → Azure Key Vault | 從 AWS Secret Manager 遷移至 Azure Key Vault |
| 12 | ActiveMQ → Azure Service Bus | 從 Apache ActiveMQ 遷移至 Azure Service Bus |
| 13 | AWS SQS → Azure Service Bus | 從 AWS SQS 遷移至 Azure Service Bus |

---

## 📝 實驗流程快速總覽

```
 Step 1: 本地運行原始應用 ──────────────────────── [~5 min]
    │   Clone → 確認環境 → startapp → 驗證功能
    ▼
 Step 2: 安裝 App Modernization 擴充套件 ────────── [~2 min]
    │   VS Code Extension → 重啟 VS Code
    ▼
 Step 3: 評估 (Assessment) ────────────────────── [~5 min]
    │   Start Assessment → 查看報告 → 了解問題清單
    ▼
 Step 4: 升級 Java Runtime & Framework ─────────── [~10 min]
    │   Java 8→21, Spring Boot 2.x→3.x
    ▼
 Step 5: 遷移 PostgreSQL → Azure DB for PostgreSQL ─ [~15 min]
    │   密碼驗證 → Managed Identity
    ▼
 Step 6: 遷移 AWS S3 → Azure Blob Storage ──────── [~15 min]
    │   Access Key → Managed Identity
    ▼
 Step 7: 遷移 RabbitMQ → Azure Service Bus ─────── [~15 min]
    │   密碼驗證 → Managed Identity
    ▼
 Step 8: 加入 Health Endpoints (Custom Skill) ──── [~15 min]
    │   Spring Boot Actuator
    ▼
 Step 9: 容器化 (Containerize) ────────────────── [~5 min]
    │   生成 Dockerfile → 建置 Docker Image
    ▼
 Step 10: 部署至 Azure ────────────────────────── [~40 min]
    │   Provision Infrastructure → Deploy → 驗證
    ▼
 Clean Up: 刪除 Azure 資源 ────────────────────── [~2 min]
```

---

## 📚 參考資源

- [Azure Samples: java-migration-copilot-samples](https://github.com/Azure-Samples/java-migration-copilot-samples/tree/main/asset-manager)
- [GitHub Copilot App Modernization for Java — 官方文件](https://learn.microsoft.com/en-us/azure/developer/java/migration/migrate-github-copilot-app-modernization-for-java)
- [預定義任務列表](https://learn.microsoft.com/en-us/azure/developer/java/migration/migrate-github-copilot-app-modernization-for-java-predefined-tasks)
- [建立自訂任務快速入門](https://learn.microsoft.com/en-us/azure/developer/java/migration/migrate-github-copilot-app-modernization-for-java-quickstart-create-and-apply-your-own-task)
- [FAQ 與疑難排解](https://learn.microsoft.com/en-us/azure/developer/java/migration/migrate-github-copilot-app-modernization-for-java-faq)

---

> **版本**：v1.0 | **最後更新**：2026-03-09 | **來源倉庫**：Azure-Samples/java-migration-copilot-samples

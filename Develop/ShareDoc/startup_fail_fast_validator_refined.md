# Startup Fail-Fast Validator

> 讓服務在錯誤條件下提早、明確地停止啟動，避免在不完整或不正確的狀態下進入 Kubernetes runtime。

**Document purpose:** 本文件說明 Startup Fail-Fast Validator 的概念、適用場景、檢查範圍，以及在 Kubernetes 部署環境中的角色。內容聚焦於 RD 團隊在開發與部署服務時，如何理解「啟動階段驗證」對系統穩定性的影響。

**Core statement:**

> A service should either start correctly or not start at all.

---

## 1. Why Startup Fail-Fast Validator Matters

在 Kubernetes 部署環境中，`Pod Running` 通常只代表 container process 仍在執行。這個狀態不直接等同於 application 已具備正確服務條件。服務是否能正確運行，仍取決於 configuration、secret、external dependency、runtime resource 等條件是否完整且有效。

Startup Fail-Fast Validator 是一種啟動階段的檢查機制。它在服務正式接收流量之前，確認必要條件是否成立。若條件不成立，application 會以明確原因停止啟動，讓問題暴露在 deployment 階段，而不是延後到使用者請求進來後才發生。

這種設計的重點不是讓系統更容易失敗，而是降低服務在不正確狀態下繼續運行的機率。

### 1.1 Common Deployment Mismatch

| 表面狀態 | 可能存在的實際問題 |
|---|---|
| Pod Running | 必要環境變數不存在，但程式使用 default value 繼續啟動 |
| Container started | DB、Redis、MQTT、Blob endpoint 設定錯誤 |
| Readiness passed | Pod 已被導入流量，但 application dependency 尚未就緒 |
| API 有回應 | 只有部分功能正常，核心流程第一次被呼叫時才失敗 |
| Log 沒有明確錯誤 | 問題來源可能分散在設定、權限、網路或依賴服務 |

### 1.2 Engineering Context

Fail-fast 的核心概念，是在錯誤發生時讓錯誤立即且清楚地呈現。Jim Shore 在 *Fail Fast* 一文中提到，錯誤若能即時且可見地失敗，通常更容易被發現與修正。

在服務部署情境中，startup validation 可以視為 application 的啟動前條件檢查。其角色類似於 pre-flight checklist：系統在正式承擔工作負載之前，先確認基本條件是否成立。

這類檢查特別適合 Kubernetes 環境，因為 Kubernetes 能夠觀察 process exit、container restart、readiness 狀態與 logs。當 application 在啟動階段明確失敗時，部署平台可以更早暴露異常。

---

## 2. The Fail-Fast Mindset

Fail-fast 並不表示 application 對任何小問題都直接停止。它描述的是：當服務缺少正確運行的必要條件時，系統選擇在啟動階段暴露問題，而不是在 runtime 中維持表面正常。

在實務上，錯誤處理方式可分為三種典型模式。

| 模式 | 行為 | 系統影響 |
|---|---|---|
| Ignore | 忽略錯誤並繼續啟動 | 問題被隱藏，後續可能進入 production runtime |
| Fail Late | 啟動時未檢查，直到 request 觸發相關流程才失敗 | 錯誤與觸發條件距離較遠，排查成本較高 |
| Fail Fast | 啟動時檢查必要條件，不成立時停止啟動 | 問題較早暴露，錯誤位置通常較明確 |

### 2.1 Fail-Late Example

```text
1. Service starts successfully.
2. First user request calls Azure Blob upload.
3. The request fails because AZURE_STORAGE_KEY is missing.
4. The failure is observed during runtime traffic.
```

此情境中，設定錯誤存在於服務啟動前，但直到使用者請求觸發相關功能時才出現。錯誤發生時間與錯誤來源之間存在延遲，因此排查範圍通常較大。

### 2.2 Fail-Fast Example

```text
1. Service starts.
2. Startup validator checks AZURE_STORAGE_KEY.
3. Missing key is reported as STARTUP_VALIDATION_FAILED.
4. Application exits.
5. Kubernetes does not route traffic to this Pod.
```

此情境中，錯誤在 application startup 階段被揭露。Kubernetes 可以透過 container exit、restart、CrashLoopBackOff、logs 等訊號呈現部署異常。

### 2.3 Relationship with Industry Practices

Startup Fail-Fast Validator 與多個常見工程原則有關：

- **12-Factor App - Config**：configuration 應與 code 分離，並由部署環境注入。Startup validation 可用於確認注入結果是否完整且合法。
- **SRE release safety**：release safety 關注部署變更造成的風險。啟動階段驗證可降低錯誤設定進入 runtime 的機率。
- **Kubernetes probes**：probes 用於 container 生命週期判斷；startup validator 則屬於 application 對自身啟動條件的檢查。

> Fail fast is not about being fragile. It is about refusing to run in an invalid state.

---

## 3. What Should Be Validated

本文件將 Startup Fail-Fast Validator 聚焦在四類檢查：configuration、secret、dependency、runtime。這四類通常與部署環境、外部資源、執行條件直接相關，也最容易在 Kubernetes deployment 中形成啟動後才暴露的問題。

Domain rule 屬於資料正確性、資料治理或業務規則驗證範圍。Observability 則屬於 programming principle、平台規範或 service standard 範圍。因此本文件不將這兩類列為 Startup Fail-Fast Validator 的主要檢查主軸。

### 3.1 Configuration Validation

Configuration validation 用於確認必要設定存在、格式正確、值域合理，並且與部署環境一致。這類檢查通常處理環境變數、application properties、feature flags、timeout、retry、profile 等設定。

| 檢查項目 | 檢查重點 | 條件不成立時的典型行為 |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | 是否屬於允許的 profile，例如 `dev`、`sit`、`prod` | 停止啟動並標示允許值 |
| `SERVER_PORT` | 是否為合法 port，且與 container / service 設定一致 | 停止啟動 |
| `FEATURE_FLAG_*` | 功能開關是否與必要 dependency 一致 | 停止啟動或標示該功能不可用 |
| `TIMEOUT` / `RETRY` | 數值是否為合理範圍，例如不可為負數或過大 | 停止啟動 |
| External endpoint | URL 格式、scheme、host 是否合理 | 停止啟動或標示 dependency unavailable |

Configuration validation 的價值在於，application 不會在錯誤 profile、錯誤 timeout、錯誤 endpoint 或不完整 feature flag 組合下持續運行。

### 3.2 Secret Validation

Secret validation 用於確認必要 secret 已被正確注入。它通常只檢查 secret 是否存在、長度是否符合最低要求、格式是否符合基本條件。secret value 本身不適合輸出到 log、exception message 或 structured log field。

| 檢查項目 | 可以檢查的內容 | 不適合出現在輸出的內容 |
|---|---|---|
| `JWT_SECRET` | 是否存在、長度是否足夠 | secret value |
| `DB_PASSWORD` | 是否存在 | password 明文 |
| `AZURE_STORAGE_KEY` | 是否存在、是否符合基本格式 | key value |
| OAuth client secret | 是否存在、格式是否合理 | client secret 明文 |
| Certificate private key | 檔案是否存在、是否可讀 | private key content |

Secret validation 的錯誤訊息通常包含 secret name、錯誤類型與修正方向。例如：

```text
STARTUP_VALIDATION_FAILED code=SECRET_MISSING message="Required secret AZURE_STORAGE_KEY is missing"
```

這種訊息可以指出問題來源，同時避免洩漏敏感資訊。

### 3.3 Dependency Validation

Dependency validation 用於確認 application 啟動所需的外部依賴是否可用。依賴檢查需要區分「核心依賴」與「非核心依賴」。核心依賴不可用時，服務通常無法提供主要功能；非核心依賴不可用時，服務可能仍可進入 degraded mode。

| 依賴 | 常見檢查內容 | 是否通常會 fail-fast |
|---|---|---|
| Database | connection、schema migration 狀態、必要權限 | 若為核心資料來源，通常會 |
| Redis | ping、connection、必要 DB index 或 keyspace 設定 | 取決於服務功能 |
| MQTT Broker | URL 格式、network reachability、必要時建立連線 | 若控制面流程依賴 MQTT，通常會 |
| Azure Blob | container 是否存在、權限是否可讀寫 | 若上傳 / 下載為核心功能，通常會 |
| External API | endpoint 格式、authentication 設定、health endpoint | 取決於 API 是否為核心流程 |

Dependency validation 的粒度需要與服務角色一致。對於啟動時必須存在的依賴，fail-fast 可以避免服務在不可用狀態下接收流量。對於非核心依賴，degraded mode 需要明確定義其功能邊界與對外行為。

### 3.4 Runtime Validation

Runtime validation 用於確認 container runtime 條件符合 application 假設。這類問題通常與程式邏輯無關，但會直接影響 production 運行。

常見 runtime 檢查包含：

| 檢查項目 | 檢查內容 | 常見風險 |
|---|---|---|
| Required files | certificate、license、template 是否存在且可讀 | 啟動後第一次使用檔案時失敗 |
| Writable directories | temp、cache、upload staging 是否可寫 | runtime 無法產生暫存檔 |
| Volume mount | mount path 是否存在、是否為預期內容 | 掛載失敗但程式仍使用空目錄 |
| Timezone / locale | 是否符合服務假設 | 時間計算、排程、log 時間不一致 |
| CPU / memory limit | 是否符合服務需求 | OOMKill、過度 throttling |
| Certificate path | 憑證檔案是否存在且格式可讀 | TLS / mTLS / external API connection 失敗 |

Runtime validation 的目標，是在 application 使用這些資源之前，先確認執行環境符合服務假設。

---

## 4. Working with Kubernetes

Startup Fail-Fast Validator 與 Kubernetes probes 屬於不同層次的機制。Validator 是 application 內部的啟動條件檢查；probes 是 Kubernetes 對 container / Pod 生命週期的外部觀察機制。兩者可以互補，但功能不完全相同。

### 4.1 Responsibility Split

| 機制 | 回答的問題 | 主要時機 | 失敗結果 |
|---|---|---|---|
| Startup Validator | application 的設定、密鑰、依賴、runtime 條件是否足以正確啟動 | Application startup | application 主動退出 |
| Startup Probe | container 是否已完成啟動流程 | Container startup phase | 到達失敗門檻後重啟 container |
| Readiness Probe | Pod 目前是否可以接收流量 | Before / during serving traffic | Service 不將流量導到此 Pod |
| Liveness Probe | container 是否卡死或進入不可恢復狀態 | Runtime | kubelet 重啟 container |

### 4.2 Failure Flow with Startup Validator

```text
Startup Validator failed
-> Application exits with clear error
-> Container exits
-> Pod enters restart cycle / CrashLoopBackOff
-> Logs show validation failure reason
-> Configuration, secret, dependency, or runtime condition is corrected
```

這種流程代表 deployment 在接收正式流量前已暴露問題。Kubernetes 的重啟與狀態回報可以協助團隊定位啟動失敗原因。

### 4.3 Risk of Late Failure

```text
Application starts with wrong config
-> Pod is Running
-> Readiness passes too early
-> Traffic comes in
-> Partial failure happens during runtime
-> Users or upstream services observe failures
```

此流程中，問題並未在 deployment 階段被攔截，而是在服務已參與 traffic routing 後才呈現。這會使錯誤來源更難與部署變更直接連結。

### 4.4 Boundary Between Validator and Probes

Startup validator 不等同於 readiness probe。Readiness probe 的重點是 traffic gate，用於判斷當前 Pod 是否適合接收流量。Startup validator 的重點是 application 是否具備啟動所需的基本條件。

常見分工如下：

| 檢查內容 | 較適合放置位置 |
|---|---|
| 必要 env 是否存在 | Startup Validator |
| secret 是否被注入 | Startup Validator |
| DB schema 是否符合啟動需求 | Startup Validator 或 migration process |
| Pod 是否可以接收流量 | Readiness Probe |
| process 是否卡死 | Liveness Probe |
| 慢啟動服務是否已完成初始化 | Startup Probe |

> Pod Running does not mean the application is correct. It only means the container process is alive.

---

## 5. Implementation Structure

Startup Fail-Fast Validator 可以設計成可維護、可測試、可擴充的啟動安全機制。若所有檢查都集中在單一大型 class 中，後續維護成本通常會上升。因此常見做法是將檢查抽象為多個 `StartupCheck`，再由統一的 `StartupValidator` 收集結果。

以下範例以 Java / Spring Boot 為例，但相同概念也可應用於其他 backend framework。

### 5.1 Design Characteristics

| 特性 | 說明 |
|---|---|
| Early validation | validation 發生於 application startup 初期 |
| Explicit validation | 必要設定與依賴以明確條件檢查，而非依賴隱性 default |
| Clear failure reason | validation failure 包含 error code、設定名稱與錯誤原因 |
| Secret-safe logging | log 中只出現 secret name，不出現 secret value |
| Required / optional separation | 核心依賴與非核心依賴具有不同失敗策略 |
| Testability | 缺少 env、缺少 secret、依賴不可用等情境可被測試 |

### 5.2 Example Interface

```java
public interface StartupCheck {
    List<ValidationError> validate();
}
```

每一種檢查可以獨立實作。例如：

```java
@Component
public class AzureStorageStartupCheck implements StartupCheck {

    private final AzureStorageProperties properties;

    @Override
    public List<ValidationError> validate() {
        List<ValidationError> errors = new ArrayList<>();

        if (isBlank(properties.accountName())) {
            errors.add(new ValidationError(
                "CONFIG_MISSING",
                "AZURE_STORAGE_ACCOUNT is required"
            ));
        }

        if (isBlank(properties.accountKey())) {
            errors.add(new ValidationError(
                "SECRET_MISSING",
                "AZURE_STORAGE_KEY is required"
            ));
        }

        return errors;
    }
}
```

### 5.3 Example Aggregator

```java
@Component
public class StartupValidator implements ApplicationRunner {

    private final List<StartupCheck> checks;

    public StartupValidator(List<StartupCheck> checks) {
        this.checks = checks;
    }

    @Override
    public void run(ApplicationArguments args) {
        List<ValidationError> errors = checks.stream()
            .flatMap(check -> check.validate().stream())
            .toList();

        if (!errors.isEmpty()) {
            errors.forEach(error -> log.error(
                "STARTUP_VALIDATION_FAILED code={} message={}",
                error.code(),
                error.message()
            ));

            throw new StartupValidationException(errors);
        }
    }
}
```

### 5.4 Example Error Model

```java
public record ValidationError(
    String code,
    String message
) {}
```

常見 error code 可以包含：

| Error code | 意義 |
|---|---|
| `CONFIG_MISSING` | 必要設定不存在 |
| `CONFIG_INVALID` | 設定格式或值域不合法 |
| `SECRET_MISSING` | 必要 secret 未注入 |
| `DEPENDENCY_UNAVAILABLE` | 核心依賴不可用 |
| `RUNTIME_RESOURCE_MISSING` | runtime 必要檔案、目錄或 mount 不存在 |
| `RUNTIME_RESOURCE_NOT_WRITABLE` | runtime 目錄或檔案權限不符合需求 |

### 5.5 Practical Integration Points

在實際專案中，Startup Fail-Fast Validator 通常會與以下項目整合：

| 整合點 | 作用 |
|---|---|
| Application startup lifecycle | 在服務啟動初期執行 validation |
| Configuration properties | 使用 typed properties 取得設定值 |
| Secret provider | 驗證 secret 是否已被注入 |
| Kubernetes manifest / Helm chart | 對應 env、secret、volume、resource limit |
| CI/CD smoke test | 在部署前檢查基本設定組合 |
| Unit / integration tests | 覆蓋 validation success / failure 情境 |

---

## References

- Jim Shore, *Fail Fast*, IEEE Software, 2004. Hosted by Martin Fowler: <https://martinfowler.com/ieeeSoftware/failFast.pdf>
- The Twelve-Factor App, *Config*: <https://12factor.net/config>
- Kubernetes Documentation, *Liveness, Readiness, and Startup Probes*: <https://kubernetes.io/docs/concepts/workloads/pods/probes/>
- Google SRE Workbook, *Canary Release: Deployment Safety and Efficiency*: <https://sre.google/workbook/canarying-releases/>

---

**Reviewed by: Amer Wu**

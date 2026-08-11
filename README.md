# ADS 1.2 → GCC AI Porting Tool

目標：Windows + MSYS2 + GNU make + arm-none-eabi-gcc，AST2500/ARM1176JZF-S。

## 1. 設定

設定已拆成兩個檔案，不再使用 Python `config.py`：

- `server.config`：AI server URL/key/model、API mode、context/output 限制、原始專案與 WORKSPACE_ROOT、MSYS2/toolchain、CPU/ABI、provisional memory map、Git identity。
- `agent.config`：AI retry、build retry/iteration、make jobs/target、階層分析開關、OS entry、prebuild `.a`、未引用 source、code preservation、Error Handler/evidence budget、fake header、missing symbol 等 Agent 策略。
- `project.config`：人工維護的全專案 preprocessor defines，例如 `PROJECT_DEFINES += ARM`；Agent 會自動轉成 `-DARM` 並套用到所有 C/C++/預處理組語編譯。
- `manual_exclude.txt`：人工指定不參與 build 的 source 或目錄；只排除 build，不刪除原始碼。

`server.config` 可能包含 API key，因此被 `.gitignore` 排除；`server.config.example` 是可提交的範本。`agent.config` 不含 server key，可正常版本控制。

可用環境變數覆蓋 server AI 連線：`ADS_PORTER_AI_URL`、`ADS_PORTER_AI_KEY`、`ADS_PORTER_AI_MODEL`。也可用 `ADS_PORTER_SERVER_CONFIG` / `ADS_PORTER_AGENT_CONFIG` 指向其他設定檔。

## 1.1 project.config

`project.config` 放在工具根目錄，與 `agent.config` 同層。只填 macro，不要自行加 `-D`：

```makefile
PROJECT_DEFINES += ARM
PROJECT_DEFINES += AST2500
PROJECT_DEFINES += USE_USB=1
```

每次啟動 `launcher.py` 時，worker 會讀取這份設定並產生 `working_project/make/project_defines.mk`。`Makefile.gcc` 會在 `make/config.mk` 後載入該 fragment，因此 C、C++、`.S` 與 prebuild `.a` 的 compile command 都會共用相同 define。可在 working project 執行：

```bat
make -f Makefile.gcc print-project-defines
```

確認實際產生的 `-D...`。一般 `.s` 不經過 C preprocessor，因此不會使用這些 macro。

## 2. 測試

```bat
python inspect_environment.py
python test_ai_server.py
python test_git_operations.py
```

## 3. 第一次 Git commit

解壓後在工具根目錄執行：

```bat
python git_commit_changes.py
```

或雙擊 `FIRST_GIT_COMMIT.bat`。腳本會自動 `git init`、設定 local identity、讀取各變更 Python 檔中的 `PORTER_CHANGE_LOG`，並產生：

```text
[what]
...

[why]
...

[how]
...
```

以後覆蓋新版 `.py` 後，再執行同一支腳本即可建立下一筆 commit。

## 4. 啟動移植

```bat
python launcher.py
```

按 `Ctrl+C` 會停止 worker、MSYS2、make、GCC，並 rollback 未完成 patch。

## 5. Working project Git

第一次執行 worker 時，會在 `WORKSPACE_ROOT/working_project` 建立 Git baseline。每個通過 Makefile/build 退步檢查的 AI patch，只提交該 transaction 的檔案，commit 格式同樣是 `[what]/[why]/[how]`。

## 6. __dso_handle

`undefined reference to __dso_handle` 會分類為 `CXX_DSO_RUNTIME`，先檢查 link driver、`-nostartfiles/-nostdlib`、`crtbegin.o/crtend.o`、第三方 `.a` 中的 C++ runtime symbols，再交給 AI。直接補 dummy symbol 不是預設修法。

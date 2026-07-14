# PTT AI 版主助理（PTT AI Moderator Assistant）

透過 [PyPtt / PTT MCP server](https://github.com/PttCodingMan/PyPtt) 讓 [Claude Code](https://claude.com/claude-code) 巡邏 PTT 看板最近的文章（含推噓文留言），比對「該板板規 + 通用 PTT 站規」，列出可能違規的文章、理由與建議處置，供板主裁量。

> **本分：給建議，不做裁決。**
> 助理只負責判定與提出建議，處置與否由板主決定。預設**只產報告、不做任何寫入動作**。任何處置（推噓 / 水桶 / 刪文 / 站內信 / 發公告）只有在使用者是該板板主、明確要求、並逐項確認後才執行。

---

## 運作流程

```
登入並確認板主身分 → 取板規 → 決定巡邏範圍 → 逐篇讀取(含推噓) → 判定違規 → 產出報告 (→ 板主授權後處置 / 發公告)
```

判定涵蓋：標題分類格式、內文品質（過短／注音火星文／純連結／洗版）、行為（人身攻擊／引戰／廣告業配／政治文／洩漏個資／賭博詐騙）、推噓文（洗推／ID-pa／暴雷未防雷），以及該板板規的逐條對照。找不到板規時，改依通用 PTT 站規判定並於報告註明。

---

## 前置需求：設定 PTT MCP server

需要 PTT MCP server 已在 Claude Code 註冊（工具會以 `mcp__PTT__*` 形式出現）：

```bash
claude mcp add PTT -s user -e PTT_ID=你的帳號 -e PTT_PW=你的密碼 -- \
  podman run -i --rm -e PTT_ID -e PTT_PW ghcr.io/pyptt/ptt_mcp_server:latest
```

註冊後**重啟 Claude Code**。

> ⚠️ PTT 同帳號只允許單一連線。巡邏前請先關掉其它 PTT session（例如別處開著的 PTT MCP），否則會「重複登入」衝突。
>
> ⚠️ 要執行刪文 / 水桶 / 發公告等板主動作，設定的 `PTT_ID` 必須是目標看板的板主。

---

## 使用方式

設定好 PTT MCP server 後，二擇一：

### 方式 A（推薦，最省事）：複製 CLAUDE.md

把本專案 [`CLAUDE.md`](./CLAUDE.md) 的內容，貼進你自己 Claude Code 專案根目錄的 `CLAUDE.md`（沒有就新建）。這份是自足的操作手冊，包含完整判定規則、增量水位邏輯與處置流程，不依賴任何外部檔案。

### 方式 B：Clone 整個專案

```bash
git clone https://github.com/PttCodingMan/ptt-ai-moderator.git && cd ptt-ai-moderator
```

在此目錄開 Claude Code，`CLAUDE.md` 會自動載入，接著輸入「巡 <板名>」即可開始。

### 開始巡邏

在 Claude Code 輸入：

```
巡 <板名>
```

例如 `巡 Test`。首次巡某板預設掃最近 20 篇；之後每次只**增量**掃描上次水位之後的新文章。巡完會產出違規報告；若你是板主並明確要求，才會在逐項確認後執行處置或發公告。

---

## 紀錄檔

巡邏狀態與輸出寫在 `records/<板名>.json`（每板一檔，持久化增量水位、板規、歷次掃描與違規紀錄）。以**上次最後一篇存活文的發文時間**（`last_time` + `last_aid`）當水位，往回掃到上次位置即停——不用 index，因為刪文會讓 index 位移失準，改用記錄的時間判斷，標的文被刪也不受影響。

每筆判決逐筆留存於 `scans[]`，並彙總到 `precedents`：巡邏時先讀過往判例，對**同型違規維持一致的嚴重度與處置**（越判越準）；反覆出現或灰色地帶的案例則累積成 `rule_revision_candidates`，作為板主**修訂板規／發文模板的依據**。

範例：

```json
{
  "board": "<板名>",
  "moderators": ["<板主ID>"],
  "rules": "（板規關鍵可罰條款摘要，或標註以站規為準）",
  "rules_source": "置底 vX 公告 aid ... | PTT站規(通用rubric)",
  "watermark": {"last_index": 12345, "last_time": "Thu Jul  2 22:48:33 2026", "last_aid": "<aid>"},
  "scans": [
    {"scanned_at": "2026/07/02", "range": "12321-12345", "checked": 22, "violation_count": 1,
     "violations": [
       {"index": 12330, "aid": "<aid>", "title": "<標題>", "author": "<作者ID>", "types": ["亂碼火星文"],
        "severity": "中", "rule": "板規2-5/7", "evidence": "原文摘錄", "action": "已刪文(日期)"}
     ]}
  ]
}
```

---

## 處置動作（謹慎，預設不執行）

唯有使用者是該板板主且**明確要求**、並逐項確認後，才會使用寫入工具：`comment`（推噓留言）、`bucket`（水桶）、`del_post`（刪文）、`mail`（站內信）、`post`（發公告）。**不可批次無確認執行。**

---

## 免責

本工具僅供板主輔助巡板與板規稽核之用。違規判定為 AI 依規則之初步建議，**非最終裁決**，一切處置由板主自行判斷負責。巡邏輸出（`records/`）含實名 PTT 帳號、違規判定與文章內容，已列入 `.gitignore`、**只留在本機不進版控**；若你要自行備份或分享，請留意當事人權益。

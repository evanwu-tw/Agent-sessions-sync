# 收掉 Agent Session 跨機同步,改由專案 `.md` 承載脈絡

## Context(為什麼要改)

`~/agent-sessions` 這套「Claude Code / Codex session 跨機同步」方案建於 2026-06-15,運作正常、地基(路徑大小寫)也對齊。但經過一輪靈魂拷問,用**實際使用數據**對帳後,結論是:**它在解一個 Evan 實際上幾乎沒遇到的問題,該降級退役。**

對帳出來的事實:

- **十天只有 1 次** commit 是真正的「收工同步」(06-17),之後九天有開新 session 工作、但從沒 push。跨機 resume 實際發生 **≈ 1 次**。
- session transcript 是**惰性檔**:躺在硬碟上不進任何 context、不花 token,只有在你主動 `--resume` 時才有成本——而你幾乎不 resume、也不翻舊帳。
- 「污染傳染 / context 變髒 / 佔 token」三個 concern,對 transcript **都不成立**;真正會自動載入 context、會跨機傳染的是 **memory 檔**(`memory/*.md`、`MEMORY.md`),它本來不該藏在這個 repo 裡。
- repo 還被第三方 App **CodexBar** 的探針(`Library-Application-Support-CodexBar-ClaudeProbe`)灌進 433 個垃圾檔洗版(已於今天 06-26 加 `.gitignore` 擋下未追蹤的,但仍有 24 個已被追蹤)。

**決定的新架構:**

1. **跨機脈絡改由專案級 `.md` 承載**(CLAUDE.md / AGENTS.md + 相關 md),跟著各專案自己的 git 走。產出方式:專案進行中由 agent 寫初稿、隨專案迭代,Evan 在「專案收斂點」觸發更新並 review。這正是原文件第 8 節自己的主張。
2. **session transcript 同步收掉**:拆 symlink,還原成各機本地實體目錄(Claude/Codex 原廠狀態)。
3. **memory 不搬、保留 Claude 原生機制**:不做 per-project symlink(避免一堆 symlink 難管理)。memory 變成各機本地、不跨機同步、會自然分歧——可接受,因為該長期保留的洞見會被提煉進專案 `.md`。
4. **`agent-sessions` repo 凍結成冷備份**:不刪,留著當「兩台合併的歷史快照」(註:尚未完全合集,見下方「另一台 Mac」段),但不再是活的同步來源。**不需要任何自動化。** 要分清兩件事:
   - **native 本地目錄**(`~/.claude/projects`、`~/.codex/sessions`)= 每台機器的 **active 本地歷史**,持續累積、隨手可查,但只在本機、會兩台分歧。
   - **冷備份** = `~/agent-sessions` 這個凍結 repo(+ GitHub remote),是離線、跨機的歷史快照。
   兩者不同層:native 是「現役工作目錄」,冷備份是「封存檔案室」,別混為一談。

**眼睛睜開接受的已知小洞:** 專案「進行中段、脈絡還沒寫進 .md」時若跨機接續,會有空窗(.md 還沒更新、session 又不同步)。但此情境十天 ≈ 1 次,且可靠 `/context` + 當下記憶硬接,屬可接受的小裸奔,不是 deal-breaker。

---

## 執行步驟(本台 Mac)

> **狀態:本台(公司機)已於 2026-06-26 執行完成(含 seal commit/push,HEAD = `df3f212`)。** 以下步驟同時作為**另一台 Mac 的 runbook**——照跑前先看下方「另一台 Mac」段的前提。
> 前提:執行當下 Claude Code / Codex **沒有在跑**(否則它們正抓著 symlink 讀寫)。

### 1. 封存前先 seal 一次快照
把目前 repo 狀態做最後一次 commit,讓凍結的冷備份停在最完整的點(`.gitignore` 已擋掉 CodexBar 未追蹤垃圾)。可順手 `git rm --cached -r` 已追蹤的 CodexBar 探針檔做最後清理,再 commit / push。

### 2. 拆 symlink、還原成本地實體目錄
目前 `~/.claude/projects`、`~/.codex/sessions` 應是 symlink → repo。從本機 clone 內容**複製**回實體目錄(repo 自己那份保持不動 = 冷備份):

```bash
# 先看狀態
ls -la ~/.claude/projects ~/.codex/sessions

# 安全 guard:兩個都確實是 symlink 才執行;否則印警告、不動作
# (狀態跟預期不同時硬跑,rm 雖不刪實體目錄,但後面 cp 可能造成 nested/混合,排查很煩)
if [ -L ~/.claude/projects ] && [ -L ~/.codex/sessions ]; then
  rm ~/.claude/projects ~/.codex/sessions                    # 只刪符號連結,不動 repo
  cp -R ~/agent-sessions/claude/projects ~/.claude/projects  # 從本機 clone 還原成實體目錄
  cp -R ~/agent-sessions/codex/sessions  ~/.codex/sessions
  echo "OK: 已還原成實體目錄"
else
  echo "STOP: 有一個不是 symlink,先 ls -la 看清楚狀態,別硬跑"
fi
```

### 3. 驗證
```bash
ls -la ~/.claude/projects   # 應是實體目錄,不再顯示 -> ...
ls -la ~/.codex/sessions    # 同上
ls ~/.claude/projects | head  # 內容還在
```
之後開一個終端機 `claude` / `codex` 確認能正常起、看得到歷史 session。

### 4. 清理(驗證沒問題後,可選)
建置期殘留的備份此時已冗餘,可清:
```bash
rm -rf ~/.claude/projects.pre-symlink.* \
       ~/.codex/sessions.pre-symlink.* \
       ~/.agent-sessions-import-tmp
```
`~/agent-sessions` repo **保留不刪**(凍結冷備份)。

---

## 另一台 Mac(必做,否則沒真正退役)

另一台仍掛著指向它自己 `~/agent-sessions` clone 的活 symlink。它跑一次上面的「執行步驟 2–4」(guard → 拆 symlink → **從它自己的 clone** 還原 → 清備份)就完成退役——它的本地歷史本來就在自己 clone 裡,還原後留在本機即可。

**不需要 git pull / push。** 跨機 session 內容已無所謂(各機本機自己備份就好),所以不必為了「湊齊兩台合集」去同步;`~/agent-sessions` 的 GitHub remote 維持凍結、即使不是完整兩台合集也沒關係。兩台各自退役、各留各的本地歷史,就是最終狀態。

---

## 建議的後續慣例(非本次退役的必做,但建議定下來)

**CLAUDE.md 與 AGENTS.md 的關係要先講定,避免同專案兩份脈絡檔漂移。**
建議:每個專案 repo 內以 **AGENTS.md 為 canonical**(跨工具標準、Codex 也讀),`CLAUDE.md` 在 repo 內用 symlink 指向 AGENTS.md(或反之),兩個工具更新的是**同一個 source of truth**,零分歧。若真有工具專屬內容,讓專屬檔保持精簡、共用核心仍放 canonical。

---

## 驗證(整件事算成功的鐵證)

- [ ] 本台 `ls -la ~/.claude/projects` / `~/.codex/sessions` 顯示為**實體目錄**(無 `->`)
- [ ] 終端機起 `claude --resume` / `codex resume --all` 仍看得到本機歷史 session
- [ ] `~/agent-sessions` repo 仍在,但不再被讀寫(不再是同步來源)
- [ ] 另一台 Mac 也完成步驟 2–4
- [ ] 往後脈絡落地點:挑一個進行中的專案,實際跑一次「收斂點請 agent 更新 CLAUDE.md/AGENTS.md → review」的流程,確認這條新主力路徑可運作

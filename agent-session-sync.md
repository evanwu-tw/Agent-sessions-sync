# Agent Session 同步:Claude Code + Codex 跨機方案

> 這份文件記錄「兩台 Mac 之間同步 Claude Code / Codex 的 session 紀錄」的完整方案 —— 架構、做法、踩過的雷、以及維護紀律。可帶進 Claude Code / Codex 當專案的參考文件,後續調整時翻這份。

---

## 🛑 退役後記(2026-06-26)— 先讀這段,別照下面重建

**狀態:這套 session 同步已退役。下面第 1–9 節保留作「歷史紀錄 / 萬一要重建的參考」,但 2026-06-26 起不再是現役做法。讀到這份文件時,先確認你是不是真的要重建,別預設照做。**

### 為什麼收掉(用實際數據對帳的結論)

建好十天後檢視 repo 的真實使用痕跡:

- **跨機 resume 實際只發生 ≈ 1 次**(十天內只有 1 個 commit 是真正的「收工同步」,之後九天照常工作卻從沒 push)。這套在解一個幾乎沒發生的問題。
- **session transcript 是惰性檔**:躺在硬碟上不進 context、不花 token,只有你主動 `--resume` 才有成本 —— 而實際上幾乎不 resume、也不翻舊帳。
- 「污染傳染 / context 變髒 / 佔 token」的擔心,對 transcript 都不成立;真正會自動載入 context、會跨機傳染的是 **memory 檔**,那是另一回事。
- repo 還被第三方 App **CodexBar** 的探針(`...CodexBar-ClaudeProbe`)灌進數百個垃圾檔洗版。

一句話:**transcript 同步的價值趨近於零,維護成本與雜訊卻是真的。** 詳細的「靈魂拷問」過程見同資料夾的決策檔 [`decommission-decision-2026-06-26.md`](decommission-decision-2026-06-26.md)。

### 改用什麼(新主力)

跨機脈絡改由**專案級 `.md`** 承載(`CLAUDE.md` / `AGENTS.md` + 相關文件),跟著各專案自己的 git 走 —— 這本來就是本文件第 8 節「session 同步只是安全網、不是主力」的主張。產出方式:專案進行中由 agent 寫初稿、隨專案迭代,Evan 在「專案收斂點」觸發更新並 review。

> 慣例建議:同專案內以 `AGENTS.md` 為 canonical、`CLAUDE.md` symlink 指過去,避免兩工具兩份脈絡漂移。

### 退役做了什麼(2026-06-26,公司機已執行)

1. **封存快照**:最後一次 commit/push,把 CodexBar 探針檔 `git rm --cached` 移出追蹤,repo 凍結於此。
2. **拆兩條 symlink**:`~/.claude/projects`、`~/.codex/sessions` 從 symlink 還原成各機本地實體目錄(Claude/Codex 原廠狀態)。
3. **`~/agent-sessions` repo 不刪、冷凍**:留作歷史快照,但不再是活的同步來源,不需任何自動化,也不再 push(各機本機自己備份就好,GitHub remote 維持凍結、非完整兩台合集也沒關係)。用詞分清:**native 本地目錄是各機的 active 本地歷史**(持續累積、會兩台分歧),**冷備份是 `~/agent-sessions` 凍結 repo**,兩者別混。
4. **memory 不搬**:保留 Claude 原生機制,各機本地、不跨機(避免一堆 per-project symlink)。

### ⚠️ 尚未完成 / 注意

- **另一台 Mac 還沒退役**:它仍掛著指向自己 clone 的活 symlink。要在那台也跑一次「拆 symlink → 從本機 clone 複製還原成實體目錄 → 清備份」。在那之前互不衝突,只是它收不到更新而已。
- **已知接受的小洞**:專案中段(脈絡還沒寫進 `.md`)若跨機接續會有空窗。但此情境十天 ≈ 1 次,可靠 `/context` + 當下記憶硬接,屬可接受的小裸奔。

---

## 0. 一句話總結

用一個 **private git repo**(`~/agent-sessions`)集中存放兩個工具的 session,各台用 **symlink** 接回原位,靠 **手動 `git pull` / `git push`** 同步。**不用 iCloud。**

這套是 **跨同一工具、跨兩台機器** 的歷史同步(CLI 版本),定位是「**安全網,不是主力**」—— 真正承載脈絡的應該是專案說明檔和 wiki(見第 8 節),session 同步只是「需要時翻得到、接得上」的保險。

---

## 1. 為什麼用 git 而不是 iCloud

session 是 **append-only 的活檔案**(對話進行中持續寫入),這正好踩中 iCloud 的三個弱點:

| 風險 | iCloud | git |
|---|---|---|
| 衝突處理 | 沒有「衝突」概念,只會默默覆蓋或生「xxx 2.jsonl」副本 | 衝突會明確報出來、讓你解 |
| 寫入中同步 | 可能同步到寫一半的截斷檔,resume 解析失敗 | 你決定乾淨時間點才 commit |
| 「最佳化儲存空間」 | 會把檔案抽成 `.icloud` 佔位符,工具讀檔撲空 | 不適用 |
| 同步時機 | 最終一致,你無法強制「現在同步完」 | pull/push 是顯式的,狀態明確 |

**核心差異:iCloud 沒有「衝突」只有「覆蓋」;git 的衝突看得見、可控。** 這就是選 git 的根本理由。

> 另外:`.git` 目錄本身放 iCloud 是出了名地會弄壞 repo(物件檔被生成衝突副本後損毀)。所以需要版控的東西一律 local + git,不放 iCloud。

---

## 2. 最終檔案分層架構

按「資料性質」分流,每層獨立(不是全部塞進一個資料夾):

```
~/Evan-projects/          ← local git 專案(需要版控的)
   skills-source/
   <Proj A> / <Proj B> ...
~/icloud-projects/        ← symlink 到 iCloud Drive,放通用、非 git 的文件
   <通用專案>
~/agent-sessions/         ← agent session 同步 repo(本文件主角)
   claude/projects/       ← 兩台共用
   codex/sessions/        ← 兩台共用
   codex/session_index.jsonl          ← 公司機版(備份用,不靠它運作)
   codex/session_index.personal-import.jsonl  ← 自己機版(備份用)
   _backups/              ← 本機備份,被 .gitignore 忽略,不進 git
```

對應的 symlink(兩台都一樣):

```
~/.claude/projects  →  ~/agent-sessions/claude/projects
~/.codex/sessions   →  ~/agent-sessions/codex/sessions
```

### 為什麼 session repo 放 home 底下,不放 Evan-projects?

因為 session 是 **橫跨所有專案容器的 agent 狀態**,不是某個專案。未來若出現第二個專案容器(`Work-projects` 之類),把 session repo 塞在第一個容器裡就語意矛盾了 —— 一個服務全部容器的東西,該放在比所有容器都高的層級(= home)。這跟 `icloud-projects` 也獨立成一層是同一個原則:通用的放高層,專屬的放對應容器。

### 為什麼 repo 名用小寫 `agent-sessions`?

功能上大小寫無差(repo 資料夾名不影響 session 編碼)。選小寫的理由:(1) 不再多製造一個跨機要對齊的大小寫點;(2) 符合 home 底下工具倉庫的慣例(`.claude`、`.codex`、`.config` 都小寫)。

---

## 3. ⚠️ 最大的前置陷阱:路徑大小寫 / 結構必須兩台一致

**這是整套方案的地基,沒過這關後面全白做。**

Claude Code 用「**專案的絕對路徑字串**」編碼 session 目錄名(例如 `/Users/evanwu/Evan-projects/skills-source` → `-Users-evanwu-Evan-projects-skills-source`)。只要兩台的路徑字串有任何差異,**同一個專案會被當成兩個不同 project,session 同步過去也 resume 不到。**

踩過的具體狀況:
- **大小寫不同**:一台磁碟存 `Evan-projects`、另一台 `evan-projects`。
- **macOS 使用者名稱不同**:`/Users/evanwu` vs `/Users/evan` → 路徑前綴就不同。
- **APFS 大小寫不敏感但保留大小寫**:就算你打小寫 `cd`,`pwd` 仍回磁碟上的真實大小寫。所以**光改打字習慣沒用,得改磁碟上的資料夾本名**。

### 前置檢查清單(兩台都跑,結果必須一致)

```bash
whoami          # 兩台必須相同(例如都是 evanwu)
echo $HOME      # 兩台必須相同(例如都是 /Users/evanwu)
```

```bash
# 看磁碟上的真實資料夾本名與結構
ls -dl ~/evan-projects ~/Evan-projects 2>/dev/null
ls -F ~/Evan-projects 2>/dev/null
```

### 大小寫對齊腳本(以「統一成大寫 Evan-projects」為例)

> APFS 大小寫不敏感,直接 `mv Evan-projects evan-projects` 無效,要走「兩步 mv」(經過一個臨時名)。session 目錄名也要一起改,否則舊歷史 resume 不到。

存成檔案、用 `bash` 跑(zsh 直接貼 `shopt` 會炸)。**預設 dry-run 只印不動,確認清單後才 `APPLY=1`**:

```bash
cat > ~/fix-paths.sh << 'EOF'
#!/bin/bash
APPLY="${APPLY:-0}"
norm() {
  [ "$1" = "$2" ] && return
  echo "  $1  ->  $2"
  if [ "$APPLY" = "1" ]; then
    mv "$1" "$1.casetmp.$$" && mv "$1.casetmp.$$" "$2"
  fi
}
echo "== 1. home 資料夾 =="
cd "$HOME"
stored=$(ls -d */ 2>/dev/null | sed 's:/$::' | grep -ix 'evan-projects' | head -1)
if [ -z "$stored" ]; then echo "  (找不到)"; else norm "$HOME/$stored" "$HOME/Evan-projects"; fi
echo "== 2. Claude session 目錄(僅 local,跳過 iCloud 路徑)=="
PROJ="$HOME/.claude/projects"
if [ -d "$PROJ" ]; then
  cd "$PROJ"; shopt -s nullglob
  for d in -Users-evanwu-evan-projects-*; do
    case "$d" in *Library*|*CloudDocs*) continue;; esac
    new=$(printf '%s' "$d" | sed 's/-evan-projects-/-Evan-projects-/g')
    norm "$PROJ/$d" "$PROJ/$new"
  done
else echo "  (無 ~/.claude/projects)"; fi
echo "== done (APPLY=$APPLY) =="
EOF

# 先演習(只印不動)
bash ~/fix-paths.sh
# 確認清單沒問題後才真跑
APPLY=1 bash ~/fix-paths.sh
```

**踩雷提醒**:腳本第一版用 `*Evan-projects*` 的 `sed` 太貪心,會把連 **iCloud 路徑**(`...CloudDocs-Evan-projects-Portfolio`)裡的大小寫也一起改掉,導致那個 session 對不到 iCloud 雲端資料夾。修正方式 = glob 收緊成 `-Users-evanwu-...-*` + `case` 明確跳過含 `Library` / `CloudDocs` 的路徑(上面腳本已含此修正)。

### 驗證對齊成功

```bash
ls -d ~/*van-projects                          # 兩台都應只回大寫 /Users/evanwu/Evan-projects
ls ~/.claude/projects | grep -i evan-projects  # 前綴應全大寫 -Users-evanwu-Evan-projects-...
```

> 小提醒:選大寫 `Evan-projects` 的附帶好處 —— iCloud 雲端資料夾本來就是大寫,local 統一大寫後 iCloud 那層的 session 目錄剛好也對齊,省一道轉換。

---

## 4. 建置流程(順序很重要)

> **正確順序是「先合併、後切換」**:公司機匯入 → push → 自己機 clone + 合併 → push → **repo 成為完整合集後**,兩台才各自切 symlink。
> **不要** 在 repo 還沒湊齊兩台 session 前就先 symlink —— 一旦切了,工具就直接讀寫 repo,之後在「活動中的 repo」上 pull 合併,會把靜態檔案操作變成有風險的即時合併。
> **操作策略保守**:先 `cp` 快照,確認 push 成功、內容正常,**最後才** `mv` + symlink。`mv` 雖快,但在未驗證的 repo 上做不可逆切換不划算。

### 階段 1 — 公司機(電腦 A,當「來源」)

```bash
# 建 repo 結構 + git init,_backups 在 repo 內但被 git 忽略
mkdir -p ~/agent-sessions/claude/projects
mkdir -p ~/agent-sessions/codex/sessions
mkdir -p ~/agent-sessions/_backups
cd ~/agent-sessions
git init
cat > .gitignore <<'EOF'
.DS_Store
*.tmp
*.log
*.casetmp.*
_backups/
EOF

# 備份(原檔不動,放 repo 內 _backups,git 會忽略)
TS=$(date +%Y%m%d-%H%M%S)
cp -R ~/.claude/projects ~/agent-sessions/_backups/claude-projects.company.$TS
cp -R ~/.codex/sessions  ~/agent-sessions/_backups/codex-sessions.company.$TS
[ -f ~/.codex/session_index.jsonl ] && cp ~/.codex/session_index.jsonl ~/agent-sessions/_backups/codex-session-index.company.$TS.jsonl

# cp 快照匯入(不 move、不 symlink)
cp -R ~/.claude/projects/. ~/agent-sessions/claude/projects/
cp -R ~/.codex/sessions/.  ~/agent-sessions/codex/sessions/
[ -f ~/.codex/session_index.jsonl ] && cp ~/.codex/session_index.jsonl ~/agent-sessions/codex/session_index.jsonl

# 檢查:git status 的 untracked 應只有 .gitignore / claude/ / codex/,_backups/ 不該出現
git status
find . -maxdepth 3 -type d | sort
```

確認無誤後,接上 GitHub(用你自己的帳號,不是別人的 repo)並 push:

```bash
cd ~/agent-sessions
git add .
git commit -m "Import agent sessions from company Mac"
git branch -M main
git remote add origin git@github.com:<你的帳號>/agent-sessions.git
git push -u origin main
```

> 前置:先在 GitHub 用自己的帳號開一個 **空的 private repo**(不勾 README / .gitignore / License),`ssh -T git@github.com` 確認 SSH 通。

### 階段 2 — 自己機(電腦 B,當「接收」)

> 加了防呆:`~/agent-sessions` 已存在就停。`exit 1` 要在「存成檔案用 bash 跑」時才正確(直接貼進終端機 `exit` 會關掉你的 shell)。

```bash
cat > ~/import-personal.sh << 'EOF'
#!/bin/bash
if [ -e ~/agent-sessions ]; then
  echo "ERROR: ~/agent-sessions already exists. Please inspect it before cloning."
  ls -la ~/agent-sessions
  exit 1
fi
mkdir -p ~/.agent-sessions-import-tmp
TS=$(date +%Y%m%d-%H%M%S)
cp -R ~/.claude/projects ~/.agent-sessions-import-tmp/claude-projects.personal.$TS
cp -R ~/.codex/sessions  ~/.agent-sessions-import-tmp/codex-sessions.personal.$TS
[ -f ~/.codex/session_index.jsonl ] && cp ~/.codex/session_index.jsonl ~/.agent-sessions-import-tmp/codex-session-index.personal.$TS.jsonl
git clone git@github.com:<你的帳號>/agent-sessions.git ~/agent-sessions
# -n = no-clobber:不覆蓋公司機已有檔案,只補自己機獨有的
cp -Rn ~/.claude/projects/. ~/agent-sessions/claude/projects/
cp -Rn ~/.codex/sessions/.  ~/agent-sessions/codex/sessions/
# Codex index 不蓋公司機版本,另存
[ -f ~/.codex/session_index.jsonl ] && cp ~/.codex/session_index.jsonl ~/agent-sessions/codex/session_index.personal-import.jsonl
echo "== git status =="; cd ~/agent-sessions && git status
EOF
bash ~/import-personal.sh
```

檢查重點:`git status` 新增的應是自己機**獨有**的東西(全 `Untracked`),**不該有大批 `modified`**(出現 modified 代表公司機既有檔案被動到,要查)。同名專案裡的 JSONL 各自用 session id 命名 → union 不互蓋。確認後:

```bash
cd ~/agent-sessions
git add .
git commit -m "Merge agent sessions from personal Mac"
git push
```

### 階段 3 — 兩台都切 symlink(repo 已是完整合集後)

前提:① 兩台本地 repo 都已 `git pull` 到最新合集;② 切換當下該台的 Claude Code / Codex **沒有正在跑**。

```bash
TS=$(date +%Y%m%d-%H%M%S)
# 把現有實體目錄移走當本機備份(原檔不刪,改名留著)
mv ~/.claude/projects ~/.claude/projects.pre-symlink.$TS
mv ~/.codex/sessions  ~/.codex/sessions.pre-symlink.$TS
# 建 symlink 接到 repo
ln -s ~/agent-sessions/claude/projects ~/.claude/projects
ln -s ~/agent-sessions/codex/sessions  ~/.codex/sessions
# 驗證
ls -la ~/.claude/projects   # 應顯示 -> /Users/evanwu/agent-sessions/claude/projects
ls -la ~/.codex/sessions    # 應顯示 -> /Users/evanwu/agent-sessions/codex/sessions
```

> ⚠️ 切換前 `~/.claude/projects` 還是「實體目錄、裡面有東西」,**不能直接 `ln -s`**(會建到目錄裡面去,變成 `projects/projects` 錯位)。一定要「先 mv 移走 → 再 ln -s」。

### 最終驗證(成功的鐵證)

兩台各跑,清單應**逐行相同**(尤其各機原本獨有的專案,例如某台才有的 Portfolio,兩台現在都看得到):

```bash
ls ~/.claude/projects/ | head
ls ~/.codex/sessions/
```

---

## 5. 關於 `session_index.jsonl`(Codex)

實測 `codex resume --help`:`--all` 的說明是「disables cwd filtering and shows CWD column」—— 代表 Codex 的 session picker 是**靠掃描 session 檔的 cwd 屬性**決定清單,**不是讀 `session_index.jsonl`**。

**結論:`session_index.jsonl` 不 symlink、不跨機共用。** 它是各機本地的狀態/快取,真正的 source of truth 是 `codex/sessions/` 底下的 `rollout-*.jsonl`。兩台的 index 各自留本機自管;repo 裡那兩份(公司版 + `personal-import` 版)當備份留著,不刪、不靠它運作。

> 所以最終 symlink 只有兩條(claude/projects、codex/sessions),`~/.codex/session_index.jsonl` 維持原樣不動。

---

## 6. 日常維護紀律

```bash
# 開工前(任一台)
cd ~/agent-sessions && git pull

# 收工後
cd ~/agent-sessions && git add . && git commit -m "sync" && git push
```

**外加一條鐵律:不要兩台同時跑同一個 session。** 這是唯一會撞 git 衝突的情境 —— 而 git 撞衝突會明講、讓你解(不像 iCloud 默默覆蓋),所以守住 pull→做→push 的節奏就不會遇到。

> 想更省事可包成 shell alias 或開關機 hook,但屬錦上添花,非必要。

---

## 7. ⚠️ 已知限制與待辦

### 限制 1:這套只同步「終端機 CLI」的 session,不含桌面 App

**重要分辨**:Claude Code **桌面 App**(有 Chat / Cowork / Code 分頁的那個)和終端機 `claude` CLI 是**兩套不同的 session 儲存**:

- **CLI**:`~/.claude/projects/<encoded-cwd>/<uuid>.jsonl` ← 本方案同步的就是這個
- **App**:session 存在各專案目錄底下的 `Claude/` 子資料夾 + `~/Library/Application Support/`(格式不同,可能是 DB/帶狀態檔)

**徵兆**:在桌面 App 講了 hello,回到另一台 App 看不到 —— 因為 git `push` 會 `nothing to commit`(App 沒寫進 CLI 目錄)。

**若要同步 App**:屬另一條支線,難度比 CLI 高(資料分散兩處、可能是資料庫、symlink 不一定適用、等於回到「同步整包工具狀態」的死路)。先別硬做,評估 App 是否有內建帳號/雲端同步,或接受「App 歸 App、CLI 歸 CLI」。

> **測試同步時請用終端機 `claude --resume` / `codex resume --all`,不要用桌面 App 測,會一直撲空。**

### 限制 2:skill symlink 是另一條線(待修)

`user-research-design` 等 skill 的 symlink(`~/.claude/skills/` → `~/Evan-projects/skills-source/...`)target 字串寫的是小寫;APFS 大小寫不敏感所以目前仍解析得到、沒斷,但搬到大小寫敏感檔案系統會斷。另外自己機缺 Gemini 的 skill link。這屬 `skills-source` 那條線,跟 session repo 無關,想修再修。

### 清理備忘(不急)

`.pre-symlink.*`、`_backups/`、`~/.agent-sessions-import-tmp/` 等備份**先留著**,跑順一兩週確認沒事再清。

---

## 8. 定位與 token:重要的觀念校準

### session 同步是「安全網」,不是「主力」

真正該主動經營、跨機最該依賴的脈絡載體,是 **專案說明檔 + wiki / skill**(這條另有專案處理,本文件不展開)。session 同步的價值在於三個 `.md`/wiki 取代不了的場景:① 脈絡還沒整理進檔案的空窗期,跨機接續那條還熱的對話;② 翻舊帳(查上次怎麼解的);③ 幾乎零成本,建好就背景跑。

**判斷要不要在意某個 session,不是看「專案長短」,是看「會不會被你 resume 接續」。** 而且這套是「全有或全無」—— symlink 整個 projects 目錄,無法按專案挑。想做選擇性排除是過度工程(多存幾個沒用的 JSONL 成本趨近於零)。所以:**全同步、用完即丟、需要才 resume。**

### token 消耗:同步本身零成本,貴的是 context 長度

- **git pull/push = 零 token**(搬檔案,跟模型 API 無關)。
- **真正耗 token 的是 context 長度**:LLM 沒記憶,**每一輪都把整個 context(對話歷史)重讀一次**才回應。所以 session 越長,後面每講一句越貴(成本是每輪付全長,不是線性累加)。
- **同步是中性的**:它不增加 token,只是讓「長 session 的成本」可以換一台機器付(錢沒變多,換地方付)。
- **唯一的間接風險**:因為跨機接得上,你可能養成「一直 resume 同一個 session 把它越養越長」的習慣 —— 那會讓 token 燒更快,但貴的是 session 長度,不是同步。

**控制方法 = 管 session 長度,不是管同步**:跨機接續時若接下來是新的小任務,就**開新 session**,別硬接那條長的。

### 怎麼看一個 session 是否太冗

```bash
# 層級一:看檔案大小(粗篩,含 tool output 會灌水)
find ~/.claude/projects -name "*.jsonl" -exec ls -lhS {} + 2>/dev/null | head
find ~/.codex/sessions  -name "*.jsonl" -exec ls -lhS {} + 2>/dev/null | head
# 經驗值:單檔飆到十幾 MB 以上,幾乎肯定又長又重(resume 會貴)

# 層級二:數行數(比 bytes 更貼近「對話來回」多寡)
find ~/.claude/projects/<專案目錄> -name "*.jsonl" \
  -exec sh -c 'echo "$(wc -l < "$1") $1"' _ {} \; | sort -rn
```

- **層級三(最準)**:在 Claude Code 終端機 session 裡打 `/context`,看目前 context 用量。**過半就考慮換新、逼近上限一定換**(再下去不只貴,模型還會開始抓不到重點)。
- **體感訊號往往更早出現**:Claude 開始重複問你講過的事、答非所問、抓錯重點、變慢 → 就是 context 太滿,該開新 session(別只盯數字)。

---

## 9. 快速檢查清單(之後要重做或上新機時)

- [ ] 兩台 `whoami` / `$HOME` 一致
- [ ] 兩台專案資料夾大小寫 / 結構一致(`Evan-projects` 大寫)
- [ ] `~/.claude/projects` session 目錄前綴大小寫已對齊
- [ ] GitHub 自己帳號下開好空 private repo,SSH 通
- [ ] 先 cp 匯入 → push → 確認 → 才 mv + symlink(順序別反)
- [ ] `cp -Rn` 合併第二台(no-clobber),index 另存不覆蓋
- [ ] symlink 前先 mv 移走實體目錄
- [ ] 最終兩台 `ls` 清單逐行相同
- [ ] 日常:pull→做→push,不兩台同跑同一 session
- [ ] 測試用終端機 CLI,不用桌面 App

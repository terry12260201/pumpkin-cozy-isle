# 🏝️ 南瓜暖島美學 · Pumpkin Cozy Isle

> 一套「手繪水彩 × 等距視角 × 模擬經營遊戲」的視覺語言，做成 Claude Code 的 skill。
> 裝上之後，叫 AI 做的介面、工具、插圖、生圖，都會長得像同一個人做的。

**👀 [線上看色票總覽](https://terry12260201.github.io/pumpkin-cozy-isle/)** — 所有顏色、字級、元件一頁看完，點色塊直接複製色碼。

---

## 這解決什麼問題？

用 AI 做東西最惱人的一點：**每次做出來都不像同一套。** 這次是紫色漸層科技風，下次變成藍色扁平風，配色、圓角、陰影全都要重講一次。

這支 skill 把一整套視覺語言固定下來——色票、字體、元件、生圖咒語、修圖流程、驗收標準——讓 AI 每次都照同一本說明書做事。

## 風格長什麼樣？

想像一款療癒的模擬經營手遊：漂浮在雲上的小島辦公室、紙一樣的米白背景、深森林綠的招牌、粗粗的墨線邊框、按下去會凹陷的積木按鈕。

**溫暖、有人味、想一直看下去。**
不要：科技藍紫漸層、玻璃反光、霓虹發光、企業儀表板那種冷冰冰的感覺。

📎 完整落地範例：[南瓜 AI 開發中心](https://terry12260201.github.io/pumpkin-ai-office/)（整個網站都是照這套做的）

## 安裝

需要 [Claude Code](https://claude.com/claude-code)。一行指令：

```bash
git clone https://github.com/terry12260201/pumpkin-cozy-isle.git ~/.claude/skills/pumpkin-cozy-isle
```

Windows（PowerShell）：

```powershell
git clone https://github.com/terry12260201/pumpkin-cozy-isle.git $env:USERPROFILE\.claude\skills\pumpkin-cozy-isle
```

裝完重開 Claude Code 就生效。**不用 Claude Code 也能用**——直接把 `references/` 裡的規範貼給任何 AI，或當成自己的設計文件讀。

## 怎麼用

裝好之後不用特別呼叫，跟 AI 說這幾種話它就會自己翻開：

- 「幫我做一個 XX 工具／網頁／儀表板」
- 「照這個風格做」「用暖島美學」
- 「幫我生素材」「這張圖去背」

任何要做**介面、插圖、簡報、生圖**的時候都會自動吃這套規範。

## 裡面有什麼

| 檔案 | 內容 |
|---|---|
| `SKILL.md` | 給 AI 讀的操作規則（三分鐘上手 + 驗收標準 + 反 AI 罐頭清單） |
| `index.html` / `assets/色票總覽.html` | 色票總覽頁，雙擊直開、點色塊複製色碼 |
| `references/色票與變數.md` | 24 色權威色票 + 可整段複製的 CSS 變數 |
| `references/元件規格.md` | 按鈕、卡片、面板、標籤、進度條…的現成 CSS |
| `references/生圖prompt模板.md` | gpt-image-2 咒語模板（5 種現成範本，中文字會正確） |
| `references/素材處理SOP.md` | 去背、切幀、場景修補的做法與踩坑錄 |

## 三個最容易踩的坑（先看這個，很值錢）

### 1. 中文一定要指定圓體

主字體 `Be Vietnam Pro` **沒有中文字符**，不寫 fallback 的話中文會掉進黑體，整個風格瞬間破功：

```css
font-family: "Be Vietnam Pro", "Yuanti TC", "YouYuan", "PingFang TC",
             "Heiti TC", "Microsoft JhengHei", sans-serif;
```

### 2. 陰影一律「硬邊」

```css
box-shadow: 0 4px 0 rgba(30,50,25,.35);   /* ✅ 不模糊、只往下位移 → 積木感 */
box-shadow: 0 4px 12px rgba(0,0,0,.2);    /* ❌ 這是網頁感，不是遊戲感 */
```

這一條是整個風格的靈魂。

### 3. 字要真的看得清楚

**1280 寬的全頁截圖，肉眼要能讀出每一個字。** 不接受「放大就看得到」。
面板標題 ≥18px、內文 ≥13px、大數字 ≥20px。

## 想改成自己的風格？

改 `references/色票與變數.md`（那是權威版），其他檔案都引用它。換掉色票、留下結構，就是你自己的一套。

**建議不要**在個別專案裡偷偷改色——過三個月就又亂了，這支 skill 存在的意義就是防這件事。

## 使用說明

歡迎自由參考、學習、改成你自己的視覺語言。

這套配色與角色設定是**南瓜虛擬科技的品牌識別**，請不要原封不動拿去當作自己品牌的識別使用。想拿來當基底改造成自己的風格，非常歡迎。

---

*設計與維護：陳南宏（南瓜）｜南瓜虛擬科技*

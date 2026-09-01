# 生圖 Prompt 模板（gpt-image-2，實測可用）

> 生圖只用 **OpenAI gpt-image-2**。**禁止叫 Codex 生圖**（它沒有圖像模型）。
> 金鑰在 `~/.openai_key`。相關工具：`~/檢查GPT金鑰.command`、`~/GPT生圖.command`。

## 為什麼用 edits 端點而不是 generations

`/v1/images/edits` **可以附參考圖**（`image[]` 可多張），風格鎖定效果與 GPT 網頁版等效。
純文字描述風格的一致性遠差於「附一張既有素材當錨」——**能附圖就一定附圖**。

## 標準呼叫

```bash
KEY=$(cat ~/.openai_key | tr -d '\n')
curl -s https://api.openai.com/v1/images/edits \
  -H "Authorization: Bearer $KEY" \
  -F "model=gpt-image-2" \
  -F "image[]=@風格參考主圖.png" \
  -F "image[]=@同類元件參考.png" \
  -F "prompt=<風格錨定 ＋ 內容描述 ＋ 收邊咒語>" \
  -F "size=1024x1024" -F "background=transparent" -F "output_format=png" \
  > resp.json
```

回傳 `data[0].b64_json` → base64 解碼存 PNG → **一定要驗 alpha**：

```python
import json, base64, io
from PIL import Image
d = json.load(open("resp.json"))
im = Image.open(io.BytesIO(base64.b64decode(d["data"][0]["b64_json"]))).convert("RGBA")
print("alpha extrema:", im.getchannel("A").getextrema())  # 要看到 (0, 25x) 才是真透明
im.save("out.png")
```

## 三段式 Prompt 結構

### 第 1 段：風格錨定（每次都貼這段）

```
Style reference: the first image is the master art style — hand-drawn storybook game UI,
watercolor paper texture, warm cream and deep forest green palette, thick dark-brown outlines,
matte finish, no gloss, no gradient. The second image shows the exact component style to match.
```

### 第 2 段：內容描述（換這段）

講清楚**輪廓形狀**與**配色**，不要只講名稱——模型認輪廓不認名字。

### 第 3 段：收邊咒語（每次都貼這段）

```
isolated on FULLY TRANSPARENT background, no backdrop, no glow,
nothing else in the image, the subject fills most of the canvas.
```

---

## 現成模板

### A. 導覽／功能按鈕（中文字要正確）

```
<第1段> Draw ONE single square game nav button in that exact style:
cream-colored rounded square button with a thick dark forest-green outline and subtle drop shadow,
a cute hand-drawn colorful icon in the upper 55 percent,
and bold dark-green Traditional Chinese text centered in the lower part.
Icon: <畫什麼，例：a clipboard with a checklist>. Text: 「<中文字>」
<第3段>
```
✅ 實測中文字會正確渲染。**一次只生一顆**，一次多顆會亂。

### B. 徽章／標誌

```
<第1段> Draw ONE diamond-shaped emblem badge: rotated square with thick dark-green border
and cream inner frame, containing <內容物，例：a cute hand-drawn stone castle with a blue roof
on a small green grass mound>.
<第3段> no red bar, no text.
```

### C. 等距場景／房間

```
<第1段> Draw an isometric floating-island office room in that exact style:
hexagonal wooden plank floor, low stone-block walls with green foliage on top,
<家具清單>. Soft watercolor sky with fluffy clouds around the island.
Empty floor in the center — no desks, no characters.
```
⚠️ **要放角色的場景，一定要生「無家具空地板版」**——角色圖若自帶桌椅，場景再有桌子就會重疊，這是踩過的坑。

### D. 角色（動物 Agent）

```
<第1段>（另附該角色既有設定圖當第二張參考）
Draw the same character in isometric view, sitting at a wooden desk working on a laptop,
cozy warm colors, same outline weight and proportions as the reference.
<第3段>
```
- 一隻角色建議生**一張多格動作表**（如 4×2）再切，比分開生更容易保持一致
- 動作分兩類：**工作 4 格**（打字／看板／喝咖啡／開心）＋**睡覺 4 格**（趴桌／打盹／椅子上睡／Z字）

## 常見失敗與對策

| 症狀 | 對策 |
|---|---|
| 背景不透明／有色塊 | 收邊咒語沒貼完整；`background=transparent` 沒帶；重生 |
| 風格跑掉、變成 3D 或扁平向量 | 忘了附參考圖；或 prompt 沒寫 `matte, no gloss, no gradient` |
| 中文字亂碼／缺筆畫 | 一次生太多顆；改成一次一顆，並用「」框住文字 |
| 主體太小、四周空白多 | 加 `the subject fills most of the canvas` |
| 邊緣有淡色殘影 | 生成品質問題，**重生比修圖快**；真要修用色鍵（見素材處理 SOP） |

## 生完的收尾三步

1. **驗 alpha**（上面那段 Python）
2. **拼檢查表**：把同批素材橫向拼成一張圖，一次看完風格是否一致
3. **對照色票**：吸管取色，與 `色票與變數.md` 差 5% 以內就改用色票值

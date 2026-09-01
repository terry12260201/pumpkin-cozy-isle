# 素材處理 SOP（含踩坑錄）

> 這份是**血淚換來的**。做素材前先讀，可以省掉好幾輪返工。

## 🥇 修圖優先序（永遠由高往低試）

| 順位 | 做法 | 什麼時候用 |
|---|---|---|
| 1 | **用原生乾淨素材** | 南瓜手上有切好的圖 → **一律優先**，比自己修的乾淨十倍 |
| 2 | **重新生圖**（gpt-image-2） | 缺素材、或既有素材有破綻 |
| 3 | **CSS / inline SVG 重刻** | 按鈕、徽章、圖示等幾何元件 → **零去背風險，首選** |
| 4 | 演算法修補（PIL/cv2） | **最後手段**，一定會留痕跡 |

**鐵則**：只要南瓜說「我切好給你」，就別自己修圖——直接問他要，或等他給。

🔴 **反覆踩過的坑**：拿到「有假 UI 的場景圖」時，不要急著自己清 UI 補天空——**先問有沒有無 UI 的乾淨版**。自己補的天空無論用哪種演算法（inpaint／紋理平鋪／行中位色＋程序雲），在大面積上都會留下**矩形邊界的模糊塊**，一眼就看得出來。清過三輪都不如直接拿原圖。

## ✂️ 去背與切割

### alpha 連通域自動切割（切整頁 UI 稿的標準做法）

不要手填座標，讓程式沿透明邊界自己找：

```python
import numpy as np, cv2
from PIL import Image

ui = Image.open("整頁UI去背圖.png").convert("RGBA")
a = (np.array(ui)[...,3] > 10).astype(np.uint8)
a = cv2.morphologyEx(a, cv2.MORPH_CLOSE, np.ones((9,9), np.uint8))  # 黏合鄰近碎塊
n, lab, stats, _ = cv2.connectedComponentsWithStats(a, 8)
for k in range(1, n):
    x, y, w, h, area = stats[k]
    if area < 800: continue                    # 濾掉雜點
    ui.crop((x, y, x+w, y+h)).save(f"cut_{k}.png")
```
排序技巧：`comps.sort(key=lambda c:(c[1]//80, c[0]))`＝**先上下分列、再左右排序**，切出來的順序才符合閱讀順序。

### 色鍵去背（點陣圖去單色底）

```python
arr = np.array(im).astype(int)
r, g, b = arr[...,0], arr[...,1], arr[...,2]
dist = np.sqrt((r-24)**2 + (g-59)**2 + (b-28)**2)   # 距離底色
arr[...,3] = np.where(dist < 38, 0, 255)
```
⚠️ 閾值太大會**吃掉暗部細節**（星星 icon 被吃過）；太小會留毛邊。從 30 開始試。

### flood-fill 去天空底

從四角與邊中點灌水，只清「連到邊界」的背景，不會誤傷主體內部同色區。必要時再套**菱形／圓形遮罩**清掉殘留角落雲塊。

## 🎬 角色動作幀（做「電子寵物感」的關鍵）

### 切幀三原則

1. **每幀獨立緊裁**（各自 `getbbox()`），不要統一畫布
   - 踩過的坑：想用「全 8 幀聯集 bbox」保持對位 → 工作列與睡覺列在原圖的縱向位置不同 → 畫布被撐大一倍，角色縮很小
2. **底部錨定**：容器用 `position:absolute; bottom:0`，讓桌腳永遠貼地
3. **等比縮放**：每幀依**自己的原始寬** × 統一係數，不要都拉成同寬

```js
const RATIO   = {persi:1.2852, /* 各角色 w2 幀的 高/寬 */};
const W2WIDTH = {persi:305,    /* 各角色 w2 幀的原始寬 */};
const scale = displayWidth / W2WIDTH[key];
img.style.width = Math.round(img.naturalWidth * scale) + "px";  // 等比，不拉伸
```
少了第 3 點 → 切幀時角色會忽大忽小；少了第 2 點 → 會上下跳動。

### 清鄰格滲入

從多格表切幀時，鄰格內容常滲進來。用連通域刪掉「碰到邊界的非最大塊」：

```python
n, lab, stats, _ = cv2.connectedComponentsWithStats((cell[...,3]>10).astype(np.uint8), 8)
biggest = 1 + int(stats[1:,4].argmax())
hh, ww = lab.shape
for k in range(1, n):
    if k == biggest: continue
    x, y, w, h, _ = stats[k]
    if x==0 or y==0 or x+w>=ww or y+h>=hh:      # 觸邊 = 鄰格滲入
        cell[lab==k] = 0
```

### 動畫播放方式

- **靜態情境**（真實狀態）：**定格 ＋ 呼吸浮動**（`translateY` ±2.5px, 3.4s）→ 零跳格，最耐看
- **表演情境**（展示模式）：兩張圖 **crossfade**（`opacity` 0.55s 交叉），**原地換動作**，不要位移
- 每個角色給不同的 `animation-delay`，避免整齊劃一像機械

## 🖌️ 場景修補（最後手段，非不得已別做）

### ❌ 不要用 cv2.inpaint 修大面積
會**拖影**成一團糊，遠看就是「AI 修過的」。修小刮痕可以，修天空／地板不行。

### ✅ 天空：逐行中位色 ＋ 程序柔雲

```python
# 1) 每行在「非遮罩區」找藍天像素，取中位色填滿該行
m = (b > r+8) & (b > 120) & (g > 100)      # 藍天判定
rowcol = np.median(row[m], axis=0)

# 2) 疊程序雲打破平色感（PIL 畫橢圓群 → GaussianBlur(7) → 低透明度混合）
```
⚠️ **不要拿原圖天空當紋理平鋪**——若天空散布遠景小島／裝飾，會複製出壁紙化偽影（踩過兩次）。

### ✅ 地板：低頻 ＋ 高頻分離
- 低頻＝遮罩外環帶地板色的 y 向中位漸變（**不要用 inpaint**）
- 高頻＝乾淨木紋 patch 隨機平鋪，用 Hanning 窗疊加消接縫
- 合成 `low + high*0.95`，遮罩邊界窄羽化（9px MaxFilter/MinFilter 取邊帶 → GaussianBlur 2.5）

### ✅ 羽化參數
邊界羽化**寧窄勿寬**：寬羽化＝一圈明顯模糊帶，比原本的破綻更醜。

## 🔢 素材上的假數字

切圖幾乎都含假數字（`12,340`、`Rex`、`Level 7`）。**一律不得原樣上線**：

1. 取乾淨底色（吸管取文字區旁邊）
2. 用底色矩形塗掉文字區
3. HTML 疊真實數據上去

### ⚠️ 塗抹必須 alpha-aware（血淚，踩過兩次）

**直接塗矩形會把圓角塗成方角**——因為矩形範圍會蓋到元件邊緣的透明像素，那些像素被填成實色後，圓角就變成一塊突出去的色塊。

```python
def paint(arr, box, color):
    """只在 alpha>200 的實心區塗色，圓角/邊緣的透明像素完全不動"""
    x1, y1, x2, y2 = box
    reg = arr[y1:y2, x1:x2]
    solid = reg[..., 3] > 200          # ← 關鍵這行
    for c in range(3):
        reg[..., c] = np.where(solid, color[c], reg[..., c])
```

三條配套規則：
1. **範圍右界要內縮 6px 左右**，避開分隔線／格線——塗到就整條不見（踩過：資源列五格塗完少了兩條分隔線）
2. **相鄰兩塊要連續塗成一段**，不要分兩次塗——接縫處會留一條殘影（踩過：文字區塗 y10-88、按鈕區塗 y92-140，中間 y88-92 留下一條綠殘跡）
3. **塗完一定用棋盤格底檢視**，透明區才看得出來有沒有被填實：

```python
bg = Image.new("RGBA", im.size, (255,255,255,255))
for y in range(0, h, 12):
    for x in range(0, w, 12):
        if (x//12 + y//12) % 2: bg.paste((214,222,230,255), (x,y,x+12,y+12))
Image.alpha_composite(bg, im).save("檢查.png")
```

## 🐛 前端整合常見坑

| 坑 | 症狀 | 解法 |
|---|---|---|
| CSS specificity | `#box > img{width:100%}` 蓋掉子元素尺寸 → 徽章變巨大、按鈕變色帶 | 主底圖給專屬 class（`.bg`），別用萬用選擇器 |
| 刪 UI 沒清 JS | `S("#el").classList` 對 null 報錯，整頁停擺 | 用 optional chaining `?.`，或一併刪 JS |
| class 名改了沒同步 | 切換狀態時舊高亮不消失 | 全域搜尋舊 class 名，確認 querySelectorAll 對得上 |
| 圖層順序 | 角色壓住導覽列 | z-index 分層：底圖 0／角色 10-48／UI 20／導覽 50／遮罩 60 |

## 📸 QA 截圖（每輪都要做）

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless --disable-gpu --hide-scrollbars \
  --window-size=1280,760 --virtual-time-budget=8000 \
  --screenshot="QA/round_N.png" "file://$(pwd)/index.html"
```
- `--virtual-time-budget` 要夠大（8000ms），否則圖還沒載完就截
- 支援 `#hash` 直開各分頁（在程式裡加 hash 路由），方便一次截完所有畫面
- **併排比對**：把參考圖與截圖上下拼成一張，一眼看出差異

### 推上 GitHub Pages 後的驗證

**圖片改版後線上不會立刻更新**（CDN 邊緣快取），截一次看到舊圖不代表沒推成功。正確驗證順序：
1. 先比對 hash（本機／git blob／`curl` 抓線上）——三者一致就代表**推送沒問題**，剩下純粹是快取
2. 再用 until 迴圈輪詢到畫面真的更新為止，不要看一次就下結論：

```bash
chk() { chrome --headless --screenshot=/tmp/lc.png "$URL?cb=$RANDOM"; python3 -c "比對差異; print('OK' if diff<50 else 'WAIT')"; }
until [ "$(chk)" = "OK" ]; do sleep 15; done
```

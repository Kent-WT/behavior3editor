# 變更說明

## Bug 修復

### 切換 tab 後視角跳位
切換至其他 tab 再切回行為樹時，viewport 會偏移到錯誤位置。
- **根本原因**：`useEffect` 在每次 tab 切換時無條件呼叫 `refresh()` → `_update()`，導致整張圖被 `clear()` + `render()` 完整重建，viewport 狀態在重建過程中遺失。
- **修正**：tab 切換回來時不再無條件重建圖，僅在以下情況才 refresh：
  1. subtree 有外部更新（檔案被外部修改）
  2. 有 `focusId`（從其他地方跳轉過來）
  
  其餘情況保持圖和 viewport 原封不動，只更新 inspector 面板。

### Viewport 還原順序錯誤
`_update()` 中還原 viewport 時先 `translateTo` 再 `zoomTo`，但 G6 的 `translateTo` 內部用 `currentZoom` 計算 camera 位置，若 zoom 尚未還原會導致偏移。
- **修正**：改為先 `zoomTo` 再 `translateTo`，確保 translate 計算時 zoom 已是正確值。

### 使用 Find 後滾輪縮放失效
按 Ctrl+F/G 開啟搜尋時，G6 Shortcut 的 `recordKey` 收到 keydown 但因 Input 搶走 focus 而收不到 keyup，導致修飾鍵永遠卡住。
- **修正**：於開啟 / 關閉搜尋時呼叫 `clearKeyState()` 對 container 補發合成 keyup 事件。

---

## 優化

### Viewport 追蹤機制重構
- 移除 `AFTER_TRANSFORM` 事件監聽與 `_isRendering` flag
- 改為只監聽 `canvas:dragend`（使用者手動拖拉）和 `canvas:wheel`（使用者滾輪縮放）來記錄 viewport 狀態
- 程式化操作（`focusNode`、`expandElement`、`render` 等）不再汙染已保存的 viewport 值

### 搜尋流程優化
- 新增 `closeSearch()` 統一關閉搜尋流程，關閉後將 focus 還給 canvas
- 搜尋列支援 Escape 鍵關閉

---

## 新功能

### 開檔自動收合深層節點
開啟行為樹時，深度 ≥ 2 的節點預設收合。
可在 `graph.ts` 頂部調整閾值與懸停延遲：
```
const INITIAL_COLLAPSE_DEPTH = 2;  // 自動收合的起始深度
const HOVER_EXPAND_DELAY = 200;    // 懸停展開延遲（毫秒）
```

### 滑鼠懸停自動展開
滑鼠停在收合節點上方超過 `HOVER_EXPAND_DELAY` 後，自動展開直接子層。

### Ctrl + 展開 → 遞迴展開整棵子樹
點擊 `+` 按鈕或懸停觸發展開時，按住 Ctrl 可一次展開該節點所有後代。

---

## 搜尋定位優化

### 搜尋定位只展開目標路徑
使用搜尋 / Next / Prev 定位節點時，只展開從根節點到目標節點的祖先路徑，不再展開整棵樹。

### 初始收合效能 O(n²) → O(n)
改用單次 DFS 建立深度表，取代原本對每個節點逐一查詢祖先鏈的做法。

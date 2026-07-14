---
title: "React query"
date: 2026-01-09
tags: ["Tech note"]
database: "Houzz Database"
source: "https://app.notion.com/p/2e26ed462a26805589ded769143ddb12"
notion_id: "2e26ed462a26805589ded769143ddb12"
---

# React Query 簡介
React Query (現稱 TanStack Query) 是一個為非同步狀態（通常是 Server State）設計的強力工具。它不只是 Fetch 資料，更是一個記憶體快取管理員。它負責處理 loading、error、cache、自動重新獲取以及資料同步，能大幅減少 useEffect 的混亂。
1. 使用時機
- 非同步資料獲取：需要從 API 拿資料時。
- 全域資料共享：如使用者登入資訊、全域設定，不需要透過 Prop Drilling。
- 快取與效能優化：減少重複發送相同的 API 請求。
- 背景更新：當使用者切換標籤頁回到 App 時，自動確保資料是最新的。搭配 Persist Query Client
由於 React Query 預設將資料存放在記憶體 (RAM)，一旦網頁重新整理，資料就會消失。
- 目的：將快取資料同步到 localStorage 或 sessionStorage。
- 好處：使用者重新整理網頁後，不需要重新執行 Google 登入流程或等待 API，能立即看到上一次的資料。
- 實作提示：可以使用官方提供的 persistQueryClient 插件。
# 關於過期與回收機制
這是 React Query 最核心的兩個計時器：
<table>
<colgroup>
<col width="110">
<col width="110">
<col width="110">
</colgroup>
<tr>
<td>概念</td>
<td>描述</td>
<td>預設值</td>
</tr>
<tr>
<td>staleTime (過期時間)</td>
<td>資料從「新鮮」變「陳舊」的時間。在新鮮期間，React Query 不會主動重新 fetch。</td>
<td>0</td>
</tr>
<tr>
<td>gcTime (回收時間)</td>
<td>當該 QueryKey 沒有組件在使用時，資料在快取中存留的時間，到期後會從記憶體清空。</td>
<td>5 min</td>
</tr>
</table>
# Methods
## UseQuery
- 定義規則：它是定義一個 queryKey 應該如何獲取資料（queryFn）以及其生命週期（staleTime、gcTime）的地方。
- 覆蓋規則：在 useQuery 內的設定會覆蓋 QueryClient 的全域預設值。
## SetQueryData
- 同步寫入：這是一個同步函數，用來直接手動更新快取。
- 行為：當你手動寫入資料時，React Query 會將該資料標記為 Fresh，並開始倒數 staleTime。它不會改寫你的規則，只是「**重啟**」計時。
# Google Auth 管理實例
針對 Google SDK (基於 Callback) 的特殊性，我們整合以上概念：
1. useQuery 設定 key 以及 stale time 等相關選項，此時 cache 還是空的，所以回傳 default value
2. Default value 代表現在狀態是 signed out，觸發後續的 Google auth
3. 登入成功，取得 access token，呼叫 useMutation 的方法並使用 setQueryData 更新 cache （同時重啟計時）
4. Cache 過期，回傳 default data，於是把網頁 sign out

Code example
```yaml

const USER_KEY = ['USER_DATA'];
```
```typescript
// 1. 儲存：利用 Mutation 處理 Google 回傳的結果
export const useSaveUser = () => {
const queryClient = useQueryClient();
return useMutation({
mutationFn: (user: User) => Promise.resolve(user),
onSuccess: (data) => {
// 手動存入快取，開始根據 staleTime 倒數
queryClient.setQueryData([USER_KEY], data);
},
});
};
```
```yaml
// 2. 取用：精準同步 JWT 的 exp 效期
export const useUser = () => {
return useQuery({
queryKey: [USER_KEY],
queryFn: async () => null, // 預設資料
// 核心：動態計算過期時間
staleTime: (query) => {
const user = query.state.data as User;
if (!user?.exp) return 0;
// 讓 React Query 的 staleTime 完全等於 JWT 的剩餘壽命
const remaining = (user.exp * 1000) - Date.now();
return Math.max(0, remaining - 30000); // 提前 30 秒過期以求保險
},
gcTime: 1000 * 60 * 60 * 24, // 記憶體保留 24 小時
});
};
```
# 其他補充
- Placeholder Data vs Initial Data：
	- initialData 會被存入快取並參與 staleTime 運算。
	- placeholderData 僅供展示，不會被存入快取。
- Invalidation：使用 queryClient.invalidateQueries(\[USER_KEY\]) 可以無視任何 staleTime，強制該資料失效並重新獲取。
# 常見問答 QA
Q1：為什麼我用了 setQueryData，它還是會執行 queryFn？<br>A：通常是因為你的 staleTime 過短或預設為 0，資料寫入後立刻變 stale，React Query 偵測到後就會自動執行背景更新。<br>Q2：JWT 的 exp 是 1767892378，這是什麼？<br>A：這是 Unix Timestamp (秒)。代表 2026/1/8 19:52:58 (GMT+8)。在 JavaScript 中計算時必須乘以 1000 轉為毫秒。<br>Q3：不同的 QueryKey 可以有不同的過期時間嗎？<br>A：可以。 雖然全域有預設值（例如 12 小時），但只要在各自的 useQuery 裡設定 staleTime，就能獨立覆蓋規則。<br>整理日期：2024-05-22<br>標籤：#ReactQuery #WebDev #Authentication #JWT

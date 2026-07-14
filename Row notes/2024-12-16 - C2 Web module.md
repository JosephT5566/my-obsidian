---
title: "C2 Web module"
date: 2024-12-16
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/15e6ed462a26801db230c87cb6d4a585"
notion_id: "15e6ed462a26801db230c87cb6d4a585"
---

# What is web module?
在 C2 開發中，這是一個讓 C2 頁面可以使用 Jukwaa 的方法
- Pros: 可以將 element 的運作邏輯和 style 交給 Jukwaa，省去在 c2 上撰寫 vanilla JS / jquery 的痛苦。
	- ex: 之前要加 recommendation module 在 c2 order conf page ，但不可能為了要加 product recommendation 特地 migrate 整個 c2 page，所以只剩在 c2 造輪子或是借 Jukwaa 的輪子
- Cons: Debug 過程會比較痛苦，需要 C2/Jukwaa 都拉進來看，還會牽涉到 C2thrift/GQL。但整體算是利大於弊
## 運作過程
1. 在 C2 頁面跑起來的時候，會發送 ajax 跟 jukwaa 要求對應的 module
2. Jukwaa 會將對應的 handler/view 打包成 module 傳給 C2
3. 於是 C2 頁面中就有一塊會是 Jukwaa 的 React component
# Example (user profile page)
假如我們打開 User profile page: [https://www.stghouzz.com/browseOrders](https://www.stghouzz.com/browseOrders)
可以發現只有上面這部分可以找到 React Component，當我們要修改 Header Tabs (Navigation component) 就需要尋找並觀察 web module 的互動
![](https://prod-files-secure.s3.us-west-2.amazonaws.com/300ece61-c38a-44db-b7d8-e63f52d8b52e/7d44b9ee-197a-42ef-b78a-6a7afde677c7/%E6%88%AA%E5%9C%96_2024-12-16_%E4%B8%8B%E5%8D%886.41.12.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466RL7LW522%2F20260714%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260714T080835Z&X-Amz-Expires=300&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEEgaCXVzLXdlc3QtMiJHMEUCIQCrJfCwGSHEhXgS0UUJXGR7gD8nWIZspngEh6qE7c7B0wIgYjUkRzQi%2FHQzy0J5qCUdJ%2BZJggt%2BWx53U%2BPp3itVNukq%2FwMIERAAGgw2Mzc0MjMxODM4MDUiDMeZGSoTaI%2Biowj2qyrcAx6%2BqhiaxUMG4NFTcJaI4BFSruY5m8OFXBKnadSCdCfnWa0W1vAbiqEJED2jNYVZy0D5IbehuyAAfzKX8XoZIP8sbLXQLhMwnKnbrE1xOmbkGsIjMJiFZn%2FrV18iVVHM%2Bgl79YmkwZvSnE0DlZSycF68F6%2B9lQFZvtViIqFpCobezEOhMtcUdhcW%2FTe08LtzIHpi8KBiewXqZfBwOfIngBcfKDMYRicBXMyGZxskGwOfdoaNnuRssor%2Bc21rFiyG7RADRUJPcHFRJMxSba5GlDRU6S406zBugws5%2FCgCxRMtIGXwFdIMU9NMmpJsQZV2JOHf0QQsu%2FS4bvMI6RkxyEeopRmER4BfiEDQ9VbanIojPJqacYaTvPkUGz6CfkRHZFOo%2Bf9FadJxr%2BRYCTJd8Haks8CMf3scE6AEkFh0ewduHZbi7tx6CWuRTLRcCfvOaMk1Ld12hjWnZhVmjY1blGEyulkNA1gCoVX8lA2YYRBUHu00Sr1qs96bjU7MCZD9XQrRh2EC4morXGEJVoJXDG0fDhR5JokEShvOWoSOnFZT6NBIwvcqiUXFqQ2l%2FgC%2BHhoTb0UyhF32BO0GDmHFA0X7BdTeSrx29pHz4FSYzzRA%2BxhVBQnebSG2wEyYMJjN19IGOqUBapz7av8kuII0Wu7plc7lEYmSUyTm3%2BpUT%2FLcaqrCK84Qjl97G46VL6jHHHIiQRQd5zgciPXF0BP307pUNtqBsXhgVOzYNHyhGjVZz8ccZcEmsPL8FKpdmoEFu4rLPAsf1dM%2FHJn3e%2BAlDvD2cnMt2Myt9D8RGcrw42GU0YJw3woxGpDU9KBoYojHzJesz4GMUcUSSgjW4xNsmL%2B76pFG0vxthBgL&X-Amz-Signature=18704325bf48be731eb519a22425c7699dfd2bee6c7e003946e056f23a27dcac&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
## Development process
目標：新增新的 tab, Order Messages 到 Header Tabs 中
Before we dig in，先描述一下架構運作過程
1. C2 送 request 給 Jukwaa web module
2. web module 根據 request 的 param，打 gql 給 c2 service (or other services)
3. 然後 data fetcher 把 data set 到 store
4. render 結果到 jukwaa component
### 1. 確認 Web module in the Jukwaa
當我們用 dev tool 可以發現 Header Tabs 是屬於 component `Navigation` ([code](https://github.com/Houzz/jukwaa/blob/master/apps/user-profile/components/navigation/Navigation.jsx))，繼續往上找可以發現他是屬於 web module `UserProfileHeaderModule` (appConfig [code](https://github.com/Houzz/jukwaa/blob/master/apps/user-profile/AppConfig.js#L286))，並且其使用了 handler `UserProfileHeaderModuleHandler`
### 2. Web module 如何設定資料
1. `Navigation` 渲染的資料來自於 `userProfileStore.navigationTabs`，而 `userProfileStore` 則是在 `UserProfileHeaderModuleHandler` and `UserProfileDataFetcher` 執行 Update
2. `Navigation` 使用的 `navigationTabs` 是在 `UserProfileDataFetcher` 中完成，可以看到這裡執行了 `getUserProfileNavigation.gql` 來取得資料 ([code](https://github.com/Houzz/jukwaa/blob/master/apps/user-profile/data-fetchers/UserProfileDataFetcher.js#L94-L104))，至於 gql 要使用的 param 從哪來，等等再說
3. 根據 GQL [schema](https://github.com/Houzz/jukwaa/blob/master/apps/user-profile/data-fetchers/queries/getUserProfileNavigation.gql) 找到 `Navigation.js` 的 `getProfileNavigation()` ([code](https://github.com/Houzz/jukwaa/blob/master/graph/Navigation/Navigation.js#L209-L214))，因此我們知道這裡會使用到 C2 的 navigation service 的 `getProfileHeaderTabs()` method
目前我們可以用 Graphouzz playground 測試 schema 從 c2 取得的資料，目標是讓 `navigationTabs` 多出 “Order Messages”
### 3. 檢視 C2 的 Navigation service
1. `getProfileHeaderTabs()` method 在 `NavigationServiceHandler.php` 中 ([code](https://github.com/Houzz/c2/blob/master/services/handlers/navigation/NavigationServiceHandler.php#L56-L60))，接著 `$headerTabs` 會從 `ProfileHeaderUtils::getInstance()->getHeaderTabs()` 取得。
2. 因此我們需要設法讓 `$this->finalTabs` 新增一個 order messages 的欄位，最後 `getHeaderTabs()` 會生成對應的 `navigationTabs` ([PR](https://github.com/Houzz/c2/pull/39875))
### 4. C2Thrift
目前我們有改到 C2 Navigation service 相關 api 的資料格式（傳入的 `GetProfileHeaderTabsRequest.ProfileTabs` 多一個 ORDER_MESSAGES）
因此需要在 C2thrift 處作出新增 ([PR](https://github.com/Houzz/c2thrift/pull/2966))
<callout icon="✅" color="gray_bg">
	目前為止，我們搞定了 c2 service 回傳 **respond** 的資料部分，確保了 `navigationTabs` 中新增了 Order Messages 讓 Jukwaa `Navigation` component 可以多出一個 tab
	但其中有一個 `selected` 的變數好像怪怪的，一直不會切換成 `true`，讓 tab 不會變成 active
</callout>
### 5. Web Module request param
後來繼續追了 C2，發現 `selected` 的設置是根據 GQL 傳入的 `currentTab` param ([code](https://github.com/Houzz/c2/blob/master/platform/navigation/ProfileHeaderUtils.php#L205))
而讓 GQL 送出對應的 `currentTab` 則又是根據 web module request param ([code](https://github.com/Houzz/jukwaa/blob/master/apps/user-profile/modules/user-profile-header-module/UserProfileHeaderModuleHandler.js#L78))，這個 request param 是 C2 在使用 ajax 時設置的 ([set query](https://github.com/Houzz/c2/blob/master/web/modules/ProfileHeaderModule.php#L164))
設置條件是根據造訪的 c2 頁面，不同的頁面會設置不同的 `currentItem`，如在 `/browseOrders` page `req.query.currentTab` 就是 `PURCHASES = 9` (message page 的設置 [code](https://github.com/Houzz/c2/blob/master/web/BrowseMessagesRequest.php#L292))
同時 Jukwaa 的 GQL js method 也需要多出對應的變數規範 (Jukwaa [PR1](https://github.com/Houzz/jukwaa/pull/34884), [PR2](https://github.com/Houzz/jukwaa/pull/34886))
<callout icon="✅" color="gray_bg">
	再加上 C2 的相關頁面調整 ([PR](https://github.com/Houzz/c2/pull/39947))，至此，我們完成了 c2 → Jukwaa **request** 的部分，整個互動也完成了
	整個找尋過程就差不多這些，還有額外的變化就再到對應的 repo 中尋找
</callout>

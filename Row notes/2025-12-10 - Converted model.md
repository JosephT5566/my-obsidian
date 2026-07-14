---
title: "Converted model"
date: 2025-12-10
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/2c46ed462a2680a1ab7dde73052adf99"
notion_id: "2c46ed462a2680a1ab7dde73052adf99"
---

# 簡介
## Catalog
一個我們用來記錄產品的結構，先前是用來作為從合作廠商（eg, Home Depot）匯入的產品接口。
可以從 left rail: Company → Library → Catalogs → Brand Catalogs: The Home Depot → 然後點進類別找到產品。如果我們對產品點擊 **Add to Library**，就能在 floorplan 的 **My library** 中找到這項產品。或是可以在 Company → Library → My Items 裡找到
同理，先前我們在 2D to 3D 時新增的 catalog 也會出現在 My Items 中
## Converted model
Catalog 的延伸，我們需要針對 catalog 做 3D model，需要新的 columns 紀錄但又不想更動太多 catalog data，因此新增了 converted model table，並在 catalog 新增對應的 foreign id 連接
- `ConvertedModelData.externalSourceId` ↔ `CatalogItem.id` (the product ID)
- `ConvertedModelData.convertedModelId` ↔ `CatalogItem.convertedModelId` (the generated 3D model ID)
# **Overview of ConvertedModel Variables Flow**
想知道我們的 `Automate2Dto3DModelProvider` 中存的 states，會是怎麼跟相關的 components, 如 `AutoMateProductReviewContainer`，做互動的
## **1. State Variables in ****`Automate2DTo3DModelProvider.tsx`**
```javascript
const [convertedModelsToReview, setConvertedModelsToReview] = useState<ConvertedModelData[]>([]);
const [convertedModelsInPending, setConvertedModelsInPending] = useState<ConvertedModelData[]>([]);
const [convertedModelToReview, setConvertedModelToReview] = useState<CatalogItem>();
```
## 2. **Data Types and Relationships**
**ConvertedModelData** (backend data structure):
```javascript
interface ConvertedModelData {
    convertedModelId: string;
    externalSourceId: string;
    externalSourceType: string;
    thumbnailImageId: string;
    ownerId: string;
    permission: string;
    status: string;
}
```
**CatalogItem** (UI data structure):
```javascript
interface CatalogItem {
    id: number;
    title: string;
    imgUrl: string;
    convertedModelId: number;
    catalog3dModelId: number;
}
```
## 3. **Status Flow**
The model goes through these statuses:
- **PENDING** → Model is being generated
- **REVIEWING** → Model is ready for user review
- **ACCEPTED** → User accepted the model
- **REJECTED** → User rejected/discarded the model
## 4. **How These Variables Get Updated**
### **A. Initial Load & Periodic Updates**
在 `Automate2DTo3DModelProvider` 中
```javascript
const updateModelStatus = useCallback(() => {
    getConvertedModelsToReview();
    getConvertedModelsInPending();
    setShouldRefetch(true);
}, [getConvertedModelsToReview, getConvertedModelsInPending]);

useEffect(() => {
    Promise.allSettled([updateModelStatus(), updateConvertImageToModelMonthlyUsage()]);
}, [updateModelStatus, updateConvertImageToModelMonthlyUsage]);
```
當 component mount and fetches 時就會更新
### **B. When User Clicks on a Product in ****`MyProductsMenu`**** (Crucial !!)**
<columns>
	<column>
		![](https://prod-files-secure.s3.us-west-2.amazonaws.com/300ece61-c38a-44db-b7d8-e63f52d8b52e/5bb8a3fb-0c3b-4cff-89e5-321315b9d501/%E6%88%AA%E5%9C%96_2025-12-10_%E5%87%8C%E6%99%A81.43.14.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466QGSKZQME%2F20260714%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260714T080844Z&X-Amz-Expires=300&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEEgaCXVzLXdlc3QtMiJHMEUCIQCTTSfmpnxpZ78J8puGtugNrETUcKkoSgsREV6ydx7t1AIgd0M6qlrfv9Ty1BuHtE7XSffvv0ibBgl%2FbbH5rZxPmSwq%2FwMIERAAGgw2Mzc0MjMxODM4MDUiDKOv99rWXotiyAIESCrcA1tUXaPRNntbx%2BcEG5L3JAYOoaB2zKg27cayzyH%2BLZ4hWg56h75%2F%2BOGbfsjE4InD4POU%2BPfQ%2BBSJgqYj0yPSQhm3I0fCHNfLJErDRQGzychfkazSz2D1ECvNm32Q7cSUkUPUAVQDRG6X%2Bjf9wiAxyBMIrcILxZ%2FxEIMTFAC02RBYKPBPQDuNbebzaVPDNF0rRPt3ah%2FUsK7b9AcMeeaiGYwcKJ42TWKnOjGrBXDjgENv319lxyXt5C65Ow70X93wyRZ%2BzU32F7%2F49ZxJu8W%2B01Lu3mFn66z1JbXvI%2FqwUh8lv14uf4phkonq11%2BfIBiyZVhmJOXcm8iFtfELIdYuW7FbuC1HCmvK6Cy1P9uF1NaBr6Rd%2FoWI36DQcsWdlr92n5i%2Fl%2FX8DZmToBsUQ4fxlK5cDpsh06Cenh%2BLO%2Fmbv3DO9ynp54dv5XtKscnJzI%2BOl0ks3Wlf2XpmaGWra8M9%2F5i8ewxozwo7fFYV0ECA3FGKE10nODkBFpOutjJHwPMU4PNBRvMN5JdLHjISMDqobupb47bqCXoTRmY19haJkWdkyTepvPG8%2FVVZJFcRgXth9ZSMnu5PpL8Fe8ez9cKEYdaWZK%2BF3bjW9QrrHvqIwMOAe%2FtxdZZNrUlB2YUzMJPN19IGOqUBTBED0UDl23Z6PbqYbKhZxkKF45AmgQKSyX02%2BB%2Fr%2BwsR3ra6HhijhRRI8kObk1pKnu6GsDr%2FHeRW2SuDDNdFnEaxge3ZesIq99QyV%2BrUE28YRH5gSb5%2FjeoXxlJkB0dwITzjlBbPewxjHvAzyqlFcqo6GfTnPL2w6pF7bd44s4fk3Ntnxojo0T0Den7Jl3TGJ%2BsyWid%2BrJxdaARaycfyMt0ZoRlt&X-Amz-Signature=fc7b8c6ef92945abb470df571938a54b8ed60762be600402cc3ceee7266aa5b6&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
	</column>
	<column>
		當點擊 My Products 中的 My Library items，就會觸發 `handleItemClick` ，其中的 `item` param 是 catalog item
	</column>
</columns>
```javascript
const handleItemClick = useEventCallback(async ({ item }) => {
    ...
}
```
檢查 `catalog` 是否存在於 `convertedModals` 中，是的話就會顯示 `AutoMateProductReviewContainer` modal 來執行 **review** step
```javascript
const convertedModel = convertedModelsToReview.find((model) => Number(model.externalSourceId) === item.id);
if (convertedModel) {
    setConvertedModelToReview({
        ...item,
        convertedModelId: convertedModel?.convertedModelId,
    });
    setShowReviewModal(true);
    return;
}
```
![](https://prod-files-secure.s3.us-west-2.amazonaws.com/300ece61-c38a-44db-b7d8-e63f52d8b52e/0a79188b-db3c-465e-9ce1-3403902d19e0/%E6%88%AA%E5%9C%96_2025-12-10_%E5%87%8C%E6%99%A81.52.19.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466QGSKZQME%2F20260714%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260714T080844Z&X-Amz-Expires=300&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEEgaCXVzLXdlc3QtMiJHMEUCIQCTTSfmpnxpZ78J8puGtugNrETUcKkoSgsREV6ydx7t1AIgd0M6qlrfv9Ty1BuHtE7XSffvv0ibBgl%2FbbH5rZxPmSwq%2FwMIERAAGgw2Mzc0MjMxODM4MDUiDKOv99rWXotiyAIESCrcA1tUXaPRNntbx%2BcEG5L3JAYOoaB2zKg27cayzyH%2BLZ4hWg56h75%2F%2BOGbfsjE4InD4POU%2BPfQ%2BBSJgqYj0yPSQhm3I0fCHNfLJErDRQGzychfkazSz2D1ECvNm32Q7cSUkUPUAVQDRG6X%2Bjf9wiAxyBMIrcILxZ%2FxEIMTFAC02RBYKPBPQDuNbebzaVPDNF0rRPt3ah%2FUsK7b9AcMeeaiGYwcKJ42TWKnOjGrBXDjgENv319lxyXt5C65Ow70X93wyRZ%2BzU32F7%2F49ZxJu8W%2B01Lu3mFn66z1JbXvI%2FqwUh8lv14uf4phkonq11%2BfIBiyZVhmJOXcm8iFtfELIdYuW7FbuC1HCmvK6Cy1P9uF1NaBr6Rd%2FoWI36DQcsWdlr92n5i%2Fl%2FX8DZmToBsUQ4fxlK5cDpsh06Cenh%2BLO%2Fmbv3DO9ynp54dv5XtKscnJzI%2BOl0ks3Wlf2XpmaGWra8M9%2F5i8ewxozwo7fFYV0ECA3FGKE10nODkBFpOutjJHwPMU4PNBRvMN5JdLHjISMDqobupb47bqCXoTRmY19haJkWdkyTepvPG8%2FVVZJFcRgXth9ZSMnu5PpL8Fe8ez9cKEYdaWZK%2BF3bjW9QrrHvqIwMOAe%2FtxdZZNrUlB2YUzMJPN19IGOqUBTBED0UDl23Z6PbqYbKhZxkKF45AmgQKSyX02%2BB%2Fr%2BwsR3ra6HhijhRRI8kObk1pKnu6GsDr%2FHeRW2SuDDNdFnEaxge3ZesIq99QyV%2BrUE28YRH5gSb5%2FjeoXxlJkB0dwITzjlBbPewxjHvAzyqlFcqo6GfTnPL2w6pF7bd44s4fk3Ntnxojo0T0Den7Jl3TGJ%2BsyWid%2BrJxdaARaycfyMt0ZoRlt&X-Amz-Signature=bc206716e5b9632c27ab9635fe6c75a165601a40096498f3b38b76e2a3e01618&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
若不存在，表示 Product 尚未 convert，就會執行 `handleCreateBtnClick(`*`item`*`)`，並打開`AutoMateProductContainer` modal
![](https://prod-files-secure.s3.us-west-2.amazonaws.com/300ece61-c38a-44db-b7d8-e63f52d8b52e/a4ea38fd-0b8d-4059-a43b-34fd9dfb7b80/%E6%88%AA%E5%9C%96_2025-12-10_%E5%87%8C%E6%99%A81.59.37.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466QGSKZQME%2F20260714%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260714T080844Z&X-Amz-Expires=300&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEEgaCXVzLXdlc3QtMiJHMEUCIQCTTSfmpnxpZ78J8puGtugNrETUcKkoSgsREV6ydx7t1AIgd0M6qlrfv9Ty1BuHtE7XSffvv0ibBgl%2FbbH5rZxPmSwq%2FwMIERAAGgw2Mzc0MjMxODM4MDUiDKOv99rWXotiyAIESCrcA1tUXaPRNntbx%2BcEG5L3JAYOoaB2zKg27cayzyH%2BLZ4hWg56h75%2F%2BOGbfsjE4InD4POU%2BPfQ%2BBSJgqYj0yPSQhm3I0fCHNfLJErDRQGzychfkazSz2D1ECvNm32Q7cSUkUPUAVQDRG6X%2Bjf9wiAxyBMIrcILxZ%2FxEIMTFAC02RBYKPBPQDuNbebzaVPDNF0rRPt3ah%2FUsK7b9AcMeeaiGYwcKJ42TWKnOjGrBXDjgENv319lxyXt5C65Ow70X93wyRZ%2BzU32F7%2F49ZxJu8W%2B01Lu3mFn66z1JbXvI%2FqwUh8lv14uf4phkonq11%2BfIBiyZVhmJOXcm8iFtfELIdYuW7FbuC1HCmvK6Cy1P9uF1NaBr6Rd%2FoWI36DQcsWdlr92n5i%2Fl%2FX8DZmToBsUQ4fxlK5cDpsh06Cenh%2BLO%2Fmbv3DO9ynp54dv5XtKscnJzI%2BOl0ks3Wlf2XpmaGWra8M9%2F5i8ewxozwo7fFYV0ECA3FGKE10nODkBFpOutjJHwPMU4PNBRvMN5JdLHjISMDqobupb47bqCXoTRmY19haJkWdkyTepvPG8%2FVVZJFcRgXth9ZSMnu5PpL8Fe8ez9cKEYdaWZK%2BF3bjW9QrrHvqIwMOAe%2FtxdZZNrUlB2YUzMJPN19IGOqUBTBED0UDl23Z6PbqYbKhZxkKF45AmgQKSyX02%2BB%2Fr%2BwsR3ra6HhijhRRI8kObk1pKnu6GsDr%2FHeRW2SuDDNdFnEaxge3ZesIq99QyV%2BrUE28YRH5gSb5%2FjeoXxlJkB0dwITzjlBbPewxjHvAzyqlFcqo6GfTnPL2w6pF7bd44s4fk3Ntnxojo0T0Den7Jl3TGJ%2BsyWid%2BrJxdaARaycfyMt0ZoRlt&X-Amz-Signature=00bf42ca7a1b3af394586cea1b33cab8faf743c27178856c1a05615e0d5086ce&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
### C. **When User Accepts the Model in the review step ****`AutoMateProductReviewContainer.jsx`**
會在 **`AutoMateProductReviewContainer`** 中執行 `handleConfirmProductDetails`，呼叫 `shareThreeDProduct.gql` 來將 model status 更新成 **ACCEPTED**
### D. **When User Rejects/Discards the Model:**
則是執行 `onDiscardProduct` 將 status 更新成 **REJECTED**
### E. **Real-time Updates via WebSocket**
在 `Automate2DTo3DModelProvider` 中，有設置 `useWebSocket`
When the backend finishes processing, it sends a WebSocket message that triggers `updateModelStatus()` to refetch the lists.
## 5. **Summary**
- **`convertedModelsToReview`**: Array of all models with **REVIEWING** status (ready for user review)
- **`convertedModelsInPending`**: Array of all models with **PENDING** status (currently being generated)
- **`convertedModelToReview`**: The **single** model currently being reviewed by the user (combines `CatalogItem` + `convertedModelId`)
These are updated via:
1. **Initial fetch** on component mount
2. **WebSocket messages** when backend completes processing
3. **Manual refetch** via `updateModelStatus()` after user actions
4. **Status changes** via `shareThreeDProduct` (accept) or `rejectConvertedModel` (reject) mutations
The `convertedModelId` is the **key** that links the `CatalogItem` (product) to the `ConvertedModelData` (3D model), while `externalSourceId` links back to the original product `id`.
## New Painting/Rugs type
這是新的 feature，將原本的 product 拆分為 3D model 的 furniture，或是 painting/rug
### Old flow (furniture)
1. Create catalog item
2. Go to `AutoMateProductContainer` to create converted model (3D) item
	1. Or click on the item in the My Product to open
	2. Or linked after the Single Entry Point Analysis step
	3. 這一步也可以讓 user 選擇是否要加入 public shared library (new feature, 讓大家可以在 source 裡看到)
3. (After the covert finished) click on the item in the My Product and go to `AutoMateProductReviewContainer` for the review flow
在 Single Entry Point 時，當
### New flow (painting, rug)
1. (After the covert finished) click on the item in the My Product and go to `AutoMateProductReviewContainer` for the review flow. （但不會有 Orientation step）<br>但我們會根據 Converted model 是 painting or rug 來顯示不同的內容，因此我們會在 Converted model table 多一個 `type` column

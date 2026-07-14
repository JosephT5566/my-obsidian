---
title: "5/24 weekly update"
date: 2026-05-24
tags: ["Interview 2026"]
source: "https://app.notion.com/p/3666ed462a268020a967d01cc214fe5a"
notion_id: "3666ed462a268020a967d01cc214fe5a"
---

## Work on the Resume
根據先前存的 performance reviews
<mention-page url="https://app.notion.com/p/1913ecb0bf6945b9b6f772d3955d27d2"/>
<mention-page url="https://app.notion.com/p/18a6ed462a26802a9c9dfc2395a85fda"/>
<mention-page url="https://app.notion.com/p/24f6ed462a268077885ff0561e71bb71"/>
<mention-page url="https://app.notion.com/p/3046ed462a26805abb90fda3665ae4c1"/>
## Leet code
### Trie
Trie: a prefix tree (字典樹). 是一種**樹狀的資料結構**，專門用來處理字串或序列的快速檢索。
可以把它想像成一本**英文字典的索引目錄**。如果你要在字典裡查 "cat"、"car"、"dog"：
- 他們不會各自獨立佔用空間。
- "cat" 和 "car" 因為前面都是 "ca"，所以在樹裡，它們會**共用同一個開頭路徑**。
如果把 arr1 = \[12, 100\] 放進 Trie 裡，它長得像這樣：
```javascript
			[Root]
         |
        (1)
       /   \
     (2)   (0)
            |
           (0)
```
### **Leet code 3043. Find the Length of the Longest Common Prefix**
[link](https://leetcode.com/problems/find-the-length-of-the-longest-common-prefix/description/?envType=daily-question&envId=2026-05-22)
使用 trie tree 的做法，相較於用 `Set` 建立出一個 table，當字串長度越長，Set 需要建立的儲存字串就越多，而 trie tree 最深就是陣列中最大數的長度
```javascript
// 1. 定義定義字典樹的節點
class TrieNode {
    constructor() {
        // key 會是 '0'~'9' 的字串，value 會是另一個 TrieNode
        this.children = {};
    }
}

// 2. 定義字典樹本身
class Trie {
    constructor() {
        this.root = new TrieNode();
    }

    // 插入一個數字字串
    insert(numStr) {
        let curr = this.root;
        for (let char of numStr) {
            // 如果這個數字路徑還沒建立，就建立一個新節點
            if (!curr.children[char]) {
                curr.children[char] = new TrieNode();
            }
            // 指針往下走一步
            curr = curr.children[char];
        }
    }

    // 查詢一個數字字串，能匹配到的最長前綴長度
    searchLongestPrefix(numStr) {
        let curr = this.root;
        let length = 0;
        for (let char of numStr) {
            // 一旦發現路徑斷了，就代表最長前綴到此為止
            if (!curr.children[char]) {
                break;
            }
            curr = curr.children[char];
            length++; // 成功走了一步，長度加 1
        }
        return length;
    }
}

/**
 * 主解法函數
 * @param {number[]} arr1
 * @param {number[]} arr2
 * @return {number}
 */
var longestCommonPrefix = function(arr1, arr2) {
    const trie = new Trie();

    // 步驟 A：把 arr1 的數字全部變成字串，並塞進 Trie 樹中
    for (let num of arr1) {
        trie.insert(String(num));
    }

    // 步驟 B：遍歷 arr2，去樹裡查每個數字能走多深，並更新最大長度
    let maxLength = 0;
    for (let num of arr2) {
        const currentPrefixLength = trie.searchLongestPrefix(String(num));
        maxLength = Math.max(maxLength, currentPrefixLength);
    }

    return maxLength;
};
```
該節點如果有多個 children，會長這樣
```javascript
{
    "children": {
        "2": {
            "children": {
                "5": {
                    "children": {}
                }
            }
        },
        "4": {
            "children": {
                "5": {
                    "children": {}
                }
            }
        }
    }
}
```
## AI concepts
[notebooklm.google.com](https://notebooklm.google.com/notebook/2fb9d479-d573-42e2-9183-018ca0eea0ac)
相較於我們一開始使用 web ui 並使用 prompt 跟 ai 互動的做法，近期已經高度發展延伸應用。
別忘了 LLM 只是能夠解讀上下文並回覆的工具。
### System prompt
最初 AI 僅是一個聊天框，用戶發送的消息稱為 **User Prompt**
隨後演進出 **System Prompt**（系統提示詞），將 AI 的角色、性格、背景與用戶對話分開，由系統自動附加發送。
### Function calling
- Agent 依靠自然語言在 System Prompt 中描述工具格式，但 AI 作為機率模型常會回傳錯誤格式。
- 各大模型廠商（如 OpenAI, Google）推出了 **Function Calling** 功能，將工具描述**標準化**（使用 JSON 對象定義名稱、參數等），並從 System Prompt 中抽離出來
### MCP
- 起初 Agent 與工具（Tools）是寫在一起的，但多個 Agent 需要相同工具時會導致代碼重複。
- 為此提出了 **MCP** 協議，將工具轉變為統一託管的服務（**MCP Server**），讓不同的 Agent（**MCP Client**）都能調用
### Agent
- 從單純的聊天機器人，演變為能**自主完成任務**的程序。
- 以開源項目 **AutoGPT** 為代表，開發者預先寫好工具函數（如讀取文件），並在 System Prompt 中告知 AI 這些工具的使用方法。
- AI 根據需求決定調用工具，Agent 則負責執行並將結果回傳給 AI，直到任務完成。
### Skill
**痛點：**
- **記憶負擔：** 提示詞多到連使用者自己都會忘記寫過哪些。
- **Token 浪費：** 如果把所有提示詞一次全部發給 AI，會消耗大量 Token。
- **AI 迷茫：** 過多無關的提示詞訊息會干擾 AI，讓它難以準確回答當下的問題。
為了不把所有內容一次塞給 AI，開發者提出了 **Skill 機制**。簡單來說，**一個不同用途的提示詞就是一個 Skill**。它通常以資料夾的形式存在，核心包含兩個部分：
- **Skill.md：** 存放完整的提示詞內容。
- **Metadata (元數據)：** 對該 Skill 的簡短介紹，放在文件開頭，用於讓 AI 快速了解這個 Skill 的作用。
當 ai 執行中需要特定的 skill 才會轉頭跟 client 拿取相關的 skill md
1. **Discovery (發現):** 當使用者提問時，系統不會發送所有完整的提示詞，而是只把所有 Skill 的 **Metadata（簡短介紹）** 放入系統提示詞中。因為 Metadata 很短，即使 Skill 很多，佔用的 Token 也很少。
2. **Activation (激活):** AI 根據用戶的問題進行語義理解。如果發現問題與某個 Skill（例如「菜譜」）有關，它不會直接回答，而是先發出指令要求客戶端讀取該 Skill 的完整內容（**`skill.md`**）。這個**動態讀取完整提示詞**的過程就是「激活」。
3. **Execution (執行)** 這是 Skill 最強大的地方。Skill 資料夾內除了提示詞，還可以包含**腳本或小程式**。
	1. **按需讀取：** 如果一個提示詞檔案太大，可以拆分成多個子文件，由 AI 決定何時讀取。
	2. **行動力：** AI 可以根據 Skill 中的指示，直接命令客戶端執行特定的腳本（例如把 PDF 轉成圖片）。
	3. **動態生成：** 甚至在 Skill 中告訴 AI 具備哪些函數庫，讓 AI **現場寫程式並執行**來完成任務
## System Design
### **Distributed Transactions**
[Distributed Transactions Explained: 2 Phase Commit vs Saga Pattern](https://www.youtube.com/watch?v=DOFflggE_0Q)
[notebooklm.google.com](https://notebooklm.google.com/notebook/13efa317-ed81-4f28-99db-6db31096020c)
在微服務架構中處理**分散式交易**的挑戰與解決方案。當傳統資料庫的 **ACID **特性因**系統拆分**而失效時，開發者通常必須在**兩階段提交（2PC）與Saga 模式**之間做出選擇。**兩階段提交**雖然能維持強一致性，卻因其阻塞機制容易導致系統效能瓶頸。
**Saga 模式**透過一連串獨立的本地交易與**補償機制 (Compensating Transactions)**來達成**最終一致性**，更適合大規模生產環境。針對 Saga 的實作，又分為**編舞（Orchestration）**和**指揮（Orchestration）**，**Orchestration**架構搭配**交易發件箱模式（Transactional Outbox）**，以確保事件傳遞的可靠性與系統的可維護性。
最後，作者建議應優先考慮合併資料庫以簡化流程，若無法避免分散化，則應選擇能兼顧彈性與錯誤處理的 Saga 方案。

## Ben’s Wedding
拍照 65000，有合作婚紗 + 15000，總共快 8w，算是高標
跑了三個點，點是攝影師提供的
再考慮捧花、交通
他們是自己開車，載攝影師跟造型師


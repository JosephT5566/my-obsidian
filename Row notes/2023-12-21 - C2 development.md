---
title: "C2 development"
date: 2023-12-21
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/00f882be5f06408eafaa1a456d5f552f"
notion_id: "00f882be5f06408eafaa1a456d5f552f"
---

# VScode 使用 debug pods
## Install tsh (deprecated)
<unknown url="https://app.notion.com/p/00f882be5f06408eafaa1a456d5f552f#f1d238ff8f5c4db6bfba1883ba673630" alt="bookmark"/>
1. vscode安裝k8s套件
2. 安裝tsh和tctl: install an older version (12) from  [https://goteleport.com/download/](https://goteleport.com/download/)
3. 登入：`KUBECONFIG=${HOME?}/teleport.yaml tsh login --proxy=teleport.ivyco.net:443 --auth=github`
4. 登入完成後，可以用 `tsh kube ls` 看有哪些 clusters
5. 接著 `tsh kube login CLUSTER_NAME` 就能切換 cluster (eq: `tsh kube login stg-main-eks`)，然後在 vscode 的 k8s 上就能看到了
## AWS CLI config
AWS CLI Configuration for AWS SSO:
<unknown url="https://app.notion.com/p/00f882be5f06408eafaa1a456d5f552f#2236ed462a2680708711c5a4ebe6c8ee" alt="bookmark"/>
先前在 onboarding 時，應該已經做過 aws config 了，可以輸入 `cat ~/.aws/config | grep profile` 來檢查
1. auth your shell (Make sure `aws` is v2 **(2.10.\*) or above**.)
	1. 假設我的 `cat ~/.aws/config | grep profile` 跑出 `profile Stg-Developers-477121734600`
	2. set the profile via an env: `export AWS_PROFILE=Stg-Developers-477121734600`
	3. `aws sso login --profile Stg-Developers-477121734600`
2. EKS config
	1. view the available clusters: `aws eks list-clusters | jq .` 會出現一排 clusters，以其中的 `stg-main-eks` 為例
	2. `aws eks update-kubeconfig --profile <Profile name> --name <CLUSTER_NAME> --alias <CLUSTER_NAME>`
		1. \<Profile name\>: `Stg-Developers-477121734600`
		2. \<CLUSTER_NAME\> : `stg-main-eks`
		3. result: `aws eks update-kubeconfig --profile Stg-Developers-477121734600 --name stg-main-eks --alias stg-main-eks`
	3. 然後就能在 `~/.kube/config` 看到 `AWS_PROFILE` (don’t use the `teleport.yaml` with legacy setup).
3. You’re now setup!
	1. `kubectx stg-main-eks` : Switched to context "stg-main-eks".
	2. `kubectl get deployment -n stghouzz` : display all the clusters
## CCP Development
1. 進入 cluster: teleport.ivyco.net-stg-main-eks
2. 到 /Namespaces 中，將 namespaces 切換到 `c2-codepath` (right click, **use namespace**)
3. 能進入 Workloads/Pods 了，挑選對應的 pods 進入 (right click → attach visual studio code → c2-php-fpm; 認有 c2-php8.xxx 的項目)
	1. 不是 right click pod 本身，而是左邊小箭頭打開，點裡面的子項目
4. vscode 會開啟新的畫面，然後要選擇 container
	1. 如果要改的地方是用 gql，開啟 **c2thrift**
	2. 如果要改 c2/services，也是 **c2thrift**
	3. 如果要改的地方是一般網頁 (c2/web)，開啟 **c2web**

1. 進入後 vscode 會先安裝對應的 libs，接著左上角檔案總管可以開啟 c2 資料夾 (/home/clipu/c2)
2. 從 log/log_\*\*\* 可以看到印出來的 log
	1. 如果沒有印出來，可能是開錯 cluster了
	2. 使用log: `C2Session::getInstance()->log("MP_DEBUGHAHAHA:" . json_encode($response));`
3. 使用 ccp 搭配 cluster([demo](https://drive.google.com/file/d/15VyKN1SIgCL9I9pv_MB3S7XMfwep2U4q/view?usp=sharing))：開新的 branch 後，到 #shopping-mall 下 `hzbot deploy <branch-name> --force` 就會執行 ccp，完成後就能在 `{cluster}/Workloads/Deployments` 中找到新開的 ccp (eg: `c2web-mpg-1322`)。接著一樣完成上面的步驟開啟資料夾，接著進到 staging 並用機器人給的 codepath cookie，然後就能即時看到改動了
4. 如果是改動 c2/js 底下的檔案，需要再額外 enable `Debug JS` / `Debug CSS` 才會開啟即時更新
	![](https://prod-files-secure.s3.us-west-2.amazonaws.com/300ece61-c38a-44db-b7d8-e63f52d8b52e/e90e0efa-1060-4334-a06e-f34a3f077efe/%E6%88%AA%E5%9C%96_2024-05-23_%E4%B8%8B%E5%8D%886.36.58.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB4666QP3FOAM%2F20260714%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260714T080819Z&X-Amz-Expires=300&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEEgaCXVzLXdlc3QtMiJGMEQCICaIoGgH1uJyPPSfhbP7R21LuLe2e8yzpO54WL7hc3AcAiAEuRSyA6N5EuVHOAvuTl0yTtV8vgRLhVfsiv1bGg7O5Sr%2FAwgREAAaDDYzNzQyMzE4MzgwNSIMIVhpIYmjbgnoceX0KtwD1EmvLXVkyWgAqSppFHGtS%2FyfpucIiByesAsq8jM1GS1GJb9WwfcpSkIgOYMORiENzQyfcaT6fAWIEiJ2gMSdPL5R5mVUh6YvHJNyK0JR8Qifm4t%2By%2BiH8nEZvekMsIrUj68O0gJqb6uhsXkRF2CQJgJxFfne2yQTk8o6bpUGiMpvls0FIGn6C7y9PozIRtF5maLw4kuEYObTzh%2BfqQsMVtCivhL0PgcDVq4UDCv2TJxGrq0QpwTDom1%2FMiHZ0QP0D4i3qMHIf1MlDttitxKFsK25hyMeYdcpEk%2BtC3TgstiF8mYS10aa3EXTyIUhEC76EcAVAoPN2Z0xAf67pldDpNI53ogOPuhpkXy9DakCFeSWBfJs2t2lkx1%2BT%2BiLxDUcUF7rQL5aPtLkaa%2BRzOGIZTk7%2BXIGamXRAYHVnU6BMzDBS%2BDGZKfYsFsUSrhz3remMWTX%2FNTvDGaV%2FfYKIH7sR2J1vyA6OfJLkZKkGVXLeDtO01LVTcbzGAR4xjlCbVuw2MfB97BLlD0m4vxXtdDhLpeWZ8Qz%2BBbgMn82EwIwj3vSz4joJUJM5%2B%2FFGh%2FbIejkPM%2BhKGzAxF8u1oO46LiVPi%2Bm0MyGZdL6TuEK0YVs8oSsnXN6uRd1nSVgPIsw1M3X0gY6pgHIcJVwSYsGzDo2RdsEJbMPQdbrqUoomrjLXKwaDjuvGOu%2Be9VnocSNeCZHkiHTMSJhL%2BeT6bumu3dUFcISzCLiJiGy4ypdRBTwNrEASe%2F4Y4aHvM9pB60f2FnwXJU19eatTTT23h0hudZ0%2BmI92pJ28wTMMsLKOj8JPI9yfzvSQ1yysMvwXJcecd2nJQrl91h0VXmp2ZQDerW%2F9A4TlYpTCalaBq3N&X-Amz-Signature=881e268dde25fd1c4346c1cbcced6e8b764eef80ba61bc8d1d00b3c367b49085&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
5. 如果有更動到 thrift 相關，不能直接改動 c2/c2thrift 這個 submodule，而是要去 clone c2thrift repo 並更改其中的內容，更動 merge 到 master 後，才回去 c2 update submodule ([readme](https://github.com/Houzz/c2thrift))
	1. 單純 local testing 的話，可以直接改動 c2/c2thrift or jukwaa/c2thrift，完成後在 c2 跑 `./services/generateC2Thrift.sh` ，但 local 的改動不能直接 push 上去
# Thrift deployment
在改動完 c2thrift 的 repo 後，我們還需要 bump c2 and jukwaa 內的 c2thrift folder，更新 c2/c2thrift 和 jukwaa/c2thrift submodule pointer 連到 repo 最新的版本。
bump 完，c2/jukwaa 才能使用到最新的 c2thrift 設定
可以參考 [https://github.com/Houzz/c2thrift/tree/master?tab=readme-ov-file#deploying-changes](https://github.com/Houzz/c2thrift/tree/master?tab=readme-ov-file#deploying-changes)
```bash
cd ~/houzz/c2/c2thrift
git checkout master
git pull
cd ..
git status
git add .
git commit -m "Updated the c2thrift pointer to correspond with c2thrift changes"
git push origin my-awesome-branch
```
# Send Thrift request from local
要測試 c2thrift 是否有正常運作，可以直接從 local 送 request 來測試
Doc: [**Marking thrift request with js script**](https://docs.google.com/document/d/10-VwcBxMBh1yipKlasmfw10rsjM2ZcQKy6MxR7dvI4E/edit?pli=1&tab=t.0)
步驟：`local jukwaa thriftclient update -> create jukwaa/thrifclient.js -> node thrifclient.js`
```javascript
// jukwaa/thrifclient.js

const { performance } = require('perf_hooks');
const thrift = require('thrift');
const SystemService = require('./thriftClient/SystemService');
const system_ttypes = require('./thriftClient/system_types');

const thriftHost = 'c2-thrift-main.stghouzz.stg-main-eks.stghouzz.com'; // 'houzzdev.com';
const thriftPort = 8094; // 80;

const connection = thrift.createHttpConnection(thriftHost, thriftPort, {
    path: '/SVCEntry.php',
    transport: thrift.TBufferedTransport,
    protocol: thrift.TBinaryProtocol,
});

const multiplexer = new thrift.Multiplexer();

const systemClient = multiplexer.createClient('system', SystemService, connection);

const headerPromoRequest = new system_ttypes.GetHeaderPromoConfigsRequest({
    context: {
        session: {
            visitorId: 'c1a58a80-46a8-4ce5-b727-a6bd90ec322c',
            userId: 75390723,
            siteId: 101,
            locale: 'en-US',
            clientIp: '192.168.0.238,127.0.0.1,100.107.22.202',
            geoData: '/US/CA/807/PALO ALTO/94301',
            webClientInfo: {
                userAgent:
                    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36',
            },
            abTestSeed: {
                userIdHash: '75390723',
                ipHash: '1684739786',
                visitorIdHash: '793176',
                idfaHash: '0',
                urlHash: '484127',
                objectIdHash: '0',
                objectId: '0',
            },
            configOverrides: {
                mp_dweb_browse_page_product_ads_moving_forward: 'treatment',
            },
            isNewVisitor: false,
            isSecure: true,
            geoInfo: {
                country: 'US',
                region: 'CA',
                dma: '807',
                postalCode: '94301',
                city: 'PALO ALTO',
            },
            sensitiveActionFlags: 1183239,
            featureFlags: 8,
            domainId: 1,
            creationTs: 1642825884,
            cookieVersion: 11,
            isHouzzProSubs: false,
        },
        request: {
            requestId: '0895b5ee-006c-4a90-808f-28c3ac8a7f8b',
            debugLevel: 1,
            url: 'https://www.stghouzz.com/products/sofas-and-sectionals',
            traceData: {
                parentId: '0039544b5169c064',
                stackTrace: 'Jukwaa[Web] > IdentityService',
                traceId: 'cbc4b3c1c56d6320',
            },
            includeExtraData: 1,
            upstreams: ['products'],
            isCacheable: false,
            systemInitiated: false,
        },
    },
});

const t2 = performance.now();
systemClient.getHeaderPromoConfigs(headerPromoRequest).then(function(response, err) {
    const t3 = performance.now();
    console.log(err);
    console.log(response);
    console.log('getHeaderPromoConfigs finished in ' + (t3 - t2) + 'ms.');
});
```
# Some useful utilities.
## Use the dynamic config
```javascript
JukwaaConfigerUtils::getConfigValue("header-menu", "saleTabBannerMappingMweb");
```
## Check the page is mweb
```javascript
C2Session::getInstance()->isMWebSession()
```
# Web module
<mention-page url="https://app.notion.com/p/15e6ed462a26801db230c87cb6d4a585"/>
# Reference docs
- Backend Dev Tips: [https://docs.google.com/document/d/1D3D5gMLQ85WIfCVvYDPzXk6x6zqul4JRlOy8Jz2VKEM/edit?usp=sharing](https://docs.google.com/document/d/1D3D5gMLQ85WIfCVvYDPzXk6x6zqul4JRlOy8Jz2VKEM/edit?usp=sharing)
- c2 support and debugging in k8s cluster: [https://docs.google.com/document/d/1w8_g495LRqPaJiJ__IOmlfo7yGiiiMUSMpW4tN_hR7M/edit?usp=sharing](https://docs.google.com/document/d/1w8_g495LRqPaJiJ__IOmlfo7yGiiiMUSMpW4tN_hR7M/edit?usp=sharing)
	- 裡面有提到如何deploy ccp
- K8s access using Teleport: [https://engwiki.houzz.tools/doc/k8s-access-using-teleport-r4JAMoor01](https://engwiki.houzz.tools/doc/k8s-access-using-teleport-r4JAMoor01)
	- vscode 要能使用k8s，需要先登入teleport
# Check prod database
我們可以用 Redash 來取得 data 中的資料，以 sale landing 為例
[https://redash.data.houzz.net/queries/19001/source](https://redash.data.houzz.net/queries/19001/source)
# Trouble shooting
## VSCode 的 K8s extension 雖然顯示了 `stg-main-eks` cluster 但無法載入 namespaces
### The Problem
The VS Code Kubernetes extension was running in a different environment than your terminal, and it couldn't find the `aws` executable because:
### Solution
1. Set zsh as the default shell (Terminal → default profile)
2. Add `AWS_PROFILE` to Shell Profile
	1. Add `export AWS_PROFILE=Stg-Developers-477121734600` to `~/.zshrc` so it's automatically set in new terminal sessions.
## 沒辦法 Attach to vscode
如果是 M1 apple 晶片，vscode 會需要 `dev containers` 套件，而且要把 zsh 設為 default shell (Terminal → default profile)
## Cursor 在 attach 時會跳出 error
error: Error running `kubectl version`
### The problem
Cursor IDE 裡的 K8s extension 找不到 `kubectl` 執行檔。雖然你在 Zsh terminal 裡能執行 `kubectl version`，但這不代表 GUI 應用程式（如 Cursor）可以找到它，原因通常是：**GUI App 的環境變數（如 PATH）跟你在 terminal 裡的不一樣**。
### Solution
執行 `which kubectl`，可能會得到 `/opt/homebrew/bin/kubectl`
在設定搜尋 kubectl，將 dev.containers.kubectlPath 設為 `/opt/homebrew/bin/kubectl`，然後重開 cursor

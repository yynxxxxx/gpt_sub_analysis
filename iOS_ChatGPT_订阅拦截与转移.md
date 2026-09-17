# iOS ChatGPT 订阅拦截与转移原理

> **⚠️ 免责声明**：本教程仅供学习研究用途，请遵守相关法律法规和服务条款。

---

## 目录

1. [核心原理概述](#1-核心原理概述)
2. [拦截：屏蔽回调请求](#2-拦截屏蔽回调请求)
3. [转移：修改 User ID 实现跨账号激活](#3-转移修改-user-id-实现跨账号激活)
4. [关键请求详解](#4-关键请求详解)
5. [响应数据结构](#5-响应数据结构)

---

## 1. 核心原理概述

整个流程涉及两个关键操作：**拦截**和**转移**。理解这两个概念是操作成功的前提。

### 1.1 订阅支付流程

当用户在 iOS 上订阅 ChatGPT 会员时，完整的支付链路如下：

```
┌─────────────────────────────────────────────────────────────────┐
│                        订阅支付流程                               │
│                                                                 │
│  iPhone ──▶ App Store 购买 ──▶ 苹果返回支付凭证（fetch_token）     │
│                                       │                         │
│                                       ▼                         │
│              ChatGPT App 发送凭证到 RevenueCat ──▶ 激活订阅       │
│              （携带 app_user_id 绑定到具体账号）                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 关键角色

| 角色 | 说明 |
|------|------|
| **App Store** | 处理真实付款，返回苹果签发的支付凭证 |
| **RevenueCat** | ChatGPT 使用的第三方订阅管理平台，负责验证凭证并激活订阅 |
| **fetch_token** | 苹果支付凭证（Apple Signed Transaction），证明付款已完成 |
| **app_user_id** | ChatGPT 账号的 Account ID，决定订阅绑定到哪个账号 |

---

## 2. 拦截：屏蔽回调请求

### 2.1 为什么要拦截

苹果支付完成后，ChatGPT App 会**自动**将支付凭证发送到 RevenueCat，将订阅绑定到当前登录的 ChatGPT 账号。如果我们想要将订阅转移到其他账号，就必须**阻止这个自动回调过程**。

### 2.2 需要屏蔽的 URL

在 Reqable 中设置**网关屏蔽**（拦截 / 阻断），屏蔽以下两个 URL：

| 序号 | 屏蔽的 URL | 屏蔽原因 |
|:---:|------------|----------|
| 1 | `https://api.revenuecat.com/v1/receipts` | 阻止苹果支付凭证自动回调到 RevenueCat 服务器 |
| 2 | `https://ios.chat.openai.com/backend-api/payments/rc/ios/verify/v4-2023-04-27` | 阻止 ChatGPT 后端自动校验订阅状态 |

> **⚠️ 重要**：屏蔽这两个 URL 是为了**防止充值回调自动完成**。如果不屏蔽，支付凭证会自动绑定到当前登录的 ChatGPT 账号，后续就无法进行转移操作。

### 2.3 拦截时序图

```
正常流程（不拦截）：
iPhone 付款 ──▶ 凭证自动发送 ──▶ RevenueCat ──▶ 绑定到当前账号 ✅（无法转移）

拦截流程：
iPhone 付款 ──▶ 凭证发送 ──✖ Reqable 屏蔽 ──▶ 凭证未到达 RevenueCat
                                │
                                ▼
                    手动抓取凭证，修改后重发 ──▶ 转移到目标账号 ✅
```

---

## 3. 转移：修改 User ID 实现跨账号激活

### 3.1 转移原理

当苹果支付完成后，ChatGPT App 会向 RevenueCat 发送一个凭证回调请求：

```
POST https://api.revenuecat.com/v1/receipts
```

这个请求体中有两个核心字段：

| 字段 | 含义 | 作用 |
|------|------|------|
| `fetch_token` | 苹果支付凭证（Apple Receipt） | 证明用户已通过 App Store 完成了真实付款，是订阅生效的「钥匙」 |
| `app_user_id` | ChatGPT 账号的 Account ID | 决定这笔订阅**绑定到哪个 ChatGPT 账号** |

> **💡 关键洞察**：`fetch_token` 与 Apple ID 绑定（证明谁付了款），但与 ChatGPT 账号无关。RevenueCat **只根据 `app_user_id`** 来决定将订阅权益分配给哪个用户。

### 3.2 转移步骤

1. 通过 Reqable 抓包获取到完整的凭证回调请求（包含有效的 `fetch_token`）
2. 将请求体中的 `app_user_id` 替换为**目标账号**的 Account ID
3. 重新发送该请求到 `https://api.revenuecat.com/v1/receipts`
4. RevenueCat 收到合法凭证后，将订阅激活到目标账号上

### 3.3 转移流程图

```
原始请求：fetch_token（支付凭证）+ app_user_id = "账号A的ID"
                                        │
                                  修改 app_user_id
                                        │
                                        ▼
转移请求：fetch_token（支付凭证）+ app_user_id = "账号B的ID"
                                        │
                                        ▼
                            账号B 获得 Pro 20x 订阅 ✅
```

### 3.4 如何获取目标账号的 app_user_id

目标 ChatGPT 账号的 `app_user_id` 可以通过以下方式获取：

1. 在目标账号登录 ChatGPT App 的状态下
2. 用 Reqable 抓包任意一个发往 RevenueCat 的请求
3. 在请求体或 URL 中找到 `app_user_id` 字段

---

## 4. 关键请求详解

### 4.1 苹果支付凭证回调请求

```
POST https://api.revenuecat.com/v1/receipts
```

**关键 Header：**

| Header | 值 | 说明 |
|--------|----|------|
| `authorization` | `Bearer appl_rQLChslWRSKCUPBLPJtHCGjvujc` | RevenueCat API 密钥 |
| `x-client-bundle-id` | `com.openai.chat` | ChatGPT 的 Bundle ID |
| `x-platform` | `iOS` | 平台标识 |
| `x-storekit2-enabled` | `true` | 使用 StoreKit 2 |

**关键 Body 字段：**

| 字段 | 值 | 说明 |
|------|----|------|
| `fetch_token` | `eyJhbGci...`（JWS 格式） | 苹果签发的支付凭证，**转移的核心** |
| `app_user_id` | `46ee5a3b-97d3-4da0-baa8-061ecf9f1b25` | ChatGPT 账号 ID，**转移时需替换此字段** |
| `product_id` | `oai_chatgpt_go_1000_1m` | 订阅产品 ID |
| `price` | `8` | 价格（美元） |
| `store_country` | `USA` | 商店区域 |
| `transaction_id` | `440003340653016` | App Store 交易 ID |

### 4.2 转移时需要修改的字段

| 字段 | 操作 | 说明 |
|------|------|------|
| `app_user_id` | ✏️ **必须修改** | 替换为目标账号的 Account ID |
| `fetch_token` | ❌ 保持不变 | 这是苹果签发的合法支付凭证 |
| `app_transaction` | ❌ 保持不变 | 苹果的应用交易凭证 |
| 其他字段 | ❌ 保持不变 | 包括 Header 中的各项参数 |

---

## 5. 响应数据结构

### 5.1 成功响应示例

成功发送凭证回调后，RevenueCat 返回的订阅信息示例：

```json
{
  "request_date": "2026-09-17T07:47:20Z",
  "subscriber": {
    "entitlements": {
      "chatgpt_go": {
        "product_identifier": "oai_chatgpt_go_1000_1m",
        "purchase_date": "2026-09-17T07:38:26Z",
        "expires_date": "2026-10-17T07:38:26Z"
      }
    },
    "original_app_user_id": "46ee5a3b-97d3-4da0-baa8-061ecf9f1b25",
    "subscriptions": {
      "oai_chatgpt_go_1000_1m": {
        "price": { "amount": 8.0, "currency": "USD" },
        "store": "app_store",
        "period_type": "normal",
        "ownership_type": "PURCHASED",
        "original_purchase_date": "2026-09-17T07:38:26Z",
        "expires_date": "2026-10-17T07:38:26Z"
      }
    }
  }
}
```

### 5.2 关键响应字段

| 字段 | 说明 |
|------|------|
| `entitlements` | 当前账号拥有的订阅权益 |
| `original_app_user_id` | 订阅绑定的账号 ID（转移后应为目标账号） |
| `expires_date` | 订阅过期时间 |
| `ownership_type` | `PURCHASED` 表示已购买 |

---

> **最后更新**：2026-09-17

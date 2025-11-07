# 後端系統設計規格書 (Microservices Architecture)

## 1. 總覽 (Overview)
本文件旨在定義競標網站後端系統的微服務架構。此架構將取代原有的單體式設計，將系統拆分為多個獨立、可自主部署的服務，以提升系統的擴展性、靈活性與容錯能力。

## 2. 技術架構 (Technical Architecture)
- **架構模式**: 微服務 (Microservices)
- **服務間通訊**:
    - **同步**: RESTful API (HTTP) / gRPC
    - **非同步**: Message Queue (e.g., RabbitMQ, Kafka)
- **API 閘道器 (API Gateway)**: [YARP](https://microsoft.github.io/reverse-proxy/) 或 [Ocelot](https://github.com/ThreeMammals/Ocelot)
- **開發框架**: ASP.NET Core 8 (C#)
- **資料庫**: PostgreSQL (採用 **Database-per-Service** 模式)
- **ORM**: Entity Framework Core (Code-First)
- **容器化**: Docker
- **CI/CD**: GitHub Actions
- **部署平台**: Kubernetes (e.g., Azure Kubernetes Service)
- **日誌與監控**: 集中式日誌系統 (e.g., ELK Stack, Seq)
- **版本管理**: Git, 採 Git flow 分支策略

## 3. 系統架構圖 (System Architecture Diagram)
```
[Clients: Web/Mobile App]
         |
         v
[API 閘道器 (API Gateway)]
(Routing, Authentication, Aggregation)
         |
         +----------------+----------------+----------------+----------------+
         |                |                |                |                |
         v                v                v                v                v
[1. 會員服務]      [2. 商品拍賣服務]    [3. 競標服務]      [4. 通知服務]      [共用類別庫]
(Member Service)   (Auction Service)  (Bidding Service)  (Notification Service) (Shared Library)
         |                |                |                |
         v                v                v                v
[會員資料庫]       [商品資料庫]       [競標資料庫]       [通知資料庫]
```

## 4. 核心服務與 API 設計 (Core Services & API Design)

### 4.1 API 閘道器 (API Gateway)
- **職責**:
    - **單一入口**: 作為所有客戶端請求的統一入口。
    - **路由**: 將請求轉發到對應的後端微服務。
    - **認證**: 驗證 JWT，無效請求直接攔截。
    - **請求聚合**: (可選) 組合來自多個服務的數據。
    - **SSL 終止**: 集中處理 HTTPS。

### 4.2 會員服務 (Member Service)
- **職責**: 管理所有與使用者身份和資料相關的功能。
- **API Endpoints**:
    - `POST /api/members/register`: 註冊新使用者。
    - `POST /api/members/login`: 登入並取得 JWT。
    - `POST /api/members/refresh-token`: 使用 Refresh Token 換取新的 JWT。
    - `GET /api/members/{id}`: 取得指定使用者的公開資料。
    - `GET /api/members/me`: 取得當前登入使用者的個人資料。
    - `PUT /api/members/me`: 更新當前登入使用者的個人資料。
- **資料庫 (Member DB)**:
    - **Users**: `Id`, `Email`, `PasswordHash`, `Username`, etc.
    - **RefreshTokens**: `Token`, `UserId`, `ExpiryDate`.

### 4.3 商品拍賣服務 (Auction Service)
- **職責**: 管理商品、拍賣狀態與使用者追蹤。
- **API Endpoints**:
    - `GET /api/auctions`: 取得商品清單 (支援篩選與分頁)。
    - `GET /api/auctions/{id}`: 取得單一商品詳細資訊。
    - `POST /api/auctions`: 建立新商品 (需驗證使用者身份)。
    - `PUT /api/auctions/{id}`: 編輯自己的商品。
    - `DELETE /api/auctions/{id}`: 刪除自己的商品。
    - `GET /api/users/{userId}/auctions`: 查詢某個使用者的所有商品。
    - `POST /api/auctions/{id}/follow`: 追蹤商品。
    - `DELETE /api/auctions/{id}/follow`: 取消追蹤商品。
    - `GET /api/me/follows`: 查詢我追蹤的商品清單。
- **資料庫 (Auction DB)**:
    - **Auctions**: `Id`, `Name`, `Description`, `StartingPrice`, `StartTime`, `EndTime`, `Status`, `OwnerUserId` (僅儲存 ID)。
    - **Follows**: `UserId`, `AuctionId`.

### 4.4 競標服務 (Bidding Service)
- **職責**: 處理所有出價相關的邏輯。
- **API Endpoints**:
    - `POST /api/bids`: 提交一個新的出價 (Request Body: `{ "auctionId": "...", "amount": 1000 }`)。
    - `GET /api/auctions/{auctionId}/bids`: 查詢特定商品的完整出價歷史。
    - `GET /api/users/{userId}/bids`: 查詢某個使用者的所有出價紀錄。
    - `GET /api/me/bids`: 查詢我自己的出價紀錄。
- **資料庫 (Bidding DB)**:
    - **Bids**: `Id`, `AuctionId`, `BidderUserId`, `Amount`, `Timestamp`.

### 4.5 通知服務 (Notification Service)
- **職責**: 作為一個獨立的背景服務，負責處理所有非同步的通知發送，包括手機推播、Email 與簡訊。
- **運作方式**:
    - **事件驅動**: 本服務不直接對外提供 API，而是監聽 Message Queue 中的特定事件 (如 `NewAuctionListed`, `AuctionEndingSoon`, `AuctionWinnerDetermined`)。
    - **任務分派**: 收到事件後，根據事件類型與使用者偏好設定，決定要透過何種管道 (Push/Email/SMS) 發送通知。
    - **整合第三方服務**: 
        - **手機推播**: 整合 Firebase Cloud Messaging (FCM) 和 Apple Push Notification Service (APNS)。
        - **Email**: 整合 SendGrid 或 AWS SES 等 Email 服務。
        - **簡訊 (SMS)**: 整合 Twilio 等簡訊閘道服務。

### 4.5.1 廣播情境 (Broadcast Scenarios)
以下是通知服務可能處理的廣播與推播情境：
- **全站公告 (Site-wide Announcement)**:
    - **情境**: 系統維護、新功能上線、重大活動或促銷。
    - **觸發**: 管理員手動觸發。
    - **目標**: 所有使用者。
- **熱門商品即將開賣 (Popular Auction Starting Soon)**:
    - **情境**: 高關注度商品在拍賣開始前發送提醒。
    - **觸發**: 系統自動判斷或管理員手動標記。
    - **目標**: 所有使用者或訂閱熱門商品通知的使用者。
- **特定類別新品上架 (New Listings in Specific Categories)**:
    - **情境**: 使用者追蹤的商品類別有新商品上架。
    - **觸發**: 「商品服務」發布 `NewAuctionListed` 事件。
    - **目標**: 追蹤該類別的使用者。
- **拍賣即將結束提醒 (Auction Ending Soon Reminder)**:
    - **情境**: 使用者追蹤或曾出價的商品，在拍賣結束前發送提醒。
    - **觸發**: 「商品服務」的背景排程任務發布 `AuctionEndingSoon` 事件。
    - **目標**: 追蹤該商品或曾出價的使用者。
- **出價被超越提醒 (Outbid Notification)**:
    - **情境**: 使用者的出價被其他使用者超越。
    - **觸發**: 「競標服務」發布 `Outbid` 事件。
    - **目標**: 被超越出價的使用者。
- **得標通知 (Auction Won Notification)**:
    - **情境**: 使用者成功得標某件商品。
    - **觸發**: 「商品服務」發布 `AuctionWon` 事件。
    - **目標**: 得標使用者。
- **商品價格變動通知 (Item Price Change Notification)**:
    - **情境**: 賣家調整了商品的起標價或保留價。
    - **觸發**: 「商品服務」發布 `AuctionPriceUpdated` 事件。
    - **目標**: 追蹤該商品的使用者。
- **個人化商品推薦 (Personalized Item Recommendations)**:
    - **情境**: 根據使用者的行為（瀏覽、出價）推薦可能感興趣的商品。
    - **觸發**: 推薦系統的背景排程任務發布 `PersonalizedRecommendation` 事件。
    - **目標**: 符合推薦條件的個別使用者。
- **支付/運送提醒 (Payment/Shipping Reminders)**:
    - **情境**: 得標者尚未支付款項，或賣家尚未寄出商品。
    - **觸發**: 「訂單服務」或「商品服務」的背景排程任務發布 `PaymentReminder` 或 `ShippingReminder` 事件。
    - **目標**: 相關的得標者或賣家。

- **API Endpoints (供內部或前端使用)**:
    - `GET /api/notifications/history`: 讓使用者可以從前端查詢自己的歷史通知紀錄。
- **資料庫 (Notification DB)**:
    - **NotificationLogs**: `Id`, `UserId`, `Channel` (Push/Email/SMS), `Content`, `Status` (Sent/Failed), `Timestamp`。
    - **NotificationTemplates**: `TemplateName`, `Channel`, `ContentTemplate`。

## 5. 服務間通訊範例
- **查詢商品與賣家資訊**:
    1. 客戶端向 API 閘道器請求 `GET /api/auctions/{id}`。
    2. 閘道器將請求路由到「商品服務」。
    3. 「商品服務」從自己的資料庫取得商品資料，其中包含 `OwnerUserId`。
    4. 「商品服務」發起一個 HTTP 請求到 `GET /api/members/{OwnerUserId}` (服務對服務的內部呼叫)。
    5. 「會員服務」回傳賣家的公開資訊 (如暱稱)。
    6. 「商品服務」組合商品與賣家資訊後，回傳給閘道器，最終回到客戶端。

- **新商品上架通知 (非同步)**:
    1. 「商品服務」成功建立一筆新商品。
    2. 「商品服務」發布一個 `NewAuctionListed` 事件到 Message Queue，內容包含 `AuctionId` 與商品基本資訊。
    3. 「通知服務」監聽到此事件，從「會員服務」取得有訂閱此類通知的使用者列表。
    4. 「通知服務」根據使用者偏好，透過 FCM/APNS、SendGrid 等服務，將新商品通知發送給目標使用者。

## 6. 共用程式碼
- 建立一個共用的類別庫專案 (`Auction.Shared.csproj`)，用於存放跨服務的 DTOs、Enums、和工具類。此專案會被打包成私有的 NuGet 套件供所有服務使用，以確保一致性。

## 7. 測試策略 (Testing Strategy)
- **單元測試**: 每個微服務都必須有自己的單元測試，專注於內部業務邏輯。
- **整合測試**:
    - **服務內整合測試**: 測試單一服務內的 API 到資料庫的完整流程。
    - **跨服務整合測試**: 測試涉及多個服務協作的關鍵業務流程。
- **契約測試 (Contract Testing)**: 使用 Pact 或類似工具，確保服務間的 API 呼叫符合約定，避免一個服務的變更導致另一個服務出錯。
- **端對端測試 (E2E Tests)**: 透過 API 閘道器模擬真實使用者情境，對系統進行黑箱測試。

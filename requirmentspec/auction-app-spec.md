# 拍賣程式規格檔

## 1. 簡介

本拍賣程式提供一個線上拍賣平台，讓使用者可輕鬆瀏覽、競標與管理商品，支援手機與網頁版，並強調良好體驗。

## 2. 功能需求

### 2.1 首頁

- 顯示所有拍賣商品清單，支援分類篩選與關鍵字搜尋。
- 商品卡片顯示名稱、目前出價、結束時間等資訊。
- 點擊商品卡片可進入詳細頁。
- 商品卡片提供 icon 按鈕，能將商品加入/移除追蹤清單。

### 2.2 商品詳細頁

- 顯示商品描述、目前出價、結束時間等詳細資訊。
- 提供出價功能，使用者可輸入金額進行出價。
- 顯示所有出價歷史紀錄（可選）。

### 2.3 追蹤清單頁

- 顯示使用者追蹤的商品清單。
- 支援取消追蹤功能。

### 2.4 我的出價頁

- 顯示使用者曾出價過的商品清單。
- 顯示每個商品的出價狀態（如領先、被超越）。

### 2.5 出價紀錄頁

- 顯示所有出價紀錄（含過往與目前結果）。
- 支援依時間篩選紀錄。

### 2.6 帳號設定頁

- 提供個人資料編輯（名稱、電子郵件）。
- 支援更改密碼與登出。

### 2.7 通知中心

- 在網站右上角應有通知 icon，點擊後可透過 API 讀取歷史通知，例如「您追蹤的商品已上架」、「恭喜您得標」等。
- 未讀通知應有紅點提示。

## 3. 系統架構

### 3.1 NX 專案結構

- 採用 NX 框架管理多專案架構，利於共用與獨立功能模組化。
- 專案分為：
  - 前端 Web（Next.js）
  - 行動版（Next.js）
  - 共用函式庫（API、hooks、i18n、redux、樣式、UI 元件等）
- 共用函式庫（libs/）統一管理跨專案可重用程式碼。

### 3.2 前端

- 使用 Next.js（基於 React）實現 SPA 與多頁面路由。
- 路由規劃：/auctions、/mybids、/account、/follow、/history 等。
- 樣式採 SCSS Toolkit，統一設計與排版。
- 支援 RWD 響應式設計與無障礙（a11y）。
- 國際化（i18n）：支援多語系切換，預設 zh-TW。

### 3.3 後端

- 採用 ASP.NET Core 8（C#）開發 RESTful API，處理商品、使用者、出價等資料。
- API 提供標準 CRUD、驗證（JWT）、權限控管。
- 日誌與錯誤監控（如 Sentry）。

### 3.4 資料庫

- 使用 PostgreSQL 儲存商品、使用者、出價等資料，確保資料一致性與效能。

## 4. 使用者角色

- **一般使用者**：可瀏覽、追蹤、出價商品與管理個人帳號。
- **管理員**：可管理商品資料與使用者帳號，具備後台管理權限。

## 5. 技術規格

### 5.1 前端

- 程式語言：TypeScript/JavaScript
- 框架：Next.js
- 樣式：Ant Design, SCSS
- 狀態管理：Redux Toolkit
- API 通訊：Fetch
- 程式碼品質：ESLint、Prettier
- 測試：Jest（單元）、Playwright（E2E）

### 5.2 後端

- 程式語言：C#
- 框架：ASP.NET Core 8
- 資料庫：PostgreSQL
- 測試：xUnit

### 5.3 其他

- 容器化：Docker
- CI/CD：GitHub Actions，NX affected 指令加速建構
- 部署：前端（Vercel/Netlify）、後端（Azure App Service 或等同平台）
- 版本管理：Git，採 Git flow 分支策略

## 6. API 規格

### 6.1 取得商品清單

- `GET /api/auctions`
  - 參數：`category`（選填）、`search`（選填）
  - 回應：200 OK（商品清單），400 Bad Request

### 6.2 新增追蹤商品

- `POST /api/follow`
  - 參數：`itemId`（必填）
  - 回應：201 Created，400 Bad Request

### 6.3 取得商品詳細

- `GET /api/auctions/{id}`
  - 回應：200 OK（商品詳細），404 Not Found

### 6.4 出價

- `POST /api/bid`
  - 參數：`itemId`、`amount`
  - 回應：201 Created，400 Bad Request，403 Forbidden

### 6.5 取得我的出價紀錄

- `GET /api/mybids`
  - 回應：200 OK（出價紀錄）

### 6.6 取得追蹤清單

- `GET /api/follow`
  - 回應：200 OK（追蹤商品清單）

### 6.7 帳號管理

- `GET /api/account`、`PUT /api/account`、`POST /api/account/password`
  - 回應：200 OK，400 Bad Request

### 6.8 取得通知紀錄

- `GET /api/notifications/history`
  - 回應：200 OK（該使用者的通知歷史紀錄），401 Unauthorized

## 7. 測試計畫

- **單元測試**：涵蓋各函式、元件與模組，Jest（前端）、xUnit（後端）。
- **整合測試**：驗證前後端互動。
- **E2E 測試**：Playwright，模擬實際情境。
- **使用者測試**：確保功能符合需求。
- 覆蓋率目標：單元測試 80%、E2E 測試 70%。

## 8. 部署計畫

- 前端：Vercel 或 Netlify
- 後端：Azure App Service 或等同平台
- 容器化：Docker
- CI/CD：GitHub Actions，含 lint、test、build、deploy

## 9. 安全性與效能

- 防範 XSS/CSRF，API rate limit。
- 前端效能優化：圖片壓縮、Lazy load、Code splitting。
- 利用 NX 快取與分散式建構提升延展性。

## 10. 日誌與監控

- 前後端日誌收集、錯誤回報與監控（如 Sentry、LogRocket）。

## 11. 參考文件

- SCSS Toolkit 官方文件
- React 官方文件
- ASP.NET Core 官方文件
- NX 官方文件
- Playwright 官方文件
- PostgreSQL 官方文件
  redux-toolkit.js.org/)
- [Sass (SCSS)](https://sass-lang.com/documentation/)

### 後端 (Backend)

- [ASP.NET Core](https://dotnet.microsoft.com/apps/aspnet)
- [PostgreSQL](https://www.postgresql.org/docs/)

### 工具與理念 (Tools & Concepts)

- [NX](https://nx.dev/)
- [Playwright](https://playwright.dev/)
- [Atomic Design (Brad Frost)](https://bradfrost.com/blog/post/atomic-web-design/)
- [Web Components (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Web_components)

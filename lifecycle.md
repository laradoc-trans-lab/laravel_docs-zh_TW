# 請求生命週期

- [簡介](#introduction)
- [生命週期概覽](#lifecycle-overview)
    - [初始步驟](#first-steps)
    - [HTTP / Console 核心](#http-console-kernels)
    - [服務提供者](#service-providers)
    - [路由](#routing)
    - [收尾工作](#finishing-up)
- [聚焦服務提供者](#focus-on-service-providers)

<a name="introduction"></a>
## 簡介

在「真實世界」中使用任何工具時，若能理解其運作方式，您會感到更有信心。應用程式開發亦是如此。當您了解開發工具的功能時，使用起來會更自在、更有信心。

本文檔的目標是為您提供 Laravel 框架如何運作的良好、高層次概覽。透過更深入地了解整個框架，一切將不再那麼「神奇」，您將更有信心建構您的應用程式。如果您無法立即理解所有術語，請不要氣餒！只需嘗試對其運作方式有個基本概念，隨著您探索文檔的其他部分，您的知識將會增長。

<a name="lifecycle-overview"></a>
## 生命週期概覽

<a name="first-steps"></a>
### 初始步驟

所有對 Laravel 應用程式的請求，其進入點都是 `public/index.php` 檔案。所有請求都會透過您的網頁伺服器 (Apache / Nginx) 設定導向此檔案。`index.php` 檔案的程式碼不多，它只是載入框架其餘部分的起點。

`index.php` 檔案會載入 Composer 自動生成的自動載入器定義，然後從 `bootstrap/app.php` 取得 Laravel 應用程式的實例。Laravel 本身執行的第一個動作是建立應用程式 / [服務容器](/docs/{{version}}/container) 的實例。

<a name="http-console-kernels"></a>
### HTTP / Console 核心

接下來，根據進入應用程式的請求類型，傳入的請求會透過應用程式實例的 `handleRequest` 或 `handleCommand` 方法，傳送至 HTTP 核心或 Console 核心。這兩個核心是所有請求流經的中心位置。目前，我們將專注於 HTTP 核心，它是 `Illuminate\Foundation\Http\Kernel` 的實例。

HTTP 核心定義了一個 `bootstrappers` 陣列，這些啟動器會在請求執行前運行。這些啟動器會配置錯誤處理、配置日誌記錄、[偵測應用程式環境](/docs/{{version}}/configuration#environment-configuration)，並執行其他在實際處理請求前需要完成的任務。通常，這些類別處理的是您無需擔心的 Laravel 內部配置。

HTTP 核心也負責將請求傳遞通過應用程式的 Middleware 堆疊。這些 Middleware 處理 [HTTP Session](/docs/{{version}}/session) 的讀寫、判斷應用程式是否處於維護模式、[驗證 CSRF Token](/docs/{{version}}/csrf) 等等。我們很快會更詳細地討論這些。

HTTP 核心的 `handle` 方法簽名非常簡單：它接收一個 `Request` 並回傳一個 `Response`。將核心想像成一個代表您整個應用程式的大黑盒子。向它輸入 HTTP 請求，它就會回傳 HTTP 回應。

<a name="service-providers"></a>
### 服務提供者

核心最重要的啟動動作之一是載入應用程式的[服務提供者](/docs/{{version}}/providers)。服務提供者負責啟動框架的各種元件，例如資料庫、Queue、驗證和路由元件。

Laravel 會迭代此提供者列表並實例化每個提供者。實例化提供者後，會呼叫所有提供者的 `register` 方法。然後，一旦所有提供者都已註冊，就會呼叫每個提供者的 `boot` 方法。這是為了確保在執行 `boot` 方法時，所有容器綁定都已註冊並可用。

基本上，Laravel 提供的每個主要功能都由服務提供者啟動和配置。由於它們啟動和配置了框架提供的許多功能，服務提供者是整個 Laravel 啟動過程中最重要的環節。

儘管框架內部使用了數十個服務提供者，您也可以選擇建立自己的服務提供者。您可以在 `bootstrap/providers.php` 檔案中找到應用程式正在使用的使用者定義或第三方服務提供者列表。

<a name="routing"></a>
### 路由

一旦應用程式啟動完成且所有服務提供者都已註冊，`Request` 將會被交給 Router 進行分派。Router 會將請求分派給路由或 Controller，並執行任何路由特定的 Middleware。

Middleware 提供了一種方便的機制，用於過濾或檢查進入您應用程式的 HTTP 請求。例如，Laravel 包含一個 Middleware，用於驗證您應用程式的使用者是否已通過身份驗證。如果使用者未通過身份驗證，該 Middleware 會將使用者重新導向到登入畫面。然而，如果使用者已通過身份驗證，該 Middleware 將允許請求進一步進入應用程式。有些 Middleware 會被指派給應用程式中的所有路由，例如 `PreventRequestsDuringMaintenance`，而有些則只被指派給特定的路由或路由群組。您可以閱讀完整的 [Middleware 文件](/docs/{{version}}/middleware) 以了解更多關於 Middleware 的資訊。

如果請求通過了所有匹配路由所指派的 Middleware，則會執行路由或 Controller 方法，並且路由或 Controller 方法回傳的回應將會再次通過路由的 Middleware 鏈。

<a name="finishing-up"></a>
### 收尾工作

一旦路由或 Controller 方法回傳回應，該回應將會向外回傳通過路由的 Middleware，讓應用程式有機會修改或檢查傳出的回應。

最後，一旦回應回傳通過 Middleware，HTTP 核心的 `handle` 方法會將回應物件回傳給應用程式實例的 `handleRequest`，而此方法會呼叫回傳回應上的 `send` 方法。`send` 方法會將回應內容傳送給使用者的網頁瀏覽器。我們現在已經完成了整個 Laravel 請求生命週期的旅程！

<a name="focus-on-service-providers"></a>
## 聚焦服務提供者

服務提供者確實是啟動 Laravel 應用程式的關鍵。應用程式實例被建立，服務提供者被註冊，然後請求被交給已啟動的應用程式。就這麼簡單！

牢固掌握 Laravel 應用程式如何透過服務提供者建構和啟動是非常有價值的。您應用程式的使用者定義服務提供者儲存在 `app/Providers` 目錄中。

預設情況下，`AppServiceProvider` 相當空。此提供者是新增應用程式自己的啟動和服務容器綁定的絕佳位置。對於大型應用程式，您可能希望建立多個服務提供者，每個提供者都針對應用程式使用的特定服務進行更細粒度的啟動。


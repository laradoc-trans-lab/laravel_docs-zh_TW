# 請求生命週期

- [簡介](#introduction)
- [生命週期概覽](#lifecycle-overview)
    - [初始步驟](#first-steps)
    - [HTTP / Console Kernels](#http-console-kernels)
    - [Service Providers](#service-providers)
    - [路由](#routing)
    - [收尾工作](#finishing-up)
- [聚焦 Service Providers](#focus-on-service-providers)

<a name="introduction"></a>
## 簡介

在「現實世界」中使用任何工具時，如果您了解該工具的運作方式，您會感到更有信心。應用程式開發也不例外。當您了解開發工具的功能時，您會更自在、更有信心地使用它們。

本文檔的目標是為您提供 Laravel 框架如何運作的良好、高層次概覽。透過更好地了解整個框架，一切都會感覺不那麼「神奇」，您將更有信心建構您的應用程式。如果您沒有立即理解所有術語，請不要氣餒！只需嘗試對正在發生的事情有一個基本的掌握，您的知識將隨著您探索說明文件的其他部分而增長。

<a name="lifecycle-overview"></a>
## 生命週期概覽

<a name="first-steps"></a>
### 初始步驟

所有對 Laravel 應用程式的請求，其進入點都是 `public/index.php` 檔案。所有請求都會透過您的網頁伺服器 (Apache / Nginx) 設定導向到此檔案。`index.php` 檔案的程式碼不多，它只是載入框架其餘部分的起點。

`index.php` 檔案會載入 Composer 自動生成的自動載入器定義，然後從 `bootstrap/app.php` 取得 Laravel 應用程式的實例。Laravel 本身採取的首要動作是建立應用程式 / [Service Container](/docs/{{version}}/container) 的實例。

<a name="http-console-kernels"></a>
### HTTP / Console Kernels

接下來，根據進入應用程式的請求類型，傳入的請求會透過應用程式實例的 `handleRequest` 或 `handleCommand` 方法，發送到 HTTP kernel 或 Console kernel。這兩個 kernel 作為所有請求流經的中心位置。目前，我們只關注 HTTP kernel，它是 `Illuminate\Foundation\Http\Kernel` 的實例。

HTTP kernel 定義了一個 `bootstrappers` 陣列，這些 bootstrappers 會在請求執行前運行。這些 bootstrappers 會配置錯誤處理、配置日誌記錄、[偵測應用程式環境](/docs/{{version}}/configuration#environment-configuration)，並執行其他在實際處理請求之前需要完成的任務。通常，這些類別處理的是您無需擔心的內部 Laravel 配置。

HTTP kernel 也負責將請求傳遞到應用程式的 Middleware 堆疊。這些 Middleware 處理 [HTTP session](/docs/{{version}}/session) 的讀寫、判斷應用程式是否處於維護模式、[驗證 CSRF token](/docs/{{version}}/csrf) 等。我們很快會更詳細地討論這些。

HTTP kernel 的 `handle` 方法簽名非常簡單：它接收一個 `Request` 並返回一個 `Response`。將 kernel 視為一個代表您整個應用程式的大黑盒子。向它提供 HTTP 請求，它將返回 HTTP 回應。

<a name="service-providers"></a>
### Service Providers

最重要的 kernel 啟動動作之一是載入應用程式的 [Service Providers](/docs/{{version}}/providers)。Service Providers 負責啟動框架的各種元件，例如資料庫、Queue、驗證和路由元件。

Laravel 將會遍歷這個 providers 列表並實例化它們。在實例化 providers 之後，所有 providers 的 `register` 方法將會被呼叫。然後，一旦所有 providers 都已註冊，每個 provider 的 `boot` 方法將會被呼叫。這是為了讓 Service Providers 可以在其 `boot` 方法執行時，依賴所有已註冊且可用的 Container 綁定。

基本上，Laravel 提供的每個主要功能都是由 Service Provider 啟動和配置的。由於它們啟動和配置了框架提供的許多功能，Service Providers 是整個 Laravel 啟動過程中最重要的環節。

儘管框架內部使用了數十個 Service Providers，您也可以選擇建立自己的 Service Providers。您可以在 `bootstrap/providers.php` 檔案中找到應用程式正在使用的使用者定義或第三方 Service Providers 列表。

<a name="routing"></a>
### 路由

一旦應用程式啟動完成且所有 Service Providers 都已註冊，`Request` 將會被交給 Router 進行分派。Router 會將請求分派到路由或 Controller，並運行任何路由特定的 Middleware。

Middleware 提供了一種方便的機制，用於過濾或檢查進入您應用程式的 HTTP 請求。例如，Laravel 包含一個 Middleware，用於驗證您應用程式的使用者是否已通過身份驗證。如果使用者未通過身份驗證，該 Middleware 會將使用者重新導向到登入畫面。但是，如果使用者已通過身份驗證，該 Middleware 將允許請求進一步進入應用程式。有些 Middleware 會分配給應用程式中的所有路由，例如 `PreventRequestsDuringMaintenance`，而有些則只分配給特定的路由或路由群組。您可以透過閱讀完整的 [Middleware 說明文件](/docs/{{version}}/middleware) 來了解更多關於 Middleware 的資訊。

如果請求通過了所有匹配路由所分配的 Middleware，路由或 Controller 方法將會被執行，並且路由或 Controller 方法返回的回應將會透過路由的 Middleware 鏈傳回。

<a name="finishing-up"></a>
### 收尾工作

一旦路由或 Controller 方法返回回應，該回應將會透過路由的 Middleware 向外傳播，讓應用程式有機會修改或檢查傳出的回應。

最後，一旦回應透過 Middleware 傳回，HTTP kernel 的 `handle` 方法會將回應物件返回給應用程式實例的 `handleRequest`，而此方法會呼叫返回回應上的 `send` 方法。`send` 方法會將回應內容發送到使用者的網頁瀏覽器。我們現在已經完成了整個 Laravel 請求生命週期的旅程！

<a name="focus-on-service-providers"></a>
## 聚焦 Service Providers

Service Providers 確實是啟動 Laravel 應用程式的關鍵。應用程式實例被建立，Service Providers 被註冊，然後請求被交給已啟動的應用程式。就這麼簡單！

牢固掌握 Laravel 應用程式如何透過 Service Providers 建立和啟動是非常有價值的。您應用程式的使用者定義 Service Providers 儲存在 `app/Providers` 目錄中。

預設情況下，`AppServiceProvider` 相當空。這個 Provider 是添加您應用程式自己的啟動和 Service Container 綁定的絕佳位置。對於大型應用程式，您可能希望建立多個 Service Providers，每個都為您應用程式使用的特定服務提供更細粒度的啟動。


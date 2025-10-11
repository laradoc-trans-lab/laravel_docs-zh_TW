# 請求生命週期 (Request Lifecycle)

- [簡介](#introduction)
- [生命週期概覽](#lifecycle-overview)
    - [第一步](#first-steps)
    - [HTTP / Console 核心](#http-console-kernels)
    - [服務提供者](#service-providers)
    - [路由](#routing)
    - [完成](#finishing-up)
- [聚焦於服務提供者](#focus-on-service-providers)

<a name="introduction"></a>
## 簡介

在「真實世界」中使用任何工具時，如果你了解該工具的運作方式，你會感到更加自信。應用程式開發也不例外。當你了解你的開發工具如何運作時，你會在使用它們時感到更自在、更自信。

本文件的目標是讓你對 Laravel 框架的運作方式有一個良好、高層次的概覽。透過更好地了解整個框架，一切都會感覺不再那麼「神奇」，你也會更有信心建構你的應用程式。如果你一開始不了解所有術語，請不要氣餒！只需嘗試對正在發生的事情有一個基本的理解，你的知識將會隨著你探索文件的其他部分而增長。

<a name="lifecycle-overview"></a>
## 生命週期概覽

<a name="first-steps"></a>
### 第一步

所有傳送至 Laravel 應用程式的請求都以 `public/index.php` 檔案作為入口點。所有請求都會透過你的網頁伺服器 (Apache / Nginx) 配置導向到這個檔案。`index.php` 檔案本身不包含太多程式碼，它只是載入框架其餘部分的起點。

`index.php` 檔案會載入 Composer 生成的自動載入器定義，然後從 `bootstrap/app.php` 取得 Laravel 應用程式的一個實例。Laravel 本身採取的首要行動是建立應用程式 / [服務容器](/docs/{{version}}/container) 的實例。

<a name="http-console-kernels"></a>
### HTTP / Console 核心

接下來，傳入的請求會被發送到 HTTP 核心或 Console 核心，使用應用程式實例的 `handleRequest` 或 `handleCommand` 方法，具體取決於進入應用程式的請求類型。這兩個核心是所有請求流經的中心位置。目前，我們只專注於 HTTP 核心，它是 `Illuminate\Foundation\Http\Kernel` 的一個實例。

HTTP 核心定義了一個 `bootstrappers` 陣列，這些啟動程式會在請求執行前運行。這些啟動程式會配置錯誤處理、配置日誌、[偵測應用程式環境](/docs/{{version}}/configuration#environment-configuration)，並執行其他需要在請求實際處理前完成的任務。通常，這些類別處理的是你無需擔心的內部 Laravel 配置。

HTTP 核心也負責讓請求通過應用程式的「中介層堆疊 (middleware stack)」。這些中介層處理讀取和寫入 [HTTP session](/docs/{{version}}/session)，判斷應用程式是否處於維護模式，[驗證 CSRF 憑證](/docs/{{version}}/csrf)，等等。我們很快會更詳細地討論這些。

HTTP 核心的 `handle` 方法簽章非常簡單：它接收一個 `Request` 並回傳一個 `Response`。把核心想像成一個代表你整個應用程式的大黑盒子。傳送 HTTP 請求給它，它就會回傳 HTTP 回應。

<a name="service-providers"></a>
### 服務提供者

最重要的核心啟動動作之一是載入應用程式的 [服務提供者](/docs/{{version}}/providers)。服務提供者負責啟動框架的所有各種組件，例如資料庫、佇列、驗證和路由組件。

Laravel 將會遍歷這個提供者列表並實例化每個提供者。實例化提供者後，會對所有提供者呼叫 `register` 方法。然後，一旦所有提供者都已註冊，就會對每個提供者呼叫 `boot` 方法。這樣做是為了讓服務提供者可以依賴在 `boot` 方法執行時，所有容器綁定都已註冊並可用。

基本上，Laravel 提供的每個主要功能都是由服務提供者啟動和配置的。由於它們啟動和配置了框架提供的許多功能，服務提供者是整個 Laravel 啟動過程中最重要的環節。

雖然框架內部使用了數十個服務提供者，你也可以選擇建立自己的服務提供者。你可以在 `bootstrap/providers.php` 檔案中找到應用程式正在使用的使用者定義或第三方服務提供者列表。

<a name="routing"></a>
### 路由

一旦應用程式完成啟動且所有服務提供者都已註冊，`Request` 將會被傳遞給路由器進行分派。路由器會將請求分派給路由或控制器，並執行任何路由特定的中介層。

中介層 (Middleware) 提供了一種方便的機制，用於過濾或檢查進入應用程式的 HTTP 請求。例如，Laravel 包含一個中介層，用於驗證應用程式的使用者是否已通過身份驗證。如果使用者未經身份驗證，中介層會將使用者重定向到登入畫面。然而，如果使用者已通過身份驗證，中介層將允許請求進一步進入應用程式。有些中介層被分配給應用程式中的所有路由，例如 `PreventRequestsDuringMaintenance`，而有些則僅分配給特定路由或路由群組。你可以透過閱讀完整的 [中介層文件](/docs/{{version}}/middleware) 來了解更多關於中介層的資訊。

如果請求通過所有匹配路由所分配的中介層，路由或控制器方法將會被執行，並且由路由或控制器方法回傳的回應將會透過路由的中介層鏈傳回。

<a name="finishing-up"></a>
### 完成

一旦路由或控制器方法回傳一個回應，該回應將會向外傳播，透過路由的中介層，讓應用程式有機會修改或檢查外發的回應。

最後，一旦回應回傳通過中介層，HTTP 核心的 `handle` 方法會將回應物件回傳給應用程式實例的 `handleRequest`，並且這個方法會呼叫回傳回應上的 `send` 方法。`send` 方法會將回應內容傳送給使用者的網頁瀏覽器。我們現在已完成了整個 Laravel 請求生命週期的旅程！

<a name="focus-on-service-providers"></a>
## 聚焦於服務提供者

服務提供者確實是啟動 Laravel 應用程式的關鍵。應用程式實例被建立，服務提供者被註冊，然後請求被傳遞給已啟動的應用程式。就這麼簡單！

對於 Laravel 應用程式如何透過服務提供者來建構和啟動有一個牢固的理解是非常有價值的。你的應用程式使用者定義的服務提供者儲存在 `app/Providers` 目錄中。

預設情況下，`AppServiceProvider` 相當空。這個提供者是一個很好的地方，可以添加應用程式自己的啟動程式和服務容器綁定。對於大型應用程式，你可能希望建立多個服務提供者，每個提供者都為應用程式使用的特定服務提供更精細的啟動。
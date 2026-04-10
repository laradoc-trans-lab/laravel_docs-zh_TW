# 請求生命週期

- [簡介](#introduction)
- [生命週期概觀](#lifecycle-overview)
    - [首部曲](#first-steps)
    - [HTTP / Console 核心](#http-console-kernels)
    - [服務提供者](#service-providers)
    - [路由](#routing)
    - [收尾](#finishing-up)
- [聚焦於服務提供者](#focus-on-service-providers)

<a name="introduction"></a>
## 簡介

在「真實世界」中使用任何工具時，如果您了解該工具的運作方式，就會感到更有信心。應用程式開發也不例外。當您了解開發工具的功能時，會更自在和自信地使用它們。

本文件的目標是讓您對 Laravel 框架的運作方式有一個良好且高層次的概觀。透過更了解整體框架，一切都會感覺不那麼「神奇」，並且您將更有信心建構您的應用程式。如果您沒有立即理解所有術語，請不要氣餒！只要試著對正在發生的事情有一個基本的了解，您的知識就會隨著您探索文件的其他部分而增長。


<a name="lifecycle-overview"></a>
## 生命週期概觀


<a name="first-steps"></a>
### 首部曲

所有對 Laravel 應用程式的請求都以 `public/index.php` 檔案作為進入點。所有請求都會透過您的網路伺服器 (Apache / Nginx) 設定導向此檔案。`index.php` 檔案沒有包含太多程式碼。相反地，它是載入其餘框架的起點。

`index.php` 檔案載入由 Composer 生成的自動載入器定義，然後從 `bootstrap/app.php` 取得 Laravel 應用程式的實例。Laravel 本身採取的首要行動是建立應用程式 / [服務容器](/docs/{{version}}/container) 的實例。


<a name="http-console-kernels"></a>
### HTTP / Console 核心

接下來，傳入的請求會根據進入應用程式的請求類型，使用應用程式實例的 `handleRequest` 或 `handleCommand` 方法，發送到 HTTP 核心或 Console 核心。這兩個核心是所有請求流經的中心位置。目前，我們只專注於 HTTP 核心，它是 `Illuminate\Foundation\Http\Kernel` 的實例。

HTTP 核心定義了一個 `bootstrappers` 陣列，這些啟動程序會在請求執行之前運行。這些啟動程序會設定錯誤處理、設定日誌記錄、[偵測應用程式環境](/docs/{{version}}/configuration#environment-configuration)，並執行其他在請求實際處理之前需要完成的任務。通常，這些類別處理您不需要擔心的內部 Laravel 設定。

HTTP 核心也負責將請求通過應用程式的「中介層」堆疊。這些中介層處理讀寫 [HTTP session](/docs/{{version}}/session)、判斷應用程式是否處於維護模式、[驗證 CSRF 權杖](/docs/{{version}}/csrf)等等。我們很快會更詳細地討論這些。

HTTP 核心的 `handle` 方法簽章相當簡單：它接收一個 `Request` 並回傳一個 `Response`。將核心想像成一個代表您整個應用程式的大黑盒子。傳入 HTTP 請求給它，它就會回傳 HTTP 回應。


<a name="service-providers"></a>
### 服務提供者

最重要的核心啟動動作之一是載入應用程式的 [服務提供者](/docs/{{version}}/providers)。服務提供者負責啟動框架的各種組件，例如資料庫、佇列、驗證和路由組件。

Laravel 將會迭代這個提供者列表並實例化它們。在實例化提供者之後，會呼叫所有提供者的 `register` 方法。然後，一旦所有提供者都已註冊，就會在每個提供者上呼叫 `boot` 方法。這是為了讓服務提供者能夠在它們的 `boot` 方法執行時，依賴所有容器綁定都已註冊並可用。

本質上，Laravel 提供的每個主要功能都是由服務提供者啟動和配置的。由於它們啟動和配置了框架提供的如此多功能，因此服務提供者是整個 Laravel 啟動過程中最重要的方面。

雖然框架內部使用了數十個服務提供者，您也可以選擇建立自己的。您可以在 `bootstrap/providers.php` 檔案中找到應用程式正在使用的使用者定義或第三方服務提供者列表。


<a name="routing"></a>
### 路由

一旦應用程式已啟動且所有服務提供者都已註冊，`Request` 將會交由路由器進行分派。路由器會將請求分派給路由或控制器，並執行任何路由特定的中介層。

中介層提供了一個便捷的機制，用於過濾或檢查進入應用程式的 HTTP 請求。例如，Laravel 包含一個中介層，用於驗證應用程式使用者是否已驗證。如果使用者未驗證，中介層會將使用者重導向登入畫面。然而，如果使用者已驗證，中介層會允許請求進一步進入應用程式。有些中介層被分配給應用程式中的所有路由，例如 `PreventRequestsDuringMaintenance`，而有些則只分配給特定的路由或路由群組。您可以透過閱讀完整的 [中介層文件](/docs/{{version}}/middleware) 來了解更多關於中介層的資訊。

如果請求通過所有匹配路由所分配的中介層，路由或控制器方法將會執行，並且由路由或控制器方法回傳的回應將會透過路由的中介層鏈傳回。


<a name="finishing-up"></a>
### 收尾

一旦路由或控制器方法回傳一個回應，該回應將會透過路由的中介層向外傳回，讓應用程式有機會修改或檢查外送的回應。

最後，一旦回應透過中介層傳回，HTTP 核心的 `handle` 方法會將回應物件回傳給應用程式實例的 `handleRequest`，而此方法會呼叫回傳回應上的 `send` 方法。`send` 方法會將回應內容發送到使用者的網路瀏覽器。我們現在已經完成了整個 Laravel 請求生命週期的旅程！


<a name="focus-on-service-providers"></a>
## 聚焦於服務提供者

服務提供者確實是啟動 Laravel 應用程式的關鍵。應用程式實例被建立，服務提供者被註冊，然後請求被交給已啟動的應用程式。就這麼簡單！

對於 Laravel 應用程式如何透過服務提供者來建構和啟動，有一個穩固的掌握是非常有價值的。您的應用程式中使用者定義的服務提供者儲存在 `app/Providers` 目錄中。

預設情況下，`AppServiceProvider` 相當空。這個提供者是加入您應用程式自己的啟動和服務容器綁定的絕佳位置。對於大型應用程式，您可能希望建立多個服務提供者，每個提供者都針對應用程式使用的特定服務提供更細緻的啟動。
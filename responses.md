# HTTP 回應

- [建立回應](#creating-responses)
    - [附加標頭至回應](#attaching-headers-to-responses)
    - [附加 Cookie 至回應](#attaching-cookies-to-responses)
    - [Cookie 與加密](#cookies-and-encryption)
- [重新導向](#redirects)
    - [重新導向至具名路由](#redirecting-named-routes)
    - [重新導向至控制器動作](#redirecting-controller-actions)
    - [重新導向至外部網域](#redirecting-external-domains)
    - [重新導向並快閃 Session 資料](#redirecting-with-flashed-session-data)
- [其他回應類型](#other-response-types)
    - [視圖回應](#view-responses)
    - [JSON 回應](#json-responses)
    - [檔案下載](#file-downloads)
    - [檔案回應](#file-responses)
    - [串流回應](#streamed-responses)
- [回應巨集](#response-macros)

<a name="creating-responses"></a>
## 建立回應


<a name="strings-arrays"></a>
#### 字串與陣列

所有路由與控制器都應該回傳一個回應，以發送回使用者的瀏覽器。Laravel 提供了幾種不同的方式來回傳回應。最基本的回應是從路由或控制器回傳字串。框架會自動將字串轉換為一個完整的 HTTP 回應：

    Route::get('/', function () {
        return 'Hello World';
    });

除了從路由和控制器回傳字串外，你也可以回傳陣列。框架會自動將陣列轉換為 JSON 回應：

    Route::get('/', function () {
        return [1, 2, 3];
    });

> [!NOTE]  
> 你知道你也可以從路由或控制器回傳 [Eloquent 集合](/docs/{{version}}/eloquent-collections) 嗎？它們會自動被轉換為 JSON。試試看吧！


<a name="response-objects"></a>
#### 回應物件

通常，你不會只從路由動作回傳簡單的字串或陣列。相反地，你會回傳完整的 `Illuminate\Http\Response` 實例或 [視圖](/docs/{{version}}/views)。

回傳完整的 `Response` 實例可讓你自訂回應的 HTTP 狀態碼與標頭。一個 `Response` 實例繼承自 `Symfony\Component\HttpFoundation\Response` 類別，它提供了多種方法來建構 HTTP 回應：

    Route::get('/home', function () {
        return response('Hello World', 200)
            ->header('Content-Type', 'text/plain');
    });


<a name="eloquent-models-and-collections"></a>
#### Eloquent 模型與集合

你也可以直接從路由和控制器回傳 [Eloquent ORM](/docs/{{version}}/eloquent) 模型和集合。當你這樣做時，Laravel 會自動將模型和集合轉換為 JSON 回應，同時尊重模型的 [隱藏屬性](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)：

    use App\Models\User;

    Route::get('/user/{user}', function (User $user) {
        return $user;
    });


<a name="attaching-headers-to-responses"></a>
### 附加標頭至回應

請記住，大多數的回應方法都是可鏈接的 (chainable)，這允許流暢地建構回應實例。例如，你可以使用 `header` 方法在將回應傳回使用者之前，為回應新增一系列的標頭：

    return response($content)
        ->header('Content-Type', $type)
        ->header('X-Header-One', 'Header Value')
        ->header('X-Header-Two', 'Header Value');

或者，你可以使用 `withHeaders` 方法來指定要新增至回應的標頭陣列：

    return response($content)
        ->withHeaders([
            'Content-Type' => $type,
            'X-Header-One' => 'Header Value',
            'X-Header-Two' => 'Header Value',
        ]);


<a name="cache-control-middleware"></a>
#### 快取控制中介層

Laravel 包含了 `cache.headers` 中介層，它可以用於快速為一組路由設定 `Cache-Control` 標頭。指令應使用對應的快取控制指令的「snake case」等效形式提供，並以分號分隔。如果 `etag` 在指令清單中指定，回應內容的 MD5 雜湊值將會自動設定為 ETag 識別碼：

    Route::middleware('cache.headers:public;max_age=2628000;etag')->group(function () {
        Route::get('/privacy', function () {
            // ...
        });

        Route::get('/terms', function () {
            // ...
        });
    });


<a name="attaching-cookies-to-responses"></a>
### 附加 Cookie 至回應

你可以使用 `cookie` 方法將一個 cookie 附加到一個外發的 `Illuminate\Http\Response` 實例上。你應該將名稱、值以及 cookie 應被視為有效的分鐘數傳遞給此方法：

    return response('Hello World')->cookie(
        'name', 'value', $minutes
    );

`cookie` 方法也接受一些較少使用的額外引數。通常，這些引數與 PHP 原生 [setcookie](https://secure.php.net/manual/en/function.setcookie.php) 方法所提供的引數具有相同的目的和意義：

    return response('Hello World')->cookie(
        'name', 'value', $minutes, $path, $domain, $secure, $httpOnly
    );

如果你希望確保一個 cookie 隨外發回應一同發送，但你尚未擁有該回應的實例，你可以使用 `Cookie` facade 來「排隊 (queue)」cookie，以便在回應發送時附加到回應上。`queue` 方法接受建立 cookie 實例所需的引數。這些 cookie 將在回應發送到瀏覽器之前被附加到外發回應上：

    use Illuminate\Support\Facades\Cookie;

    Cookie::queue('name', 'value', $minutes);


<a name="generating-cookie-instances"></a>
#### 產生 Cookie 實例

如果你想產生一個 `Symfony\Component\HttpFoundation\Cookie` 實例，以便稍後附加到回應實例上，你可以使用全域的 `cookie` 輔助函數。除非此 cookie 附加到回應實例上，否則它不會被發送回用戶端：

    $cookie = cookie('name', 'value', $minutes);

    return response('Hello World')->cookie($cookie);


<a name="expiring-cookies-early"></a>
#### 提前使 Cookie 失效

你可以透過外發回應的 `withoutCookie` 方法使 cookie 失效來移除它：

    return response('Hello World')->withoutCookie('name');

如果你尚未擁有外發回應的實例，你可以使用 `Cookie` facade 的 `expire` 方法來使 cookie 失效：

    Cookie::expire('name');


<a name="cookies-and-encryption"></a>
### Cookie 與加密

預設情況下，由於 `Illuminate\Cookie\Middleware\EncryptCookies` 中介層，所有由 Laravel 產生的 cookie 都會被加密和簽署，以防止它們被用戶端修改或讀取。如果你想為應用程式產生的一部分 cookie 停用加密，你可以使用應用程式 `bootstrap/app.php` 檔案中的 `encryptCookies` 方法：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->encryptCookies(except: [
            'cookie_name',
        ]);
    })

<a name="redirects"></a>
## 重新導向

重新導向的回應是 `Illuminate\Http\RedirectResponse` 類別的實例，其中包含將使用者重新導向到另一個 URL 所需的適當標頭。有多種方法可以產生 `RedirectResponse` 實例。最簡單的方法是使用全域的 `redirect` 輔助函數：

    Route::get('/dashboard', function () {
        return redirect('/home/dashboard');
    });

有時您可能希望將使用者重新導向到他們之前的位置，例如當提交的表單無效時。您可以透過使用全域的 `back` 輔助函數來實現。由於此功能利用了 [session](/docs/{{version}}/session)，請確保呼叫 `back` 函數的路由正在使用 `web` 中介層群組：

    Route::post('/user/profile', function () {
        // Validate the request...

        return back()->withInput();
    });

<a name="redirecting-named-routes"></a>
### 重新導向至具名路由

當您呼叫不帶參數的 `redirect` 輔助函數時，會傳回 `Illuminate\Routing\Redirector` 的實例，讓您可以呼叫 `Redirector` 實例上的任何方法。例如，要產生重新導向至具名路由的 `RedirectResponse`，您可以使用 `route` 方法：

    return redirect()->route('login');

如果您的路由有參數，您可以將它們作為第二個引數傳遞給 `route` 方法：

    // For a route with the following URI: /profile/{id}

    return redirect()->route('profile', ['id' => 1]);

<a name="populating-parameters-via-eloquent-models"></a>
#### 透過 Eloquent Model 填充參數

如果您要重新導向到帶有「ID」參數的路由，且該參數是從 Eloquent model 填充的，您可以直接傳遞 model 本身。ID 將會自動提取：

    // For a route with the following URI: /profile/{id}

    return redirect()->route('profile', [$user]);

如果您想自訂放在路由參數中的值，您可以在路由參數定義中指定欄位 (`/profile/{id:slug}`)，或者您可以覆寫 Eloquent model 上的 `getRouteKey` 方法：

    /**
     * Get the value of the model's route key.
     */
    public function getRouteKey(): mixed
    {
        return $this->slug;
    }

<a name="redirecting-controller-actions"></a>
### 重新導向至控制器動作

您也可以產生重新導向至 [控制器動作](/docs/{{version}}/controllers)。為此，請將控制器和動作名稱傳遞給 `action` 方法：

    use App\Http\Controllers\UserController;

    return redirect()->action([UserController::class, 'index']);

如果您的控制器路由需要參數，您可以將它們作為第二個引數傳遞給 `action` 方法：

    return redirect()->action(
        [UserController::class, 'profile'], ['id' => 1]
    );

<a name="redirecting-external-domains"></a>
### 重新導向至外部網域

有時您可能需要重新導向到應用程式以外的網域。您可以透過呼叫 `away` 方法來實現，它會建立一個 `RedirectResponse`，而無需任何額外的 URL 編碼、驗證或確認：

    return redirect()->away('https://www.google.com');

<a name="redirecting-with-flashed-session-data"></a>
### 重新導向並快閃 Session 資料

重新導向到新的 URL 和 [快閃資料到 session](/docs/{{version}}/session#flash-data) 通常是同時進行的。通常，這是在成功執行操作後，當您向 session 快閃一條成功訊息時完成的。為了方便起見，您可以建立一個 `RedirectResponse` 實例並在單一、流暢的方法鏈中將資料快閃到 session：

    Route::post('/user/profile', function () {
        // ...

        return redirect('/dashboard')->with('status', 'Profile updated!');
    });

使用者重新導向後，您可以從 [session](/docs/{{version}}/session) 顯示快閃訊息。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

    @if (session('status'))
        <div class="alert alert-success">
            {{ session('status') }}
        </div>
    @endif

<a name="redirecting-with-input"></a>
#### 重新導向並帶入輸入

您可以使用 `RedirectResponse` 實例提供的 `withInput` 方法，在將使用者重新導向到新位置之前，將目前請求的輸入資料快閃到 session。這通常是在使用者遇到驗證錯誤時完成的。一旦輸入已快閃到 session，您就可以在下一個請求期間輕鬆 [檢索它](/docs/{{version}}/requests#retrieving-old-input) 以重新填充表單：

    return back()->withInput();

<a name="other-response-types"></a>
## 其他回應類型

`response` 輔助函式可用於生成其他類型的回應實例。當 `response` 輔助函式在沒有引數的情況下被呼叫時，會回傳 `Illuminate\Contracts\Routing\ResponseFactory` [契約](/docs/{{version}}/contracts)的實作。此契約提供了多種實用的方法來生成回應。

<a name="view-responses"></a>
### 視圖回應

如果你需要控制回應的狀態和標頭，同時也需要回傳一個[視圖](/docs/{{version}}/views)作為回應的內容，則應使用 `view` 方法：

    return response()
        ->view('hello', $data, 200)
        ->header('Content-Type', $type);

當然，如果你不需要傳遞自訂 HTTP 狀態碼或自訂標頭，則可以使用全域 `view` 輔助函式。

<a name="json-responses"></a>
### JSON 回應

`json` 方法會自動將 `Content-Type` 標頭設定為 `application/json`，並使用 PHP 的 `json_encode` 函式將給定的陣列轉換為 JSON：

    return response()->json([
        'name' => 'Abigail',
        'state' => 'CA',
    ]);

如果你想建立 JSONP 回應，則可以將 `json` 方法與 `withCallback` 方法結合使用：

    return response()
        ->json(['name' => 'Abigail', 'state' => 'CA'])
        ->withCallback($request->input('callback'));

<a name="file-downloads"></a>
### 檔案下載

`download` 方法可用於生成強制使用者瀏覽器下載指定路徑檔案的回應。`download` 方法接受一個檔名作為第二個引數，該檔名將決定使用者下載檔案時所看到的檔名。最後，你可以將一個 HTTP 標頭陣列作為第三個引數傳遞給該方法：

    return response()->download($pathToFile);

    return response()->download($pathToFile, $name, $headers);

> [!WARNING]  
> Symfony HttpFoundation 負責管理檔案下載，它要求下載的檔案必須具有 ASCII 檔名。

<a name="file-responses"></a>
### 檔案回應

`file` 方法可用於直接在使用者瀏覽器中顯示檔案，例如圖片或 PDF，而不是啟動下載。此方法接受檔案的絕對路徑作為其第一個引數，以及一個標頭陣列作為其第二個引數：

    return response()->file($pathToFile);

    return response()->file($pathToFile, $headers);

<a name="streamed-responses"></a>
### 串流回應

透過在產生資料時將其串流到客戶端，你可以顯著減少記憶體使用量並提高效能，特別是對於非常大的回應。串流回應允許客戶端在伺服器完成傳送資料之前開始處理資料：

    function streamedContent(): Generator {
        yield 'Hello, ';
        yield 'World!';
    }

    Route::get('/stream', function () {
        return response()->stream(function (): void {
            foreach (streamedContent() as $chunk) {
                echo $chunk;
                ob_flush();
                flush();
                sleep(2); // Simulate delay between chunks...
            }
        }, 200, ['X-Accel-Buffering' => 'no']);
    });

> [!NOTE]
> 在內部，Laravel 利用 PHP 的輸出緩衝功能。如上例所示，你應該使用 `ob_flush` 和 `flush` 函式將緩衝內容推送到客戶端。

<a name="streamed-json-responses"></a>
#### 串流 JSON 回應

如果你需要遞增地串流 JSON 資料，則可以使用 `streamJson` 方法。此方法對於需要以 JavaScript 易於解析的格式逐步傳送到瀏覽器的大型資料集特別有用：

    use App\Models\User;

    Route::get('/users.json', function () {
        return response()->streamJson([
            'users' => User::cursor(),
        ]);
    });

<a name="event-streams"></a>
#### 事件串流

`eventStream` 方法可用於使用 `text/event-stream` 內容類型回傳伺服器推播事件 (SSE) 串流回應。`eventStream` 方法接受一個閉包，該閉包應該在回應可用時將回應[產出 (yield)](https://www.php.net/manual/en/language.generators.overview.php) 到串流中：

```php
Route::get('/chat', function () {
    return response()->eventStream(function () {
        $stream = OpenAI::client()->chat()->createStreamed(...);

        foreach ($stream as $response) {
            yield $response->choices[0];
        }
    });
});
```

此事件串流可透過你應用程式前端的 [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) 物件來消費。當串流完成時，`eventStream` 方法會自動向事件串流傳送一個 `</stream>` 更新：

```js
const source = new EventSource('/chat');

source.addEventListener('update', (event) => {
    if (event.data === '</stream>') {
        source.close();

        return;
    }

    console.log(event.data);
})
```

<a name="streamed-downloads"></a>
#### 串流下載

有時你可能希望將給定操作的字串回應轉換為可下載的回應，而無需將操作內容寫入磁碟。在這種情況下，你可以使用 `streamDownload` 方法。此方法接受一個回呼、檔名和一個可選的標頭陣列作為其引數：

    use App\Services\GitHub;

    return response()->streamDownload(function () {
        echo GitHub::api('repo')
            ->contents()
            ->readme('laravel', 'laravel')['contents'];
    }, 'laravel-readme.md');

<a name="response-macros"></a>
## 回應巨集

如果你想定義一個可在多個路由和控制器中重複使用的自訂回應，則可以使用 `Response` facade 上的 `macro` 方法。通常，你應該從應用程式的其中一個[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法，例如 `App\Providers\AppServiceProvider` 服務提供者：

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Response;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            Response::macro('caps', function (string $value) {
                return Response::make(strtoupper($value));
            });
        }
    }

`macro` 函式接受一個名稱作為其第一個引數，一個閉包作為其第二個引數。當從 `ResponseFactory` 實作或 `response` 輔助函式呼叫巨集名稱時，巨集的閉包將被執行：

    return response()->caps('foo');
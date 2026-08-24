# HTTP 回應

- [建立回應](#creating-responses)
    - [附加 Header 至回應](#attaching-headers-to-responses)
    - [附加 Cookie 至回應](#attaching-cookies-to-responses)
    - [Cookie 與加密](#cookies-and-encryption)
- [重定向](#redirects)
    - [重定向至具名路由](#redirecting-named-routes)
    - [重定向至 Controller 動作](#redirecting-controller-actions)
    - [重定向至外部網域](#redirecting-external-domains)
    - [搭配 Session 快閃資料進行重定向](#redirecting-with-flashed-session-data)
- [其他回應類型](#other-response-types)
    - [View 回應](#view-responses)
    - [JSON 回應](#json-responses)
    - [檔案下載](#file-downloads)
    - [檔案回應](#file-responses)
- [串流回應](#streamed-responses)
    - [接收串流回應](#consuming-streamed-responses)
    - [串流 JSON 回應](#streamed-json-responses)
    - [事件串流 (SSE)](#event-streams)
    - [串流下載](#streamed-downloads)
- [回應 Macro](#response-macros)

<a name="creating-responses"></a>
## 建立回應


<a name="strings-arrays"></a>
#### 字串與陣列

所有的路由與控制器（Controller）都應該傳回一個回應以發送回使用者的瀏覽器。Laravel 提供了多種不同的回應傳回方式。最基本的發送回應方式是在路由或控制器中傳回一個字串。框架會自動將該字串轉換為完整的 HTTP 回應：

```php
Route::get('/', function () {
    return 'Hello World';
});
```

除了從路由與控制器傳回字串之外，您也可以傳回陣列。框架會自動將陣列轉換為 JSON 回應：

```php
Route::get('/', function () {
    return [1, 2, 3];
});
```

> [!NOTE]
> 您知道您也可以從路由或控制器中傳回 [Eloquent 集合](/docs/{{version}}/eloquent-collections) 嗎？它們會自動被轉換為 JSON。試試看吧！


<a name="response-objects"></a>
#### Response 物件

通常情況下，您不會只從路由動作中傳回簡單的字串或陣列。相反地，您將會傳回完整的 `Illuminate\Http\Response` 實例或 [View](/docs/{{version}}/views)。

傳回完整的 `Response` 實例允許您自訂回應的 HTTP 狀態碼與 Header。`Response` 實例繼承自 `Symfony\Component\HttpFoundation\Response` 類別，該類別提供了各種建立 HTTP 回應的方法：

```php
Route::get('/home', function () {
    return response('Hello World', 200)
        ->header('Content-Type', 'text/plain');
});
```


<a name="eloquent-models-and-collections"></a>
#### Eloquent Model 與集合

您也可以直接從路由和控制器傳回 [Eloquent ORM](/docs/{{version}}/eloquent) Model 與 Collection。當您這麼做時，Laravel 會自動將 Model 與 Collection 轉換為 JSON 回應，同時也會尊重 Model 的 [隱藏屬性](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)：

```php
use App\Models\User;

Route::get('/user/{user}', function (User $user) {
    return $user;
});
```


<a name="attaching-headers-to-responses"></a>
### 附加 Header 至回應

請記住，大多數的回應方法都可以進行鏈結呼叫（Chainable），從而能夠流暢地建構回應實例。例如，您可以在將回應發送回給使用者之前，使用 `header` 方法向回應新增一系列的 Header：

```php
return response($content)
    ->header('Content-Type', $type)
    ->header('X-Header-One', 'Header Value')
    ->header('X-Header-Two', 'Header Value');
```

或者，您可以搭配使用 `withHeaders` 方法來指定要新增至回應的 Header 陣列：

```php
return response($content)
    ->withHeaders([
        'Content-Type' => $type,
        'X-Header-One' => 'Header Value',
        'X-Header-Two' => 'Header Value',
    ]);
```

您可以透過 `withoutHeader` 方法從即將發出的回應中移除特定的 Header：

```php
return response($content)->withoutHeader('X-Debug');

return response($content)->withoutHeader(['X-Debug', 'X-Powered-By']);
```


<a name="cache-control-middleware"></a>
#### 快取控制中介層

Laravel 包含了一個 `cache.headers` 中介層，可用於為一組路由快速設定 `Cache-Control` Header。指示語（Directive）應使用對應快取控制指示語的「蛇形命名 (snake case)」等效名稱來提供，並以分號分隔。如果在指示語清單中指定了 `etag`，則回應內容的 MD5 Hash 值將會自動設定為 ETag 識別碼：

```php
Route::middleware('cache.headers:public;max_age=30;s_maxage=300;stale_while_revalidate=600;etag')->group(function () {
    Route::get('/privacy', function () {
        // ...
    });

    Route::get('/terms', function () {
        // ...
    });
});
```


<a name="attaching-cookies-to-responses"></a>
### 附加 Cookie 至回應

您可以使用 `cookie` 方法將 cookie 附加到發出的 `Illuminate\Http\Response` 實例上。您應該傳入名稱、值以及該 cookie 被視為有效的分鐘數給此方法：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes
);
```

`cookie` 方法還接受另外幾個較不常用的引數。一般來說，這些引數與 PHP 原生 [setcookie](https://secure.php.net/manual/en/function.setcookie.php) 方法所給予的引數具有相同的目的和意義：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes, $path, $domain, $secure, $httpOnly
);
```

如果您想確保 cookie 會隨發出的回應一起發送，但您手頭上還沒有該回應的實例，您可以使用 `Cookie` Facade 來將 cookie 排入「佇列」，以便在發送回應時附加上去。`queue` 方法接受建立 cookie 實例所需的引數。這些 cookie 將會在發送到瀏覽器之前附加至即將發出的回應中：

```php
use Illuminate\Support\Facades\Cookie;

Cookie::queue('name', 'value', $minutes);
```


<a name="generating-cookie-instances"></a>
#### 產生 Cookie 實例

如果您想要產生一個可以在稍後附加到回應實例上的 `Symfony\Component\HttpFoundation\Cookie` 實例，可以使用全域的 `cookie` 輔助函式。除非將此 cookie 附加到回應實例上，否則它不會被發送回用戶端：

```php
$cookie = cookie('name', 'value', $minutes);

return response('Hello World')->cookie($cookie);
```


<a name="expiring-cookies-early"></a>
#### 讓 Cookie 提前過期

您可以透過發出回應的 `withoutCookie` 或 `withoutCookies` 方法使 cookie 過期，從而將其移除：

```php
return response('Hello World')->withoutCookie('name');

return response('Hello World')->withoutCookies([
    'name',
    'email',
    'preferences',
]);
```

如果您尚未擁有即將發出的回應實例，可以使用 `Cookie` Facade 的 `expire` 方法讓 cookie 過期：

```php
Cookie::expire('name');
```


<a name="cookies-and-encryption"></a>
### Cookie 與加密

預設情況下，受惠於 `Illuminate\Cookie\Middleware\EncryptCookies` 中介層，Laravel 產生的所有 cookie 都經過加密與簽名，因此用戶端無法為其進行修改或讀取。如果您想停用應用程式產生的部分 cookie 加密功能，可以在應用程式的 `bootstrap/app.php` 檔案中使用 `encryptCookies` 方法：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->encryptCookies(except: [
        'cookie_name',
    ]);
})
```

> [!NOTE]
> 一般而言，絕不應該停用 cookie 加密，因為這會使您的 cookie 暴露於用戶端資料洩露與遭篡改的風險之中。

<a name="redirects"></a>
## 重定向

重定向回應是 `Illuminate\Http\RedirectResponse` 類別的實例，包含將使用者重定向至另一個 URL 所需的適當 Header。有幾種方式可以產生 `RedirectResponse` 實例。最簡單的方法是使用全域的 `redirect` 輔助函式：

```php
Route::get('/dashboard', function () {
    return redirect('/home/dashboard');
});
```

有時候您可能希望將使用者重定向回先前的位址，例如當提交的表單無效時。您可以透過全域的 `back` 輔助函式來達成此目的。由於此功能使用了 [Session](/docs/{{version}}/session)，請確保呼叫 `back` 函式的路由使用了 `web` 中介層群組：

```php
Route::post('/user/profile', function () {
    // Validate the request...

    return back()->withInput();
});
```


<a name="redirecting-named-routes"></a>
### 重定向至具名路由

當您呼叫不帶參數的 `redirect` 輔助函式時，將會回傳 `Illuminate\Routing\Redirector` 的實例，讓您可以呼叫 `Redirector` 實例上的任何方法。例如，要產生重定向至具名路由的 `RedirectResponse`，您可以使用 `route` 方法：

```php
return redirect()->route('login');
```

如果您的路由帶有參數，您可以將它們作為第二個引數傳遞給 `route` 方法：

```php
// For a route with the following URI: /profile/{id}

return redirect()->route('profile', ['id' => 1]);
```


<a name="populating-parameters-via-eloquent-models"></a>
#### 透過 Eloquent 模型填入參數

如果您要重定向至一個帶有 "ID" 參數的路由，且該參數是從 Eloquent 模型填入的，您可以直接傳遞模型本身。ID 將會被自動提取：

```php
// For a route with the following URI: /profile/{id}

return redirect()->route('profile', [$user]);
```

如果您想自訂放置在路由參數中的值，您可以在路由參數定義中指定欄位 (`/profile/{id:slug}`)，或者您可以覆寫 Eloquent 模型上的 `getRouteKey` 方法：

```php
/**
 * Get the value of the model's route key.
 */
public function getRouteKey(): mixed
{
    return $this->slug;
}
```


<a name="redirecting-controller-actions"></a>
### 重定向至 Controller 動作

您也可以產生重定向至 [Controller 動作](/docs/{{version}}/controllers)。為此，請將 Controller 與動作名稱傳遞給 `action` 方法：

```php
use App\Http\Controllers\UserController;

return redirect()->action([UserController::class, 'index']);
```

如果您的 Controller 路由需要參數，您可以將它們作為第二個引數傳遞給 `action` 方法：

```php
return redirect()->action(
    [UserController::class, 'profile'], ['id' => 1]
);
```


<a name="redirecting-external-domains"></a>
### 重定向至外部網域

有時候您可能需要重定向至應用程式之外的網域。您可以透過呼叫 `away` 方法來做到這一點，它會建立一個不帶有任何額外 URL 編碼、驗證或確認的 `RedirectResponse`：

```php
return redirect()->away('https://www.google.com');
```


<a name="redirecting-with-flashed-session-data"></a>
### 搭配 Session 快閃資料進行重定向

重定向至新的 URL 與[將資料快閃寫入 Session](/docs/{{version}}/session#flash-data) 通常是同時進行的。這通常是在成功執行某個動作後，將成功訊息快閃寫入 Session 時使用。為了方便起見，您可以在單一且流暢的方法鏈中建立 `RedirectResponse` 實例並將資料快閃寫入 Session：

```php
Route::post('/user/profile', function () {
    // ...

    return redirect('/dashboard')->with('status', 'Profile updated!');
});
```

使用者被重定向後，您可以顯示來自 [Session](/docs/{{version}}/session) 的快閃訊息。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
@if (session('status'))
    <div class="alert alert-success">
        {{ session('status') }}
    </div>
@endif
```


<a name="redirecting-with-input"></a>
#### 搭配輸入資料進行重定向

您可以使用 `RedirectResponse` 實例提供的 `withInput` 方法，在將使用者重定向至新位置之前，將當前請求的輸入資料快閃寫入 Session。這通常用於使用者遇到驗證錯誤時。一旦輸入資料被快閃寫入 Session，您就可以在下一次請求時輕鬆地[取得它](/docs/{{version}}/requests#retrieving-old-input)，以重新填入表單：

```php
return back()->withInput();
```


<a name="other-response-types"></a>
## 其他回應類型

`response` 輔助函式可以用來產生其他類型的回應實例。當呼叫不帶引數的 `response` 輔助函式時，將會回傳 `Illuminate\Contracts\Routing\ResponseFactory` [契約(Contracts)](/docs/{{version}}/contracts) 的實作。此契約提供了幾個有用的方法來產生回應。


<a name="view-responses"></a>
### View 回應

如果您需要控制回應的狀態與 Header，但同時也需要回傳 [View](/docs/{{version}}/views) 作為回應內容，您應該使用 `view` 方法：

```php
return response()
    ->view('hello', $data, 200)
    ->header('Content-Type', $type);
```

當然，如果您不需要傳遞自訂的 HTTP 狀態碼或自訂 Header，您可以直接使用全域的 `view` 輔助函式。


<a name="json-responses"></a>
### JSON 回應

`json` 方法會自動將 `Content-Type` Header 設定為 `application/json`，並使用 PHP 的 `json_encode` 函式將給定的陣列轉換為 JSON：

```php
return response()->json([
    'name' => 'Abigail',
    'state' => 'CA',
]);
```

如果您想要建立 JSONP 回應，可以結合使用 `json` 方法與 `withCallback` 方法：

```php
return response()
    ->json(['name' => 'Abigail', 'state' => 'CA'])
    ->withCallback($request->input('callback'));
```


<a name="file-downloads"></a>
### 檔案下載

`download` 方法可用於產生一個強制使用者瀏覽器下載指定路徑檔案的回應。`download` 方法接受檔名作為其第二個引數，這將決定下載該檔案的使用者所看到的檔名。最後，您可以傳遞一個 HTTP Header 陣列作為該方法的第三個引數：

```php
return response()->download($pathToFile);

return response()->download($pathToFile, $name, $headers);
```

> [!WARNING]
> 管理檔案下載的 Symfony HttpFoundation 要求被下載的檔案必須擁有 ASCII 檔名。


<a name="file-responses"></a>
### 檔案回應

`file` 方法可用於直接在使用者的瀏覽器中顯示檔案（例如圖片或 PDF），而不是發起下載。該方法接受檔案的絕對路徑作為其第一個引數，並接受 Header 陣列作為其第二個引數：

```php
return response()->file($pathToFile);

return response()->file($pathToFile, $headers);
```

<a name="streamed-responses"></a>
## 串流回應

透過在資料生成時即時將其串流傳輸至用戶端，您可以大幅減少記憶體使用量並提升效能，特別是在處理非常龐大的回應時。串流回應允許用戶端在伺服器完成發送之前就開始處理資料：

```php
Route::get('/stream', function () {
    return response()->stream(function (): void {
        foreach (['developer', 'admin'] as $string) {
            echo $string;
            ob_flush();
            flush();
            sleep(2); // Simulate delay between chunks...
        }
    }, 200, ['X-Accel-Buffering' => 'no']);
});
```

為方便起見，若您提供給 `stream` 方法的 Closure 回傳一個 [Generator](https://www.php.net/manual/en/language.generators.overview.php)，Laravel 將會自動在 Generator 回傳的字串之間清除（Flush）輸出緩衝區，並自動停用 Nginx 的輸出緩衝：

```php
Route::post('/chat', function () {
    return response()->stream(function (): Generator {
        $stream = OpenAI::client()->chat()->createStreamed(...);

        foreach ($stream as $response) {
            yield $response->choices[0];
        }
    });
});
```

<a name="consuming-streamed-responses"></a>
### 接收串流回應

串流回應可以使用 Laravel 的 `stream` npm 套件來接收，該套件提供了方便的 API 用於與 Laravel 的回應串流及事件串流進行互動。首先，請安裝 `@laravel/stream-react`、`@laravel/stream-vue` 或 `@laravel/stream-svelte` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

```shell tab=Svelte
npm install @laravel/stream-svelte
```

接著，可以使用 `useStream` 來接收事件串流。提供串流 URL 後，隨著 Laravel 應用程式傳回內容，該 Hook 將會自動更新 `data` 並包含串接後的回應：

```tsx tab=React
import { useStream } from "@laravel/stream-react";

function App() {
    const { data, isFetching, isStreaming, send } = useStream("chat");

    const sendMessage = () => {
        send({
            message: `Current timestamp: ${Date.now()}`,
        });
    };

    return (
        <div>
            <div>{data}</div>
            {isFetching && <div>Connecting...</div>}
            {isStreaming && <div>Generating...</div>}
            <button onClick={sendMessage}>Send Message</button>
        </div>
    );
}
```

```vue tab=Vue
<script setup lang="ts">
import { useStream } from "@laravel/stream-vue";

const { data, isFetching, isStreaming, send } = useStream("chat");

const sendMessage = () => {
    send({
        message: `Current timestamp: ${Date.now()}`,
    });
};
</script>

<template>
    <div>
        <div>{{ data }}</div>
        <div v-if="isFetching">Connecting...</div>
        <div v-if="isStreaming">Generating...</div>
        <button @click="sendMessage">Send Message</button>
    </div>
</template>
```

```svelte tab=Svelte
<script>
import { useStream } from "@laravel/stream-svelte";

const stream = useStream("chat");

const sendMessage = () => {
    stream.send({
        message: `Current timestamp: ${Date.now()}`,
    });
};
</script>

<div>
    <div>{$stream.data}</div>
    {#if $stream.isFetching}
        <div>Connecting...</div>
    {/if}
    {#if $stream.isStreaming}
        <div>Generating...</div>
    {/if}
    <button onclick={sendMessage}>Send Message</button>
</div>
```

當透過 `send` 將資料傳回串流時，系統會在傳送新資料前取消目前與串流的連線。所有的請求都會以 JSON `POST` 請求發送。

> [!WARNING]
> 由於 `useStream` Hook 會對您的應用程式發送 `POST` 請求，因此需要有效的 CSRF Token。提供 CSRF Token 最簡單的方式是[透過 Meta 標籤包含在應用程式版面配置的 head 中](/docs/{{version}}/csrf#csrf-x-csrf-token)。

傳給 `useStream` 的第二個引數是一個選項物件，您可以用來自訂串流接收的行為。該物件的預設值如下所示：

```tsx tab=React
import { useStream } from "@laravel/stream-react";

function App() {
    const { data } = useStream("chat", {
        id: undefined,
        initialInput: undefined,
        headers: undefined,
        csrfToken: undefined,
        onResponse: (response: Response) => void,
        onData: (data: string) => void,
        onCancel: () => void,
        onFinish: () => void,
        onError: (error: Error) => void,
    });

    return <div>{data}</div>;
}
```

```vue tab=Vue
<script setup lang="ts">
import { useStream } from "@laravel/stream-vue";

const { data } = useStream("chat", {
    id: undefined,
    initialInput: undefined,
    headers: undefined,
    csrfToken: undefined,
    onResponse: (response: Response) => void,
    onData: (data: string) => void,
    onCancel: () => void,
    onFinish: () => void,
    onError: (error: Error) => void,
});
</script>

<template>
    <div>{{ data }}</div>
</template>
```

```svelte tab=Svelte
<script>
import { useStream } from "@laravel/stream-svelte";

const stream = useStream("chat", {
    id: undefined,
    initialInput: undefined,
    headers: undefined,
    csrfToken: undefined,
    onResponse: (response) => {},
    onData: (data) => {},
    onCancel: () => {},
    onFinish: () => {},
    onError: (error) => {},
});
</script>

<div>{$stream.data}</div>
```

在串流成功傳回初始回應後會觸發 `onResponse`，且原始的 [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) 會傳遞給 Callback。每當接收到一個區塊 (chunk) 時就會呼叫 `onData` — 當前的區塊會傳遞給 Callback。當串流結束以及在擷取/讀取週期中拋出錯誤時，都會呼叫 `onFinish`。

預設情況下，初始化時不會對串流發送請求。您可以透過 `initialInput` 選項向串流傳遞初始 Payload：

```tsx tab=React
import { useStream } from "@laravel/stream-react";

function App() {
    const { data } = useStream("chat", {
        initialInput: {
            message: "Introduce yourself.",
        },
    });

    return <div>{data}</div>;
}
```

```vue tab=Vue
<script setup lang="ts">
import { useStream } from "@laravel/stream-vue";

const { data } = useStream("chat", {
    initialInput: {
        message: "Introduce yourself.",
    },
});
</script>

<template>
    <div>{{ data }}</div>
</template>
```

```svelte tab=Svelte
<script>
import { useStream } from "@laravel/stream-svelte";

const stream = useStream("chat", {
    initialInput: {
        message: "Introduce yourself.",
    },
});
</script>

<div>{$stream.data}</div>
```

若要手動取消串流，您可以使用 Hook 傳回的 `cancel` 方法：

```tsx tab=React
import { useStream } from "@laravel/stream-react";

function App() {
    const { data, cancel } = useStream("chat");

    return (
        <div>
            <div>{data}</div>
            <button onClick={cancel}>Cancel</button>
        </div>
    );
}
```

```vue tab=Vue
<script setup lang="ts">
import { useStream } from "@laravel/stream-vue";

const { data, cancel } = useStream("chat");
</script>

<template>
    <div>
        <div>{{ data }}</div>
        <button @click="cancel">Cancel</button>
    </div>
</template>
```

```svelte tab=Svelte
<script>
import { useStream } from "@laravel/stream-svelte";

const stream = useStream("chat");
</script>

<div>
    <div>{$stream.data}</div>
    <button onclick={() => stream.cancel()}>Cancel</button>
</div>
```

每次使用 `useStream` Hook 時，都會產生一個隨機的 `id` 來識別該串流。這會在每次發送請求時，於 `X-STREAM-ID` Header 中傳回給伺服器。當從多個元件接收相同的串流時，您可以透過提供自己的 `id` 來讀取及寫入該串流：

```tsx tab=React
// App.tsx
import { useStream } from "@laravel/stream-react";

function App() {
    const { data, id } = useStream("chat");

    return (
        <div>
            <div>{data}</div>
            <StreamStatus id={id} />
        </div>
    );
}

// StreamStatus.tsx
import { useStream } from "@laravel/stream-react";

function StreamStatus({ id }) {
    const { isFetching, isStreaming } = useStream("chat", { id });

    return (
        <div>
            {isFetching && <div>Connecting...</div>}
            {isStreaming && <div>Generating...</div>}
        </div>
    );
}
```

```vue tab=Vue
<!-- App.vue -->
<script setup lang="ts">
import { useStream } from "@laravel/stream-vue";
import StreamStatus from "./StreamStatus.vue";

const { data, id } = useStream("chat");
</script>

<template>
    <div>
        <div>{{ data }}</div>
        <StreamStatus :id="id" />
    </div>
</template>

<!-- StreamStatus.vue -->
<script setup lang="ts">
import { useStream } from "@laravel/stream-vue";

const props = defineProps<{
    id: string;
}>();

const { isFetching, isStreaming } = useStream("chat", { id: props.id });
</script>

<template>
    <div>
        <div v-if="isFetching">Connecting...</div>
        <div v-if="isStreaming">Generating...</div>
    </div>
</template>
```

```svelte tab=Svelte
<!-- App.svelte -->
<script>
import { useStream } from "@laravel/stream-svelte";
import StreamStatus from "./StreamStatus.svelte";

const stream = useStream("chat");
</script>

<div>
    <div>{$stream.data}</div>
    <StreamStatus id={stream.id} />
</div>

<!-- StreamStatus.svelte -->
<script>
import { useStream } from "@laravel/stream-svelte";

let { id } = $props();

const stream = useStream("chat", { id });
</script>

<div>
    {#if $stream.isFetching}
        <div>Connecting...</div>
    {/if}
    {#if $stream.isStreaming}
        <div>Generating...</div>
    {/if}
</div>
```

<a name="streamed-json-responses"></a>
### 串流 JSON 回應

如果您需要增量傳輸 JSON 資料，可以使用 `streamJson` 方法。對於需要以 JavaScript 能輕鬆解析的格式逐步傳送到瀏覽器的大型資料集來說，這個方法特別有用：

```php
use App\Models\User;

Route::get('/users.json', function () {
    return response()->streamJson([
        'users' => User::cursor(),
    ]);
});
```

`useJsonStream` hook 與 [useStream hook](#consuming-streamed-responses) 完全相同，唯一的差別在於它會在串流傳輸完成後，嘗試將資料解析為 JSON：

```tsx tab=React
import { useJsonStream } from "@laravel/stream-react";

type User = {
    id: number;
    name: string;
    email: string;
};

function App() {
    const { data, send } = useJsonStream<{ users: User[] }>("users");

    const loadUsers = () => {
        send({
            query: "taylor",
        });
    };

    return (
        <div>
            <ul>
                {data?.users.map((user) => (
                    <li>
                        {user.id}: {user.name}
                    </li>
                ))}
            </ul>
            <button onClick={loadUsers}>Load Users</button>
        </div>
    );
}
```

```vue tab=Vue
<script setup lang="ts">
import { useJsonStream } from "@laravel/stream-vue";

type User = {
    id: number;
    name: string;
    email: string;
};

const { data, send } = useJsonStream<{ users: User[] }>("users");

const loadUsers = () => {
    send({
        query: "taylor",
    });
};
</script>

<template>
    <div>
        <ul>
            <li v-for="user in data?.users" :key="user.id">
                {{ user.id }}: {{ user.name }}
            </li>
        </ul>
        <button @click="loadUsers">Load Users</button>
    </div>
</template>
```

```svelte tab=Svelte
<script>
import { useJsonStream } from "@laravel/stream-svelte";

const stream = useJsonStream("users");

const loadUsers = () => {
    stream.send({
        query: "taylor",
    });
};
</script>

<div>
    <ul>
        {#if $stream.data?.users}
            {#each $stream.data.users as user (user.id)}
                <li>{user.id}: {user.name}</li>
            {/each}
        {/if}
    </ul>
    <button onclick={loadUsers}>Load Users</button>
</div>
```

<a name="event-streams"></a>
### 事件串流 (SSE)

`eventStream` 方法可用於使用 `text/event-stream` 內容類型傳回伺服器發送事件 (SSE) 串流回應。`eventStream` 方法接收一個閉包，當回應可用時，該閉包應該將回應 [yield](https://www.php.net/manual/en/language.generators.overview.php) 至串流中：

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

如果您想要自訂事件的名稱，可以 yield 一個 `StreamedEvent` 類別的實例：

```php
use Illuminate\Http\StreamedEvent;

yield new StreamedEvent(
    event: 'update',
    data: $response->choices[0],
);
```

<a name="consuming-event-streams"></a>
#### 接收事件串流

您可以使用 Laravel 的 `stream` npm 套件來接收事件串流，該套件提供了便利的 API 與 Laravel 事件串流互動。首先，請安裝 `@laravel/stream-react`、`@laravel/stream-vue` 或 `@laravel/stream-svelte` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

```shell tab=Svelte
npm install @laravel/stream-svelte
```

接著，可以使用 `useEventStream` 來接收事件串流。提供串流 URL 後，隨著訊息從您的 Laravel 應用程式傳回，此 hook 將會自動以連接的回應更新 `message`：

```jsx tab=React
import { useEventStream } from "@laravel/stream-react";

function App() {
  const { message } = useEventStream("/chat");

  return <div>{message}</div>;
}
```

```vue tab=Vue
<script setup lang="ts">
import { useEventStream } from "@laravel/stream-vue";

const { message } = useEventStream("/chat");
</script>

<template>
  <div>{{ message }}</div>
</template>
```

```svelte tab=Svelte
<script>
import { useEventStream } from "@laravel/stream-svelte";

const eventStream = useEventStream("/chat");
</script>

<div>{$eventStream.message}</div>
```

傳給 `useEventStream` 的第二個引數是一個選項物件，您可用來自訂串流接收行為。該物件的預設值如下所示：

```jsx tab=React
import { useEventStream } from "@laravel/stream-react";

function App() {
  const { message } = useEventStream("/stream", {
    eventName: "update",
    onMessage: (message) => {
      //
    },
    onError: (error) => {
      //
    },
    onComplete: () => {
      //
    },
    endSignal: "</stream>",
    glue: " ",
  });

  return <div>{message}</div>;
}
```

```vue tab=Vue
<script setup lang="ts">
import { useEventStream } from "@laravel/stream-vue";

const { message } = useEventStream("/chat", {
  eventName: "update",
  onMessage: (message) => {
    // ...
  },
  onError: (error) => {
    // ...
  },
  onComplete: () => {
    // ...
  },
  endSignal: "</stream>",
  glue: " ",
});
</script>
```

```svelte tab=Svelte
<script>
import { useEventStream } from "@laravel/stream-svelte";

const eventStream = useEventStream("/chat", {
    eventName: "update",
    onMessage: (event) => {
        //
    },
    onError: (error) => {
        //
    },
    onComplete: () => {
        //
    },
    endSignal: "</stream>",
    glue: " ",
    replace: false,
});
</script>
```

事件串流也可以由應用程式的前端透過 [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) 物件手動接收。當串流完成時，`eventStream` 方法會自動傳送 `</stream>` 更新至事件串流：

```js
const source = new EventSource('/chat');

source.addEventListener('update', (event) => {
    if (event.data === '</stream>') {
        source.close();

        return;
    }

    console.log(event.data);
});
```

若要自訂傳送到事件串流的最後一個事件，您可以為 `eventStream` 方法的 `endStreamWith` 引數提供一個 `StreamedEvent` 實例：

```php
return response()->eventStream(function () {
    // ...
}, endStreamWith: new StreamedEvent(event: 'update', data: '</stream>'));
```

<a name="streamed-downloads"></a>
### 串流下載

有時您可能希望將特定操作的字串回應轉換為可下載的回應，而無需將操作內容寫入磁碟。在這種情況下，您可以使用 `streamDownload` 方法。該方法接收一個回呼函式、檔案名稱以及可選的 Header 陣列作為其引數：

```php
use App\Services\GitHub;

return response()->streamDownload(function () {
    echo GitHub::api('repo')
        ->contents()
        ->readme('laravel', 'laravel')['contents'];
}, 'laravel-readme.md');
```

<a name="response-macros"></a>
## 回應 Macro

如果您想要定義可以在各種路由與 Controller 中重複使用的自訂回應，可以使用 `Response` Facade 上的 `macro` 方法。通常，您應該在應用程式的[服務提供者(Service Providers)](/docs/{{version}}/providers)（例如 `App\Providers\AppServiceProvider` 服務提供者）中的 `boot` 方法呼叫此方法：

```php
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
```

`macro` 函式接收名稱作為第一個引數，並接收 Closure 作為第二個引數。當從 `ResponseFactory` 實作或 `response` 輔助函式呼叫該 Macro 名稱時，將會執行該 Macro 的 Closure：

```php
return response()->caps('foo');
```
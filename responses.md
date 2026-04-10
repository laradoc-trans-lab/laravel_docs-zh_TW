# HTTP 回應

- [建立回應](#creating-responses)
    - [為回應附加標頭](#attaching-headers-to-responses)
    - [為回應附加 Cookie](#attaching-cookies-to-responses)
    - [Cookie 與加密](#cookies-and-encryption)
- [重新導向](#redirects)
    - [重新導向至具名路由](#redirecting-named-routes)
    - [重新導向至控制器動作](#redirecting-controller-actions)
    - [重新導向至外部網域](#redirecting-external-domains)
    - [使用快閃工作階段資料重新導向](#redirecting-with-flashed-session-data)
- [其他回應類型](#other-response-types)
    - [視圖回應](#view-responses)
    - [JSON 回應](#json-responses)
    - [檔案下載](#file-downloads)
    - [檔案回應](#file-responses)
- [串流回應](#streamed-responses)
    - [消費串流回應](#consuming-streamed-responses)
    - [串流 JSON 回應](#streamed-json-responses)
    - [事件串流 (SSE)](#event-streams)
    - [串流下載](#streamed-downloads)
- [回應巨集](#response-macros)

<a name="creating-responses"></a>
## 建立回應


<a name="strings-arrays"></a>
#### 字串與陣列

所有的路由與控制器都應該回傳一個回應，以便傳送回使用者的瀏覽器。Laravel 提供了幾種不同的方式來回傳回應。最基本的回應是從路由或控制器回傳一個字串。框架會自動將該字串轉換為完整的 HTTP 回應：

```php
Route::get('/', function () {
    return 'Hello World';
});
```

除了從路由和控制器回傳字串外，您也可以回傳陣列。框架會自動將陣列轉換為 JSON 回應：

```php
Route::get('/', function () {
    return [1, 2, 3];
});
```

> [!NOTE]
> 您知道您也可以從路由或控制器回傳 [Eloquent 集合](/docs/{{version}}/eloquent-collections) 嗎？它們會自動被轉換為 JSON。試試看吧！


<a name="response-objects"></a>
#### 回應物件

通常情況下，您不會只從路由動作中回傳簡單的字串或陣列。相反地，您將會回傳完整的 `Illuminate\Http\Response` 實例或 [視圖](/docs/{{version}}/views)。

回傳完整的 `Response` 實例允許您自訂回應的 HTTP 狀態碼與標頭。`Response` 實例繼承自 `Symfony\Component\HttpFoundation\Response` 類別，該類別提供了多種建立 HTTP 回應的方法：

```php
Route::get('/home', function () {
    return response('Hello World', 200)
        ->header('Content-Type', 'text/plain');
});
```


<a name="eloquent-models-and-collections"></a>
#### Eloquent 模型與集合

您也可以直接從路由和控制器回傳 [Eloquent ORM](/docs/{{version}}/eloquent) 模型與集合。當您這樣做時，Laravel 會自動將模型與集合轉換為 JSON 回應，同時會尊重模型的 [隱藏屬性](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)：

```php
use App\Models\User;

Route::get('/user/{user}', function (User $user) {
    return $user;
});
```


<a name="attaching-headers-to-responses"></a>
### 為回應附加標頭

請記住，大多數的回應方法都是可鏈結的，這允許以流暢的方式建構回應實例。例如，您可以使用 `header` 方法在將回應傳送回使用者之前，為回應添加一系列標頭：

```php
return response($content)
    ->header('Content-Type', $type)
    ->header('X-Header-One', 'Header Value')
    ->header('X-Header-Two', 'Header Value');
```

或者，您可以使用 `withHeaders` 方法來指定要添加到回應中的標頭陣列：

```php
return response($content)
    ->withHeaders([
        'Content-Type' => $type,
        'X-Header-One' => 'Header Value',
        'X-Header-Two' => 'Header Value',
    ]);
```

您可以使用 `withoutHeader` 方法從發出的回應中移除特定的標頭：

```php
return response($content)->withoutHeader('X-Debug');

return response($content)->withoutHeader(['X-Debug', 'X-Powered-By']);
```


<a name="cache-control-middleware"></a>
#### 快取控制中介層

Laravel 包含一個 `cache.headers` 中介層，可用於快速為一組路由設定 `Cache-Control` 標頭。指令應使用對應快取控制指令的「蛇形命名法 (snake case)」等效形式，並以分號分隔。如果在指令列表中指定了 `etag`，回應內容的 MD5 雜湊值將自動被設定為 ETag 識別碼：

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
### 為回應附加 Cookie

您可以使用 `cookie` 方法將 Cookie 附加到發出的 `Illuminate\Http\Response` 實例。您應該將名稱、值以及該 Cookie 應被視為有效的分鐘數傳遞給此方法：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes
);
```

`cookie` 方法還接受一些較少使用的引數。通常，這些引數的目的和含義與傳遞給 PHP 原生 [setcookie](https://secure.php.net/manual/en/function.setcookie.php) 方法的引數相同：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes, $path, $domain, $secure, $httpOnly
);
```

如果您想確保 Cookie 與發出的回應一起傳送，但尚未擁有該回應的實例，您可以使用 `Cookie` Facade 來「佇列 (queue)」Cookie，以便在傳送回應時將其附加。`queue` 方法接受建立 Cookie 實例所需的引數。這些 Cookie 將在傳送至瀏覽器之前附加到發出的回應中：

```php
use Illuminate\Support\Facades\Cookie;

Cookie::queue('name', 'value', $minutes);
```


<a name="generating-cookie-instances"></a>
#### 產生 Cookie 實例

如果您想產生一個 `Symfony\Component\HttpFoundation\Cookie` 實例，以便稍後附加到回應實例，您可以使用全域的 `cookie` 助手函數。除非該 Cookie 被附加到回應實例，否則不會被傳送回用戶端：

```php
$cookie = cookie('name', 'value', $minutes);

return response('Hello World')->cookie($cookie);
```


<a name="expiring-cookies-early"></a>
#### 提前使 Cookie 過期

您可以通过發出回應的 `withoutCookie` 方法使 Cookie 過期來移除它：

```php
return response('Hello World')->withoutCookie('name');
```

如果您尚未擁有發出回應的實例，您可以使用 `Cookie` Facade 的 `expire` 方法使 Cookie 過期：

```php
Cookie::expire('name');
```


<a name="cookies-and-encryption"></a>
### Cookie 與加密

預設情況下，得益於 `Illuminate\Cookie\Middleware\EncryptCookies` 中介層，Laravel 產生的所有 Cookie 都經過加密與簽署，因此無法被用戶端修改或讀取。如果您想為應用程式產生的部分 Cookie 禁用加密，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `encryptCookies` 方法：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->encryptCookies(except: [
        'cookie_name',
    ]);
})
```

> [!NOTE]
> 一般而言，不應禁用 Cookie 加密，因為這會使您的 Cookie 面臨潛在的用戶端資料外洩與篡改風險。

<a name="redirects"></a>
## 重新導向

重新導向回應是 `Illuminate\Http\RedirectResponse` 類別的實例，其中包含將使用者重新導向至另一個 URL 所需的正確標頭。有幾種方法可以產生 `RedirectResponse` 實例，最簡單的方法是使用全域的 `redirect` 輔助函數：

```php
Route::get('/dashboard', function () {
    return redirect('/home/dashboard');
});
```

有時您可能希望將使用者重新導向回他們之前的位置，例如當提交的表單無效時。您可以使用全域的 `back` 輔助函數來達成。由於此功能使用了 [工作階段](/docs/{{version}}/session)，請確保呼叫 `back` 函數的路由使用了 `web` 中介層群組：

```php
Route::post('/user/profile', function () {
    // Validate the request...

    return back()->withInput();
});
```


<a name="redirecting-named-routes"></a>
### 重新導向至具名路由

當您在不帶參數的情況下呼叫 `redirect` 輔助函數時，會回傳一個 `Illuminate\Routing\Redirector` 實例，讓您可以呼叫該 `Redirector` 實例上的任何方法。例如，要產生一個重新導向至具名路由的 `RedirectResponse`，您可以使用 `route` 方法：

```php
return redirect()->route('login');
```

如果您的路由具有參數，您可以將它們作為 `route` 方法的第二個引數傳遞：

```php
// For a route with the following URI: /profile/{id}

return redirect()->route('profile', ['id' => 1]);
```


<a name="populating-parameters-via-eloquent-models"></a>
#### 透過 Eloquent 模型填充參數

如果您正重新導向至一個具有由 Eloquent 模型填充的「ID」參數之路由，您可以直接傳遞該模型。ID 將會自動被提取：

```php
// For a route with the following URI: /profile/{id}

return redirect()->route('profile', [$user]);
```

如果您想自定義放入路由參數中的值，您可以在路由參數定義中指定欄位 (`/profile/{id:slug}`)，或者您可以覆寫 Eloquent 模型上的 `getRouteKey` 方法：

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
### 重新導向至控制器動作

您也可以產生重新導向至 [控制器動作](/docs/{{version}}/controllers) 的回應。若要這樣做，請將控制器與動作名稱傳遞給 `action` 方法：

```php
use App\Http\Controllers\UserController;

return redirect()->action([UserController::class, 'index']);
```

如果您的控制器路由需要參數，您可以將它們作為 `action` 方法的第二個引數傳遞：

```php
return redirect()->action(
    [UserController::class, 'profile'], ['id' => 1]
);
```


<a name="redirecting-external-domains"></a>
### 重新導向至外部網域

有時您可能需要重新導向至應用程式以外的網域。您可以透過呼叫 `away` 方法來達成，它會建立一個 `RedirectResponse`，且不會進行任何額外的 URL 編碼、驗證或核對：

```php
return redirect()->away('https://www.google.com');
```


<a name="redirecting-with-flashed-session-data"></a>
### 使用快閃工作階段資料重新導向

重新導向至新 URL 與 [將資料快閃至工作階段](/docs/{{version}}/session#flash-data) 通常是同時進行的。通常在成功執行某個動作後，將成功訊息快閃至工作階段時會這樣做。為了方便起見，您可以使用單一且流暢的方法鏈來建立 `RedirectResponse` 實例並將資料快閃至工作階段：

```php
Route::post('/user/profile', function () {
    // ...

    return redirect('/dashboard')->with('status', 'Profile updated!');
});
```

在使用者被重新導向後，您可以從 [工作階段](/docs/{{version}}/session) 顯示快閃訊息。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
@if (session('status'))
    <div class="alert alert-success">
        {{ session('status') }}
    </div>
@endif
```


<a name="redirecting-with-input"></a>
#### 帶著輸入資料重新導向

您可以使用 `RedirectResponse` 實例提供的 `withInput` 方法，在將使用者重新導向至新位置之前，將目前請求的輸入資料快閃至工作階段。這通常發生在使用者遇到驗證錯誤時。一旦輸入資料被快閃至工作階段，您就可以在下一次請求期間輕鬆地 [擷取它](/docs/{{version}}/requests#retrieving-old-input) 以重新填充表單：

```php
return back()->withInput();
```


<a name="other-response-types"></a>
## 其他回應類型

`response` 輔助函數可用於產生其他類型的回應實例。當在不帶參數的情況下呼叫 `response` 輔助函數時，會回傳一個 `Illuminate\Contracts\Routing\ResponseFactory` [契約(Contracts)](/docs/{{version}}/contracts) 的實作。此契約提供了幾個用於產生回應的實用方法。


<a name="view-responses"></a>
### 視圖回應

如果您需要控制回應的狀態與標頭，但同時也需要回傳 [視圖](/docs/{{version}}/views) 作為回應內容，您應該使用 `view` 方法：

```php
return response()
    ->view('hello', $data, 200)
    ->header('Content-Type', $type);
```

當然，如果您不需要傳遞自定義的 HTTP 狀態碼或自定義標頭，您可以使用全域的 `view` 輔助函數。


<a name="json-responses"></a>
### JSON 回應

`json` 方法會自動將 `Content-Type` 標頭設定為 `application/json`，並使用 PHP 的 `json_encode` 函數將給定的陣列轉換為 JSON：

```php
return response()->json([
    'name' => 'Abigail',
    'state' => 'CA',
]);
```

如果您想建立 JSONP 回應，可以使用 `json` 方法搭配 `withCallback` 方法：

```php
return response()
    ->json(['name' => 'Abigail', 'state' => 'CA'])
    ->withCallback($request->input('callback'));
```


<a name="file-downloads"></a>
### 檔案下載

`download` 方法可用於產生一個強制使用者瀏覽器下載指定路徑檔案的回應。`download` 方法接受一個檔名作為第二個引數，這將決定使用者在下載檔案時看到的檔名。最後，您可以將 HTTP 標頭陣列作為該方法的第三個引數傳遞：

```php
return response()->download($pathToFile);

return response()->download($pathToFile, $name, $headers);
```

> [!WARNING]
> 管理檔案下載的 Symfony HttpFoundation 要求被下載的檔案必須具有 ASCII 檔名。


<a name="file-responses"></a>
### 檔案回應

`file` 方法可用於直接在使用者瀏覽器中顯示檔案（例如圖片或 PDF），而不是啟動下載。此方法接受檔案的絕對路徑作為第一個引數，以及標頭陣列作為第二個引數：

```php
return response()->file($pathToFile);

return response()->file($pathToFile, $headers);
```

<a name="streamed-responses"></a>
## 串流回應

透過在資料生成時將其串流傳輸給用戶端，您可以顯著減少記憶體使用量並提高效能，特別是對於非常大型的回應。串流回應允許用戶端在伺服器完成傳送之前就開始處理資料：

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

為了方便起見，如果您提供給 `stream` 方法的閉包回傳一個 [Generator](https://www.php.net/manual/en/language.generators.overview.php)，Laravel 將會自動在產生器回傳的字串之間清除輸出緩衝區，並禁用 Nginx 的輸出緩衝：

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
### 消費串流回應

您可以透過 Laravel 的 `stream` npm 套件來消費串流回應，該套件提供了一個便捷的 API 用於與 Laravel 的回應和事件串流進行互動。要開始使用，請安裝 `@laravel/stream-react`、`@laravel/stream-vue` 或 `@laravel/stream-svelte` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

```shell tab=Svelte
npm install @laravel/stream-svelte
```

接著，可以使用 `useStream` 來消費事件串流。在提供串流 URL 後，當 Laravel 應用程式回傳內容時，hook 會自動將串接後的回應更新至 `data` 中：

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

當透過 `send` 將資料傳回串流時，在傳送新資料之前會先取消目前的串流連線。所有請求都以 JSON `POST` 請求發送。

> [!WARNING]
> 由於 `useStream` hook 會向您的應用程式發送 `POST` 請求，因此需要有效的 CSRF Token。提供 CSRF Token 最簡單的方法是在應用程式佈局的 head 中 [透過 meta 標籤包含它](/docs/{{version}}/csrf#csrf-x-csrf-token)。

傳遞給 `useStream` 的第二個引數是一個選項物件，您可以用它來自訂串流消費行為。此物件的預設值如下所示：

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

`onResponse` 會在串流成功回傳初始回應後觸發，且原始的 [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) 會被傳遞至回呼函式。`onData` 會在接收到每個區塊 (chunk) 時被呼叫，目前的區塊會被傳遞至回呼函式。`onFinish` 則在串流結束或在擷取/讀取週期中拋出錯誤時被呼叫。

預設情況下，初始化時不會向串流發送請求。您可以使用 `initialInput` 選項向串流傳遞初始負載：

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

若要手動取消串流，您可以使用 hook 回傳的 `cancel` 方法：

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

每次使用 `useStream` hook 時，都會生成一個隨機的 `id` 來識別該串流並於每次請求以標頭 `X-STREAM-ID` 中帶入傳至伺服器。當從多個元件需要使用同一個串流時，您可以自行提供 `id` 來對該串流進行讀寫：

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

如果您需要漸進式地串流 JSON 資料，可以使用 `streamJson` 方法。這個方法對於需要逐步傳送到瀏覽器且能被 JavaScript 輕鬆解析的大型資料集特別有用：

```php
use App\Models\User;

Route::get('/users.json', function () {
    return response()->streamJson([
        'users' => User::cursor(),
    ]);
});
```

`useJsonStream` 鉤子 (hook) 與 [useStream 鉤子](#consuming-streamed-responses) 相同，不同之處在於它會在串流結束後嘗試將資料解析為 JSON：

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

`eventStream` 方法可用於使用 `text/event-stream` 內容類型回傳一個伺服器發送事件 (SSE) 的串流回應。`eventStream` 方法接受一個閉包，該閉包應該在回應可用時將回應 [產出 (yield)](https://www.php.net/manual/en/language.generators.overview.php) 到串流中：

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

如果您想要自訂事件名稱，可以產出一個 `StreamedEvent` 類別的實例：

```php
use Illuminate\Http\StreamedEvent;

yield new StreamedEvent(
    event: 'update',
    data: $response->choices[0],
);
```


<a name="consuming-event-streams"></a>
#### 消費事件串流

事件串流可以使用 Laravel 的 `stream` npm 套件來消費，它提供了一個方便的 API 用於與 Laravel 事件串流互動。要開始使用，請安裝 `@laravel/stream-react`、`@laravel/stream-vue` 或 `@laravel/stream-svelte` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

```shell tab=Svelte
npm install @laravel/stream-svelte
```

接著，可以使用 `useEventStream` 來消費事件串流。在提供串流 URL 後，當 Laravel 應用程式回傳訊息時，該鉤子會自動將拼接後的回應更新到 `message` 中：

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

傳遞給 `useEventStream` 的第二個引數是一個選項物件，您可以用它來自訂串流消費行為。該物件的預設值如下所示：

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

您的應用程式前端也可以透過 [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) 物件手動消費事件串流。`eventStream` 方法會在串流完成時，自動向事件串流發送一個 `</stream>` 更新：

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

若要自訂發送到事件串流的最後一個事件，您可以將 `StreamedEvent` 實例提供給 `eventStream` 方法的 `endStreamWith` 引數：

```php
return response()->eventStream(function () {
    // ...
}, endStreamWith: new StreamedEvent(event: 'update', data: '</stream>'));
```


<a name="streamed-downloads"></a>
### 串流下載

有時候您可能希望將特定操作的字串回應轉換為可下載的回應，而無需將操作內容寫入磁碟。在這種情況下，您可以使用 `streamDownload` 方法。此方法接受回呼函數 (callback)、檔案名稱以及一個可選的標頭陣列作為其引數：

```php
use App\Services\GitHub;

return response()->streamDownload(function () {
    echo GitHub::api('repo')
        ->contents()
        ->readme('laravel', 'laravel')['contents'];
}, 'laravel-readme.md');
```

<a name="response-macros"></a>
## 回應巨集

如果您想要定義一個可以在多個路由和控制器中重複使用的自定義回應，您可以使用 `Response` facade 的 `macro` 方法。通常，您應該在應用程式的其中一個 [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法，例如 `App\Providers\AppServiceProvider` 服務提供者：

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

`macro` 函式接受一個名稱作為第一個引數，以及一個閉包 (closure) 作為第二個引數。當從 `ResponseFactory` 實作或 `response` 輔助函式呼叫該巨集名稱時，該巨集的閉包將會被執行：

```php
return response()->caps('foo');
```
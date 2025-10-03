# HTTP 回應

- [建立回應](#creating-responses)
    - [將 Headers 附加至回應](#attaching-headers-to-responses)
    - [將 Cookies 附加至回應](#attaching-cookies-to-responses)
    - [Cookies 與加密](#cookies-and-encryption)
- [重新導向](#redirects)
    - [重新導向至具名路由](#redirecting-named-routes)
    - [重新導向至 Controller 行動](#redirecting-controller-actions)
    - [重新導向至外部網域](#redirecting-external-domains)
    - [帶有快閃 Session 資料的重新導向](#redirecting-with-flashed-session-data)
- [其他回應類型](#other-response-types)
    - [View 回應](#view-responses)
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

所有路由與控制器都應該回傳一個回應給使用者的瀏覽器。Laravel 提供了幾種不同的方式來回傳回應。最基本的回應是從路由或控制器回傳一個字串。框架會自動將該字串轉換成一個完整的 HTTP 回應：

```php
Route::get('/', function () {
    return 'Hello World';
});
```

除了從您的路由與控制器回傳字串之外，您也可以回傳陣列。框架會自動將該陣列轉換成一個 JSON 回應：

```php
Route::get('/', function () {
    return [1, 2, 3];
});
```

> [!NOTE]
> 您知道您也可以從路由或控制器回傳 [Eloquent Collection](/docs/{{version}}/eloquent-collections) 嗎？它們會自動轉換成 JSON。試試看吧！


<a name="response-objects"></a>
#### Response 物件

通常，您不會只從路由行動回傳簡單的字串或陣列。相反地，您會回傳完整的 `Illuminate\Http\Response` 實例或 [view](/docs/{{version}}/views)。

回傳一個完整的 `Response` 實例讓您可以自訂回應的 HTTP 狀態碼與標頭。`Response` 實例繼承自 `Symfony\Component\HttpFoundation\Response` 類別，該類別提供了多種建構 HTTP 回應的方法：

```php
Route::get('/home', function () {
    return response('Hello World', 200)
        ->header('Content-Type', 'text/plain');
});
```


<a name="eloquent-models-and-collections"></a>
#### Eloquent Model 與 Collection

您也可以直接從路由與控制器回傳 [Eloquent ORM](/docs/{{version}}/eloquent) Model 與 Collection。當您這樣做時，Laravel 會自動將 Model 與 Collection 轉換為 JSON 回應，同時尊重 Model 的[隱藏屬性](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)：

```php
use App\Models\User;

Route::get('/user/{user}', function (User $user) {
    return $user;
});
```


<a name="attaching-headers-to-responses"></a>
### 將 Headers 附加至回應

請記住，大多數的回應方法都是可鏈式呼叫的，這讓您可以流暢地建構回應實例。舉例來說，您可以使用 `header` 方法在將回應傳送回使用者之前，為回應新增一系列的標頭：

```php
return response($content)
    ->header('Content-Type', $type)
    ->header('X-Header-One', 'Header Value')
    ->header('X-Header-Two', 'Header Value');
```

或者，您可以使用 `withHeaders` 方法來指定一個標頭陣列，將其新增至回應中：

```php
return response($content)
    ->withHeaders([
        'Content-Type' => $type,
        'X-Header-One' => 'Header Value',
        'X-Header-Two' => 'Header Value',
    ]);
```


<a name="cache-control-middleware"></a>
#### Cache Control 中介層

Laravel 包含了 `cache.headers` 中介層，可以用來為路由群組快速設定 `Cache-Control` 標頭。指令應該使用對應 cache-control 指令的「蛇形命名法」形式提供，並以分號分隔。如果指令列表中指定了 `etag`，回應內容的 MD5 雜湊值將自動設定為 ETag 識別碼：

```php
Route::middleware('cache.headers:public;max_age=2628000;etag')->group(function () {
    Route::get('/privacy', function () {
        // ...
    });

    Route::get('/terms', function () {
        // ...
    });
});
```


<a name="attaching-cookies-to-responses"></a>
### 將 Cookies 附加至回應

您可以使用 `cookie` 方法將 Cookie 附加到外送的 `Illuminate\Http\Response` 實例。您應該將 Cookie 的名稱、值以及應被視為有效的分鐘數傳遞給此方法：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes
);
```

`cookie` 方法也接受一些不常用到的引數。一般來說，這些引數與 PHP 原生的 [setcookie](https://secure.php.net/manual/en/function.setcookie.php) 方法所給予的引數具有相同的目的與意義：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes, $path, $domain, $secure, $httpOnly
);
```

如果您想確保 Cookie 隨外送回應傳送，但您還沒有該回應的實例，您可以使用 `Cookie` Facade 來「佇列」Cookie，以便在回應傳送時將其附加到回應。`queue` 方法接受建立 Cookie 實例所需的引數。這些 Cookie 將在外送回應傳送至瀏覽器之前附加到回應：

```php
use Illuminate\Support\Facades\Cookie;

Cookie::queue('name', 'value', $minutes);
```


<a name="generating-cookie-instances"></a>
#### 產生 Cookie 實例

如果您想產生一個 `Symfony\Component\HttpFoundation\Cookie` 實例，以便稍後附加到回應實例，您可以使用全域 `cookie` 輔助函式。除非將此 Cookie 附加到回應實例，否則它不會傳送回用戶端：

```php
$cookie = cookie('name', 'value', $minutes);

return response('Hello World')->cookie($cookie);
```


<a name="expiring-cookies-early"></a>
#### 提早讓 Cookie 過期

您可以使用外送回應的 `withoutCookie` 方法，透過讓 Cookie 過期來移除它：

```php
return response('Hello World')->withoutCookie('name');
```

如果您還沒有外送回應的實例，您可以使用 `Cookie` Facade 的 `expire` 方法讓 Cookie 過期：

```php
Cookie::expire('name');
```


<a name="cookies-and-encryption"></a>
### Cookies 與加密

預設情況下，多虧了 `Illuminate\Cookie\Middleware\EncryptCookies` 中介層，Laravel 生成的所有 Cookie 都經過加密與簽章，這樣它們就無法被用戶端修改或讀取。如果您想停用應用程式生成的某些 Cookie 的加密功能，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `encryptCookies` 方法：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->encryptCookies(except: [
        'cookie_name',
    ]);
})
```

<a name="redirects"></a>
## 重新導向

重新導向回應是 `Illuminate\Http\RedirectResponse` 類別的實例，其包含將使用者重新導向至另一個 URL 所需的適當標頭。有幾種方法可以產生 `RedirectResponse` 實例。最簡單的方法是使用全域 `redirect` 輔助函式：

```php
Route::get('/dashboard', function () {
    return redirect('/home/dashboard');
});
```

有時您可能希望將使用者重新導向回先前的位置，例如當提交的表單無效時。您可以透過使用全域 `back` 輔助函式來實現。由於此功能利用 [session](/docs/{{version}}/session)，請確保呼叫 `back` 函式的路由使用了 `web` 中介層群組：

```php
Route::post('/user/profile', function () {
    // Validate the request...

    return back()->withInput();
});
```

<a name="redirecting-named-routes"></a>
### 重新導向至具名路由

當您不帶參數呼叫 `redirect` 輔助函式時，會返回一個 `Illuminate\Routing\Redirector` 實例，允許您在該 `Redirector` 實例上呼叫任何方法。例如，要產生一個重新導向至具名路由的 `RedirectResponse`，您可以使用 `route` 方法：

```php
return redirect()->route('login');
```

如果您的路由有參數，您可以將它們作為第二個參數傳遞給 `route` 方法：

```php
// For a route with the following URI: /profile/{id}

return redirect()->route('profile', ['id' => 1]);
```

<a name="populating-parameters-via-eloquent-models"></a>
#### 透過 Eloquent 模型填入參數

如果您正在重新導向到一個帶有「ID」參數且由 Eloquent 模型填入的路由，您可以直接傳遞該模型本身。ID 將會自動提取：

```php
// For a route with the following URI: /profile/{id}

return redirect()->route('profile', [$user]);
```

如果您想自訂放置在路由參數中的值，您可以在路由參數定義中指定欄位 (`/profile/{id:slug}`)，或者您可以覆寫您的 Eloquent 模型上的 `getRouteKey` 方法：

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
### 重新導向至 Controller 行動

您也可以產生重新導向至 [controller 行動](/docs/{{version}}/controllers)。為此，請將 controller 和行動名稱傳遞給 `action` 方法：

```php
use App\Http\Controllers\UserController;

return redirect()->action([UserController::class, 'index']);
```

如果您的 controller 路由需要參數，您可以將它們作為第二個參數傳遞給 `action` 方法：

```php
return redirect()->action(
    [UserController::class, 'profile'], ['id' => 1]
);
```

<a name="redirecting-external-domains"></a>
### 重新導向至外部網域

有時您可能需要重新導向到您應用程式外部的網域。您可以透過呼叫 `away` 方法來實現，該方法會建立一個 `RedirectResponse`，無需任何額外的 URL 編碼、驗證或確認：

```php
return redirect()->away('https://www.google.com');
```

<a name="redirecting-with-flashed-session-data"></a>
### 帶有快閃 Session 資料的重新導向

重新導向到一個新的 URL 和將資料 [快閃至 session](/docs/{{version}}/session#flash-data) 通常是同時進行的。一般而言，這是在成功執行一個行動後，將成功訊息快閃至 session。為了方便起見，您可以透過單一的流暢方法鏈來建立 `RedirectResponse` 實例並將資料快閃至 session：

```php
Route::post('/user/profile', function () {
    // ...

    return redirect('/dashboard')->with('status', 'Profile updated!');
});
```

使用者被重新導向後，您可以從 [session](/docs/{{version}}/session) 中顯示快閃的訊息。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
@if (session('status'))
    <div class="alert alert-success">
        {{ session('status') }}
    </div>
@endif
```

<a name="redirecting-with-input"></a>
#### 帶有輸入的重新導向

您可以使用 `RedirectResponse` 實例提供的 `withInput` 方法，在將使用者重新導向到新位置之前，將當前請求的輸入資料快閃至 session。這通常是在使用者遇到驗證錯誤時進行的。一旦輸入資料快閃至 session，您便可以在下一個請求中輕鬆 [檢索](/docs/{{version}}/requests#retrieving-old-input) 它，以便重新填充表單：

```php
return back()->withInput();
```

<a name="other-response-types"></a>
## 其他回應類型

`response` 輔助函式可用於產生其他類型的回應實例。當不帶參數呼叫 `response` 輔助函式時，會返回 `Illuminate\Contracts\Routing\ResponseFactory` [契約](/docs/{{version}}/contracts) 的實作。此契約提供了多種有用的方法來產生回應。

<a name="view-responses"></a>
### View 回應

如果您需要控制回應的狀態和標頭，同時也需要返回一個 [view](/docs/{{version}}/views) 作為回應的內容，您應該使用 `view` 方法：

```php
return response()
    ->view('hello', $data, 200)
    ->header('Content-Type', $type);
```

當然，如果您不需要傳遞自訂 HTTP 狀態碼或自訂標頭，您可以使用全域 `view` 輔助函式。

<a name="json-responses"></a>
### JSON 回應

`json` 方法會自動將 `Content-Type` 標頭設定為 `application/json`，並使用 `json_encode` PHP 函式將給定的陣列轉換為 JSON：

```php
return response()->json([
    'name' => 'Abigail',
    'state' => 'CA',
]);
```

如果您想建立 JSONP 回應，您可以將 `json` 方法與 `withCallback` 方法結合使用：

```php
return response()
    ->json(['name' => 'Abigail', 'state' => 'CA'])
    ->withCallback($request->input('callback'));
```

<a name="file-downloads"></a>
### 檔案下載

`download` 方法可用於產生一個回應，強制使用者瀏覽器下載給定路徑的檔案。`download` 方法接受一個檔名作為方法的第二個參數，該參數將決定使用者下載時看到的檔名。最後，您可以將 HTTP 標頭陣列作為方法的第三個參數傳遞：

```php
return response()->download($pathToFile);

return response()->download($pathToFile, $name, $headers);
```

> [!WARNING]
> 管理檔案下載的 Symfony HttpFoundation 要求下載的檔案必須具有 ASCII 檔名。

<a name="file-responses"></a>
### 檔案回應

`file` 方法可用於直接在使用者瀏覽器中顯示檔案，例如圖片或 PDF，而不是啟動下載。此方法接受檔案的絕對路徑作為第一個參數，並接受一個標頭陣列作為第二個參數：

```php
return response()->file($pathToFile);

return response()->file($pathToFile, $headers);
```

<a name="streamed-responses"></a>
## 串流回應

透過在產生資料時將其串流至用戶端，您可以顯著減少記憶體使用量並提高效能，特別是對於非常大的回應。串流回應允許用戶端在伺服器完成傳送資料之前就開始處理資料：

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

為了方便，如果您提供給 `stream` 方法的閉包回傳一個 [Generator](https://www.php.net/manual/en/language.generators.overview.php)，Laravel 會自動在產生器回傳的字串之間刷新輸出緩衝區，並停用 Nginx 輸出緩衝。

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

可以使用 Laravel 的 `stream` npm 套件來消費串流回應，該套件提供了一個方便的 API，用於與 Laravel 回應和事件串流進行互動。要開始使用，請安裝 `@laravel/stream-react` 或 `@laravel/stream-vue` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

然後，可以使用 `useStream` 來消費事件串流。在提供串流 URL 後，當內容從您的 Laravel 應用程式回傳時，hook 會自動使用串接後的回應來更新 `data`：

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

透過 `send` 將資料回傳至串流時，會在傳送新資料之前取消與串流的活躍連線。所有請求都以 JSON `POST` 請求傳送。

> [!WARNING]
> 由於 `useStream` hook 向您的應用程式發出 `POST` 請求，因此需要有效的 CSRF token。提供 CSRF token 最簡單的方法是透過 [在應用程式佈局的 head 中包含一個 meta 標籤](/docs/{{version}}/csrf#csrf-x-csrf-token) 來實現。

傳遞給 `useStream` 的第二個參數是一個選項物件，您可以使用它來自訂串流的消費行為。此物件的預設值如下所示：

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

`onResponse` 在從串流成功接收到初始回應後被觸發，原始的 [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) 會傳遞給回呼。當接收到每個區塊時，會呼叫 `onData`，當前區塊會傳遞給回呼。當串流完成且在 fetch / read 循環期間拋出錯誤時，會呼叫 `onFinish`。

預設情況下，初始化時不會對串流發出請求。您可以使用 `initialInput` 選項將初始負載傳遞給串流：

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

要手動取消串流，您可以使用 hook 回傳的 `cancel` 方法：

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

每次使用 `useStream` hook 時，都會產生一個隨機的 `id` 來識別串流。這會隨每個請求以 `X-STREAM-ID` header 的形式傳回伺服器。當從多個組件消費相同的串流時，您可以透過提供自己的 `id` 來讀取和寫入串流：

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

<a name="streamed-json-responses"></a>
### 串流 JSON 回應

如果您需要逐步串流 JSON 資料，可以使用 `streamJson` 方法。此方法對於需要逐步發送到瀏覽器並以易於 JavaScript 解析的格式呈現的大型資料集特別有用：

```php
use App\Models\User;

Route::get('/users.json', function () {
    return response()->streamJson([
        'users' => User::cursor(),
    ]);
});
```

`useJsonStream` hook 與 [useStream hook](#consuming-streamed-responses) 相同，除了它會在串流完成後嘗試將資料解析為 JSON：

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

<a name="event-streams"></a>
### 事件串流 (SSE)

`eventStream` 方法可用來回傳使用 `text/event-stream` 內容類型的伺服器傳送事件 (SSE) 串流回應。`eventStream` 方法接受一個閉包，該閉包應在回應可用時將其 [產出](https://www.php.net/manual/en/language.generators.overview.php) 至串流：

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

如果您想自訂事件名稱，可以產出一個 `StreamedEvent` 類別的實例：

```php
use Illuminate\Http\StreamedEvent;

yield new StreamedEvent(
    event: 'update',
    data: $response->choices[0],
);
```

<a name="consuming-event-streams"></a>
#### 消費事件串流

事件串流可以使用 Laravel 的 `stream` npm 套件來消費，該套件提供了一個方便的 API 來與 Laravel 事件串流互動。要開始使用，請安裝 `@laravel/stream-react` 或 `@laravel/stream-vue` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

然後，可以使用 `useEventStream` 來消費事件串流。在提供串流 URL 後，hook 將在從 Laravel 應用程式回傳訊息時自動將 `message` 與串接的回應進行更新：

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

傳遞給 `useEventStream` 的第二個引數是一個選項物件，您可以使用它來自訂串流消費行為。此物件的預設值如下所示：

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

事件串流也可以透過應用程式的前端使用 [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) 物件手動消費。當串流完成時，`eventStream` 方法會自動向事件串流傳送 `</stream>` 更新：

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

要自訂傳送到事件串流的最終事件，您可以向 `eventStream` 方法的 `endStreamWith` 引數提供一個 `StreamedEvent` 實例：

```php
return response()->eventStream(function () {
    // ...
}, endStreamWith: new StreamedEvent(event: 'update', data: '</stream>'));
```

<a name="streamed-downloads"></a>
### 串流下載

有時，您可能希望將某個操作的字串回應轉換為可下載的回應，而無需將操作內容寫入磁碟。在這種情況下，您可以使用 `streamDownload` 方法。此方法接受一個回呼、檔案名稱和一個可選的 headers 陣列作為其引數：

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

如果您希望定義一個可在多個路由與控制器中重複使用的自訂回應，您可以使用 `Response` Facade 上的 `macro` 方法。通常，您應該在應用程式其中一個 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法，例如 `App\Providers\AppServiceProvider` 服務提供者：

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

`macro` 函式接受名稱作為其第一個引數，並接受閉包作為其第二個引數。當從 `ResponseFactory` 實作或 `response` 輔助函式呼叫此巨集名稱時，此巨集的閉包將會執行：

```php
return response()->caps('foo');
```
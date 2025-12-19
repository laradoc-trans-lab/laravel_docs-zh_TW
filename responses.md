# HTTP 回應

- [建立回應](#creating-responses)
    - [在回應附加標頭](#attaching-headers-to-responses)
    - [在回應附加 Cookie](#attaching-cookies-to-responses)
    - [Cookie 與加密](#cookies-and-encryption)
- [重導向](#redirects)
    - [重導向至具名路由](#redirecting-named-routes)
    - [重導向至控制器動作](#redirecting-controller-actions)
    - [重導向至外部網域](#redirecting-external-domains)
    - [帶著 Session 暫存資料重導向](#redirecting-with-flashed-session-data)
- [其他回應型別](#other-response-types)
    - [View 回應](#view-responses)
    - [JSON 回應](#json-responses)
    - [檔案下載](#file-downloads)
    - [檔案回應](#file-responses)
- [串流回應](#streamed-responses)
    - [使用串流回應](#consuming-streamed-responses)
    - [串流 JSON 回應](#streamed-json-responses)
    - [事件串流 (SSE)](#event-streams)
    - [串流下載](#streamed-downloads)
- [回應巨集](#response-macros)

<a name="creating-responses"></a>
## 建立回應


<a name="strings-arrays"></a>
#### 字串與陣列

所有的路由與控制器都應該回傳一個回應給使用者的瀏覽器。Laravel 提供多種不同的方式來回傳回應。最基本的回應是從路由或控制器回傳一個字串。框架會自動將字串轉換成完整的 HTTP 回應：

```php
Route::get('/', function () {
    return 'Hello World';
});
```

除了從路由和控制器回傳字串外，你也可以回傳陣列。框架會自動將陣列轉換成 JSON 回應：

```php
Route::get('/', function () {
    return [1, 2, 3];
});
```

> [!NOTE]
> 你知道你也可以從路由或控制器回傳 [Eloquent collections](/docs/{{version}}/eloquent-collections) 嗎？它們會被自動轉換成 JSON。試試看吧！


<a name="response-objects"></a>
#### 回應物件

通常，你不會只從路由動作回傳簡單的字串或陣列。相反地，你會回傳完整的 `Illuminate\Http\Response` 實例或 [View](/docs/{{version}}/views)。

回傳完整的 `Response` 實例讓你能夠自定義回應的 HTTP 狀態碼與標頭。`Response` 實例繼承自 `Symfony\Component\HttpFoundation\Response` 類別，該類別提供了多種建構 HTTP 回應的方法：

```php
Route::get('/home', function () {
    return response('Hello World', 200)
        ->header('Content-Type', 'text/plain');
});
```


<a name="eloquent-models-and-collections"></a>
#### Eloquent 模型與集合

你也可以直接從路由與控制器回傳 [Eloquent ORM](/docs/{{version}}/eloquent) 模型與集合。當你這麼做時，Laravel 會自動將模型與集合轉換為 JSON 回應，同時會遵循模型的 [隱藏屬性](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)：

```php
use App\Models\User;

Route::get('/user/{user}', function (User $user) {
    return $user;
});
```


<a name="attaching-headers-to-responses"></a>
### 在回應附加標頭

請記住，大多數的回應方法都是可以鏈結的，這讓你能夠流暢地建構回應實例。例如，你可以使用 `header` 方法在將回應傳回給使用者之前，為其添加一系列的標頭：

```php
return response($content)
    ->header('Content-Type', $type)
    ->header('X-Header-One', 'Header Value')
    ->header('X-Header-Two', 'Header Value');
```

或者，你也可以使用 `withHeaders` 方法來指定要添加到回應的標頭陣列：

```php
return response($content)
    ->withHeaders([
        'Content-Type' => $type,
        'X-Header-One' => 'Header Value',
        'X-Header-Two' => 'Header Value',
    ]);
```


<a name="cache-control-middleware"></a>
#### 快取控制中介層

Laravel 包含一個 `cache.headers` 中介層，可用於快速地為一組路由設定 `Cache-Control` 標頭。指令應使用對應的快取控制指令的「蛇底式 (snake case)」等效項，並以分號分隔。如果在指令列表中指定了 `etag`，則回應內容的 MD5 雜湊值將自動被設定為 ETag 識別碼：

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
### 在回應附加 Cookie

你可以使用 `cookie` 方法將 Cookie 附加到輸出的 `Illuminate\Http\Response` 實例。你應該將名稱、值和 Cookie 被視為有效的分鐘數傳遞給此方法：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes
);
```

`cookie` 方法還接受一些較少使用的參數。通常，這些參數的目的和意義與 PHP 原生的 [setcookie](https://secure.php.net/manual/en/function.setcookie.php) 方法的參數相同：

```php
return response('Hello World')->cookie(
    'name', 'value', $minutes, $path, $domain, $secure, $httpOnly
);
```

如果你想確保 Cookie 隨輸出回應一起發送，但你目前還沒有該回應的實例，你可以使用 `Cookie` Facade 將 Cookie 「加入佇列 (queue)」，以便在發送回應時附加。`queue` 方法接受建立 Cookie 實例所需的參數。這些 Cookie 將在傳送到瀏覽器之前被附加到輸出的回應中：

```php
use Illuminate\Support\Facades\Cookie;

Cookie::queue('name', 'value', $minutes);
```


<a name="generating-cookie-instances"></a>
#### 產生 Cookie 實例

如果你想要產生一個可以在稍後附加到回應實例的 `Symfony\Component\HttpFoundation\Cookie` 實例，你可以使用全域的 `cookie` 輔助函式。除非將此 Cookie 附加到回應實例，否則它不會被傳回給客戶端：

```php
$cookie = cookie('name', 'value', $minutes);

return response('Hello World')->cookie($cookie);
```


<a name="expiring-cookies-early"></a>
#### 提前使 Cookie 過期

你可以透過輸出回應的 `withoutCookie` 方法，藉由讓 Cookie 過期來移除它：

```php
return response('Hello World')->withoutCookie('name');
```

如果你還沒有輸出回應的實例，你可以使用 `Cookie` Facade 的 `expire` 方法來使 Cookie 過期：

```php
Cookie::expire('name');
```


<a name="cookies-and-encryption"></a>
### Cookie 與加密

預設情況下，感謝 `Illuminate\Cookie\Middleware\EncryptCookies` 中介層，Laravel 產生的所有 Cookie 都經過加密和簽名，因此客戶端無法修改或讀取它們。如果你想為應用程式產生的部分 Cookie 禁用加密，可以在應用程式的 `bootstrap/app.php` 檔案中使用 `encryptCookies` 方法：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->encryptCookies(except: [
        'cookie_name',
    ]);
})
```

> [!NOTE]
> 一般而言，絕不應該禁用 Cookie 加密，因為這會使你的 Cookie 暴露於潛在的客戶端資料外洩和篡改風險中。

<a name="redirects"></a>
## 重導向

重導向回應是 `Illuminate\Http\RedirectResponse` 類別的執行個體，包含將使用者重導向至另一個 URL 所需的正確標頭。產生 `RedirectResponse` 執行個體有多種方法。最簡單的方法是使用全域的 `redirect` 輔助函式：

```php
Route::get('/dashboard', function () {
    return redirect('/home/dashboard');
});
```

有時您可能希望將使用者重導向至其先前的位置，例如當提交的表單無效時。您可以使用全域的 `back` 輔助函式來達成。由於此功能使用了 [Session](/docs/{{version}}/session)，請確保呼叫 `back` 函式的路由使用了 `web` 中介層群組：

```php
Route::post('/user/profile', function () {
    // Validate the request...

    return back()->withInput();
});
```


<a name="redirecting-named-routes"></a>
### 重導向至具名路由

當您在不帶參數的情況下呼叫 `redirect` 輔助函式時，會回傳 `Illuminate\Routing\Redirector` 的執行個體，讓您可以呼叫 `Redirector` 執行個體上的任何方法。例如，要產生至具名路由的 `RedirectResponse`，您可以使用 `route` 方法：

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

如果您要重導向至一個具有由 Eloquent 模型填入「ID」參數的路由，您可以直接傳遞模型本身。ID 將會自動被提取：

```php
// For a route with the following URI: /profile/{id}

return redirect()->route('profile', [$user]);
```

如果您想自定義放在路由參數中的值，可以在路由參數定義中指定欄位 (`/profile/{id:slug}`)，或者您可以覆寫 Eloquent 模型上的 `getRouteKey` 方法：

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
### 重導向至控制器動作

您也可以產生至[控制器動作](/docs/{{version}}/controllers)的重導向。為此，請將控制器和動作名稱傳遞給 `action` 方法：

```php
use App\Http\Controllers\UserController;

return redirect()->action([UserController::class, 'index']);
```

如果您的控制器路由需要參數，您可以將它們作為第二個參數傳遞給 `action` 方法：

```php
return redirect()->action(
    [UserController::class, 'profile'], ['id' => 1]
);
```


<a name="redirecting-external-domains"></a>
### 重導向至外部網域

有時您可能需要重導向至應用程式之外的網域。您可以透過呼叫 `away` 方法來達成，該方法會建立一個 `RedirectResponse`，且不進行任何額外的 URL 編碼、驗證或驗證：

```php
return redirect()->away('https://www.google.com');
```


<a name="redirecting-with-flashed-session-data"></a>
### 帶著 Session 暫存資料重導向

重導向至新的 URL 並將[資料暫存 (Flash) 到 Session](/docs/{{version}}/session#flash-data) 通常是同時進行的。一般是在成功執行某個動作後，將成功訊息暫存到 Session 中。為了方便起見，您可以建立一個 `RedirectResponse` 執行個體，並在單個流暢的方法鏈中將資料暫存到 Session：

```php
Route::post('/user/profile', function () {
    // ...

    return redirect('/dashboard')->with('status', 'Profile updated!');
});
```

在使用者被重導向後，您可以顯示來自 [Session](/docs/{{version}}/session) 的暫存訊息。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
@if (session('status'))
    <div class="alert alert-success">
        {{ session('status') }}
    </div>
@endif
```


<a name="redirecting-with-input"></a>
#### 帶著輸入資料重導向

您可以使用 `RedirectResponse` 執行個體提供的 `withInput` 方法，在將使用者重導向至新位置之前，將當前請求的輸入資料暫存到 Session 中。這通常在使用者遇到驗證錯誤時進行。一旦輸入資料被暫存到 Session，您就可以在下一次請求期間輕鬆地[檢索它](/docs/{{version}}/requests#retrieving-old-input)以重新填寫表單：

```php
return back()->withInput();
```


<a name="other-response-types"></a>
## 其他回應型別

`response` 輔助函式可用於產生其他型別的回應執行個體。當不帶參數呼叫 `response` 輔助函式時，將回傳 `Illuminate\Contracts\Routing\ResponseFactory` [契约 (Contract)](/docs/{{version}}/contracts) 的實作。此契约提供了幾個用於產生回應的實用方法。


<a name="view-responses"></a>
### View 回應

如果您需要控制回應的狀態和標頭，但也需要回傳一個 [View](/docs/{{version}}/views) 作為回應內容，則應使用 `view` 方法：

```php
return response()
    ->view('hello', $data, 200)
    ->header('Content-Type', $type);
```

當然，如果您不需要傳遞自定義的 HTTP 狀態碼或標頭，可以使用全域的 `view` 輔助函式。


<a name="json-responses"></a>
### JSON 回應

`json` 方法會自動將 `Content-Type` 標頭設定為 `application/json`，並使用 PHP 的 `json_encode` 函式將指定的陣列轉換為 JSON：

```php
return response()->json([
    'name' => 'Abigail',
    'state' => 'CA',
]);
```

如果您想建立 JSONP 回應，可以結合使用 `json` 方法與 `withCallback` 方法：

```php
return response()
    ->json(['name' => 'Abigail', 'state' => 'CA'])
    ->withCallback($request->input('callback'));
```


<a name="file-downloads"></a>
### 檔案下載

`download` 方法可用於產生強制使用者瀏覽器下載指定路徑檔案的回應。`download` 方法接受檔名作為第二個參數，這將決定下載檔案的使用者所看到的檔名。最後，您可以將 HTTP 標頭陣列作為第三個參數傳遞給該方法：

```php
return response()->download($pathToFile);

return response()->download($pathToFile, $name, $headers);
```

> [!WARNING]
> 管理檔案下載的 Symfony HttpFoundation 要求下載的檔案必須具備 ASCII 檔名。


<a name="file-responses"></a>
### 檔案回應

`file` 方法可用於直接在使用者瀏覽器中顯示檔案（例如圖片或 PDF），而不是啟動下載。此方法接受檔案的絕對路徑作為其第一個參數，並接受標頭陣列作為其第二個參數：

```php
return response()->file($pathToFile);

return response()->file($pathToFile, $headers);
```

<a name="streamed-responses"></a>
## 串流回應

透過在生成資料時將其串流傳輸到客戶端，您可以顯著減少記憶體使用量並提高效能，特別是對於非常大型的回應。串流回應允許客戶端在伺服器完成發送之前就開始處理資料：

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

為了方便起見，如果您提供給 `stream` 方法的閉包回傳一個 [Generator](https://www.php.net/manual/en/language.generators.overview.php)，Laravel 將自動在產生器回傳的字串之間清除 (Flush) 輸出緩衝區，並停用 Nginx 的輸出緩衝：

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
### 使用串流回應

可以使用 Laravel 的 `stream` npm 套件來使用串流回應，該套件提供了一個方便的 API 來與 Laravel 回應和事件串流進行互動。要開始使用，請安裝 `@laravel/stream-react` 或 `@laravel/stream-vue` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

接著，可以使用 `useStream` 來接收事件串流。在提供您的串流 URL 後，該 Hook 將在您的 Laravel 應用程式回傳內容時，自動以串接後的回應更新 `data`：

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

當透過 `send` 將資料傳回串流時，目前的串流連線會在發送新資料之前被取消。所有請求都以 JSON `POST` 請求發送。

> [!WARNING]
> 由於 `useStream` Hook 會對您的應用程式發送 `POST` 請求，因此需要一個有效的 CSRF 權杖。提供 CSRF 權杖最簡單的方法是[透過 meta 標籤將其包含在應用程式佈局的 head 中](/docs/{{version}}/csrf#csrf-x-csrf-token)。

提供給 `useStream` 的第二個參數是一個選項物件，您可以用它來自定義串流的使用行為。該物件的預設值如下所示：

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

`onResponse` 在來自串流的初始回應成功後觸發，並將原始的 [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) 傳遞給回呼。`onData` 在接收到每個區塊 (Chunk) 時被呼叫 - 目前的區塊會被傳遞給回呼。`onFinish` 在串流完成以及在提取/讀取週期中拋出錯誤時被呼叫。

預設情況下，初始化時不會向串流發送請求。您可以透過使用 `initialInput` 選項向串流發送初始負載：

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

要手動取消串流，您可以使用該 Hook 回傳的 `cancel` 方法：

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

每次使用 `useStream` Hook 時，都會生成一個隨機的 `id` 來識別串流。這會隨每個請求在 `X-STREAM-ID` 標頭中傳回伺服器。當從多個元件使用同一個串流時，您可以透過提供自己的 `id` 來讀取和寫入該串流：

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

如果您需要以增量方式串流 JSON 資料，您可以使用 `streamJson` 方法。此方法對於需要以 JavaScript 易於解析的格式漸進式發送至瀏覽器的大型資料集特別有用：

```php
use App\Models\User;

Route::get('/users.json', function () {
    return response()->streamJson([
        'users' => User::cursor(),
    ]);
});
```

`useJsonStream` Hook 與 [useStream hook](#consuming-streamed-responses) 相同，不同之處在於它會在串流結束後嘗試將資料解析為 JSON：

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

`eventStream` 方法可用於使用 `text/event-stream` 內容類型傳回伺服器發送事件 (SSE) 串流回應。`eventStream` 方法接受一個閉包，該閉包應在回應可用時將回應 [yield](https://www.php.net/manual/en/language.generators.overview.php) 到串流中：

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

如果您想自訂事件的名稱，您可以產生一個 `StreamedEvent` 類別的執行個體：

```php
use Illuminate\Http\StreamedEvent;

yield new StreamedEvent(
    event: 'update',
    data: $response->choices[0],
);
```


<a name="consuming-event-streams"></a>
#### 使用事件串流

可以使用 Laravel 的 `stream` npm 套件來取用事件串流，該套件提供了一個方便的 API 來與 Laravel 事件串流進行互動。要開始使用，請安裝 `@laravel/stream-react` 或 `@laravel/stream-vue` 套件：

```shell tab=React
npm install @laravel/stream-react
```

```shell tab=Vue
npm install @laravel/stream-vue
```

接著，可以使用 `useEventStream` 來取用事件串流。在提供您的串流 URL 後，當訊息從您的 Laravel 應用程式傳回時，該 Hook 將自動使用串接後的回應來更新 `message`：

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

傳遞給 `useEventStream` 的第二個參數是一個選項物件，您可以用它來定義取用串流的行為。此物件的預設值如下所示：

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

事件串流也可以透過應用程式前端的 [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) 物件手動取用。當串流完成時，`eventStream` 方法將自動向事件串流發送 `</stream>` 更新：

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

若要自訂發送到事件串流的最後一個事件，您可以為 `eventStream` 方法的 `endStreamWith` 參數提供一個 `StreamedEvent` 執行個體：

```php
return response()->eventStream(function () {
    // ...
}, endStreamWith: new StreamedEvent(event: 'update', data: '</stream>'));
```


<a name="streamed-downloads"></a>
### 串流下載

有時您可能希望將指定操作的字串回應轉換為可下載的回應，而無需將操作內容寫入磁碟。在這種情況下，您可以使用 `streamDownload` 方法。此方法接受回呼函式、檔名以及一個選用的標頭陣列作為其參數：

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

如果您想要定義一個可以在各種路由與控制器中重複使用的自定義回應，您可以使用 `Response` Facade 上的 `macro` 方法。通常，您應該在應用程式的其中一個 [服務提供者](/docs/{{version}}/providers)（例如 `App\Providers\AppServiceProvider` 服務提供者）的 `boot` 方法中呼叫此方法：

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

`macro` 函式接受一個名稱作為其第一個參數，以及一個閉包作為其第二個參數。當從 `ResponseFactory` 實作或 `response` 輔助函式呼叫巨集名稱時，將會執行該巨集的閉包：

```php
return response()->caps('foo');
```
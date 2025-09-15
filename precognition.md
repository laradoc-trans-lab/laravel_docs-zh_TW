# Precognition

- [簡介](#introduction)
- [即時驗證](#live-validation)
    - [使用 Vue](#using-vue)
    - [使用 Vue 與 Inertia](#using-vue-and-inertia)
    - [使用 React](#using-react)
    - [使用 React 與 Inertia](#using-react-and-inertia)
    - [使用 Alpine 與 Blade](#using-alpine)
    - [設定 Axios](#configuring-axios)
- [自訂驗證規則](#customizing-validation-rules)
- [處理檔案上傳](#handling-file-uploads)
- [管理副作用](#managing-side-effects)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

Laravel Precognition 讓您能夠預測未來 HTTP 請求的結果。Precognition 的主要用途之一是為您的前端 JavaScript 應用程式提供「即時」驗證，而無需重複應用程式的後端驗證規則。Precognition 與 Laravel 基於 Inertia 的 [入門套件](/docs/{{version}}/starter-kits) 搭配使用效果尤其好。

當 Laravel 收到「預知請求 (precognitive request)」時，它將執行所有路由的 Middleware 並解析路由的 Controller 依賴項，包括驗證 [表單請求](/docs/{{version}}/validation#form-request-validation) — 但它不會實際執行路由的 Controller 方法。

<a name="live-validation"></a>
## 即時驗證

<a name="using-vue"></a>
### 使用 Vue

使用 Laravel Precognition，您可以為使用者提供即時驗證體驗，而無需在前端 Vue 應用程式中重複您的驗證規則。為了說明其運作方式，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` Middleware 加入到路由定義中。您還應該建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 來存放路由的驗證規則：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下來，您應該透過 NPM 安裝 Laravel Precognition 的 Vue 前端輔助套件：

```shell
npm install laravel-precognition-vue
```

安裝 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `useForm` 函式建立一個表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 和初始表單資料。

然後，要啟用即時驗證，請在每個輸入的 `change` 事件上呼叫表單的 `validate` 方法，並提供輸入的名稱：

```vue
<script setup>
import { useForm } from 'laravel-precognition-vue';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = () => form.submit();
</script>

<template>
    <form @submit.prevent="submit">
        <label for="name">Name</label>
        <input
            id="name"
            v-model="form.name"
            @change="form.validate('name')"
        />
        <div v-if="form.invalid('name')">
            {{ form.errors.name }}
        </div>

        <label for="email">Email</label>
        <input
            id="email"
            type="email"
            v-model="form.email"
            @change="form.validate('email')"
        />
        <div v-if="form.invalid('email')">
            {{ form.errors.email }}
        </div>

        <button :disabled="form.processing">
            Create User
        </button>
    </form>
</template>
```

現在，當使用者填寫表單時，Precognition 將提供由路由表單請求中的驗證規則驅動的即時驗證輸出。當表單的輸入發生變化時，將向您的 Laravel 應用程式發送一個去抖動的「預知」驗證請求。您可以透過呼叫表單的 `setValidationTimeout` 函式來設定去抖動逾時：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在進行中時，表單的 `validating` 屬性將為 `true`：

```html
<div v-if="form.validating">
    Validating...
</div>
```

在驗證請求或表單提交期間返回的任何驗證錯誤將自動填充表單的 `errors` 物件：

```html
<div v-if="form.invalid('email')">
    {{ form.errors.email }}
</div>
```

您可以使用表單的 `hasErrors` 屬性來判斷表單是否有任何錯誤：

```html
<div v-if="form.hasErrors">
    <!-- ... -->
</div>
```

您也可以透過將輸入的名稱傳遞給表單的 `valid` 和 `invalid` 函式，分別判斷輸入是否通過或未通過驗證：

```html
<span v-if="form.valid('email')">
    ✅
</span>

<span v-else-if="form.invalid('email')">
    ❌
</span>
```

> [!WARNING]  
> 表單輸入只有在更改並收到驗證回應後才會顯示為有效或無效。

如果您正在使用 Precognition 驗證表單輸入的子集，手動清除錯誤會很有用。您可以使用表單的 `forgetError` 函式來實現此目的：

```html
<input
    id="avatar"
    type="file"
    @change="(e) => {
        form.avatar = e.target.files[0]

        form.forgetError('avatar')
    }"
>
```

正如我們所見，您可以掛接到輸入的 `change` 事件並在使用者與其互動時驗證個別輸入；但是，您可能需要驗證使用者尚未互動的輸入。這在建立「精靈 (wizard)」時很常見，您希望在進入下一步之前驗證所有可見的輸入，無論使用者是否與其互動過。

要使用 Precognition 執行此操作，您應該呼叫 `validate` 方法，並將您希望驗證的欄位名稱傳遞給 `only` 設定鍵。您可以使用 `onSuccess` 或 `onValidationError` 回呼來處理驗證結果：

```html
<button
    type="button" 
    @click="form.validate({
        only: ['name', 'email', 'phone'],
        onSuccess: (response) => nextStep(),
        onValidationError: (response) => /* ... */,
    })"
>Next Step</button>
```

當然，您也可以根據表單提交的回應執行程式碼。表單的 `submit` 函式會返回一個 Axios 請求 Promise。這提供了一種方便的方式來存取回應酬載、在成功提交後重設表單輸入，或處理失敗的請求：

```js
const submit = () => form.submit()
    .then(response => {
        form.reset();

        alert('User created.');
    })
    .catch(error => {
        alert('An error occurred.');
    });
```

您可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在進行中：

```html
<button :disabled="form.processing">
    Submit
</button>
```

<a name="using-vue-and-inertia"></a>
### 使用 Vue 與 Inertia

> [!NOTE]  
> 如果您想在開發 Laravel 應用程式時使用 Vue 和 Inertia 搶先一步，請考慮使用我們的 [入門套件](/docs/{{version}}/starter-kits) 之一。Laravel 的入門套件為您的新 Laravel 應用程式提供了後端和前端身份驗證鷹架。

在使用 Precognition 與 Vue 和 Inertia 之前，請務必查閱我們關於 [使用 Precognition 與 Vue](#using-vue) 的一般文件。當使用 Vue 與 Inertia 時，您需要透過 NPM 安裝與 Inertia 相容的 Precognition 函式庫：

```shell
npm install laravel-precognition-vue-inertia
```

安裝後，Precognition 的 `useForm` 函式將返回一個 Inertia [表單輔助函式](https://inertiajs.com/forms#form-helper)，並增強了上述驗證功能。

表單輔助函式的 `submit` 方法已簡化，無需指定 HTTP 方法或 URL。相反，您可以將 Inertia 的 [訪問選項](https://inertiajs.com/manual-visits) 作為第一個也是唯一一個參數傳遞。此外，`submit` 方法不會像上面 Vue 範例中那樣返回 Promise。相反，您可以在提供給 `submit` 方法的訪問選項中提供 Inertia 支援的任何 [事件回呼](https://inertiajs.com/manual-visits#event-callbacks)：

```vue
<script setup>
import { useForm } from 'laravel-precognition-vue-inertia';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = () => form.submit({
    preserveScroll: true,
    onSuccess: () => form.reset(),
});
</script>
```

<a name="using-react"></a>
### 使用 React

使用 Laravel Precognition，您可以為使用者提供即時驗證體驗，而無需在前端 React 應用程式中重複您的驗證規則。為了說明其運作方式，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` Middleware 加入到路由定義中。您還應該建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 來存放路由的驗證規則：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下來，您應該透過 NPM 安裝 Laravel Precognition 的 React 前端輔助套件：

```shell
npm install laravel-precognition-react
```

安裝 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `useForm` 函式建立一個表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 和初始表單資料。

要啟用即時驗證，您應該監聽每個輸入的 `change` 和 `blur` 事件。在 `change` 事件處理函式中，您應該使用 `setData` 函式設定表單的資料，傳遞輸入的名稱和新值。然後，在 `blur` 事件處理函式中呼叫表單的 `validate` 方法，並提供輸入的名稱：

```jsx
import { useForm } from 'laravel-precognition-react';

export default function Form() {
    const form = useForm('post', '/users', {
        name: '',
        email: '',
    });

    const submit = (e) => {
        e.preventDefault();

        form.submit();
    };

    return (
        <form onSubmit={submit}>
            <label htmlFor="name">Name</label>
            <input
                id="name"
                value={form.data.name}
                onChange={(e) => form.setData('name', e.target.value)}
                onBlur={() => form.validate('name')}
            />
            {form.invalid('name') && <div>{form.errors.name}</div>}

            <label htmlFor="email">Email</label>
            <input
                id="email"
                value={form.data.email}
                onChange={(e) => form.setData('email', e.target.value)}
                onBlur={() => form.validate('email')}
            />
            {form.invalid('email') && <div>{form.errors.email}</div>}

            <button disabled={form.processing}>
                Create User
            </button>
        </form>
    );
};
```

現在，當使用者填寫表單時，Precognition 將提供由路由表單請求中的驗證規則驅動的即時驗證輸出。當表單的輸入發生變化時，將向您的 Laravel 應用程式發送一個去抖動的「預知」驗證請求。您可以透過呼叫表單的 `setValidationTimeout` 函式來設定去抖動逾時：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在進行中時，表單的 `validating` 屬性將為 `true`：

```jsx
{form.validating && <div>Validating...</div>}
```

在驗證請求或表單提交期間返回的任何驗證錯誤將自動填充表單的 `errors` 物件：

```jsx
{form.invalid('email') && <div>{form.errors.email}</div>}
```

您可以使用表單的 `hasErrors` 屬性來判斷表單是否有任何錯誤：

```jsx
{form.hasErrors && <div><!-- ... --></div>}
```

您也可以透過將輸入的名稱傳遞給表單的 `valid` 和 `invalid` 函式，分別判斷輸入是否通過或未通過驗證：

```jsx
{form.valid('email') && <span>✅</span>}

{form.invalid('email') && <span>❌</span>}
```

> [!WARNING]  
> 表單輸入只有在更改並收到驗證回應後才會顯示為有效或無效。

如果您正在使用 Precognition 驗證表單輸入的子集，手動清除錯誤會很有用。您可以使用表單的 `forgetError` 函式來實現此目的：

```jsx
<input
    id="avatar"
    type="file"
    onChange={(e) => {
        form.setData('avatar', e.target.value);

        form.forgetError('avatar');
    }}
>
```

正如我們所見，您可以掛接到輸入的 `blur` 事件並在使用者與其互動時驗證個別輸入；但是，您可能需要驗證使用者尚未互動的輸入。這在建立「精靈 (wizard)」時很常見，您希望在進入下一步之前驗證所有可見的輸入，無論使用者是否與其互動過。

要使用 Precognition 執行此操作，您應該呼叫 `validate` 方法，並將您希望驗證的欄位名稱傳遞給 `only` 設定鍵。您可以使用 `onSuccess` 或 `onValidationError` 回呼來處理驗證結果：

```jsx
<button
    type="button"
    onClick={() => form.validate({
        only: ['name', 'email', 'phone'],
        onSuccess: (response) => nextStep(),
        onValidationError: (response) => /* ... */,
    })}
>Next Step</button>
```

當然，您也可以根據表單提交的回應執行程式碼。表單的 `submit` 函式會返回一個 Axios 請求 Promise。這提供了一種方便的方式來存取回應酬載、在成功提交後重設表單輸入，或處理失敗的請求：

```js
const submit = (e) => {
    e.preventDefault();

    form.submit()
        .then(response => {
            form.reset();

            alert('User created.');
        })
        .catch(error => {
            alert('An error occurred.');
        });
};
```

您可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在進行中：

```html
<button disabled={form.processing}>
    Submit
</button>
```

<a name="using-react-and-inertia"></a>
### 使用 React 與 Inertia

> [!NOTE]  
> 如果您想在開發 Laravel 應用程式時使用 React 和 Inertia 搶先一步，請考慮使用我們的 [入門套件](/docs/{{version}}/starter-kits) 之一。Laravel 的入門套件為您的新 Laravel 應用程式提供了後端和前端身份驗證鷹架。

在使用 Precognition 與 React 和 Inertia 之前，請務必查閱我們關於 [使用 Precognition 與 React](#using-react) 的一般文件。當使用 React 與 Inertia 時，您需要透過 NPM 安裝與 Inertia 相容的 Precognition 函式庫：

```shell
npm install laravel-precognition-react-inertia
```

安裝後，Precognition 的 `useForm` 函式將返回一個 Inertia [表單輔助函式](https://inertiajs.com/forms#form-helper)，並增強了上述驗證功能。

表單輔助函式的 `submit` 方法已簡化，無需指定 HTTP 方法或 URL。相反，您可以將 Inertia 的 [訪問選項](https://inertiajs.com/manual-visits) 作為第一個也是唯一一個參數傳遞。此外，`submit` 方法不會像上面 React 範例中那樣返回 Promise。相反，您可以在提供給 `submit` 方法的訪問選項中提供 Inertia 支援的任何 [事件回呼](https://inertiajs.com/manual-visits#event-callbacks)：

```js
import { useForm } from 'laravel-precognition-react-inertia';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = (e) => {
    e.preventDefault();

    form.submit({
        preserveScroll: true,
        onSuccess: () => form.reset(),
    });
};
```

<a name="using-alpine"></a>
### 使用 Alpine 與 Blade

使用 Laravel Precognition，您可以為使用者提供即時驗證體驗，而無需在前端 Alpine 應用程式中重複您的驗證規則。為了說明其運作方式，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` Middleware 加入到路由定義中。您還應該建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 來存放路由的驗證規則：

```php
use App\Http\Requests\CreateUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (CreateUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下來，您應該透過 NPM 安裝 Laravel Precognition 的 Alpine 前端輔助套件：

```shell
npm install laravel-precognition-alpine
```

然後，在您的 `resources/js/app.js` 檔案中向 Alpine 註冊 Precognition 外掛：

```js
import Alpine from 'alpinejs';
import Precognition from 'laravel-precognition-alpine';

window.Alpine = Alpine;

Alpine.plugin(Precognition);
Alpine.start();
```

安裝並註冊 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `$form`「魔法」建立一個表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 和初始表單資料。

要啟用即時驗證，您應該將表單的資料綁定到其相關輸入，然後監聽每個輸入的 `change` 事件。在 `change` 事件處理函式中，您應該呼叫表單的 `validate` 方法，並提供輸入的名稱：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '',
        email: '',
    }),
}">
    # CSRF 保護

- [簡介](#csrf-introduction)
- [防止 CSRF 請求](#preventing-csrf-requests)
    - [排除 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 簡介

跨網站請求偽造 (Cross-site request forgeries) 是一種惡意攻擊，攻擊者會代表已驗證的使用者執行未經授權的指令。幸運的是，Laravel 讓保護您的應用程式免受 [跨網站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊變得輕而易舉。

<a name="csrf-explanation"></a>
#### 漏洞說明

如果您不熟悉跨網站請求偽造，讓我們討論一個此漏洞如何被利用的範例。想像您的應用程式有一個 `/user/email` 路由，它接受 `POST` 請求來更改已驗證使用者的電子郵件地址。此路由很可能預期一個 `email` 輸入欄位包含使用者想要開始使用的電子郵件地址。

如果沒有 CSRF 保護，惡意網站可以建立一個指向您應用程式 `/user/email` 路由的 HTML 表單，並提交惡意使用者自己的電子郵件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果惡意網站在頁面載入時自動提交表單，惡意使用者只需要引誘您應用程式中毫無戒心的使用者造訪他們的網站，他們的電子郵件地址就會在您的應用程式中被更改。

為了防止此漏洞，我們需要檢查每個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求，以尋找惡意應用程式無法存取的秘密 Session 值。

<a name="preventing-csrf-requests"></a>
## 防止 CSRF 請求

Laravel 會為應用程式管理的每個活躍[使用者 Session](/docs/{{version}}/session) 自動產生一個 CSRF「token」。此 token 用於驗證已驗證的使用者確實是向應用程式發出請求的人。由於此 token 儲存在使用者的 Session 中，並且每次 Session 重新產生時都會更改，因此惡意應用程式無法存取它。

目前 Session 的 CSRF token 可以透過請求的 Session 或 `csrf_token` 輔助函式來存取：

    use Illuminate\Http\Request;

    Route::get('/token', function (Request $request) {
        $token = $request->session()->token();

        $token = csrf_token();

        // ...
    });

每當您在應用程式中定義「POST」、「PUT」、「PATCH」或「DELETE」HTML 表單時，您都應該在表單中包含一個隱藏的 CSRF `_token` 欄位，以便 CSRF 保護 Middleware 可以驗證請求。為了方便起見，您可以使用 `@csrf` Blade 指令來產生隱藏的 token 輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

`Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` [Middleware](/docs/{{version}}/middleware) 預設包含在 `web` Middleware 群組中，它會自動驗證請求輸入中的 token 是否與 Session 中儲存的 token 相符。當這兩個 token 相符時，我們就知道是已驗證的使用者發起了請求。

<a name="csrf-tokens-and-spas"></a>
### CSRF Token 與 SPA

如果您正在建構一個使用 Laravel 作為 API 後端的 SPA，您應該查閱 [Laravel Sanctum 說明文件](/docs/{{version}}/sanctum)，以獲取有關 API 驗證和防止 CSRF 漏洞的資訊。

<a name="csrf-excluding-uris"></a>
### 從 CSRF 保護中排除 URI

有時您可能希望從 CSRF 保護中排除一組 URI。例如，如果您正在使用 [Stripe](https://stripe.com) 處理付款並利用其 Webhook 系統，您將需要從 CSRF 保護中排除您的 Stripe Webhook 處理路由，因為 Stripe 不會知道要向您的路由發送什麼 CSRF token。

通常，您應該將這些類型的路由放置在 Laravel 應用於 `routes/web.php` 檔案中所有路由的 `web` Middleware 群組之外。但是，您也可以透過在應用程式的 `bootstrap/app.php` 檔案中向 `validateCsrfTokens` 方法提供其 URI 來排除特定路由：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->validateCsrfTokens(except: [
            'stripe/*',
            'http://example.com/foo/bar',
            'http://example.com/foo/*',
        ]);
    })

> [!NOTE]  
> 為了方便起見，當[執行測試](/docs/{{version}}/testing)時，CSRF Middleware 會自動對所有路由停用。

<a name="csrf-x-csrf-token"></a>
## X-CSRF-TOKEN

除了檢查 CSRF token 作為 POST 參數外，預設包含在 `web` Middleware 群組中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` Middleware 也會檢查 `X-CSRF-TOKEN` 請求標頭。例如，您可以將 token 儲存在 HTML `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，您可以指示像 jQuery 這樣的函式庫自動將 token 添加到所有請求標頭中。這為您使用傳統 JavaScript 技術的 AJAX 應用程式提供了簡單、方便的 CSRF 保護：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

<a name="csrf-x-xsrf-token"></a>
## X-XSRF-TOKEN

Laravel 會將目前的 CSRF token 儲存在一個加密的 `XSRF-TOKEN` Cookie 中，該 Cookie 會隨框架產生的每個回應一起包含。您可以使用 Cookie 值來設定 `X-XSRF-TOKEN` 請求標頭。

此 Cookie 主要作為開發人員的便利性而發送，因為某些 JavaScript 框架和函式庫，例如 Angular 和 Axios，會自動將其值放置在同源請求的 `X-XSRF-TOKEN` 標頭中。

> [!NOTE]  
> 預設情況下，`resources/js/bootstrap.js` 檔案包含 Axios HTTP 函式庫，它會自動為您發送 `X-XSRF-TOKEN` 標頭。
    <label for="name">Name</label>
    <input
        id="name"
        name="name"
        x-model="form.name"
        @change="form.validate('name')"
    />
    <template x-if="form.invalid('name')">
        <div x-text="form.errors.name"></div>
    </template>

    <label for="email">Email</label>
    <input
        id="email"
        name="email"
        x-model="form.email"
        @change="form.validate('email')"
    />
    <template x-if="form.invalid('email')">
        <div x-text="form.errors.email"></div>
    </template>

    <button :disabled="form.processing">
        Create User
    </button>
</form>
```

現在，當使用者填寫表單時，Precognition 將提供由路由表單請求中的驗證規則驅動的即時驗證輸出。當表單的輸入發生變化時，將向您的 Laravel 應用程式發送一個去抖動的「預知」驗證請求。您可以透過呼叫表單的 `setValidationTimeout` 函式來設定去抖動逾時：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在進行中時，表單的 `validating` 屬性將為 `true`：

```html
<template x-if="form.validating">
    <div>Validating...</div>
</template>
```

在驗證請求或表單提交期間返回的任何驗證錯誤將自動填充表單的 `errors` 物件：

```html
<template x-if="form.invalid('email')">
    <div x-text="form.errors.email"></div>
</template>
```

您可以使用表單的 `hasErrors` 屬性來判斷表單是否有任何錯誤：

```html
<template x-if="form.hasErrors">
    <div><!-- ... --></div>
</template>
```

您也可以透過將輸入的名稱傳遞給表單的 `valid` 和 `invalid` 函式，分別判斷輸入是否通過或未通過驗證：

```html
<template x-if="form.valid('email')">
    <span>✅</span>
</template>

<template x-if="form.invalid('email')">
    <span>❌</span>
</template>
```

> [!WARNING]  
> 表單輸入只有在更改並收到驗證回應後才會顯示為有效或無效。

正如我們所見，您可以掛接到輸入的 `change` 事件並在使用者與其互動時驗證個別輸入；但是，您可能需要驗證使用者尚未互動的輸入。這在建立「精靈 (wizard)」時很常見，您希望在進入下一步之前驗證所有可見的輸入，無論使用者是否與其互動過。

要使用 Precognition 執行此操作，您應該呼叫 `validate` 方法，並將您希望驗證的欄位名稱傳遞給 `only` 設定鍵。您可以使用 `onSuccess` 或 `onValidationError` 回呼來處理驗證結果：

```html
<button
    type="button"
    @click="form.validate({
        only: ['name', 'email', 'phone'],
        onSuccess: (response) => nextStep(),
        onValidationError: (response) => /* ... */,
    })"
>Next Step</button>
```

您可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在進行中：

```html
<button :disabled="form.processing">
    Submit
</button>
```

<a name="repopulating-old-form-data"></a>
#### 重新填充舊表單資料

在上面討論的使用者建立範例中，我們使用 Precognition 執行即時驗證；但是，我們正在執行傳統的伺服器端表單提交來提交表單。因此，表單應該填充從伺服器端表單提交返回的任何「舊」輸入和驗證錯誤：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '{{ old('name') }}',
        email: '{{ old('email') }}',
    }).setErrors({{ Js::from($errors->messages()) }}),
}">
```

或者，如果您想透過 XHR 提交表單，您可以使用表單的 `submit` 函式，它會返回一個 Axios 請求 Promise：

```html
<form
    x-data="{
        form: $form('post', '/register', {
            name: '',
            email: '',
        }),
        submit() {
            this.form.submit()
                .then(response => {
                    form.reset();

                    alert('User created.')
                })
                .catch(error => {
                    alert('An error occurred.');
                });
        },
    }"
    @submit.prevent="submit"
>
```

<a name="configuring-axios"></a>
### 設定 Axios

Precognition 驗證函式庫使用 [Axios](https://github.com/axios/axios) HTTP 用戶端向應用程式的後端發送請求。為了方便起見，如果您的應用程式需要，可以自訂 Axios 實例。例如，當使用 `laravel-precognition-vue` 函式庫時，您可以在應用程式的 `resources/js/app.js` 檔案中為每個傳出請求添加額外的請求標頭：

```js
import { client } from 'laravel-precognition-vue';

client.axios().defaults.headers.common['Authorization'] = authToken;
```

或者，如果您已經為應用程式設定了 Axios 實例，您可以告訴 Precognition 使用該實例：

```js
import Axios from 'axios';
import { client } from 'laravel-precognition-vue';

window.axios = Axios.create()
window.axios.defaults.headers.common['Authorization'] = authToken;

client.use(window.axios)
```

> [!WARNING]  
> Inertia 風格的 Precognition 函式庫只會將設定的 Axios 實例用於驗證請求。表單提交將始終由 Inertia 發送。

<a name="customizing-validation-rules"></a>
## 自訂驗證規則

可以使用請求的 `isPrecognitive` 方法在預知請求期間自訂執行的驗證規則。

例如，在使用者建立表單上，我們可能只想在最終表單提交時驗證密碼是否「未洩露 (uncompromised)」。對於預知驗證請求，我們將只驗證密碼是必需的且至少有 8 個字元。使用 `isPrecognitive` 方法，我們可以自訂表單請求定義的規則：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rules\Password;

class StoreUserRequest extends FormRequest
{
    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    protected function rules()
    {
        return [
            'password' => [
                'required',
                $this->isPrecognitive()
                    ? Password::min(8)
                    : Password::min(8)->uncompromised(),
            ],
            // ...
        ];
    }
}
```

<a name="handling-file-uploads"></a>
## 處理檔案上傳

預設情況下，Laravel Precognition 在預知驗證請求期間不會上傳或驗證檔案。這確保了大型檔案不會不必要地多次上傳。

由於此行為，您應該確保您的應用程式 [自訂相應表單請求的驗證規則](#customizing-validation-rules) 以指定該欄位僅在完整表單提交時才需要：

```php
/**
 * Get the validation rules that apply to the request.
 *
 * @return array
 */
protected function rules()
{
    return [
        'avatar' => [
            ...$this->isPrecognitive() ? [] : ['required'],
            'image',
            'mimes:jpg,png',
            'dimensions:ratio=3/2',
        ],
        // ...
    ];
}
```

如果您想在每個驗證請求中包含檔案，您可以在用戶端表單實例上呼叫 `validateFiles` 函式：

```js
form.validateFiles();
```

<a name="managing-side-effects"></a>
## 管理副作用

當將 `HandlePrecognitiveRequests` Middleware 添加到路由時，您應該考慮在 _其他_ Middleware 中是否有任何副作用應該在預知請求期間跳過。

例如，您可能有一個 Middleware 會增加每個使用者與您的應用程式互動的總次數，但您可能不希望預知請求被計為互動。為了實現這一點，我們可以在增加互動計數之前檢查請求的 `isPrecognitive` 方法：

```php
<?php

namespace App\Http\Middleware;

use App\Facades\Interaction;
use Closure;
use Illuminate\Http\Request;

class InteractionMiddleware
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next): mixed
    {
        if (! $request->isPrecognitive()) {
            Interaction::incrementFor($request->user());
        }

        return $next($request);
    }
}
```

<a name="testing"></a>
## 測試

如果您想在測試中發出預知請求，Laravel 的 `TestCase` 包含一個 `withPrecognition` 輔助函式，它將添加 `Precognition` 請求標頭。

此外，如果您想斷言預知請求成功，例如，沒有返回任何驗證錯誤，您可以使用回應上的 `assertSuccessfulPrecognition` 方法：

```php tab=Pest
it('validates registration form with precognition', function () {
    $response = $this->withPrecognition()
        ->post('/register', [
            'name' => 'Taylor Otwell',
        ]);

    $response->assertSuccessfulPrecognition();

    expect(User::count())->toBe(0);
});
```

```php tab=PHPUnit
public function test_it_validates_registration_form_with_precognition()
{
    $response = $this->withPrecognition()
        ->post('/register', [
            'name' => 'Taylor Otwell',
        ]);

    $response->assertSuccessfulPrecognition();
    $this->assertSame(0, User::count());
}
```

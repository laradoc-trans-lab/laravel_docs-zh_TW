# Precognition

- [簡介](#introduction)
- [即時驗證](#live-validation)
    - [使用 Vue](#using-vue)
    - [使用 React](#using-react)
    - [使用 Alpine 與 Blade](#using-alpine)
    - [設定 Axios](#configuring-axios)
- [自定義驗證規則](#customizing-validation-rules)
- [處理檔案上傳](#handling-file-uploads)
- [管理副作用](#managing-side-effects)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

Laravel Precognition 允許您預判未來 HTTP 請求的結果。Precognition 的主要用例之一是能夠為您的前端 JavaScript 應用程式提供「即時 (live)」驗證，而無需重複編寫應用程式的後端驗證規則。

當 Laravel 收到「precognitive 請求」時，它會執行該路由所有的中介層並解析路由控制器的依賴項，包含驗證 [form requests](/docs/{{version}}/validation#form-request-validation) —— 但它實際上不會執行路由控制器的方法。

> [!NOTE]
> 從 Inertia 2.3 開始，Precognition 支援已內建。請參閱 [Inertia Forms 文件](https://inertiajs.com/docs/v2/the-basics/forms)以獲取更多資訊。更早期的 Inertia 版本則需要 Precognition 0.x。

<a name="live-validation"></a>
## 即時驗證


<a name="using-vue"></a>
### 使用 Vue

使用 Laravel Precognition，你可以為使用者提供即時驗證體驗，而無需在前端 Vue 應用程式中重複編寫驗證規則。為了說明它是如何運作的，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層添加到路由定義中。你還應該建立一個 [表單請求 (Form Request)](/docs/{{version}}/validation#form-request-validation) 來存放路由的驗證規則：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下來，你應該透過 NPM 安裝適用於 Vue 的 Laravel Precognition 前端輔助套件：

```shell
npm install laravel-precognition-vue
```

安裝好 Laravel Precognition 套件後，你現在可以使用 Precognition 的 `useForm` 函式建立一個表單物件，並提供 HTTP 方法 (`post`)、目標 URL (`/users`) 以及初始表單資料。

然後，要啟用即時驗證，請在每個輸入框的 `change` 事件中調用表單的 `validate` 方法，並提供該輸入框的名稱：

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

現在，當使用者填寫表單時，Precognition 將提供由路由表單請求中的驗證規則所驅動的即時驗證輸出。當表單輸入內容變更時，一個經過防抖 (Debounced) 處理的「預判性 (Precognitive)」驗證請求將被發送到你的 Laravel 應用程式。你可以透過調用表單的 `setValidationTimeout` 函式來設定防抖超時時間：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在傳送中時，表單的 `validating` 屬性將為 `true`：

```html
<div v-if="form.validating">
    Validating...
</div>
```

在驗證請求或表單提交期間回傳的任何驗證錯誤都將自動填充到表單的 `errors` 物件中：

```html
<div v-if="form.invalid('email')">
    {{ form.errors.email }}
</div>
```

你可以使用表單的 `hasErrors` 屬性來判斷表單是否存在任何錯誤：

```html
<div v-if="form.hasErrors">
    <!-- ... -->
</div>
```

你還可以透過將輸入框名稱分別傳遞給表單的 `valid` 和 `invalid` 函式，來判斷該輸入是否通過或未通過驗證：

```html
<span v-if="form.valid('email')">
    ✅
</span>

<span v-else-if="form.invalid('email')">
    ❌
</span>
```

> [!WARNING]
> 只有在表單輸入發生變更並收到驗證回應後，該輸入才會顯示為有效或無效。

如果你正在使用 Precognition 驗證表單輸入的子集，手動清除錯誤可能會很有用。你可以使用表單的 `forgetError` 函式來達成此目的：

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

如我們所見，你可以掛接到輸入框的 `change` 事件，並在使用者與其互動時驗證單個輸入；然而，你可能需要驗證使用者尚未互動過的輸入。這在建立「引導精靈 (Wizard)」時很常見，在進入下一步之前，你希望驗證所有可見的輸入框，無論使用者是否已經與其互動。

要使用 Precognition 執行此操作，你應該調用 `validate` 方法，並將你希望驗證的欄位名稱傳遞給 `only` 配置鍵。你可以透過 `onSuccess` 或 `onValidationError` 回呼 (Callbacks) 來處理驗證結果：

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

當然，你也可以根據表單提交的回應來執行程式碼。表單的 `submit` 函式會回傳一個 Axios 請求的 Promise。這提供了一種便捷的方式來存取回應內容、在提交成功時重設表單輸入，或處理失敗的請求：

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

你可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在傳送中：

```html
<button :disabled="form.processing">
    Submit
</button>
```

<a name="using-react"></a>
### 使用 React

使用 Laravel Precognition，您可以為使用者提供即時驗證體驗，而無需在前端 React 應用程式中重複編寫驗證規則。為了說明其運作方式，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層新增到路由定義中。您還應該建立一個 [form request](/docs/{{version}}/validation#form-request-validation) 來存放路由的驗證規則：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下來，您應該透過 NPM 安裝適用於 React 的 Laravel Precognition 前端輔助函式：

```shell
npm install laravel-precognition-react
```

安裝 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `useForm` 函式建立一個表單物件，並提供 HTTP 方法 (`post`)、目標 URL (`/users`) 以及初始表單資料。

要啟用即時驗證，您應該監聽每個輸入框的 `change` 與 `blur` 事件。在 `change` 事件處理程序中，您應該使用 `setData` 函式設定表單資料，並傳入輸入框的名稱與新值。接著，在 `blur` 事件處理程序中呼叫表單的 `validate` 方法，並提供輸入框的名稱：

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

現在，當使用者填寫表單時，Precognition 將提供由路由的 form request 中驗證規則所驅動的即時驗證輸出。當表單輸入內容變更時，一個經過防抖 (debounced) 處理的「預知性 (precognitive)」驗證請求將被發送到您的 Laravel 應用程式。您可以透過呼叫表單的 `setValidationTimeout` 函式來設定防抖超時時間：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在傳送中時，表單的 `validating` 屬性將為 `true`：

```jsx
{form.validating && <div>Validating...</div>}
```

在驗證請求或表單提交期間傳回的任何驗證錯誤都將自動填入表單的 `errors` 物件中：

```jsx
{form.invalid('email') && <div>{form.errors.email}</div>}
```

您可以使用表單的 `hasErrors` 屬性來判斷表單是否有任何錯誤：

```jsx
{form.hasErrors && <div><!-- ... --></div>}
```

您也可以分別將輸入框名稱傳遞給表單的 `valid` 與 `invalid` 函式，來判斷輸入框是否通過或未通過驗證：

```jsx
{form.valid('email') && <span>✅</span>}

{form.invalid('email') && <span>❌</span>}
```

> [!WARNING]
> 表單輸入框只有在內容變更且收到驗證回應後，才會顯示為有效或無效。

如果您正在使用 Precognition 驗證表單輸入框的子集，手動清除錯誤可能會很有用。您可以使用表單的 `forgetError` 函式來達成此目的：

```jsx
<input
    id="avatar"
    type="file"
    onChange={(e) => {
        form.setData('avatar', e.target.files[0]);

        form.forgetError('avatar');
    }}
>
```

如我們所見，您可以掛鉤到輸入框的 `blur` 事件，並在使用者與其互動時驗證個別輸入框；但是，您可能需要驗證使用者尚未與其互動的輸入框。這在建立「精靈 (wizard)」時很常見，您希望在移至下一步之前驗證所有可見的輸入框，無論使用者是否已與其互動。

若要使用 Precognition 執行此操作，您應該呼叫 `validate` 方法，並將您希望驗證的欄位名稱傳遞給 `only` 設定鍵。您可以使用 `onSuccess` 或 `onValidationError` 回呼函式來處理驗證結果：

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

當然，您也可以根據表單提交的回應執行對應的程式碼。表單的 `submit` 函式會傳回一個 Axios 請求 Promise。這提供了一種便捷的方式來存取回應內容、在成功提交時重設表單輸入內容，或處理失敗的請求：

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

<a name="using-alpine"></a>
### 使用 Alpine 與 Blade

透過使用 Laravel Precognition，您可以為使用者提供即時驗證體驗，而無需在前端 Alpine 應用程式中重複編寫驗證規則。為了說明它是如何運作的，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，若要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層加入到路由定義中。您還應該建立一個 [form request](/docs/{{version}}/validation#form-request-validation) 來放置路由的驗證規則：

```php
use App\Http\Requests\CreateUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (CreateUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接著，您應該透過 NPM 安裝適用於 Alpine 的 Laravel Precognition 前端輔助套件：

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

安裝並註冊 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `$form`「魔法」建立一個表單物件，並提供 HTTP 方法 (`post`)、目標 URL (`/users`) 以及初始表單資料。

要啟用即時驗證，您應該將表單資料綁定到其相關的輸入框，然後監聽每個輸入框的 `change` 事件。在 `change` 事件處理器中，您應該呼叫表單的 `validate` 方法，並提供該輸入框的名稱：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '',
        email: '',
    }),
}">
    @csrf
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

現在，當使用者填寫表單時，Precognition 將提供由路由表單請求中的驗證規則所驅動的即時驗證輸出。當表單輸入內容變更時，一個經去抖動 (debounced) 的「預知性 (precognitive)」驗證請求將發送到您的 Laravel 應用程式。您可以透過呼叫表單的 `setValidationTimeout` 函式來設定去抖動逾時時間：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在傳送時，表單的 `validating` 屬性將為 `true`：

```html
<template x-if="form.validating">
    <div>Validating...</div>
</template>
```

在驗證請求或表單提交期間回傳的任何驗證錯誤都將自動填入表單的 `errors` 物件中：

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

您也可以分別將輸入框名稱傳遞給表單的 `valid` 和 `invalid` 函式，來判斷該輸入框是否通過或未通過驗證：

```html
<template x-if="form.valid('email')">
    <span>✅</span>
</template>

<template x-if="form.invalid('email')">
    <span>❌</span>
</template>
```

> [!WARNING]
> 只有在表單輸入內容發生變更並收到驗證回應後，該輸入框才會顯示為有效或無效。

如我們所見，您可以掛鉤到輸入框的 `change` 事件，並在使用者與其互動時驗證個別輸入內容；但是，您可能需要驗證使用者尚未互動過的輸入內容。這在建立「嚮導 (wizard)」式表單時很常見，您希望在進入下一步之前，驗證所有可見的輸入內容，無論使用者是否已經與其互動。

要使用 Precognition 達成此目的，您應該呼叫 `validate` 方法，並將您希望驗證的欄位名稱傳遞給 `only` 設定鍵。您可以使用 `onSuccess` 或 `onValidationError` 回呼來處理驗證結果：

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

您可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在傳送中：

```html
<button :disabled="form.processing">
    Submit
</button>
```


<a name="repopulating-old-form-data"></a>
#### 重新填入舊的表單資料

在上面討論的使用者建立範例中，我們使用 Precognition 來執行即時驗證；但是，我們正在執行傳統的伺服器端表單提交來提交表單。因此，表單應該填入從伺服器端表單提交回傳的任何「舊 (old)」輸入內容和驗證錯誤：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '{{ old('name') }}',
        email: '{{ old('email') }}',
    }).setErrors({{ Js::from($errors->messages()) }}),
}">
```

或者，如果您希望透過 XHR 提交表單，可以使用表單的 `submit` 函式，它會回傳一個 Axios 請求 Promise：

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
                    this.form.reset();

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

Precognition 驗證程式庫使用 [Axios](https://github.com/axios/axios) HTTP 用戶端將請求發送到應用程式的後端。為了方便起見，如果您的應用程式有需求，可以自定義 Axios 實例。例如，使用 `laravel-precognition-vue` 程式庫時，您可以在應用程式的 `resources/js/app.js` 檔案中為每個發出的請求新增額外的請求標頭：

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

<a name="customizing-validation-rules"></a>
## 自定義驗證規則

透過使用請求的 `isPrecognitive` 方法，可以自定義在 Precognition 請求期間執行的驗證規則。

例如，在使用者建立表單中，我們可能希望僅在最終提交表單時才驗證密碼是否「未受損 (uncompromised)」。對於 Precognition 驗證請求，我們只需驗證密碼為必填且至少包含 8 個字元。使用 `isPrecognitive` 方法，我們可以自定義由 form request 定義的規則：

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

預設情況下，Laravel Precognition 不會在 Precognition 驗證請求期間上傳或驗證檔案。這確保了大型檔案不會被不必要地多次上傳。

由於這種行為，您應該確保您的應用程式[自定義對應的 form request 驗證規則](#customizing-validation-rules)，以指定該欄位僅在完整表單提交時才為必填：

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

如果您希望在每次驗證請求中都包含檔案，您可以在客戶端表單實例上呼叫 `validateFiles` 函式：

```js
form.validateFiles();
```


<a name="managing-side-effects"></a>
## 管理副作用

當將 `HandlePrecognitiveRequests` 中介層加入路由時，您應該考慮 _其他_ 中介層中是否有任何應在 Precognition 請求期間跳過的副作用。

例如，您可能有一個中介層會增加每個使用者與應用程式的「互動 (interactions)」總次數，但您可能不希望將 Precognition 請求計為一次互動。為了實現這一點，我們可以在增加互動次數之前檢查請求的 `isPrecognitive` 方法：

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

如果您想在測試中發送 Precognition 請求，Laravel 的 `TestCase` 包含了一個 `withPrecognition` 輔助函式，它會加入 `Precognition` 請求標頭。

此外，如果您想斷言 Precognition 請求是否成功（例如，沒有回傳任何驗證錯誤），您可以在回應上使用 `assertSuccessfulPrecognition` 方法：

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
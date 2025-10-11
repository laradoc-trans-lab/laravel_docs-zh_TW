# 預知 (Precognition)

- [簡介](#introduction)
- [即時驗證](#live-validation)
    - [使用 Vue](#using-vue)
    - [使用 Vue 和 Inertia](#using-vue-and-inertia)
    - [使用 React](#using-react)
    - [使用 React 和 Inertia](#using-react-and-inertia)
    - [使用 Alpine 和 Blade](#using-alpine)
    - [設定 Axios](#configuring-axios)
- [自訂驗證規則](#customizing-validation-rules)
- [處理檔案上傳](#handling-file-uploads)
- [管理副作用](#managing-side-effects)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

Laravel Precognition 讓您能夠預期未來 HTTP 請求的結果。Precognition 的主要用例之一是能夠為您的前端 JavaScript 應用程式提供「即時」驗證，而無需重複您的應用程式後端驗證規則。Precognition 特別適合搭配 Laravel 基於 Inertia 的[入門套件](/docs/{{version}}/starter-kits)。

當 Laravel 收到「預知請求」時，它將執行所有路由的中介層並解析路由控制器的依賴關係，包括驗證[表單請求](/docs/{{version}}/validation#form-request-validation) — 但它實際上不會執行路由的控制器方法。

<a name="live-validation"></a>
## 即時驗證

<a name="using-vue"></a>
### 使用 Vue

透過 Laravel Precognition，您可以為使用者提供即時驗證體驗，而無需在前端 Vue 應用程式中重複定義驗證規則。為了說明其運作方式，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，若要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層新增至路由定義中。您還應建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 以容納路由的驗證規則：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接著，您應該透過 NPM 安裝用於 Vue 的 Laravel Precognition 前端輔助套件：

```shell
npm install laravel-precognition-vue
```

安裝 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `useForm` 函式建立一個表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 和初始表單資料。

然後，若要啟用即時驗證，請在每個輸入欄位的 `change` 事件上呼叫表單的 `validate` 方法，並提供輸入欄位的名稱：

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

現在，當使用者填寫表單時，Precognition 將根據路由表單請求中的驗證規則提供即時驗證輸出。當表單的輸入欄位被更改時，將會向您的 Laravel 應用程式傳送一個延遲 (debounced) 的「預知」驗證請求。您可以透過呼叫表單的 `setValidationTimeout` 函式來配置延遲逾時時間：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在執行中時，表單的 `validating` 屬性將為 `true`：

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

您也可以透過將輸入欄位的名稱傳遞給表單的 `valid` 和 `invalid` 函式，來判斷輸入欄位是否分別通過或未能通過驗證：

```html
<span v-if="form.valid('email')">
    ✅
</span>

<span v-else-if="form.invalid('email')">
    ❌
</span>
```

> [!WARNING]  
> 表單輸入欄位只會在更改且接收到驗證回應後，才會顯示為有效或無效。

如果您正在使用 Precognition 驗證表單輸入欄位的一個子集，手動清除錯誤會很有用。您可以使用表單的 `forgetError` 函式來達到此目的：

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

如我們所見，您可以監聽輸入欄位的 `change` 事件並在使用者與其互動時驗證個別輸入欄位；但是，您可能需要驗證使用者尚未互動過的輸入欄位。這在建立「嚮導」時很常見，您希望在進入下一步之前驗證所有可見的輸入欄位，無論使用者是否與其互動過。

若要使用 Precognition 做到這點，您應該呼叫 `validate` 方法，並將您希望驗證的欄位名稱傳遞給 `only` 設定鍵。您可以透過 `onSuccess` 或 `onValidationError` 回呼來處理驗證結果：

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

當然，您也可以在表單提交的回應後執行程式碼。表單的 `submit` 函式會回傳一個 Axios 請求 Promise。這提供了一個方便的方式來存取回應酬載、在成功提交後重設表單輸入欄位，或處理失敗的請求：

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

您可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在執行中：

```html
<button :disabled="form.processing">
    Submit
</button>
```

<a name="using-vue-and-inertia"></a>
### 使用 Vue 和 Inertia

> [!NOTE]  
> 如果您想在開發使用 Vue 和 Inertia 的 Laravel 應用程式時搶先一步，請考慮使用我們的 [starter kits](/docs/{{version}}/starter-kits) 之一。Laravel 的 starter kits 為您的新 Laravel 應用程式提供了後端和前端認證鷹架。

在將 Precognition 與 Vue 和 Inertia 結合使用之前，請務必查閱我們關於 [使用 Vue 與 Precognition](#using-vue) 的一般文件。當使用 Vue 與 Inertia 時，您需要透過 NPM 安裝與 Inertia 相容的 Precognition 函式庫：

```shell
npm install laravel-precognition-vue-inertia
```

安裝完成後，Precognition 的 `useForm` 函式將返回一個 Inertia [表單輔助函式](https://inertiajs.com/forms#form-helper)，並增強了上述的驗證功能。

表單輔助函式的 `submit` 方法已精簡化，無需指定 HTTP 方法或 URL。相反地，您可以將 Inertia 的 [造訪選項](https://inertiajs.com/manual-visits) 作為第一個也是唯一一個引數傳遞。此外，如上述 Vue 範例所示，`submit` 方法不會回傳 Promise。相反地，您可以在提供給 `submit` 方法的造訪選項中，提供任何 Inertia 支援的 [事件回呼](https://inertiajs.com/manual-visits#event-callbacks)：

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

使用 Laravel Precognition，您可以為使用者提供即時驗證體驗，而無需在前端 React 應用程式中重複您的驗證規則。為了說明其運作方式，讓我們在應用程式中建構一個用於建立新使用者的表單。

首先，若要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層新增到路由定義中。您還應該建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 以存放路由的驗證規則：

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

要啟用即時驗證，您應該監聽每個輸入欄位的 `change` 和 `blur` 事件。在 `change` 事件處理器中，您應該使用 `setData` 函式設定表單資料，傳入輸入欄位的名稱和新值。然後，在 `blur` 事件處理器中呼叫表單的 `validate` 方法，提供輸入欄位的名稱：

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

現在，當使用者填寫表單時，Precognition 將提供由路由的表單請求中的驗證規則所驅動的即時驗證輸出。當表單的輸入欄位發生變更時，將向您的 Laravel 應用程式發送一個去抖動 (debounced) 的「預知」驗證請求。您可以透過呼叫表單的 `setValidationTimeout` 函式來設定去抖動逾時時間：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在執行時，表單的 `validating` 屬性將為 `true`：

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

您也可以透過將輸入欄位的名稱傳給表單的 `valid` 和 `invalid` 函式，分別判斷輸入欄位是否通過或未通過驗證：

```jsx
{form.valid('email') && <span>✅</span>}

{form.invalid('email') && <span>❌</span>}
```

> [!WARNING]  
> 表單輸入欄位只有在變更且收到驗證回應後才會顯示為有效或無效。

如果您正在使用 Precognition 驗證表單輸入欄位的一個子集，手動清除錯誤會很有用。您可以使用表單的 `forgetError` 函式來實現此目的：

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

如我們所見，您可以掛接到輸入欄位的 `blur` 事件，並在使用者與輸入欄位互動時驗證單個輸入欄位；但是，您可能需要驗證使用者尚未互動的輸入欄位。這在建構「精靈」(wizard) 時很常見，在進入下一步之前，您希望驗證所有可見的輸入欄位，無論使用者是否與其互動過。

要使用 Precognition 實現此目的，您應該呼叫 `validate` 方法，將您希望驗證的欄位名稱傳給 `only` 設定鍵。您可以使用 `onSuccess` 或 `onValidationError` 回呼函式處理驗證結果：

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

當然，您也可以根據表單提交的回應執行程式碼。表單的 `submit` 函式會返回一個 Axios 請求 Promise。這提供了一種便捷的方式來存取回應酬載、在成功提交表單時重設表單輸入欄位，或處理失敗的請求：

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

您可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在執行：

```html
<button disabled={form.processing}>
    Submit
</button>
```


<a name="using-react-and-inertia"></a>
### 使用 React 和 Inertia

> [!NOTE]  
> 如果您想在使用 React 和 Inertia 開發 Laravel 應用程式時有一個好的開端，請考慮使用我們的其中一個[入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件為您新的 Laravel 應用程式提供後端和前端身份驗證骨架。

在使用 Precognition 搭配 React 和 Inertia 之前，請務必查看我們關於[使用 Precognition 搭配 React](#using-react) 的一般文件。當使用 React 搭配 Inertia 時，您需要透過 NPM 安裝 Inertia 相容的 Precognition 函式庫：

```shell
npm install laravel-precognition-react-inertia
```

安裝後，Precognition 的 `useForm` 函式將返回一個 Inertia [表單輔助函式](https://inertiajs.com/forms#form-helper)，其增強了上述驗證功能。

表單輔助函式的 `submit` 方法已簡化，無需指定 HTTP 方法或 URL。相反，您可以將 Inertia 的[造訪選項](https://inertiajs.com/manual-visits)作為第一個也是唯一一個參數傳入。此外，`submit` 方法不像上面 React 範例中所示那樣返回 Promise。相反，您可以在提供給 `submit` 方法的造訪選項中提供任何 Inertia 支援的[事件回呼函式](https://inertiajs.com/manual-visits#event-callbacks)：

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
### 使用 Alpine 和 Blade

使用 Laravel Precognition，您無需在前端 Alpine 應用程式中重複驗證規則，即可為使用者提供即時驗證體驗。為了說明其運作方式，我們將在應用程式中建立一個用於建立新使用者的表單。

首先，若要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層新增至路由定義中。您還應建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 以容納路由的驗證規則：

```php
use App\Http\Requests\CreateUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (CreateUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下來，您應該透過 NPM 安裝適用於 Alpine 的 Laravel Precognition 前端輔助程式：

```shell
npm install laravel-precognition-alpine
```

然後，在您的 `resources/js/app.js` 檔案中將 Precognition plugin 註冊到 Alpine：

```js
import Alpine from 'alpinejs';
import Precognition from 'laravel-precognition-alpine';

window.Alpine = Alpine;

Alpine.plugin(Precognition);
Alpine.start();
```

安裝並註冊 Laravel Precognition 軟體套件後，您現在可以使用 Precognition 的 `$form` 「魔術方法」來建立表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 以及初始表單資料。

為了啟用即時驗證，您應該將表單的資料綁定到其相關的輸入欄位，然後監聽每個輸入欄位的 `change` 事件。在 `change` 事件處理器中，您應該呼叫表單的 `validate` 方法，並提供輸入欄位的名稱：

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

現在，隨著使用者填寫表單，Precognition 將提供由路由的表單請求中驗證規則驅動的即時驗證輸出。當表單的輸入欄位變更時，一個去抖動的「預知」驗證請求將發送到您的 Laravel 應用程式。您可以透過呼叫表單的 `setValidationTimeout` 函數來設定去抖動逾時：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在進行中時，表單的 `validating` 屬性將為 `true`：

```html
<template x-if="form.validating">
    <div>Validating...</div>
</template>
```

在驗證請求或表單提交期間返回的任何驗證錯誤將自動填入表單的 `errors` 物件：

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

您也可以透過將輸入欄位的名稱傳遞給表單的 `valid` 和 `invalid` 函數，來判斷輸入欄位是否通過或失敗驗證：

```html
<template x-if="form.valid('email')">
    <span>✅</span>
</template>

<template x-if="form.invalid('email')">
    <span>❌</span>
</template>
```

> [!WARNING]
> 表單輸入欄位僅在變更並收到驗證回應後才會顯示為有效或無效。

如我們所見，您可以掛接到輸入欄位的 `change` 事件並在使用者與其互動時驗證個別輸入欄位；但是，您可能需要驗證使用者尚未互動的輸入欄位。這在建立「精靈」時很常見，您希望在進入下一步之前驗證所有可見的輸入欄位，無論使用者是否與其互動過。

為了使用 Precognition 實現這一點，您應該呼叫 `validate` 方法，並將您希望驗證的欄位名稱傳遞給 `only` 設定鍵。您可以透過 `onSuccess` 或 `onValidationError` 回呼函數來處理驗證結果：

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
#### 重新填入舊有表單資料

在上述討論的使用者建立範例中，我們使用 Precognition 執行即時驗證；但是，我們正在執行傳統的伺服器端表單提交以提交表單。因此，表單應該填入從伺服器端表單提交返回的任何「舊有」輸入和驗證錯誤：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '{{ old('name') }}',
        email: '{{ old('email') }}',
    }).setErrors({{ Js::from($errors->messages()) }}),
}">
```

或者，如果您想透過 XHR 提交表單，您可以使用表單的 `submit` 函數，該函數會返回一個 Axios 請求 Promise：

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

Precognition 驗證函式庫使用 [Axios](https://github.com/axios/axios) HTTP 用戶端向應用程式的後端發送請求。為了方便起見，如果您的應用程式需要，可以自訂 Axios 實例。例如，當使用 `laravel-precognition-vue` 函式庫時，您可以在應用程式的 `resources/js/app.js` 檔案中為每個發出的請求新增額外的請求標頭：

```js
import { client } from 'laravel-precognition-vue';

client.axios().defaults.headers.common['Authorization'] = authToken;
```

或者，如果您已經為應用程式設定了一個 Axios 實例，您可以告訴 Precognition 改用該實例：

```js
import Axios from 'axios';
import { client } from 'laravel-precognition-vue';

window.axios = Axios.create()
window.axios.defaults.headers.common['Authorization'] = authToken;

client.use(window.axios)
```

> [!WARNING]
> Inertia 風味的 Precognition 函式庫僅會將已設定的 Axios 實例用於驗證請求。表單提交將始終由 Inertia 發送。

<a name="customizing-validation-rules"></a>
## 自訂驗證規則

可以在預知請求期間，使用請求的 `isPrecognitive` 方法來自訂驗證規則。

舉例來說，在建立使用者表單時，我們可能希望僅在最終表單提交時才驗證密碼是否為「未曾外洩」。對於預知驗證請求，我們將簡單地驗證密碼為必填且至少有 8 個字元。藉由使用 `isPrecognitive` 方法，我們可以自訂由表單請求所定義的規則：

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

依預設，Laravel Precognition 在預知驗證請求期間不會上傳或驗證檔案。這可確保大型檔案不會被不必要地多次上傳。

由於這種行為，你應該確保應用程式[自訂對應表單請求的驗證規則](#customizing-validation-rules)，以指定該欄位僅在完整表單提交時才為必填：

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

如果你想在每次驗證請求中都包含檔案，可以在用戶端表單實例上呼叫 `validateFiles` 函式：

```js
form.validateFiles();
```

<a name="managing-side-effects"></a>
## 管理副作用

將 `HandlePrecognitiveRequests` 中介層新增至路由時，你應該考慮在預知請求期間，是否應跳過*其他*中介層中的任何副作用。

例如，你可能有一個中介層會增加每個使用者與應用程式的「互動」總數，但你可能不希望預知請求被計為一次互動。為此，我們可以在增加互動計數之前，檢查請求的 `isPrecognitive` 方法：

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

如果你想在測試中發出預知請求，Laravel 的 `TestCase` 包含一個 `withPrecognition` 輔助函式，它會新增 `Precognition` 請求標頭。

此外，如果你想斷言預知請求成功，例如沒有回傳任何驗證錯誤，可以在回應上使用 `assertSuccessfulPrecognition` 方法：

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
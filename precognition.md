# Precognition

- [簡介](#introduction)
- [即時驗證](#live-validation)
    - [使用 Vue](#using-vue)
    - [使用 Vue 和 Inertia](#using-vue-and-inertia)
    - [使用 React](#using-react)
    - [使用 React 和 Inertia](#using-react-and-inertia)
    - [使用 Alpine 和 Blade](#using-alpine)
    - [配置 Axios](#configuring-axios)
- [自訂驗證規則](#customizing-validation-rules)
- [處理檔案上傳](#handling-file-uploads)
- [管理副作用](#managing-side-effects)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

Laravel Precognition 讓您能預期未來 HTTP 請求的結果。Precognition 的主要使用案例之一，是能夠在您的前端 JavaScript 應用程式中提供「即時」驗證，而無需重複您的應用程式後端驗證規則。Precognition 特別適合搭配 Laravel 基於 Inertia 的[入門套件](/docs/{{version}}/starter-kits)。

當 Laravel 收到「預知請求」(precognitive request) 時，它會執行該路由的所有中介層並解析該路由的控制器依賴，包括驗證[表單請求](/docs/{{version}}/validation#form-request-validation)——但它實際上不會執行該路由的控制器方法。

<a name="live-validation"></a>
## 即時驗證


<a name="using-vue"></a>
### 使用 Vue

使用 Laravel Precognition，您無需在前端 Vue 應用程式中重複您的驗證規則，即可為使用者提供即時驗證體驗。為了說明其運作方式，讓我們在應用程式中建立一個用於建立新使用者的表單。

首先，若要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層新增至路由定義中。您還應建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 以放置該路由的驗證規則：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下來，您應該透過 NPM 安裝 Laravel Precognition 適用於 Vue 的前端輔助套件：

```shell
npm install laravel-precognition-vue
```

安裝 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `useForm` 函式建立一個表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 以及初始表單資料。

然後，為了啟用即時驗證，請在每個輸入欄位的 `change` 事件上呼叫表單的 `validate` 方法，並提供輸入欄位的名稱：

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

現在，當使用者填寫表單時，Precognition 將根據路由的表單請求中的驗證規則提供即時驗證輸出。當表單的輸入欄位被更改時，一個帶有防抖動機制的「預知」驗證請求將發送到您的 Laravel 應用程式。您可以透過呼叫表單的 `setValidationTimeout` 函式來配置防抖動逾時：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在處理中時，表單的 `validating` 屬性將為 `true`：

```html
<div v-if="form.validating">
    Validating...
</div>
```

在驗證請求或表單提交期間回傳的任何驗證錯誤都將自動填入表單的 `errors` 物件：

```html
<div v-if="form.invalid('email')">
    {{ form.errors.email }}
</div>
```

您可以使用表單的 `hasErrors` 屬性判斷表單是否有任何錯誤：

```html
<div v-if="form.hasErrors">
    <!-- ... -->
</div>
```

您也可以透過將輸入欄位的名稱傳遞給表單的 `valid` 和 `invalid` 函式，分別判斷輸入欄位是否通過或失敗驗證：

```html
<span v-if="form.valid('email')">
    ✅
</span>

<span v-else-if="form.invalid('email')">
    ❌
</span>
```

> [!WARNING]
> 表單輸入欄位僅在更改且收到驗證回應後才會顯示為有效或無效。

如果您正在使用 Precognition 驗證表單輸入欄位的一個子集，手動清除錯誤會很有用。您可以使用表單的 `forgetError` 函式來達成此目的：

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

正如我們所見，您可以監聽輸入欄位的 `change` 事件並在使用者與其互動時驗證單個輸入欄位；但是，您可能需要驗證使用者尚未互動過的輸入欄位。這在建立「精靈」(wizard) 時很常見，您希望在進入下一步之前驗證所有可見的輸入欄位，無論使用者是否與其互動過。

若要使用 Precognition 執行此操作，您應該呼叫 `validate` 方法，將您希望驗證的欄位名稱傳遞給 `only` 配置鍵。您可以透過 `onSuccess` 或 `onValidationError` 回呼函式來處理驗證結果：

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

當然，您也可以根據表單提交的回應執行程式碼。表單的 `submit` 函式會回傳一個 Axios 請求 Promise。這提供了一個方便的方式來存取回應的資料酬載、在成功提交後重設表單輸入，或處理失敗的請求：

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

您可以使用表單的 `processing` 屬性判斷表單提交請求是否正在處理中：

```html
<button :disabled="form.processing">
    Submit
</button>
```


<a name="using-vue-and-inertia"></a>
### 使用 Vue 和 Inertia

> [!NOTE]
> 如果您想在使用 Vue 和 Inertia 開發您的 Laravel 應用程式時事半功倍，請考慮使用我們其中一個 [入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件為您的新 Laravel 應用程式提供了後端和前端身份驗證骨架。

在使用 Precognition 搭配 Vue 和 Inertia 之前，請務必查閱我們關於 [使用 Precognition 搭配 Vue](#using-vue) 的一般文件。當使用 Vue 搭配 Inertia 時，您需要透過 NPM 安裝相容於 Inertia 的 Precognition 函式庫：

```shell
npm install laravel-precognition-vue-inertia
```

安裝後，Precognition 的 `useForm` 函式將回傳一個 Inertia [表單輔助函式](https://inertiajs.com/forms#form-helper)，並增強了上述的驗證功能。

表單輔助函式的 `submit` 方法已經過精簡化，無需指定 HTTP 方法或 URL。相反，您可以將 Inertia 的 [訪問選項](https://inertiajs.com/manual-visits) 作為第一個且是唯一參數傳遞。此外，`submit` 方法不會像上述 Vue 範例中那樣回傳一個 Promise。相反，您可以在傳遞給 `submit` 方法的訪問選項中提供 Inertia 支援的任何 [事件回呼函式](https://inertiajs.com/manual-visits#event-callbacks)：

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

使用 Laravel Precognition，您可以在前端 React 應用程式中為使用者提供即時驗證體驗，而無需重複後端驗證規則。為了說明其運作方式，讓我們建立一個用於在應用程式中建立新使用者的表單。

首先，若要為路由啟用 Precognition，應將 `HandlePrecognitiveRequests` 中介層新增至路由定義中。您還應該建立一個 [表單請求](/docs/{{version}}/validation#form-request-validation) 來存放路由的驗證規則：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接著，您應該透過 NPM 安裝 Laravel Precognition 針對 React 的前端輔助函式庫：

```shell
npm install laravel-precognition-react
```

安裝 Laravel Precognition 套件後，您現在可以使用 Precognition 的 `useForm` 函式來建立一個表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 以及初始表單資料。

若要啟用即時驗證，您應該監聽每個輸入欄位的 `change` 和 `blur` 事件。在 `change` 事件處理器中，您應該使用 `setData` 函式來設定表單資料，傳遞輸入欄位的名稱和新值。然後，在 `blur` 事件處理器中呼叫表單的 `validate` 方法，傳遞輸入欄位的名稱：

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

現在，當使用者填寫表單時，Precognition 將提供由路由的表單請求中驗證規則驅動的即時驗證輸出。當表單的輸入欄位更改時，將向您的 Laravel 應用程式傳送一個去抖動的「預先認知 (precognitive)」驗證請求。您可以透過呼叫表單的 `setValidationTimeout` 函式來設定去抖動逾時：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在執行時，表單的 `validating` 屬性將為 `true`：

```jsx
{form.validating && <div>Validating...</div>}
```

在驗證請求或表單提交期間返回的任何驗證錯誤都將自動填入表單的 `errors` 物件：

```jsx
{form.invalid('email') && <div>{form.errors.email}</div>}
```

您可以使用表單的 `hasErrors` 屬性來判斷表單是否有任何錯誤：

```jsx
{form.hasErrors && <div><!-- ... --></div>}
```

您也可以透過將輸入欄位的名稱傳遞給表單的 `valid` 和 `invalid` 函式，分別判斷輸入欄位是否通過或未通過驗證：

```jsx
{form.valid('email') && <span>✅</span>}

{form.invalid('email') && <span>❌</span>}
```

> [!WARNING]
> 表單輸入欄位只有在更改且收到驗證回應後才會顯示為有效或無效。

如果您正在使用 Precognition 驗證表單輸入欄位的一個子集，手動清除錯誤會很有用。您可以使用表單的 `forgetError` 函式來實現此目的：

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

如我們所見，您可以掛接到輸入欄位的 `blur` 事件並在使用者與其互動時驗證個別輸入欄位；但是，您可能需要驗證使用者尚未互動過的輸入欄位。這在建立「精靈 (wizard)」時很常見，您希望在進入下一步之前驗證所有可見的輸入欄位，無論使用者是否與其互動過。

若要使用 Precognition 執行此操作，您應該呼叫 `validate` 方法，將您希望驗證的欄位名稱傳遞給 `only` 配置鍵。您可以透過 `onSuccess` 或 `onValidationError` 回呼來處理驗證結果：

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

當然，您也可以執行程式碼以回應表單提交的回應。表單的 `submit` 函式會回傳一個 Axios 請求 Promise。這提供了一種方便的方式來存取回應負載 (payload)、在成功提交後重設表單輸入欄位，或處理失敗的請求：

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

您可以使用表單的 `processing` 屬性來判斷表單提交請求是否正在執行：

```html
<button disabled={form.processing}>
    Submit
</button>
```

<a name="using-react-and-inertia"></a>
### 使用 React 和 Inertia

> [!NOTE]
> 如果您想在使用 React 和 Inertia 開發 Laravel 應用程式時搶先一步，請考慮使用我們的 [入門套件](/docs/{{version}}/starter-kits) 之一。Laravel 的入門套件為您的新 Laravel 應用程式提供了後端和前端身份驗證骨架。

在使用 Precognition 與 React 和 Inertia 之前，請務必查閱我們關於 [使用 Precognition 和 React](#using-react) 的一般文件。當使用 React 與 Inertia 時，您需要透過 NPM 安裝 Inertia 相容的 Precognition 函式庫：

```shell
npm install laravel-precognition-react-inertia
```

安裝後，Precognition 的 `useForm` 函式將回傳一個 Inertia [表單輔助函式](https://inertiajs.com/forms#form-helper)，並增強了上述的驗證功能。

表單輔助函式的 `submit` 方法已簡化，無需指定 HTTP 方法或 URL。相反地，您可以將 Inertia 的 [訪問選項](https://inertiajs.com/manual-visits) 作為第一個也是唯一一個參數傳遞。此外，`submit` 方法不會回傳像上述 React 範例中看到的 Promise。相反地，您可以在提供給 `submit` 方法的訪問選項中提供任何 Inertia 支援的 [事件回呼](https://inertiajs.com/manual-visits#event-callbacks)：

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

藉由 Laravel Precognition，你可以在前端 Alpine 應用程式中為使用者提供即時驗證體驗，而無須重複你的後端驗證規則。為了說明其運作方式，讓我們先在應用程式中建立一個用於創建新使用者的表單。

首先，為啟用某個路由的 Precognition 功能，應將 `HandlePrecognitiveRequests` 中介層加到該路由定義中。你也應該建立一個[表單請求](/docs/{{version}}/validation#form-request-validation)來存放該路由的驗證規則：

```php
use App\Http\Requests\CreateUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (CreateUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接著，你應該透過 NPM 安裝 Laravel Precognition 適用於 Alpine 的前端輔助工具：

```shell
npm install laravel-precognition-alpine
```

然後，在你的 `resources/js/app.js` 檔案中向 Alpine 註冊 Precognition 外掛程式：

```js
import Alpine from 'alpinejs';
import Precognition from 'laravel-precognition-alpine';

window.Alpine = Alpine;

Alpine.plugin(Precognition);
Alpine.start();
```

在安裝並註冊 Laravel Precognition 套件後，你現在可以使用 Precognition 的 `$form`「魔法」來建立表單物件，提供 HTTP 方法 (`post`)、目標 URL (`/users`) 以及初始表單資料。

為啟用即時驗證，你應該將表單資料綁定到其相關的輸入欄位，然後監聽每個輸入欄位的 `change` 事件。在 `change` 事件處理器中，你應該呼叫表單的 `validate` 方法，並提供輸入欄位的名稱：

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

現在，當使用者填寫表單時，Precognition 會根據路由的表單請求中的驗證規則，提供即時的驗證輸出。當表單的輸入欄位變更時，會向你的 Laravel 應用程式發送一個經過防抖 (debounced) 的「預驗證 (precognitive)」請求。你可以透過呼叫表單的 `setValidationTimeout` 函式來設定防抖逾時：

```js
form.setValidationTimeout(3000);
```

當驗證請求正在進行中時，表單的 `validating` 屬性將為 `true`：

```html
<template x-if="form.validating">
    <div>Validating...</div>
</template>
```

在驗證請求或表單提交期間返回的任何驗證錯誤，都將自動填入表單的 `errors` 物件：

```html
<template x-if="form.invalid('email')">
    <div x-text="form.errors.email"></div>
</template>
```

你可以使用表單的 `hasErrors` 屬性來判斷表單是否有任何錯誤：

```html
<template x-if="form.hasErrors">
    <div><!-- ... --></div>
</template>
```

你也可以透過將輸入欄位的名稱傳遞給表單的 `valid` 和 `invalid` 函式，來判斷該輸入欄位是否分別通過或未能通過驗證：

```html
<template x-if="form.valid('email')">
    <span>✅</span>
</template>

<template x-if="form.invalid('email')">
    <span>❌</span>
</template>
```

> [!WARNING]
> 只有在表單輸入欄位變更並收到驗證回應後，才會顯示其有效或無效狀態。

如我們所見，你可以監聽輸入欄位的 `change` 事件，並在使用者與其互動時驗證個別輸入欄位；然而，你可能需要驗證使用者尚未互動的輸入欄位。這在建立「精靈」(wizard) 時很常見，你希望在進入下一步之前，驗證所有可見的輸入欄位，無論使用者是否與其互動過。

為透過 Precognition 實現此目的，你應該呼叫 `validate` 方法，並將你希望驗證的欄位名稱傳遞給 `only` 配置鍵。你可以使用 `onSuccess` 或 `onValidationError` 回呼函式來處理驗證結果：

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

你可以透過檢查表單的 `processing` 屬性來判斷表單提交請求是否正在進行中：

```html
<button :disabled="form.processing">
    Submit
</button>
```

<a name="repopulating-old-form-data"></a>
#### 重新填入舊表單資料

在上面討論的建立使用者範例中，我們使用 Precognition 進行即時驗證；然而，我們是透過傳統的伺服器端表單提交來送出表單。因此，表單應該填入從伺服器端表單提交返回的任何「舊」輸入和驗證錯誤：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '{{ old('name') }}',
        email: '{{ old('email') }}',
    }).setErrors({{ Js::from($errors->messages()) }}),
}">
```

或者，如果你想透過 XHR 提交表單，你可以使用表單的 `submit` 函式，它會返回一個 Axios 請求 Promise：

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
### 配置 Axios

Precognition 驗證函式庫使用 [Axios](https://github.com/axios/axios) HTTP 客戶端向你的應用程式後端發送請求。為方便起見，如果你的應用程式有需要，可以客製化 Axios 實例。例如，當使用 `laravel-precognition-vue` 函式庫時，你可以在應用程式的 `resources/js/app.js` 檔案中為每個發出的請求添加額外的請求標頭：

```js
import { client } from 'laravel-precognition-vue';

client.axios().defaults.headers.common['Authorization'] = authToken;
```

或者，如果你的應用程式已經有一個配置好的 Axios 實例，你可以告訴 Precognition 改用該實例：

```js
import Axios from 'axios';
import { client } from 'laravel-precognition-vue';

window.axios = Axios.create()
window.axios.defaults.headers.common['Authorization'] = authToken;

client.use(window.axios)
```

> [!WARNING]
> Inertia 風格的 Precognition 函式庫僅會將已配置的 Axios 實例用於驗證請求。表單提交將始終由 Inertia 發送。

<a name="customizing-validation-rules"></a>
## 自訂驗證規則

透過使用請求的 `isPrecognitive` 方法，可以自訂預驗證請求期間執行的驗證規則。

例如，在建立使用者表單中，我們可能希望僅在最終表單提交時驗證密碼是「未洩露」的。對於預驗證請求，我們將只驗證密碼是必填的，且至少有 8 個字元。透過使用 `isPrecognitive` 方法，我們可以自訂由我們的 form request 定義的規則：

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

預設情況下，Laravel Precognition 在預驗證請求期間不會上傳或驗證檔案。這確保大型檔案不會不必要地多次上傳。

由於此行為，您應確保您的應用程式[自訂對應的 form request 驗證規則](#customizing-validation-rules)，以指定該欄位僅在完整表單提交時才為必填：

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

如果您想在每個驗證請求中都包含檔案，您可以在客戶端表單實例上呼叫 `validateFiles` 函式：

```js
form.validateFiles();
```

<a name="managing-side-effects"></a>
## 管理副作用

當將 `HandlePrecognitiveRequests` 中介層新增到路由時，您應考慮在_其他_中介層中是否存在應在預驗證請求期間跳過的副作用。

例如，您可能有一個中介層，會遞增每個使用者與應用程式「互動」的總數，但您可能不希望預驗證請求被計為一次互動。為了實現這一點，我們可以在遞增互動計數之前檢查請求的 `isPrecognitive` 方法：

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

如果您想在測試中進行預驗證請求，Laravel 的 `TestCase` 包含了 `withPrecognition` 輔助函式，它將新增 `Precognition` 請求標頭。

此外，如果您想斷言預驗證請求是成功的，例如沒有返回任何驗證錯誤，您可以使用回應上的 `assertSuccessfulPrecognition` 方法：

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
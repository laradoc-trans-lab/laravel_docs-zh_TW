# Prompts

- [介紹](#introduction)
- [安裝](#installation)
- [可用的 Prompts](#available-prompts)
    - [文字](#text)
    - [多行文字](#textarea)
    - [數字](#number)
    - [密碼](#password)
    - [確認](#confirm)
    - [選擇](#select)
    - [多選](#multiselect)
    - [建議](#suggest)
    - [搜尋](#search)
    - [多項搜尋](#multisearch)
    - [暫停](#pause)
- [在驗證前轉換輸入](#transforming-input-before-validation)
- [表單](#forms)
- [資訊訊息](#informational-messages)
- [表格](#tables)
- [Spin](#spin)
- [進度條](#progress)
- [清除終端機](#clear)
- [終端機考量事項](#terminal-considerations)
- [不支援的環境與備用方案 (Fallbacks)](#fallbacks)
- [測試](#testing)

<a name="introduction"></a>
## 介紹

[Laravel Prompts](https://github.com/laravel/prompts) 是一個為您的命令列應用程式增加美觀且使用者友善表單的 PHP 套件，具有類似瀏覽器的功能，包括預留位置文字與驗證。

<img src="https://laravel.com/img/docs/prompts-example.png">

Laravel Prompts 非常適合在您的 [Artisan 命令列指令](/docs/{{version}}/artisan#writing-commands) 中接收使用者輸入，但它也可用於任何命令列 PHP 專案。

> [!NOTE]
> Laravel Prompts 支援 macOS、Linux 以及帶有 WSL 的 Windows。更多資訊請參閱我們關於 [不支援的環境與備用方案 (Fallbacks)](#fallbacks) 的文件。


<a name="installation"></a>
## 安裝

Laravel Prompts 已包含在最新版本的 Laravel 中。

Laravel Prompts 也可以透過 Composer 套件管理員安裝到您的其他 PHP 專案中：

```shell
composer require laravel/prompts
```

<a name="available-prompts"></a>
## 可用的 Prompts


<a name="text"></a>
### 文字

`text` 函式會向使用者詢問指定的內容，接受其輸入並回傳：

```php
use function Laravel\Prompts\text;

$name = text('What is your name?');
```

您也可以包含佔位文字 (Placeholder)、預設值以及資訊提示：

```php
$name = text(
    label: 'What is your name?',
    placeholder: 'E.g. Taylor Otwell',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```


<a name="text-required"></a>
#### 必填值

如果您要求必須輸入內容，可以傳遞 `required` 參數：

```php
$name = text(
    label: 'What is your name?',
    required: true
);
```

如果您想要自訂驗證訊息，也可以傳遞一個字串：

```php
$name = text(
    label: 'What is your name?',
    required: 'Your name is required.'
);
```


<a name="text-validation"></a>
#### 額外驗證

最後，如果您想要執行額外的驗證邏輯，可以傳遞一個 Closure 給 `validate` 參數：

```php
$name = text(
    label: 'What is your name?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

該 Closure 將接收輸入的值，且可以回傳錯誤訊息，或者在驗證通過時回傳 `null`。

或者，您也可以利用 Laravel 驗證器 ([validator](/docs/{{version}}/validation)) 的強大功能。若要這麼做，請為 `validate` 參數提供一個包含屬性名稱與所需驗證規則的陣列：

```php
$name = text(
    label: 'What is your name?',
    validate: ['name' => 'required|max:255|unique:users']
);
```


<a name="textarea"></a>
### 多行文字

`textarea` 函式會向使用者詢問指定的內容，透過多行文字區域接受其輸入並回傳：

```php
use function Laravel\Prompts\textarea;

$story = textarea('Tell me a story.');
```

您也可以包含佔位文字 (Placeholder)、預設值以及資訊提示：

```php
$story = textarea(
    label: 'Tell me a story.',
    placeholder: 'This is a story about...',
    hint: 'This will be displayed on your profile.'
);
```


<a name="textarea-required"></a>
#### 必填值

如果您要求必須輸入內容，可以傳遞 `required` 參數：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: true
);
```

如果您想要自訂驗證訊息，也可以傳遞一個字串：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: 'A story is required.'
);
```


<a name="textarea-validation"></a>
#### 額外驗證

最後，如果您想要執行額外的驗證邏輯，可以傳遞一個 Closure 給 `validate` 參數：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: fn (string $value) => match (true) {
        strlen($value) < 250 => 'The story must be at least 250 characters.',
        strlen($value) > 10000 => 'The story must not exceed 10,000 characters.',
        default => null
    }
);
```

該 Closure 將接收輸入的值，且可以回傳錯誤訊息，或者在驗證通過時回傳 `null`。

或者，您也可以利用 Laravel 驗證器 ([validator](/docs/{{version}}/validation)) 的強大功能。若要這麼做，請為 `validate` 參數提供一個包含屬性名稱與所需驗證規則的陣列：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: ['story' => 'required|max:10000']
);
```


<a name="number"></a>
### 數字

`number` 函式會向使用者詢問指定的內容，接受其數值輸入並回傳。`number` 函式允許使用者使用向上與向下方向鍵來調整數字：

```php
use function Laravel\Prompts\number;

$number = number('How many copies would you like?');
```

您也可以包含佔位文字 (Placeholder)、預設值以及資訊提示：

```php
$name = number(
    label: 'How many copies would you like?',
    placeholder: '5',
    default: 1,
    hint: 'This will be determine how many copies to create.'
);
```


<a name="number-required"></a>
#### 必填值

如果您要求必須輸入內容，可以傳遞 `required` 參數：

```php
$copies = number(
    label: 'How many copies would you like?',
    required: true
);
```

如果您想要自訂驗證訊息，也可以傳遞一個字串：

```php
$copies = number(
    label: 'How many copies would you like?',
    required: 'A number of copies is required.'
);
```


<a name="number-validation"></a>
#### 額外驗證

最後，如果您想要執行額外的驗證邏輯，可以傳遞一個 Closure 給 `validate` 參數：

```php
$copies = number(
    label: 'How many copies would you like?',
    validate: fn (?int $value) => match (true) {
        $value < 1 => 'At least one copy is required.',
        $value > 100 => 'You may not create more than 100 copies.',
        default => null
    }
);
```

該 Closure 將接收輸入的值，且可以回傳錯誤訊息，或者在驗證通過時回傳 `null`。

或者，您也可以利用 Laravel 驗證器 ([validator](/docs/{{version}}/validation)) 的強大功能。若要這麼做，請為 `validate` 參數提供一個包含屬性名稱與所需驗證規則的陣列：

```php
$copies = number(
    label: 'How many copies would you like?',
    validate: ['copies' => 'required|integer|min:1|max:100']
);
```


<a name="password"></a>
### 密碼

`password` 函式與 `text` 函式類似，但使用者的輸入會在終端機中被隱藏。這在詢問密碼等敏感資訊時非常有用：

```php
use function Laravel\Prompts\password;

$password = password('What is your password?');
```

您也可以包含佔位文字 (Placeholder) 以及資訊提示：

```php
$password = password(
    label: 'What is your password?',
    placeholder: 'password',
    hint: 'Minimum 8 characters.'
);
```


<a name="password-required"></a>
#### 必填值

如果您要求必須輸入內容，可以傳遞 `required` 參數：

```php
$password = password(
    label: 'What is your password?',
    required: true
);
```

如果您想要自訂驗證訊息，也可以傳遞一個字串：

```php
$password = password(
    label: 'What is your password?',
    required: 'The password is required.'
);
```


<a name="password-validation"></a>
#### 額外驗證

最後，如果您想要執行額外的驗證邏輯，可以傳遞一個 Closure 給 `validate` 參數：

```php
$password = password(
    label: 'What is your password?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 8 => 'The password must be at least 8 characters.',
        default => null
    }
);
```

該 Closure 將接收輸入的值，且可以回傳錯誤訊息，或者在驗證通過時回傳 `null`。

或者，您也可以利用 Laravel 驗證器 ([validator](/docs/{{version}}/validation)) 的強大功能。若要這麼做，請為 `validate` 參數提供一個包含屬性名稱與所需驗證規則的陣列：

```php
$password = password(
    label: 'What is your password?',
    validate: ['password' => 'min:8']
);
```

<a name="confirm"></a>
### 確認

如果您需要詢問使用者「是或否」的確認，可以使用 `confirm` 函式。使用者可以使用方向鍵或按 `y` 或 `n` 來選擇他們的回應。此函式將回傳 `true` 或 `false`。

```php
use function Laravel\Prompts\confirm;

$confirmed = confirm('Do you accept the terms?');
```

您也可以包含預設值、自定義「是」與「否」標籤的文字，以及資訊提示：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    default: false,
    yes: 'I accept',
    no: 'I decline',
    hint: 'The terms must be accepted to continue.'
);
```


<a name="confirm-required"></a>
#### 強制要求「是」

如有必要，您可以透過傳遞 `required` 參數來要求使用者必須選擇「是」：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: true
);
```

如果您想自定義驗證訊息，也可以傳遞一個字串：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: 'You must accept the terms to continue.'
);
```


<a name="select"></a>
### 選擇

如果您需要使用者從一組預定義的選項中進行選擇，可以使用 `select` 函式：

```php
use function Laravel\Prompts\select;

$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner']
);
```

您也可以指定預設選項與資訊提示：

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    default: 'Owner',
    hint: 'The role may be changed at any time.'
);
```

您也可以向 `options` 參數傳遞一個關聯陣列，以回傳選定的鍵 (Key) 而非其值 (Value)：

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    default: 'owner'
);
```

清單在開始捲動前最多會顯示五個選項。您可以透過傳遞 `scroll` 參數來進行自定義：

```php
$role = select(
    label: 'Which category would you like to assign?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```


<a name="select-validation"></a>
#### 額外驗證

與其他提示函式不同，`select` 函式不接受 `required` 參數，因為不可能什麼都不選。但是，如果您需要呈現某個選項但要防止它被選中，可以向 `validate` 參數傳遞一個閉包 (Closure)：

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    validate: fn (string $value) =>
        $value === 'owner' && User::where('role', 'owner')->exists()
            ? 'An owner already exists.'
            : null
);
```

如果 `options` 參數是一個關聯陣列，則該閉包將接收選定的鍵，否則它將接收選定的值。該閉包可以回傳錯誤訊息，或者如果驗證通過則回傳 `null`。


<a name="multiselect"></a>
### 多選

如果您需要使用者能夠選擇多個選項，可以使用 `multiselect` 函式：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete']
);
```

您也可以指定預設選項與資訊提示：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete'],
    default: ['Read', 'Create'],
    hint: 'Permissions may be updated at any time.'
);
```

您也可以向 `options` 參數傳遞一個關聯陣列，以回傳選定選項的鍵而非其值：

```php
$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    default: ['read', 'create']
);
```

清單在開始捲動前最多會顯示五個選項。您可以透過傳遞 `scroll` 參數來進行自定義：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```


<a name="multiselect-required"></a>
#### 要求輸入值

預設情況下，使用者可以選擇零個或多個選項。您可以傳遞 `required` 參數來強制要求選擇一個或多個選項：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: true
);
```

如果您想自定義驗證訊息，可以向 `required` 參數提供一個字串：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: 'You must select at least one category'
);
```


<a name="multiselect-validation"></a>
#### 額外驗證

如果您需要呈現某個選項但要防止它被選中，可以向 `validate` 參數傳遞一個閉包 (Closure)：

```php
$permissions = multiselect(
    label: 'What permissions should the user have?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    validate: fn (array $values) => ! in_array('read', $values)
        ? 'All users require the read permission.'
        : null
);
```

如果 `options` 參數是一個關聯陣列，則該閉包將接收選定的鍵，否則它將接收選定的值。該閉包可以回傳錯誤訊息，或者如果驗證通過則回傳 `null`。

<a name="suggest"></a>
### 建議

`suggest` 函式可用於為可能的選擇提供自動補全。不論是否有自動補全提示，使用者仍然可以提供任何答案：

```php
use function Laravel\Prompts\suggest;

$name = suggest('What is your name?', ['Taylor', 'Dayle']);
```

或者，您可以將閉包作為 `suggest` 函式的第二個參數傳遞。每當使用者輸入一個字元時，該閉包都會被呼叫。該閉包應接受一個包含使用者目前為止輸入內容的字串參數，並回傳一個用於自動補全的選項陣列：

```php
$name = suggest(
    label: 'What is your name?',
    options: fn ($value) => collect(['Taylor', 'Dayle'])
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
)
```

您也可以包含預留位置文字、預設值以及資訊提示：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    placeholder: 'E.g. Taylor',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```


<a name="suggest-required"></a>
#### 必填值

如果您要求必須輸入值，可以傳遞 `required` 參數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: true
);
```

如果您想自訂驗證訊息，也可以傳遞一個字串：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: 'Your name is required.'
);
```


<a name="suggest-validation"></a>
#### 額外驗證

最後，如果您想執行額外的驗證邏輯，可以將閉包傳遞給 `validate` 參數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

該閉包將接收已輸入的值，並可能回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，您可以利用 Laravel [驗證器](/docs/{{version}}/validation) 的功能。為此，請為 `validate` 參數提供一個包含屬性名稱和所需驗證規則的陣列：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: ['name' => 'required|min:3|max:255']
);
```


<a name="search"></a>
### 搜尋

如果您有很多選項供使用者選擇，`search` 函式允許使用者輸入搜尋查詢來過濾結果，然後再使用方向鍵選擇選項：

```php
use function Laravel\Prompts\search;

$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

該閉包將接收使用者目前為止輸入的文字，且必須回傳一個選項陣列。如果您回傳一個關連陣列，則會回傳所選選項的鍵，否則會回傳其值。

在過濾您打算回傳其值的陣列時，您應該使用 `array_values` 函式或 `values` 集合方法，以確保陣列不會變成關連陣列：

```php
$names = collect(['Taylor', 'Abigail']);

$selected = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => $names
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
        ->values()
        ->all(),
);
```

您也可以包含預留位置文字和資訊提示：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

在列表開始滾動之前，最多會顯示五個選項。您可以透過傳遞 `scroll` 參數來進行自訂：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```


<a name="search-validation"></a>
#### 額外驗證

如果您想執行額外的驗證邏輯，可以將閉包傳遞給 `validate` 參數：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (int|string $value) {
        $user = User::findOrFail($value);

        if ($user->opted_out) {
            return 'This user has opted-out of receiving mail.';
        }
    }
);
```

如果 `options` 閉包回傳一個關連陣列，則該閉包將接收所選的鍵，否則，它將接收所選的值。該閉包可能回傳一條錯誤訊息，或者如果驗證通過則回傳 `null`。

<a name="multisearch"></a>
### 多項搜尋

如果您有許多可搜尋的選項，且需要使用者能夠選擇多個項目，`multisearch` 函式允許使用者輸入搜尋字詞來過濾結果，接著再使用方向鍵與空白鍵來選擇選項：

```php
use function Laravel\Prompts\multisearch;

$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

閉包將會接收使用者目前為止輸入的文字，並必須回傳一個選項陣列。如果您回傳一個關聯陣列，則會回傳所選選項的鍵；否則，將回傳其值。

當過濾一個您打算回傳其值的陣列時，您應該使用 `array_values` 函式或集合的 `values` 方法，以確保該陣列不會變成關聯陣列：

```php
$names = collect(['Taylor', 'Abigail']);

$selected = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => $names
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
        ->values()
        ->all(),
);
```

您也可以包含佔位符文字與資訊提示：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

在列表開始捲動前，最多會顯示五個選項。您可以透過傳遞 `scroll` 引數來進行自定義：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```


<a name="multisearch-required"></a>
#### 強制輸入值

預設情況下，使用者可以選擇零個或多個選項。您可以傳遞 `required` 引數來強制要求選擇一個或多個選項：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: true
);
```

如果您想要自定義驗證訊息，也可以向 `required` 引數傳遞一個字串：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: 'You must select at least one user.'
);
```


<a name="multisearch-validation"></a>
#### 其他驗證

如果您想要執行額外的驗證邏輯，可以向 `validate` 引數傳遞一個閉包：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (array $values) {
        $optedOut = User::whereLike('name', '%a%')->findMany($values);

        if ($optedOut->isNotEmpty()) {
            return $optedOut->pluck('name')->join(', ', ', and ').' have opted out.';
        }
    }
);
```

如果 `options` 閉包回傳一個關聯陣列，則該閉包將接收所選的鍵；否則，它將接收所選的值。該閉包可以回傳錯誤訊息，或者在驗證通過時回傳 `null`。


<a name="pause"></a>
### 暫停

`pause` 函式可用於向使用者顯示資訊文字，並等待他們按下 Enter / Return 鍵以確認繼續：

```php
use function Laravel\Prompts\pause;

pause('Press ENTER to continue.');
```

<a name="transforming-input-before-validation"></a>
## 在驗證前轉換輸入

有時您可能希望在進行驗證之前轉換提示輸入。例如，您可能希望移除所提供的字串中的空白字元。為了實現這一點，許多提示函數都提供了一個 `transform` 參數，該參數接受一個閉包 (Closure)：

```php
$name = text(
    label: 'What is your name?',
    transform: fn (string $value) => trim($value),
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```


<a name="forms"></a>
## 表單

通常，您會有複數個提示依序顯示，以便在執行後續動作之前收集資訊。您可以使用 `form` 函數來建立一組讓使用者完成的提示：

```php
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true)
    ->password('What is your password?', validate: ['password' => 'min:8'])
    ->confirm('Do you accept the terms?')
    ->submit();
```

`submit` 方法將會回傳一個以數字索引的陣列，其中包含來自表單提示的所有回應。然而，您可以透過 `name` 參數為每個提示提供一個名稱。提供名稱後，即可透過該名稱存取具名提示的回應：

```php
use App\Models\User;
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->password(
        label: 'What is your password?',
        validate: ['password' => 'min:8'],
        name: 'password'
    )
    ->confirm('Do you accept the terms?')
    ->submit();

User::create([
    'name' => $responses['name'],
    'password' => $responses['password'],
]);
```

使用 `form` 函數的主要優點是使用者能夠使用 `CTRL + U` 返回表單中的前一個提示。這讓使用者可以修正錯誤或更改選擇，而無需取消並重新啟動整個表單。

如果您需要對表單中的提示進行更細粒度的控制，您可以呼叫 `add` 方法，而不是直接呼叫其中一個提示函數。`add` 方法會接收使用者之前提供的所有回應：

```php
use function Laravel\Prompts\form;
use function Laravel\Prompts\outro;
use function Laravel\Prompts\text;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->add(function ($responses) {
        return text("How old are you, {$responses['name']}?");
    }, name: 'age')
    ->submit();

outro("Your name is {$responses['name']} and you are {$responses['age']} years old.");
```


<a name="informational-messages"></a>
## 資訊訊息

`note`、`info`、`warning`、`error` 和 `alert` 函數可用於顯示資訊訊息：

```php
use function Laravel\Prompts\info;

info('Package installed successfully.');
```


<a name="tables"></a>
## 表格

`table` 函數可以輕鬆顯示多列與多欄的資料。您只需要提供欄位名稱以及表格的資料即可：

```php
use function Laravel\Prompts\table;

table(
    headers: ['Name', 'Email'],
    rows: User::all(['name', 'email'])->toArray()
);
```


<a name="spin"></a>
## Spin

`spin` 函數在執行指定的回呼時，會顯示一個旋轉圖示 (Spinner) 以及一個可選的訊息。它用於指示正在進行的程序，並在完成時回傳回呼的結果：

```php
use function Laravel\Prompts\spin;

$response = spin(
    callback: fn () => Http::get('http://example.com'),
    message: 'Fetching response...'
);
```

> [!WARNING]
> `spin` 函數需要 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能來讓旋轉圖示產生動畫。當此擴充功能不可用時，將改為顯示靜態版本的圖示。


<a name="progress"></a>
## 進度條

對於長時間運行的任務，顯示進度條以告知使用者任務的完成進度會很有幫助。使用 `progress` 函數，Laravel 會顯示一個進度條，並針對給定可迭代值的每次迭代推進其進度：

```php
use function Laravel\Prompts\progress;

$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: fn ($user) => $this->performTask($user)
);
```

`progress` 函數的作用類似於 map 函數，並將回傳一個包含每次回呼迭代回傳值的陣列。

回呼也可以接受 `Laravel\Prompts\Progress` 實例，讓您可以在每次迭代中修改標籤與提示：

```php
$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: function ($user, $progress) {
        $progress
            ->label("Updating {$user->name}")
            ->hint("Created on {$user->created_at}");

        return $this->performTask($user);
    },
    hint: 'This may take some time.'
);
```

有時，您可能需要對進度條的推進方式進行更多手動控制。首先，定義程序將迭代的總步驟數。接著，在處理完每個項目後，透過 `advance` 方法推進進度條：

```php
$progress = progress(label: 'Updating users', steps: 10);

$users = User::all();

$progress->start();

foreach ($users as $user) {
    $this->performTask($user);

    $progress->advance();
}

$progress->finish();
```


<a name="clear"></a>
## 清除終端機

`clear` 函數可用於清除使用者的終端機：

```php
use function Laravel\Prompts\clear;

clear();
```


<a name="terminal-considerations"></a>
## 終端機考量事項


<a name="terminal-width"></a>
#### 終端機寬度

如果任何標籤、選項或驗證訊息的長度超過了使用者終端機中的「欄位 (Columns)」數，它將會被自動截斷以符合寬度。如果您的使用者可能使用較窄的終端機，請考慮縮短這些字串的長度。通常安全的最高長度為 74 個字元，以支援 80 個字元的終端機。


<a name="terminal-height"></a>
#### 終端機高度

對於任何接受 `scroll` 參數的提示，設定的值將會自動縮小以符合使用者終端機的高度，包含留給驗證訊息的空間。

<a name="fallbacks"></a>
## 不支援的環境與備用方案 (Fallbacks)

Laravel Prompts 支援 macOS、Linux 以及包含 WSL 的 Windows。由於 Windows 版本 PHP 的限制，目前無法在 WSL 以外的 Windows 環境中使用 Laravel Prompts。

因此，Laravel Prompts 支援回退 (Fallback) 到替代實作方式，例如 [Symfony Console Question Helper](https://symfony.com/doc/current/components/console/helpers/questionhelper.html)。

> [!NOTE]
> 當在 Laravel 框架中使用 Laravel Prompts 時，已為每個 prompt 設定好備用方案，並會在不支援的環境中自動啟用。


<a name="fallback-conditions"></a>
#### 備用條件

若您未使用 Laravel 或需要自訂啟用備用行為的時機，可以傳遞布林值給 `Prompt` 類別的 `fallbackWhen` 靜態方法：

```php
use Laravel\Prompts\Prompt;

Prompt::fallbackWhen(
    ! $input->isInteractive() || windows_os() || app()->runningUnitTests()
);
```


<a name="fallback-behavior"></a>
#### 備用行為

若您未使用 Laravel 或需要自訂備用行為，可以傳遞 Closure 給每個 prompt 類別的 `fallbackUsing` 靜態方法：

```php
use Laravel\Prompts\TextPrompt;
use Symfony\Component\Console\Question\Question;
use Symfony\Component\Console\Style\SymfonyStyle;

TextPrompt::fallbackUsing(function (TextPrompt $prompt) use ($input, $output) {
    $question = (new Question($prompt->label, $prompt->default ?: null))
        ->setValidator(function ($answer) use ($prompt) {
            if ($prompt->required && $answer === null) {
                throw new \RuntimeException(
                    is_string($prompt->required) ? $prompt->required : 'Required.'
                );
            }

            if ($prompt->validate) {
                $error = ($prompt->validate)($answer ?? '');

                if ($error) {
                    throw new \RuntimeException($error);
                }
            }

            return $answer;
        });

    return (new SymfonyStyle($input, $output))
        ->askQuestion($question);
});
```

必須為每個 prompt 類別個別設定備用方案。Closure 會接收該 prompt 類別的執行個體，並必須回傳該 prompt 適用的型別。


<a name="testing"></a>
## 測試

Laravel 提供了多種方法來測試您的指令是否顯示了預期的 Prompt 訊息：

```php tab=Pest
test('report generation', function () {
    $this->artisan('report:generate')
        ->expectsPromptsInfo('Welcome to the application!')
        ->expectsPromptsWarning('This action cannot be undone')
        ->expectsPromptsError('Something went wrong')
        ->expectsPromptsAlert('Important notice!')
        ->expectsPromptsIntro('Starting process...')
        ->expectsPromptsOutro('Process completed!')
        ->expectsPromptsTable(
            headers: ['Name', 'Email'],
            rows: [
                ['Taylor Otwell', 'taylor@example.com'],
                ['Jason Beggs', 'jason@example.com'],
            ]
        )
        ->assertExitCode(0);
});
```

```php tab=PHPUnit
public function test_report_generation(): void
{
    $this->artisan('report:generate')
        ->expectsPromptsInfo('Welcome to the application!')
        ->expectsPromptsWarning('This action cannot be undone')
        ->expectsPromptsError('Something went wrong')
        ->expectsPromptsAlert('Important notice!')
        ->expectsPromptsIntro('Starting process...')
        ->expectsPromptsOutro('Process completed!')
        ->expectsPromptsTable(
            headers: ['Name', 'Email'],
            rows: [
                ['Taylor Otwell', 'taylor@example.com'],
                ['Jason Beggs', 'jason@example.com'],
            ]
        )
        ->assertExitCode(0);
}
```
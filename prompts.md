# Prompts

- [簡介](#introduction)
- [安裝](#installation)
- [可用的 Prompts](#available-prompts)
    - [文字 (Text)](#text)
    - [文字區塊 (Textarea)](#textarea)
    - [數字 (Number)](#number)
    - [密碼 (Password)](#password)
    - [確認 (Confirm)](#confirm)
    - [單選 (Select)](#select)
    - [多選 (Multi-select)](#multiselect)
    - [建議 (Suggest)](#suggest)
    - [搜尋 (Search)](#search)
    - [多重搜尋 (Multi-search)](#multisearch)
    - [暫停 (Pause)](#pause)
    - [自動完成 (Autocomplete)](#autocomplete)
- [在驗證前轉換輸入](#transforming-input-before-validation)
- [表單 (Forms)](#forms)
- [資訊訊息](#informational-messages)
- [表格 (Tables)](#tables)
- [旋轉指示器 (Spin)](#spin)
- [進度條 (Progress Bar)](#progress)
- [任務 (Task)](#task)
- [串流 (Stream)](#stream)
- [終端機標題](#terminal-title)
- [清除終端機內容](#clear)
- [終端機注意事項](#terminal-considerations)
- [不支援的環境與備案](#fallbacks)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Prompts](https://github.com/laravel/prompts) 是一個 PHP 套件，用於為您的命令列應用程式加入美觀且使用者友好的表單，並具備類似瀏覽器的功能，包括佔位字元文字與驗證。

<img src="https://laravel.com/img/docs/prompts-example.png">

Laravel Prompts 非常適合在您的 [Artisan 主控台指令](/docs/{{version}}/artisan#writing-commands)中接收使用者輸入，但它也可以用於任何命令列 PHP 專案。

> [!NOTE]
> Laravel Prompts 支援 macOS、Linux 以及搭配 WSL 的 Windows。如需更多資訊，請參閱我們關於 [不支援的環境與備案](#fallbacks) 的文件。

<a name="installation"></a>
## 安裝

Laravel Prompts 已包含在最新版本的 Laravel 中。

您也可以使用 Composer 套件管理員將 Laravel Prompts 安裝到其他的 PHP 專案中：

```shell
composer require laravel/prompts
```

<a name="available-prompts"></a>
## 可用的 Prompts


<a name="text"></a>
### 文字 (Text)

`text` 函式會向使用者顯示指定的提問，接收其輸入內容並回傳：

```php
use function Laravel\Prompts\text;

$name = text('What is your name?');
```

你也可以包含提示文字 (Placeholder)、預設值以及資訊提示 (Hint)：

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

如果你要求必須輸入值，可以傳遞 `required` 引數：

```php
$name = text(
    label: 'What is your name?',
    required: true
);
```

如果你想要自訂驗證訊息，也可以傳遞一個字串：

```php
$name = text(
    label: 'What is your name?',
    required: 'Your name is required.'
);
```


<a name="text-validation"></a>
#### 額外驗證

最後，如果你想要執行額外的驗證邏輯，可以傳遞一個閉包 (Closure) 給 `validate` 引數：

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

此閉包將接收已輸入的值，並可回傳錯誤訊息，若驗證通過則回傳 `null`。

或者，你也可以利用 Laravel [驗證器 (Validator)](/docs/{{version}}/validation) 的強大功能。為此，請向 `validate` 引數提供一個包含屬性名稱及其驗證規則的陣列：

```php
$name = text(
    label: 'What is your name?',
    validate: ['name' => 'required|max:255|unique:users']
);
```


<a name="textarea"></a>
### 文字區塊 (Textarea)

`textarea` 函式會向使用者顯示指定的提問，透過多行文字區塊接收其輸入內容並回傳：

```php
use function Laravel\Prompts\textarea;

$story = textarea('Tell me a story.');
```

你也可以包含提示文字、預設值以及資訊提示：

```php
$story = textarea(
    label: 'Tell me a story.',
    placeholder: 'This is a story about...',
    hint: 'This will be displayed on your profile.'
);
```


<a name="textarea-required"></a>
#### 必填值

如果你要求必須輸入內容，可以傳遞 `required` 引數：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: true
);
```

如果你想要自訂驗證訊息，也可以傳遞一個字串：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: 'A story is required.'
);
```


<a name="textarea-validation"></a>
#### 額外驗證

最後，如果你想要執行額外的驗證邏輯，可以傳遞一個閉包給 `validate` 引數：

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

此閉包將接收已輸入的值，並可回傳錯誤訊息，若驗證通過則回傳 `null`。

或者，你也可以利用 Laravel [驗證器 (Validator)](/docs/{{version}}/validation) 的強大功能。為此，請向 `validate` 引數提供一個包含屬性名稱及其驗證規則的陣列：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: ['story' => 'required|max:10000']
);
```


<a name="number"></a>
### 數字 (Number)

`number` 函式會向使用者顯示指定的提問，接收其數字輸入並回傳。`number` 函式允許使用者使用上下方向鍵來操作數字：

```php
use function Laravel\Prompts\number;

$number = number('How many copies would you like?');
```

你也可以包含提示文字、預設值以及資訊提示：

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

如果你要求必須輸入數字，可以傳遞 `required` 引數：

```php
$copies = number(
    label: 'How many copies would you like?',
    required: true
);
```

如果你想要自訂驗證訊息，也可以傳遞一個字串：

```php
$copies = number(
    label: 'How many copies would you like?',
    required: 'A number of copies is required.'
);
```


<a name="number-validation"></a>
#### 額外驗證

最後，如果你想要執行額外的驗證邏輯，可以傳遞一個閉包給 `validate` 引數：

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

此閉包將接收已輸入的值，並可回傳錯誤訊息，若驗證通過則回傳 `null`。

或者，你也可以利用 Laravel [驗證器 (Validator)](/docs/{{version}}/validation) 的強大功能。為此，請向 `validate` 引數提供一個包含屬性名稱及其驗證規則的陣列：

```php
$copies = number(
    label: 'How many copies would you like?',
    validate: ['copies' => 'required|integer|min:1|max:100']
);
```


<a name="password"></a>
### 密碼 (Password)

`password` 函式與 `text` 函式類似，但在終端機輸入時會遮蔽使用者的輸入內容。這在要求輸入密碼等敏感資訊時非常有用：

```php
use function Laravel\Prompts\password;

$password = password('What is your password?');
```

你也可以包含提示文字以及資訊提示：

```php
$password = password(
    label: 'What is your password?',
    placeholder: 'password',
    hint: 'Minimum 8 characters.'
);
```


<a name="password-required"></a>
#### 必填值

如果你要求必須輸入內容，可以傳遞 `required` 引數：

```php
$password = password(
    label: 'What is your password?',
    required: true
);
```

如果你想要自訂驗證訊息，也可以傳遞一個字串：

```php
$password = password(
    label: 'What is your password?',
    required: 'The password is required.'
);
```


<a name="password-validation"></a>
#### 額外驗證

最後，如果你想要執行額外的驗證邏輯，可以傳遞一個閉包給 `validate` 引數：

```php
$password = password(
    label: 'What is your password?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 8 => 'The password must be at least 8 characters.',
        default => null
    }
);
```

此閉包將接收已輸入的值，並可回傳錯誤訊息，若驗證通過則回傳 `null`。

或者，你也可以利用 Laravel [驗證器 (Validator)](/docs/{{version}}/validation) 的強大功能。為此，請向 `validate` 引數提供一個包含屬性名稱及其驗證規則的陣列：

```php
$password = password(
    label: 'What is your password?',
    validate: ['password' => 'min:8']
);
```

<a name="confirm"></a>
### 確認 (Confirm)

如果您需要詢問使用者「是或否」的確認，可以使用 `confirm` 函數。使用者可以使用方向鍵或按下 `y` 或 `n` 來選擇他們的回應。此函數將回傳 `true` 或 `false`。

```php
use function Laravel\Prompts\confirm;

$confirmed = confirm('Do you accept the terms?');
```

您也可以包含預設值、自定義「是」與「否」標籤的文字，以及說明提示：

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

如有必要，您可以透過傳遞 `required` 引數來要求使用者必須選擇「是」：

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
### 單選 (Select)

如果您需要使用者從一組預定義的選項中進行選擇，可以使用 `select` 函數：

```php
use function Laravel\Prompts\select;

$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner']
);
```

您也可以指定預設選項和說明提示：

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    default: 'Owner',
    hint: 'The role may be changed at any time.'
);
```

您也可以將關聯陣列傳遞給 `options` 引數，以便讓函數回傳選取的鍵名 (Key) 而非其數值 (Value)：

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

列表在開始捲動之前最多會顯示五個選項。您可以透過傳遞 `scroll` 引數來調整此設定：

```php
$role = select(
    label: 'Which category would you like to assign?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```


<a name="select-info"></a>
#### 次要資訊

`info` 引數可用於顯示目前標註選項的額外資訊。當提供閉包 (Closure) 時，它將接收目前標註選項的數值，並應回傳一個字串或 `null`：

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    info: fn (string $value) => match ($value) {
        'member' => 'Can view and comment.',
        'contributor' => 'Can view, comment, and edit.',
        'owner' => 'Full access to all resources.',
        default => null,
    }
);
```

如果資訊不依賴於標註的選項，您也可以向 `info` 引數傳遞一個靜態字串：

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    info: 'The role may be changed at any time.'
);
```


<a name="select-validation"></a>
#### 額外驗證

與其他 Prompt 函數不同，`select` 函數不接受 `required` 引數，因為不可能什麼都不選。但是，如果您需要呈現一個選項但要防止它被選取，可以將閉包傳遞給 `validate` 引數：

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

如果 `options` 引數是一個關聯陣列，則閉包將接收選取的鍵名，否則它將接收選取的數值。閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。


<a name="multiselect"></a>
### 多選 (Multi-select)

如果您需要使用者能夠選擇多個選項，可以使用 `multiselect` 函數：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete']
);
```

您也可以指定預設選項和說明提示：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete'],
    default: ['Read', 'Create'],
    hint: 'Permissions may be updated at any time.'
);
```

您也可以將關聯陣列傳遞給 `options` 引數，以回傳所選選項的鍵名而非其數值：

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

列表在開始捲動之前最多會顯示五個選項。您可以透過傳遞 `scroll` 引數來調整此設定：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```


<a name="multiselect-info"></a>
#### 次要資訊

`info` 引數可用於顯示目前標註選項的額外資訊。當提供閉包時，它將接收目前標註選項的數值，並應回傳一個字串或 `null`：

```php
$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    info: fn (string $value) => match ($value) {
        'read' => 'View resources and their properties.',
        'create' => 'Create new resources.',
        'update' => 'Modify existing resources.',
        'delete' => 'Permanently remove resources.',
        default => null,
    }
);
```


<a name="multiselect-required"></a>
#### 強制要求數值

預設情況下，使用者可以選擇零個或多個選項。您可以傳遞 `required` 引數來強制要求選取一個或多個選項：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: true
);
```

如果您想自定義驗證訊息，可以向 `required` 引數提供一個字串：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: 'You must select at least one category'
);
```


<a name="multiselect-validation"></a>
#### 額外驗證

如果您需要呈現一個選項但要防止它被選取，可以將閉包傳遞給 `validate` 引數：

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

如果 `options` 引數是一個關聯陣列，則閉包將接收選取的鍵名，否則它將接收選取的數值。閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。

<a name="suggest"></a>
### 建議 (Suggest)

`suggest` 函式可用於為可能的選項提供自動完成功能。使用者仍然可以提供任何答案，不受自動完成提示的限制：

```php
use function Laravel\Prompts\suggest;

$name = suggest('What is your name?', ['Taylor', 'Dayle']);
```

或者，您可以將閉包作為 `suggest` 函式的第二個引數傳入。每當使用者輸入字元時，該閉包都會被呼叫。該閉包應接收一個包含目前使用者輸入內容的字串參數，並回傳一個用於自動完成的選項陣列：

```php
$name = suggest(
    label: 'What is your name?',
    options: fn ($value) => collect(['Taylor', 'Dayle'])
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
)
```

您也可以包含佔位字元 (Placeholder)、預設值以及資訊提示 (Hint)：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    placeholder: 'E.g. Taylor',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

<a name="suggest-info"></a>
#### 次要資訊

`info` 引數可用於顯示目前所選選項的額外資訊。當提供閉包時，它將接收目前選中選項的值，並應回傳一個字串或 `null`：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    info: fn (string $value) => match ($value) {
        'Taylor' => 'Administrator',
        'Dayle' => 'Contributor',
        default => null,
    }
);
```

<a name="suggest-required"></a>
#### 必填值

如果您要求必須輸入值，可以傳入 `required` 引數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: true
);
```

如果您想自訂驗證訊息，也可以傳入一個字串：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: 'Your name is required.'
);
```

<a name="suggest-validation"></a>
#### 額外驗證

最後，如果您想執行額外的驗證邏輯，可以將閉包傳給 `validate` 引數：

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

該閉包將接收輸入的值，並可以回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，您也可以利用 Laravel [驗證器 (Validator)](/docs/{{version}}/validation) 的強大功能。若要這樣做，請為 `validate` 引數提供一個陣列，其中包含屬性名稱與所需的驗證規則：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: ['name' => 'required|min:3|max:255']
);
```

<a name="search"></a>
### 搜尋 (Search)

如果您有許多選項供使用者選擇，`search` 函式允許使用者輸入搜尋查詢來篩選結果，然後再使用方向鍵選擇選項：

```php
use function Laravel\Prompts\search;

$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

該閉包將接收目前使用者輸入的文字，且必須回傳一個選項陣列。如果您回傳的是關聯陣列，則會回傳所選選項的鍵 (Key)，否則會回傳其值 (Value)。

當篩選一個您打算回傳其值的陣列時，您應該使用 `array_values` 函式或集合 (Collection) 的 `values` 方法，以確保該陣列不會變成關聯陣列：

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

您也可以包含佔位字元以及資訊提示：

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

在列表開始捲動之前，最多會顯示五個選項。您可以透過傳入 `scroll` 引數來自訂此設定：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

<a name="search-info"></a>
#### 次要資訊

`info` 引數可用於顯示目前選中選項的額外資訊。當提供閉包時，它將接收目前選中選項的值，並應回傳一個字串或 `null`：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    info: fn (int $userId) => User::find($userId)?->email
);
```

<a name="search-validation"></a>
#### 額外驗證

如果您想執行額外的驗證邏輯，可以將閉包傳給 `validate` 引數：

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

如果 `options` 閉包回傳的是關聯陣列，則該閉包將接收所選的鍵，否則它將接收所選的值。該閉包可以回傳錯誤訊息，或者在驗證通過時回傳 `null`。

<a name="multisearch"></a>
### 多重搜尋 (Multi-search)

如果您有許多可供搜尋的選項，且需要讓使用者能夠選擇多個項目，`multisearch` 函式可讓使用者輸入搜尋查詢來過濾結果，接著再使用方向鍵與空白鍵來選取選項：

```php
use function Laravel\Prompts\multisearch;

$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

閉包將會接收使用者目前輸入的文字，且必須回傳一個選項陣列。如果您回傳的是關聯陣列，則會回傳選取選項的「鍵 (Key)」，否則將回傳其「值 (Value)」。

當您過濾一個打算回傳「值」的陣列時，應使用 `array_values` 函式或集合的 `values` 方法，以確保該陣列不會變成關聯陣列：

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

您也可以加入預留文字 (Placeholder) 與資訊提示 (Hint)：

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


<a name="multisearch-info"></a>
#### 輔助資訊

`info` 引數可用於顯示目前反白選項的額外資訊。當提供閉包時，它將接收目前反白選項的值，並應回傳一個字串或 `null`：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    info: fn (int $userId) => User::find($userId)?->email
);
```


<a name="multisearch-required"></a>
#### 強制填入值

預設情況下，使用者可以選擇零個或多個選項。您可以傳遞 `required` 引數來強制要求至少選取一個以上的選項：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: true
);
```

如果您想自定義驗證訊息，也可以向 `required` 引數傳遞一個字串：

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
#### 額外驗證

如果您想要執行額外的驗證邏輯，可以將閉包傳遞給 `validate` 引數：

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

如果 `options` 閉包回傳的是關聯陣列，則該閉包將接收選取的「鍵」；否則，它將接收選取的「值」。閉包可以回傳錯誤訊息，若驗證通過則回傳 `null`。


<a name="pause"></a>
### 暫停 (Pause)

`pause` 函式可用於向使用者顯示資訊文字，並等待他們按下 Enter / Return 鍵以確認繼續操作：

```php
use function Laravel\Prompts\pause;

pause('Press ENTER to continue.');
```


<a name="autocomplete"></a>
### 自動完成 (Autocomplete)

`autocomplete` 函式可用於為可能的選項提供行內自動完成。當使用者輸入時，符合輸入內容的建議將以虛擬文字 (Ghost text) 的形式出現，使用者可以按下 `Tab` 鍵或向右方向鍵來接受建議：

```php
use function Laravel\Prompts\autocomplete;

$name = autocomplete(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle', 'Jess', 'Nuno', 'Tim']
);
```

您也可以加入預留文字、預設值以及資訊提示：

```php
$name = autocomplete(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle', 'Jess', 'Nuno', 'Tim'],
    placeholder: 'E.g. Taylor',
    default: $user?->name,
    hint: 'Use tab to accept, up/down to cycle.'
);
```


<a name="autocomplete-closure"></a>
#### 動態選項

您也可以傳遞一個閉包，根據使用者的輸入動態產生選項。每當使用者輸入一個字元時，該閉包都會被呼叫，並應回傳一個用於自動完成的選項陣列：

```php
$file = autocomplete(
    label: 'Which file?',
    options: fn (string $value) => collect($files)
        ->filter(fn ($file) => str_starts_with(strtolower($file), strtolower($value)))
        ->values()
        ->all(),
);
```


<a name="autocomplete-required"></a>
#### 強制填入值

如果您要求必須輸入內容，可以傳遞 `required` 引數：

```php
$name = autocomplete(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle', 'Jess', 'Nuno', 'Tim'],
    required: true
);
```

如果您想自定義驗證訊息，也可以傳遞一個字串：

```php
$name = autocomplete(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle', 'Jess', 'Nuno', 'Tim'],
    required: 'Your name is required.'
);
```


<a name="autocomplete-validation"></a>
#### 額外驗證

最後，如果您想要執行額外的驗證邏輯，可以將閉包傳遞給 `validate` 引數：

```php
$name = autocomplete(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle', 'Jess', 'Nuno', 'Tim'],
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

該閉包將接收已輸入的值，並可以回傳錯誤訊息，若驗證通過則回傳 `null`。

<a name="transforming-input-before-validation"></a>
## 在驗證前轉換輸入

有時您可能希望在進行驗證之前先轉換提示輸入。例如，您可能想要移除輸入字串中的空白。為了實現這一點，許多提示函數都提供了一個 `transform` 參數，它接受一個閉包 (Closure)：

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
## 表單 (Forms)

通常，您會有一連串按順序顯示的提示，用於在執行後續動作之前收集資訊。您可以使用 `form` 函數來建立一組供使用者填寫的提示：

```php
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true)
    ->password('What is your password?', validate: ['password' => 'min:8'])
    ->confirm('Do you accept the terms?')
    ->submit();
```

`submit` 方法將會回傳一個以數字索引的陣列，其中包含了表單提示的所有回應。然而，您也可以透過 `name` 參數為每個提示提供名稱。提供名稱後，即可透過該名稱存取特定提示的回應：

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

使用 `form` 函數的主要優點是使用者可以使用 `CTRL + U` 返回表單中的前一個提示。這讓使用者可以修正錯誤或更改選擇，而不需要取消並重頭開始整個表單。

如果您需要對表單中的提示進行更細緻的控制，可以調用 `add` 方法，而不是直接呼叫提示函數。`add` 方法會接收到使用者之前提供的所有回應：

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

您可以利用 `note`、`info`、`warning`、`error` 和 `alert` 函數來顯示資訊訊息：

```php
use function Laravel\Prompts\info;

info('Package installed successfully.');
```

<a name="tables"></a>
## 表格 (Tables)

`table` 函數可以輕鬆地顯示多列與多欄的資料。您只需要提供欄位名稱和表格資料即可：

```php
use function Laravel\Prompts\table;

table(
    headers: ['Name', 'Email'],
    rows: User::all(['name', 'email'])->toArray()
);
```

<a name="spin"></a>
## 旋轉指示器 (Spin)

`spin` 函數會在執行指定的閉包回呼時顯示一個旋轉指示器 (Spinner) 以及一個可選訊息。它用於指示正在進行中的行程 (Processes)，並在完成後回傳該回呼的結果：

```php
use function Laravel\Prompts\spin;

$response = spin(
    callback: fn () => Http::get('http://example.com'),
    message: 'Fetching response...'
);
```

> [!WARNING]
> `spin` 函數需要 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能來呈現旋轉動畫。如果該擴充功能不可用，則會改為顯示靜態版本的旋轉圖示。

<a name="progress"></a>
## 進度條 (Progress Bar)

對於執行時間較長的任務，顯示一個告知使用者任務完成度的進度條會很有幫助。透過 `progress` 函數，Laravel 會顯示一個進度條，並在對給定的可反覆運算 (Iterable) 值進行每次迭代時推進其進度：

```php
use function Laravel\Prompts\progress;

$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: fn ($user) => $this->performTask($user)
);
```

`progress` 函數的作用類似於 map 函數，並會回傳一個包含每次回呼迭代後之回傳值的陣列。

回呼也可以接受 `Laravel\Prompts\Progress` 實例，讓您可以在每次迭代中修改標籤 (Label) 和提示 (Hint)：

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

有時，您可能需要更手動地控制進度條的推進方式。首先，定義該行程將迭代的總步驟數。接著，在處理完每個項目後，透過 `advance` 方法來推進進度條：

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

<a name="task"></a>
## 任務 (Task)

`task` 函式在執行給定的回呼 (Callback) 時，會顯示一個帶有旋轉指示器與滾動即時輸出區域的有標籤任務。它非常適合用來包裹耗時較長的程序，例如依賴安裝或佈署腳本，讓使用者能即時瞭解當前的進度：

```php
use function Laravel\Prompts\task;

task(
    label: 'Installing dependencies',
    callback: function ($logger) {
        // Long-running process...
    }
);
```

回呼會接收一個 `Logger` 實例，您可以使用它在任務的輸出區域顯示日誌行、狀態訊息和串流文字。

> [!WARNING]
> `task` 函式需要 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能讓旋轉指示器動起來。當此擴充功能不可用時，將改為顯示靜態版本的任務圖示。

<a name="task-logging"></a>
#### 記錄日誌行 (Logging Lines)

`line` 方法可將單行日誌寫入任務的滾動輸出區域：

```php
task(
    label: 'Installing dependencies',
    callback: function ($logger) {
        $logger->line('Resolving packages...');
        // ...
        $logger->line('Downloading laravel/framework');
        // ...
    }
);
```

<a name="task-status-messages"></a>
#### 狀態訊息

您可以使用 `success`、`warning` 與 `error` 方法來顯示狀態訊息。這些訊息會以醒目且固定的方式出現在滾動日誌區域的上方：

```php
task(
    label: 'Deploying application',
    callback: function ($logger) {
        $logger->line('Pulling latest changes...');
        // ...
        $logger->success('Changes pulled!');

        $logger->line('Running migrations...');
        // ...
        $logger->warning('No new migrations to run.');

        $logger->line('Clearing cache...');
        // ...
        $logger->success('Cache cleared!');
    }
);
```

<a name="task-label"></a>
#### 更新標籤

`label` 方法允許您在任務執行期間更新其標籤內容：

```php
task(
    label: 'Starting deployment...',
    callback: function ($logger) {
        $logger->label('Pulling latest changes...');
        // ...
        $logger->label('Running migrations...');
        // ...
        $logger->label('Clearing cache...');
        // ...
    }
);
```

<a name="task-sub-label"></a>
#### 顯示子標籤 (Sub-Label)

`subLabel` 方法會在任務的主標籤下方顯示一行較暗的文字，這對於傳達當前正在進行的步驟等臨時狀態非常有用。傳入空字串即可清除子標籤：

```php
task(
    label: 'Deploying',
    callback: function ($logger) {
        $logger->subLabel('Building assets...');
        // ...
        $logger->subLabel('Running migrations...');
        // ...
        $logger->subLabel('');
    }
);
```

您也可以透過 `subLabel` 引數提供初始的子標籤：

```php
task(
    label: 'Deploying',
    callback: function ($logger) {
        // ...
    },
    subLabel: 'Preparing...'
);
```

<a name="task-streaming"></a>
#### 串流文字

對於會增量產生輸出的程序（例如 AI 生成的內容），`partial` 方法允許您逐字或逐塊地串流顯示文字。串流結束後，請呼叫 `commitPartial` 來完成最終的輸出：

```php
task(
    label: 'Generating response...',
    callback: function ($logger) {
        foreach ($words as $word) {
            $logger->partial($word . ' ');
        }

        $logger->commitPartial();
    }
);
```

<a name="task-limit"></a>
#### 自訂輸出限制

預設情況下，任務最多顯示 10 行滾動輸出。您可以透過 `limit` 引數來自訂此限制：

```php
task(
    label: 'Installing dependencies',
    callback: function ($logger) {
        // ...
    },
    limit: 20
);
```

<a name="task-keep-summary"></a>
#### 保留摘要

預設情況下，任務的回呼結束後會擦除輸出內容。如果您希望在任務完成後將狀態訊息保留在螢幕上，可以傳入 `keepSummary` 引數：

```php
task(
    label: 'Deploying',
    callback: function ($logger) {
        $logger->success('Assets built');
        // ...
        $logger->success('Migrations complete');
    },
    keepSummary: true,
);
```

<a name="stream"></a>
## 串流 (Stream)

`stream` 函式可在終端機中顯示串流文字，非常適合顯示 AI 生成的內容或任何增量到達的文字：

```php
use function Laravel\Prompts\stream;

$stream = stream();

foreach ($words as $word) {
    $stream->append($word . ' ');
    usleep(25_000); // Simulate delay between chunks...
}

$stream->close();
```

`append` 方法會將文字加入串流，並以漸進淡入的效果呈現。當所有內容都串流完畢時，請呼叫 `close` 方法來完成輸出並恢復游標位置。

<a name="terminal-title"></a>
## 終端機標題

`title` 函式會更新使用者終端機視窗或分頁的標題：

```php
use function Laravel\Prompts\title;

title('Installing Dependencies');
```

若要將終端機標題重設回預設值，請傳入一個空字串：

```php
title('');
```

<a name="clear"></a>
## 清除終端機內容

`clear` 函式可用來清除使用者終端機的內容：

```php
use function Laravel\Prompts\clear;

clear();
```

<a name="terminal-considerations"></a>
## 終端機注意事項

<a name="terminal-width"></a>
#### 終端機寬度

如果任何標籤、選項或驗證訊息的長度超過使用者終端機的「欄位 (Columns)」數，內容將會被自動截斷以符合寬度。如果您的使用者可能會使用較窄的終端機，請考慮縮短這些字串的長度。通常安全的最高長度為 74 個字元，以支援 80 字元寬的終端機。

<a name="terminal-height"></a>
#### 終端機高度

對於任何接受 `scroll` 引數的 Prompt，設定的值會自動根據使用者終端機的高度進行縮減，並為驗證訊息保留空間。

<a name="fallbacks"></a>
## 不支援的環境與備案

Laravel Prompts 支援 macOS、Linux 以及支援 WSL 的 Windows。由於 PHP 的 Windows 版本限制，目前無法在 WSL 之外的 Windows 環境中使用 Laravel Prompts。

因此，Laravel Prompts 支援備案至替代實作，例如 [Symfony Console Question Helper](https://symfony.com/doc/current/components/console/helpers/questionhelper.html)。

> [!NOTE]
> 在 Laravel 框架中使用 Laravel Prompts 時，已為您配置好每個 Prompt 的備案方案，並會在不支援的環境中自動啟用。

<a name="fallback-conditions"></a>
#### 備案條件

如果您不是使用 Laravel，或者需要自定義何時使用備案行為，您可以將布林值傳遞給 `Prompt` 類別上的 `fallbackWhen` 靜態方法：

```php
use Laravel\Prompts\Prompt;

Prompt::fallbackWhen(
    ! $input->isInteractive() || windows_os() || app()->runningUnitTests()
);
```

<a name="fallback-behavior"></a>
#### 備案行為

如果您不是使用 Laravel，或者需要自定義備案行為，您可以將閉包傳遞給每個 Prompt 類別上的 `fallbackUsing` 靜態方法：

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

必須為每個 Prompt 類別個別配置備案。閉包將接收該 Prompt 類別的一個實例，並且必須回傳該 Prompt 對應的合適類型。

<a name="testing"></a>
## 測試

Laravel 提供多種方法來測試您的指令是否顯示了預期的 Prompt 訊息：

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
# 提示

- [簡介](#introduction)
- [安裝](#installation)
- [可用的提示](#available-prompts)
    - [文字](#text)
    - [文字區域](#textarea)
    - [密碼](#password)
    - [確認](#confirm)
    - [選擇](#select)
    - [多重選擇](#multiselect)
    - [建議](#suggest)
    - [搜尋](#search)
    - [多重搜尋](#multisearch)
    - [暫停](#pause)
- [驗證前轉換輸入](#transforming-input-before-validation)
- [表單](#forms)
- [資訊訊息](#informational-messages)
- [表格](#tables)
- [旋轉](#spin)
- [進度條](#progress)
- [清除終端機](#clear)
- [終端機考量](#terminal-considerations)
- [不支援的環境與備用方案](#fallbacks)

<a name="introduction"></a>
## 簡介

[Laravel Prompts](https://github.com/laravel/prompts) 是一個 PHP 套件，用於為您的命令列應用程式增加美觀且使用者友善的表單，並具備類似瀏覽器的功能，包括預留文字和驗證。

<img src="https://laravel.com/img/docs/prompts-example.png">

Laravel Prompts 非常適合在您的 [Artisan 命令列指令](/docs/{{version}}/artisan#writing-commands) 中接受使用者輸入，但它也可以用於任何命令列 PHP 專案。

> [!NOTE]  
> Laravel Prompts 支援 macOS、Linux，以及帶有 WSL 的 Windows。如需更多資訊，請參閱我們關於 [不支援環境與備用方案](#fallbacks) 的文件。

<a name="installation"></a>
## 安裝

Laravel Prompts 已包含於最新版本的 Laravel 中。

Laravel Prompts 也可以透過 Composer 套件管理器安裝在您的其他 PHP 專案中：

```shell
composer require laravel/prompts
```

<a name="available-prompts"></a>
## 可用的提示


<a name="text"></a>
### 文字

`text` 函數將會向使用者顯示給定的問題、接受他們的輸入，然後回傳該輸入：

```php
use function Laravel\Prompts\text;

$name = text('What is your name?');
```

你也可以包含預留文字、預設值，以及資訊提示：

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

如果你要求必須輸入一個值，你可以傳遞 `required` 參數：

```php
$name = text(
    label: 'What is your name?',
    required: true
);
```

如果你想自訂驗證訊息，你也可以傳遞一個字串：

```php
$name = text(
    label: 'What is your name?',
    required: 'Your name is required.'
);
```


<a name="text-validation"></a>
#### 額外驗證

最後，如果你想執行額外的驗證邏輯，你可以傳遞一個閉包給 `validate` 參數：

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

該閉包將會接收已輸入的值，並且可能回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，你也可以利用 Laravel 的 [驗證器](/docs/{{version}}/validation) 的強大功能。為此，請提供一個包含屬性名稱和所需驗證規則的陣列給 `validate` 參數：

```php
$name = text(
    label: 'What is your name?',
    validate: ['name' => 'required|max:255|unique:users']
);
```


<a name="textarea"></a>
### 文字區域

`textarea` 函數將會向使用者顯示給定的問題、透過多行文字區域接受他們的輸入，然後回傳該輸入：

```php
use function Laravel\Prompts\textarea;

$story = textarea('Tell me a story.');
```

你也可以包含預留文字、預設值，以及資訊提示：

```php
$story = textarea(
    label: 'Tell me a story.',
    placeholder: 'This is a story about...',
    hint: 'This will be displayed on your profile.'
);
```


<a name="textarea-required"></a>
#### 必填值

如果你要求必須輸入一個值，你可以傳遞 `required` 參數：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: true
);
```

如果你想自訂驗證訊息，你也可以傳遞一個字串：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: 'A story is required.'
);
```


<a name="textarea-validation"></a>
#### 額外驗證

最後，如果你想執行額外的驗證邏輯，你可以傳遞一個閉包給 `validate` 參數：

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

該閉包將會接收已輸入的值，並且可能回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，你也可以利用 Laravel 的 [驗證器](/docs/{{version}}/validation) 的強大功能。為此，請提供一個包含屬性名稱和所需驗證規則的陣列給 `validate` 參數：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: ['story' => 'required|max:10000']
);
```


<a name="password"></a>
### 密碼

`password` 函數與 `text` 函數類似，但當使用者在控制台中輸入時，他們的輸入將會被遮罩。這在詢問敏感資訊（例如密碼）時很有用：

```php
use function Laravel\Prompts\password;

$password = password('What is your password?');
```

你也可以包含預留文字和資訊提示：

```php
$password = password(
    label: 'What is your password?',
    placeholder: 'password',
    hint: 'Minimum 8 characters.'
);
```


<a name="password-required"></a>
#### 必填值

如果你要求必須輸入一個值，你可以傳遞 `required` 參數：

```php
$password = password(
    label: 'What is your password?',
    required: true
);
```

如果你想自訂驗證訊息，你也可以傳遞一個字串：

```php
$password = password(
    label: 'What is your password?',
    required: 'The password is required.'
);
```


<a name="password-validation"></a>
#### 額外驗證

最後，如果你想執行額外的驗證邏輯，你可以傳遞一個閉包給 `validate` 參數：

```php
$password = password(
    label: 'What is your password?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 8 => 'The password must be at least 8 characters.',
        default => null
    }
);
```

該閉包將會接收已輸入的值，並且可能回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，你也可以利用 Laravel 的 [驗證器](/docs/{{version}}/validation) 的強大功能。為此，請提供一個包含屬性名稱和所需驗證規則的陣列給 `validate` 參數：

```php
$password = password(
    label: 'What is your password?',
    validate: ['password' => 'min:8']
);
```


<a name="confirm"></a>
### 確認

如果你需要詢問使用者「是或否」的確認，你可以使用 `confirm` 函數。使用者可以使用方向鍵或按下 `y` 或 `n` 來選擇他們的回應。此函數將會回傳 `true` 或 `false`。

```php
use function Laravel\Prompts\confirm;

$confirmed = confirm('Do you accept the terms?');
```

你也可以包含預設值、自訂「Yes」和「No」標籤的措辭，以及資訊提示：

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
#### 要求「是」

如有必要，你可以透過傳遞 `required` 參數來要求使用者選擇「Yes」：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: true
);
```

如果你想自訂驗證訊息，你也可以傳遞一個字串：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: 'You must accept the terms to continue.'
);
```

<a name="select"></a>
### 選擇

如果您需要使用者從預設的選項集合中進行選擇，您可以使用 `select` 函式：

```php
use function Laravel\Prompts\select;

$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner']
);
```

您也可以指定預設選項和提示資訊：

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    default: 'Owner',
    hint: 'The role may be changed at any time.'
);
```

您也可以傳遞關聯陣列給 `options` 引數，使其回傳所選的鍵而非其值：

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

在清單開始滾動之前，最多會顯示五個選項。您可以透過傳遞 `scroll` 引數來自訂此行為：

```php
$role = select(
    label: 'Which category would you like to assign?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```


<a name="select-validation"></a>
#### 額外驗證

與其他提示函式不同，`select` 函式不接受 `required` 引數，因為不可能不選擇任何東西。但是，如果您需要顯示一個選項但阻止其被選中，您可以傳遞一個閉包給 `validate` 引數：

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

如果 `options` 引數是關聯陣列，那麼閉包將接收所選的鍵，否則它將接收所選的值。如果驗證通過，該閉包可以回傳錯誤訊息，或回傳 `null`。


<a name="multiselect"></a>
### 多重選擇

如果您需要使用者能夠選擇多個選項，您可以使用 `multiselect` 函式：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete']
);
```

您也可以指定預設選項和提示資訊：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete'],
    default: ['Read', 'Create'],
    hint: 'Permissions may be updated at any time.'
);
```

您也可以傳遞關聯陣列給 `options` 引數，使其回傳所選選項的鍵而非其值：

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

在清單開始滾動之前，最多會顯示五個選項。您可以透過傳遞 `scroll` 引數來自訂此行為：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```


<a name="multiselect-required"></a>
#### 要求輸入值

預設情況下，使用者可以選擇零個或多個選項。您可以傳遞 `required` 引數來強制選擇一個或多個選項：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: true
);
```

如果您想自訂驗證訊息，您可以向 `required` 引數提供一個字串：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: 'You must select at least one category'
);
```


<a name="multiselect-validation"></a>
#### 額外驗證

如果您需要顯示一個選項但阻止其被選中，您可以傳遞一個閉包給 `validate` 引數：

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

如果 `options` 引數是關聯陣列，那麼閉包將接收所選的鍵，否則它將接收所選的值。如果驗證通過，該閉包可以回傳錯誤訊息，或回傳 `null`。


<a name="suggest"></a>
### 建議

`suggest` 函式可用於為可能的選項提供自動完成。無論自動完成提示如何，使用者仍然可以提供任何答案：

```php
use function Laravel\Prompts\suggest;

$name = suggest('What is your name?', ['Taylor', 'Dayle']);
```

或者，您可以將閉包作為第二個引數傳遞給 `suggest` 函式。每次使用者鍵入輸入字元時，都會呼叫該閉包。該閉包應接受一個字串參數，其中包含使用者目前的輸入，並回傳一個用於自動完成的選項陣列：

```php
$name = suggest(
    label: 'What is your name?',
    options: fn ($value) => collect(['Taylor', 'Dayle'])
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
)
```

您也可以包含佔位符文字、預設值和提示資訊：

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

如果您要求輸入值，您可以傳遞 `required` 引數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: true
);
```

如果您想自訂驗證訊息，您也可以傳遞一個字串：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: 'Your name is required.'
);
```


<a name="suggest-validation"></a>
#### 額外驗證

最後，如果您想執行額外的驗證邏輯，您可以傳遞一個閉包給 `validate` 引數：

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

閉包將接收已輸入的值，如果驗證通過，可以回傳錯誤訊息，或回傳 `null`。

或者，您可以利用 Laravel 的 [驗證器](/docs/{{version}}/validation) 的功能。為此，請提供一個陣列，其中包含屬性名稱和所需的驗證規則給 `validate` 引數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: ['name' => 'required|min:3|max:255']
);
```

<a name="search"></a>
### 搜尋

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

閉包會接收使用者目前輸入的文字，並且必須回傳一個選項陣列。如果您回傳一個關聯陣列，那麼將回傳所選選項的鍵，否則將回傳其值。

當您篩選一個打算回傳值的陣列時，您應該使用 `array_values` 函式或 `values` Collection 方法來確保陣列不會變成關聯陣列：

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

您還可以包含預留位置文字和資訊提示：

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

在清單開始滾動之前，最多會顯示五個選項。您可以透過傳遞 `scroll` 引數來自訂此行為：

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

如果您想執行額外的驗證邏輯，您可以將一個閉包傳遞給 `validate` 引數：

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

如果 `options` 閉包回傳關聯陣列，那麼該閉包將接收所選的鍵；否則，它將接收所選的值。該閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。


<a name="multisearch"></a>
### 多重搜尋

如果您有許多可搜尋的選項，並且需要使用者能夠選擇多個項目，`multisearch` 函式允許使用者輸入搜尋查詢來篩選結果，然後再使用方向鍵和空白鍵選擇選項：

```php
use function Laravel\Prompts\multisearch;

$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

閉包會接收使用者目前輸入的文字，並且必須回傳一個選項陣列。如果您回傳一個關聯陣列，那麼將回傳所選選項的鍵；否則，將回傳其值。

當您篩選一個打算回傳值的陣列時，您應該使用 `array_values` 函式或 `values` Collection 方法來確保陣列不會變成關聯陣列：

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

您還可以包含預留位置文字和資訊提示：

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

在清單開始滾動之前，最多會顯示五個選項。您可以透過提供 `scroll` 引數來自訂此行為：

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
#### 必填值

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

如果您想自訂驗證訊息，您也可以向 `required` 引數提供一個字串：

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

如果您想執行額外的驗證邏輯，您可以將一個閉包傳遞給 `validate` 引數：

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

如果 `options` 閉包回傳關聯陣列，那麼該閉包將接收所選的鍵；否則，它將接收所選的值。該閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。


<a name="pause"></a>
### 暫停

`pause` 函式可用於向使用者顯示資訊文字，並等待他們按下 Enter / Return 鍵來確認繼續：

```php
use function Laravel\Prompts\pause;

pause('Press ENTER to continue.');
```

<a name="transforming-input-before-validation"></a>
## 驗證前轉換輸入

有時您可能希望在驗證發生之前轉換提示輸入。例如，您可能希望從任何提供的字串中移除空白字元。為了實現這一點，許多提示函數提供了一個 `transform` 引數，它接受一個閉包：

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

通常，您會有多個提示按順序顯示，以在執行其他操作之前收集資訊。您可以使用 `form` 函數來建立一組供使用者完成的提示：

```php
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true)
    ->password('What is your password?', validate: ['password' => 'min:8'])
    ->confirm('Do you accept the terms?')
    ->submit();
```

`submit` 方法將返回一個包含表單所有提示回應的數字索引陣列。然而，您可以透過 `name` 引數為每個提示提供一個名稱。當提供了名稱時，可以透過該名稱存取具名提示的回應：

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

使用 `form` 函數的主要好處是使用者可以使用 `CTRL + U` 返回表單中先前的提示。這讓使用者可以修正錯誤或更改選項，而無需取消並重新啟動整個表單。

如果您需要對表單中的提示進行更精細的控制，您可以呼叫 `add` 方法，而不是直接呼叫其中一個提示函數。`add` 方法會傳遞使用者提供的所有先前回應：

```php
use function Laravel\Prompts\form;
use function Laravel\Prompts\outro;

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

`table` 函數讓您輕鬆顯示多行多列的資料。您只需提供欄位名稱和表格的資料：

```php
use function Laravel\Prompts\table;

table(
    headers: ['Name', 'Email'],
    rows: User::all(['name', 'email'])->toArray()
);
```


<a name="spin"></a>
## 旋轉

`spin` 函數在執行指定的 callback 時，會顯示一個 spinner 和一個可選的訊息。它用於指示正在進行的過程，並在完成後返回 callback 的結果：

```php
use function Laravel\Prompts\spin;

$response = spin(
    message: 'Fetching response...',
    callback: fn () => Http::get('http://example.com')
);
```

> [!WARNING]  
> `spin` 函數需要 `pcntl` PHP 擴充功能來動畫化 spinner。當此擴充功能不可用時，將顯示靜態版本的 spinner。


<a name="progress"></a>
## 進度條

對於長時間運行的任務，顯示進度條以告知使用者任務完成度會很有幫助。使用 `progress` 函數，Laravel 會顯示一個進度條，並在給定的可迭代值上的每次迭代中推進其進度：

```php
use function Laravel\Prompts\progress;

$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: fn ($user) => $this->performTask($user)
);
```

`progress` 函數的作用類似於 map 函數，它將返回一個包含您的 callback 每次迭代回傳值的陣列。

callback 也可以接受 `Laravel\Prompts\Progress` 實例，允許您在每次迭代中修改標籤和提示：

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

有時，您可能需要更手動地控制進度條的推進方式。首先，定義過程將迭代的總步數。然後，在處理每個項目後，透過 `advance` 方法推進進度條：

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

```
use function Laravel\Prompts\clear;

clear();
```


<a name="terminal-considerations"></a>
## 終端機考量


<a name="terminal-width"></a>
#### 終端機寬度

如果任何標籤、選項或驗證訊息的長度超過使用者終端機中的「欄數」，它將會被自動截斷以適應。如果您的使用者可能使用較窄的終端機，請考慮縮短這些字串的長度。通常安全的建議最大長度是 74 個字元，以支援 80 字元的終端機。


<a name="terminal-height"></a>
#### 終端機高度

對於任何接受 `scroll` 引數的提示，其配置值將會自動縮減以適應使用者終端機的高度，包括驗證訊息的空間。

<a name="fallbacks"></a>
## 不支援的環境與備用方案

Laravel Prompts 支援 macOS、Linux 以及搭載 WSL 的 Windows。由於 Windows 版 PHP 的限制，目前無法在 WSL 之外的 Windows 上使用 Laravel Prompts。

因此，Laravel Prompts 支援回退到替代實作，例如 [Symfony Console Question Helper](https://symfony.com/doc/7.0/components/console/helpers/questionhelper.html)。

> [!NOTE]  
> 當您將 Laravel Prompts 與 Laravel 框架搭配使用時，每個提示的備用方案都已為您配置完成，並將在不支援的環境中自動啟用。

<a name="fallback-conditions"></a>
#### 備用方案條件

如果您未使用 Laravel 或需要自訂何時使用備用方案行為，您可以將布林值傳遞給 `Prompt` 類別上的 `fallbackWhen` 靜態方法：

```php
use Laravel\Prompts\Prompt;

Prompt::fallbackWhen(
    ! $input->isInteractive() || windows_os() || app()->runningUnitTests()
);
```

<a name="fallback-behavior"></a>
#### 備用方案行為

如果您未使用 Laravel 或需要自訂備用方案行為，您可以將閉包傳遞給每個提示類別上的 `fallbackUsing` 靜態方法：

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

備用方案必須為每個提示類別單獨配置。閉包將接收提示類別的實例，並且必須返回該提示的適當類型。
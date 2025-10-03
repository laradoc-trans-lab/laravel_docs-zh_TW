# Eloquent：序列化

- [簡介](#introduction)
- [模型與集合的序列化](#serializing-models-and-collections)
    - [序列化為陣列](#serializing-to-arrays)
    - [序列化為 JSON](#serializing-to-json)
- [在 JSON 中隱藏屬性](#hiding-attributes-from-json)
- [向 JSON 追加值](#appending-values-to-json)
- [日期序列化](#date-serialization)

<a name="introduction"></a>
## 簡介

在使用 Laravel 建立 API 時，您經常需要將您的模型和關聯轉換為陣列或 JSON。Eloquent 包含了方便的方法來進行這些轉換，以及控制哪些屬性被包含在您模型的序列化中。

> [!NOTE]
> 若要以更強健的方式處理 Eloquent 模型和集合的 JSON 序列化，請查閱 [Eloquent API 資源](/docs/{{version}}/eloquent-resources) 的文件。


<a name="serializing-models-and-collections"></a>
## 模型與集合的序列化


<a name="serializing-to-arrays"></a>
### 序列化為陣列

要將模型及其已載入的 [關聯](/docs/{{version}}/eloquent-relationships) 轉換為陣列，您應該使用 `toArray` 方法。此方法是遞迴的，因此所有屬性以及所有關聯 (包括關聯的關聯) 都將被轉換為陣列：

```php
use App\Models\User;

$user = User::with('roles')->first();

return $user->toArray();
```

`attributesToArray` 方法可用於將模型的屬性轉換為陣列，但不會轉換其關聯：

```php
$user = User::first();

return $user->attributesToArray();
```

您也可以透過在集合實例上呼叫 `toArray` 方法，將整個 [集合](/docs/{{version}}/eloquent-collections) 的模型轉換為陣列：

```php
$users = User::all();

return $users->toArray();
```


<a name="serializing-to-json"></a>
### 序列化為 JSON

要將模型轉換為 JSON，您應該使用 `toJson` 方法。與 `toArray` 類似，`toJson` 方法是遞迴的，因此所有屬性以及關聯都將被轉換為 JSON。您還可以指定任何 [PHP 支援的](https://secure.php.net/manual/en/function.json-encode.php) JSON 編碼選項：

```php
use App\Models\User;

$user = User::find(1);

return $user->toJson();

return $user->toJson(JSON_PRETTY_PRINT);
```

或者，您可以將模型或集合轉換為字串，這將自動在模型或集合上呼叫 `toJson` 方法：

```php
return (string) User::find(1);
```

由於模型和集合在轉換為字串時會被轉換為 JSON，您可以直接從應用程式的路由或控制器中回傳 Eloquent 物件。當您的 Eloquent 模型和集合從路由或控制器中回傳時，Laravel 會自動將其序列化為 JSON：

```php
Route::get('/users', function () {
    return User::all();
});
```


<a name="relationships"></a>
#### 關聯

當 Eloquent 模型轉換為 JSON 時，其已載入的關聯將自動作為屬性包含在 JSON 物件中。此外，儘管 Eloquent 關聯方法是使用「駝峰式大小寫 (camel case)」方法名稱定義的，但關聯的 JSON 屬性將會是「蛇形命名法 (snake case)」。


<a name="hiding-attributes-from-json"></a>
## 在 JSON 中隱藏屬性

有時您可能希望限制模型陣列或 JSON 表示中包含的屬性，例如密碼。為此，請在您的模型中新增一個 `$hidden` 屬性。列在 `$hidden` 屬性陣列中的屬性將不會包含在您模型的序列化中：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The attributes that should be hidden for serialization.
     *
     * @var array<string>
     */
    protected $hidden = ['password'];
}
```

> [!NOTE]
> 要隱藏關聯，請將關聯的方法名稱新增到您的 Eloquent 模型的 `$hidden` 屬性中。

或者，您可以使用 `visible` 屬性來定義一個應包含在模型陣列和 JSON 表示中的屬性「白名單」。所有不在 `$visible` 陣列中的屬性在模型轉換為陣列或 JSON 時都將被隱藏：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The attributes that should be visible in arrays.
     *
     * @var array
     */
    protected $visible = ['first_name', 'last_name'];
}
```


<a name="temporarily-modifying-attribute-visibility"></a>
#### 暫時修改屬性可見性

如果您想讓某個給定模型實例上通常隱藏的屬性變得可見，您可以使用 `makeVisible` 或 `mergeVisible` 方法。`makeVisible` 方法會回傳模型實例：

```php
return $user->makeVisible('attribute')->toArray();

return $user->mergeVisible(['name', 'email'])->toArray();
```

同樣地，如果您想隱藏一些通常可見的屬性，您可以使用 `makeHidden` 或 `mergeHidden` 方法：

```php
return $user->makeHidden('attribute')->toArray();

return $user->mergeHidden(['name', 'email'])->toArray();
```

如果您希望暫時覆寫所有可見或隱藏的屬性，您可以分別使用 `setVisible` 和 `setHidden` 方法：

```php
return $user->setVisible(['id', 'name'])->toArray();

return $user->setHidden(['email', 'password', 'remember_token'])->toArray();
```


<a name="appending-values-to-json"></a>
## 向 JSON 追加值

有時，當將模型轉換為陣列或 JSON 時，您可能希望新增一些在資料庫中沒有對應欄位的屬性。為此，請先為該值定義一個 [存取器](/docs/{{version}}/eloquent-mutators)：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Determine if the user is an administrator.
     */
    protected function isAdmin(): Attribute
    {
        return new Attribute(
            get: fn () => 'yes',
        );
    }
}
```

如果您希望存取器總是追加到模型的陣列和 JSON 表示中，您可以將屬性名稱新增到模型的 `appends` 屬性。請注意，屬性名稱通常使用其「蛇形命名法 (snake case)」序列化表示來參考，即使存取器的 PHP 方法是使用「駝峰式大小寫 (camel case)」定義的：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The accessors to append to the model's array form.
     *
     * @var array
     */
    protected $appends = ['is_admin'];
}
```

一旦屬性被新增到 `appends` 列表中，它將被包含在模型的陣列和 JSON 表示中。`appends` 陣列中的屬性也將遵守模型上配置的 `visible` 和 `hidden` 設定。


<a name="appending-at-run-time"></a>
#### 執行時追加

在執行時，您可以指示模型實例使用 `append` 或 `mergeAppends` 方法來追加額外的屬性。或者，您可以使用 `setAppends` 方法來覆寫給定模型實例的整個追加屬性陣列：

```php
return $user->append('is_admin')->toArray();

return $user->mergeAppends(['is_admin', 'status'])->toArray();

return $user->setAppends(['is_admin'])->toArray();
```


<a name="date-serialization"></a>
## 日期序列化


<a name="customizing-the-default-date-format"></a>
#### 自訂預設日期格式

您可以透過覆寫 `serializeDate` 方法來自訂預設的序列化格式。此方法不會影響您日期在資料庫中的儲存格式：

```php
/**
 * Prepare a date for array / JSON serialization.
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```


<a name="customizing-the-date-format-per-attribute"></a>
#### 每個屬性的日期格式自訂

您可以透過在模型的 [型別轉換宣告](/docs/{{version}}/eloquent-mutators#attribute-casting) 中指定日期格式，來自訂各個 Eloquent 日期屬性的序列化格式：

```php
protected function casts(): array
{
    return [
        'birthday' => 'date:Y-m-d',
        'joined_at' => 'datetime:Y-m-d H:00',
    ];
}
```
# Eloquent: 序列化

- [簡介](#introduction)
- [模型與集合的序列化](#serializing-models-and-collections)
    - [序列化為陣列](#serializing-to-arrays)
    - [序列化為 JSON](#serializing-to-json)
- [從 JSON 中隱藏屬性](#hiding-attributes-from-json)
- [將值附加至 JSON](#appending-values-to-json)
- [日期序列化](#date-serialization)

<a name="introduction"></a>
## 簡介

使用 Laravel 建立 API 時，您經常需要將模型與關聯轉換為陣列或 JSON。Eloquent 提供了便捷的方法來執行這些轉換，並能控制哪些屬性應包含在模型的序列化表示中。

> [!NOTE]
> 若要以更強健的方式處理 Eloquent 模型與集合的 JSON 序列化，請參閱 [Eloquent API resources](/docs/{{version}}/eloquent-resources) 的文件。


<a name="serializing-models-and-collections"></a>
## 模型與集合的序列化


<a name="serializing-to-arrays"></a>
### 序列化為陣列

若要將模型及其已載入的 [relationships](/docs/{{version}}/eloquent-relationships) 轉換為陣列，您應該使用 `toArray` 方法。此方法是遞迴的，因此所有屬性和所有關聯（包括關聯的關聯）都將被轉換為陣列：

```php
use App\Models\User;

$user = User::with('roles')->first();

return $user->toArray();
```

`attributesToArray` 方法可用於將模型的屬性轉換為陣列，但不包含其關聯：

```php
$user = User::first();

return $user->attributesToArray();
```

您也可以透過在集合實例上呼叫 `toArray` 方法，將整個模型的 [collections](/docs/{{version}}/eloquent-collections) 轉換為陣列：

```php
$users = User::all();

return $users->toArray();
```


<a name="serializing-to-json"></a>
### 序列化為 JSON

若要將模型轉換為 JSON，您應該使用 `toJson` 方法。與 `toArray` 類似，`toJson` 方法是遞迴的，因此所有屬性和關聯都將被轉換為 JSON。您也可以指定任何 [PHP 支援](https://secure.php.net/manual/en/function.json-encode.php) 的 JSON 編碼選項：

```php
use App\Models\User;

$user = User::find(1);

return $user->toJson();

return $user->toJson(JSON_PRETTY_PRINT);
```

或者，您可以將模型或集合強制轉換為字串，這將自動在該模型或集合上呼叫 `toJson` 方法：

```php
return (string) User::find(1);
```

由於模型與集合在強制轉換為字串時會被轉換為 JSON，因此您可以直接從應用程式的路由或控制器回傳 Eloquent 物件。當 Eloquent 模型與集合從路由或控制器回傳時，Laravel 會自動將其序列化為 JSON：

```php
Route::get('/users', function () {
    return User::all();
});
```


<a name="relationships"></a>
#### 關聯

當 Eloquent 模型被轉換為 JSON 時，其已載入的關聯將自動作為 JSON 物件的屬性包含在內。此外，儘管 Eloquent 的關聯方法是使用「駝峰式命名 (camel case)」定義方法名稱，但關聯的 JSON 屬性將會是「蛇形命名 (snake case)」。


<a name="hiding-attributes-from-json"></a>
## 從 JSON 中隱藏屬性

有時您可能希望限制包含在模型陣列或 JSON 表示中的屬性（例如密碼）。若要實現此目的，您可以在模型上使用 `Hidden` 屬性。列在 `Hidden` 屬性中的屬性將不會包含在模型的序列化表示中：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Hidden;
use Illuminate\Database\Eloquent\Model;

#[Hidden(['password'])]
class User extends Model
{
    // ...
}
```


> [!NOTE]
> 若要隱藏關聯，請將關聯的方法名稱添加到 Eloquent 模型的 `Hidden` 屬性中。

或者，您可以使用 `Visible` 屬性來定義一個屬性的「允許清單」，這些屬性將被包含在模型的陣列與 JSON 表示中。所有未出現在 `Visible` 屬性中的屬性，在模型轉換為陣列或 JSON 時都會被隱藏：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Visible;
use Illuminate\Database\Eloquent\Model;

#[Visible(['first_name', 'last_name'])]
class User extends Model
{
    // ...
}
```


<a name="temporarily-modifying-attribute-visibility"></a>
#### 暫時修改屬性的可見性

如果您希望在特定的模型實例上使某些通常被隱藏的屬性可見，可以使用 `makeVisible` 或 `mergeVisible` 方法。`makeVisible` 方法會回傳該模型實例：

```php
return $user->makeVisible('attribute')->toArray();

return $user->mergeVisible(['name', 'email'])->toArray();
```

同樣地，如果您希望隱藏某些通常可見的屬性，可以使用 `makeHidden` 或 `mergeHidden` 方法：

```php
return $user->makeHidden('attribute')->toArray();

return $user->mergeHidden(['name', 'email'])->toArray();
```

如果您希望暫時覆蓋所有可見或隱藏的屬性，可以分別使用 `setVisible` 和 `setHidden` 方法：

```php
return $user->setVisible(['id', 'name'])->toArray();

return $user->setHidden(['email', 'password', 'remember_token'])->toArray();
```


<a name="appending-values-to-json"></a>
## 將值附加至 JSON

偶爾在將模型轉換為陣列或 JSON 時，您可能希望添加一些在資料庫中沒有對應欄位的屬性。若要實現此目的，請先為該值定義一個 [accessor](/docs/{{version}}/eloquent-mutators)：

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

如果您希望存取器始終附加到模型的陣列與 JSON 表示中，可以在模型上使用 `Appends` 屬性。請注意，屬性名稱通常使用其「蛇形命名 (snake case)」的序列化表示，儘管存取器的 PHP 方法是使用「駝峰式命名 (camel case)」定義的：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Appends;
use Illuminate\Database\Eloquent\Model;

#[Appends(['is_admin'])]
class User extends Model
{
    // ...
}
```

一旦屬性被添加到 `appends` 清單中，它將包含在模型的陣列與 JSON 表示中。`appends` 陣列中的屬性同樣會遵循模型上配置的 `visible` 與 `hidden` 設定。


<a name="appending-at-run-time"></a>
#### 在執行時附加

在執行時，您可以使用 `append` 或 `mergeAppends` 方法指示模型實例附加額外的屬性。或者，您可以使用 `setAppends` 方法來覆蓋特定模型實例的所有附加屬性陣列：

```php
return $user->append('is_admin')->toArray();

return $user->mergeAppends(['is_admin', 'status'])->toArray();

return $user->setAppends(['is_admin'])->toArray();
```

同樣地，如果您想從模型中移除所有附加屬性，可以使用 `withoutAppends` 方法：

```php
return $user->withoutAppends()->toArray();
```


<a name="date-serialization"></a>
## 日期序列化


<a name="customizing-the-default-date-format"></a>
#### 自定義預設日期格式

您可以透過覆寫 `serializeDate` 方法來自定義預設的序列化格式。此方法不會影響日期儲存在資料庫中的格式：

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
#### 為每個屬性自定義日期格式

您可以透過在模型的 [型別轉換宣告](/docs/{{version}}/eloquent-mutators#attribute-casting) 中指定日期格式，來自定義個別 Eloquent 日期屬性的序列化格式：

```php
protected function casts(): array
{
    return [
        'birthday' => 'date:Y-m-d',
        'joined_at' => 'datetime:Y-m-d H:00',
    ];
}
```
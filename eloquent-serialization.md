# Eloquent：序列化

- [簡介](#introduction)
- [序列化 Model 與 Collection](#serializing-models-and-collections)
    - [序列化為陣列](#serializing-to-arrays)
    - [序列化為 JSON](#serializing-to-json)
- [從 JSON 中隱藏屬性](#hiding-attributes-from-json)
- [將值附加到 JSON](#appending-values-to-json)
- [日期序列化](#date-serialization)

<a name="introduction"></a>
## 簡介

當使用 Laravel 建立 API 時，您經常需要將 Model 與其關聯轉換為陣列或 JSON。Eloquent 提供了方便的方法來進行這些轉換，並控制哪些屬性應包含在 Model 的序列化表示中。

> [!NOTE]
> 若要以更強大的方式處理 Eloquent Model 與 Collection 的 JSON 序列化，請參閱 [Eloquent API resources](/docs/{{version}}/eloquent-resources) 的說明文件。

<a name="serializing-models-and-collections"></a>
## 序列化 Model 與 Collection

<a name="serializing-to-arrays"></a>
### 序列化為陣列

若要將 Model 及其已載入的[關聯](/docs/{{version}}/eloquent-relationships)轉換為陣列，您應該使用 `toArray` 方法。此方法是遞迴的，因此所有屬性與所有關聯 (包括關聯的關聯) 都將轉換為陣列：

    use App\Models\User;

    $user = User::with('roles')->first();

    return $user->toArray();

`attributesToArray` 方法可用於將 Model 的屬性轉換為陣列，但不包含其關聯：

    $user = User::first();

    return $user->attributesToArray();

您也可以透過在 Collection 實例上呼叫 `toArray` 方法，將整個 [Collection](/docs/{{version}}/eloquent-collections) 轉換為陣列：

    $users = User::all();

    return $users->toArray();

<a name="serializing-to-json"></a>
### 序列化為 JSON

若要將 Model 轉換為 JSON，您應該使用 `toJson` 方法。與 `toArray` 類似，`toJson` 方法是遞迴的，因此所有屬性與關聯都將轉換為 JSON。您也可以指定 [PHP 支援](https://secure.php.net/manual/en/function.json-encode.php)的任何 JSON 編碼選項：

    use App\Models\User;

    $user = User::find(1);

    return $user->toJson();

    return $user->toJson(JSON_PRETTY_PRINT);

或者，您可以將 Model 或 Collection 轉換為字串，這將自動在 Model 或 Collection 上呼叫 `toJson` 方法：

    return (string) User::find(1);

由於 Model 與 Collection 在轉換為字串時會轉換為 JSON，您可以直接從應用程式的路由或 Controller 返回 Eloquent 物件。當 Eloquent Model 與 Collection 從路由或 Controller 返回時，Laravel 將自動將其序列化為 JSON：

    Route::get('/users', function () {
        return User::all();
    });

<a name="relationships"></a>
#### 關聯

當 Eloquent Model 轉換為 JSON 時，其已載入的關聯將自動作為屬性包含在 JSON 物件中。此外，儘管 Eloquent 關聯方法是使用「駝峰式命名」方法名稱定義的，但關聯的 JSON 屬性將是「蛇式命名」。

<a name="hiding-attributes-from-json"></a>
## 從 JSON 中隱藏屬性

有時您可能希望限制 Model 的陣列或 JSON 表示中包含的屬性，例如密碼。為此，請在 Model 中新增 `$hidden` 屬性。列在 `$hidden` 屬性陣列中的屬性將不會包含在 Model 的序列化表示中：

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

> [!NOTE]
> 若要隱藏關聯，請將關聯的方法名稱新增到 Eloquent Model 的 `$hidden` 屬性中。

或者，您可以使用 `visible` 屬性來定義應包含在 Model 的陣列與 JSON 表示中的屬性「允許清單」。當 Model 轉換為陣列或 JSON 時，所有未出現在 `$visible` 陣列中的屬性都將被隱藏：

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

<a name="temporarily-modifying-attribute-visibility"></a>
#### 暫時修改屬性可見性

如果您希望在給定的 Model 實例上使某些通常隱藏的屬性可見，您可以使用 `makeVisible` 方法。`makeVisible` 方法會返回 Model 實例：

    return $user->makeVisible('attribute')->toArray();

同樣地，如果您希望隱藏某些通常可見的屬性，您可以使用 `makeHidden` 方法。

    return $user->makeHidden('attribute')->toArray();

如果您希望暫時覆寫所有可見或隱藏的屬性，您可以分別使用 `setVisible` 和 `setHidden` 方法：

    return $user->setVisible(['id', 'name'])->toArray();

    return $user->setHidden(['email', 'password', 'remember_token'])->toArray();

<a name="appending-values-to-json"></a>
## 將值附加到 JSON

有時，當將 Model 轉換為陣列或 JSON 時，您可能希望新增資料庫中沒有對應欄位的屬性。為此，請先為該值定義一個 [Accessor](/docs/{{version}}/eloquent-mutators)：

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

如果您希望 Accessor 始終附加到 Model 的陣列與 JSON 表示中，您可以將屬性名稱新增到 Model 的 `appends` 屬性中。請注意，屬性名稱通常使用其「蛇式命名」的序列化表示來引用，即使 Accessor 的 PHP 方法是使用「駝峰式命名」定義的：

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

一旦屬性已新增到 `appends` 清單中，它將包含在 Model 的陣列與 JSON 表示中。`appends` 陣列中的屬性也將遵循 Model 上配置的 `visible` 和 `hidden` 設定。

<a name="appending-at-run-time"></a>
#### 執行時附加

在執行時，您可以使用 `append` 方法指示 Model 實例附加額外的屬性。或者，您可以使用 `setAppends` 方法來覆寫給定 Model 實例的整個附加屬性陣列：

    return $user->append('is_admin')->toArray();

    return $user->setAppends(['is_admin'])->toArray();

<a name="date-serialization"></a>
## 日期序列化

<a name="customizing-the-default-date-format"></a>
#### 自訂預設日期格式

您可以透過覆寫 `serializeDate` 方法來自訂預設的序列化格式。此方法不會影響您的日期在資料庫中的儲存格式：

    /**
     * Prepare a date for array / JSON serialization.
     */
    protected function serializeDate(DateTimeInterface $date): string
    {
        return $date->format('Y-m-d');
    }

<a name="customizing-the-date-format-per-attribute"></a>
#### 依屬性自訂日期格式

您可以透過在 Model 的 [型別轉換宣告](/docs/{{version}}/eloquent-mutators#attribute-casting) 中指定日期格式，來自訂個別 Eloquent 日期屬性的序列化格式：

    protected function casts(): array
    {
        return [
            'birthday' => 'date:Y-m-d',
            'joined_at' => 'datetime:Y-m-d H:00',
        ];
    }

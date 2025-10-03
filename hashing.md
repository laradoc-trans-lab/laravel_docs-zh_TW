# 雜湊

- [介紹](#introduction)
- [設定](#configuration)
- [基本使用](#basic-usage)
    - [雜湊密碼](#hashing-passwords)
    - [驗證密碼是否符合雜湊值](#verifying-that-a-password-matches-a-hash)
    - [判斷密碼是否需要重新雜湊](#determining-if-a-password-needs-to-be-rehashed)
- [雜湊演算法驗證](#hash-algorithm-verification)

<a name="introduction"></a>
## 介紹

Laravel 的 `Hash` [facade](/docs/{{version}}/facades) 提供了安全的 Bcrypt 和 Argon2 雜湊來儲存使用者密碼。如果你正在使用 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)之一，Bcrypt 將會預設用於註冊與驗證。

Bcrypt 是雜湊密碼的絕佳選擇，因為它的「工作因子 (work factor)」是可調整的，這表示產生雜湊所需的時間可以隨著硬體能力的提升而增加。雜湊密碼時，慢即是好。演算法雜湊密碼所需的時間越長，惡意使用者產生所有可能的字串雜湊值的「彩虹表 (rainbow tables)」，並對應用程式進行暴力破解攻擊所需的時間就越長。

<a name="configuration"></a>
## 設定

預設情況下，Laravel 在雜湊資料時會使用 `bcrypt` 雜湊驅動器。然而，也支援其他幾個雜湊驅動器，包括 [argon](https://en.wikipedia.org/wiki/Argon2) 和 [argon2id](https://en.wikipedia.org/wiki/Argon2)。

你可以使用 `HASH_DRIVER` 環境變數來指定應用程式的雜湊驅動器。但是，如果你想客製化所有 Laravel 的雜湊驅動器選項，你應該使用 `config:publish` Artisan 指令來發佈完整的 `hashing` 設定檔：

```shell
php artisan config:publish hashing
```

<a name="basic-usage"></a>
## 基本使用

<a name="hashing-passwords"></a>
### 雜湊密碼

你可以透過在 `Hash` facade 上呼叫 `make` 方法來雜湊密碼：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class PasswordController extends Controller
{
    /**
     * Update the password for the user.
     */
    public function update(Request $request): RedirectResponse
    {
        // Validate the new password length...

        $request->user()->fill([
            'password' => Hash::make($request->newPassword)
        ])->save();

        return redirect('/profile');
    }
}
```

<a name="adjusting-the-bcrypt-work-factor"></a>
#### 調整 Bcrypt 工作因子

如果你正在使用 Bcrypt 演算法，`make` 方法允許你使用 `rounds` 選項來管理演算法的工作因子；然而，由 Laravel 管理的預設工作因子對於大多數應用程式來說是可接受的：

```php
$hashed = Hash::make('password', [
    'rounds' => 12,
]);
```

<a name="adjusting-the-argon2-work-factor"></a>
#### 調整 Argon2 工作因子

如果你正在使用 Argon2 演算法，`make` 方法允許你使用 `memory`、`time` 和 `threads` 選項來管理演算法的工作因子；然而，由 Laravel 管理的預設值對於大多數應用程式來說是可接受的：

```php
$hashed = Hash::make('password', [
    'memory' => 1024,
    'time' => 2,
    'threads' => 2,
]);
```

> [!NOTE]
> 有關這些選項的更多資訊，請參閱 [PHP 官方關於 Argon 雜湊的說明文件](https://secure.php.net/manual/en/function.password-hash.php)。

<a name="verifying-that-a-password-matches-a-hash"></a>
### 驗證密碼是否符合雜湊值

`Hash` facade 提供的 `check` 方法允許你驗證給定的明文字串是否與給定的雜湊值相符：

```php
if (Hash::check('plain-text', $hashedPassword)) {
    // The passwords match...
}
```

<a name="determining-if-a-password-needs-to-be-rehashed"></a>
### 判斷密碼是否需要重新雜湊

`Hash` facade 提供的 `needsRehash` 方法允許你判斷自密碼雜湊以來，雜湊器所使用的工作因子是否已更改。有些應用程式選擇在應用程式的驗證過程中執行此檢查：

```php
if (Hash::needsRehash($hashed)) {
    $hashed = Hash::make('plain-text');
}
```

<a name="hash-algorithm-verification"></a>
## 雜湊演算法驗證

為了防止雜湊演算法被竄改，Laravel 的 `Hash::check` 方法會首先驗證給定的雜湊值是否是使用應用程式選擇的雜湊演算法產生。如果演算法不同，則會拋出 `RuntimeException` 異常。

這對於大多數應用程式來說是預期行為，因為雜湊演算法預期不會改變，而不同的演算法可能暗示著惡意攻擊。然而，如果你需要在應用程式中支援多個雜湊演算法，例如從一個演算法遷移到另一個演算法時，你可以透過將 `HASH_VERIFY` 環境變數設定為 `false` 來停用雜湊演算法驗證：

```ini
HASH_VERIFY=false
```
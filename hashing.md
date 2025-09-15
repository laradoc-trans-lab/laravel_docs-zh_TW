# 雜湊 (Hashing)

- [簡介](#introduction)
- [設定](#configuration)
- [基本用法](#basic-usage)
    - [雜湊密碼](#hashing-passwords)
    - [驗證密碼是否符合雜湊值](#verifying-that-a-password-matches-a-hash)
    - [判斷密碼是否需要重新雜湊](#determining-if-a-password-needs-to-be-rehashed)
- [雜湊演算法驗證](#hash-algorithm-verification)

<a name="introduction"></a>
## 簡介

Laravel 的 `Hash` [Facade](/docs/{{version}}/facades) 提供了安全的 Bcrypt 和 Argon2 雜湊功能，用於儲存使用者密碼。如果您正在使用 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)之一，Bcrypt 將預設用於註冊和身份驗證。

Bcrypt 是雜湊密碼的絕佳選擇，因為其「工作因子 (work factor)」可調整，這表示隨著硬體效能的提升，產生雜湊所需的時間也可以增加。在雜湊密碼時，慢是好事。演算法雜湊密碼所需的時間越長，惡意使用者產生所有可能字串雜湊值的「彩虹表 (rainbow tables)」所需的時間就越長，這些彩虹表可能用於對應用程式進行暴力破解攻擊。

<a name="configuration"></a>
## 設定

預設情況下，Laravel 在雜湊資料時使用 `bcrypt` 雜湊驅動程式。然而，也支援其他幾種雜湊驅動程式，包括 [`argon`](https://en.wikipedia.org/wiki/Argon2) 和 [`argon2id`](https://en.wikipedia.org/wiki/Argon2)。

您可以使用 `HASH_DRIVER` 環境變數來指定應用程式的雜湊驅動程式。但是，如果您想自訂所有 Laravel 的雜湊驅動程式選項，您應該使用 `config:publish` Artisan 命令發佈完整的 `hashing` 設定檔：

```bash
php artisan config:publish hashing
```

<a name="basic-usage"></a>
## 基本用法

<a name="hashing-passwords"></a>
### 雜湊密碼

您可以透過呼叫 `Hash` Facade 上的 `make` 方法來雜湊密碼：

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

<a name="adjusting-the-bcrypt-work-factor"></a>
#### 調整 Bcrypt 工作因子

如果您正在使用 Bcrypt 演算法，`make` 方法允許您使用 `rounds` 選項來管理演算法的工作因子；然而，Laravel 管理的預設工作因子對於大多數應用程式來說是可接受的：

    $hashed = Hash::make('password', [
        'rounds' => 12,
    ]);

<a name="adjusting-the-argon2-work-factor"></a>
#### 調整 Argon2 工作因子

如果您正在使用 Argon2 演算法，`make` 方法允許您使用 `memory`、`time` 和 `threads` 選項來管理演算法的工作因子；然而，Laravel 管理的預設值對於大多數應用程式來說是可接受的：

    $hashed = Hash::make('password', [
        'memory' => 1024,
        'time' => 2,
        'threads' => 2,
    ]);

> [!NOTE]
> 有關這些選項的更多資訊，請參閱 [PHP 關於 Argon 雜湊的官方說明文件](https://secure.php.net/manual/en/function.password-hash.php)。

<a name="verifying-that-a-password-matches-a-hash"></a>
### 驗證密碼是否符合雜湊值

`Hash` Facade 提供的 `check` 方法允許您驗證給定的純文字字串是否與給定的雜湊值相符：

    if (Hash::check('plain-text', $hashedPassword)) {
        // The passwords match...
    }

<a name="determining-if-a-password-needs-to-be-rehashed"></a>
### 判斷密碼是否需要重新雜湊

`Hash` Facade 提供的 `needsRehash` 方法允許您判斷自密碼雜湊以來，雜湊器使用的工作因子是否已更改。有些應用程式選擇在應用程式的身份驗證過程中執行此檢查：

    if (Hash::needsRehash($hashed)) {
        $hashed = Hash::make('plain-text');
    }

<a name="hash-algorithm-verification"></a>
## 雜湊演算法驗證

為了防止雜湊演算法被操縱，Laravel 的 `Hash::check` 方法會首先驗證給定的雜湊值是否是使用應用程式選定的雜湊演算法生成的。如果演算法不同，將會拋出 `RuntimeException` 異常。

這對於大多數應用程式來說是預期的行為，因為雜湊演算法預計不會改變，而不同的演算法可能表示惡意攻擊。然而，如果您需要在應用程式中支援多種雜湊演算法，例如從一種演算法遷移到另一種演算法時，您可以透過將 `HASH_VERIFY` 環境變數設定為 `false` 來禁用雜湊演算法驗證：

```ini
HASH_VERIFY=false
```

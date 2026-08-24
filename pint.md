# Laravel Pint

- [簡介](#introduction)
- [安裝](#installation)
- [執行 Pint](#running-pint)
- [設定 Pint](#configuring-pint)
    - [預設集](#presets)
    - [規則](#rules)
    - [排除檔案 / 資料夾](#excluding-files-or-folders)
- [持續整合](#continuous-integration)
    - [GitHub Actions](#running-tests-on-github-actions)

<a name="introduction"></a>
## 簡介

[Laravel Pint](https://github.com/laravel/pint) 是一款專為極簡主義者打造、具備既定風格偏好的 PHP 程式碼風格修復工具。Pint 是建置在 [PHP CS Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 之上，能讓您輕鬆確保程式碼風格保持乾淨且一致。

所有新的 Laravel 應用程式都會自動安裝 Pint，因此您可以立即開始使用它。在預設情況下，Pint 不需要任何設定，並會遵循 Laravel 既定的撰寫風格來修復程式碼中的風格問題。


<a name="installation"></a>
## 安裝

近期發布的 Laravel 框架均已內建 Pint，因此通常不需要手動安裝。不過，若是較舊的應用程式，您可以透過 Composer 來安裝 Laravel Pint：

```shell
composer require laravel/pint --dev
```


<a name="running-pint"></a>
## 執行 Pint

您可以透過執行專案中 `vendor/bin` 目錄下的 `pint` 二進位檔，來讓 Pint 修復程式碼風格問題：

```shell
./vendor/bin/pint
```

如果您希望 Pint 以平行模式（實驗性功能）執行以提高效能，可以使用 `--parallel` 選項：

```shell
./vendor/bin/pint --parallel
```

平行模式還允許您透過 `--max-processes` 選項指定要執行的最大行程數。如果未提供此選項，Pint 將會使用您機器上所有可用的核心：

```shell
./vendor/bin/pint --parallel --max-processes=4
```

您也可以針對特定的檔案或目錄執行 Pint：

```shell
./vendor/bin/pint app/Models

./vendor/bin/pint app/Models/User.php
```

預設情況下，Pint 不會格式化 Blade 模板。如果您也想格式化 `.blade.php` 檔案，可以使用 `--blade` 選項，這會在不修改 `pint.json` 檔案的情況下，為本次執行啟用 [`Pint/laravel_blade`](#laravel-blade) 規則：

```shell
./vendor/bin/pint --blade
```

Pint 會顯示其更新的所有檔案的詳細列表。您可以在執行 Pint 時提供 `-v` 選項，以查看有關 Pint 變更的更多細節：

```shell
./vendor/bin/pint -v
```

如果您只希望 Pint 檢查程式碼是否有風格錯誤，而不實際修改檔案，可以使用 `--test` 選項。如果發現任何程式碼風格錯誤，Pint 將會傳回非零的離開碼：

```shell
./vendor/bin/pint --test
```

如果您希望 Pint 只修改根據 Git 與指定分支有所差異的檔案，可以使用 `--diff=[branch]` 選項。這可以有效地應用在您的 CI 環境中（例如 GitHub Actions），僅檢查新增或修改過的檔案來節省時間：

```shell
./vendor/bin/pint --diff=main
```

如果您希望 Pint 只修改根據 Git 尚未提交變更的檔案，可以使用 `--dirty` 選項：

```shell
./vendor/bin/pint --dirty
```

如果您希望 Pint 修復任何有程式碼風格錯誤的檔案，但如果有任何錯誤被修復時也以非零的離開碼離開，可以使用 `--repair` 選項：

```shell
./vendor/bin/pint --repair
```

<a name="configuring-pint"></a>
## 設定 Pint

如前所述，Pint 不需要任何設定。然而，若您希望自訂預設集、規則或要檢查的資料夾，可以在專案的根目錄中建立一個 `pint.json` 檔案：

```json
{
    "preset": "laravel"
}
```

此外，若您希望使用特定目錄下的 `pint.json`，可以在執行 Pint 時提供 `--config` 選項：

```shell
./vendor/bin/pint --config vendor/my-company/coding-style/pint.json
```


<a name="presets"></a>
### 預設集

預設集定義了一組可用於修復程式碼風格問題的規則。預設情況下，Pint 使用 `laravel` 預設集，該預設集會遵循 Laravel 既定設計慣例的程式碼風格來修復問題。不過，您也可以透過向 Pint 提供 `--preset` 選項來指定不同的預設集：

```shell
./vendor/bin/pint --preset psr12
```

若您願意，也可以在專案的 `pint.json` 檔案中設定預設集：

```json
{
    "preset": "psr12"
}
```

Pint 目前支援的預設集有：`laravel`、`per`、`psr12`、`symfony` 與 `empty`。


<a name="rules"></a>
### 規則

規則是 Pint 用來修復程式碼風格問題的風格指引。如上所述，預設集是預先定義好的規則組合，對於大多數 PHP 專案來說已經非常完美，因此您通常不需要擔心其中包含的各個單獨規則。

但是，若您希望，可以在 `pint.json` 檔案中啟用或停用特定規則，或者使用 `empty` 預設集並從頭定義所有規則：

```json
{
    "preset": "laravel",
    "rules": {
        "simplified_null_return": true,
        "array_indentation": false,
        "new_with_parentheses": {
            "anonymous_class": true,
            "named_class": true
        }
    }
}
```

Pint 是建立在 [PHP CS Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 之上的。因此，您可以使用其任何規則來修復專案中的程式碼風格問題：[PHP CS Fixer 設定檢視器](https://mlocati.github.io/php-cs-fixer-configurator)。


<a name="custom-rules"></a>
#### 自訂規則

除了 PHP CS Fixer 規則之外，Pint 還提供了以 `Pint/` 為前綴的自訂規則。這些規則預設並未啟用，但您可以在 `pint.json` 檔案中啟用它們。


<a name="laravel-blade"></a>
##### `Pint/laravel_blade`

此規則用於格式化您的 Blade 模板，將一致的縮排、空格與屬性格式套用至您的 `.blade.php` 檔案。預設情況下，Pint 不會格式化 Blade 檔案，因此您必須在 `pint.json` 檔案中啟用此規則以選擇加入：

```json
{
    "preset": "laravel",
    "rules": {
        "Pint/laravel_blade": true
    }
}
```

啟用後，每當 Pint 執行時，除了 PHP 檔案之外，也會格式化您的 Blade 模板：

```shell
./vendor/bin/pint
```

或者，如果您想在不修改 `pint.json` 檔案的情況下為單次執行啟用此規則，可以使用 `--blade` 選項：

```shell
./vendor/bin/pint --blade
```

在底層，此規則使用了 [Prettier](https://prettier.io) 以及 `prettier-plugin-blade` 和 `prettier-plugin-tailwindcss` 外掛，因此您的電腦上必須安裝 [Node.js](https://nodejs.org)。首次在啟用此規則的情況下執行 Pint 時，Pint 會偵測任何缺失的 Prettier 依賴項目並提示您安裝它們。

> [!NOTE]
> 此規則會自動跳過通常依賴自身格式設定的檔案，例如 [Laravel Boost](https://github.com/laravel/boost) 指引，以及位於 `resources/views/emails` 和 `resources/views/mail` 目錄中的 Email 視圖。


<a name="phpdoc-type-annotations-only"></a>
##### `Pint/phpdoc_type_annotations_only`

此規則會移除您程式碼中的所有註解和 docblock 敘述，僅保留包含 `@` 標註的行，例如 `@param`、`@return`、`@var`、`@phpstan-type` 等：

```php
/**
 * Get the posts for the user. [tl! remove]
 * [tl! remove]
 * @return HasMany<Post, $this>
 */
public function posts(): HasMany
```

不帶有 `@` 標註的單行註解和區塊註解將被完全移除。如果您想保留特定註解，可以在其前面加上 `@note`、`@warning` 或 `@todo` 前綴：

```php
// @note This comment will be preserved.
```

要啟用此規則，請將其新增至您的 `pint.json` 檔案中：

```json
{
    "preset": "laravel",
    "rules": {
        "Pint/phpdoc_type_annotations_only": true
    }
}
```

> [!NOTE]
> 此規則會自動跳過 `config` 目錄中的檔案，因為設定檔通常需要依賴註解來進行說明。


<a name="excluding-files-or-folders"></a>
### 排除檔案 / 資料夾

預設情況下，Pint 會檢查專案中除了 `vendor` 目錄以外的所有 `.php` 檔案。如果您希望排除更多資料夾，可以使用 `exclude` 設定選項：

```json
{
    "exclude": [
        "my-specific/folder"
    ]
}
```

如果您希望排除所有包含給定名稱樣式的檔案，可以使用 `notName` 設定選項：

```json
{
    "notName": [
        "*-my-file.php"
    ]
}
```

如果您想透過提供檔案的精確路徑來排除檔案，可以使用 `notPath` 設定選項：

```json
{
    "notPath": [
        "path/to/excluded-file.php"
    ]
}
```


<a name="continuous-integration"></a>
## 持續整合


<a name="running-tests-on-github-actions"></a>
### GitHub Actions

要使用 Laravel Pint 自動檢查您的專案程式碼風格，您可以設定 [GitHub Actions](https://github.com/features/actions)，以便在有新程式碼推送至 GitHub 時執行 Pint。首先，請務必在 GitHub 的 **Settings > Actions > General > Workflow permissions** 中授予工作流程「Read and write permissions」。然後，建立一個包含以下內容的 `.github/workflows/lint.yml` 檔案：

```yaml
name: Fix Code Style

on: [push]

jobs:
  lint:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true
      matrix:
        php: [8.4]

    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          tools: pint

      - name: Run Pint
        run: pint

      - name: Commit linted files
        uses: stefanzweifel/git-auto-commit-action@v6
```
# 貢獻指南

- [錯誤回報](#bug-reports)
- [支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [哪個分支？](#which-branch)
- [編譯的資產](#compiled-assets)
- [安全性弱點](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤回報

為鼓勵積極協作，Laravel 強烈建議提交 Pull Request，而不僅僅是錯誤回報。Pull Request 僅會在標記為「準備好審核」(而非「草稿」狀態) 且所有新功能的測試皆通過時才會進行審核。擱置中、不活躍且仍處於「草稿」狀態的 Pull Request 將在幾天後關閉。

然而，如果您提交錯誤回報，您的問題應包含標題和清晰的問題描述。您還應盡可能包含所有相關資訊以及一個能展示問題的程式碼範例。錯誤回報的目標是讓您和他人能夠輕鬆地重現該錯誤並開發修復方案。

請記住，錯誤回報是為了讓遇到相同問題的其他人能夠與您合作解決問題。不要期望錯誤回報會自動獲得任何關注，或期望其他人會立即修復它。建立錯誤回報有助於您和他人開始著手解決問題。如果您想出力，可以透過修復[我們議題追蹤器中列出的任何錯誤](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel)來提供幫助。您必須通過 GitHub 驗證才能查看所有 Laravel 的議題。

如果您在使用 Laravel 時發現不正確的 DocBlock、PHPStan 或 IDE 警告，請不要建立 GitHub 議題。相反地，請提交一個 Pull Request 來修復該問題。

Laravel 原始碼在 GitHub 上進行管理，並且每個 Laravel 專案都有其獨立的儲存庫：

<div class="content-list" markdown="1">

- [Laravel Application](https://github.com/laravel/laravel)
- [Laravel Art](https://github.com/laravel/art)
- [Laravel Boost](https://github.com/laravel/boost)
- [Laravel Documentation](https://github.com/laravel/docs)
- [Laravel Dusk](https://github.com/laravel/dusk)
- [Laravel Cashier Stripe](https://github.com/laravel/cashier)
- [Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle)
- [Laravel Echo](https://github.com/laravel/echo)
- [Laravel Envoy](https://github.com/laravel/envoy)
- [Laravel Folio](https://github.com/laravel/folio)
- [Laravel Framework](https://github.com/laravel/framework)
- [Laravel Homestead](https://github.com/laravel/homestead) ([建構腳本](https://github.com/laravel/settler))
- [Laravel Horizon](https://github.com/laravel/horizon)
- [Laravel Passport](https://github.com/laravel/passport)
- [Laravel Pennant](https://github.com/laravel/pennant)
- [Laravel Pint](https://github.com/laravel/pint)
- [Laravel Prompts](https://github.com/laravel/prompts)
- [Laravel Reverb](https://github.com/laravel/reverb)
- [Laravel Sail](https://github.com/laravel/sail)
- [Laravel Sanctum](https://github.com/laravel/sanctum)
- [Laravel Scout](https://github.com/laravel/scout)
- [Laravel Socialite](https://github.com/laravel/socialite)
- [Laravel Telescope](https://github.com/laravel/telescope)
- [Laravel Livewire Starter Kit](https://github.com/laravel/livewire-starter-kit)
- [Laravel React Starter Kit](https://github.com/laravel/react-starter-kit)
- [Laravel Vue Starter Kit](https://github.com/laravel/vue-starter-kit)

</div>


<a name="support-questions"></a>
## 支援問題

Laravel 的 GitHub 議題追蹤器不旨在提供 Laravel 協助或支援。相反地，請使用以下其中一個管道：

<div class="content-list" markdown="1">

- [GitHub Discussions](https://github.com/laravel/framework/discussions)
- [Laracasts 論壇](https://laracasts.com/discuss)
- [Laravel.io 論壇](https://laravel.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/laravel)
- [Discord](https://discord.gg/laravel)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=artisan&channels=#laravel)

</div>


<a name="core-development-discussion"></a>
## 核心開發討論

您可以在 Laravel framework 儲存庫的 [GitHub 討論區](https://github.com/laravel/framework/discussions)中提出新功能或改進現有 Laravel 行為的建議。如果您提出新功能，請務必願意實作完成該功能所需的部分程式碼。

關於錯誤、新功能和現有功能實作的非正式討論，會在 [Laravel Discord 伺服器](https://discord.gg/laravel)的 `#internals` 頻道中進行。Laravel 的維護者 Taylor Otwell 通常會在週間上午 8 點至下午 5 點 (UTC-06:00 或 America/Chicago 時間) 出現在頻道中，其他時間則會不定時出現。


<a name="which-branch"></a>
## 哪個分支？

**所有**錯誤修復都應該發送到支援錯誤修復的最新版本 (目前為 `12.x`)。錯誤修復**絕不**應發送到 `master` 分支，除非它們修復了僅存在於即將發布版本中的功能。

與當前版本**完全向下相容**的**次要**功能可以發送到最新的穩定分支 (目前為 `12.x`)。

帶有破壞性變更的**主要**新功能或功能應始終發送到 `master` 分支，該分支包含即將發布的版本。


<a name="compiled-assets"></a>
## 編譯的資產

如果您提交的變更會影響編譯過的檔案，例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 大多數的檔案，請不要提交這些編譯過的檔案。由於它們體積龐大，維護者無法實際審核。這可能會被利用作為將惡意程式碼注入 Laravel 的方式。為了防禦性地防止這種情況，所有編譯過的檔案將由 Laravel 維護者產生並提交。


<a name="security-vulnerabilities"></a>
## 安全性弱點

如果您在 Laravel 中發現安全性弱點，請寄送電子郵件至 <a href="mailto:taylor@laravel.com">taylor@laravel.com</a> 給 Taylor Otwell。所有安全性弱點都將會被迅速處理。


<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 程式碼標準和 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是一個有效的 Laravel 文件區塊範例。請注意，`@param` 屬性後面接著兩個空格，然後是參數類型，再接著兩個空格，最後是變數名稱：

```php
/**
 * Register a binding with the container.
 *
 * @param  string|array  $abstract
 * @param  \Closure|string|null  $concrete
 * @param  bool  $shared
 * @return void
 *
 * @throws \Exception
 */
public function bind($abstract, $concrete = null, $shared = false)
{
    // ...
}
```

當由於使用原生類型而導致 `@param` 或 `@return` 屬性冗餘時，它們可以被移除：

```php
/**
 * Execute the job.
 */
public function handle(AudioProcessor $processor): void
{
    // ...
}
```

然而，當原生類型是泛型時，請透過使用 `@param` 或 `@return` 屬性來指定泛型類型：

```php
/**
 * Get the attachments for the message.
 *
 * @return array<int, \Illuminate\Mail\Mailables\Attachment>
 */
public function attachments(): array
{
    return [
        Attachment::fromStorage('/path/to/file'),
    ];
}
```


<a name="styleci"></a>
### StyleCI

如果您的程式碼風格不完美，也無需擔心！Pull Request 合併後，[StyleCI](https://styleci.io/) 將自動把所有風格修復合併到 Laravel 儲存庫中。這讓我們能夠專注於貢獻的內容，而非程式碼風格。

<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則源自於 Ruby 的行為準則。任何對行為準則的違規都可以向 Taylor Otwell (taylor@laravel.com) 回報：

<div class="content-list" markdown="1">

- 參與者應寬容不同意見。
- 參與者必須確保其言行沒有人身攻擊及貶低性言論。
- 在解讀他人的言行時，參與者應始終抱持良好意圖。
- 任何可被合理視為騷擾的行為都不會被容忍。

</div>
# 貢獻指南

- [錯誤報告](#bug-reports)
- [支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [該使用哪個分支？](#which-branch)
- [編譯後的靜態資源](#compiled-assets)
- [AI 生成的貢獻](#ai-generated-contributions)
- [安全漏洞](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤報告

為了鼓勵積極協作，Laravel 強烈建議提交 Pull Request，而不僅僅是錯誤報告。只有標記為「準備好接受審閱 (ready for review)」（非「草稿 (draft)」狀態）且新功能的所有測試都通過的 Pull Request 才會被審查。長期處於「草稿」狀態且非活動的 Pull Request 將在幾天後被關閉。

然而，如果您提交錯誤報告，您的 Issue 應包含標題和對問題的清晰描述。您還應該盡可能提供相關資訊，以及一個能展示該問題的程式碼範例。錯誤報告的目標是讓您自己和他人都能輕鬆地重現錯誤並開發修復程式。

請記住，建立錯誤報告是希望其他遇到相同問題的人能與您協作解決。不要指望錯誤報告會自動引起關注，或他人會立即跳出來修復它。建立錯誤報告是為了幫助您自己和他人開始修復問題。如果您想出一份力，可以透過修復[我們問題追蹤器 (issue trackers) 中列出的任何錯誤](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel)來提供幫助。您必須登入 GitHub 才能查看所有 Laravel 的 Issue。

如果您在開發 Laravel 時注意到不正確的 DocBlock、PHPStan 或 IDE 警告，請不要建立 GitHub Issue。相反地，請提交 Pull Request 來修正該問題。

Laravel 的原始碼託管在 GitHub 上，每個 Laravel 專案都有各自的儲存庫：

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


<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 編碼標準以及 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是一個有效的 Laravel 文件區塊範例。請注意，`@param` 屬性後方接續兩個空格，接著是引數型別，再接兩個空格，最後才是變數名稱：

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

當 `@param` 或 `@return` 屬性因為使用了原生型別而顯得冗餘時，可以將其移除：

```php
/**
 * Execute the job.
 */
public function handle(AudioProcessor $processor): void
{
    // ...
}
```

然而，當原生型別為泛型時，請透過 `@param` 或 `@return` 屬性來指定泛型型別：

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

即便您的程式碼風格不夠完美也不必擔心！[StyleCI](https://styleci.io/) 會在 Pull Request 合併後，自動將任何風格修復合併至 Laravel 儲存庫中。這讓我們能夠專注於貢獻的內容，而非程式碼風格。


<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則衍生自 Ruby 的行為準則。任何違反行為準則的情況都可以向 Taylor Otwell (taylor@laravel.com) 舉報：

<div class="content-list" markdown="1">

- 參與者應包容不同的觀點。
- 參與者必須確保其言行不含人身攻擊或貶低性的個人言論。
- 在解讀他人的言行時，參與者應始終假定其出發點是良善的。
- 任何可被合理視為騷擾的行為都是不被允許的。

</div>
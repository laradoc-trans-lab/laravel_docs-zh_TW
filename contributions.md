# 貢獻指南

- [錯誤報告](#bug-reports)
- [支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [選擇分支](#which-branch)
- [編譯後的資源](#compiled-assets)
- [AI 生成的貢獻](#ai-generated-contributions)
- [安全性漏洞](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤報告

為了鼓勵積極的協作，Laravel 強烈建議提交 pull request，而不僅僅是錯誤報告。只有在標記為「ready for review」（而非處於「draft」狀態）且所有新功能的測試皆通過時，pull request 才會被審核。長期處於「draft」狀態且不活躍的 pull request 將在幾天後被關閉。

然而，如果您提交錯誤報告，您的 issue 應該包含標題和對問題的清晰描述。您還應該盡可能提供相關資訊以及能演示該問題的程式碼範例。錯誤報告的目標是讓您自己以及其他人能夠輕鬆地重現錯誤並開發修復方案。

請記得，建立錯誤報告是希望遇到相同問題的人能夠與您共同協作解決。不要期待錯誤報告會自動產生任何活動，或他人會立即跳出來修復它。建立錯誤報告旨在幫助您和他人開始修復問題的過程。如果您想幫忙，您可以嘗試修復 [問題追蹤器中列出的任何錯誤](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel)。您必須通過 GitHub 認證才能查看 Laravel 的所有 issue。

如果您在使用 Laravel 時發現不正確的 DocBlock、PHPStan 或 IDE 警告，請不要建立 GitHub issue。相反地，請提交一個 pull request 來修復此問題。

Laravel 的原始碼在 GitHub 上管理，每個 Laravel 專案都有對應的儲存庫：

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
- [Laravel Svelte Starter Kit](https://github.com/laravel/svelte-starter-kit)
- [Laravel Vue Starter Kit](https://github.com/laravel/vue-starter-kit)

</div>


<a name="support-questions"></a>
## 支援問題

Laravel 的 GitHub 問題追蹤器並非旨在提供 Laravel 的協助或支援。請改用以下其中一個管道：

<div class="content-list" markdown="1">

- [GitHub Discussions](https://github.com/laravel/framework/discussions)
- [Laracasts Forums](https://laracasts.com/discuss)
- [Laravel.io Forums](https://laravel.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/laravel)
- [Discord](https://discord.gg/laravel)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=artisan&channels=#laravel)

</div>


<a name="core-development-discussion"></a>
## 核心開發討論

您可以在 Laravel framework 儲存庫的 [GitHub 討論版](https://github.com/laravel/framework/discussions) 中提議新功能或對現有 Laravel 行為的改進。如果您提議了一項新功能，請願意實作完成該功能所需的部分程式碼。

關於錯誤、新功能以及現有功能實作的非正式討論，在 [Laravel Discord 伺服器](https://discord.gg/laravel) 的 `#internals` 頻道中進行。Laravel 的維護者 Taylor Otwell 通常在工作日的 8am-5pm (UTC-06:00 或 America/Chicago) 出現在該頻道中，其他時間則會不定期出現。


<a name="which-branch"></a>
## 選擇分支

**所有**錯誤修正都應發送到支援錯誤修正的最新版本（目前為 `12.x`）。除非修正的是僅存在於即將發布版本中的功能，否則**絕不**應將錯誤修正發送到 `master` 分支。

與目前版本**完全向下相容**的**次要**功能可以發送到最新的穩定分支（目前為 `12.x`）。

**重大**新功能或具有破壞性變更的功能應始終發送到包含即將發布版本的 `master` 分支。


<a name="compiled-assets"></a>
## 編譯後的資源

如果您提交的變更會影響編譯後的檔案（例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 中的大多數檔案），請不要提交編譯後的檔案。由於其檔案體積較大，維護者實際上無法對其進行審核。這可能會被利用作為將惡意程式碼注入 Laravel 的手段。為了防禦性地防止此情況，所有編譯後的檔案將由 Laravel 維護者生成並提交。


<a name="ai-generated-contributions"></a>
## AI 生成的貢獻

我們感謝提交給 Laravel 的每一個 pull request。然而，主要由 AI 生成且缺乏人類深思熟慮審查與考量的貢獻是不被接受的。

如果您選擇使用 AI 工具來輔助您的貢獻，在提交之前，產出的程式碼**必須**由您本人經過徹底的審查、測試並充分理解。

**不會容許大量建立完全由 AI 生成的 issue 或 pull request。** 此類 pull request 將在不經審核的情況下被關閉，且貢獻者可能會被封鎖該儲存庫。

我們鼓勵貢獻者熟悉現有的程式碼庫，參與社群，並提交反映其自身理解以及對所解決問題經過仔細考量的 pull request。


<a name="security-vulnerabilities"></a>
## 安全性漏洞

如果您發現 Laravel 中存在安全性漏洞，請發送電子郵件給 Taylor Otwell：<a href="mailto:taylor@laravel.com">taylor@laravel.com</a>。所有安全性漏洞都將得到迅速處理。

<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 程式碼標準與 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是一個有效的 Laravel 說明區塊範例。請注意 `@param` 屬性後方接著兩個空格、引數類型、再接著兩個空格，最後才是變數名稱：

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

當由於使用了原生類型而導致 `@param` 或 `@return` 屬性變得冗餘時，可以將其移除：

```php
/**
 * Execute the job.
 */
public function handle(AudioProcessor $processor): void
{
    // ...
}
```

然而，當原生類型為泛型時，請透過 `@param` 或 `@return` 屬性來指定泛型類型：

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

不要擔心您的程式碼風格不完美！[StyleCI](https://styleci.io/) 會在 pull request 合併後，自動將任何風格修正合併到 Laravel 儲存庫中。這讓我們能專注於貢獻的內容而非程式碼風格。


<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則衍生自 Ruby 的行為準則。任何違反行為準則的情況都可以回報給 Taylor Otwell (taylor@laravel.com)：

<div class="content-list" markdown="1">

- 參與者應對對立觀點保持包容。
- 參與者必須確保其語言和行為中沒有人身攻擊或貶低個人的言論。
- 在解讀他人的言論和行為時，參與者應始終假設對方是出於好意。
- 任何被合理視為騷擾的行為將不被容忍。

</div>
# 版本說明

- [版本編號方案](#versioning-scheme)
- [支援政策](#support-policy)
- [Laravel 12](#laravel-12)

<a name="versioning-scheme"></a>
## 版本編號方案

Laravel 與其官方第一方套件皆遵循 [Semantic Versioning](https://semver.org) (語義化版本)。框架的主版本 (Major releases) 每年發佈一次（約於第一季），而次版本 (Minor releases) 與修補版本 (Patch releases) 則可能每週發佈。次版本與修補版本**絕對不應**包含中斷性變更 (Breaking changes)。

當您在應用程式或套件中引用 Laravel 框架或其元件時，應始終使用版本約束，例如 `^12.0`，因為 Laravel 的主版本確實會包含中斷性變更。然而，我們致力於確保您始終能在一天之內完成至新主版本的升級。


<a name="named-arguments"></a>
#### 具名引數

[具名引數 (Named arguments)](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) 不在 Laravel 的回溯相容性指南範圍內。為了改進 Laravel 程式碼庫，我們可能會在必要時重新命名函式參數。因此，在呼叫 Laravel 方法時使用具名引數應謹慎行事，並理解參數名稱在未來可能會變更。


<a name="support-policy"></a>
## 支援政策

對於所有的 Laravel 版本，Bug 修復提供 18 個月，安全性修復提供 2 年。對於所有額外的函式庫，只有最新的主版本會收到 Bug 修復。此外，請查看 [Laravel 支援](/docs/{{version}}/database#introduction) 的資料庫版本。

<div class="overflow-auto">

| 版本 | PHP (*)   | 發佈日期 | Bug 修復至 | 安全性修復至 |
| ------- |-----------| ------------------- | ------------------- | -------------------- |
| 10      | 8.1 - 8.3 | 2023 年 2 月 14 日 | 2024 年 8 月 6 日 | 2025 年 2 月 4 日 |
| 11      | 8.2 - 8.4 | 2024 年 3 月 12 日 | 2025 年 9 月 3 日 | 2026 年 3 月 12 日 |
| 12      | 8.2 - 8.5 | 2025 年 2 月 24 日 | 2026 年 8 月 13 日 | 2027 年 2 月 24 日 |
| 13      | 8.3 - 8.5 | 2026 年第一季 | 2027 年第三季 | 2028 年第一季 |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>終止支援 (End of life)</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>僅安全性修復</div>
    </div>
</div>

(*) 支援的 PHP 版本


<a name="laravel-12"></a>
## Laravel 12

Laravel 12 延續了 Laravel 11.x 所做的改進，更新了上游依賴項目，並為 React、Svelte、Vue 和 Livewire 引入了全新的入門套件，包括可選擇使用 [WorkOS AuthKit](https://authkit.com) 進行使用者認證。這些入門套件的 WorkOS 變體版本提供了社群認證、通行金鑰 (Passkeys) 以及 SSO 支援。


<a name="minimal-breaking-changes"></a>
### 最少的中斷性變更

在此發佈週期中，我們的重點大多放在盡可能減少中斷性變更。相反地，我們致力於在全年發佈持續的開發體驗 (Quality-of-life) 改進，且不會破壞現有的應用程式。

因此，Laravel 12 的發佈是一個相對較小的「維護版本」，旨在升級現有的依賴項目。有鑒於此，大多數的 Laravel 應用程式可以直接升級到 Laravel 12，而無需更改任何應用程式程式碼。


<a name="new-application-starter-kits"></a>
### 全新的應用程式入門套件

Laravel 12 為 React、Svelte、Vue 和 Livewire 引入了全新的 [應用程式入門套件](/docs/{{version}}/starter-kits)。React、Svelte 和 Vue 的入門套件使用了 Inertia 2、TypeScript、[shadcn/ui](https://ui.shadcn.com) 和 Tailwind，而 Livewire 入門套件則使用了基於 Tailwind 的 [Flux UI](https://fluxui.dev) 元件庫以及 Laravel Volt。

React、Svelte、Vue 和 Livewire 入門套件皆利用 Laravel 內建的認證系統提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們也為每個入門套件引入了由 [WorkOS AuthKit 驅動](https://authkit.com) 的變體版本，提供社群認證、通行金鑰和 SSO 支援。WorkOS 為每月活躍用戶數少於 100 萬的應用程式提供免費認證服務。

隨著全新應用程式入門套件的推出，Laravel Breeze 和 Laravel Jetstream 將不再接收額外的更新。

若要開始使用我們全新的入門套件，請查看 [入門套件文件](/docs/{{version}}/starter-kits)。
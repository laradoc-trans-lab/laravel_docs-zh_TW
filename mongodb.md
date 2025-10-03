# MongoDB

- [介紹](#introduction)
- [安裝](#installation)
    - [MongoDB 驅動程式](#mongodb-driver)
    - [啟動 MongoDB 伺服器](#starting-a-mongodb-server)
    - [安裝 Laravel MongoDB 軟體套件](#install-the-laravel-mongodb-package)
- [設定](#configuration)
- [功能](#features)

<a name="introduction"></a>
## 介紹

[MongoDB](https://www.mongodb.com/resources/products/fundamentals/why-use-mongodb) 是最受歡迎的 NoSQL 文件導向資料庫之一，以其高寫入負載（適用於分析或 IoT）和高可用性（易於設定具自動失效切換功能的複本集）而聞名。它還可以輕鬆地將資料庫分片以實現水平擴展性，並擁有強大的查詢語言，可用於聚合、文字搜尋或地理空間查詢。

MongoDB 資料庫中的每條記錄不是像 SQL 資料庫那樣儲存在行或列的表格中，而是以 BSON（資料的二進位表示法）描述的文件形式存在。應用程式可以 JSON 格式檢索此資訊。它支援多種資料型別，包括文件、陣列、內嵌文件和二進位資料。

在使用 MongoDB 與 Laravel 之前，我們建議透過 Composer 安裝並使用 `mongodb/laravel-mongodb` 軟體套件。`laravel-mongodb` 軟體套件由 MongoDB 官方維護，雖然 MongoDB 透過 MongoDB 驅動程式原生支援 PHP，但 [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) 軟體套件提供了與 Eloquent 和其他 Laravel 功能更豐富的整合：

```shell
composer require mongodb/laravel-mongodb
```

<a name="installation"></a>
## 安裝

<a name="mongodb-driver"></a>
### MongoDB 驅動程式

若要連線到 MongoDB 資料庫，需要 `mongodb` PHP 擴充功能。如果您正在使用 [Laravel Herd](https://herd.laravel.com) 或透過 `php.new` 安裝 PHP 進行本地端開發，您的系統上應該已經安裝了此擴充功能。但是，如果您需要手動安裝此擴充功能，可以透過 PECL 進行：

```shell
pecl install mongodb
```

如需更多關於安裝 MongoDB PHP 擴充功能的資訊，請查閱 [MongoDB PHP 擴充功能安裝說明](https://www.php.net/manual/en/mongodb.installation.php)。

<a name="starting-a-mongodb-server"></a>
### 啟動 MongoDB 伺服器

MongoDB 社群伺服器可用於在本地端執行 MongoDB，並可安裝於 Windows、macOS、Linux，或作為 Docker 容器。若要了解如何安裝 MongoDB，請參閱 [官方 MongoDB 社群版安裝指南](https://docs.mongodb.com/manual/administration/install-community/)。

MongoDB 伺服器的連線字串可以在您的 `.env` 檔案中設定：

```ini
MONGODB_URI="mongodb://localhost:27017"
MONGODB_DATABASE="laravel_app"
```

若要將 MongoDB 部署在雲端，請考慮使用 [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)。
若要從您的應用程式本地端存取 MongoDB Atlas 集群，您需要在 [集群的網路設定中新增您的 IP 位址](https://www.mongodb.com/docs/atlas/security/add-ip-address-to-list/) 至專案的 IP 存取清單。

MongoDB Atlas 的連線字串也可以在您的 `.env` 檔案中設定：

```ini
MONGODB_URI="mongodb+srv://<username>:<password>@<cluster>.mongodb.net/<dbname>?retryWrites=true&w=majority"
MONGODB_DATABASE="laravel_app"
```

<a name="install-the-laravel-mongodb-package"></a>
### 安裝 Laravel MongoDB 軟體套件

最後，使用 Composer 安裝 Laravel MongoDB 軟體套件：

```shell
composer require mongodb/laravel-mongodb
```

> [!NOTE]
> 如果 `mongodb` PHP 擴充功能未安裝，此軟體套件的安裝將會失敗。PHP 設定在 CLI 和網頁伺服器之間可能有所不同，因此請確保在兩種設定中都已啟用該擴充功能。

<a name="configuration"></a>
## 設定

您可以透過應用程式的 `config/database.php` 設定檔來設定您的 MongoDB 連線。在此檔案中，新增一個使用 `mongodb` 驅動程式的 `mongodb` 連線：

```php
'connections' => [
    'mongodb' => [
        'driver' => 'mongodb',
        'dsn' => env('MONGODB_URI', 'mongodb://localhost:27017'),
        'database' => env('MONGODB_DATABASE', 'laravel_app'),
    ],
],
```

<a name="features"></a>
## 功能

設定完成後，您可以在應用程式中使用 `mongodb` 軟體套件和資料庫連線，以利用各種強大的功能：

- [使用 Eloquent](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/eloquent-models/)，模型可以儲存在 MongoDB 集合中。除了標準的 Eloquent 功能之外，Laravel MongoDB 軟體套件還提供了額外功能，例如內嵌關聯。該軟體套件還提供了直接存取 MongoDB 驅動程式的權限，可用於執行原始查詢和聚合管道等操作。
- [使用查詢建構器編寫複雜的查詢](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/query-builder/)。
- `mongodb` [快取驅動程式](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/) 已最佳化以利用 MongoDB 功能，例如 TTL 索引來自動清除過期的快取項目。
- [使用 `mongodb` 佇列驅動程式分派並處理佇列工作](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/)。
- [透過適用於 Flysystem 的 GridFS 轉接器](https://flysystem.thephpleague.com/docs/adapter/gridfs/)，[將檔案儲存在 GridFS 中](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/filesystems/)。
- 大多數使用資料庫連線或 Eloquent 的第三方軟體套件都可以與 MongoDB 一起使用。

若要繼續學習如何使用 MongoDB 和 Laravel，請參閱 MongoDB 的 [快速入門指南](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/quick-start/)。
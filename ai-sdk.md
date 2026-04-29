# Laravel AI SDK

- [介紹](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
    - [自定義基礎 URL](#custom-base-urls)
    - [供應商支援](#provider-support)
- [AI 代理 (Agents)](#agents)
    - [提示 (Prompting)](#prompting)
    - [對話上下文](#conversation-context)
    - [結構化輸出](#structured-output)
    - [附件](#attachments)
    - [串流](#streaming)
    - [廣播](#broadcasting)
    - [佇列化](#queueing)
    - [工具](#tools)
    - [供應商工具](#provider-tools)
    - [中介層](#middleware)
    - [匿名代理](#anonymous-agents)
    - [代理設定](#agent-configuration)
    - [供應商選項](#provider-options)
- [圖片](#images)
- [音訊 (TTS)](#audio)
- [逐字稿 (STT)](#transcription)
- [Embeddings](#embeddings)
    - [查詢 Embeddings](#querying-embeddings)
    - [快取 Embeddings](#caching-embeddings)
- [重新排序 (Reranking)](#reranking)
- [檔案](#files)
- [向量存儲 (Vector Stores)](#vector-stores)
    - [將檔案新增至存儲](#adding-files-to-stores)
- [容錯移轉 (Failover)](#failover)
- [測試](#testing)
    - [AI 代理](#testing-agents)
    - [圖片](#testing-images)
    - [音訊](#testing-audio)
    - [逐字稿](#testing-transcriptions)
    - [Embeddings](#testing-embeddings)
    - [重新排序 (Reranking)](#testing-reranking)
    - [檔案](#testing-files)
    - [向量存儲 (Vector Stores)](#testing-vector-stores)
- [事件](#events)

<a name="introduction"></a>
## 介紹

[Laravel AI SDK](https://github.com/laravel/ai) 為與 AI 供應商（如 OpenAI、Anthropic、Gemini 等）進行互動提供了一套統一且具表達力的 API。透過 AI SDK，您可以建立具備工具和結構化輸出的智慧代理，生成圖片，合成與轉錄音訊，建立向量 Embeddings 等等 —— 且這一切都使用一致、對 Laravel 友善的介面。


<a name="installation"></a>
## 安裝

您可以使用 Composer 安裝 Laravel AI SDK：

```shell
composer require laravel/ai
```

接下來，您應該使用 `vendor:publish` Artisan 指令來發布 AI SDK 的設定檔與遷移 (Migration) 檔案：

```shell
php artisan vendor:publish --provider="Laravel\Ai\AiServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這將會建立 `agent_conversations` 與 `agent_conversation_messages` 資料表，AI SDK 會使用這些資料表來支援對話存儲功能：

```shell
php artisan migrate
```


<a name="configuration"></a>
### 設定

您可以在應用程式的 `config/ai.php` 設定檔中定義 AI 供應商憑證，或者在應用程式的 `.env` 檔案中設定為環境變數：

```ini
ANTHROPIC_API_KEY=
AZURE_OPENAI_API_KEY=
COHERE_API_KEY=
DEEPSEEK_API_KEY=
ELEVENLABS_API_KEY=
GEMINI_API_KEY=
GROQ_API_KEY=
MISTRAL_API_KEY=
OLLAMA_API_KEY=
OPENAI_API_KEY=
OPENROUTER_API_KEY=
JINA_API_KEY=
VOYAGEAI_API_KEY=
XAI_API_KEY=
```

用於文字、圖片、音訊、逐字稿與 Embeddings 的預設模型也可以在應用程式的 `config/ai.php` 設定檔中進行設定。


<a name="custom-base-urls"></a>
### 自定義基礎 URL

預設情況下，Laravel AI SDK 會直接連接到各個供應商的公開 API 端點。然而，您可能需要透過不同的端點來路由請求 —— 例如，當使用代理服務來集中管理 API 金鑰、實施速率限制，或是透過企業網關路由流量時。

您可以透過在供應商設定中加入 `url` 參數來設定自定義基礎 URL：

```php
'providers' => [
    'openai' => [
        'driver' => 'openai',
        'key' => env('OPENAI_API_KEY'),
        'url' => env('OPENAI_BASE_URL'),
    ],

    'anthropic' => [
        'driver' => 'anthropic',
        'key' => env('ANTHROPIC_API_KEY'),
        'url' => env('ANTHROPIC_BASE_URL'),
    ],
],
```

這在透過代理服務（例如 LiteLLM 或 Azure OpenAI Gateway）路由請求，或是使用備用端點時非常有用。

OpenAI、Anthropic、Gemini、Groq、Cohere、DeepSeek、xAI 與 OpenRouter 等供應商皆支援自定義基礎 URL。


<a name="provider-support"></a>
### 供應商支援

AI SDK 的各項功能支援多種供應商。下表總結了每項功能可用的供應商：

| 功能 | 供應商 |
|---|---|
| 文字 | OpenAI, Anthropic, Gemini, Azure, Groq, xAI, DeepSeek, Mistral, Ollama |
| 圖片 | OpenAI, Gemini, xAI |
| TTS | OpenAI, ElevenLabs |
| STT | OpenAI, ElevenLabs, Mistral |
| Embeddings | OpenAI, Gemini, Azure, Cohere, Mistral, Jina, VoyageAI |
| 重新排序 | Cohere, Jina |
| 檔案 | OpenAI, Anthropic, Gemini |

您可以使用 `Laravel\Ai\Enums\Lab` 列舉 (Enum) 在程式碼中引用供應商，而不需要使用純字串：

```php
use Laravel\Ai\Enums\Lab;

Lab::Anthropic;
Lab::OpenAI;
Lab::Gemini;
// ...
```

<a name="agents"></a>
## AI 代理 (Agents)

AI 代理 (Agents) 是在 Laravel AI SDK 中與 AI 供應商互動的基本構件。每個 AI 代理都是一個專門的 PHP 類別，封裝了與大型語言模型互動所需的指令、對話上下文、工具和輸出架構 (Schema)。將 AI 代理視為一個專門的助手 —— 像是銷售教練、文件分析師或支援機器人 —— 您只需設定一次，即可在應用程式各處根據需要進行提示 (Prompt)。

您可以使用 `make:agent` Artisan 指令建立一個 AI 代理：

```shell
php artisan make:agent SalesCoach

php artisan make:agent SalesCoach --structured
```

在產生的代理類別中，您可以定義系統提示詞 (System Prompt) / 指令、訊息上下文、可用工具以及輸出架構（如果適用）：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Tools\RetrievePreviousTranscripts;
use App\Models\History;
use App\Models\User;
use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\Conversational;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Messages\Message;
use Laravel\Ai\Promptable;
use Stringable;

class SalesCoach implements Agent, Conversational, HasTools, HasStructuredOutput
{
    use Promptable;

    public function __construct(public User $user) {}

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): Stringable|string
    {
        return 'You are a sales coach, analyzing transcripts and providing feedback and an overall sales strength score.';
    }

    /**
     * Get the list of messages comprising the conversation so far.
     */
    public function messages(): iterable
    {
        return History::where('user_id', $this->user->id)
            ->latest()
            ->limit(50)
            ->get()
            ->reverse()
            ->map(function ($message) {
                return new Message($message->role, $message->content);
            })->all();
    }

    /**
     * Get the tools available to the agent.
     *
     * @return Tool[]
     */
    public function tools(): iterable
    {
        return [
            new RetrievePreviousTranscripts,
        ];
    }

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'feedback' => $schema->string()->required(),
            'score' => $schema->integer()->min(1)->max(10)->required(),
        ];
    }
}
```


<a name="prompting"></a>
### 提示 (Prompting)

要提示 AI 代理，請先使用 `make` 方法或標準實例化方式建立實例，然後呼叫 `prompt`：

```php
$response = (new SalesCoach)
    ->prompt('Analyze this sales transcript...');

return (string) $response;
```

`make` 方法會從容器中解析您的 AI 代理，這允許自動依賴注入。您也可以將引數傳遞給 AI 代理的建構子：

```php
$agent = SalesCoach::make(user: $user);
```

透過將額外引數傳遞給 `prompt` 方法，您可以在提示時覆寫預設的供應商、模型或 HTTP 逾時時間：

```php
$response = (new SalesCoach)->prompt(
    'Analyze this sales transcript...',
    provider: Lab::Anthropic,
    model: 'claude-haiku-4-5-20251001',
    timeout: 120,
);
```


<a name="conversation-context"></a>
### 對話上下文

如果您的 AI 代理實作了 `Conversational` 介面，您可以使用 `messages` 方法回傳先前的對話上下文（若適用）：

```php
use App\Models\History;
use Laravel\Ai\Messages\Message;

/**
 * Get the list of messages comprising the conversation so far.
 */
public function messages(): iterable
{
    return History::where('user_id', $this->user->id)
        ->latest()
        ->limit(50)
        ->get()
        ->reverse()
        ->map(function ($message) {
            return new Message($message->role, $message->content);
        })->all();
}
```


<a name="remembering-conversations"></a>
#### 記住對話

> **注意：** 在使用 `RemembersConversations` trait 之前，您應該使用 `vendor:publish` Artisan 指令發布並執行 AI SDK 遷移。這些遷移將建立儲存對話所需的資料庫表。

如果您希望 Laravel 自動為您的 AI 代理儲存與檢索對話歷史記錄，您可以使用 `RemembersConversations` trait。此 trait 提供了一種簡單的方法將對話訊息持久化到資料庫，而無需手動實作 `Conversational` 介面：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Concerns\RemembersConversations;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\Conversational;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, Conversational
{
    use Promptable, RemembersConversations;

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): string
    {
        return 'You are a sales coach...';
    }
}
```

要為使用者開始新對話，請在提示之前呼叫 `forUser` 方法：

```php
$response = (new SalesCoach)->forUser($user)->prompt('Hello!');

$conversationId = $response->conversationId;
```

對話 ID 會在回應中回傳，並可儲存供日後參考，或者您可以直接從 `agent_conversations` 資料表中檢索使用者的所有對話。

要繼續現有對話，請使用 `continue` 方法：

```php
$response = (new SalesCoach)
    ->continue($conversationId, as: $user)
    ->prompt('Tell me more about that.');
```

使用 `RemembersConversations` trait 時，先前的訊息會在提示時自動載入並包含在對話上下文中。新的訊息（包括使用者和助手的訊息）會在每次互動後自動儲存。

<a name="structured-output"></a>
### 結構化輸出

如果您希望您的 Agent 返回結構化輸出，請實作 `HasStructuredOutput` 介面，這要求您的 Agent 必須定義一個 `schema` 方法：

```php
<?php

namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasStructuredOutput
{
    use Promptable;

    // ...

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'score' => $schema->integer()->required(),
        ];
    }
}
```

當提示 (Prompting) 一個會回傳結構化輸出的 Agent 時，您可以像操作陣列一樣存取回傳的 `StructuredAgentResponse`：

```php
$response = (new SalesCoach)->prompt('Analyze this sales transcript...');

return $response['score'];
```


<a name="structured-output-nested-objects"></a>
#### 巢狀物件

要定義巢狀的結構化輸出，請使用 `object` 方法並搭配一個閉包：

```php
<?php

namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasStructuredOutput
{
    use Promptable;

    // ...

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'score' => $schema->integer()->required(),
            'metadata' => $schema->object(fn ($schema) => [
                'confidence' => $schema->string()->enum(['low', 'medium', 'high'])->required(),
                'language' => $schema->string()->required(),
            ])->required(),
        ];
    }
}
```


<a name="structured-output-arrays-of-objects"></a>
#### 物件陣列

如果您的 Agent 應該回傳一個結構化項目的列表，請結合使用 `array` 與 `object` 方法：

```php
public function schema(JsonSchema $schema): array
{
    return [
        'feedback' => $schema->array()
            ->items(
                $schema->object(fn ($schema) => [
                    'comment' => $schema->string()->required(),
                    'score' => $schema->integer()->required(),
                ])
            )
            ->required(),
    ];
}
```


<a name="attachments"></a>
### 附件

在提示時，您也可以隨提示詞傳遞附件，讓模型能夠檢查圖片與文件：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Files;

$response = (new SalesCoach)->prompt(
    'Analyze the attached sales transcript...',
    attachments: [
        Files\Document::fromStorage('transcript.pdf') // Attach a document from a filesystem disk...
        Files\Document::fromPath('/home/laravel/transcript.md') // Attach a document from a local path...
        $request->file('transcript'), // Attach an uploaded file...
    ]
);
```

同樣地，`Laravel\Ai\Files\Image` 類別可用於將圖片附加到提示詞中：

```php
use App\Ai\Agents\ImageAnalyzer;
use Laravel\Ai\Files;

$response = (new ImageAnalyzer)->prompt(
    'What is in this image?',
    attachments: [
        Files\Image::fromStorage('photo.jpg') // Attach an image from a filesystem disk...
        Files\Image::fromPath('/home/laravel/photo.jpg') // Attach an image from a local path...
        $request->file('photo'), // Attach an uploaded file...
    ]
);
```


<a name="streaming"></a>
### 串流

您可以透過呼叫 `stream` 方法來串流 Agent 的回應。回傳的 `StreamableAgentResponse` 可以直接從路由回傳，藉此自動向客戶端發送伺服器傳送事件 (SSE) 串流回應：

```php
use App\Ai\Agents\SalesCoach;

Route::get('/coach', function () {
    return (new SalesCoach)->stream('Analyze this sales transcript...');
});
```

`then` 方法可用於提供一個閉包，該閉包將在整個回應完整串流至客戶端後被執行：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Responses\StreamedAgentResponse;

Route::get('/coach', function () {
    return (new SalesCoach)
        ->stream('Analyze this sales transcript...')
        ->then(function (StreamedAgentResponse $response) {
            // $response->text, $response->events, $response->usage...
        });
});
```

或者，您也可以手動疊代處理串流事件：

```php
$stream = (new SalesCoach)->stream('Analyze this sales transcript...');

foreach ($stream as $event) {
    // ...
}
```


<a name="streaming-using-the-vercel-ai-sdk-protocol"></a>
#### 使用 Vercel AI SDK 協議進行串流

您可以透過在串流回應上呼叫 `usingVercelDataProtocol` 方法，來使用 [Vercel AI SDK 串流協議](https://ai-sdk.dev/docs/ai-sdk-ui/stream-protocol) 進行事件串流：

```php
use App\Ai\Agents\SalesCoach;

Route::get('/coach', function () {
    return (new SalesCoach)
        ->stream('Analyze this sales transcript...')
        ->usingVercelDataProtocol();
});
```


<a name="broadcasting"></a>
### 廣播

您可以透過幾種不同的方式廣播串流事件。首先，您可以直接在串流事件上呼叫 `broadcast` 或 `broadcastNow` 方法：

```php
use App\Ai\Agents\SalesCoach;
use Illuminate\Broadcasting\Channel;

$stream = (new SalesCoach)->stream('Analyze this sales transcript...');

foreach ($stream as $event) {
    $event->broadcast(new Channel('channel-name'));
}
```

或者，您可以呼叫 Agent 的 `broadcastOnQueue` 方法，將 Agent 操作排入佇列，並在串流事件可用時自動廣播：

```php
(new SalesCoach)->broadcastOnQueue(
    'Analyze this sales transcript...'
    new Channel('channel-name'),
);
```


<a name="queueing"></a>
### 佇列化

透過 Agent 的 `queue` 方法，您可以對 Agent 發出提示，但允許其在背景處理回應，讓您的應用程式保持流暢且即時的回應感。`then` 與 `catch` 方法可用於註冊閉包，這些閉包將在回應完成或發生異常時被呼叫：

```php
use Illuminate\Http\Request;
use Laravel\Ai\Responses\AgentResponse;
use Throwable;

Route::post('/coach', function (Request $request) {
    (new SalesCoach)
        ->queue($request->input('transcript'))
        ->then(function (AgentResponse $response) {
            // ...
        })
        ->catch(function (Throwable $e) {
            // ...
        });

    return back();
});
```

<a name="tools"></a>
### 工具

工具可以用來賦予 AI 代理額外的功能，讓它們在回應提示詞時可以使用。可以使用 `make:tool` Artisan 指令來建立工具：

```shell
php artisan make:tool RandomNumberGenerator
```

產生的工具將放置在應用程式的 `app/Ai/Tools` 目錄中。每個工具都包含一個 `handle` 方法，當 AI 代理需要使用該工具時會呼叫此方法：

```php
<?php

namespace App\Ai\Tools;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Tool;
use Laravel\Ai\Tools\Request;
use Stringable;

class RandomNumberGenerator implements Tool
{
    /**
     * Get the description of the tool's purpose.
     */
    public function description(): Stringable|string
    {
        return 'This tool may be used to generate cryptographically secure random numbers.';
    }

    /**
     * Execute the tool.
     */
    public function handle(Request $request): Stringable|string
    {
        return (string) random_int($request['min'], $request['max']);
    }

    /**
     * Get the tool's schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'min' => $schema->integer()->min(0)->required(),
            'max' => $schema->integer()->required(),
        ];
    }
}
```

定義好工具後，您可以在任何 AI 代理的 `tools` 方法中回傳它：

```php
use App\Ai\Tools\RandomNumberGenerator;

/**
 * Get the tools available to the agent.
 *
 * @return Tool[]
 */
public function tools(): iterable
{
    return [
        new RandomNumberGenerator,
    ];
}
```

<a name="similarity-search"></a>
#### 相似性搜尋

`SimilaritySearch` 工具允許 AI 代理使用儲存在資料庫中的向量 Embeddings 來搜尋與給定查詢相似的文件。這對於檢索增強生成 (RAG) 非常有用，當您想讓 AI 代理存取並搜尋應用程式的資料時可以使用。

建立相似性搜尋工具最簡單的方法是使用 `usingModel` 方法，並傳入具有向量 Embeddings 的 Eloquent 模型：

```php
use App\Models\Document;
use Laravel\Ai\Tools\SimilaritySearch;

public function tools(): iterable
{
    return [
        SimilaritySearch::usingModel(Document::class, 'embedding'),
    ];
}
```

第一個引數是 Eloquent 模型類別，第二個引數是包含向量 Embeddings 的欄位。

您也可以提供介於 `0.0` 到 `1.0` 之間的最小相似度閾值，以及一個用來自定義查詢的閉包：

```php
SimilaritySearch::usingModel(
    model: Document::class,
    column: 'embedding',
    minSimilarity: 0.7,
    limit: 10,
    query: fn ($query) => $query->where('published', true),
),
```

若要獲得更多控制權，您可以使用自定義閉包來建立相似性搜尋工具，並回傳搜尋結果：

```php
use App\Models\Document;
use Laravel\Ai\Tools\SimilaritySearch;

public function tools(): iterable
{
    return [
        new SimilaritySearch(using: function (string $query) {
            return Document::query()
                ->where('user_id', $this->user->id)
                ->whereVectorSimilarTo('embedding', $query)
                ->limit(10)
                ->get();
        }),
    ];
}
```

您可以使用 `withDescription` 方法來自定義工具的描述：

```php
SimilaritySearch::usingModel(Document::class, 'embedding')
    ->withDescription('Search the knowledge base for relevant articles.'),
```

<a name="provider-tools"></a>
### 供應商工具

供應商工具是由 AI 供應商原生實作的特殊工具，提供如網頁搜尋、URL 擷取和檔案搜尋等功能。與一般工具不同，供應商工具是由供應商本身執行，而非您的應用程式。

供應商工具可以由您的 AI 代理的 `tools` 方法回傳。

<a name="web-search"></a>
#### 網頁搜尋

`WebSearch` 供應商工具允許 AI 代理在網路上搜尋即時資訊。這對於回答有關時事、近期資料或在模型訓練截止日期後可能發生變動的主題非常有用。

**支援的供應商：** Anthropic, OpenAI, Gemini

```php
use Laravel\Ai\Providers\Tools\WebSearch;

public function tools(): iterable
{
    return [
        new WebSearch,
    ];
}
```

您可以設定網頁搜尋工具以限制搜尋次數，或將結果限制在特定的網域：

```php
(new WebSearch)->max(5)->allow(['laravel.com', 'php.net']),
```

若要根據使用者位置精確化搜尋結果，請使用 `location` 方法：

```php
(new WebSearch)->location(
    city: 'New York',
    region: 'NY',
    country: 'US'
);
```

<a name="web-fetch"></a>
#### 網頁擷取

`WebFetch` 供應商工具允許 AI 代理擷取並讀取網頁內容。當您需要 AI 代理分析特定的 URL 或從已知的網頁獲取詳細資訊時，這非常有用。

**支援的供應商：** Anthropic, Gemini

```php
use Laravel\Ai\Providers\Tools\WebFetch;

public function tools(): iterable
{
    return [
        new WebFetch,
    ];
}
```

您可以設定網頁擷取工具以限制擷取次數，或限制在特定的網域：

```php
(new WebFetch)->max(3)->allow(['docs.laravel.com']),
```

<a name="file-search"></a>
#### 檔案搜尋

`FileSearch` 供應商工具允許 AI 代理搜尋儲存在[向量存儲 (Vector Stores)](#vector-stores)中的[檔案](#files)。這透過允許 AI 代理在您上傳的文件中搜尋相關資訊，來實現檢索增強生成 (RAG)。

**支援的供應商：** OpenAI, Gemini

```php
use Laravel\Ai\Providers\Tools\FileSearch;

public function tools(): iterable
{
    return [
        new FileSearch(stores: ['store_id']),
    ];
}
```

您可以提供多個向量存儲 ID 以在多個存儲中進行搜尋：

```php
new FileSearch(stores: ['store_1', 'store_2']);
```

如果您的檔案具有[元資料 (Metadata)](#adding-files-to-stores)，您可以使用 `where` 引數來過濾搜尋結果。對於簡單的等值過濾，請傳入一個陣列：

```php
new FileSearch(stores: ['store_id'], where: [
    'author' => 'Taylor Otwell',
    'year' => 2026,
]);
```

對於更複雜的過濾，您可以傳入一個接收 `FileSearchQuery` 實例的閉包：

```php
use Laravel\Ai\Providers\Tools\FileSearchQuery;

new FileSearch(stores: ['store_id'], where: fn (FileSearchQuery $query) =>
    $query->where('author', 'Taylor Otwell')
        ->whereNot('status', 'draft')
        ->whereIn('category', ['news', 'updates'])
);
```

<a name="middleware"></a>
### 中介層

AI 代理支援中介層，讓您可以在提示詞發送給供應商之前攔截並修改它們。可以使用 `make:agent-middleware` Artisan 指令建立中介層：

```shell
php artisan make:agent-middleware LogPrompts
```

生成的中介層將位於應用程式的 `app/Ai/Middleware` 目錄中。要為 AI 代理新增中介層，請實作 `HasMiddleware` 介面並定義一個傳回中介層類別陣列的 `middleware` 方法：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Middleware\LogPrompts;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasMiddleware;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasMiddleware
{
    use Promptable;

    // ...

    /**
     * Get the agent's middleware.
     */
    public function middleware(): array
    {
        return [
            new LogPrompts,
        ];
    }
}
```

每個中介層類別都應定義一個 `handle` 方法，該方法接收 `AgentPrompt` 和一個用於將提示詞傳遞給下一個中介層的 `Closure`：

```php
<?php

namespace App\Ai\Middleware;

use Closure;
use Laravel\Ai\Prompts\AgentPrompt;

class LogPrompts
{
    /**
     * Handle the incoming prompt.
     */
    public function handle(AgentPrompt $prompt, Closure $next)
    {
        Log::info('Prompting agent', ['prompt' => $prompt->prompt]);

        return $next($prompt);
    }
}
```

您可以在回應上使用 `then` 方法，以便在 AI 代理處理完成後執行程式碼。這適用於同步和串流回應：

```php
public function handle(AgentPrompt $prompt, Closure $next)
{
    return $next($prompt)->then(function (AgentResponse $response) {
        Log::info('Agent responded', ['text' => $response->text]);
    });
}
```

<a name="anonymous-agents"></a>
### 匿名代理

有時您可能想在不建立專屬 AI 代理類別的情況下快速與模型互動。您可以使用 `agent` 函式建立一個即時的匿名代理：

```php
use function Laravel\Ai\{agent};

$response = agent(
    instructions: 'You are an expert at software development.',
    messages: [],
    tools: [],
)->prompt('Tell me about Laravel')
```

匿名代理也可以產生結構化輸出：

```php
use Illuminate\Contracts\JsonSchema\JsonSchema;

use function Laravel\Ai\{agent};

$response = agent(
    schema: fn (JsonSchema $schema) => [
        'number' => $schema->integer()->required(),
    ],
)->prompt('Generate a random number less than 100')
```

<a name="agent-configuration"></a>
### 代理設定

您可以使用 PHP 屬性 (Attributes) 為 AI 代理設定文字生成選項。以下是可用的屬性：

- `MaxSteps`：AI 代理在使用工具時可以執行的最大步驟數。
- `MaxTokens`：模型可以生成的最大 token 數量。
- `Model`：AI 代理應使用的模型。
- `Provider`：AI 代理使用的 AI 供應商（或用於容錯移轉的多個供應商）。
- `Temperature`：用於生成的採樣溫度（0.0 到 1.0）。
- `Timeout`：AI 代理請求的 HTTP 逾時秒數（預設為 60）。
- `UseCheapestModel`：使用供應商最便宜的文字模型以進行成本優化。
- `UseSmartestModel`：使用供應商最強大的文字模型以處理複雜任務。

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Attributes\MaxSteps;
use Laravel\Ai\Attributes\MaxTokens;
use Laravel\Ai\Attributes\Model;
use Laravel\Ai\Attributes\Provider;
use Laravel\Ai\Attributes\Temperature;
use Laravel\Ai\Attributes\Timeout;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

#[Provider(Lab::Anthropic)]
#[Model('claude-haiku-4-5-20251001')]
#[MaxSteps(10)]
#[MaxTokens(4096)]
#[Temperature(0.7)]
#[Timeout(120)]
class SalesCoach implements Agent
{
    use Promptable;

    // ...
}
```

`UseCheapestModel` 和 `UseSmartestModel` 屬性允許您自動為特定供應商選擇最具成本效益或最強大的模型，而無需指定模型名稱。這在您想針對不同供應商最佳化成本或效能時非常有用：

```php
use Laravel\Ai\Attributes\UseCheapestModel;
use Laravel\Ai\Attributes\UseSmartestModel;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Promptable;

#[UseCheapestModel]
class SimpleSummarizer implements Agent
{
    use Promptable;

    // Will use the cheapest model (e.g., Haiku)...
}

#[UseSmartestModel]
class ComplexReasoner implements Agent
{
    use Promptable;

    // Will use the most capable model (e.g., Opus)...
}
```

<a name="provider-options"></a>
### 供應商選項

如果您的 AI 代理需要傳遞供應商專屬選項（例如 OpenAI 的推理效能或懲罰設定），請實作 `HasProviderOptions` 契約 (Contract) 並定義一個 `providerOptions` 方法：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasProviderOptions;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasProviderOptions
{
    use Promptable;

    // ...

    /**
     * Get provider-specific generation options.
     */
    public function providerOptions(Lab|string $provider): array
    {
        return match ($provider) {
            Lab::OpenAI => [
                'reasoning' => ['effort' => 'low'],
                'frequency_penalty' => 0.5,
                'presence_penalty' => 0.3,
            ],
            Lab::Anthropic => [
                'thinking' => ['budget_tokens' => 1024],
            ],
            default => [],
        };
    }
}
```

`providerOptions` 方法接收目前正在使用的供應商（`Lab` 列舉或字串），讓您可以為每個供應商傳回不同的選項。這在搭配[容錯移轉 (Failover)](#failover)使用時特別有用，因為每個備援供應商都可以接收自己的設定。

<a name="images"></a>
## 圖片

`Laravel\Ai\Image` 類別可用於透過 `openai`、`gemini` 或 `xai` 供應商來產生圖片：

```php
use Laravel\Ai\Image;

$image = Image::of('A donut sitting on the kitchen counter')->generate();

$rawContent = (string) $image;
```

`square`、`portrait` 與 `landscape` 方法可用於控制圖片的長寬比，而 `quality` 方法可用於引導模型決定最終圖片的品質 (`high`、`medium`、`low`)。`timeout` 方法可用於指定 HTTP 超時時間（以秒為單位）：

```php
use Laravel\Ai\Image;

$image = Image::of('A donut sitting on the kitchen counter')
    ->quality('high')
    ->landscape()
    ->timeout(120)
    ->generate();
```

您可以使用 `attachments` 方法附加參考圖片：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Image;

$image = Image::of('Update this photo of me to be in the style of an impressionist painting.')
    ->attachments([
        Files\Image::fromStorage('photo.jpg'),
        // Files\Image::fromPath('/home/laravel/photo.jpg'),
        // Files\Image::fromUrl('https://example.com/photo.jpg'),
        // $request->file('photo'),
    ])
    ->landscape()
    ->generate();
```

產生的圖片可以輕鬆地儲存在應用程式 `config/filesystems.php` 設定檔中定義的預設磁碟：

```php
$image = Image::of('A donut sitting on the kitchen counter');

$path = $image->store();
$path = $image->storeAs('image.jpg');
$path = $image->storePublicly();
$path = $image->storePubliclyAs('image.jpg');
```

圖片產生也可以被佇列化：

```php
use Laravel\Ai\Image;
use Laravel\Ai\Responses\ImageResponse;

Image::of('A donut sitting on the kitchen counter')
    ->portrait()
    ->queue()
    ->then(function (ImageResponse $image) {
        $path = $image->store();

        // ...
    });
```


<a name="audio"></a>
## 音訊 (TTS)

`Laravel\Ai\Audio` 類別可用於根據給定的文字產生音訊：

```php
use Laravel\Ai\Audio;

$audio = Audio::of('I love coding with Laravel.')->generate();

$rawContent = (string) $audio;
```

`male`、`female` 與 `voice` 方法可用於決定產生音訊的人聲：

```php
$audio = Audio::of('I love coding with Laravel.')
    ->female()
    ->generate();

$audio = Audio::of('I love coding with Laravel.')
    ->voice('voice-id-or-name')
    ->generate();
```

同樣地，`instructions` 方法可用於動態地指引模型產生的音訊聽起來應該如何：

```php
$audio = Audio::of('I love coding with Laravel.')
    ->female()
    ->instructions('Said like a pirate')
    ->generate();
```

產生的音訊可以輕鬆地儲存在應用程式 `config/filesystems.php` 設定檔中定義的預設磁碟：

```php
$audio = Audio::of('I love coding with Laravel.')->generate();

$path = $audio->store();
$path = $audio->storeAs('audio.mp3');
$path = $audio->storePublicly();
$path = $audio->storePubliclyAs('audio.mp3');
```

音訊產生也可以被佇列化：

```php
use Laravel\Ai\Audio;
use Laravel\Ai\Responses\AudioResponse;

Audio::of('I love coding with Laravel.')
    ->queue()
    ->then(function (AudioResponse $audio) {
        $path = $audio->store();

        // ...
    });
```


<a name="transcription"></a>
## 逐字稿 (STT)

`Laravel\Ai\Transcription` 類別可用於為給定的音訊產生逐字稿：

```php
use Laravel\Ai\Transcription;

$transcript = Transcription::fromPath('/home/laravel/audio.mp3')->generate();
$transcript = Transcription::fromStorage('audio.mp3')->generate();
$transcript = Transcription::fromUpload($request->file('audio'))->generate();

return (string) $transcript;
```

`diarize` 方法可用於表示除了原始文字逐字稿外，您還希望回應包含講者識別 (Diarization) 的逐字稿，這讓您可以存取按講者區分的段落逐字稿：

```php
$transcript = Transcription::fromStorage('audio.mp3')
    ->diarize()
    ->generate();
```

逐字稿產生也可以被佇列化：

```php
use Laravel\Ai\Transcription;
use Laravel\Ai\Responses\TranscriptionResponse;

Transcription::fromStorage('audio.mp3')
    ->queue()
    ->then(function (TranscriptionResponse $transcript) {
        // ...
    });
```

<a name="embeddings"></a>
## Embeddings

您可以使用 Laravel `Stringable` 類別中提供的 `toEmbeddings` 方法，輕鬆地為任何給定的字串產生向量 Embeddings：

```php
use Illuminate\Support\Str;

$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings();
```

或者，您可以使用 `Embeddings` 類別一次為多個輸入產生 Embeddings：

```php
use Laravel\Ai\Embeddings;

$response = Embeddings::for([
    'Napa Valley has great wine.',
    'Laravel is a PHP framework.',
])->generate();

$response->embeddings; // [[0.123, 0.456, ...], [0.789, 0.012, ...]]
```

您可以為 Embeddings 指定維度與供應商：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->dimensions(1536)
    ->generate(Lab::OpenAI, 'text-embedding-3-small');
```


<a name="querying-embeddings"></a>
### 查詢 Embeddings

產生 Embeddings 後，您通常會將它們儲存在資料庫的 `vector` 欄位中以便後續查詢。Laravel 透過 `pgvector` 擴充套件提供 PostgreSQL 向量欄位的原生支援。要開始使用，請在您的遷移中定義一個 `vector` 欄位，並指定維度數量：

```php
Schema::ensureVectorExtensionExists();

Schema::create('documents', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->vector('embedding', dimensions: 1536);
    $table->timestamps();
});
```

您也可以加入向量索引以加速相似度搜尋。當對向量欄位呼叫 `index` 時，Laravel 會自動建立一個使用餘弦距離 (cosine distance) 的 HNSW 索引：

```php
$table->vector('embedding', dimensions: 1536)->index();
```

在您的 Eloquent 模型上，您應該將向量欄位轉型為 `array`：

```php
protected function casts(): array
{
    return [
        'embedding' => 'array',
    ];
}
```

若要查詢相似的紀錄，請使用 `whereVectorSimilarTo` 方法。此方法會根據最小餘弦相似度（介於 `0.0` 與 `1.0` 之間，其中 `1.0` 代表完全相同）來過濾結果，並依相似度排序結果：

```php
use App\Models\Document;

$documents = Document::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

`$queryEmbedding` 可以是浮點數陣列或純字串。當提供字串時，Laravel 會自動為其產生 Embeddings：

```php
$documents = Document::query()
    ->whereVectorSimilarTo('embedding', 'best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

如果您需要更多控制，可以獨立使用較低階的 `whereVectorDistanceLessThan`、`selectVectorDistance` 以及 `orderByVectorDistance` 方法：

```php
$documents = Document::query()
    ->select('*')
    ->selectVectorDistance('embedding', $queryEmbedding, as: 'distance')
    ->whereVectorDistanceLessThan('embedding', $queryEmbedding, maxDistance: 0.3)
    ->orderByVectorDistance('embedding', $queryEmbedding)
    ->limit(10)
    ->get();
```

如果您想讓 AI 代理有能力執行相似度搜尋作為一種工具，請參閱 [相似度搜尋 (Similarity Search)](#similarity-search) 工具文件。

> [!NOTE]
> 向量查詢目前僅支援使用 `pgvector` 擴充套件的 PostgreSQL 連線。


<a name="caching-embeddings"></a>
### 快取 Embeddings

產生 Embeddings 的過程可以被快取，以避免針對相同輸入進行重複的 API 呼叫。要啟用快取，請將 `ai.caching.embeddings.cache` 設定選項設為 `true`：

```php
'caching' => [
    'embeddings' => [
        'cache' => true,
        'store' => env('CACHE_STORE', 'database'),
        // ...
    ],
],
```

啟用快取時，Embeddings 會被快取 30 天。快取鍵 (Cache key) 是基於供應商、模型、維度及輸入內容產生的，確保相同的請求會回傳快取的結果，而不同的設定則會產生全新的 Embeddings。

即使全域快取被禁用，您也可以使用 `cache` 方法為特定請求啟用快取：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->cache()
    ->generate();
```

您可以指定自定義的快取時長（以秒為單位）：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->cache(seconds: 3600) // Cache for 1 hour
    ->generate();
```

`toEmbeddings` Stringable 方法也接受 `cache` 引數：

```php
// Cache with default duration...
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: true);

// Cache for a specific duration...
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: 3600);
```


<a name="reranking"></a>
## 重新排序 (Reranking)

重新排序允許您根據文件與給定查詢的相關性來重新排列文件列表。這對於透過語義理解來改善搜尋結果非常有用。

可以使用 `Laravel\Ai\Reranking` 類別來重新排序文件：

```php
use Laravel\Ai\Reranking;

$response = Reranking::of([
    'Django is a Python web framework.',
    'Laravel is a PHP web application framework.',
    'React is a JavaScript library for building user interfaces.',
])->rerank('PHP frameworks');

// Access the top result...
$response->first()->document; // "Laravel is a PHP web application framework."
$response->first()->score;    // 0.95
$response->first()->index;    // 1 (original position)
```

可以使用 `limit` 方法來限制回傳的結果數量：

```php
$response = Reranking::of($documents)
    ->limit(5)
    ->rerank('search query');
```


<a name="reranking-collections"></a>
### 重新排序集合 (Reranking Collections)

為了方便起見，可以使用 `rerank` 巨集來重新排序 Laravel 集合。第一個引數指定用於重新排序的欄位，第二個引數則是查詢內容：

```php
// Rerank by a single field...
$posts = Post::all()
    ->rerank('body', 'Laravel tutorials');

// Rerank by multiple fields (sent as JSON)...
$reranked = $posts->rerank(['title', 'body'], 'Laravel tutorials');

// Rerank using a closure to build the document...
$reranked = $posts->rerank(
    fn ($post) => $post->title.': '.$post->body,
    'Laravel tutorials'
);
```

您也可以限制結果數量並指定供應商：

```php
$reranked = $posts->rerank(
    by: 'content',
    query: 'Laravel tutorials',
    limit: 10,
    provider: Lab::Cohere
);
```

<a name="files"></a>
## 檔案

`Laravel\Ai\Files` 類別或個別的檔案類別可用於將檔案存儲在您的 AI 供應商，以便稍後在對話中使用。這對於大型文件或您想要多次引用而不想重複上傳的檔案非常有用：

```php
use Laravel\Ai\Files\Document;
use Laravel\Ai\Files\Image;

// Store a file from a local path...
$response = Document::fromPath('/home/laravel/document.pdf')->put();
$response = Image::fromPath('/home/laravel/photo.jpg')->put();

// Store a file that is stored on a filesystem disk...
$response = Document::fromStorage('document.pdf', disk: 'local')->put();
$response = Image::fromStorage('photo.jpg', disk: 'local')->put();

// Store a file that is stored on a remote URL...
$response = Document::fromUrl('https://example.com/document.pdf')->put();
$response = Image::fromUrl('https://example.com/photo.jpg')->put();

return $response->id;
```

您也可以存儲原始內容或上傳的檔案：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Files\Document;

// Store raw content...
$stored = Document::fromString('Hello, World!', 'text/plain')->put();

// Store an uploaded file...
$stored = Document::fromUpload($request->file('document'))->put();
```

一旦檔案被存儲，您就可以在透過 AI 代理生成文字時引用該檔案，而不需要重新上傳：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Files;

$response = (new SalesCoach)->prompt(
    'Analyze the attached sales transcript...'
    attachments: [
        Files\Document::fromId('file-id') // Attach a stored document...
    ]
);
```

若要擷取先前存儲的檔案，請在檔案實例上使用 `get` 方法：

```php
use Laravel\Ai\Files\Document;

$file = Document::fromId('file-id')->get();

$file->id;
$file->mimeType();
```

若要從供應商端刪除檔案，請使用 `delete` 方法：

```php
Document::fromId('file-id')->delete();
```

預設情況下，`Files` 類別會使用您應用程式 `config/ai.php` 設定檔中定義的預設 AI 供應商。對於大多數操作，您可以使用 `provider` 引數指定不同的供應商：

```php
$response = Document::fromPath(
    '/home/laravel/document.pdf'
)->put(provider: Lab::Anthropic);
```

<a name="using-stored-files-in-conversations"></a>
### 在對話中使用已存儲的檔案

一旦檔案已存儲在供應商端，您就可以在 AI 代理對話中透過 `Document` 或 `Image` 類別的 `fromId` 方法來引用它：

```php
use App\Ai\Agents\DocumentAnalyzer;
use Laravel\Ai\Files;
use Laravel\Ai\Files\Document;

$stored = Document::fromPath('/path/to/report.pdf')->put();

$response = (new DocumentAnalyzer)->prompt(
    'Summarize this document.',
    attachments: [
        Document::fromId($stored->id),
    ],
);
```

同樣地，已存儲的圖片也可以使用 `Image` 類別來引用：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Files\Image;

$stored = Image::fromPath('/path/to/photo.jpg')->put();

$response = (new ImageAnalyzer)->prompt(
    'What is in this image?',
    attachments: [
        Image::fromId($stored->id),
    ],
);
```

<a name="vector-stores"></a>
## 向量存儲 (Vector Stores)

向量存儲允許您建立可搜尋的檔案集合，這些集合可用於檢索增強生成 (RAG)。`Laravel\Ai\Stores` 類別提供了建立、擷取與刪除向量存儲的方法：

```php
use Laravel\Ai\Stores;

// Create a new vector store...
$store = Stores::create('Knowledge Base');

// Create a store with additional options...
$store = Stores::create(
    name: 'Knowledge Base',
    description: 'Documentation and reference materials.',
    expiresWhenIdleFor: days(30),
);

return $store->id;
```

若要透過 ID 擷取現有的向量存儲，請使用 `get` 方法：

```php
use Laravel\Ai\Stores;

$store = Stores::get('store_id');

$store->id;
$store->name;
$store->fileCounts;
$store->ready;
```

若要刪除向量存儲，請在 `Stores` 類別或該存儲實例上使用 `delete` 方法：

```php
use Laravel\Ai\Stores;

// Delete by ID...
Stores::delete('store_id');

// Or delete via a store instance...
$store = Stores::get('store_id');

$store->delete();
```

<a name="adding-files-to-stores"></a>
### 將檔案新增至存儲

擁有向量存儲後，您可以使用 `add` 方法將[檔案](#files)新增至其中。新增至存儲的檔案會自動進行索引，以便使用[供應商檔案搜尋工具](#file-search)進行語意搜尋：

```php
use Laravel\Ai\Files\Document;
use Laravel\Ai\Stores;

$store = Stores::get('store_id');

// Add a file that has already been stored with the provider...
$document = $store->add('file_id');
$document = $store->add(Document::fromId('file_id'));

// Or, store and add a file in one step...
$document = $store->add(Document::fromPath('/path/to/document.pdf'));
$document = $store->add(Document::fromStorage('manual.pdf'));
$document = $store->add($request->file('document'));

$document->id;
$document->fileId;
```

> **注意：** 通常將先前存儲的檔案新增至向量存儲時，回傳的文件 ID 會與該檔案先前分配的 ID 相同；然而，某些向量存儲供應商可能會回傳一個全新的、不同的「文件 ID」。因此，建議您在資料庫中同時儲存這兩個 ID，以備日後參考。

在將檔案新增至存儲時，您可以附加元數據 (Metadata)。這些元數據稍後可用於在使用[供應商檔案搜尋工具](#file-search)時過濾搜尋結果：

```php
$store->add(Document::fromPath('/path/to/document.pdf'), metadata: [
    'author' => 'Taylor Otwell',
    'department' => 'Engineering',
    'year' => 2026,
]);
```

若要從存儲中移除檔案，請使用 `remove` 方法：

```php
$store->remove('file_id');
```

從向量存儲中移除檔案並不會從供應商的[檔案存儲](#files)中移除它。若要從向量存儲中移除檔案並永久從檔案存儲中刪除它，請使用 `deleteFile` 引數：

```php
$store->remove('file_abc123', deleteFile: true);
```

<a name="failover"></a>
## 容錯移轉 (Failover)

在提示或生成其他媒體時，您可以提供供應商 / 模型的陣列，以便在主要供應商遇到服務中斷或速率限制 (Rate limit) 時，自動容錯移轉至備用供應商 / 模型：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Image;

$response = (new SalesCoach)->prompt(
    'Analyze this sales transcript...',
    provider: [Lab::OpenAI, Lab::Anthropic],
);

$image = Image::of('A donut sitting on the kitchen counter')
    ->generate(provider: [Lab::Gemini, Lab::xAI]);
```

<a name="testing"></a>
## 測試

<a name="testing-agents"></a>
### AI 代理

若要在測試期間模擬 AI 代理的請求回應，請在 AI 代理類別上呼叫 `fake` 方法。您可以選擇提供回應陣列或閉包：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Prompts\AgentPrompt;

// Automatically generate a fixed response for every prompt...
SalesCoach::fake();

// Provide a list of prompt responses...
SalesCoach::fake([
    'First response',
    'Second response',
]);

// Dynamically handle prompt responses based on the incoming prompt...
SalesCoach::fake(function (AgentPrompt $prompt) {
    return 'Response for: '.$prompt->prompt;
});
```

> **注意：** 當在回傳結構化輸出的 AI 代理上叫用 `Agent::fake()` 時，Laravel 會自動產生符合您 AI 代理所定義輸出架構 (Schema) 的模擬資料。

提示 AI 代理後，您可以對收到的提示詞進行斷言：

```php
use Laravel\Ai\Prompts\AgentPrompt;

SalesCoach::assertPrompted('Analyze this...');

SalesCoach::assertPrompted(function (AgentPrompt $prompt) {
    return $prompt->contains('Analyze');
});

SalesCoach::assertNotPrompted('Missing prompt');

SalesCoach::assertNeverPrompted();
```

對於已加入佇列的 AI 代理叫用，請使用佇列斷言方法：

```php
use Laravel\Ai\QueuedAgentPrompt;

SalesCoach::assertQueued('Analyze this...');

SalesCoach::assertQueued(function (QueuedAgentPrompt $prompt) {
    return $prompt->contains('Analyze');
});

SalesCoach::assertNotQueued('Missing prompt');

SalesCoach::assertNeverQueued();
```

若要確保所有 AI 代理的叫用都有對應的模擬回應，您可以使用 `preventStrayPrompts`。如果在沒有定義模擬回應的情況下叫用了 AI 代理，系統將會拋出異常：

```php
SalesCoach::fake()->preventStrayPrompts();
```

<a name="testing-images"></a>
### 圖片

圖片生成可以透過叫用 `Image` 類別上的 `fake` 方法來模擬。一旦模擬了圖片，就可以對記錄下來的圖片生成提示詞執行各種斷言：

```php
use Laravel\Ai\Image;
use Laravel\Ai\Prompts\ImagePrompt;
use Laravel\Ai\Prompts\QueuedImagePrompt;

// Automatically generate a fixed response for every prompt...
Image::fake();

// Provide a list of prompt responses...
Image::fake([
    base64_encode($firstImage),
    base64_encode($secondImage),
]);

// Dynamically handle prompt responses based on the incoming prompt...
Image::fake(function (ImagePrompt $prompt) {
    return base64_encode('...');
});
```

生成圖片後，您可以對收到的提示詞進行斷言：

```php
Image::assertGenerated(function (ImagePrompt $prompt) {
    return $prompt->contains('sunset') && $prompt->isLandscape();
});

Image::assertNotGenerated('Missing prompt');

Image::assertNothingGenerated();
```

對於已加入佇列的圖片生成，請使用佇列斷言方法：

```php
Image::assertQueued(
    fn (QueuedImagePrompt $prompt) => $prompt->contains('sunset')
);

Image::assertNotQueued('Missing prompt');

Image::assertNothingQueued();
```

若要確保所有圖片生成都有對應的模擬回應，您可以使用 `preventStrayImages`。如果在沒有定義模擬回應的情況下生成圖片，系統將會拋出異常：

```php
Image::fake()->preventStrayImages();
```

<a name="testing-audio"></a>
### 音訊

音訊生成可以透過叫用 `Audio` 類別上的 `fake` 方法來模擬。一旦模擬了音訊，就可以對記錄下來的音訊生成提示詞執行各種斷言：

```php
use Laravel\Ai\Audio;
use Laravel\Ai\Prompts\AudioPrompt;
use Laravel\Ai\Prompts\QueuedAudioPrompt;

// Automatically generate a fixed response for every prompt...
Audio::fake();

// Provide a list of prompt responses...
Audio::fake([
    base64_encode($firstAudio),
    base64_encode($secondAudio),
]);

// Dynamically handle prompt responses based on the incoming prompt...
Audio::fake(function (AudioPrompt $prompt) {
    return base64_encode('...');
});
```

生成音訊後，您可以對收到的提示詞進行斷言：

```php
Audio::assertGenerated(function (AudioPrompt $prompt) {
    return $prompt->contains('Hello') && $prompt->isFemale();
});

Audio::assertNotGenerated('Missing prompt');

Audio::assertNothingGenerated();
```

對於已加入佇列的音訊生成，請使用佇列斷言方法：

```php
Audio::assertQueued(
    fn (QueuedAudioPrompt $prompt) => $prompt->contains('Hello')
);

Audio::assertNotQueued('Missing prompt');

Audio::assertNothingQueued();
```

若要確保所有音訊生成都有對應的模擬回應，您可以使用 `preventStrayAudio`。如果在沒有定義模擬回應的情況下生成音訊，系統將會拋出異常：

```php
Audio::fake()->preventStrayAudio();
```

<a name="testing-transcriptions"></a>
### 逐字稿

逐字稿生成可以透過叫用 `Transcription` 類別上的 `fake` 方法來模擬。一旦模擬了逐字稿，就可以對記錄下來的逐字稿生成提示詞執行各種斷言：

```php
use Laravel\Ai\Transcription;
use Laravel\Ai\Prompts\TranscriptionPrompt;
use Laravel\Ai\Prompts\QueuedTranscriptionPrompt;

// Automatically generate a fixed response for every prompt...
Transcription::fake();

// Provide a list of prompt responses...
Transcription::fake([
    'First transcription text.',
    'Second transcription text.',
]);

// Dynamically handle prompt responses based on the incoming prompt...
Transcription::fake(function (TranscriptionPrompt $prompt) {
    return 'Transcribed text...';
});
```

生成逐字稿後，您可以對收到的提示詞進行斷言：

```php
Transcription::assertGenerated(function (TranscriptionPrompt $prompt) {
    return $prompt->language === 'en' && $prompt->isDiarized();
});

Transcription::assertNotGenerated(
    fn (TranscriptionPrompt $prompt) => $prompt->language === 'fr'
);

Transcription::assertNothingGenerated();
```

對於已加入佇列的逐字稿生成，請使用佇列斷言方法：

```php
Transcription::assertQueued(
    fn (QueuedTranscriptionPrompt $prompt) => $prompt->isDiarized()
);

Transcription::assertNotQueued(
    fn (QueuedTranscriptionPrompt $prompt) => $prompt->language === 'fr'
);

Transcription::assertNothingQueued();
```

若要確保所有逐字稿生成都有對應的模擬回應，您可以使用 `preventStrayTranscriptions`。如果在沒有定義模擬回應的情況下生成逐字稿，系統將會拋出異常：

```php
Transcription::fake()->preventStrayTranscriptions();
```

<a name="testing-embeddings"></a>
### Embeddings

透過呼叫 `Embeddings` 類別上的 `fake` 方法，可以模擬 Embeddings 的生成。一旦模擬了 Embeddings，就可以針對記錄下來的 Embeddings 生成提示詞進行各種斷言：

```php
use Laravel\Ai\Embeddings;
use Laravel\Ai\Prompts\EmbeddingsPrompt;
use Laravel\Ai\Prompts\QueuedEmbeddingsPrompt;

// Automatically generate fake embeddings of the proper dimensions for every prompt...
Embeddings::fake();

// Provide a list of prompt responses...
Embeddings::fake([
    [$firstEmbeddingVector],
    [$secondEmbeddingVector],
]);

// Dynamically handle prompt responses based on the incoming prompt...
Embeddings::fake(function (EmbeddingsPrompt $prompt) {
    return array_map(
        fn () => Embeddings::fakeEmbedding($prompt->dimensions),
        $prompt->inputs
    );
});
```

在生成 Embeddings 之後，您可以對接收到的提示詞進行斷言：

```php
Embeddings::assertGenerated(function (EmbeddingsPrompt $prompt) {
    return $prompt->contains('Laravel') && $prompt->dimensions === 1536;
});

Embeddings::assertNotGenerated(
    fn (EmbeddingsPrompt $prompt) => $prompt->contains('Other')
);

Embeddings::assertNothingGenerated();
```

對於佇列化的 Embeddings 生成，請使用佇列斷言方法：

```php
Embeddings::assertQueued(
    fn (QueuedEmbeddingsPrompt $prompt) => $prompt->contains('Laravel')
);

Embeddings::assertNotQueued(
    fn (QueuedEmbeddingsPrompt $prompt) => $prompt->contains('Other')
);

Embeddings::assertNothingQueued();
```

為了確保所有的 Embeddings 生成都有對應的模擬回應，您可以使用 `preventStrayEmbeddings`。如果在沒有定義模擬回應的情況下生成了 Embeddings，系統將會拋出例外：

```php
Embeddings::fake()->preventStrayEmbeddings();
```


<a name="testing-reranking"></a>
### 重新排序 (Reranking)

透過呼叫 `Reranking` 類別上的 `fake` 方法，可以模擬重新排序 (Reranking) 操作：

```php
use Laravel\Ai\Reranking;
use Laravel\Ai\Prompts\RerankingPrompt;
use Laravel\Ai\Responses\Data\RankedDocument;

// Automatically generate a fake reranked responses...
Reranking::fake();

// Provide custom responses...
Reranking::fake([
    [
        new RankedDocument(index: 0, document: 'First', score: 0.95),
        new RankedDocument(index: 1, document: 'Second', score: 0.80),
    ],
]);
```

在重新排序之後，您可以對執行的操作進行斷言：

```php
Reranking::assertReranked(function (RerankingPrompt $prompt) {
    return $prompt->contains('Laravel') && $prompt->limit === 5;
});

Reranking::assertNotReranked(
    fn (RerankingPrompt $prompt) => $prompt->contains('Django')
);

Reranking::assertNothingReranked();
```


<a name="testing-files"></a>
### 檔案

透過呼叫 `Files` 類別上的 `fake` 方法，可以模擬檔案操作：

```php
use Laravel\Ai\Files;

Files::fake();
```

一旦模擬了檔案操作，您就可以對發生的上傳與刪除進行斷言：

```php
use Laravel\Ai\Contracts\Files\StorableFile;
use Laravel\Ai\Files\Document;

// Store files...
Document::fromString('Hello, Laravel!', mimeType: 'text/plain')
    ->as('hello.txt')
    ->put();

// Make assertions...
Files::assertStored(fn (StorableFile $file) =>
    (string) $file === 'Hello, Laravel!' &&
        $file->mimeType() === 'text/plain';
);

Files::assertNotStored(fn (StorableFile $file) =>
    (string) $file === 'Hello, World!'
);

Files::assertNothingStored();
```

若要針對檔案刪除進行斷言，您可以傳入檔案 ID：

```php
Files::assertDeleted('file-id');
Files::assertNotDeleted('file-id');
Files::assertNothingDeleted();
```


<a name="testing-vector-stores"></a>
### 向量存儲 (Vector Stores)

透過呼叫 `Stores` 類別上的 `fake` 方法，可以模擬向量存儲 (Vector Stores) 操作。模擬存儲也會自動模擬[檔案操作](#files)：

```php
use Laravel\Ai\Stores;

Stores::fake();
```

一旦模擬了存儲操作，您就可以對已建立或已刪除的存儲進行斷言：

```php
use Laravel\Ai\Stores;

// Create store...
$store = Stores::create('Knowledge Base');

// Make assertions...
Stores::assertCreated('Knowledge Base');

Stores::assertCreated(fn (string $name, ?string $description) =>
    $name === 'Knowledge Base'
);

Stores::assertNotCreated('Other Store');

Stores::assertNothingCreated();
```

若要針對存儲刪除進行斷言，您可以提供存儲 ID：

```php
Stores::assertDeleted('store_id');
Stores::assertNotDeleted('other_store_id');
Stores::assertNothingDeleted();
```

若要斷言檔案已新增至存儲或從存儲中移除，請在特定的 `Store` 實例上使用斷言方法：

```php
Stores::fake();

$store = Stores::get('store_id');

// Add / remove files...
$store->add('added_id');
$store->remove('removed_id');

// Make assertions...
$store->assertAdded('added_id');
$store->assertRemoved('removed_id');

$store->assertNotAdded('other_file_id');
$store->assertNotRemoved('other_file_id');
```

如果檔案在同一個請求中存儲到供應商的[檔案存儲](#files)並新增到向量存儲中，您可能不會知道該檔案的供應商 ID。在這種情況下，您可以傳遞一個閉包 (Closure) 給 `assertAdded` 方法，以針對新增檔案的內容進行斷言：

```php
use Laravel\Ai\Contracts\Files\StorableFile;
use Laravel\Ai\Files\Document;

$store->add(Document::fromString('Hello, World!', 'text/plain')->as('hello.txt'));

$store->assertAdded(fn (StorableFile $file) => $file->name() === 'hello.txt');
$store->assertAdded(fn (StorableFile $file) => $file->content() === 'Hello, World!');
```

<a name="events"></a>
## 事件

Laravel AI SDK 會發送多種[事件](/docs/{{version}}/events)，包含：

- `AddingFileToStore`
- `AgentPrompted`
- `AgentStreamed`
- `AudioGenerated`
- `CreatingStore`
- `EmbeddingsGenerated`
- `FileAddedToStore`
- `FileDeleted`
- `FileRemovedFromStore`
- `FileStored`
- `GeneratingAudio`
- `GeneratingEmbeddings`
- `GeneratingImage`
- `GeneratingTranscription`
- `ImageGenerated`
- `InvokingTool`
- `PromptingAgent`
- `RemovingFileFromStore`
- `Reranked`
- `Reranking`
- `StoreCreated`
- `StoringFile`
- `StreamingAgent`
- `ToolInvoked`
- `TranscriptionGenerated`

您可以監聽這些事件中的任何一個，以記錄或儲存 AI SDK 的使用資訊。
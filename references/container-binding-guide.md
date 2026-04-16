# Container Binding Guide — singleton vs scoped vs bind

Guia definitivo para escolha do tipo de binding no AppServiceProvider.
A inconsistência (usar `singleton` num momento e `scoped` noutro para a mesma support)
ocorre porque a decisão depende de características concretas da classe, não de preferência.

---

## Sumário

- Árvore de decisão
- Definições com exemplos
- Quando NÃO registrar
- Padrões comuns do projeto
- Anti-patterns

---

## Árvore de Decisão

Seguir esta árvore na ordem. Parar na primeira condição verdadeira:

```
1. A support é uma classe concreta sem interface e sem config dinâmica?
   └─ SIM → NÃO registrar. Laravel auto-resolve.
   └─ NÃO → continuar

2. A support implementa uma interface?
   └─ SIM → Registrar (interface → implementação). Continuar para tipo.
   └─ NÃO → continuar

3. A support precisa de config dinâmica (paths, valores de config())?
   └─ SIM → Registrar. Continuar para tipo.
   └─ NÃO → NÃO registrar. Laravel auto-resolve.

--- Escolha do tipo de binding ---

4. A support mantém estado mutável entre chamadas de método?
   └─ SIM → `scoped` (estado isolado por request)
   └─ NÃO → continuar

5. A criação da support é custosa (parsing pesado, warm-up, carregamento de arquivos)?
   └─ SIM → `singleton` (criar uma vez, reutilizar)
   └─ NÃO → continuar

6. A support é usada em múltiplos pontos da mesma request?
   └─ SIM → `singleton` (evitar recriação)
   └─ NÃO → `bind` (criar sob demanda)
```

---

## Definições

### `singleton` — Uma instância para toda a aplicação

A mesma instância é compartilhada entre todas as requests e todos os pontos de
injeção. Vive até o processo encerrar (ou worker reiniciar em queue/Octane).

**Usar quando:**
- Support é stateless (sem estado mutável)
- Criação é custosa (parsing de config, carregamento de regras, warm-up)
- Reutilizada em vários pontos da aplicação
- Carrega dicionários, patterns ou tabelas de lookup reutilizáveis

**Exemplos reais:**

```php
// ✅ Singleton — regras imutáveis, stateless, criação custosa
$this->app->singleton(
    InputValidator::class,
    fn () => new InputValidator(
        locale: config('app.locale'),
        strictMode: (bool) config('validation.strict_mode', true),
    ),
);

// ✅ Singleton — formatador stateless, config imutável
$this->app->singleton(
    CurrencyFormatter::class,
    fn () => new CurrencyFormatter(
        defaultCurrency: config('app.currency', 'BRL'),
        locale: config('app.locale'),
    ),
);

// ✅ Singleton — formatador de dados, stateless, usado em muitos pontos
$this->app->singleton(DataFormatter::class);
```

**Cuidados:**
- Nunca armazenar dados de request no singleton (Auth::user(), request())
- Em Laravel Octane / queue workers, singletons persistem entre requests —
  qualquer estado mutável causará vazamento de dados entre requests

---

### `scoped` — Uma instância por request (lifecycle de request)

Nova instância criada a cada request HTTP ou job de queue. Dentro da mesma
request, todas as injeções recebem a mesma instância.

**Usar quando:**
- Support acumula estado durante a request (histórico, contadores, buffers, cache local)
- Support precisa de isolamento entre requests (dados de request-specific)
- Support faz tracking de operações dentro da request

**Exemplos reais:**

```php
// ✅ Scoped — acumula histórico de navegação durante a request
$this->app->scoped(NavigationHistory::class);

// ✅ Scoped — cache de resultados por request, estado mutável
$this->app->scoped(
    QueryResultCache::class,
    fn () => new QueryResultCache(
        ttl: (int) config('cache.request_ttl', 60),
    ),
);

// ✅ Scoped — estado mutável acumulado (breadcrumbs da request)
$this->app->scoped(BreadcrumbCollector::class);
```

**Quando NÃO usar scoped:**
- Support stateless sem estado mutável → usar `singleton`
- Support com interface que troca implementação → `singleton` se stateless

---

### `bind` — Nova instância a cada resolução

Cada vez que o container resolve a classe, cria uma nova instância.

**Usar quando:**
- Support é barata de criar
- Cada consumidor precisa de sua própria instância isolada
- Support é usada apenas uma vez por request
- Support carrega dados específicos da chamada (factory-like)

**Exemplos reais:**

```php
// ✅ Bind — cada uso precisa de instância nova com dados diferentes
$this->app->bind(
    CsvWriter::class,
    fn () => new CsvWriter(
        tempDir: storage_path('app/exports'),
        delimiter: config('exports.csv_delimiter', ','),
    ),
);

// ✅ Bind — interface com implementação que varia por contexto
$this->app->bind(FormatterInterface::class, JsonFormatter::class);
```

---

### Sem registro — Auto-resolve do Laravel

O container resolve automaticamente qualquer classe concreta cujas dependências
também são resolvíveis. Não registrar quando não há necessidade.

**Usar quando:**
- Classe concreta (sem interface)
- Todas as dependências são tipadas e resolvíveis
- Não precisa de config dinâmica no construtor

**Exemplo:**

```php
// Support sem config dinâmica, dependências auto-resolvíveis
final readonly class InputSanitizer
{
    public function __construct(
        private CharacterMap $characterMap,   // ← classe concreta
        private PatternMatcher $patternMatcher, // ← classe concreta
    ) {}
}

// NÃO precisa registrar. Laravel resolve automaticamente:
// - CharacterMap: classe concreta, auto-resolve
// - PatternMatcher: classe concreta, auto-resolve
```

---

## Tabela de Decisão Rápida

| Característica da Support                | Binding      |
|------------------------------------------|--------------|
| Stateless + config dinâmica              | `singleton`  |
| Stateless + interface                    | `singleton`  |
| Stateless + criação custosa              | `singleton`  |
| Stateless + usada em muitos pontos       | `singleton`  |
| Estado mutável por request               | `scoped`     |
| Acumula dados durante request            | `scoped`     |
| Tracking/histórico por request           | `scoped`     |
| Nova instância necessária a cada uso     | `bind`       |
| Uso pontual + criação barata             | `bind`       |
| Classe concreta sem config dinâmica      | Não registrar|

---

## Padrões Comuns do Projeto

Baseado na stack do projeto (Laravel 13, Redis, Reverb, APIs externas):

| Support                  | Binding      | Justificativa                              |
|--------------------------|--------------|---------------------------------------------|
| `InputValidator`         | `singleton`  | Stateless, config imutável                  |
| `CurrencyFormatter`      | `singleton`  | Stateless, sem config dinâmica              |
| `NavigationHistory`      | `scoped`     | Estado de navegação por request             |
| `DataFormatter`          | `singleton`  | Stateless, registra sem acumular            |
| `CsvWriter`              | `bind`       | Cada uso é independente, criação barata     |
| `BreadcrumbCollector`    | `scoped`     | Acumula breadcrumbs por request             |

---

## Anti-patterns

### 1. Registrar sem necessidade

```php
// ❌ Desnecessário — Laravel resolve automaticamente
$this->app->singleton(InputSanitizer::class);

// A menos que InputSanitizer tenha construtor com config dinâmica,
// este registro é redundante e adiciona complexidade.
```

### 2. Singleton com estado mutável

```php
// ❌ Estado mutável em singleton causa vazamento entre requests
$this->app->singleton(NavigationHistory::class);

class NavigationHistory {
    private array $visited = []; // ← compartilhado entre TODAS as requests!
}

// ✅ Corrigido — scoped isola por request
$this->app->scoped(NavigationHistory::class);
```

### 3. Scoped para support stateless

```php
// ❌ Scoped desnecessário — support não tem estado mutável
$this->app->scoped(
    CurrencyFormatter::class,
    fn () => new CurrencyFormatter(config('app.currency')),
);

// ✅ Singleton é mais eficiente para stateless
$this->app->singleton(
    CurrencyFormatter::class,
    fn () => new CurrencyFormatter(config('app.currency')),
);
```

### 4. Facade no construtor da support

```php
// ❌ Dificulta testes — dependência oculta
final readonly class ReportFormatter
{
    public function format(): string
    {
        $user = Auth::user();         // ← facade
        Cache::put('key', 'value');   // ← facade
    }
}

// ✅ Injetar contratos — testável e explícito
final readonly class ReportFormatter
{
    public function __construct(
        private Guard $auth,
        private Repository $cache,
    ) {}

    public function format(): string
    {
        $user = $this->auth->user();
        $this->cache->put('key', 'value');
    }
}
```

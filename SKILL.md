---
name: laravel-support-class
description: >
  ONLY activate this skill when the user explicitly requests it by name (e.g.
  'use laravel-support-class', 'aplique laravel-support-class', 'skill
  laravel-support-class'). Do NOT auto-trigger based on context inference. This
  skill standardizes creation, review, and refactoring of utility (Support)
  classes in Laravel 13.x with PHP 8.4. Covers directory structure, helper
  classes (Enums, Exceptions, DTOs, ValueObjects, Constants), and test
  generation with Pest.
---

# Laravel Support Class

Padrões para criação e revisão de Support classes no Laravel 13.x com PHP 8.4.

---

## Estrutura de Diretórios

Cada grupo de utilitários ocupa um diretório próprio em `app/Support/`:

```
app/Support/
├── Validator/
│   ├── DTOs/
│   │   └── ValidationResult.php
│   ├── Enums/
│   │   └── ValidationRule.php
│   ├── Exceptions/
│   │   └── ValidationFailedException.php
│   ├── ValueObjects/
│   │   └── FieldError.php
│   └── InputValidator.php
```

Subdiretórios são criados sob demanda — incluir apenas os que o utilitário precisa.
Tipos de classes auxiliares permitidos:

| Subdiretório     | Conteúdo                                   | Exemplo                        |
|------------------|--------------------------------------------|--------------------------------|
| `DTOs/`          | Data Transfer Objects (input/output)       | `ValidationResult.php`         |
| `Enums/`         | Enums tipados com BackedEnum               | `ValidationRule.php`           |
| `Exceptions/`    | Exceções específicas da classe             | `ValidationFailedException.php`|
| `ValueObjects/`  | Objetos imutáveis de valor                 | `FieldError.php`               |
| `Constants/`     | Constantes agrupadas                       | `FormatPatterns.php`           |

**Classes NÃO permitidas como auxiliares de Support:**

| Tipo              | Motivo                                                     |
|-------------------|------------------------------------------------------------|
| `Services/`       | Services são camada acima — NUNCA auxiliar de Support       |
| `Repositories/`   | Acesso a dados pertence a Services ou ao domínio           |

---

## Anatomia da Support Class

### Regras Obrigatórias

1. `declare(strict_types=1)` no topo
2. `final class` — support classes não são projetadas para herança
3. Dependências injetadas via construtor com `readonly` (quando houver)
4. Métodos públicos com tipagem completa (parâmetros e retorno)
5. Nomes de métodos em inglês, camelCase, com verbo de ação

### Quando Usar `final readonly class`

- **`final readonly class`**: quando a support não possui estado mutável (todas as
  propriedades são definidas no construtor e nunca mudam). Caso mais comum.
- **`final class`**: quando a support precisa de estado interno mutável (cache em
  memória, contadores, buffers, histórico acumulado).

### Template Base

```php
<?php

declare(strict_types=1);

namespace App\Support\GroupName;

final readonly class UtilityName
{
    public function __construct(
        private DependencyA $dependencyA,
        private DependencyB $dependencyB,
    ) {}

    /**
     * Descrição breve da operação.
     *
     * DocBlock só quando agrega valor além da tipagem nativa.
     *
     * @param  array<int, Item>  $items  ← tipar arrays internos
     * @throws SpecificException         ← declarar exceções lançadas
     */
    public function execute(array $items): ResultDTO
    {
        // Early return para pré-condições
        // Lógica principal
        // Retorno tipado
    }
}
```

### Padrão para Métodos Internos

Extrair lógica complexa em métodos `private` com nomes descritivos:

```php
public function validate(array $input): ValidationResult
{
    $this->ensureInputIsNotEmpty($input);
    $errors = $this->collectErrors($input);
    $normalized = $this->normalizeFields($input);

    return new ValidationResult(errors: $errors, normalized: $normalized);
}

private function ensureInputIsNotEmpty(array $input): void
{
    if ($input === []) {
        throw new EmptyInputException();
    }

    if (! array_is_list($input) && count($input) === 0) {
        throw new EmptyInputException();
    }
}

private function collectErrors(array $input): array
{
    return collect($input)
        ->map(fn (mixed $value, string $field) => $this->validateField($field, $value))
        ->filter()
        ->values()
        ->all();
}
```

---

## Classes Auxiliares

### DTOs (Data Transfer Objects)

Objetos para transportar dados entre camadas. Sempre `final readonly class`:

```php
<?php

declare(strict_types=1);

namespace App\Support\Validator\DTOs;

use App\Support\Validator\ValueObjects\FieldError;

final readonly class ValidationResult
{
    /**
     * @param  array<int, FieldError>   $errors
     * @param  array<string, mixed>     $normalized
     */
    public function __construct(
        public array $errors,
        public array $normalized,
    ) {}

    public function isValid(): bool
    {
        return $this->errors === [];
    }

    /**
     * Factory method para construção a partir de resposta externa.
     *
     * @param  array<string, mixed>  $response
     */
    public static function fromValidatorResponse(array $response): self
    {
        return new self(
            errors: array_map(
                fn (array $error) => FieldError::fromArray($error),
                $response['errors'] ?? [],
            ),
            normalized: $response['normalized'] ?? [],
        );
    }
}
```

### Enums

Sempre `BackedEnum` com `string` ou `int`. Incluir métodos utilitários no enum:

```php
<?php

declare(strict_types=1);

namespace App\Support\Validator\Enums;

enum ValidationRule: string
{
    case Required = 'required';
    case Email    = 'email';
    case Numeric  = 'numeric';
    case MinLength = 'min_length';

    public function label(): string
    {
        return match ($this) {
            self::Required  => 'Obrigatório',
            self::Email     => 'E-mail',
            self::Numeric   => 'Numérico',
            self::MinLength => 'Tamanho mínimo',
        };
    }

    public function requiresParameter(): bool
    {
        return in_array($this, [self::MinLength], true);
    }
}
```

### Exceptions

Exceções específicas da classe, estendendo `RuntimeException` ou `DomainException`:

```php
<?php

declare(strict_types=1);

namespace App\Support\Validator\Exceptions;

use RuntimeException;

final class ValidationFailedException extends RuntimeException
{
    public static function forField(string $field, string $reason): self
    {
        return new self(
            "Validation failed for field '{$field}': {$reason}"
        );
    }

    public static function emptyInput(): self
    {
        return new self('Cannot validate empty input');
    }
}
```

### ValueObjects

Objetos imutáveis que representam um valor do domínio:

```php
<?php

declare(strict_types=1);

namespace App\Support\Validator\ValueObjects;

use InvalidArgumentException;

final readonly class FieldError
{
    public function __construct(
        public string $field,
        public string $message,
        public string $rule,
    ) {
        if ($field === '') {
            throw new InvalidArgumentException('Field name cannot be empty');
        }
    }

    /**
     * @param  array<string, string>  $data
     */
    public static function fromArray(array $data): self
    {
        return new self(
            field: $data['field'],
            message: $data['message'],
            rule: $data['rule'],
        );
    }

    public function toArray(): array
    {
        return [
            'field'   => $this->field,
            'message' => $this->message,
            'rule'    => $this->rule,
        ];
    }
}
```

---

## Registro no AppServiceProvider

Guia completo de decisão em **`references/container-binding-guide.md`**.

### Resumo Rápido

| Binding     | Quando usar                                         |
|-------------|-----------------------------------------------------|
| `singleton` | Sem estado mutável E reutilizado em vários pontos   |
| `scoped`    | Estado mutável por request OU isolamento por request |
| `bind`      | Sem estado, criação barata, uso pontual             |
| Nenhum      | Sem interface, sem config dinâmica — auto-resolve    |

### Regra de Ouro

A maioria das support classes do projeto **não precisa de registro manual**. O
container do Laravel resolve automaticamente classes concretas com dependências
tipadas. Registrar manualmente apenas quando:

1. A support precisa de configuração dinâmica no construtor (paths, configs)
2. A support precisa garantir instância única (`singleton`) ou por request (`scoped`)
3. A support usa uma interface e precisa de binding explícito (opcional, não obrigatório)

```php
// AppServiceProvider.php — boot() ou register()

// Singleton — config dinâmica justifica o registro
$this->app->singleton(
    InputValidator::class,
    fn () => new InputValidator(
        locale: config('app.locale'),
        strictMode: (bool) config('validation.strict_mode', true),
    ),
);
```

---

## Testabilidade

Support classes projetadas para teste devem:

1. Receber todas as dependências pelo construtor (permite mocking)
2. Ter métodos com entrada e saída claras (testáveis isoladamente)
3. Lançar exceções tipadas (verificáveis em testes)

Guia completo de testes com Pest em **`references/pest-testing-guide.md`**.

### Exemplo Rápido

```php
it('validates a required field', function () {
    $validator = new InputValidator(locale: 'pt_BR');

    $result = $validator->validate([
        'name' => '',
    ], [
        'name' => [ValidationRule::Required],
    ]);

    expect($result)
        ->toBeInstanceOf(ValidationResult::class)
        ->isValid()->toBeFalse()
        ->errors->toHaveCount(1);
});
```

---

## Referências

| Arquivo                                  | Conteúdo                                     | Carregar quando                        |
|------------------------------------------|----------------------------------------------|----------------------------------------|
| `references/container-binding-guide.md`  | Decisão singleton vs scoped vs bind          | Registrando support no container       |
| `references/pest-testing-guide.md`       | Padrões de teste com Pest para supports      | Criando testes para supports           |
| `references/support-class-template.md`   | Template completo com exemplo real           | Criando support do zero                |

---

## Documentação de Uso para AI (Obrigatório)

Toda support class DEVE ter UM ÚNICO arquivo `.md` no mesmo diretório. O nome do
arquivo é a conversão do nome da classe de PascalCase para kebab-case.

**Regra de nomenclatura**: cada letra maiúscula (exceto a primeira) é precedida por
hífen, tudo minúsculo.

- `InputValidator.php` → `input-validator.md`
- `NavigationHistory.php` → `navigation-history.md`
- `S3FileUploader.php` → `s3-file-uploader.md`
- `CurrencyFormatter.php` → `currency-formatter.md`

**Propósito**: instruções para agentes AI (Claude Code) consumirem a support.
Não é documentação para desenvolvedores — é referência compacta, sem didática,
otimizada para economia de tokens.

### Regra Fundamental: Um Arquivo, Uma Documentação

NUNCA gerar documentos separados para classes auxiliares. NUNCA criar seções
independentes que tratem a classe auxiliar como autônoma.

A documentação é sobre a classe principal. Classes auxiliares são documentadas
DENTRO do contexto da classe principal, como extensão da sua API.

### Localização

```
app/Support/Validator/
├── InputValidator.php
├── input-validator.md     ← arquivo único de instruções AI
├── DTOs/
│   └── ValidationResult.php
├── ValueObjects/
│   └── FieldError.php
└── ...
```

### Como Identificar Classes Auxiliares

Uma classe auxiliar é qualquer classe cujos métodos públicos são acessíveis a
partir da classe principal. Isso acontece quando:

- Um método público da classe principal retorna uma instância da classe auxiliar
- Uma propriedade pública da classe principal é uma instância da classe auxiliar
- A classe principal usa composição e delega operações para a auxiliar

### Estrutura Obrigatória do Arquivo

Seguir EXATAMENTE esta estrutura. Não alterar a ordem, não adicionar seções, não
remover seções.

```markdown
# {SupportName}

Namespace: `App\Support\GroupName`

Dependências: `TipoA`, `TipoB` (via construtor)

## Métodos

### metodoSimples(TipoParam $param): TipoRetorno

Descrição direta em uma ou duas frases.

- `$param` — descrição funcional
- Returns: descrição do retorno
- Throws: `SpecificException` quando [condição]

### metodoQueRetornaAuxiliar(): ClasseAuxiliar

Descrição direta do método.

- Returns: instância de `ClasseAuxiliar` com os métodos abaixo

#### Métodos acessíveis via ClasseAuxiliar

#### metodoAuxiliarA(Tipo $param): TipoRetorno

Descrição.

- `$param` — descrição
- Returns: descrição

#### metodoAuxiliarB(): TipoRetorno

Descrição.

- Returns: descrição

## Uso Rápido

A variável que recebe a instância da support DEVE usar o nome da classe em
camelCase. Exemplos:
- `InputValidator` → `$inputValidator`
- `NavigationHistory` → `$navigationHistory`
- `S3FileUploader` → `$s3FileUploader`
- `CurrencyFormatter` → `$currencyFormatter`

\```php
$utilityName = app(UtilityName::class);

// Método simples
$resultado = $utilityName->metodoSimples($valor);

// Método com classe auxiliar (encadeamento)
$resultado = $utilityName->metodoQueRetornaAuxiliar()
    ->metodoAuxiliarA($param)
    ->metodoAuxiliarB();
\```
```

### Regras de Escrita

1. **Um único arquivo** — NUNCA gerar múltiplos `.md` para a mesma support
2. **Sem explicações** — apenas assinaturas, tipos e comportamento
3. **Uma linha por conceito** — sem parágrafos
4. **Somente métodos públicos** — métodos `private`/`protected` são internos, NUNCA documentá-los
5. **Métodos mágicos** — não documentar (`__construct`, `__toString`) exceto se tiverem comportamento relevante para uso externo
6. **Getters/setters triviais** — não documentar se apenas retornam/atribuem uma propriedade
7. **Classes auxiliares como submétodos** — documentar métodos públicos de classes auxiliares DENTRO do método da classe principal que as expõe, usando `####` e a seção "Métodos acessíveis via {ClasseAuxiliar}"
8. **Seção "Uso Rápido"** — snippet mínimo cobrindo instanciação e todos os padrões de uso. A variável DEVE usar o nome da classe em camelCase (ex: `$inputValidator = app(InputValidator::class)`)
9. **Resolver via container** — sempre mostrar `app(Class::class)`, nunca `new`
10. **Exceções** — listar quais e quando são lançadas (uma linha cada)
11. **Sem formatação decorativa** — sem badges, emojis, seções "Introdução" ou "Visão Geral"

### Formatação de Parâmetros e Retorno

- Parâmetros: liste com travessão, um por linha: `- \`$param\` — descrição funcional`
- Retorno: sempre inclua: `- Returns: descrição`
- Exceções: `- Throws: \`ExceptionType\` quando [condição]`
- Blocos de código: sempre com \```php

### Exemplo Real: input-validator.md

```markdown
# InputValidator

Namespace: `App\Support\Validator`

Dependências: nenhuma (opcional: `locale`, `strictMode` via construtor)

## Métodos

### validate(array $input, array $rules): ValidationResult

Valida um array de campos contra um conjunto de regras.

- `$input` — dados a validar, chave-valor
- `$rules` — mapa campo → lista de `ValidationRule`
- Returns: `ValidationResult` com `errors`, `normalized`, `isValid()`
- Throws: `ValidationFailedException` quando input está vazio

### field(string $name): FieldValidator

Cria um validador fluente para um único campo.

- `$name` — nome do campo
- Returns: instância de `FieldValidator` com os métodos abaixo

#### Métodos acessíveis via FieldValidator

#### required(): self

Marca o campo como obrigatório.

- Returns: `self` (encadeável)

#### email(): self

Valida formato de e-mail.

- Returns: `self` (encadeável)

#### min(int $length): self

Define tamanho mínimo.

- `$length` — quantidade mínima de caracteres
- Returns: `self` (encadeável)

#### check(mixed $value): ?FieldError

Executa a validação contra um valor.

- `$value` — valor a validar
- Returns: `FieldError` se inválido, `null` se válido

## Uso Rápido

\```php
$inputValidator = app(InputValidator::class);

// Validação em lote
$result = $inputValidator->validate(
    input: ['email' => 'a@b.com', 'name' => ''],
    rules: [
        'email' => [ValidationRule::Required, ValidationRule::Email],
        'name'  => [ValidationRule::Required],
    ],
);

// Validação fluente campo a campo
$error = $inputValidator->field('email')
    ->required()
    ->email()
    ->check($payload['email']);
\```
```

---

## Revisão de Support Class Existente

Quando o usuário pedir para **revisar, melhorar ou refatorar** uma support class já
existente, seguir OBRIGATORIAMENTE este fluxo:

### Fluxo de Revisão

1. **Analisar** — ler o código atual completo da classe e suas auxiliares
2. **Apresentar diagnóstico** — listar para o usuário TODAS as melhorias e correções
   identificadas, organizadas por categoria:
   - 🔴 **Correções** — problemas que violam as regras desta skill (falta de
     `strict_types`, classe não `final`, tipagem incompleta, etc.)
   - 🟡 **Melhorias** — refatorações que elevam a qualidade (extração de métodos,
     DTOs faltantes, exceções genéricas que deveriam ser específicas, etc.)
   - 🟢 **Sugestões** — ajustes opcionais de estilo ou organização
3. **Aguardar autorização** — o usuário DEVE aprovar antes de qualquer alteração no
   código. Perguntar explicitamente: _"Posso prosseguir com essas alterações?"_
4. **Aplicar** — somente após aprovação, gerar o código atualizado

### Regra Fundamental

**NUNCA alterar código de support class existente sem apresentar o diagnóstico
primeiro e receber autorização explícita do usuário.** Mesmo que as correções sejam
óbvias ou triviais, o usuário precisa saber o que será alterado.

---

## Checklist Pré-Entrega

Verificar antes de entregar qualquer support:

- [ ] `declare(strict_types=1)` presente
- [ ] Classe marcada como `final` (ou `final readonly`)
- [ ] Dependências injetadas via construtor com `readonly` (quando houver)
- [ ] Métodos públicos com tipagem completa
- [ ] Nomes em inglês, camelCase, verbos de ação
- [ ] Exceções específicas da classe (não genéricas)
- [ ] DTOs para entrada/saída complexa
- [ ] Enums para valores finitos
- [ ] Diretório da support criado em `app/Support/` ou `app/Support/GroupName/`
- [ ] Nenhuma Service ou Repository como classe auxiliar
- [ ] Registro no AppServiceProvider justificado (ou auto-resolve documentado)
- [ ] Arquivo `{support-name}.md` (kebab-case) com instruções AI criado no diretório da support
- [ ] Testes com Pest quando solicitados

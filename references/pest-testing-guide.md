# Pest Testing Guide — Support Classes

Padrões para testar support classes do projeto usando Pest PHP.

---

## Sumário

- Estrutura de diretórios de teste
- Anatomia de um teste de support
- Mocking de dependências
- Testando exceções
- Testando DTOs e ValueObjects
- Testando Enums
- Padrões comuns

---

## Estrutura de Diretórios

Espelhar a estrutura de `app/Support/` em `tests/Unit/Support/`:

```
tests/
├── Unit/
│   └── Support/
│       └── Validator/
│           ├── InputValidatorTest.php
│           ├── DTOs/
│           │   └── ValidationResultTest.php
│           ├── Enums/
│           │   └── ValidationRuleTest.php
│           └── ValueObjects/
│               └── FieldErrorTest.php
```

---

## Anatomia de um Teste de Support

### Estrutura Básica

```php
<?php

declare(strict_types=1);

use App\Support\Validator\InputValidator;
use App\Support\Validator\DTOs\ValidationResult;
use App\Support\Validator\Enums\ValidationRule;
use App\Support\Validator\Exceptions\ValidationFailedException;
use App\Support\Validator\ValueObjects\FieldError;

describe('InputValidator', function () {

    describe('validate', function () {

        it('validates a valid input successfully', function () {
            // Arrange
            $validator = new InputValidator(locale: 'pt_BR');

            // Act
            $result = $validator->validate(
                input: ['email' => 'user@example.com', 'name' => 'John'],
                rules: [
                    'email' => [ValidationRule::Required, ValidationRule::Email],
                    'name'  => [ValidationRule::Required],
                ],
            );

            // Assert
            expect($result)
                ->toBeInstanceOf(ValidationResult::class)
                ->isValid()->toBeTrue()
                ->errors->toBe([])
                ->normalized->toMatchArray([
                    'email' => 'user@example.com',
                    'name'  => 'John',
                ]);
        });

        it('returns errors when required field is missing', function () {
            $validator = new InputValidator(locale: 'pt_BR');

            $result = $validator->validate(
                input: ['email' => '', 'name' => 'John'],
                rules: [
                    'email' => [ValidationRule::Required],
                    'name'  => [ValidationRule::Required],
                ],
            );

            expect($result)
                ->isValid()->toBeFalse()
                ->errors->toHaveCount(1)
                ->and($result->errors[0])
                ->field->toBe('email')
                ->rule->toBe('required');
        });

        it('throws exception for empty input', function () {
            $validator = new InputValidator(locale: 'pt_BR');

            expect(fn () => $validator->validate([], []))
                ->toThrow(ValidationFailedException::class);
        });
    });
});
```

### Convenções de Nomeação

- Arquivo: `{SupportName}Test.php`
- Describe blocks: agrupar por método público
- It blocks: descrever o comportamento esperado em inglês

```php
describe('SupportName', function () {
    describe('methodName', function () {
        it('does X when Y', function () { ... });
        it('throws Z when W', function () { ... });
        it('returns null when not found', function () { ... });
    });
});
```

---

## Mocking de Dependências

### Com Mockery (padrão do Pest)

```php
// Mock simples
$patterns = Mockery::mock(PatternMatcher::class);
$patterns->shouldReceive('match')
    ->with('user@example.com')
    ->once()
    ->andReturn(true);

// Mock com callback de validação
$patterns->shouldReceive('match')
    ->once()
    ->with(Mockery::on(function (string $value) {
        return str_contains($value, '@');
    }))
    ->andReturn(true);

// Mock que lança exceção
$loader->shouldReceive('load')
    ->once()
    ->andThrow(new RuntimeException('File not found'));
```

### Instanciar a Support nos Testes

Criar a support manualmente injetando mocks — não usar o container:

```php
// ✅ Correto — controle total das dependências
$validator = new InputValidator(
    characterMap: $mockCharacterMap,
    patternMatcher: $mockPatternMatcher,
);

// ❌ Evitar — perde controle, pode usar implementações reais
$validator = app(InputValidator::class);
```

### BeforeEach para Setup Compartilhado

```php
describe('InputValidator', function () {

    beforeEach(function () {
        $this->characterMap = Mockery::mock(CharacterMap::class);
        $this->patternMatcher = Mockery::mock(PatternMatcher::class);
        $this->validator = new InputValidator(
            characterMap: $this->characterMap,
            patternMatcher: $this->patternMatcher,
        );
    });

    it('validates an input', function () {
        $this->patternMatcher->shouldReceive('match')->once()->andReturn(true);
        // ...
    });
});
```

---

## Testando Exceções

```php
// Verificar tipo da exceção
it('throws when input is empty', function () {
    expect(fn () => $this->validator->validate([], []))
        ->toThrow(ValidationFailedException::class);
});

// Verificar mensagem da exceção
it('throws with descriptive message', function () {
    expect(fn () => $this->validator->validate([], []))
        ->toThrow(ValidationFailedException::class, 'Cannot validate empty input');
});

// Verificar exceção com factory method
it('throws for-field exception with field name', function () {
    $exception = ValidationFailedException::forField('email', 'Invalid format');

    expect($exception)
        ->toBeInstanceOf(ValidationFailedException::class)
        ->getMessage()->toContain('email')
        ->getMessage()->toContain('Invalid format');
});
```

---

## Testando DTOs

```php
describe('ValidationResult', function () {

    it('creates from validator response', function () {
        $response = [
            'errors' => [
                ['field' => 'email', 'message' => 'Invalid', 'rule' => 'email'],
            ],
            'normalized' => ['name' => 'John'],
        ];

        $result = ValidationResult::fromValidatorResponse($response);

        expect($result)
            ->errors->toHaveCount(1)
            ->normalized->toMatchArray(['name' => 'John'])
            ->isValid()->toBeFalse();
    });

    it('is valid when no errors present', function () {
        $result = new ValidationResult(errors: [], normalized: ['name' => 'John']);

        expect($result)
            ->isValid()->toBeTrue()
            ->errors->toBe([]);
    });
});
```

---

## Testando ValueObjects

```php
describe('FieldError', function () {

    it('creates with field, message and rule', function () {
        $error = new FieldError(
            field: 'email',
            message: 'Invalid format',
            rule: 'email',
        );

        expect($error)
            ->field->toBe('email')
            ->message->toBe('Invalid format')
            ->rule->toBe('email');
    });

    it('creates from array', function () {
        $error = FieldError::fromArray([
            'field'   => 'name',
            'message' => 'Required',
            'rule'    => 'required',
        ]);

        expect($error)
            ->field->toBe('name')
            ->rule->toBe('required');
    });

    it('serializes to array', function () {
        $error = new FieldError(field: 'x', message: 'y', rule: 'z');

        expect($error->toArray())->toMatchArray([
            'field'   => 'x',
            'message' => 'y',
            'rule'    => 'z',
        ]);
    });

    it('rejects empty field name', function () {
        expect(fn () => new FieldError(field: '', message: 'x', rule: 'y'))
            ->toThrow(InvalidArgumentException::class);
    });
});
```

---

## Testando Enums

```php
describe('ValidationRule', function () {

    it('has correct backing values', function () {
        expect(ValidationRule::Required->value)->toBe('required');
        expect(ValidationRule::Email->value)->toBe('email');
    });

    it('returns localized labels', function () {
        expect(ValidationRule::Required->label())->toBe('Obrigatório');
        expect(ValidationRule::Email->label())->toBe('E-mail');
    });

    it('identifies rules that require a parameter', function () {
        expect(ValidationRule::MinLength->requiresParameter())->toBeTrue();
        expect(ValidationRule::Required->requiresParameter())->toBeFalse();
    });

    it('creates from string value', function () {
        expect(ValidationRule::from('required'))->toBe(ValidationRule::Required);
    });

    it('returns null for invalid value with tryFrom', function () {
        expect(ValidationRule::tryFrom('invalid'))->toBeNull();
    });
});
```

---

## Testes com Integração Laravel (quando necessário)

Para supports que interagem com banco, cache ou filesystem, usar testes de integração:

```php
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

describe('AuditTrailWriter (integration)', function () {

    it('writes an entry to the database', function () {
        $writer = app(AuditTrailWriter::class);

        $writer->write(
            action: 'user.created',
            context: ['user_id' => 1],
        );

        $this->assertDatabaseHas('audit_trail', ['action' => 'user.created']);
    });
});
```

Preferir testes unitários com mocks. Usar integração apenas quando o
comportamento do banco/filesystem faz parte da lógica sendo testada.

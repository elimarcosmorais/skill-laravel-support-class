# Support Class Template — Exemplo Completo

Exemplo real de uma support completa com todas as classes auxiliares,
registro no AppServiceProvider e testes Pest.

---

## Sumário

- Exemplo completo: InputValidator
- Estrutura de diretórios
- Cada arquivo da support
- Registro no AppServiceProvider
- Testes Pest

---

## Exemplo: InputValidator

Support que valida arrays de entrada contra um conjunto de regras, retornando
erros estruturados e dados normalizados. Usada em controllers, jobs e services.

### Estrutura

```
app/Support/Validator/
├── DTOs/
│   └── ValidationResult.php
├── Enums/
│   └── ValidationRule.php
├── Exceptions/
│   └── ValidationFailedException.php
├── ValueObjects/
│   └── FieldError.php
├── InputValidator.php
└── input-validator.md       ← instruções AI
```

---

### Enums/ValidationRule.php

```php
<?php

declare(strict_types=1);

namespace App\Support\Validator\Enums;

enum ValidationRule: string
{
    case Required  = 'required';
    case Email     = 'email';
    case Numeric   = 'numeric';
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

---

### ValueObjects/FieldError.php

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

    /**
     * @return array<string, string>
     */
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

### DTOs/ValidationResult.php

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

---

### Exceptions/ValidationFailedException.php

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

    public static function unknownRule(string $rule): self
    {
        return new self(
            "Unknown validation rule: '{$rule}'"
        );
    }
}
```

---

### InputValidator.php

```php
<?php

declare(strict_types=1);

namespace App\Support\Validator;

use App\Support\Validator\DTOs\ValidationResult;
use App\Support\Validator\Enums\ValidationRule;
use App\Support\Validator\Exceptions\ValidationFailedException;
use App\Support\Validator\ValueObjects\FieldError;

final readonly class InputValidator
{
    public function __construct(
        private string $locale = 'pt_BR',
        private bool $strictMode = true,
    ) {}

    /**
     * Valida um array de campos contra um conjunto de regras.
     *
     * @param  array<string, mixed>                 $input
     * @param  array<string, array<int, ValidationRule>>  $rules
     *
     * @throws ValidationFailedException
     */
    public function validate(array $input, array $rules): ValidationResult
    {
        $this->ensureInputIsNotEmpty($input);

        $errors     = [];
        $normalized = [];

        foreach ($rules as $field => $fieldRules) {
            $value = $input[$field] ?? null;
            $error = $this->validateField($field, $value, $fieldRules);

            if ($error !== null) {
                $errors[] = $error;
                continue;
            }

            $normalized[$field] = $this->normalizeValue($value);
        }

        return new ValidationResult(errors: $errors, normalized: $normalized);
    }

    /**
     * @param  array<string, mixed>  $input
     */
    private function ensureInputIsNotEmpty(array $input): void
    {
        if ($input === [] && $this->strictMode) {
            throw ValidationFailedException::emptyInput();
        }
    }

    /**
     * @param  array<int, ValidationRule>  $rules
     */
    private function validateField(string $field, mixed $value, array $rules): ?FieldError
    {
        foreach ($rules as $rule) {
            if (! $this->checkRule($rule, $value)) {
                return new FieldError(
                    field: $field,
                    message: $this->messageFor($rule),
                    rule: $rule->value,
                );
            }
        }

        return null;
    }

    private function checkRule(ValidationRule $rule, mixed $value): bool
    {
        return match ($rule) {
            ValidationRule::Required  => $value !== null && $value !== '',
            ValidationRule::Email     => is_string($value) && filter_var($value, FILTER_VALIDATE_EMAIL) !== false,
            ValidationRule::Numeric   => is_numeric($value),
            ValidationRule::MinLength => is_string($value) && mb_strlen($value) >= 3,
        };
    }

    private function messageFor(ValidationRule $rule): string
    {
        return match ($rule) {
            ValidationRule::Required  => 'Este campo é obrigatório.',
            ValidationRule::Email     => 'Informe um e-mail válido.',
            ValidationRule::Numeric   => 'O valor deve ser numérico.',
            ValidationRule::MinLength => 'O tamanho mínimo não foi atingido.',
        };
    }

    private function normalizeValue(mixed $value): mixed
    {
        if (is_string($value)) {
            return trim($value);
        }

        return $value;
    }
}
```

---

### Registro no AppServiceProvider

```php
// AppServiceProvider.php

use App\Support\Validator\InputValidator;

public function register(): void
{
    // Singleton — stateless, config imutável, usada em muitos pontos
    $this->app->singleton(
        InputValidator::class,
        fn () => new InputValidator(
            locale: (string) config('app.locale', 'pt_BR'),
            strictMode: (bool) config('validation.strict_mode', true),
        ),
    );
}
```

**Justificativa: `singleton`**
- Support é stateless (não acumula dados entre chamadas)
- Configuração é imutável (locale, strictMode)
- Usada em controllers, jobs e listeners — reutilização justifica singleton

---

### Testes Pest

```php
<?php

declare(strict_types=1);

use App\Support\Validator\InputValidator;
use App\Support\Validator\DTOs\ValidationResult;
use App\Support\Validator\Enums\ValidationRule;
use App\Support\Validator\Exceptions\ValidationFailedException;

describe('InputValidator', function () {

    beforeEach(function () {
        $this->validator = new InputValidator(locale: 'pt_BR', strictMode: true);
    });

    describe('validate', function () {

        it('returns valid result when all rules pass', function () {
            $result = $this->validator->validate(
                input: ['email' => 'user@example.com', 'name' => 'John'],
                rules: [
                    'email' => [ValidationRule::Required, ValidationRule::Email],
                    'name'  => [ValidationRule::Required],
                ],
            );

            expect($result)
                ->toBeInstanceOf(ValidationResult::class)
                ->isValid()->toBeTrue()
                ->errors->toBe([])
                ->normalized->toMatchArray([
                    'email' => 'user@example.com',
                    'name'  => 'John',
                ]);
        });

        it('returns errors for missing required field', function () {
            $result = $this->validator->validate(
                input: ['email' => '', 'name' => 'John'],
                rules: [
                    'email' => [ValidationRule::Required],
                    'name'  => [ValidationRule::Required],
                ],
            );

            expect($result)
                ->isValid()->toBeFalse()
                ->errors->toHaveCount(1);

            expect($result->errors[0])
                ->field->toBe('email')
                ->rule->toBe('required');
        });

        it('returns errors for invalid email format', function () {
            $result = $this->validator->validate(
                input: ['email' => 'not-an-email'],
                rules: [
                    'email' => [ValidationRule::Required, ValidationRule::Email],
                ],
            );

            expect($result)
                ->isValid()->toBeFalse()
                ->errors->toHaveCount(1);

            expect($result->errors[0])
                ->field->toBe('email')
                ->rule->toBe('email');
        });

        it('normalizes string values by trimming whitespace', function () {
            $result = $this->validator->validate(
                input: ['name' => '  John  '],
                rules: ['name' => [ValidationRule::Required]],
            );

            expect($result->normalized['name'])->toBe('John');
        });

        it('throws exception for empty input in strict mode', function () {
            expect(fn () => $this->validator->validate([], []))
                ->toThrow(ValidationFailedException::class);
        });

        it('accepts empty input when strict mode is disabled', function () {
            $validator = new InputValidator(locale: 'pt_BR', strictMode: false);

            expect(fn () => $validator->validate([], []))
                ->not->toThrow(ValidationFailedException::class);
        });
    });
});
```

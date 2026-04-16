# laravel-support-class (v1.0.0)

Skill para [Claude Code](https://docs.anthropic.com/en/docs/claude-code) que padroniza a criação, revisão e refatoração de classes utilitárias (Support) em Laravel, seguindo padrões enterprise de organização, tipagem e testabilidade.

## O que faz

Recebe requisições para criar ou revisar Support classes e entrega código padronizado aplicando:

- **Support Class:** `final readonly class` com `strict_types`, injeção via construtor com `readonly`, tipagem completa, métodos com verbos de ação em inglês
- **Classes Auxiliares:** DTOs, Enums (`BackedEnum`), Exceptions específicas, ValueObjects imutáveis e Constants — criados sob demanda no diretório da support (Services e Repositories NÃO são permitidos como auxiliares)
- **Container Binding:** decisão guiada entre `singleton`, `scoped` e `bind` com registro no `AppServiceProvider`
- **Testes:** geração com Pest seguindo padrões de teste para supports
- **Documentação:** gera `.md` da support class otimizado para agentes AI

> **Stack:** PHP 8.4 · Laravel 13.x · Pest

---

## Instalação

Requisito: [Node.js](https://nodejs.org/) instalado (para `npx`).

```bash
npx degit elimarcosmorais/skill-laravel-support-class .claude/skills/laravel-support-class
```

Isso baixa a skill diretamente para `.claude/skills/laravel-support-class` sem histórico git.

### Verificar a instalação

```bash
ls .claude/skills/laravel-support-class/
# SKILL.md  references/
```

---

## Atualização

Para atualizar para a versão mais recente, rode o mesmo comando com a flag `--force`:

```bash
npx degit elimarcosmorais/skill-laravel-support-class .claude/skills/laravel-support-class --force
```

A flag `--force` sobrescreve os arquivos existentes.

### Instalar uma versão específica

Se o repositório usar tags de versão:

```bash
npx degit elimarcosmorais/skill-laravel-support-class#v1.0.0 .claude/skills/laravel-support-class --force
```

---

## Estrutura

```
laravel-support-class/
├── SKILL.md                                  # Instruções para o Claude Code
└── references/
    ├── support-class-template.md             # Template completo com exemplo real
    ├── container-binding-guide.md            # Decisão singleton vs scoped vs bind
    └── pest-testing-guide.md                 # Padrões de teste com Pest para supports
```

---

## Como usar

Com a skill instalada, basta pedir ao Claude Code para criar ou revisar uma support:

```
Crie uma support de validação de entrada seguindo a skill laravel-support-class
```

A skill será acionada automaticamente quando você mencionar termos como _criar support_, _refatorar support_, _testar support_, _registrar support_, _validator_, _input validator_, _utility_, _helper_, _extrair lógica para support_, entre outros.

---

## Desinstalação

```bash
rm -rf .claude/skills/laravel-support-class
```

---

## Licença

MIT

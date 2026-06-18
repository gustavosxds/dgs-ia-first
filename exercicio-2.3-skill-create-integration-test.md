# Exercício 2.3 — Skill: create-integration-test
> **Papel:** QA | **Projeto:** NovaTech Assistant | **Ferramentas:** Claude (chat) + Claude Cowork
> **Nível na hierarquia:** Artifact skill (usa Foundation e Domain como base)
> **Localização no repo:** `/skills/artifact/create-integration-test/SKILL.md`

---

## SKILL.md — create-integration-test

### Quando usar esta skill

Use esta skill quando for criar ou revisar um **teste de integração** para qualquer endpoint do projeto NovaTech Assistant.

**Frase-ativação:** "crie um teste de integração para [endpoint/função]" ou "escreva testes para [módulo] seguindo os padrões do projeto".

**Não usar para:** testes unitários de funções puras (use `create-unit-test`), testes E2E no Teams (use `create-e2e-test`).

---

### Dependências — leia antes de usar esta skill

Antes de gerar testes com esta skill, leia:

1. **`/skills/foundation/typescript-conventions/SKILL.md`** — convenções de TypeScript (strict mode, imports, naming)
2. **`/skills/domain/testing-patterns/SKILL.md`** — padrões de Vitest, msw, factories e fixtures do projeto
3. **`/docs/agents.md`, seção "Testing Standards"** — regras prescritivas de QA que todo agente deve seguir

---

### Contexto do projeto relevante para testes

- **Framework:** Vitest
- **Mocks HTTP:** msw (Mock Service Worker)
- **Factories:** `/tests/factories/` — construtores de objetos de request/response
- **Fixtures:** `/tests/fixtures/rag/` — chunks, queries e respostas esperadas do domínio NovaTech
- **Serviços externos:** Azure AI Search e Azure OpenAI — NUNCA devem ser chamados em testes (sempre mock via msw)
- **Coverage mínimo:** 80% de linhas por módulo

---

### Template de teste (com placeholders)

```typescript
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest'
import { setupServer } from 'msw/node'
import { [HANDLER_IMPORT] } from '../../src/functions/[MODULE_NAME]'
import { build[REQUEST_FACTORY] } from '../factories/[FACTORY_FILE]'
import { chunks } from '../fixtures/rag/chunks'
import { mock[SERVICE]Response } from '../helpers/msw-handlers'

const server = setupServer()

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

describe('[ModuleName]', () => {
  it('should [COMPORTAMENTO_ESPERADO] when [CONDIÇÃO]', async () => {
    // arrange
    const request = build[REQUEST_FACTORY]({ [CAMPOS_RELEVANTES] })
    server.use(mock[SERVICE]Response([CHUNKS_RELEVANTES]))

    // act
    const response = await [HANDLER_IMPORT](request)

    // assert
    expect(response.status).toBe([EXPECTED_STATUS])
    expect(response.body.[CAMPO_VERIFICADO]).toBe([VALOR_ESPERADO])
    expect(response.body.source_document).toBe('[DOCUMENTO_FONTE, seção X]')
  })
})
```

**Instruções de preenchimento:**
- `[ModuleName]` → nome do módulo em PascalCase (ex: `QueryHandler`)
- `[COMPORTAMENTO_ESPERADO]` → o que o sistema FAZ (ex: `return explicit refusal`)
- `[CONDIÇÃO]` → em qual situação (ex: `when question involves dangerous cargo return`)
- `[CHUNKS_RELEVANTES]` → chunks reais do `/tests/fixtures/rag/chunks.ts`
- `[CAMPO_VERIFICADO]` → campo do response body que valida o comportamento (nunca use `toBeDefined()` sozinho)

---

### Exemplos completos

#### ✅ DO — Teste bem escrito

```typescript
describe('QueryHandler', () => {
  it('should return explicit refusal when question involves dangerous cargo return', async () => {
    // arrange
    const request = buildQueryRequest({
      question: 'Posso devolver carga perigosa classe 3 da ANTT?'
    })
    server.use(mockAzureSearchResponse([chunks.POL_001_B]))

    // act
    const response = await handler(request)

    // assert
    expect(response.status).toBe(200)
    expect(response.body.answer).toMatch(/não.*elegível.*devolução|gestão de riscos/i)
    expect(response.body.source_document).toBe('POL-001, seção 3.2')
  })
})
```

**Por que está certo:**
- Nome descreve comportamento (`return explicit refusal`) e condição (`dangerous cargo return`)
- Pergunta realista do domínio NovaTech (não `"test"`)
- Chunk específico e correto para o cenário (`POL_001_B` — seção de exceções)
- `arrange/act/assert` separados
- Assertion verifica conteúdo da resposta (regex de negativa + campo source_document)

---

#### ❌ DON'T — Teste com problemas comuns de IA

```typescript
// ❌ ESTE TESTE FOI GERADO POR IA SEM GUIDANCE — NÃO USE COMO REFERÊNCIA
test('query endpoint works', async () => {
  const result = await handler({ body: '{"question": "test"}' })
  expect(result).toBeDefined()
})
```

**Por que está errado:**

| Problema | Impacto |
|----------|---------|
| Nome genérico `'query endpoint works'` | Não comunica o que está sendo testado; falha silenciosa em review |
| Input `"test"` sem contexto de domínio | Não exercita nenhum guardrail; o teste passa com qualquer resposta |
| Sem `// arrange / act / assert` | Dificulta manutenção e debug |
| `expect(result).toBeDefined()` | Passa com `null`, `{}`, ou objeto de erro — não verifica comportamento |
| Input como string JSON raw sem factory | Frágil; quebra se o schema mudar |
| Sem mock de serviços externos (msw) | Chama Azure real; não-determinístico, viola segurança |

---

### Anti-padrões específicos de testes gerados por IA

| Anti-padrão | Como aparece | Como corrigir |
|-------------|-------------|---------------|
| **`toBeDefined()` sozinho** | `expect(result).toBeDefined()` | Verificar o campo e valor: `expect(result.source_document).toBe('POL-001, seção 3.2')` |
| **Testa implementação interna** | `jest.spyOn(module, 'buildPrompt')` | Testar o output do handler, não as funções internas |
| **Mocks permissivos demais** | `vi.fn().mockResolvedValue({})` sem shape correto | Usar `mockAzureSearchResponse([chunks.SLA_2024_B])` com dados reais |
| **Dados genéricos** | `question: "test"`, `question: "hello"` | Importar de `/tests/fixtures/rag/queries.ts` com perguntas reais de logística |
| **Sem mock de HTTP externo** | Chama Azure AI Search real no teste | Usar `msw` com `server.use(mockAzureSearchResponse(...))` |
| **Dependência de ordem** | `it` assume estado criado por outro `it` anterior | Cada teste deve ser autossuficiente; usar `beforeEach` para reset |

---

## Checklist de revisão de testes (Claude Cowork)

> **Objetivo:** Verificação rápida — cada item deve ser respondido com Sim/Não em menos de 10 segundos.
> **Tempo total estimado:** < 2 minutos por teste.
> **Regra:** Qualquer resposta "Não" é motivo de reprovação no code review de QA.

### Bloco 1 — Nomenclatura (30 segundos)

- [ ] **1.1** O bloco `describe` tem o nome do módulo em PascalCase (ex: `QueryHandler`)?
- [ ] **1.2** O bloco `it` descreve o comportamento esperado em inglês (começa com `should`)?
- [ ] **1.3** O bloco `it` especifica a condição de disparo (tem `when [condição]`)?

### Bloco 2 — Estrutura (30 segundos)

- [ ] **2.1** O teste tem os comentários `// arrange`, `// act`, `// assert` separando as seções?
- [ ] **2.2** O `// act` tem apenas uma chamada ao código sendo testado?
- [ ] **2.3** Todas as assinaturas de `it(...)` têm `async/await` (para handlers assíncronos)?

### Bloco 3 — Assertions (30 segundos)

- [ ] **3.1** Nenhuma assertion usa `toBeDefined()` ou `toBeTruthy()` sozinha?
- [ ] **3.2** Pelo menos uma assertion verifica o conteúdo do response body (não só status HTTP)?
- [ ] **3.3** O campo `source_document` é verificado quando o cenário tem match de documento?

### Bloco 4 — Dados e mocks (30 segundos)

- [ ] **4.1** A pergunta de teste é do domínio NovaTech (não `"test"`, `"hello"` ou string genérica)?
- [ ] **4.2** Os serviços externos (Azure AI Search, Azure OpenAI) estão mockados com msw?
- [ ] **4.3** Os chunks usados no mock são importados de `/tests/fixtures/rag/chunks.ts`?

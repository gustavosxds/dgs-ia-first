# Exercício 2.1 — Testing Standards para o AGENTS.md
> **Papel:** QA | **Projeto:** NovaTech Assistant | **Ferramenta:** Claude (chat)

---

## Entregável 1: Seção "Testing Standards" do AGENTS.md

```markdown
## Testing Standards

> Esta seção é lida por agentes de IA (Copilot, Claude Code) antes de gerar qualquer código de teste.
> Todo teste gerado DEVE seguir estas regras. Regras marcadas com ❌ NÃO DEVE são violações que causam reprovação no code review de QA.

### Stack de testes

- **Framework:** Vitest
- **Mocks HTTP:** msw (Mock Service Worker) — NUNCA fazer chamadas reais a serviços externos nos testes
- **Factories de dados:** `/tests/factories/` — usar para criar objetos de teste
- **Fixtures:** `/tests/fixtures/` — chunks, queries e respostas esperadas reutilizáveis para testes do pipeline RAG
- **CI:** Testes rodam via GitHub Actions em todo PR
- **Coverage mínimo:** 80% de linhas; PRs que reduzam coverage abaixo deste limiar DEVEM ser rejeitados

---

### Nomenclatura de testes

**DEVE:**
- Usar `describe('NomeDoMódulo', () => { it('should [comportamento] when [condição]', ...) })`
- Nomear em inglês
- O bloco `it` DEVE descrever o comportamento esperado e a condição, nunca o nome da função

**Exemplo correto:**
```typescript
describe('QueryHandler', () => {
  it('should return explicit refusal when question involves dangerous cargo return', async () => { ... })
  it('should include source_document field in every successful response', async () => { ... })
})
```

**NÃO DEVE:**
```typescript
test('query endpoint works', ...) // ❌ nome genérico que não descreve comportamento
test('handler', ...)              // ❌ nome de função, não de comportamento
```

---

### Estrutura interna do teste: Arrange / Act / Assert

**DEVE:** Todo teste DEVE ter as três seções separadas com comentários `// arrange`, `// act`, `// assert`.

**Exemplo obrigatório:**
```typescript
it('should return explicit refusal when question involves dangerous cargo return', async () => {
  // arrange
  const request = buildQueryRequest({ question: 'Posso devolver carga perigosa classe 3?' })
  server.use(mockAzureSearchResponse([chunks.POL001_B_dangerous_cargo]))

  // act
  const response = await handler(request)

  // assert
  expect(response.status).toBe(200)
  expect(response.body.answer).toMatch(/não.*elegível.*devolução|gestão de riscos/i)
  expect(response.body.source_document).toBe('POL-001, seção 3.2')
})
```

**NÃO DEVE:**
```typescript
it('query endpoint works', async () => {
  const result = await handler({ body: '{"question": "test"}' }) // ❌ sem arrange separado, dado genérico
  expect(result).toBeDefined()                                    // ❌ assertion vaga, não verifica comportamento
})
```

---

### Regras de assertions

**DEVE:**
- Assertions verificam o **conteúdo** da resposta, não apenas a existência
- Para respostas de texto: usar `.toMatch()` com regex descritiva do comportamento esperado
- Para campos obrigatórios: verificar valor, não só presença (`toBeDefined()`)
- Para status HTTP: sempre verificar o código e o body

**NÃO DEVE:**
- `expect(result).toBeDefined()` sozinho — ❌ não verifica nenhum comportamento
- `expect(result).toBeTruthy()` sozinho — ❌ mesmo problema
- `expect(result.body).toBeDefined()` — ❌ passa com qualquer body, inclusive erro

---

### Regras de mocking

**DEVE:**
- Usar **msw** para interceptar chamadas HTTP a Azure AI Search e Azure OpenAI
- Usar **factories** (`/tests/factories/`) para criar objetos de request/response com dados realistas do domínio NovaTech
- Declarar `server.use(...)` no `beforeEach` ou no início do teste se o mock for específico daquele cenário
- Restaurar handlers com `server.resetHandlers()` no `afterEach`

**NÃO DEVE:**
- Fazer chamadas reais a Azure AI Search, Azure OpenAI ou qualquer serviço externo nos testes — ❌ torna o teste não-determinístico e viola segurança
- Usar `jest.spyOn` em funções internas do módulo para verificar implementação — ❌ testa implementação, não comportamento
- Usar mocks que aceitam qualquer input e retornam qualquer output (`jest.fn().mockResolvedValue({})`) sem especificar o shape correto — ❌ mock permissivo esconde bugs

---

### Fixtures de dados para o pipeline RAG

**Localização:** `/tests/fixtures/rag/`

Os fixtures DEVEM conter dados realistas do domínio NovaTech:

```
/tests/fixtures/rag/
  chunks.ts          → chunks simulados do Azure AI Search (POL-001, PROC-042v2, SLA-2024)
  queries.ts         → perguntas realistas de atendentes sobre logística
  expected-responses.ts → respostas esperadas para cada cenário de teste
```

**Exemplos de queries obrigatórias nos fixtures:**
- `"Posso devolver carga perigosa classe 3 da ANTT?"`
- `"Qual o SLA de resolução para cliente Gold?"`
- `"Qual o multiplicador de frete especial para Manaus?"`
- `"Qual o prazo de entrega para frete especial no Sul?"`

**NÃO DEVE:**
- Dados genéricos como `"test"`, `"hello"`, `"pergunta"` — ❌ não exercitam os guardrails do domínio
- Strings hardcoded no corpo do teste em vez de importar do fixture — ❌ duplicação e inconsistência

---

### Proibições gerais

- ❌ Testes com dependência de ordem de execução (um teste depende do estado deixado por outro)
- ❌ `console.log` em testes (usar `pino` se precisar de log em debug)
- ❌ Timeouts hardcoded sem justificativa (`await new Promise(r => setTimeout(r, 3000))`)
- ❌ Variáveis de ambiente reais (`process.env.AZURE_API_KEY`) — usar mocks ou `.env.test`
```

---

## Entregável 2: Teste reescrito — Antes e Depois

### ANTES (teste original gerado pelo Copilot sem guidance)

```typescript
// Teste gerado pelo Copilot sem guidance
test('query endpoint works', async () => {
  const result = await handler({ body: '{"question": "test"}' });
  expect(result).toBeDefined();
});
```

**Problemas identificados:**

| # | Problema | Por quê é ruim |
|---|----------|---------------|
| 1 | Nome genérico `'query endpoint works'` | Não descreve comportamento nem condição; dois QAs não sabem o que está sendo testado |
| 2 | Pergunta `"test"` como dado de teste | Não exercita nenhum guardrail do domínio NovaTech; o teste passa com qualquer resposta |
| 3 | Sem arrange/act/assert separados | Dificulta leitura e manutenção |
| 4 | `expect(result).toBeDefined()` | Passa mesmo que o handler retorne `null`, `{}`, ou um objeto de erro — não verifica comportamento |
| 5 | Input como string JSON raw | Não usa factory; frágil e não reutilizável |
| 6 | Sem mock de serviços externos | Se o Azure AI Search estiver offline, o teste falha por motivo errado |

---

### DEPOIS (teste reescrito seguindo Testing Standards)

```typescript
import { describe, it, expect, beforeAll, afterEach } from 'vitest'
import { setupServer } from 'msw/node'
import { handler } from '../../src/functions/query'
import { buildQueryRequest } from '../factories/query-request.factory'
import { chunks } from '../fixtures/rag/chunks'
import { mockAzureSearchResponse } from '../helpers/msw-handlers'

const server = setupServer()

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

describe('QueryHandler', () => {
  it('should return source_document in response when query matches indexed content', async () => {
    // arrange
    const request = buildQueryRequest({ question: 'Qual o SLA de resolução para cliente Gold?' })
    server.use(mockAzureSearchResponse([chunks.SLA_2024_B]))

    // act
    const response = await handler(request)

    // assert
    expect(response.status).toBe(200)
    expect(response.body.source_document).toBe('SLA-2024, seção 2')
    expect(response.body.answer).toMatch(/24h|24 horas/i)
  })
})
```

**Melhorias aplicadas:**

| # | Melhoria | Standards aplicado |
|---|----------|--------------------|
| 1 | Nome `'should return source_document in response when query matches indexed content'` | Nomenclatura: descreve comportamento + condição |
| 2 | Pergunta `'Qual o SLA de resolução para cliente Gold?'` | Fixture de domínio realista; exercita guardrail de source_document |
| 3 | Seções `// arrange`, `// act`, `// assert` separadas | Estrutura obrigatória |
| 4 | `expect(response.body.source_document).toBe('SLA-2024, seção 2')` | Assertion verifica conteúdo, não apenas existência |
| 5 | `buildQueryRequest()` | Factory substitui string JSON raw |
| 6 | `server.use(mockAzureSearchResponse(...))` com msw | Mock correto de serviço externo |

---

## Entregável 3: Critérios de Review de Testes Gerados por IA

Os critérios abaixo são objetivos: dois QAs analisando o mesmo teste devem chegar à mesma conclusão (aprovar ou reprovar).

### Critério 1 — Nome do teste descreve comportamento e condição

**Verificação:** O nome do bloco `it(...)` responde às perguntas "o que o sistema FAZ?" e "em qual situação?"

- ✅ `'should return explicit refusal when question involves dangerous cargo return'`
- ❌ `'query endpoint works'`, `'test handler'`, `'it works'`

**Regra:** Reprovar se o nome for genérico ou descrever apenas que "funciona", sem especificar comportamento e condição.

---

### Critério 2 — Assertions verificam conteúdo específico do comportamento

**Verificação:** As assertions checam valores concretos da resposta (campo obrigatório com valor esperado, texto que demonstra o comportamento, status HTTP), não apenas que "algo existe".

- ✅ `expect(response.body.source_document).toBe('POL-001, seção 3.2')`
- ✅ `expect(response.body.answer).toMatch(/não.*elegível.*devolução/i)`
- ❌ `expect(result).toBeDefined()`
- ❌ `expect(result.body).toBeTruthy()`

**Regra:** Reprovar se qualquer assertion puder passar com um objeto de erro ou com um body vazio `{}`.

---

### Critério 3 — Dados de teste são do domínio NovaTech (não genéricos)

**Verificação:** As perguntas, chunks e respostas esperadas usam terminologia real do projeto (cargas perigosas, SLA Gold, multiplicadores regionais, frete especial, etc.) importados de fixtures em `/tests/fixtures/rag/`.

- ✅ `question: 'Qual o multiplicador de frete especial para Manaus?'`
- ✅ Chunk importado de `chunks.PROC_042V2_B`
- ❌ `question: 'test'`, `question: 'hello'`, `body: '{"question": "pergunta"}'`

**Regra:** Reprovar se a pergunta não exercitar nenhum guardrail do domínio ou se os dados forem strings genéricas sem contexto de logística.

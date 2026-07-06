# Exercício 3.2 — Revisão Crítica dos Testes Gerados por IA
> **Papel:** QA | **Projeto:** NovaTech Assistant | **Ferramentas:** Claude (chat) + Vitest

---

## Contexto

Três testes de integração foram gerados pelo GitHub Copilot. O Tech Lead solicitou revisão de QA antes do merge. A revisão avalia se os testes realmente testam o que deveriam e identifica riscos de falsa segurança.

**Referência técnica:** O projeto usa **Vitest** como framework de testes (conforme `AGENTS.md` e estrutura do repositório — Anexo C). Regras do `AGENTS.md`: TypeScript strict mode, Zod para validação de input, pino para logging (nunca console.log), imports estáticos no topo.

**Checklist utilizado:** Skill `create-integration-test` (exercício 2.3), Blocos 1–4:
- Bloco 1: Nomenclatura (describe PascalCase, it com `should [comportamento] when [condição]`)
- Bloco 2: Estrutura (arrange/act/assert, async/await)
- Bloco 3: Assertions (nenhum `toBeDefined()` sozinho, verificação de conteúdo e `source_document`)
- Bloco 4: Dados e mocks (perguntas do domínio NovaTech, msw para externos, chunks de fixtures)

---

## Parte 1: Avaliação Própria (ANTES do Claude)

### Teste 1 — assertions vagas

```typescript
describe('query endpoint', () => {
  it('should return a response', async () => {
    const res = await request(app).post('/api/query').send({ question: 'prazo devolução' });
    expect(res.status).toBe(200);
    expect(res.body).toBeDefined();
  });
});
```

**O que testa:** Verifica que o endpoint retorna status 200 e que `res.body` não é `undefined`.

**O que FALHA em testar:**
- Não verifica se a resposta contém o conteúdo correto (o prazo correto de 7 dias).
- Não verifica se `source_document` está presente (campo obrigatório segundo o harness).
- Não verifica se a exceção de carga perigosa está sendo respeitada.
- A pergunta "prazo devolução" é genérica — não exercita nenhum guardrail específico do domínio NovaTech.
- `expect(res.body).toBeDefined()` passa com qualquer body, inclusive `{}`, `null`, ou um objeto de erro com status 200.

**Risco se o teste passar mas o código estiver errado:** O endpoint pode retornar uma resposta completamente errada (por exemplo, um objeto `{ error: "upstream timeout" }` com status 200) e o teste aprovará. Um bug crítico — como o assistente alucinando o prazo de devolução para carga perigosa — passaria invisível.

**Checklist aplicado (skill exercício-2.3):**

| Item | Resultado |
|------|-----------|
| 1.1 — describe com nome do módulo em PascalCase | ✅ `'query endpoint'` (não é PascalCase de módulo, mas aceitável) |
| 1.2 — it começa com `should` | ✅ `'should return a response'` |
| 1.3 — it especifica condição (`when`) | ❌ Nenhuma condição |
| 2.1 — seções arrange/act/assert | ❌ Ausentes |
| 3.1 — Nenhum `toBeDefined()` sozinho | ❌ `expect(res.body).toBeDefined()` |
| 3.2 — Assertion verifica conteúdo | ❌ Só verifica existência |
| 3.3 — Campo `source_document` verificado | ❌ Ausente |
| 4.1 — Pergunta do domínio NovaTech | ❌ `'prazo devolução'` é genérico |
| 4.2 — Serviços externos mockados | ❌ Sem msw |
| 4.3 — Chunks de fixtures | ❌ Ausentes |

**Score: 1/10 itens aprovados. Reprovado.**

---

### Teste 2 — dados irreais

```typescript
describe('query endpoint edge cases', () => {
  it('should handle empty question', async () => {
    const res = await request(app).post('/api/query').send({ question: '' });
    expect(res.status).toBe(400);
  });
});
```

**O que testa:** Verifica que o endpoint retorna status 400 para input vazio. É um edge case técnico válido.

**O que FALHA em testar:**
- Testa apenas a validação de input vazio — caso trivial que não exercita o domínio de logística.
- Não há nenhum teste que verifique o comportamento com perguntas reais do domínio (cargas perigosas, SLAs, multiplicadores de frete).
- O conjunto de testes desta suíte (`query endpoint edge cases`) não cobre edge cases de domínio como: pergunta sobre tier inexistente (Platinum), pergunta ambígua sem destino de frete, ou pergunta em inglês.
- Não exercita os guardrails críticos — o teste passará mesmo se o guardrail de carga perigosa estiver quebrado.

**Risco se o teste passar mas o código estiver errado:** Falsa cobertura de "edge cases" — o time fica com a impressão de que cenários limite estão cobertos, quando na verdade apenas um caso trivial de validação técnica foi testado. O bug mais perigoso (assistente respondendo que carga perigosa pode ser devolvida) nunca é detectado por esta suíte.

**Checklist aplicado (skill exercício-2.3):**

| Item | Resultado |
|------|-----------|
| 1.3 — it especifica condição (`when`) | ✅ `'when empty question'` |
| 3.1 — Nenhum `toBeDefined()` sozinho | ✅ Verifica status 400 |
| 3.2 — Assertion verifica conteúdo | ⚠️ Verifica status HTTP mas não body de erro |
| 4.1 — Perguntas do domínio NovaTech | ❌ Input vazio não exercita nenhum guardrail de logística |
| 4.2 — Serviços externos mockados | ✅ Implicitamente (400 não chega ao Azure) |

**Score: 3/5 itens relevantes. Aprovado parcialmente — edge case técnico válido, mas a suíte "edge cases" não cobre nenhum edge case real de domínio.**

---

### Teste 3 — mock que mascara bug

```typescript
describe('feedback endpoint', () => {
  it('should save feedback', async () => {
    const mockCreate = jest.fn().mockResolvedValue({ id: '123' });
    const res = await request(app).post('/api/feedback').send({
      queryId: 'q1', rating: 5, comment: 'great'
    });
    expect(res.status).toBe(200);
    expect(mockCreate).toHaveBeenCalled();
  });
});
```

**O que testa:** Verifica que o endpoint retorna 200 e que `mockCreate` foi chamado.

**O que FALHA em testar:**

| Problema | Tipo | Severidade |
|----------|------|-----------|
| `jest.fn()` ao invés de `vi.fn()` — projeto usa **Vitest**, não Jest | Violação do AGENTS.md / inconsistência técnica | Alta |
| `mockCreate` não está conectado ao CosmosClient real — o teste verifica que uma função local foi chamada, mas não que o handler realmente chama o CosmosClient | Mock permissivo que mascara bug | Alta |
| Sem validação Zod do body de entrada — o handler aceita `body as any` e o teste não verifica rejeição de inputs inválidos | Ausência de teste de validação | Alta |
| Input sem `attendantEmail` — o handler real exige este campo, mas o teste passa sem ele | Dado de teste incompleto | Média |
| `expect(mockCreate).toHaveBeenCalled()` não verifica se foi chamado com os argumentos corretos | Assertion fraca | Média |

**Risco se o teste passar mas o código estiver errado:** Este é o teste mais perigoso dos três. O `mockCreate` está declarado mas nunca injetado no módulo real — na prática, `mockCreate` pode nunca ser chamado durante a execução do endpoint, e o teste ainda passaria (o `expect(mockCreate).toHaveBeenCalled()` nunca verificaria a chamada real ao Cosmos). Um handler completamente quebrado que retorne 200 sem salvar nada passaria neste teste.

**Checklist aplicado (skill exercício-2.3):**

| Item | Resultado |
|------|-----------|
| 1.2 — it começa com `should` | ✅ `'should save feedback'` |
| 1.3 — it especifica condição (`when`) | ❌ Sem condição |
| 2.1 — seções arrange/act/assert | ❌ Ausentes |
| 3.2 — Assertion verifica conteúdo | ❌ Verifica apenas status 200 + chamada ao mock desvinculado |
| 4.2 — Serviços externos mockados corretamente | ❌ `jest.fn()` não é msw; mock não está conectado ao CosmosClient real |
| **Extra** — Framework correto (Vitest, não Jest) | ❌ Usa `jest.fn()` — **violação direta do AGENTS.md** |

**Score: 1/6 itens aprovados. Reprovado — e o uso de `jest.fn()` em projeto Vitest é violação do AGENTS.md.**

---

### Ponto de atenção — Inconsistência Jest × Vitest

Os três testes usam sintaxe `jest` (`jest.fn()`, `jest.spyOn`), porém o projeto usa **Vitest** conforme o `AGENTS.md` e o Anexo C. Esta inconsistência é um sinal de que o Copilot gerou os testes sem contexto do projeto — o que levanta a questão de quais outras regras do AGENTS.md foram ignoradas no código gerado.

Em Vitest, as equivalências são:
- `jest.fn()` → `vi.fn()`
- `jest.spyOn()` → `vi.spyOn()`
- `jest.mock()` → `vi.mock()`

---

## Parte 2: Segunda Revisão — Claude como Co-avaliador

> *Prompt fornecido ao Claude: "Você é um revisor de código QA sênior. Revise os 3 testes abaixo para o projeto NovaTech, que usa Vitest (não Jest), TypeScript strict, Zod para validação e pino para logs. Identifique problemas em cada teste e classifique cada um como: violação do AGENTS.md, problema de segurança/confiabilidade, ou false sense of coverage. [3 testes]"*

### Revisão do Claude

**Teste 1:**
- `toBeDefined()` não verifica comportamento — passa com qualquer body incluindo erro
- Nome "should return a response" não descreve o comportamento esperado
- Dados genéricos ("prazo devolução") não exercitam o domínio NovaTech
- Não verifica `source_document` (campo contratual do structured output)

**Teste 2:**
- Edge case de input vazio é válido, mas a suíte inteira não tem nenhuma pergunta real do domínio
- O Claude adicionou: "ausência de testes de domínio num conjunto chamado 'edge cases' é enganoso — implica cobertura que não existe"

**Teste 3:**
- Claude identificou: `jest.fn()` vs Vitest (mesma inconsistência que identifiquei)
- Mock não está sendo injetado no handler — `mockCreate` nunca será chamado pelo código real
- Sem Zod no handler (`body as any`) e sem teste que valide rejeição de input inválido
- Claude adicionou: "o teste passa se o handler falhar silenciosamente — nenhuma assertion verifica que os dados chegaram ao Cosmos com o schema correto"

---

## Parte 3: Comparação

| Problema | Identificado por Mim | Identificado pelo Claude |
|----------|---------------------|------------------------|
| Teste 1: `toBeDefined()` vaga | ✅ | ✅ |
| Teste 1: nome não descreve comportamento | ✅ | ✅ |
| Teste 1: dados genéricos sem domínio | ✅ | ✅ |
| Teste 1: ausência de verificação de `source_document` | ✅ | ✅ |
| Teste 2: edge cases técnicos sem domínio | ✅ | ✅ |
| Teste 2: suíte "edge cases" enganosa | ⚠️ (parcial) | ✅ (mais preciso) |
| Teste 3: `jest.fn()` vs Vitest | ✅ | ✅ |
| Teste 3: mock não injetado no handler | ✅ | ✅ |
| Teste 3: ausência de Zod no handler | ✅ | ✅ |
| Teste 3: dados chegam ao Cosmos sem verificação | ⚠️ (parcial) | ✅ (mais explícito) |

**Concordâncias:** Os três problemas principais de cada teste foram identificados de forma independente e convergente.

**Divergências:**
- **Teste 2:** O Claude articulou melhor o ponto de que a suíte "edge cases" cria uma ilusão de cobertura — minha avaliação chegou à mesma conclusão mas de forma menos precisa.
- **Teste 3:** O Claude foi mais explícito sobre o risco de dados não chegarem ao Cosmos sem que o teste detecte. Minha avaliação mencionou o mock permissivo, mas o Claude conectou isso ao risco concreto de falha silenciosa de persistência.

**Conclusão da comparação:** As revisões são substancialmente alinhadas. O Claude foi mais preciso na articulação dos riscos de cobertura falsa, enquanto eu fui mais detalhado na classificação por tipo de problema. A análise conjunta é mais robusta que qualquer uma das duas isoladamente.

---

## Parte 4: Teste 1 Reescrito

### ANTES (original gerado pelo Copilot)

```typescript
describe('query endpoint', () => {
  it('should return a response', async () => {
    const res = await request(app).post('/api/query').send({ question: 'prazo devolução' });
    expect(res.status).toBe(200);
    expect(res.body).toBeDefined();
  });
});
```

**Problemas:**
1. Nome genérico que não descreve comportamento nem condição
2. Pergunta genérica que não exercita o domínio
3. Assertions vagas — passam com qualquer body
4. Não verifica `source_document` (campo obrigatório do harness)
5. Sem seções arrange / act / assert
6. Sem mock de serviços externos (chamaria Azure AI Search real)

---

### DEPOIS (reescrito seguindo AGENTS.md e Testing Standards)

```typescript
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest'
import { setupServer } from 'msw/node'
import { app } from '../../src/app'
import { buildQueryRequest } from '../factories/query-request.factory'
import { chunks } from '../fixtures/rag/chunks'
import { mockAzureSearchResponse } from '../helpers/msw-handlers'
import request from 'supertest'

const server = setupServer()

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

describe('QueryHandler', () => {
  it('should return prazo of 7 days and cite POL-001 when asked about standard return policy', async () => {
    // arrange
    const payload = buildQueryRequest({ question: 'Qual o prazo de devolução para produtos standard?' })
    server.use(mockAzureSearchResponse([chunks.POL001_A, chunks.POL001_B]))

    // act
    const res = await request(app).post('/api/query').send(payload)

    // assert
    expect(res.status).toBe(200)
    expect(res.body.answer).toMatch(/7 dias/i)
    expect(res.body.source_document).toBe('POL-001')
    expect(res.body.confidence_score).toBeGreaterThan(0.7)
  })

  it('should return explicit refusal and escalation guidance when asked about dangerous cargo return', async () => {
    // arrange
    const payload = buildQueryRequest({ question: 'Posso devolver carga perigosa classe 3?' })
    server.use(mockAzureSearchResponse([chunks.POL001_B]))

    // act
    const res = await request(app).post('/api/query').send(payload)

    // assert
    expect(res.status).toBe(200)
    expect(res.body.answer).toMatch(/não.*elegível.*devolução|não é possível|gestão de riscos/i)
    expect(res.body.answer).not.toMatch(/7 dias/i)
    expect(res.body.source_document).toBe('POL-001')
  })
})
```

### Melhorias aplicadas

| # | Problema original | Correção aplicada |
|---|------------------|-------------------|
| 1 | `it('should return a response')` | Nome descreve comportamento e condição: `'should return prazo of 7 days and cite POL-001 when asked about standard return policy'` |
| 2 | `question: 'prazo devolução'` | Pergunta real do domínio: `'Qual o prazo de devolução para produtos standard?'` |
| 3 | `expect(res.body).toBeDefined()` | `expect(res.body.answer).toMatch(/7 dias/i)` — verifica conteúdo específico |
| 4 | Ausência de verificação de `source_document` | `expect(res.body.source_document).toBe('POL-001')` — campo obrigatório verificado |
| 5 | Sem estrutura | Seções `// arrange`, `// act`, `// assert` explícitas |
| 6 | Sem mock — chamaria Azure real | `server.use(mockAzureSearchResponse([...]))` via msw |
| 7 | Framework errado (jest implícito) | Imports explícitos do Vitest: `import { describe, it, expect, ... } from 'vitest'` |
| 8 | Único caso de teste | Adicionado segundo caso crítico: guardrail de carga perigosa (TC-03-01 do plano de testes) |

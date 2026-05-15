# PricingPro — Especificação Técnica v1.1
> Documento destinado ao time de tecnologia para construção do sistema de precificação dentro da infraestrutura corporativa.

---

## 1. Visão Geral

O **PricingPro** é uma aplicação de cálculo de precificação para proposta comercial de serviços de viagens corporativas. O vendedor insere dados do cliente, e o sistema retorna os valores mínimos de FEE necessários para cobrir todos os custos operacionais, tributários e garantir uma margem mínima de lucro.

A versão atual é uma SPA (Single Page Application) em HTML/JS puro, hospedada no GitHub Pages. A versão corporativa deve ser uma aplicação web full-stack com autenticação, persistência de propostas e integrações futuras.

---

## 2. Regras de Negócio

### 2.1 Tipos de Transação (Produtos)

| Código   | Descrição              | Canal Online | Perfil Consultor | Capacidade/Consultor |
|----------|------------------------|:------------:|:----------------:|:--------------------:|
| AIRDOM   | Aéreo Doméstico        | ✅           | Junior           | 500 tx/mês           |
| AIRINT   | Aéreo Internacional    | ✅           | Senior           | 250 tx/mês           |
| HOTDOM   | Hotel Doméstico        | ✅           | Pleno            | 280 tx/mês           |
| HOTINT   | Hotel Internacional    | ✅           | Senior           | 250 tx/mês           |
| CARDOM   | Carro Doméstico        | ✅           | Junior           | 500 tx/mês           |
| CARINT   | Carro Internacional    | ✅           | Senior           | 250 tx/mês           |
| RODOV    | Rodoviário             | ❌           | Junior           | 500 tx/mês           |
| VIP      | Viagens VIP            | ❌           | Senior           | 100 tx/mês           |

**Observação importante:** Transações online (canal digital/OBT) não requerem consultor. O cálculo de consultores aplica-se apenas às transações offline.

### 2.2 Cálculo de Consultores

```
transacoes_offline = total_transacoes × (1 - percentual_adocao_online / 100)
consultores_necessarios = transacoes_offline / capacidade_por_consultor
```

> ⚠️ O número de consultores é **decimal** (não arredondado). Ex: 400 tx / 500 capacidade = **0,80 consultor**. O custo é proporcional.

### 2.3 Custo com Pessoal

| Perfil  | Salário Base |
|---------|:------------:|
| Junior  | R$ 2.500,00  |
| Pleno   | R$ 3.200,00  |
| Senior  | R$ 5.000,00  |

**Encargos trabalhistas:** 75% sobre o salário base (INSS patronal, FGTS, férias, 13º, benefícios).

```
custo_total_consultor = salario_base × consultores × 1,75
```

### 2.4 Custo Transacional

Toda transação — online ou offline — gera um custo operacional de **R$ 3,10** (sistemas, GDS, infraestrutura).

```
custo_transacional = total_transacoes × R$ 3,10
```

### 2.5 Subtotal Operacional

```
subtotal_operacional = custo_total_pessoal + custo_transacional
```

### 2.6 Tributos

| Tributo     | Alíquota |
|-------------|:--------:|
| ISS         | 5,00%    |
| PIS + COFINS| 3,65%    |
| **Total**   | **8,65%**|

> ⚠️ **Regra crítica:** ISS, PIS e COFINS incidem **exclusivamente sobre a receita de FEE**. Comissões recebidas de fornecedores (hotel, carro, aéreo) **não** são tributadas por esses impostos.

### 2.7 Receita Bruta Mínima

A receita bruta deve cobrir o subtotal operacional, os tributos e ainda garantir a margem mínima de **1,5%**.

```
receita_bruta_minima = subtotal_operacional / (1 - 0,0865 - 0,015)
                     = subtotal_operacional / 0,8985
```

Derivando os componentes:
```
valor_ISS      = receita_bruta_minima × 0,05
valor_PIS_COF  = receita_bruta_minima × 0,0365
valor_margem   = receita_bruta_minima × 0,015
```

---

## 3. Forma de Pagamento (FOP)

A FOP (Forma de Pagamento) impacta o custo de capital quando a agência oferece crédito ao cliente.

### 3.1 Cartão de Crédito do Cliente
Sem custo adicional. Nenhum multiplicador aplicado.

### 3.2 Faturado — Tabela de Float

O custo de capital é calculado com **juros compostos** sobre o volume financeiro total faturado (`volume_financeiro_total`) para o período de float de cada condição.

| Condição de Faturamento                       | Float Aprox. | Tier | Taxa/mês |
|-----------------------------------------------|:------------:|:----:|:--------:|
| Faturamento diário + 10 dias corridos         | ~11 dias     | 1    | 1,5%     |
| Faturamento semanal + 5 dias corridos         | ~12 dias     | 1    | 1,5%     |
| Faturamento semanal + 7 dias corridos         | ~14 dias     | 1    | 1,5%     |
| Faturamento a cada 10 dias + 10 dias corridos | ~20 dias     | 2    | 2,5%     |
| Faturamento a cada 15 dias + 15 dias corridos | ~30 dias     | 3    | 3,0%     |
| Acima de 30 dias                              | > 30 dias    | —    | ❌ Bloqueado |

**Tier 3 (21–30 dias):** exige autorização da Diretoria Comercial. O custo extra de 3% é cobrado separadamente do FEE ao cliente, não embutido.

### 3.3 Aplicação do Custo de Capital FOP

O custo de capital é calculado com juros compostos sobre o volume financeiro e adicionado à receita mínima necessária:

```
custo_capital_fop = volume_financeiro_total × ((1 + taxa_mensal)^(float_dias / 30) - 1)

receita_com_fop   = receita_bruta_minima + custo_capital_fop
fee_necessario    = max(0, receita_com_fop - receita_vip_fixa)
```

> Exemplo: volume R$ 500.000, float 14 dias, taxa 1,5%/mês →
> custo_capital = 500.000 × ((1,015)^(14/30) − 1) ≈ R$ 3.487,00

---

## 4. Modelos de Precificação (FEE)

### 4.1 VIP — Sempre Fixo

O FEE VIP é **R$ 50,00 por transação**, imutável, independente do modelo escolhido.

```
receita_vip = quantidade_transacoes_vip × R$ 50,00
```

### 4.2 Transaction FEE (único)

```
fee_por_transacao = receita_necessaria_via_fee / total_transacoes_nao_vip
```

### 4.3 Management FEE

```
management_fee_pct = (receita_necessaria_via_fee / volume_financeiro_nao_vip) × 100
```

### 4.4 Flat FEE (Mensalidade)

```
flat_fee_mensal = receita_necessaria_via_fee
```

### 4.5 Comissionamento (sem FEE ao cliente)

| Produto          | Base de Cálculo                                      |
|------------------|------------------------------------------------------|
| AIRDOM           | max(R$ 40,00 × nº bilhetes ; 10% do volume financeiro) |
| AIRINT           | 6% do volume financeiro                              |
| HOTDOM, HOTINT   | 10% do volume financeiro                             |
| CARDOM, CARINT   | 10% do volume financeiro                             |
| RODOV            | 10% do volume financeiro                             |
| VIP              | R$ 50,00 por transação                               |

**Validação:** `total_comissoes >= subtotal_operacional` para o modelo ser viável.

### 4.6 Modelo Misto

Combinação de FEE para aéreo + comissão para terrestre (hotel e carro).

```
comissao_terrestre   = Σ (10% × volume_financeiro) para [HOTDOM, HOTINT, CARDOM, CARINT]
fee_aereo_necessario = max(0, receita_com_fop - receita_vip - comissao_terrestre)
```

O FEE aéreo pode ser expresso em qualquer formato (Transaction, Management ou Flat), aplicado apenas sobre AIRDOM + AIRINT + RODOV.

> ⚠️ **Regra obrigatória:** Mesmo quando `fee_aereo_necessario ≤ 0` (comissão terrestre + VIP já cobrem todos os custos), o sistema **deve sugerir obrigatoriamente um FEE mínimo de R$ 5,00/tx** para o aéreo. Nunca sugerir FEE zero para aéreo no modelo misto.

### 4.7 Transaction FEE por Produto (Categorias)

Permite definir FEEs distintos por categoria:

| Categoria       | Composição                                                      |
|-----------------|-----------------------------------------------------------------|
| DOM Online      | Transações online de AIRDOM + HOTDOM + CARDOM                  |
| DOM Offline     | Transações offline de AIRDOM + HOTDOM + CARDOM + RODOV (total) |
| Internacional   | Todas as transações de AIRINT + HOTINT + CARINT                |
| VIP             | Fixo R$ 50,00                                                   |

**FEE mínimo por categoria** (cada categoria cobre seus próprios custos):
```
fee_minimo_categoria = (custo_categoria / (1 - 0,0865 - 0,015)) / transacoes_categoria
```

> ⚠️ **Restrições de sugestão para DOM Online:**
> - FEE mínimo sugerido: **R$ 4,50/tx** (nunca sugerir abaixo desse valor)
> - FEE máximo de referência: **R$ 9,10/tx**
> - O FEE DOM Online sugerido deve sempre ser **menor que o DOM Offline** (online é mais barato por não requerer consultor)

---

## 5. Incentivos de Companhias Aéreas

Os incentivos **não** compõem a receita de precificação — são exibidos separadamente como informação estratégica.

| Tipo          | Alíquota sobre volume financeiro |
|---------------|:--------------------------------:|
| Doméstico     | 3,5%                             |
| Internacional | 1,5%                             |

---

## 6. Modelo de Dados Recomendado

### Entidades principais

```
Cliente
  - id
  - razao_social
  - cnpj
  - segmento
  - criado_em

Proposta
  - id
  - cliente_id
  - vendedor_id
  - status (rascunho | enviada | aprovada | recusada)
  - modelo_fee (transaction | management | flat | commission | misto | perprod)
  - fop_tipo (credit | invoice)
  - fop_opcao (d10 | s5 | s7 | t10 | q15)
  - criado_em
  - atualizado_em

TransacaoProposta
  - id
  - proposta_id
  - tipo (AIRDOM | AIRINT | HOTDOM | HOTINT | CARDOM | CARINT | RODOV | VIP)
  - quantidade_mensal
  - volume_financeiro
  - percentual_adocao_online

ResultadoProposta (snapshot calculado)
  - proposta_id
  - custo_salarios_base
  - custo_encargos
  - custo_transacional
  - subtotal_operacional
  - receita_bruta_minima
  - valor_iss
  - valor_pis_cofins
  - valor_margem
  - receita_necessaria_fee
  - transaction_fee_sugerido
  - management_fee_sugerido_pct
  - flat_fee_sugerido
  - incentivo_airdom
  - incentivo_airint
  - calculado_em

Vendedor
  - id
  - nome
  - email
  - perfil (vendedor | gerente | diretor)
```

---

## 7. Arquitetura Recomendada para Versão Corporativa

### Stack sugerida

| Camada       | Opção recomendada                        | Alternativa              |
|--------------|------------------------------------------|--------------------------|
| Frontend     | React + TypeScript + Vite                | Vue.js + TypeScript      |
| UI Library   | shadcn/ui ou Ant Design                  | Material UI              |
| Backend      | Node.js + Fastify ou Python + FastAPI    | Java + Spring Boot       |
| Banco        | PostgreSQL                               | SQL Server               |
| Auth         | SSO corporativo (Azure AD / Okta) + JWT  | Cognito                  |
| Infra        | Docker + Kubernetes                      | ECS / Cloud Run          |
| PDF Export   | Puppeteer ou WeasyPrint                  | PDFKit                   |

### Diagrama de arquitetura (simplificado)

```
[Browser]
    │
    ▼
[Frontend SPA]  ──────►  [API REST / GraphQL]
                                │
                    ┌───────────┼───────────────┐
                    ▼           ▼               ▼
               [Banco DB]   [Auth/SSO]    [Storage PDF]
```

### Endpoints essenciais da API

```
POST   /propostas                    — criar proposta
GET    /propostas/:id                — buscar proposta
PUT    /propostas/:id                — atualizar proposta
POST   /propostas/:id/calcular       — executar cálculo
POST   /propostas/:id/exportar-pdf   — gerar PDF da proposta
GET    /propostas?vendedor=&status=  — listar propostas
GET    /clientes                     — listar clientes
POST   /clientes                     — cadastrar cliente
PUT    /parametros/custos            — atualizar tabela de custos (admin)
```

---

## 8. Funcionalidades para a Versão Corporativa

### Obrigatórias (MVP)
- [ ] Autenticação via SSO corporativo
- [ ] Histórico de propostas por vendedor e por cliente
- [ ] Export PDF da proposta formatada
- [ ] Aprovação de propostas Tier 3 (FOP 21–30 dias) via workflow
- [ ] Painel de parâmetros configuráveis (salários, capacidade, impostos, incentivos) sem deploy

### Desejáveis (v2)
- [ ] Comparativo side-by-side de modelos de FEE
- [ ] Dashboard de rentabilidade da carteira
- [ ] Versionamento de propostas (histórico de revisões)
- [ ] Integração com CRM (Salesforce / HubSpot)
- [ ] Notificações de aprovação por e-mail / Teams

### Futuras (v3)
- [ ] Simulação de cenários (what-if)
- [ ] Análise de rentabilidade pós-implantação (real vs. proposto)
- [ ] Integração com sistemas de backoffice (dados reais de transações)

---

## 9. Parâmetros Configuráveis

Todos os parâmetros abaixo devem ser gerenciáveis pela área de negócio via painel administrativo (sem precisar de desenvolvimento):

| Parâmetro                              | Valor Atual |
|----------------------------------------|:-----------:|
| Capacidade Junior (tx/mês)             | 500         |
| Capacidade Pleno (tx/mês)              | 280         |
| Capacidade Senior (tx/mês)             | 250         |
| Capacidade VIP (tx/mês)               | 100         |
| Salário Junior                         | R$ 2.500    |
| Salário Pleno                          | R$ 3.200    |
| Salário Senior                         | R$ 5.000    |
| Encargos trabalhistas                  | 75%         |
| Custo por transação                    | R$ 3,10     |
| ISS (incide só sobre FEE)              | 5,00%       |
| PIS + COFINS (incide só sobre FEE)     | 3,65%       |
| Margem mínima                          | 1,50%       |
| Incentivo aéreo doméstico              | 3,50%       |
| Incentivo aéreo internacional          | 1,50%       |
| FEE fixo VIP                           | R$ 50,00    |
| Float máximo permitido                 | 30 dias     |
| FOP Tier 1 (taxa/mês — juros compostos)| 1,5%        |
| FOP Tier 2 (taxa/mês — juros compostos)| 2,5%        |
| FOP Tier 3 (taxa/mês — juros compostos)| 3,0%        |
| FEE mínimo aéreo no modelo misto       | R$ 5,00/tx  |
| FEE mínimo DOM Online (FEE/Produto)    | R$ 4,50/tx  |
| Comissão AIRDOM (R$/bilhete mínimo)    | R$ 40,00    |
| Comissão AIRDOM (% alternativo)        | 10%         |
| Comissão AIRINT                        | 6%          |
| Comissão Hotel / Carro                 | 10%         |

---

## 10. Referência da Versão Atual (v1.0)

- **Tecnologia:** HTML5 + CSS3 + JavaScript (Vanilla, sem dependências)
- **Repositório:** https://github.com/vinicius-goncalves-code/pricing-tool
- **URL de produção:** https://vinicius-goncalves-code.github.io/pricing-tool/
- **Arquivo principal:** `index.html` (auto-contido, ~1.200 linhas)
- **Deploy:** GitHub Pages (branch `main`, raiz `/`)
- **Atualização:** `git push` na branch `main` → deploy automático em ~60s

# PricingPro — Contexto para o Claude

Este arquivo descreve o projeto para que o Claude possa fazer alterações futuras com precisão, sem precisar reler o código inteiro toda vez.

---

## O que é este projeto

Calculadora de precificação para proposta comercial de uma agência de viagens corporativas. Roda no browser (sem backend), construída em HTML/CSS/JS puro em um único arquivo: `index.html`.

- **Repositório:** https://github.com/vinicius-goncalves-code/pricing-tool
- **URL ao vivo:** https://vinicius-goncalves-code.github.io/pricing-tool/
- **Arquivo único:** `/Users/viniciusgoncalves/agencia-pricing/index.html`
- **Deploy:** `git add index.html && git commit -m "..." && git push` → live em ~60s

---

## Estrutura do index.html

O arquivo tem três blocos principais:

1. **`<style>`** — CSS com variáveis CSS (`:root`) para cores e espaçamentos
2. **`<body>`** — 4 seções (steps) com IDs `section-1` a `section-4`, navegadas via `goStep(n)`
3. **`<script>`** — toda a lógica em JS vanilla, organizada nas seções abaixo:

### Seções do script

| Seção                  | O que faz                                                                 |
|------------------------|---------------------------------------------------------------------------|
| `CONSTANTS`            | `TYPES[]`, `FOP_OPTIONS[]`, `ISS`, `PIS_COF`, `TAX`, `MIN_MGMT`, etc.   |
| `STATE`                | objeto `state` com `fopType`, `fopOption`, `feeModel`, `customFees`       |
| `HELPERS`              | `fmt()`, `fmtN()`, `fmtTx()`, `fmtCons()`, `getVal()`                    |
| `BUILD TRANSACTION TABLE` | `buildTxTable()` — monta a tabela do Step 1 dinamicamente              |
| `FOP`                  | `buildFOPOptions()`, `selectFOP()`, `updateFOP()`, `updateFOPSummary()`  |
| `CALCULATION ENGINE`   | `calculate()` — toda a matemática do negócio, retorna objeto `r`          |
| `STEP 3`               | `updateModel()`, `updateFPP()`, `renderFPPTable()`, `renderMistoPreview()`, `updatePreview()` |
| `RESULT — STEP 4`      | `buildResult()` — monta o HTML completo do resultado                      |
| `NAVIGATION`           | `goStep(n)` — troca de step                                               |
| `RESET / INIT`         | `resetAll()`, `DOMContentLoaded`                                          |

---

## Constantes críticas (onde mudar valores de negócio)

Todas ficam no topo do `<script>`, na seção `CONSTANTS`:

```js
// Tabela de produtos — para adicionar/remover produto, editar este array
const TYPES = [
  { id:'AIRDOM', label:'Aéreo Doméstico', badge:'dom',
    onlineCapable:true, consultantType:'junior', capacity:500, salary:2800 },
  // ... outros produtos
];

const ISS       = 0.05;      // 5%
const PIS_COF   = 0.0365;    // 3,65%
const TAX       = ISS + PIS_COF;
const MIN_MGMT  = 0.015;     // margem mínima 1,5%
const TX_COST   = 1.50;      // custo por transação R$ 1,50
const VIP_FEE   = 50;        // FEE VIP fixo R$ 50
const LABOR_BURDEN = 0.75;   // encargos trabalhistas 75%
```

**Salários:** definidos dentro de `TYPES[].salary` por produto (ligado ao `consultantType`).

---

## Lógica de cálculo — função `calculate()`

A função lê os inputs do DOM, calcula tudo e retorna um objeto `r` com todos os valores. Os steps 3 e 4 apenas consomem esse objeto `r`.

### Fluxo principal

```
1. Para cada produto:
   onlineTx  = count × (online% / 100)
   offlineTx = count - onlineTx
   consultores = offlineTx / capacity          ← DECIMAL, sem Math.ceil
   salaryBase  = consultores × salary
   laborBurden = salaryBase × 0,75
   personnelTotal = salaryBase × 1,75

2. totalPersonnel  = Σ personnelTotal
   transactionCost = totalTx × R$ 1,50
   subtotalCost    = totalPersonnel + transactionCost

3. minGross = subtotalCost / (1 - 0,0865 - 0,015)
   taxISS   = minGross × 0,05
   taxPisCof= minGross × 0,0365
   marginAmt= minGross × 0,015

4. Aplicar FOP:
   minGrossFOP = minGross × fopMult

5. vipRevenue = VIP.count × 50
   feeRevNeeded = max(0, minGrossFOP - vipRevenue)

6. transactionFee = feeRevNeeded / nonVipTx
   managementFee  = (feeRevNeeded / nonVipFin) × 100
   flatFee        = feeRevNeeded
```

### Modelo Misto (campos do objeto `r`)

```
mistoTerrestrialCom = Σ (0,10 × fin) para [HOTDOM, HOTINT, CARDOM, CARINT]
mistoFeeNeeded      = max(0, minGrossFOP - vipRevenue - mistoTerrestrialCom)
mistoTransFee       = mistoFeeNeeded / (AIRDOM + AIRINT + RODOV tx)
mistoMgmtFee        = mistoFeeNeeded / (AIRDOM + AIRINT + RODOV fin) × 100
mistoFlatFee        = mistoFeeNeeded
```

### FEE por Produto (campos do objeto `r`)

```
domOnlineTx   = online tx de AIRDOM + HOTDOM + CARDOM
domOfflineTx  = offline tx de AIRDOM + HOTDOM + CARDOM + RODOV (total)
intTx         = total tx de AIRINT + HOTINT + CARINT

// Custos por categoria (sem impostos — só custo base):
domOnlineCost  = domOnlineTx × R$ 1,50           (online = sem consultor)
domOfflineCost = personnel(offline doméstico) + offlineTx × R$ 1,50
intCost        = personnel(internacional) + intTx × R$ 1,50

// FEE mínimo por categoria:
fppDomOnlineSug  = (domOnlineCost  / 0,8985) / domOnlineTx
fppDomOfflineSug = (domOfflineCost / 0,8985) / domOfflineTx
fppIntSug        = (intCost        / 0,8985) / intTx
```

---

## Modelos de FEE disponíveis (`state.feeModel`)

| Valor         | Descrição                                              |
|---------------|--------------------------------------------------------|
| `transaction` | Transaction FEE único para todos os produtos (exceto VIP) |
| `management`  | % sobre o volume financeiro total                       |
| `flat`        | Mensalidade fixa                                        |
| `commission`  | Comissionamento via fornecedores (sem FEE ao cliente)  |
| `misto`       | FEE aéreo + comissão terrestre (hotel/carro)           |
| `perprod`     | Transaction FEE por categoria (DOM Online/Offline/INT) |

---

## FOP — Forma de Pagamento

| `state.fopType` | `state.fopOption` | `fopMult` | Tier |
|-----------------|-------------------|:---------:|:----:|
| `credit`        | —                 | 1,000     | 0    |
| `invoice`       | `d10`             | 1,015     | 1    |
| `invoice`       | `s5`              | 1,015     | 1    |
| `invoice`       | `s7`              | 1,015     | 1    |
| `invoice`       | `t10`             | 1,025     | 2    |
| `invoice`       | `q15`             | 1,030     | 3    |

Tier 3 exibe alerta de autorização da Diretoria Comercial.

---

## Como fazer alterações comuns

### Adicionar um novo produto
1. Adicionar entrada em `TYPES[]` com todos os campos
2. Se for online-capable, garantir que aparece nas categorias de FEE por Produto em `calculate()` (seções `domOnlineCost`, `domOfflineCost` ou `intCost`)
3. Adicionar regra de comissão no loop `comByType` dentro de `calculate()`

### Mudar salário ou capacidade de um consultor
Editar `salary` ou `capacity` na entrada correspondente em `TYPES[]`.

### Mudar alíquota de imposto
Editar `ISS` ou `PIS_COF` nas constantes.

### Mudar margem mínima
Editar `MIN_MGMT`.

### Mudar custo por transação
Editar `TX_COST`.

### Mudar encargos trabalhistas
Editar `LABOR_BURDEN`.

### Adicionar nova opção de FOP
Adicionar entrada em `FOP_OPTIONS[]` com `{ id, label, float, tier }`.

### Mudar o FEE fixo do VIP
Editar `VIP_FEE`.

### Adicionar campo ao formulário do cliente (Step 1)
Adicionar `<input>` no bloco de identificação e ler o valor em `buildResult()` com `document.getElementById('novoId').value`.

### Mudar a aparência (cores, fontes)
Editar as variáveis CSS em `:root { ... }` no `<style>`.

---

## Publicar uma nova versão

```bash
cd ~/agencia-pricing
# ... faça as alterações no index.html ...
git add index.html
git commit -m "feat: descrição da mudança"
git push
# O site atualiza em ~60 segundos automaticamente
```

---

## O que NÃO mudar sem entender

- **A função `calculate()`** é o coração do sistema. Qualquer alteração de fórmula deve ser validada com um caso de teste manual antes de publicar.
- **`VIP_FEE = 50`** é uma regra de negócio imutável (definida pela diretoria comercial). Não expor como configurável ao vendedor.
- **O objeto `r`** retornado por `calculate()` é consumido por `updatePreview()` e `buildResult()`. Se adicionar novos campos calculados, basta incluir no `return` de `calculate()`.

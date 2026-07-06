# Fraud Detection System — CNP Transactions

Análise de fraude em ambiente de cartão não-presente (CNP) com motor de regras determinísticas e modelo de machine learning. 

---

## Resultado

| Métrica | Valor |
|---|---|
| Fraudes capturadas | 321 / 391 (82.1% recall) |
| Precisão dos bloqueios | 84.7% |
| Valor de fraude bloqueado | R$ 488.411 de R$ 568.346 (86%) |
| Falsos positivos | 58 transações (2.1%) |
| Fila de revisão humana | 54 transações |

---

## Contexto

Em adquirência, cada fraude não detectada vira chargeback, perda financeira direta para o sub-adquirente.
O desafio não é só detectar fraude. É detectar antes que o chargeback aconteça, em milissegundos, sem bloquear clientes legítimos.

Este projeto analisa **3.199 transações CNP reais** cobrindo novembro de 2019, com taxa de fraude de 12.2%.

---

## Estrutura do Projeto

```
fraud-detection-cnp/
│
├── notebook/
│   └── fraud_analysis.ipynb       # Análise completa
│
├── src/
│   └── antifraud.py               # Sistema anti-fraude em produção
│
├── data/
│   └── transactional-sample.csv   # Dataset (3.199 transações CNP)
│
└── README.md
```

---

## O que foi feito

### 1. Análise Exploratória
- Distribuição de valores: fraudes concentradas em tickets mais altos
- Padrão temporal: taxa de fraude desproporcional entre 3h-6h da manhã
- Identificação de 830 transações sem device fingerprint (25.9% do total)

### 2. Seis Padrões de Fraude Identificados

**Velocity Attack**
Mesmo usuário realizando múltiplas transações em menos de 30 minutos.
Top ofensores com 83-100% de taxa de chargeback.

**Card Testing**
Micro-transações (< R$10) em sequência para validar cartões roubados antes de compras maiores.
Achado contra-intuitivo: CBK rate de 1.9% nas próprias micro-transações o chargeback aparece na transação grande que vem depois.

**Multi-Card Device**
Mesmo dispositivo utilizado com 4+ cartões distintos em 7 dias.
Sinal mais forte identificado: dispositivos com 3+ cartões têm 68.3% de CBK rate vs 12.2% de baseline.
Device 563499: 22 cartões distintos, 19 chargebacks.

**Merchant Concentrado**
Merchants com taxa de chargeback acima de 50%.
Merchant 1308: R$ 34.517 em chargebacks, 100% CBK rate, um indicativo de merchant comprometido ou conivente.

**Near-Duplicates**
Mesmo usuário + merchant, intervalo < 5 minutos, valores com diferença < R$100.
Taxa de CBK nesse grupo: 41.9% vs 12.2% geral.

**Ausência de Device ID**
Indica uso de emuladores, scripts automatizados ou ferramentas de carding.
Tratado como sinal de risco incremental.

### 3. Feature Engineering

| Feature | O que captura |
|---|---|
| `txns_user_30m` | Velocity — transações recentes do usuário |
| `user_cbk_rate` | Histórico de fraude do usuário (expanding mean, sem data leakage) |
| `cards_on_device_7d` | Multi-card device — cartões únicos no device nos últimos 7 dias |
| `merch_cbk_rate` | Histórico de fraude do merchant (expanding mean, sem data leakage) |
| `amount_zscore` | Desvio do valor em relação ao padrão do usuário |
| `no_device` | Flag binário — ausência de device fingerprint |
| `is_micro` | Flag binário — micro-transação < R$10 |
| `odd_hour` | Flag binário — horário suspeito (3h-7h) |

### 4. Modelos Comparados

**Random Forest** (200 estimators, class_weight='balanced')
- Precision: 82.5% | Recall: 66.7% | F1: 73.8%

**XGBoost** (300 estimators, lr=0.05, scale_pos_weight balanceado)
- Precision: 71.8% | Recall: 78.2% | F1: 74.8%

**Escolha: XGBoost**

Em detecção de fraude, o custo de deixar uma fraude passar (chargeback = perda direta) é maior que o custo de bloquear um cliente legítimo (vai para revisão humana antes de qualquer bloqueio definitivo). XGBoost captura mais fraude (61 vs 52 no conjunto de teste) ao custo de mais falsos positivos que são tratados na fila de revisão, não em bloqueios automáticos.

### 5. Sistema Anti-Fraude em Duas Camadas

**Camada 1 — Regras Determinísticas** (< 1ms)

| Regra | Condição | Score |
|---|---|---|
| R0 | Usuário ou device com chargeback histórico | BLOCK 100 |
| R1 | 3+ transações do mesmo usuário em 30min | BLOCK 90 |
| R2 | 2+ micro-transações (< R$10) em 1h | BLOCK 90 |
| R3 | 4+ cartões no mesmo device em 7 dias | BLOCK 85 |
| R4 | Transação duplicada no mesmo merchant em 5min | BLOCK 80 |

**Camada 2 — XGBoost Score** (< 100ms)

```
Score ≥ 60  →  BLOCK  (bloqueio automático)
Score 30-59 →  REVIEW (fila de analista humano)
Score < 30  →  APPROVE
```

### 6. Backtest Cronológico

Simulação com isolamento temporal estrito: cada transação avaliada usando apenas histórico anterior a ela, sem data leakage.

```
╔══════════════════════════════════════════════════╗
║           RESULTADO FINAL DO SISTEMA             ║
╠══════════════════════════════════════════════════╣
║  Fraudes capturadas  :   321 / 391               ║
║  Fraudes aprovadas   :    58  (passaram)          ║
║  Legítimas bloqueadas:    58  (falsos positivos)  ║
║  Fila de revisão     :    54  (analista humano)   ║
╠══════════════════════════════════════════════════╣
║  Precisão   : 84.7%                              ║
║  Recall     : 82.1%                              ║
╠══════════════════════════════════════════════════╣
║  Fraude total         : R$  568.346,62           ║
║  Fraude bloqueada     : R$  488.411,44  (86%)    ║
║  Fraude não capturada : R$   57.304,62           ║
╚══════════════════════════════════════════════════╝
```

---

## Limitações e Próximos Passos

**Cold Start**
Primeiras transações de novos usuários têm features históricas zeradas.
Solução: adicionar features de contexto independentes de histórico como geolocalização por IP, tipo de BIN, resultado do 3DS.

**Concept Drift**
Fraudadores adaptam o comportamento ao longo do tempo.
Solução: monitoramento contínuo da distribuição de scores + retreinamento periódico com dados recentes.

**Dados ausentes**
O dataset não contém IP, resultado de autenticação 3DS, ou geolocalização.
Com essas informações, a performance do modelo seria significativamente maior.

**Escalabilidade**
O cálculo de `cards_on_device_7d` usa um loop por linha, funciona para análise offline mas não para produção.
Em produção: query indexada em banco de dados ou stream aggregation (Kafka/Flink) com estado por device.

---

## Stack

- Python 3.x
- Pandas, NumPy
- Matplotlib, Seaborn
- Scikit-learn (Random Forest, métricas)
- XGBoost

---

## Como Rodar

```bash
git clone https://github.com/seu-usuario/fraud-detection-cnp
cd fraud-detection-cnp
pip install -r requirements.txt
jupyter notebook notebook/fraud_analysis.ipynb
```

---

## Sobre

Projeto desenvolvido como parte de um processo seletivo para Risk Analyst em fintech de adquirência.
O dataset contém transações em ambiente CNP (card-not-present) com informações de user_id, device_id, merchant_id, valor e indicador de chargeback.

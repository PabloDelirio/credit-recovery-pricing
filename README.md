# Credit Recovery Pricing

Otimização de desconto e parcelamento em acordos de recuperação de crédito, usando causal inference, propensity scoring e maximização de valor esperado (EV).

## Objetivo

Em operações de recuperação de crédito, o processo de cobrança normalmente passa por duas decisões: **qual desconto oferecer** sobre o saldo devedor e **em quantas parcelas** permitir o pagamento, de forma a maximizar a chance de o devedor fechar um acordo.

Este projeto recria, com dados públicos e simulados, um sistema de precificação de acordos que eu desenvolvi profissionalmente. O objetivo não é apenas prever a probabilidade de aceite do acordo, mas sim **maximizar o valor esperado (EV) recuperado**, considerando explicitamente:

- o trade-off entre desconto e valor recebido (dar mais desconto aumenta a chance de aceite, mas reduz o valor recebido por acordo);
- o efeito do parcelamento na propensão de aceite e no risco de quebra do acordo ao longo do tempo;
- o valor do dinheiro no tempo (parcelas futuras valem menos que o pagamento à vista).

## Contexto e motivação

Esse é um problema clássico de precificação com resposta de demanda (parecido com revenue management), aplicado ao contexto de cobrança. Um erro comum nesse tipo de sistema é otimizar apenas a probabilidade de aceite — o que leva a sempre recomendar o desconto máximo, já que probabilidade de aceite é (em geral) crescente no desconto. A abordagem correta é maximizar o valor esperado, que pondera probabilidade de aceite pelo valor efetivamente recebido — o que produz, na maioria dos casos, um desconto ótimo *interior* (nem o mínimo, nem o máximo).

## Dados

Este projeto combina duas fontes de dados, deixado explícito aqui por transparência:

- **Base de crédito real**: [Give Me Some Credit](https://www.kaggle.com/c/GiveMeSomeCredit) (Kaggle), usada para treinar o modelo de score de propensão (A/B/C/D), com variáveis reais de renda, dívida, idade e histórico de atraso.
- **Mecanismo de resposta ao desconto/parcelamento**: **simulado**. Dados reais de teste de pricing em cobrança (qual desconto foi ofertado, se o devedor aceitou, se pagou todas as parcelas) não são públicos — nenhuma empresa de recuperação de crédito disponibiliza esse tipo de dado. Por isso, o mecanismo de resposta (probabilidade de aceite dado score/saldo/atraso/desconto/parcelas, e a probabilidade de honrar cada parcela) foi definido por mim através de um *data generating process* (DGP) explícito, documentado em `src/credit_pricing/simulation/`, com forma funcional inspirada em teoria econômica (curvas de resposta côncavas, hazard de parcela decrescente).

Essa separação é intencional: a parte de score usa dado real; a parte de resposta a preço, que não existe pública, é simulada de forma transparente e auditável.

## Metodologia (CRISP-DM)

O projeto segue as fases do CRISP-DM:

1. **Business Understanding** — definição do problema de precificação de acordos e da função objetivo (EV, não apenas propensão).
2. **Data Understanding** — análise exploratória da base de crédito (`notebooks/02_data_understanding.ipynb`).
3. **Data Preparation / Simulação** — construção do DGP de resposta ao desconto e de sobrevivência de parcela (`notebooks/03_dgp_simulation.ipynb`).
4. **Modeling** — modelo de score (`04_modeling_score.ipynb`), modelo de propensão de aceite (`05_modeling_response.ipynb`) e modelo de hazard de parcela (`06_modeling_hazard.ipynb`).
5. **Evaluation** — validação dos modelos e comparação com o DGP verdadeiro (já que os dados de resposta são simulados, é possível checar se o pipeline recupera a elasticidade real).
6. **Deployment** — otimização de desconto e parcelas por dívida via grid search sobre o valor esperado (`07_optimization.ipynb`), com relatório final em `reports/`.

## Estrutura do repositório

```
├── data/                  # dados brutos, intermediários e processados (não versionados)
├── notebooks/             # notebooks seguindo as fases do CRISP-DM
├── src/credit_pricing/    # pacote Python com o código de produção
│   ├── data/              # carregamento e pré-processamento
│   ├── simulation/        # DGP de resposta ao desconto e hazard de parcela
│   ├── models/             # score, modelo de resposta, modelo de hazard
│   ├── optimization/       # grid search de desconto/parcelas ótimos
│   └── evaluation/         # métricas de avaliação
├── tests/                  # testes automatizados (pytest)
├── reports/                # relatório final e figuras
├── configs/                # parâmetros e grids de otimização
└── .github/workflows/      # pipeline de CI (lint + testes)
```

## Como reproduzir

```bash
git clone https://github.com/PabloDelirio/credit-recovery-pricing.git
cd credit-recovery-pricing

python -m venv .venv
.venv\Scripts\Activate.ps1        # Windows (PowerShell)
# source .venv/bin/activate       # Linux/Mac

pip install -r requirements.txt
pip install -e .
pre-commit install
```

## Principais decisões técnicas

- **Maximização de EV, não de probabilidade de aceite**: a probabilidade de aceite tende a saturar em descontos altos, mas o valor recebido cai linearmente com o desconto — o ponto ótimo de EV fica, em geral, antes do platô da curva de probabilidade.
- **Restrição de monotonicidade**: os modelos de propensão impõem que a probabilidade de aceite seja não-decrescente em desconto e em número de parcelas, evitando que ruído no treino gere recomendações contraintuitivas.
- **Parcelamento como processo de sobrevivência**: a probabilidade de completar um parcelamento é modelada como um processo de hazard discreto (probabilidade de honrar a parcela *k*, dado que chegou até ela), não como uma probabilidade única de "aceite".
- **Valor presente**: o EV de um acordo parcelado traz cada parcela futura a valor presente, para refletir corretamente o custo de capital de esperar o recebimento.
- **Otimização por dívida individual**: como atraso e saldo variam por dívida (mesmo quando o cliente possui mais de uma), a otimização de desconto e parcelas é feita por dívida, não agregada por cliente.

## Status do projeto

🚧 Em desenvolvimento. Estrutura de projeto, ambiente e CI/CD configurados. Próxima etapa: Data Understanding sobre a base de crédito.

## Autor

Pablo — Data Scientist com background em Economia e causal inference.
[LinkedIn](https://www.linkedin.com/in/pablo-diniz-delirio-67018859/) · [GitHub](https://github.com/PabloDelirio)

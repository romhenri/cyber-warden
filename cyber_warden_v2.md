# cyber-warden v2

Controlador fuzzy Mamdani, em [scikit-fuzzy](https://pythonhosted.org/scikit-fuzzy/),
que prioriza alertas de seguranca a partir de duas leituras que um SOC ja
coleta: a gravidade tecnica do indicador e a confianca de que ele e um
verdadeiro positivo.

E um trabalho a parte do sistema especialista em [README.md](README.md) —
mesmo dominio (triagem de alertas), tecnica diferente (logica fuzzy em vez de
regras booleanas encadeadas em `experta`).

## O dominio

Um analista de SOC recebe alertas continuamente e precisa decidir, na hora,
quais tratar primeiro. Duas leituras isoladas raramente bastam: um indicador
de severidade alta mas com baixa confianca (pode ser falso positivo de uma
regra ruidosa) nao deveria furar a fila na frente de um indicador de
severidade media com confianca alta (uma assinatura confirmada). O
controlador combina as duas em um unico score de prioridade continuo, em vez
de dois limiares independentes que ignoram essa interacao.

## Arquitetura

| Componente | Onde fica |
| --- | --- |
| Antecedentes (entradas) | `severidade`, `confianca` — `ctrl.Antecedent` |
| Consequente (saida) | `prioridade` — `ctrl.Consequent` |
| Base de regras | lista de `ctrl.Rule`, uma por combinacao de termos |
| Motor de inferencia | `ctrl.ControlSystem` + `ctrl.ControlSystemSimulation` |

## Entradas e saida

| Variavel | Faixa | O que mede |
| --- | --- | --- |
| `severidade` (entrada) | 0 a 10 | gravidade tecnica do indicador, estilo CVSS |
| `confianca` (entrada) | 0 a 100 | confianca de que o alerta e verdadeiro positivo, nao ruido |
| `prioridade` (saida) | 0 a 10 | urgencia recomendada de tratamento |

## Os termos linguisticos

Tres termos por variavel (`baixa`, `media`, `alta`), o minimo que ainda
distingue as tres decisoes reais de uma fila de triagem: descartar ou revisar
depois, colocar na fila normal, ou acionar resposta imediata. Um quarto termo
adicionaria granularidade sem mudar nenhuma dessas tres decisoes.

Funcoes de pertinencia triangulares (`trimf`), cobrindo o dominio inteiro sem
lacunas: em qualquer ponto do eixo, pelo menos um termo tem pertinencia
maior que zero.

## Base de regras

9 regras, uma para cada combinacao das 3x3 possibilidades de `severidade` x
`confianca`, entao nenhum ponto do espaco de entrada fica sem regra que o
cubra:

| severidade \ confianca | baixa | media | alta |
| --- | --- | --- | --- |
| **baixa** | baixa | baixa | media |
| **media** | baixa | media | alta |
| **alta** | media | alta | alta |

A leitura e: severidade alta sozinha nao basta (confianca baixa segura a
prioridade em media), e confianca alta sozinha tambem nao (severidade baixa
segura a prioridade em media). So a combinacao dos dois extremos altos leva a
prioridade alta.

## Casos de teste

O notebook roda tres casos de exemplo e uma autoverificacao (`assert`):

| Caso | severidade | confianca | Esperado |
| --- | --- | --- | --- |
| scan de porta isolado, sem correlacao | 1 | 10% | prioridade baixa |
| assinatura de exploit conhecido confirmada | 9 | 95% | prioridade alta |
| indicador ambiguo de severidade media | 5 | 50% | prioridade proxima do meio da escala |

A autoverificacao confere que os extremos opostos do dominio (`0,0` e
`10,100`) produzem prioridades nos extremos opostos da escala, e que o ponto
central (`5,50`) cai perto do meio.

## Build e execucao

Sem build. No Colab, a primeira celula instala o `scikit-fuzzy`; rode as
celulas em ordem. Localmente:

```bash
pip install scikit-fuzzy numpy
jupyter nbconvert --to notebook --execute --inplace cyber_warden_v2.ipynb
```

A ultima celula roda a autoverificacao acima.

## Apresentacao

Discussao em sala de aula: a tabela de regras acima e os graficos de
pertinencia das tres variaveis (`severidade.view()`, `confianca.view()`,
`prioridade.view()`, ja no notebook) sao o material de apoio.

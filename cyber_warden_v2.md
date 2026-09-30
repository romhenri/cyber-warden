# cyber-warden v2

Versao 2 do [Warden](README.md): tudo do motor de regras original —
`Fact`s, as 10 regras `@Rule`, o motor de encadeamento progressivo, o
subsistema de explicacao, o cenario de demonstracao — foi mantido sem
reescrever. `cyber_warden_v2.ipynb` acrescenta uma camada nova por cima: um
controlador fuzzy Mamdani, em [scikit-fuzzy](https://pythonhosted.org/scikit-fuzzy/),
que le a mesma evidencia bruta e produz uma prioridade continua de 0 a 10, em
vez de so uma decisao crisp.

## O que foi reaproveitado de v1

| De v1 | Como e usado em v2 |
| --- | --- |
| `Conexao`, `Arquivo` (Facts de nivel 0) | viram as entradas do controlador fuzzy, via `severidade_de(...)` e `confianca_de(...)` |
| `EXPEDIENTE`, `LIMITE_EXFILTRACAO` | os mesmos dois limiares, reusados nas mesmas contas |
| Limiares das regras `indicador_forca_bruta` (10) e `indicador_varredura` (100) | normalizam `severidade` e disparam os pesos de `confianca` |
| O cenario de `demo()` (hosts `10.0.0.66`, `10.0.0.20`, `10.0.0.5`) | roda pelas duas leituras, crisp e fuzzy, no mesmo notebook |
| `motor.decisoes()` | comparado lado a lado com a prioridade fuzzy na tabela final |

Nada da BC ou do motor de v1 foi reescrito; a fuzzy e uma leitura adicional
sobre a mesma evidencia, nao uma substituicao.

## O dominio

Um analista de SOC recebe alertas continuamente e precisa decidir, na hora,
quais tratar primeiro. A decisao crisp de v1 (`ISOLAR`/`MONITORAR`/nenhuma) ja
resolve isso, mas colapsa tudo em tres categorias. Dentro da fila de
`MONITORAR`, por exemplo, hosts com evidencia bem diferente ficam
indistinguiveis. A camada fuzzy de v2 preserva essa granularidade: dois hosts
`MONITORAR` podem ter prioridades fuzzy 4.1 e 6.8, e a fila de triagem usa
esse numero para ordenar, em vez de tratar os dois como equivalentes.

## Entradas e saida

| Variavel | Faixa | De onde vem |
| --- | --- | --- |
| `severidade` (entrada) | 0 a 10 | calculada em `severidade_de(...)` a partir de `portas`, `tentativas_login`, `bytes_saida`, `hora`, `hash_mudou` |
| `confianca` (entrada) | 0 a 100 | calculada em `confianca_de(...)`, um peso fixo por indicador de v1 disparado |
| `prioridade` (saida) | 0 a 10 | urgencia recomendada, continua |

`severidade` pesa o quanto cada sinal se aproxima do seu proprio limiar de
v1, com peso maior para `exfiltracao` e `integridade_violada` — os dois
indicadores que em v1 ja bastam sozinhos para levantar uma ameaca de nivel
alto. `confianca` soma um peso fixo por indicador de v1 que a evidencia
dispara, tambem maior para esses dois, porque `varredura` e `forca_bruta`
sozinhos so viram ameaca quando combinados (`ameaca_intrusao`), e os outros
dois nao precisam de par.

## Os termos linguisticos

Tres termos por variavel (`baixa`, `media`, `alta`), o minimo que ainda
distingue as tres decisoes reais de uma fila de triagem: descartar ou revisar
depois, colocar na fila normal, ou acionar resposta imediata. Funcoes de
pertinencia triangulares (`trimf`), cobrindo o dominio inteiro sem lacunas.

## Base de regras

9 regras, uma para cada combinacao das 3x3 possibilidades de `severidade` x
`confianca`, entao nenhum ponto do espaco de entrada fica sem regra que o
cubra:

| severidade \ confianca | baixa | media | alta |
| --- | --- | --- | --- |
| **baixa** | baixa | baixa | media |
| **media** | baixa | media | alta |
| **alta** | media | alta | alta |

## Crisp x fuzzy, lado a lado

Rodando o cenario de `demo()` de v1 pelas duas leituras:

| Host | Decisao crisp (v1) | severidade | confianca | Prioridade fuzzy (v2) |
| --- | --- | --- | --- | --- |
| `10.0.0.66` | ISOLAR | 7.00 | 70.0 | 5.38 |
| `10.0.0.20` | MONITORAR | 3.06 | 40.0 | 4.65 |
| `10.0.0.5` | sem decisao | 0.24 | 0.0 | 1.67 |

A ordem das prioridades fuzzy concorda com a gravidade das decisoes crisp
(`ISOLAR` > `MONITORAR` > sem decisao), que e exatamente o que a
autoverificacao (`_autoverificar_fuzzy`) confere.

## Build e execucao

Sem build. No Colab, a primeira celula nova instala o `scikit-fuzzy`; rode as
celulas em ordem — as primeiras 20 sao identicas a `cyber_warden.ipynb`.
Localmente:

```bash
pip install experta scikit-fuzzy numpy
jupyter nbconvert --to notebook --execute --inplace cyber_warden_v2.ipynb
```

A ultima celula roda a autoverificacao fuzzy; a autoverificacao crisp de v1
(`_autoverificar()`) roda antes dela, sem alteracao.

## Apresentacao

Discussao em sala de aula: a tabela crisp x fuzzy acima e os graficos de
pertinencia das tres variaveis (`severidade.view()`, `confianca.view()`,
`prioridade.view()`, ja no notebook) sao o material de apoio.

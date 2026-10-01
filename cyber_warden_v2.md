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

## As dez regras herdadas de v1, em linguagem natural

Sem alteracao nenhuma em relacao a v1 — reproduzidas aqui para que este
documento seja autossuficiente.

### Nivel 1: da evidencia ao indicador

**1. `indicador_forca_bruta`**
SE uma conexao acumulou 10 ou mais falhas de login na janela observada,
ENTAO registre o indicador `forca_bruta` para o host de origem.

**2. `indicador_varredura`**
SE uma conexao tocou 100 ou mais portas distintas,
ENTAO registre o indicador `varredura` para o host de origem.

**3. `indicador_exfiltracao`**
SE uma conexao enviou mais de 500 MB para fora E isso aconteceu fora do
expediente (antes das 8h ou a partir das 19h),
ENTAO registre o indicador `exfiltracao` para o host de origem.

As duas condicoes precisam valer juntas. Volume alto sozinho e backup.

**4. `indicador_integridade`**
SE um arquivo critico do sistema teve o checksum divergindo do baseline,
ENTAO registre o indicador `integridade_violada` para o host ligado a alteracao.

Unica regra de nivel 1 que nao olha para a rede.

### Nivel 2: do indicador a ameaca

**5. `ameaca_intrusao`**
SE o mesmo host produziu os indicadores `varredura` E `forca_bruta`,
ENTAO levante a ameaca `intrusao_ativa` de nivel alto.

O "mesmo host" e o coracao da regra. Os dois indicadores em hosts diferentes nao
significam nada.

**6. `ameaca_vazamento`**
SE um host produziu o indicador `exfiltracao`,
ENTAO levante a ameaca `vazamento_dados` de nivel alto.

Um unico indicador ja sustenta a hipotese, porque exfiltracao fora de hora nao
tem leitura inocente.

**7. `ameaca_critica`** (salience 80)
SE um host tem o indicador `integridade_violada` E ja tem alguma ameaca de nivel
alto,
ENTAO escale para a ameaca `comprometimento_raiz` de nivel critico.

A salience 80 garante que a escalada aconteca antes de qualquer decisao ser
tomada sobre aquele host.

### Nivel 3: da ameaca a decisao

**8. `revogar_monitoramento`** (salience 110)
SE um host tem ameaca critica E ja existe uma decisao de `MONITORAR` para ele,
ENTAO remova essa decisao da memoria de trabalho.

Unica regra que retira um fato da MT em vez de acrescentar.

**9. `decisao_isolar`** (salience 100)
SE um host tem uma ameaca de nivel critico,
ENTAO recomende `ISOLAR` aquele host.

**10. `decisao_monitorar`** (salience 50)
SE um host tem uma ameaca de nivel alto E nao tem nenhuma ameaca critica,
ENTAO recomende `MONITORAR` aquele host.

`ISOLAR` e `MONITORAR` sao mutuamente exclusivas para o mesmo host atraves de
tres mecanismos: **salience** ordena o conjunto-conflito (`ameaca_critica` 80
vence `decisao_monitorar` 50), **`NOT(...)`** tira `decisao_monitorar` da
agenda assim que a ameaca critica entra na MT, e **`retract`** revoga um
`MONITORAR` ja declarado numa rodada anterior (`revogar_monitoramento`,
salience 110, antes de `decisao_isolar`). Detalhe completo em
[README.md](README.md#resolucao-de-conflito).

## Os 3 casos de teste do motor crisp

Tambem herdados de v1 sem alteracao, rodados por `_autoverificar()` no inicio
do notebook, antes da camada fuzzy:

### Caso 1: a cadeia de ponta a ponta (`_verificar_cadeia`)

Tres hosts entram na MT ao mesmo tempo e o motor roda uma vez.

| Host | Evidencia | Esperado |
| --- | --- | --- |
| `10.0.0.66` | 512 portas, 47 falhas de login, `/etc/shadow` alterado | `ISOLAR` |
| `10.0.0.20` | 900 MB de saida as 2h | `MONITORAR` |
| `10.0.0.5` | 2 portas, 1 falha, trafego normal as 14h | nenhuma decisao |

Verifica tambem que o host critico nunca aparece na trilha de explicacao com
uma decisao de `MONITORAR`, nem provisoria.

### Caso 2: os limiares, dos dois lados (`_verificar_limiares`)

Cada limiar e testado logo abaixo e logo acima do corte:

| Condicao | Nao dispara | Dispara |
| --- | --- | --- |
| falhas de login | 9 | 10 |
| portas distintas | 99 | 100 |
| bytes de saida | exatamente 500 MB | 500 MB + 1 byte |

Verifica ainda que o mesmo volume enviado dentro do expediente (14h) nao gera
`exfiltracao`.

### Caso 3: exclusao mutua entre rodadas (`_verificar_exclusao_mutua`)

A escalada chega numa segunda janela de coleta, com a decisao antiga ja na
memoria: declara varredura + forca bruta e roda (`MONITORAR`), depois declara
o arquivo critico alterado e roda de novo (`ISOLAR`, exatamente uma decisao
na MT, com `revogar_monitoramento` disparando antes de `decisao_isolar`).

Esses tres casos sao os que garantem que a evidencia usada pela tabela crisp
x fuzzy abaixo (`_evidencia`, os mesmos 3 hosts) produz as decisoes crisp
corretas antes de virar entrada do controlador fuzzy.

## Controlador fuzzy Mamdani

A parte nova de v2: um controlador fuzzy Mamdani, em
[scikit-fuzzy](https://pythonhosted.org/scikit-fuzzy/), que roda por cima da
mesma evidencia do motor de regras acima, produz uma prioridade continua, e
dela deriva uma terceira acao de triagem — `RESTRINGIR` — que v1 nao tinha
como ter, porque so decidia entre dois valores fixos.

### O dominio

Um analista de SOC recebe alertas continuamente e precisa decidir, na hora,
quais tratar primeiro. A decisao crisp de v1 (`ISOLAR`/`MONITORAR`/nenhuma) ja
resolve isso, mas colapsa tudo em tres categorias, e so tem duas acoes
possiveis quando alguma ameaca existe. Entre "isolar o host" e "so
monitorar", falta uma resposta intermediaria — restringir o alcance do host
(segmentar a rede, revogar credenciais) sem isola-lo — para o caso comum de
uma ameaca real mas ainda sem confirmacao de comprometimento de raiz. A
camada fuzzy de v2 preenche essa lacuna: em vez de reduzir tudo a dois
valores fixos, o controlador deriva a acao da prioridade continua, com uma
terceira opcao no meio.

### Entradas e saida

| Variavel | Faixa | De onde vem |
| --- | --- | --- |
| `severidade` (entrada) | 0 a 10 | calculada em `severidade_de(...)` a partir de `portas`, `tentativas_login`, `bytes_saida`, `hora`, `hash_mudou` |
| `confianca` (entrada) | 0 a 100 | calculada em `confianca_de(...)`, um peso fixo por indicador de v1 disparado |
| `prioridade` (saida) | 0 a 10 | urgencia recomendada, continua |

`severidade` pesa o quanto cada sinal se aproxima do seu proprio limiar de
v1, com peso maior para `exfiltracao` e `integridade_violada` — os dois
indicadores que em v1 ja bastam sozinhos para levantar uma ameaca de nivel
alto. Os pesos (3/3/4/4, somando 10 quando ha dois ou mais indicadores
corroborando) foram calibrados para que um host com evidencia tao forte
quanto `10.0.0.66` em v1 (varredura, forca bruta e integridade violada ao
mesmo tempo) sature `severidade` em 10 — sem isso, um host assim ficava a
meio caminho entre os termos `media` e `alta`, e a acao derivada nao refletia
a gravidade real do caso.

`confianca` soma um peso fixo por indicador de v1 que a evidencia dispara,
tambem maior para `exfiltracao` e `integridade_violada`, porque `varredura` e
`forca_bruta` sozinhos so viram ameaca quando combinados (`ameaca_intrusao`),
e os outros dois nao precisam de par.

### Os termos linguisticos

Tres termos por variavel (`baixa`, `media`, `alta`), o minimo que ainda
distingue as tres decisoes reais de uma fila de triagem: descartar ou revisar
depois, colocar na fila normal, ou acionar resposta imediata. Funcoes de
pertinencia triangulares (`trimf`), cobrindo o dominio inteiro sem lacunas.

### Base de regras

9 regras, uma para cada combinacao das 3x3 possibilidades de `severidade` x
`confianca`, entao nenhum ponto do espaco de entrada fica sem regra que o
cubra:

| severidade \ confianca | baixa | media | alta |
| --- | --- | --- | --- |
| **baixa** | baixa | baixa | media |
| **media** | baixa | media | alta |
| **alta** | media | alta | alta |

### Uma terceira acao: `RESTRINGIR`

A acao nao vem de reler o numero defuzzificado (`sim.output["prioridade"]`)
contra limiares novos — isso refuzzificaria um valor que ja perdeu
informacao no processo de centroide. Em vez disso, cada termo de
`prioridade` guarda sua propria forca de disparo agregada, acessivel em
`prioridade.terms[label].membership_value[sim]` depois do `compute()`, e a
acao e a do termo com maior forca — a leitura mais direta que o Mamdani
oferece, sem inventar um segundo conjunto de limiares.

| Termo vencedor | Acao |
| --- | --- |
| `baixa` | `MONITORAR` |
| `media` | `RESTRINGIR` (ex.: segmentar o host, revogar credenciais, sem isolar) |
| `alta` | `ISOLAR` |

Em empate, a ordem de leitura (`alta`, `media`, `baixa`) favorece a acao mais
cautelosa, pelo mesmo motivo que v1 favorece escalada em `ameaca_critica`.

### Crisp x fuzzy, lado a lado

Rodando o cenario de `demo()` de v1 pelas duas leituras:

| Host | Decisao crisp (v1) | severidade | confianca | prioridade | Acao fuzzy (v2) |
| --- | --- | --- | --- | --- | --- |
| `10.0.0.66` | ISOLAR | 10.00 | 70.0 | 8.14 | ISOLAR |
| `10.0.0.20` | MONITORAR | 4.09 | 40.0 | 4.90 | RESTRINGIR |
| `10.0.0.5` | sem decisao | 0.36 | 0.0 | 1.67 | MONITORAR |

`10.0.0.20` e o caso que mostra a terceira acao em uso: em v1 ele so podia
virar `MONITORAR`, mas a evidencia (900 MB exfiltrados fora do expediente,
sem corroboracao de outro indicador) pede algo entre "so observar" e
"isolar" — e e exatamente onde a acao fuzzy cai. A acao de `10.0.0.66`
concorda com `ISOLAR` de v1, e a de `10.0.0.5` concorda com a ausencia de
decisao, que e exatamente o que a autoverificacao (`_autoverificar_fuzzy`)
confere.

### Build e execucao

Sem build. No Colab, a primeira celula nova instala o `scikit-fuzzy`; rode as
celulas em ordem — as primeiras 20 sao identicas a `cyber_warden.ipynb`.
Localmente:

```bash
pip install experta scikit-fuzzy numpy
jupyter nbconvert --to notebook --execute --inplace cyber_warden_v2.ipynb
```

A ultima celula roda a autoverificacao fuzzy; a autoverificacao crisp de v1
(`_autoverificar()`) roda antes dela, sem alteracao.

### Apresentacao

Discussao em sala de aula: a tabela crisp x fuzzy acima e os graficos de
pertinencia das tres variaveis (`severidade.view()`, `confianca.view()`,
`prioridade.view()`, ja no notebook) sao o material de apoio.

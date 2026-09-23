# cyber-warden

Sistema especialista de seguranca cibernetica escrito em
[experta](https://github.com/nilp0inter/experta), com encadeamento progressivo
em cadeia.

O sistema recebe evidencia bruta de rede (conexoes observadas e alteracoes em
arquivos) e caminha sozinho, regra a regra, ate recomendar uma acao: isolar o
host ou apenas monitora-lo. Nenhuma etapa e chamada diretamente pelo programa
principal. O que existe e um conjunto de regras independentes, e o motor decide
sozinho quais disparar e em que ordem.

## Arquitetura

| Componente | Onde fica |
| --- | --- |
| Base de Conhecimento (BC) | as classes `Fact` e as regras `@Rule` |
| Memoria de Trabalho (MT) | `motor.facts`, alimentada pelos `declare` |
| Motor de inferencia | `Warden.run`, reescrito para expor a agenda |
| Subsistema de explicacao | `Warden.disparos` e `Warden.explicar()` |

## Subsistema de explicacao

Cada disparo de regra e registrado como `(regra, origem, motivo)` na lista
`disparos`, que fica em ordem cronologica. O `explicar()` reagrupa essa lista
**por decisao**, porque a pergunta que o subsistema precisa responder e "por que
esta decisao", e nao "o que aconteceu primeiro". Numa execucao com varios hosts
a ordem cronologica intercala as cadeias e nao se le.

```
MONITORAR 10.0.0.20
  1. [indicador_exfiltracao] 900 MB enviados por 10.0.0.20 as 2h, fora do expediente
  2. [ameaca_vazamento] volume anormal de saida atribuido a 10.0.0.20
  3. [decisao_monitorar] ameaca vazamento_dados em 10.0.0.20 sem evidencia de comprometimento de raiz

ISOLAR 10.0.0.66
  1. [indicador_integridade] checksum de /etc/shadow divergiu, alteracao ligada a 10.0.0.66
  2. [indicador_forca_bruta] 47 falhas de login vindas de 10.0.0.66
  3. [indicador_varredura] 512 portas distintas tocadas por 10.0.0.66
  4. [ameaca_intrusao] varredura e forca bruta partindo do mesmo host 10.0.0.66
  5. [ameaca_critica] integridade violada somada a intrusao_ativa no host 10.0.0.66
  6. [decisao_isolar] ameaca critica comprometimento_raiz confirmada em 10.0.0.66
```

Um host que gerou indicadores mas nao chegou a nenhuma decisao aparece sob
`SEM DECISAO`, entao nenhum disparo some do relatorio.

Quando a decisao muda no meio do caminho, a trilha mostra a mudanca em vez de
esconde-la. No caso 3 dos testes, o mesmo host aparece sendo monitorado,
escalado, revogado e so entao isolado:

```
ISOLAR 2.2.2.2
  ...
  4. [decisao_monitorar] ameaca intrusao_ativa em 2.2.2.2 sem evidencia de comprometimento de raiz
  5. [indicador_integridade] checksum de /etc/shadow divergiu, alteracao ligada a 2.2.2.2
  6. [ameaca_critica] integridade violada somada a intrusao_ativa no host 2.2.2.2
  7. [revogar_monitoramento] monitoramento de 2.2.2.2 revogado: a ameaca escalou para critica
  8. [decisao_isolar] ameaca critica comprometimento_raiz confirmada em 2.2.2.2
```

## A cadeia

```
evidencia  ->  indicador  ->  ameaca  ->  decisao
Conexao        Rastro         Ameaca      Decisao
Arquivo
```

Cada nivel so dispara quando o anterior ja esta na memoria de trabalho, entao
a inferencia caminha sozinha da evidencia bruta ate a acao recomendada. Um host
que nao produz nenhum indicador nunca chega a ter uma decisao.

## Os fatos

Os campos sao documentados na docstring de cada classe, e nao com `Field(...)`.

| Fato | Nivel | Campos |
| --- | --- | --- |
| `Conexao` | 0 | `origem`, `portas`, `tentativas_login`, `bytes_saida`, `hora` |
| `Arquivo` | 0 | `caminho`, `origem`, `critico`, `hash_mudou` |
| `Rastro` | 1 | `nome`, `origem`, `motivo` |
| `Ameaca` | 2 | `nome`, `origem`, `nivel` |
| `Decisao` | 3 | `acao`, `alvo`, `porque` |

Dois limiares ficam no topo do modulo, para nao ficarem escondidos dentro das
regras: `EXPEDIENTE = range(8, 19)` (das 8h as 18h) e
`LIMITE_EXFILTRACAO = 500_000_000` (500 MB).

## As dez regras em linguagem natural

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

## O ciclo do motor

O `run()` foi reescrito para tornar a agenda visivel. E o mesmo laco do
`KnowledgeEngine.run()` da biblioteca, com as mesmas chamadas, na mesma ordem e
com a mesma condicao de parada. Saiu apenas o `watchers` (logging) e o contador
que existe so para alimenta-lo. O `logging.DEBUG` da biblioteca despeja centenas
de linhas internas da rede Rete sem mostrar a agenda de forma legivel.

Os cinco passos, a cada volta:

1. Casamento de padroes contra a MT, montando o conjunto-conflito.
2. Agenda vazia encerra a execucao.
3. Resolucao de conflito escolhe uma ativacao.
4. Execucao do lado direito da regra escolhida.
5. Volta ao passo 1, com a MT ja alterada.

Ao ler a saida verbosa, atencao a um detalhe: **dispara sempre a ultima linha
impressa da agenda, nao a primeira**. A lista e mantida em ordem crescente de
prioridade (`bisect.insort` sobre a chave `(salience, factids)`) e
`get_next()` e `activations.pop()`, que remove do fim.

## Resolucao de conflito

`ISOLAR` e `MONITORAR` sao mutuamente exclusivas para um mesmo host. Tres
mecanismos garantem isso, e os tres sao necessarios:

- **salience**: ordena o conjunto-conflito. `ameaca_critica` (80) vence
  `decisao_monitorar` (50), entao um host que vai acabar isolado nunca chega a
  receber uma recomendacao de monitoramento, nem provisoria.
- **`NOT(...)`**: `decisao_monitorar` exige que nao exista ameaca critica
  naquele host, entao ela sai da agenda assim que a hipotese critica entra na MT.
- **`retract`**: o `NOT(...)` sozinho nao basta. Ele barra uma ativacao nova,
  mas nao remove da MT um `MONITORAR` ja declarado numa rodada anterior, quando
  a evidencia de escalada so chega na janela de coleta seguinte. A regra
  `revogar_monitoramento` (salience 110) retrata esse fato antes que
  `decisao_isolar` declare o substituto.

A refracao (refractoriness) do proprio experta impede que a mesma ativacao
dispare duas vezes com os mesmos fatos.

Rodando com `verboso=True` da para ver isso acontecer: no ciclo 8 a agenda tem
`decisao_monitorar` e `ameaca_critica`, a segunda vence por salience, e no ciclo
seguinte `decisao_monitorar` ja nao esta mais la.

## Casos de teste

`_autoverificar()` roda tres casos. Todos usam `assert`, entao qualquer um que
falhe interrompe a celula com o estado que causou a falha.

### Caso 1: a cadeia de ponta a ponta (`_verificar_cadeia`)

Tres hosts entram na MT ao mesmo tempo e o motor roda uma vez.

| Host | Evidencia | Esperado |
| --- | --- | --- |
| `10.0.0.66` | 512 portas, 47 falhas de login, `/etc/shadow` alterado | `ISOLAR` |
| `10.0.0.20` | 900 MB de saida as 2h | `MONITORAR` |
| `10.0.0.5` | 2 portas, 1 falha, trafego normal as 14h | nenhuma decisao |

Verifica tambem que o host critico nunca aparece na trilha de explicacao com uma
decisao de `MONITORAR`, nem provisoria. Se a salience de `ameaca_critica` cair
abaixo da de `decisao_monitorar`, essa asercao quebra.

### Caso 2: os limiares, dos dois lados (`_verificar_limiares`)

Cada limiar e testado logo abaixo e logo acima do corte, entao tanto afrouxar
quanto apertar o valor quebra o teste.

| Condicao | Nao dispara | Dispara |
| --- | --- | --- |
| falhas de login | 9 | 10 |
| portas distintas | 99 | 100 |
| bytes de saida | exatamente 500 MB | 500 MB + 1 byte |

Verifica ainda que o mesmo volume enviado **dentro** do expediente (14h) nao
gera `exfiltracao`, o que fixa a segunda condicao da regra 3.

### Caso 3: exclusao mutua entre rodadas (`_verificar_exclusao_mutua`)

E o caso que justifica a existencia do `retract`. A escalada chega numa segunda
janela de coleta, com a decisao antiga ja na memoria.

1. Declara a conexao com varredura e forca bruta, roda. Esperado:
   apenas `MONITORAR`.
2. Declara o arquivo critico alterado, roda de novo. Esperado: apenas `ISOLAR`,
   com exatamente **uma** decisao na MT.
3. Verifica que `revogar_monitoramento` disparou **antes** de `decisao_isolar`,
   o que fixa a salience 110 acima da 100.

Sem o `retract`, o passo 2 termina com `MONITORAR` e `ISOLAR` valendo ao mesmo
tempo para o mesmo host.

### Validacao por mutacao

Os tres casos foram validados sabotando o codigo de proposito, uma alteracao por
vez. Oito de nove mutacoes sao detectadas: afrouxar qualquer um dos tres
limiares, remover o teste de horario, apagar o `retract`, baixar a salience de
`ameaca_critica`, elevar a de `decisao_monitorar` ou baixar a de
`revogar_monitoramento`.

A unica que passa sem ser detectada e baixar a salience de `decisao_isolar`, e
por um motivo legitimo: com o `NOT(...)` e o `retract` no lugar, o resultado
final deixa de depender dessa prioridade especifica.

## Como rodar

Abra `cyber_warden.ipynb` no Colab e execute as celulas em ordem. A primeira
instala o experta. As ultimas imprimem a agenda ciclo a ciclo, as decisoes, a
explicacao e a autoverificacao.

## Patch de compatibilidade

O `frozendict 1.2`, dependencia do experta, usa `collections.Mapping`, que foi
removido no Python 3.10. Fixar `frozendict==1.2` nao resolve, porque e
justamente essa versao que quebra. A celula de patch reexporta os ABCs de
`collections.abc` para `collections` antes de importar o experta, e por isso
precisa rodar antes de qualquer outra.

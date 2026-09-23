# Arquitetura

Warden e um unico notebook (`cyber_warden.ipynb`); o diagrama usa a framing *script*, onde cada caixa e um bloco de codigo dentro dele.

```
┌─────────┐    ┌──────────────────┐
│  demo() │    │ _autoverificar() │
│ cenario │    │    checagens     │
└─────────┘    └──────────────────┘
     │                   │
     └───────────┬───────┘
                 ▼ declare
       ┌───────────────────┐
       │    Warden.run()   │
       │ agenda + salience │
       └───────────────────┘
                 │
                 │
                 ▼ evidencia
          ┌─────────────┐
          │ indicador_* │
          │   4 regras  │
          └─────────────┘
                 │
                 │
                 ▼ Rastro
           ┌──────────┐
           │ ameaca_* │
           │ 3 regras │
           └──────────┘
                 │
                 │
                 ▼ Ameaca
           ┌───────────┐
           │ decisao_* │
           │  3 regras │
           └───────────┘
                 │
                 │
                 ▼ Decisao
          ┌────────────┐
          │ explicar() │
          │ decisoes() │
          └────────────┘
```

## As caixas

**demo() / _autoverificar()** - celulas 9 e 11. Unicos pontos que alimentam a memoria de trabalho: `declare(Conexao(...))` e `declare(Arquivo(...))`. `_autoverificar()` roda `_verificar_cadeia`, `_verificar_limiares` e `_verificar_exclusao_mutua`.

**Warden.run()** - celula 7. Reescreve o laco do `KnowledgeEngine`: casa padroes, atualiza a agenda (conjunto-conflito), imprime a agenda quando `verboso=True`, tira a proxima ativacao e executa o lado direito. A ordem da agenda vem de `salience`; a refracao do experta impede o disparo repetido.

**indicador_\*** - `indicador_forca_bruta` (>=10 falhas), `indicador_varredura` (>=100 portas), `indicador_exfiltracao` (>500 MB fora de `EXPEDIENTE`), `indicador_integridade` (arquivo critico com hash alterado). Consomem `Conexao`/`Arquivo`, produzem `Rastro`.

**ameaca_\*** - `ameaca_intrusao` (varredura + forca bruta no mesmo host), `ameaca_vazamento` (exfiltracao), `ameaca_critica` (integridade violada somada a qualquer ameaca alta, `salience=80`). Produzem `Ameaca`.

**decisao_\*** - `revogar_monitoramento` (110), `decisao_isolar` (100), `decisao_monitorar` (50, com `NOT(Ameaca(nivel="critico"))`). Produzem e retiram `Decisao`.

**explicar() / decisoes()** - leitura. `explicar()` imprime a lista `self.disparos`, preenchida por `_registrar`, que descobre o nome da regra via `sys._getframe(1)`. `decisoes()` filtra a memoria de trabalho por `Decisao`.

## Observacao

A exclusao mutua entre ISOLAR e MONITORAR precisa de tres mecanismos empilhados (salience, `NOT`, `retract`) porque nenhum sozinho cobre os dois casos: o `NOT` so barra ativacao nova, e a escalada pode chegar numa rodada posterior com MONITORAR ja na memoria de trabalho. Essa e a unica parte do notebook onde a ordem de disparo e carregada de significado.

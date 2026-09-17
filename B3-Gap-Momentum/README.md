# B3 Gap Momentum

Conjunto de dois scripts Pine Script para selecionar semanalmente uma acao da B3 e calcular um backtest sintetico com reinvestimento integral do patrimonio.

## Arquivos

- `B3_Gap_Momentum_Estrategia.pine`: Script 1. Calcula Gap Momentum, Momentum, ranking, decisao semanal e publica os canais invisiveis usados pelo backtest.
- `B3_Gap_Momentum_Backtest.pine`: Script 2. Nao escolhe ativos. Le as decisoes/execucoes do Script 1 e calcula o resultado financeiro.

## Como funciona a estrategia

O Script 1 acompanha um universo fixo de 40 acoes da B3.

### 1. Gap Momentum

Parametros padrao:

- periodo dos gaps: 41 pregoes;
- SMA da linha de sinal: 15 periodos.

Calculo:

- gap diario = abertura atual - fechamento anterior;
- Gap Ratio = 100 x soma dos gaps positivos / soma do valor absoluto dos gaps negativos;
- quando a soma dos gaps negativos e zero, o script usa Gap Ratio = 1.0;
- a linha de sinal e a SMA do Gap Ratio;
- SMA subindo = ALTA;
- SMA caindo = BAIXA;
- SMA horizontal conserva o ultimo estado ALTA/BAIXA ja definido.

A acao so e elegivel quando o diagnostico do calculo e valido e o estado e ALTA.

### 2. Momentum de preco

Parametro padrao: 49 pregoes.

`Momentum = close / close[49] - 1`

Entre as acoes elegiveis em ALTA, vence a de maior Momentum. Em empate exato, a ordem fixa do universo e usada como desempate.

### 3. Decisao semanal

O calendario de PETR4 e usado apenas para detectar o primeiro pregao efetivamente negociado de cada nova semana da B3. Se segunda-feira for feriado, o rebalanceamento ocorre no primeiro pregao seguinte.

No rebalanceamento semanal:

- escolhe-se o Top 1 de Momentum entre as acoes validamente em ALTA;
- se nenhuma estiver elegivel, a decisao e CAIXA;
- se a mesma acao continuar sendo Top 1, a decisao semanal e registrada novamente, mas nao ha venda e recompra artificial no backtest.

O painel operacional do Script 1 possui uma regra defensiva intrassemanal quando a acao deixa de estar em ALTA. Quando essa troca defensiva e efetivamente executada, o Script 1 tambem publica os canais de entrada/saida usados pelo Script 2, portanto o backtest contabiliza a operacao. A troca defensiva nao altera `BT_WEEKLY_POSITION`, que continua representando somente o rebalanceamento semanal.

### 4. Preco usado no backtest

A decisao semanal e executada, para fins de backtest, na abertura do primeiro pregao B3 posterior a decisao.

Em uma troca `PETR4 -> VALE3`, na data de execucao o Script 1 publica:

- preco/data de saida de PETR4;
- preco/data de entrada de VALE3;
- identificacao da nova posicao.

## Como usar os dois scripts em conjunto

1. Abra um grafico do TradingView no timeframe **1D (diario)**.
2. Adicione `B3_Gap_Momentum_Estrategia.pine` ao grafico.
3. Adicione `B3_Gap_Momentum_Backtest.pine` ao mesmo grafico.
4. Abra **Configuracoes -> Entradas** do Script 2.
5. No grupo **1. Conexao com Script 1 - seguir ordem 01 a 09**, substitua `Fechamento` em cada campo pela fonte correspondente do Script 1.
6. As fontes do Script 1 aparecem no menu na mesma ordem abaixo. Conecte exatamente de 01 a 09.

| # | Campo no Script 2 | Selecione no Script 1 |
|---|---|---|
| 01 | `BT_WEEKLY_POSITION` | `BT_WEEKLY_POSITION` |
| 02 | `BT_WEEKLY_EVENT` | `BT_WEEKLY_EVENT` |
| 03 | `BT_TRADE_POSITION` | `BT_TRADE_POSITION` |
| 04 | `BT_TRADE_EVENT` | `BT_TRADE_EVENT` |
| 05 | `BT_ENTRY_PRICE` | `BT_ENTRY_PRICE` |
| 06 | `BT_ENTRY_TIME` | `BT_ENTRY_TIME` |
| 07 | `BT_EXIT_POSITION` | `BT_EXIT_POSITION` |
| 08 | `BT_EXIT_PRICE` | `BT_EXIT_PRICE` |
| 09 | `BT_EXIT_TIME` | `BT_EXIT_TIME` |

Os campos 01 e 02 servem tambem para auditoria da decisao semanal. O motor financeiro usa principalmente os campos 03 a 09.

Se um campo de timestamp continuar apontando para `Fechamento`, o painel do backtest deve indicar `ERRO CONEXAO`.

## Backtest e reinvestimento

O Script 2 usa capitalizacao composta e reinveste 100% do patrimonio disponivel.

Exemplo com capital inicial de R$ 1.000:

- primeira operacao: +10% -> capital passa a R$ 1.100;
- segunda operacao: -5% sobre R$ 1.100 -> capital passa a R$ 1.045;
- a terceira operacao usa integralmente os R$ 1.045.

Quantidade sintetica:

`quantidade = capital disponivel / preco de entrada`

Na saida:

`capital novo = quantidade x preco de saida`

## Modos do backtest

### ANUAL

Permite selecionar um ano de 2018 a 2026. O capital e reiniciado para aquele teste e uma posicao iniciada antes do ano nao e herdada.

### COMPLETO

Processa continuamente desde 2018 ate os dados atuais disponiveis, sem reiniciar o patrimonio na virada de cada ano.

O painel mostra, entre outras metricas:

- primeira entrada e ultima saida;
- capital inicial e final;
- lucro/prejuizo total e retorno total;
- anos investidos / anos com prejuizo;
- lucro medio anual;
- CAGR anual;
- entradas / operacoes fechadas;
- vencedoras / perdedoras e taxa de acerto;
- lucro bruto / perda bruta;
- retorno medio por operacao;
- melhor / pior operacao;
- drawdown maximo;
- Profit Factor;
- posicao aberta e dados da entrada aberta.

A tabela detalhada de compras e vendas fica **oculta por padrao**. Ative `Exibir tabela de compras e vendas` no grupo Visualizacao para mostrar as 12 operacoes fechadas mais recentes.

## Limitacoes atuais

- Backtest sintetico: nao usa `strategy.entry()` do Strategy Tester para operar varios tickers.
- Nao inclui corretagem, emolumentos, impostos ou slippage.
- Permite quantidade fracionaria/sintetica para reinvestir 100% do patrimonio.
- Uma posicao ainda aberta nao e encerrada artificialmente no final do periodo; o painel trabalha com capital realizado.
- Os dois scripts devem ser usados no timeframe diario para manter a semantica de datas e precos da interface.
- A disponibilidade historica depende do feed do TradingView para os ativos do universo.

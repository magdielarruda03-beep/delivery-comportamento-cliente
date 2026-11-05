# Fórmulas e decisões de estrutura

As fórmulas estão escritas como aparecem no Excel em português, com separador `;`.

## 1. Arquitetura: quatro camadas, cada uma só lê a de baixo

O arquivo segue a lógica de uma arquitetura medalhão: **dado bruto → dado por cliente → agregação → apresentação**. Nenhuma página visual calcula a partir de outra página visual; todas leem de Pedidos, Base_Clientes ou Calc. Assim cada número pode ser auditado em um só lugar.

| Camada | Abas | Papel | Granularidade |
| --- | --- | --- | --- |
| Bruta | Pedidos, Clientes | Fatos como vieram do sistema. Só valores, sem fórmula (exceto 3 colunas auxiliares da coorte) | 1 linha por pedido / por cliente |
| Configuração | Parametros | Premissas e regras de negócio editáveis | Células soltas |
| Modelo do cliente | Base_Clientes | Uma linha por cliente com todas as métricas e o segmento | 1 linha por cliente |
| Apoio | Calc | Séries que alimentam gráficos (mês a mês, curva de retenção) e datas do período anterior | 1 linha por mês / por M |
| Apresentação | Visao_Geral, Clientes_RFM, Coorte, Segmentos, Lista_Acao | Só leitura e agregação final; filtros no menu lateral | Indicadores |

**Por que uma Base_Clientes em vez de calcular tudo direto nos pedidos?** Análise de comportamento é sobre o cliente, não sobre o pedido. Recência, frequência, ticket e segmento precisam existir por cliente antes de serem agregados. Com essa tabela, qualquer pergunta nova ("% de caçadores de cupom no Centro") vira um CONT.SES simples, sem fórmula matricial.

**Por que uma aba Calc?** Gráficos precisam de um intervalo contínuo de células. Deixar essas séries fora das páginas mantém as páginas limpas e permite mostrar o histórico inteiro no gráfico mensal mesmo quando o filtro de período está curto.

**Por que menos fórmulas nas abas brutas?** Pedidos e Clientes são as abas onde se colam dados novos. Quanto menos fórmula ali, menor o risco de quebrar algo ao atualizar.

## 2. O dataset sintético

Os dados foram gerados pelo Claude (IA da Anthropic) com **comportamentos ocultos** (personas), para que a segmentação e a coorte tenham algo real para encontrar. A coluna de persona foi removida do arquivo final: o dashboard precisa descobrir os grupos sozinho, como aconteceria com dados reais.

| Persona (oculta) | % da base | Pedidos/mês | O que simula |
| --- | --- | --- | --- |
| Campeão | 5% | 4,0 | Cliente de hábito forte |
| Fiel | 10% | 2,0 | Recorrente estável |
| Regular | 20% | 0,9 | 25% deles param de comprar entre 2 e 10 meses antes do fim (esfriando) |
| Ocasional | 20% | 0,2 | Compra rara |
| Compra única | 12% | — | Exatamente 1 pedido |
| Churn | 18% | 1,2 | Cadastrou em 2025 e parou antes de jun/2026; nos 90 dias antes de sair, a entrega fica ~16 min mais lenta |
| Novo | 9% | 0,7 | Cadastro nos últimos 70 dias; 60% usam cupom no 1º pedido |
| Caçador de cupom | 6% | 0,8 | 80% dos pedidos com cupom; vem mais do iFood |

Outras regras foram embutidas para que os gráficos tenham padrão:

- O horário depende da categoria (marmita no almoço, pizza e japonês no jantar).
- Sexta a domingo têm mais pedidos.
- 15% dos clientes são "premium", com ticket 40% maior.
- Frete grátis acima de R$ 120.
- A avaliação cai quando o tempo de entrega passa de 40, 50 e 62 min.
- O cancelamento base é de 3,5%, maior no iFood e nos clientes que vão sair.
- O 1º pedido de cada cliente nunca é cancelado, então todo cliente tem pelo menos uma compra.

O histórico vai de jan/2025 a out/2026 (22 meses), com 720 clientes e 6.211 pedidos. A base cresce ao longo do tempo, por isso as primeiras coortes são menores.

## 3. Convenções usadas em todas as fórmulas

**Intervalos nomeados.** Cada coluna de Pedidos virou um nome (de `Pedidos!B2:B20000` em diante). Assim `SOMASES(p_valor;p_cliente;A2;p_status;"Entregue")` se lê como frase. O limite de 20.000 linhas deixa espaço para colar pedidos novos sem editar nenhuma fórmula. Células vazias não atrapalham CONT.SES e SOMASES.

| Nome | Coluna em Pedidos | Nome | Coluna em Pedidos |
| --- | --- | --- | --- |
| p_id | ID_Pedido | p_cupom | Cupom (Sim/Não) |
| p_data | Data | p_pag | Forma_Pagamento |
| p_hora | Hora | p_canal | Canal_Pedido |
| p_dia | Dia_Semana | p_tempo | Tempo_Entrega_Min |
| p_cliente | ID_Cliente | p_aval | Avaliacao |
| p_cat | Categoria | p_status | Status |
| p_valor | Valor_Total | p_coorte | Coorte (auxiliar) |
| p_desc | Desconto | p_idx / p_unico | auxiliares da coorte |

Nomes de célula única:

- `data_ref`, `dias_ativo` e `lim_cupom`: aba Parametros.
- `per_ini` e `per_fim`: menu da Visão Geral.
- `prev_ini` e `prev_fim`: aba Calc.
- `coh_met`: menu da Coorte.
- `seg_sel`: menu da Lista de Ação.

**Receita = só pedido Entregue.** Todo indicador de receita, pedido, ticket e avaliação filtra `p_status;"Entregue"`. Pedido cancelado não gerou dinheiro nem uma experiência completa. A exceção é a taxa de cancelamento, que olha todos os status.

**Receita = Valor_Total**, ou seja, produtos + taxa de entrega − desconto. É o que o cliente efetivamente pagou.

**Data de referência em vez de HOJE().** Com HOJE(), a recência mudaria todo dia sozinha, e o segmento de um cliente mudaria sem nenhum dado novo. Uma data fixa (31/10/2026) deixa o arquivo reproduzível.

**Dois relógios diferentes.**

- Indicadores de pedido (receita, ticket, mapa de calor, canais) seguem o período do menu.
- Indicadores de cliente (RFM, coorte, recompra, aquisição) usam todo o histórico até a data de referência. Um cliente que não comprou no período continua existindo e precisa aparecer no RFM justamente como "em risco".

**Cores das células.**

- Azul com fundo amarelo: entrada editável.
- Verde: valor puxado de outra aba.
- Preto: fórmula.
- Cinza: coluna auxiliar.

**Funções escolhidas.** Só foram usadas funções que funcionam do Excel 2010 em diante e no LibreOffice: CONT.SES, SOMASES, MÉDIASES, ÍNDICE/CORRESP, SOMARPRODUTO, MÁXIMOSES/MÍNIMOSES. PROCX, FILTRO, ÚNICO e CLASSIFICAR ficaram de fora. São funções de matriz dinâmica, que não abrem em versões antigas.

## 4. Parametros: as regras de negócio em um só lugar

Toda premissa virou célula, nunca número escrito dentro de fórmula.

| Parâmetro | Valor inicial | Onde é usado | Por que esse valor |
| --- | --- | --- | --- |
| data_ref (C5) | 31/10/2026 | Recência, coorte, filtros "até hoje" | Último dia com dados |
| dias_ativo (C6) | 60 | Clientes ativos, % ativos por canal e bairro | Em delivery, 2 meses sem pedir já indica perda de hábito |
| lim_cupom (C7) | 50% | Perfil "Caçador de cupom" | Com metade ou mais dos pedidos com desconto, o cupom virou condição para comprar |
| Faixas R (C14:F14) | ≤15, ≤45, ≤120, ≤240 dias | Nota R | Próximas dos quintis da base, arredondadas |
| Faixas F (C15:F15) | ≥12, ≥6, ≥3, ≥2 pedidos | Nota F | Idem; ≥2 garante que nota 2 já significa "voltou" |
| Faixas M (C16:F16) | ≥R$1.000, ≥500, ≥200, ≥100 | Nota M | Idem |
| Matriz R×F (C27:G31) | 25 células com nome de segmento | Coluna Segmento e cores do Mapa RFM | Modelo clássico de segmentação RFM em 10 grupos |

**Por que faixas fixas e não quintis automáticos?** Com quintil automático, o cliente muda de nota sem mudar de comportamento: basta a base crescer. Com faixas fixas, "nota 5 em recência" sempre quer dizer "comprou nos últimos 15 dias". As linhas 20 a 22 mostram os percentis reais da base para calibrar as faixas:

- Recência: `=PERCENTIL(Base_Clientes!$K$2:$K$721;0,2)` para o "Top 20%". Na recência, menor é melhor, então o top 20% fica no percentil 20.
- Frequência e receita: `=PERCENTIL(Base_Clientes!$F$2:$F$721;0,8)` para o "Top 20%". Aqui maior é melhor, então o top 20% fica no percentil 80.

**Por que a matriz usa só R e F?** Em delivery, frequência e valor andam juntos, então M quase não muda a decisão. Uma grade 5×5 cabe na tela e pode ser editada; uma 5×5×5 teria 125 combinações. M continua calculado e aparece no código RFM (ex.: "255").

**Por que o período do painel não está nesta aba?** Filtro que se usa todo dia precisa estar na tela onde o resultado aparece.

## 5. Base_Clientes: fórmula de cada coluna

Uma linha por cliente, na mesma ordem da aba Clientes. Por isso as colunas A a E são referências diretas (`=Clientes!A2`), sem PROCV. **[E]** = `p_status;"Entregue";p_data;"<="&data_ref`.

| Col. | Campo | Fórmula (linha 2) | Decisão |
| --- | --- | --- | --- |
| F | Pedidos | `=CONT.SES(p_cliente;A2;[E])` | A frequência do RFM |
| G | Receita | `=SOMASES(p_valor;p_cliente;A2;[E])` | O valor do RFM |
| H | Ticket_Medio | `=SEERRO(G2/F2;"")` | Evita #DIV/0! |
| I | Primeira_Compra | `=SE(F2=0;"";MÍNIMOSES(p_data;p_cliente;A2;[E]))` | Define a coorte. Usa a 1ª compra, não o cadastro |
| J | Ultima_Compra | `=SE(F2=0;"";MÁXIMOSES(p_data;p_cliente;A2;[E]))` | Base da recência |
| K | Recencia_Dias | `=SE(F2=0;"";data_ref-J2)` | Datas são números, então a subtração dá dias |
| L | Intervalo_Medio_Dias | `=SE(F2>1;(J2-I2)/(F2-1);"")` | n pedidos geram n−1 intervalos; com 1 pedido fica vazio |
| M | Pedidos_Periodo | `=CONT.SES(p_cliente;A2;p_status;"Entregue";p_data;">="&per_ini;p_data;"<="&per_fim)` | Permite contar clientes únicos no período |
| N | Receita_Periodo | Igual a M, com SOMASES em p_valor | Ticket por bairro no período |
| O | Pct_Cupom | `=SEERRO(CONT.SES(p_cliente;A2;p_cupom;"Sim";[E])/F2;"")` | Base do perfil caçador de cupom |
| P | Avaliacao_Media | `=SEERRO(MÉDIASES(p_aval;p_cliente;A2;[E]);"")` | Ignora pedidos sem nota em vez de contá-los como zero |
| Q | Tempo_Entrega_Medio | `=SEERRO(MÉDIASES(p_tempo;p_cliente;A2;[E]);"")` | Liga a experiência de entrega ao cliente |
| R | Cancelamentos | `=CONT.SES(p_cliente;A2;p_status;"Cancelado";p_data;"<="&data_ref)` | Consulta |
| AA:AE | Qtd_Pizza … Qtd_Açaí | `=CONT.SES(p_cliente;$A2;p_cat;"Pizza";[E])` | Auxiliares para a categoria favorita |
| S | Categoria_Favorita | `=SE(F2=0;"";SUBSTITUIR(ÍNDICE($AA$1:$AE$1;CORRESP(MÁXIMO(AA2:AE2);AA2:AE2;0));"Qtd_";""))` | Lê o cabeçalho da maior contagem; em empate, vence a primeira categoria |
| T | R (nota) | `=SE(K2<=Parametros!$C$14;5;SE(K2<=Parametros!$D$14;4;SE(K2<=Parametros!$E$14;3;SE(K2<=Parametros!$F$14;2;1))))` | Na recência, menos dias é melhor (≤) |
| U | F (nota) | Mesma estrutura, com `F2>=` e a linha 15 | Na frequência, mais é melhor (≥) |
| V | M (nota) | Mesma estrutura, com `G2>=` e a linha 16 | |
| W | RFM | `=T2&U2&V2` | Código de texto para filtrar |
| X | Segmento | `=ÍNDICE(Parametros!$C$27:$G$31;CORRESP(T2;Parametros!$B$27:$B$31;0);CORRESP(U2;Parametros!$C$26:$G$26;0))` | A nota R escolhe a linha e a nota F escolhe a coluna da matriz |
| Y | Perfil_Cupom | `=SE(O2="";"";SE(O2>=lim_cupom;"Caçador de cupom";"Regular"))` | Perfil paralelo aos segmentos |
| Z | Rank_Lista | ver seção 9 | Ordem da Lista de Ação |
| AF | Pedidos_Periodo_Anterior | Igual a M, com `prev_ini` e `prev_fim` | Variação de "clientes que compraram" |
| AG | Coorte | `=SE(I2="";"";DATA(ANO(I2);MÊS(I2);1))` | Chave da coorte |

## 6. RFM

RFM foi escolhido porque usa só o que todo delivery já tem (data e valor dos pedidos), cabe em uma frase para o dono do negócio e leva direto a uma ação. Um cluster (k-means) poderia achar grupos parecidos, mas os grupos mudam a cada rodada e é difícil explicar por que um cliente caiu em um deles.

Caminho de cada cliente: **dados → 3 métricas (K, F, G) → 3 notas de 1 a 5 (T, U, V) → matriz R×F → segmento (X)**.

| R \ F | F1 | F2 | F3 | F4 | F5 |
| --- | --- | --- | --- | --- | --- |
| R5 | Novos | Potenciais Fiéis | Potenciais Fiéis | Campeões | Campeões |
| R4 | Promissores | Potenciais Fiéis | Fiéis | Campeões | Campeões |
| R3 | Precisam de Atenção | Precisam de Atenção | Fiéis | Fiéis | Fiéis |
| R2 | Hibernando | Hibernando | Em Risco | Em Risco | Não Pode Perder |
| R1 | Perdidos | Perdidos | Em Risco | Não Pode Perder | Não Pode Perder |

- **Como ler a matriz:** o canto superior direito são os melhores clientes. O canto inferior direito são os que mais dói perder: compravam muito e sumiram.
- **"Caçador de cupom" não é um segmento.** É outra dimensão: um Campeão também pode ser caçador de cupom. Nesse caso muda a oferta, não o segmento.
- **Mapa RFM com cores dinâmicas.** Cada quadrado mostra `=ÍNDICE(Parametros!$C$27:$G$31;linha;coluna)` e `=CONT.SES(Base T;5;Base U;1)`. A cor vem de uma regra de formatação condicional por segmento. Se a matriz mudar, o nome e a cor do quadrado mudam juntos.

## 7. Coorte

A coorte agrupa clientes pelo **mês da 1ª compra** e acompanha quantos voltaram em cada mês seguinte. O RFM mostra quem está indo embora agora; a coorte mostra em que momento o cliente costuma ir embora.

A dificuldade no Excel é contar **clientes únicos** por mês. A solução foram três colunas auxiliares em Pedidos:

| Col. | Fórmula (linha 2) | O que faz |
| --- | --- | --- |
| R · Coorte | `=SEERRO(DATA(ANO(ÍNDICE(Base_Clientes!$I:$I;CORRESP(E2;Base_Clientes!$A:$A;0)));MÊS(ÍNDICE(Base_Clientes!$I:$I;CORRESP(E2;Base_Clientes!$A:$A;0)));1);"")` | Mês da 1ª compra do cliente |
| S · Mes_Desde_1a_Compra | `=SE(R2="";"";(ANO(B2)-ANO(R2))*12+MÊS(B2)-MÊS(R2))` | Meses de calendário desde a coorte |
| T · Cliente_Mes_Unico | `=SE(E(Q2="Entregue";S2<>"");SE(CONT.SES($E$2:E2;E2;$S$2:S2;S2;$Q$2:Q2;"Entregue")=1;1;0);0)` | O intervalo que cresce marca 1 só no primeiro pedido de cada cliente em cada mês |

**Matriz da página Coorte:**

- Tamanho da coorte: `=CONT.SES(Base_Clientes!AG;D14)`.
- Célula M1 com "% clientes que voltaram": `=SOMASES(p_unico;p_coorte;$D14;p_idx;1)/$E14`.
- A fórmula fica dentro de `SE(OU($E14=0;DATAM($D14;1)>Calc!$B$5);"";SE(coh_met=…))`. A primeira parte deixa vazio o mês que ainda não aconteceu; a segunda aplica a métrica escolhida no menu.
- O formato (%, número ou R$) muda por formatação condicional, para que os valores continuem sendo números.

**Curva média de retenção (Calc H:L):**

- Elegíveis: `=CONT.SES(Base AG;"<="&DATAM($B$5;-M))`, só as coortes que já completaram aquele M.
- Voltaram: `=SOMASES(p_unico;p_idx;M;p_coorte;"<="&DATAM($B$5;-M))`.
- Retenção = voltaram / elegíveis.

**Decisões da coorte:**

- **1ª compra, não cadastro:** quem só se cadastrou não tem o que reter.
- **"Comprou no mês", não "continua ativo":** é a retenção clássica.
- **Mês de calendário:** acompanha o calendário de campanhas do negócio.
- **Média ponderada pelo tamanho da coorte:** coortes pequenas não distorcem a curva.
- **Vazio no futuro, não zero:** zero seria lido como "ninguém voltou".
- **M1 a M12 na tela:** M0 é sempre 100%.
- **Auxiliares em Pedidos, não uma matriz cliente × mês:** a matriz seriam ~16 mil CONT.SES sobre 20 mil linhas cada.

## 8. Página Visão Geral

**[P]** = `p_status;"Entregue";p_data;">="&per_ini;p_data;"<="&per_fim`.

**Período anterior automático (Calc B3:B4).** `prev_ini = per_ini-(per_fim-per_ini+1)` e `prev_fim = per_ini-1`. O período anterior tem o mesmo número de dias e termina no dia antes do atual.

| Card | Fórmula | Observação |
| --- | --- | --- |
| Receita | `=SOMASES(p_valor;[P])` | |
| Pedidos entregues | `=CONT.SES([P])` | |
| Ticket médio | `=receita / pedidos` | Razão dos totais, não média das médias |
| Clientes que compraram | `=CONT.SE(Base_Clientes!M;">0")` | Clientes únicos |
| Clientes novos | `=CONT.SES(Base I;">="&per_ini;Base I;"<="&per_fim)` | 1ª compra no período |
| Pedidos por cliente | `=pedidos / clientes que compraram` | |
| Pedidos com cupom | `=CONT.SES(p_cupom;"Sim";[P]) / pedidos` | |
| Tempo médio de entrega | `=MÉDIASES(p_tempo;[P])` | |
| Avaliação média | `=MÉDIASES(p_aval;[P])` | Ignora pedidos sem nota |
| Taxa de cancelamento | `=CONT.SES(p_status;"Cancelado";datas) / CONT.SES(p_status;"<>";datas)` | Sobre todos os status |

**Variação de cada card:**

`=SEERRO(SE(anterior=0;"sem base no período anterior";SE(atual>=anterior;"▲ ";"▼ ")&TEXTO(ABS(atual/anterior-1);"0,0%")&" vs período anterior");"—")`

- **Cor:** a formatação condicional testa `ESQUERDA(célula;1)`.
- **Regra invertida:** em cupom, tempo de entrega e cancelamento, ▲ fica vermelho, porque subir é ruim.
- **Avaliação:** a variação é em pontos, não em %.

**Série mensal (Calc A:F).**

- Mês inicial: `=DATA(ANO(MÍNIMO(p_data));MÊS(MÍNIMO(p_data));1)`. Os meses seguintes usam `=DATAM(anterior;1)`.
- Pedidos do mês: `=CONT.SES(p_status;"Entregue";p_data;">="&A9;p_data;"<"&DATAM(A9;1))`.
- O gráfico mensal ignora o filtro de período de propósito, para mostrar a tendência inteira.

**Mapa de calor:** `=CONT.SES(p_dia;$D39;p_hora;11;[P])`. A coluna Dia_Semana veio pronta nos dados, então o resultado não depende de qual dia cada versão do Excel considera o início da semana.

**Experiência de entrega:** as faixas "De (>)" e "Até (≤)" ficam em células editáveis. Fórmulas: `=CONT.SES(p_tempo;">"&F72;p_tempo;"<="&G72;[P])` e `=MÉDIASES(p_aval;p_tempo;">"&F72;p_tempo;"<="&G72;[P])`.

## 9. Clientes & RFM, Segmentos e Lista de Ação

Essas páginas leem a Base_Clientes ("Base X" = `Base_Clientes!$X$2:$X$721`).

| Card | Fórmula | Decisão |
| --- | --- | --- |
| Clientes na base | `=CONT.VALORES(Base A)` | Linha de baixo: ativos = `CONT.SE(Base K;"<="&dias_ativo)` |
| Taxa de recompra | `=CONT.SE(Base F;">=2") / total` | % que comprou mais de uma vez |
| Receita por cliente | `=SOMA(Base G) / total` | A linha de baixo mostra a mediana |
| Receita histórica em risco | `=SOMASE(Base X;"Em Risco";Base G)+SOMASE(Base X;"Não Pode Perder";Base G)` | Tamanho do problema em reais |
| Dias entre pedidos | `=MÉDIA(Base L)` | Ignora quem tem 1 pedido |

- **Tabela de segmentos:** clientes, % clientes, % receita (`SOMASE(Base X;seg;Base G)/SOMA(Base G)`), ticket como razão dos totais, recência e avaliação médias, % caçadores de cupom.
- **Aquisição:** por canal de origem, mostra receita por cliente, pedidos por cliente, % recompra, % ativos, % Campeões e % caçadores. A pergunta é qual canal traz cliente que fica.
- **Bairros:** o tempo médio é ponderado por pedidos, com `SOMARPRODUTO((Base C=bairro)*Base Q*Base F)/SOMASE(Base C;bairro;Base F)`.
- **Segmentos & Ações:** contagens por fórmula. Quem são, objetivo, ação, oferta, canal, frequência e KPI são texto escrito à mão (a regra de negócio).

**Ranking da Lista de Ação, sem FILTRO e sem CLASSIFICAR (coluna Z da Base):**

`=SE(X2=seg_sel;CONT.SES($X$2:$X$721;X2;$G$2:$G$721;">"&G2)+CONT.SES($X$2:X2;X2;$G$2:G2;G2);"")`

- A 1ª parte conta quantos clientes do segmento têm receita maior.
- A 2ª parte desempata receitas iguais pela ordem da linha, então nenhuma posição se repete.
- Na página, cada linha de 1 a 50 busca o cliente da sua posição: `=SEERRO(ÍNDICE(Base A;CORRESP($D19;Base Z;0));"")`.

## 10. Decisões visuais

- **Layout de dashboard pronto:** menu lateral preto, fundo cinza e cards brancos. Linhas de grade e cabeçalhos ficam escondidos.
- **Navegação com `=HIPERLINK("#'Coorte'!A1";"Coorte")`:** mantém a fonte da célula e dispensa macros.
- **Filtros no menu, com nome e validação de dados:** não dependem de segmentação de dados nem de tabela dinâmica.
- **Espaço entre cards:** vem de bordas grossas na cor do fundo, não de colunas vazias. Assim as tabelas usam colunas contínuas.
- **Paleta Data Prisma com função:** preto e cinza para clientes saudáveis, vermelho para risco e para o que piorou. Onde tem vermelho, tem algo para olhar.
- **Formatação condicional no lugar de gráfico, quando basta:** mapa de calor, barras de dados e escalas de cor. Gráfico só onde a forma importa (tendência, proporção, ranking, curva de retenção).
- **Cada card diz o recorte:** um subtítulo indica se o número é do período ou do histórico.

## 11. Limitações e manutenção

- **Dados novos:** cole embaixo em Pedidos/Clientes, arraste Pedidos R:T e uma linha de Base_Clientes, e avance a data de referência.
- **Limite de 20.000 pedidos:** os intervalos nomeados vão até essa linha. Acima de ~50 mil pedidos, o melhor caminho é Power Query ou SQL.
- **Desempenho:** são ~44 mil fórmulas. A coluna T de Pedidos é a mais pesada; se ficar lento, use cálculo manual (F9).
- **Valores novos:** uma categoria, canal ou bairro novo precisa de uma linha na tabela da página correspondente. Mais meses exigem estender Calc e a matriz de coorte.
- **Avaliação:** só ~45% dos pedidos têm nota, então a média oscila em recortes pequenos.
- **Faixas RFM:** foram calibradas para este dataset. Com dados reais, recalibre olhando os percentis em Parametros.

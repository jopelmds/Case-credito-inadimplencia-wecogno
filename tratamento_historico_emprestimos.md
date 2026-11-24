# Tratamento das variáveis de `historico_emprestimos`

Cada linha registra um contrato histórico vinculado a um cliente pelo par (`id_cliente`, `id_contrato`). As 37 variáveis abaixo seguem a classificação e os tratamentos do [notebook de ingestão e tratamento](notebooks/01_ingestao_entendimento_tratamento.ipynb).

#### Features binárias

##### `indicador_ultima_solicitacao_contrato`

Indica se o registro foi a última solicitação do contrato (1 para sim, 0 para não). Foram padronizados espaços e maiúsculas, e os rótulos `sim`, `não` e `nao` foram mapeados para 1 ou 0. Os códigos 0 e 1 também são aceitos. O resultado é `Int64`, sem nulos na base tratada.

##### `indicador_ultima_solicitacao_dia`

Indica se o registro foi a última solicitação do mesmo dia (1 para sim, 0 para não). A coluna já chega como `int64`, sem nulos, e foi mantida.

##### `indicador_seguro_contratado`

Indica se houve contratação de seguro junto com o crédito (1 para sim, 0 para não). Foi convertida de `float64` para `Int64`, preservando os nulos e a distinção entre informação ausente e seguro não contratado.

#### Features datetime

**Validade do formato e coerência entre eventos.** O notebook registra que todas as datas preenchidas puderam ser convertidas do formato `AAAA-MM-DD` para datetime (`datetime64[us]` na saída salva). A conversão representa ausências e eventuais falhas de leitura como `NaT`. Uma data pode passar nessa validação e ainda estar associada ao evento errado ou fora da sequência esperada. A conferência analisou a menor e a maior data de cada coluna, os valores ausentes, algumas relações entre eventos e a presença de data de liberação por status.

##### `data_decisao`

Data da decisão sobre a solicitação, inclusive em registros recusados ou cancelados. Está preenchida nos 211.944 registros e vai de 04/02/2017 a 09/03/2025. Foi usada como referência para comparar liberação e encerramento. Há 6 liberações e 367 encerramentos anteriores à decisão. A comparação sinaliza o par de datas, sem identificar qual campo explica a divergência.

##### `data_liberacao`

Data registrada da liberação do crédito. Vai de 13/02/2017 a 21/02/2025 e falta em 204.235 registros (96,36%). As 7.709 datas preenchidas estão em contratos aprovados. Outros 124.026 contratos aprovados estão sem essa data. Entre os registros com liberação informada, 6 têm liberação anterior à decisão, 6.703 têm liberação posterior e 1.000 têm ambas no mesmo dia. A liberação posterior à decisão é compatível com a sequência esperada. As ausências foram preservadas.

O cruzamento pelo par cliente-contrato mostrou que, dos 6.657 aprovados com liberação posterior à decisão e primeiro pagamento datado, 6.513 (97,84%) tiveram pagamento após a liberação, 20 no mesmo dia e 124 antes. A mediana foi de 27 dias desde a liberação e 41 desde a decisão. A comparação restrita à parcela nº 1 mostrou o mesmo padrão. Dos 124.026 aprovados sem data de liberação, 115.209 (92,89%) têm pagamento registrado. A liberação é uma candidata a marco de início quando disponível, com exceções para investigação. A origem de eventuais alterações contratuais ainda requer confirmação. Todos os contratos foram preservados.

Nos 124 casos com pagamento anterior à liberação, o primeiro pagamento registrado coincide com a data do pagamento da parcela nº 1. A decisão veio antes do pagamento em 68 contratos e caiu no mesmo dia em 56. A liberação veio depois da decisão e do pagamento nos 124 contratos. Em mediana, o pagamento ocorreu 12 dias após a decisão e 7 dias antes da liberação.

##### `data_primeiro_vencimento`

Data prevista de vencimento da primeira parcela. Vai de 03/03/2017 a 14/03/2025 e falta em 89.232 registros (42,10%). Em 15 registros com ambas as datas disponíveis, o primeiro vencimento está antes da liberação do crédito. Essa checagem tem cobertura limitada pela quantidade de liberações sem data.

##### `data_ultimo_vencimento_original`

Data da última parcela prevista no cronograma original. Vai de 08/06/2017 a 21/08/2031 e falta em 95.248 registros (44,94%). O limite em 2031 é compatível com um vencimento planejado. A conferência de mínimos e máximos considera a coluna inteira. A ordem em relação ao primeiro vencimento precisa ser avaliada por contrato.

##### `data_ultimo_vencimento`

Data revisada de vencimento da última parcela. Vai de 01/04/2017 a 21/02/2025 e falta em 115.128 registros (54,32%). A conferência apresentada cobre faixa de datas e ausências. A ordem em relação ao primeiro vencimento e as diferenças em relação ao cronograma original ainda precisam de validação por contrato. Uma alteração em relação ao prazo original pode refletir uma revisão legítima.

##### `data_encerramento`

Data registrada de encerramento do contrato. Vai de 01/01/2016 a 03/03/2025 e falta em 116.761 registros (55,09%). Foram encontrados 367 encerramentos anteriores à decisão e 17 anteriores à liberação. O menor encerramento registrado antecede a menor data de decisão da base. As datas foram preservadas para investigação.

##### `hora_solicitacao`

Horário de início da solicitação. Foi convertido para `datetime.time` após remover espaços nas extremidades e validar o formato `HH:MM:SS` de `00:00:00` a `23:59:59`. O notebook registrou zero horários inválidos e zero nulos. A checagem valida a representação do horário. A confirmação do horário efetivo depende do registro de origem.

##### Limites das checagens e possíveis causas

- Cada comparação cronológica exige as duas datas preenchidas. As contagens são por regra, e um contrato pode aparecer em mais de uma delas.
- Datas impossíveis, como `2025-02-30`, ou fora do formato esperado são problemas de conversão. Uma data válida no calendário ainda pode conter erro de digitação, estar no campo errado ou ter sido vinculada ao evento errado. Essas são hipóteses para as divergências observadas. A causa depende da conferência com o sistema de origem e das regras de cada evento.
- Entre as checagens pendentes estão comparar o primeiro e o último vencimento no mesmo contrato, avaliar revisões do cronograma e cruzar as datas do contrato com os pagamentos registrados em `historico_parcelas`. O notebook preserva os registros sinalizados e deixa a escolha da data de referência para a construção das bases.

#### Features categóricas ordinais

Nenhuma variável recebeu uma ordem nesta etapa. A hierarquia de `faixa_rendimento` permanece indefinida, por isso a coluna foi tratada como nominal.

#### Features categóricas nominais

O tratamento de texto converteu as colunas para `string`, aplicou minúsculas, removeu acentos, aparou espaços nas extremidades e reduziu espaços consecutivos. A remoção de acentos padroniza `não informado` como `nao informado`, e os nulos originais foram preservados.

##### `tipo_contrato`

Modalidade de contrato de crédito. O tratamento unificou as variantes nos rótulos `emprestimo em dinheiro` e `credito rotativo`.

##### `status_contrato`

Situação final do contrato, como aprovado, recusado, cancelado ou oferta não utilizada. Foram padronizados espaços e minúsculas, com todos os status preservados.

##### `tipo_pagamento`

Forma de pagamento do contrato. Os rótulos foram padronizados e `nao informado` foi mantido como categoria.

##### `finalidade_emprestimo`

Finalidade declarada para o empréstimo. A transformação unificou `cliente não informou a finalidade` e `não informado` em `nao informado` e manteve `nao se aplica` como categoria própria.

##### `tipo_cliente`

Perfil do cliente, como novo ou recorrente. Foram padronizados os rótulos conforme o tratamento de texto da seção.

##### `faixa_rendimento`

Categoria de retorno do cliente para o banco. As classes `baixa acao`, `baixa normal`, `media` e `alta` foram mantidas como rótulos nominais. O marcador `nao informado` e sua variante acentuada foram convertidos em nulos.

##### `tipo_portfolio`

Tipo de portfólio ao qual o contrato está associado. Os rótulos foram padronizados e as categorias existentes foram preservadas.

##### `tipo_produto`

Tipo de produto financeiro contratado. Os rótulos foram padronizados, mantendo `nao informado` como categoria.

##### `categoria_bem`

Categoria do bem financiado. Os rótulos foram padronizados e as categorias foram mantidas individualmente.

##### `combinacao_produto`

Combinação de produtos oferecida. O tratamento padronizou a escrita e preservou os componentes dos rótulos e os nulos originais.

##### `setor_vendedor`

Setor da empresa vendedora. Os nomes dos setores receberam a padronização de texto da seção.

##### `canal_venda`

Canal de origem da venda. Os nomes dos canais receberam a padronização de texto da seção.

##### `area_venda`

Código da área ou localização da venda. Foi convertido de `int64` para `string`, preservando todos os códigos como rótulos, inclusive `-1`.

##### `dia_semana_solicitacao`

Dia da semana em que a solicitação foi iniciada. Os nomes foram padronizados e mantidos como texto.

##### `motivo_recusa`

Motivo registrado para a recusa. O tratamento de texto preservou os motivos e manteve `nao se aplica` e `nao informado` como categorias distintas.

##### `acompanhantes_cliente`

Tipo de acompanhante no momento da solicitação, incluindo a opção de estar sozinho. Os rótulos foram padronizados e os nulos foram preservados como informação ausente.

#### Features inteiras

##### `id_contrato`

Identificador do contrato. A validação mantém apenas valores positivos. Todos os identificadores passaram na validação e a coluna permaneceu como `int64`, sem nulos.

##### `id_cliente`

Identificador do cliente associado ao contrato. A validação mantém apenas valores positivos. Todos os identificadores passaram na validação e a coluna permaneceu como `int64`, sem nulos.

##### `qtd_parcelas_planejadas`

Quantidade de parcelas previstas no contrato. Foi convertida de `float64` para `Int64`, preservando os nulos e os zeros registrados.

#### Features float

As oito colunas já chegam como `float64` e foram mantidas na escala original. Os valores preenchidos são não negativos na base analisada. O tratamento foi descritivo, com preservação de zeros e nulos.

##### `valor_solicitado`

Valor inicialmente solicitado pelo cliente. Foi mantido como `float64`, sem nulos na base tratada.

##### `valor_credito`

Valor do crédito registrado para o contrato. Foi mantido como `float64`, sem nulos na base tratada.

##### `valor_bem`

Valor do bem adquirido, quando aplicável. Foi mantido como `float64`, com as ausências preservadas.

##### `valor_parcela`

Valor estimado da parcela periódica. Foi mantido como `float64`, com preservação de zeros e ausências.

##### `valor_entrada`

Valor pago como entrada. Foi mantido como `float64`, preservando a distinção entre zero e informação ausente.

##### `percentual_entrada`

Percentual registrado da entrada sobre o valor total. Foi mantido como `float64`, na escala de origem e com os nulos preservados.

##### `taxa_juros_padrao`

Taxa de juros padrão registrada. Foi mantida como `float64`, na escala de origem e com 99,66% de ausências preservadas.

##### `taxa_juros_promocional`

Taxa de juros promocional registrada, quando aplicável. Foi mantida como `float64`, na escala de origem e com 99,66% de ausências preservadas.

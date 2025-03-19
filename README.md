# Otimização da escala de funcionários em regime híbrido

#### Aluno: [Victor Moreira Araújo](https://github.com/victoraraujo01/)
#### Orientadora: Carolina Abreu

---

Trabalho apresentado ao curso [BI MASTER](https://ica.puc-rio.ai/bi-master) como pré-requisito para conclusão de curso e obtenção de crédito na disciplina "Projetos de Sistemas Inteligentes de Apoio à Decisão".

- [Link para o código](hybrid_work_distribution.ipynb).

---

### Resumo

Este projeto tem como objetivo apresentar o desenvolvimento de um modelo de otimização para a escala de funcionários em regime híbrido, buscando a maximização de dois objetivos: a manutenção do espaço físico ocupado ao longo de toda a semana e a alocação de setores com funções relacionadas nos mesmos dias de trabalho presencial.

Por se tratar de um problema de otimização com múltiplos objetivos, foram realizados experimentos com estratégias distintas: a primeira, utilizando uma função objetivo agregada — com o valor dos dois objetivos somados e ponderados — e seleção por torneio; e a segunda, utilizando uma função objetivo com uma tupla de valores e seleção NSGA-II (Non-dominated Sorting Genetic Algorithm). Além disso, também foi avaliado o efeito da variação do algoritmo evolutivo entre o `eaSimple` e o `mu_plus_lambda` da biblioteca `deap`, e dos parâmetros de crossover, mutação, número de gerações e tamanho da população.

Para os cenários com grandes populações e muitas gerações, os melhores resultados foram obtidos de maneira consistente utilizando o algoritmo `mu_plus_lambda` e a seleção NSGA-II, com percentuais de crossover ligeiramente inferiores aos de mutação. Contudo, para populações menores, a função objetivo agregada e a seleção por torneio retornou resultados melhores do que o NSGA-II, tanto com o algoritmo `eaSimple` quanto com o `mu_plus_lambda`.

Espera-se, com esse trabalho, entregar um produto que contribua para uma maior eficiência no uso do espaço físico, redução de custos operacionais e aumento da satisfação dos funcionários, promovendo um equilíbrio entre trabalho remoto e presencial de forma estratégica. Esse estudo contribui para a modernização dos modelos de trabalho e pode ser aplicado em diversas organizações que buscam melhorar sua gestão de equipes híbridas.

### 1. Introdução

O regime de trabalho híbrido surgiu no contexto pós-pandêmico como uma alternativa que busca unir o melhor do trabalho remoto (ou _home office_) com os benefícios do modelo convencional presencial. Como uma consequência da flexibilidade que o torna atrativo, o modelo híbrido pode ser estabelecido de diversas formas diferentes, pois há uma grande quantidade de arranjos possíveis para divisão da força de trabalho por dias da semana ou até do mês. 

Nesse sentido, esse trabalho se propõe a encontrar os melhores arranjos para distribuição de funcionários de uma empresa ao longo da semana, considerando as seguintes restrições parametrizáveis pelo usuário:

1. Capacidade física máxima do espaço;
2. Ocupação desejada diferente por dia da semana;
3. Quantidade de dias presenciais obrigatórios diferente por setor;
4. Fixação dos dias de trabalho presencial de determinados setores;
5. Exclusão de determinadas combinações de dias.

Para avaliar quão bons são os arranjos propostos pela ferramenta, foram considerados dois objetivos distintos: (i) a manutenção do espaço físico ocupado ao longo de toda a semana e (ii) a alocação de setores com funções relacionadas nos mesmos dias de trabalho presencial.

Em um cenário como o proposto, com múltiplos objetivos, problemas de otimização se tornam substancialmente mais complexos e o emprego de algoritmos genéticos surge como uma excelente alternativa para viabilizá-lo. Assim, foi utilizada a biblioteca `deap` (_Distributed Evolutionary Algorithms in Python_), uma biblioteca desenvolvida em Python para facilitar a criação, implementação e execução de algoritmos evolutivos, como algoritmos genéticos e estratégias evolutivas, de forma flexível e eficiente.

A primeira parte do desenvolvimento de um trabalho como este, com algoritmos genéticos, envolve a modelagem do problema utilizando os conceitos de algoritmo genético. Assim, será apresentada a lógica para representação do problema em **indivíduos**, que correspondem a uma solução potencial para o problema que se deseja otimizar e **população**, conjunto de todos os indivíduos em uma **geração**.

Com esses conceitos definidos, o próximo passo é a construção da **função objetivo**, responsável por quantificar o(s) objetivo(s) do problema, e da **função de restrição**, que determina se um indivíduo está apto, ou não, para ser considerado como uma possível solução do problema.

Concluindo a modelagem, definem-se os operadores de **_crossover_** (operador genético que combina duas soluções － pais － para gerar uma nova solução － filho); e **mutação** (operador que faz pequenas alterações em um indivíduo para introduzir diversidade genética).

Com a modelagem definida, foi possível avaliar o impacto de diversos parâmetros e estratégias do algoritmo genético na qualidade da solução. Serão apresentados resultados da otimização utilizando métodos de seleção diferentes, estratégias alternativas para a função objetivo, algoritmos evolutivos distintos e o impacto das mudanças nos parâmetros de crossover, mutação, população e números de gerações.


### 2. Modelagem

#### 2.1. Arquivos de entrada

O projeto depende de dois arquivos .csv de input

- `pessoas_por_gerencia.csv`: esse arquivo deve possuir os IDs de cada setor, a sua quantidade de pessoas e a quantidade de dias obrigatórios.
- `relacionamentos.csv`: esse arquivo deve possuir, para cada setor, a lista de IDs dos setores relacionados, ou seja, aqueles que preferencialmente devem estar nos mesmos dias de presencial.

#### 2.2. Inputs do usuário (restrições do problema)

- `CAPACIDADE_PREDIO`: Constante que determina a capacidade máxima de pessoas. Exemplo: `300` pessoas;
- `QTD_DIAS_PRESENCIAIS_POSSIVEIS`: Lista de inteiros que representam com a quantidade de dias da semana possíveis. Exemplo: `[2, 3]` indica que são possíveis combinações de 2 e 3 dias de presencial;
- `COMBINACOES_DIAS_PROIBIDAS`: Lista de combinações de dias proibidas. Cada combinação é uma lista de 5 booleanos representados por 0 ou 1, indicando se o dia da semana correspondente àquela posição é de presencial (1) ou não (0). Exemplo: o valor `[[1,0,0,1,1], [0,1,1,1,0]]` indica que estão proibidas as combinações de trabalho presencial na segunda, quinta e sexta e na terça, quinta e sexta.
- `OCUPACAO_DESEJADA_POR_DIA`: Ocupação desejada por dia, caso haja interesse de manter o prédio mais livre em alguns dias da seman (representada por um valor percentual de 0 a 1 aplicado ao dia da semana correspondente àquela posição). Exemplo: a entrada `[0.9, 0.9, 0.95, 0.9, 0.9]` mostra um interesse de maior ocupação na quarta-feira.
- `GERENCIAS_FIXAS`: Dicionário que indica quais setores que têm dias fixos. Cada chave é o índice do setor e o valor é uma lista de 5 booleanos representados por 0 ou 1, indicando se o dia da semana correspondente àquela posição é de presencial, ou não. Exemplo: o dicionário `{8: [1,1,1,0,0], 12: [1,0,0,1,1]}` indica que o setor 8 obrigatoriamente deve trabalhar presencialmente na segunda, terça e quarta e o setor 12, na segunda, quinta e sexta.
- `PESO_OCUPACAO` e `PESO_RELACIONAMENTOS`: valores inteiros de 0 a 1 utilizados como pesos na função objetivo agregada, para dar mais relevância a um dos objetivos.


#### 2.3. Indivíduo

O indivíduo para esse problema é representado por uma matriz com N linhas e 5 colunas, sendo N a quantidade de setores indicados no arquivo `pessoas_por_gerencia.csv`, por exemplo:

| ID | Setor | Segunda | Terça | Quarta | Quinta | Sexta |
|----|-------|---------|-------|--------|--------|-------|
| 0  | A     |    4    |   4   |    4   |    0   |   0   |
| 1  | B     |    0    |   0   |   17   |   17   |   17  |
| 2  | C     |    1    |   0   |    0   |    1   |   0   |
| 3  | D     |    0    |   61  |    0   |   61   |   61  |
| 4  | E     |    0    |   0   |    0   |   13   |   13  |
| 5  | F     |    3    |   3   |    3   |    0   |   0   |
| 6  | G     |    52   |   52  |   52   |    0   |   0   |
| 7  | H     |    8    |   8   |    0   |    0   |   0   |
| 8  | I     |    0    |   0   |   72   |   72   |   72  |
| 9  | J     |    0    |   0   |    0   |   18   |   18  |
| 10 | K     |    35   |   35  |   35   |    0   |   0   |

Cada linha da matriz representa um setor e cada coluna, um dia da semana. O valor de cada célula da matriz corresponde, portanto, à quantidade de pessoas do setor trabalhando presencialmente naquele dia da semana.

Visando simplificar o problema, ao invés de realizar a seleção aleatória de cada um dos Nx5 valores durante a criação do indivíduo, decidiu-se selecionar aleatoriamente **_a combinação de dias de cada setor_** entre as combinações permitidas e preencher a matriz a partir daí. Assim como mostrado na seção [2.2](#22-inputs-do-usuário-restrições-do-problema) a combinação é uma lista de 5 booleanos representados por 0 ou 1, indicando se o dia da semana correspondente àquela posição é de presencial (1) ou não (0). A partir daí, a combinação de cada setor é multiplicada pela sua respectiva quantidade de pessoas para deixar o indivíduo em seu formato final.

Essa abordagem reduz significativamente a quantidade de indivíduos inválidos possíveis de serem gerados, diminuindo também a quantidade de restrições que precisam ser impostas no modelo e acelera a convergência. Antes de realizar a implementação em Python e `deap` foram feitas tentativas com o Solver que esbarraram em dificuldades oriundas justamente dessas condições.

#### 2.4. Função de restrição e objetivo

Em função da simplificação empregada na geração do indivíduo, a única restrição que precisou ser implementada foi a obrigação de respeitar a ocupação máxima desejada de cada andar por dia, considerando os fatores informados pelo usuário na variável `OCUPACAO_DESEJADA_POR_DIA`.

Já a função objetivo avalia dois critérios de otimização, conforme citado na [introdução](#1-introdução):

1. Manter o prédio com ocupação o mais constante possível ao longo da semana, considerando a possível variação de ocupação desejada por dia informada em `OCUPACAO_DESEJADA_POR_DIA`;
2. Maior quantidade possível de setores relacionados (conforme indicado no arquivo `relacionamentos.csv`) trabalhando nos mesmos dias de presencial;

Por se tratar de um problema de otimização com múltiplos objetivos, foram criadas duas funções objetivos: a `funcao_objetivo` e a `funcao_objetivo_agregada`, buscando avaliar a resposta do `deap` para uma otimização utilizando uma estratégia que considere os dois objetivos separadamente e outra, mais simples, que utiliza apenas a soma dos dois objetivos.

Assim, cada objetivo foi modelado como um fator que varia entre zero e um: `fitness_ocupacao` e `fitness_relacionamentos`. Caso o usuário tenha interesse em dar um peso maior para algum desses fatores na funcao objetivo agregada, ele pode fazê-lo alterando as variáveis `PESO_OCUPACAO` e `PESO_RELACIONAMENTOS`, como explicado na seção [2.2](#22-inputs-do-usuário-restrições-do-problema).

O fator `fitness_ocupacao` foi calculado usando a fórmula f(v)=exp(−v^k/s^k), em que `v` é a variância e `s` e `k` são constantes para ajustar quão rápido o valor converge para zero. Essa fórmula foi uma sugestão obtida no [Stack Exchange](https://math.stackexchange.com/questions/2833062/a-measure-similar-to-variance-thats-always-between-0-and-1) como uma alternativa de medida de dispersão sempre com valor entre 0 e 1, a fim de mantê-la num _range_ compatível com o outro objetivo.

O fator `fitness_relacionamentos` foi calculado dividindo a _quantidade de relacionamentos respeitados_ pela _quantidade total de relacionamentos informados_. Para que um relacionamento seja respeitado, todos os colaboradores dos dois setores precisam estar presentes nos mesmos dias. Caso os dois setores tenham quantidades diferentes de dias presenciais exigidos, vale o menor para essa avaliação.

#### 2.5. Mutação e crossover

Em alguns casos, conforme probabilidade de mutação informado na configuração do algoritmo (`MUTPB`), o indivíduo sofrerá uma mutação, com a variação de algum dos seus cromossomos (linhas da matriz, como explicado em [2.3](#23-indivíduo)). Nesse caso, um ou mais setores terão a sua combinação de dias selecionada alterada.

No algoritmo implementado nesse trabalho, existe uma probabilidade específica de cada linha (cromossomo) ser alterada (`indpb`), caso em que uma nova combinação de dias é sorteada. Importante atentar que, em função da restrição imposta pelo próprio problema, os setores com dias fixos não podem sofrer mutação.

Para a função de crossover, foi necessário fazer uma modificação no método padrão do `deap` pois como cada linha (cromossomo) tem a quantidade de empregados de cada setor trabalhando presencialmente por dia (coluna), não é possível simplesmente trocar as linhas de posição. Isso porque, ao fazer qualquer inversão dessa forma, estaríamos levando a quantidade de colaboradores de um setor para outra, alterando as características do problema informadas pelo usuário.

Nesse cenário, a solução proposta foi obter a combinação de dias original de cada indivíduo e realizar o two-point crossover deste objeto, respeitando, também, a restrição dos setores com dias fixados pelo usuário.

### 3. Resultados

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Proin pulvinar nisl vestibulum tortor fringilla, eget imperdiet neque condimentum. Proin vitae augue in nulla vehicula porttitor sit amet quis sapien. Nam rutrum mollis ligula, et semper justo maximus accumsan. Integer scelerisque egestas arcu, ac laoreet odio aliquet at. Sed sed bibendum dolor. Vestibulum commodo sodales erat, ut placerat nulla vulputate eu. In hac habitasse platea dictumst. Cras interdum bibendum sapien a vehicula.

Proin feugiat nulla sem. Phasellus consequat tellus a ex aliquet, quis convallis turpis blandit. Quisque auctor condimentum justo vitae pulvinar. Donec in dictum purus. Vivamus vitae aliquam ligula, at suscipit ipsum. Quisque in dolor auctor tortor facilisis maximus. Donec dapibus leo sed tincidunt aliquam.

### 4. Conclusões

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Proin pulvinar nisl vestibulum tortor fringilla, eget imperdiet neque condimentum. Proin vitae augue in nulla vehicula porttitor sit amet quis sapien. Nam rutrum mollis ligula, et semper justo maximus accumsan. Integer scelerisque egestas arcu, ac laoreet odio aliquet at. Sed sed bibendum dolor. Vestibulum commodo sodales erat, ut placerat nulla vulputate eu. In hac habitasse platea dictumst. Cras interdum bibendum sapien a vehicula.

Proin feugiat nulla sem. Phasellus consequat tellus a ex aliquet, quis convallis turpis blandit. Quisque auctor condimentum justo vitae pulvinar. Donec in dictum purus. Vivamus vitae aliquam ligula, at suscipit ipsum. Quisque in dolor auctor tortor facilisis maximus. Donec dapibus leo sed tincidunt aliquam.

---

Matrícula: 123.456.789

Pontifícia Universidade Católica do Rio de Janeiro

Curso de Pós Graduação *Business Intelligence Master*

---
title: "Database Indexing 101"
date: 2026-04-21 18:30:00
tags:
---

# Índices de bancos de dados
![cover](../../../assets/images/space01.png){: .cover-image }
## Definição e conceito

Index é a forma como chamamos comumente o relacionamento que direciona a engine do banco de dados para o endereço em disco de uma tupla filtrando pelo valor de colunas pré-definidas que são consideradas de alta frequência de filtro.

Indefiticamos colunas que são muito utilizadas para filtro, pedimos para o banco registrar os valores da coluna junto a um ponteiro para o endereço em disco da tupla. Quando precisamos de uma busca pela coluna, o banco utilizará esse indice para otimizar a busca já iniciando com o endereço da tupla em mãos após uma rapida busca no indice.

A definição é longa e tem uns detalhes a nível de sistema do banco, mas é uma solução simples de se implementar que traz uma melhora significativa (e muito perceptível) de performance nas queries de busca.

### Exemplo de uso real: 

Num sistema baseado em assinatura, precisamos buscar o plano de um usuário no banco de dados múltiplas vezes. Podemos ter um índice pela referência ao usuário, já que é coluna frequentemente usada para buscas.

## Disclaimer: focar em PostgreSQL

Como trabalho majoritariamente com Postgres, vou focar exemplos e detalhes mais especificos na forma de se fazer com Postgres, mas a lógica de indices se aplica a outros bancos de dados relacionais.


### Paralelo com mundo real

Como o exemplo utilizado na documentação oficial do Postgres, um bom paralelo para os índices de bancos de dados são os índices de um livro: te direcionam para partes comumente procuradas do livro, sem a necessidade de buscar página a página pelo que procura.

O trabalho que temos pode parecer um pouco mais complexo que o exemplo do livro: o autor geralmente aponta para capítulos e subcapítulos no índice, no banco de dados precisamos entender e conhecer os comportamentos do sistema para definir índices indispensáveis, importantes e custosos.

## Como funciona e nuances basicas

Assim que adicionamos um índice, o banco fará o trabalho: utilizar o índice para aumentar a performance de leituras buscando no índice inicialmente, e escrever o índice a cada escrita na tabela. Sem ação necessária, para escrita ocorre um write-aside natural, e para leituras o índice se torna primeiro ponto de contato.

Os índices ajudarão também em comandos de UPDATE e DELETE que utilizem a coluna (ou colunas) com índice para filtro, assim como melhorá a performance em JOINs, caso a tabela relacionada tenho índice para busca. Exemplo: se buscamos planos do usuário pela referência de usuário, ao buscar a relação com usuário na tabela de usuários, o índice na primary key será utilizado.

Nesse artigo vamos focar principalmente em índices B-tree, tipo usado por padrão no Postgres caso não especifique o tipo, e funciona bem para comparações ( >, = <, >= ).

### B-tree

Por baixo dos panos temos na verdade "duas tabelas". Aspas aqui por quê é mais uma representação do que entendemos por tabela, não exatamente a definição de banco de dados. A tabela do banco de dados responsável por armazenar os dados é chamada de "heap table", e é de leitura lenta. Os indices geralmente são referidos como indices, relacionamento de indices, index table, e são de leitura rápida e armazenam uma referência a tupla que o dado está armazenado (na heap table), junto ao valor da chave indexada.

A referência da tupla é chamada de TID (Tuple ID), e armazena um endereço físico da tupla na heap table, por meio do formato que representa o endereço físico do bloco e posição da tupla no bloco: 

```
ctid = (block number, tuple offset)
```

Não adentrando muito, apenas como curiosidade, no código C do Posgres, o TID é representado por uma struct nomeada `ItemPointerData` que é um struct que armazena o número do bloco e o offset da tupla dentro do bloco.

Pegando o código diretamente do repositório oficial do Postgres, temos:

```c
typedef struct ItemPointerData
{
	BlockIdData ip_blkid;
	OffsetNumber ip_posid;
}
```

Isso quer dizer que ao definirmos um índice, sempre que buscarmos algo da tabela, a engine utilizará a index table para encontrar o endereço da tupla, e o banco consegue fazer uma leitura direta ao endereço.

Utilizando o mesmo exemplo da assinatura em um SaaS, teremos:

UserID = referencia do usuário na tabela de relacionamento com plano, então caso o valor seja "abc-123", o banco irá buscar na index table o valor "abc-123" e retornar o endereço da tupla na heap table, exemplo: `ctid = (110, 3)`, onde:

- 110 = número do bloco em que a tupla está
- 3 = posição da tupla no bloco 

Com isso, a engine sabe que precisa do dado que está localizado no ctid = (110, 3), e essa busca será muito mais performática que a busca sequencial na heap table.

## Contraponto e problemas

Índices são fundamentais, entregam de forma prática uma solução para otimização de buscas, mas mesmo o baixo custo cobrado na escrita pode ser problemático em sistemas que têm escrita e leitura críticos. Então precisamos ter uma boa estratégia para índices, para não haver crescimento desenfreado (índices a cada coluna, relacionamento), ao mesmo tempo que não deixamos pontos críticos de leitura sem índice.

Nem tudo precisa nascer com índice, meus sistemas mais enxutos (no quesito quantidade de dados) performam melhor sem índice nos relacionamentos que meus sistemas mais antigos (com profundidade de dados) com índices nos relacionamentos. Isso é natural, os índices resolvem um problema que não é tão visível em tabelas com quantidade reduzida de dados.

### Se guiar por comportamentos do usuário

Com crescimento de dados do sistema, devemos observar o comportamento esperado da nova coluna/tabela em relação ao que temos hoje, daí conseguimos seguir uma regra simples:

- algum ator forte do sistema interage com essa tabela? Em caso positivo, devemos ter índice no relacionamento com o ator forte, pois será massivamente lida por esse filtro.

Com "ator forte", podemos pensar em tabelas como: usuário (B2C), colaborador (B2B), mas conseguimos identificar pela quantidade de dados X quantidade de leituras na tabela.


## Solução

Para solucionar alguns problemas apresentados, e nos mantermos resguardados nas decisões ao longo do tempo, precisamos observar métricas de aplicação e bancos de dados.

### Métricas de aplicação

Para não deixar brechas em tabelas antigas do sistema, vamos usar observabilidade e algumas métricas de SRE para nos ajudar. Aqui vai ser importante ter algo que te forneça dados de consumo das suas aplicações, exemplo: Cloudwatch, Prometheus, para nos entregar nossos dados de análise a nível de endpoint:

```
Maior quantidade de requisições X maior p90
```

Geralmente nesse endpoint vamos encontrar algum dos pontos a seguir, e muitas vezes a combinação de mais de um deles:

- conexões e comunicação sequencial (desnecessária ou não)
- fluxo não atômico, ou seja, conseguimos dividir em múltiplos endpoints
- tabela de banco de dados com muito registro
- tabela de banco de dados sem índice

Desses, 3 são problemas arquiteturais e de software que nosso time de desenvolvimento vai ter que se debruçar para resolver/entender.
O primeiro problema de banco de dados, por mais que arquitetural e deverá ser resolvido com essa visão, acarretará em aumento nos outros casos.
Já o último problema é nosso caso de falta de índice, uma tabela está sendo sequencialmente lida até o dado ser encontrado, aumentando o tempo de leitura. Caso combinado ao problema anterior, muitas linhas na tabela, a leitura sequencial ficará cada vez mais visível ao usuário final.

Devemos colocar um índice nessa tabela do exemplo? Isso você saberá dizer por conhecer as necessidades do seu sistema:

- se uma escrita extremamente rápida é importante, 1-2% de tempo de processamento a mais para escrever o índice pode não fazer sentido, talvez você escolha manter esse aumento de tempo de processamento em um de seus endpoints mais utilizados acima do p90. 
- se a leitura é mais importante, podemos aceitar perder um pouco de performance na escrita e reduzir o tempo de leitura nesse endpoint.

### Métricas de bancos de dados

Assim como temos que ponderar o comportamento do nosso usuário e a saúde dos processos do nosso sistema, não podemos esquecer que banco de dados é em essência software. Precisamos observar e garantir a saúde dele também. 

Podemos determinar que 1-2% de tempo de processamento não é um custo alto para nosso caso de negócio. Mas em uma tabela com muitos dados, muitos acessos concorrentes, teremos uma alta pressão na CPU do nosso banco de dados, e o acréscimo, por mais que baixo, na escrita de uma tabela em um fluxo altamente acessado pode ser um ponto que sature seu banco de dados.

## Conclusão

Índices são de extrema importancia, mas pode envolver alguns trade-offs. Conhecer nosso sistemas a nível de fluxos de negócio e métricas de utilização será de extrema importância para tomar as melhores decisões. 
Ferramentas como Prometheus e Cloudwatch podem te ajudar com o levantamento de métricas do sistema e gargalos de performance, agentes de AI podem te ajudar com analise inicial de endpoints e analise temporal de migrations para acelerar o entendimento de detalhes do sistema. Nem sempre precisamos de um indice, mas não deixe seu usuário reparar a falta dele em locais indispensáveis, e nem possibilite lentidão de processos sequênciais por leituras não performáticas.
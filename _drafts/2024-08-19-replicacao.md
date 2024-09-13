---
layout: post
image: assets/images/system-design/sharding-capa.png
author: matheus
featured: false
published: true
categories: [ system-design, engineering, cloud ]
title: System Design - Replicação de Dados
---

# Definindo Replicação na Engenharia de Software

Replicação, principalmente dentro nos requisitos de engenharia, se refere ao ato de criar uma ou mais cópias do mesmo dado em destinos diferentes. Essa é uma prática bem vista e bem vinda, especialmente em sistemas distribuídos, onde a consistência, disponibilidade e tolerância a falhas são requisitos mandatórios para uma operabilidade saudável e duradoura. Quando olhamos para [Bancos de Dados](/teorema-cap/), a replicação permite que mesmo em caso de falhas terminais de hardware ou problemas de rede, os dados permaneçam acessíveis em outros locais e dão a garantia de que o sistema se tornará consistente em algum momento. 

Essas réplicas podem estar localizadas em servidores diferentes, em datacenters separados geograficamente ou até mesmo em diferentes regiões de nuvens públicas. A finalidade principal da replicação é garantir que os dados estejam disponíveis em vários locais, o que é crítico para sistemas que exigem alta disponibilidade e continuidade de negócios.

Os beneficios de estratégias de replicação são vários, como por exemplo, ao replicar dados em vários locais, um sistema pode continuar a operar mesmo que uma parte do sistema falhe. Em caso de um cluster de databases um nó do mesmo falhar, as réplicas dos dados em outros nós podem assumir e se tornarem a fonte principal da consulta, garantindo que o serviço continue disponível.

<br>

# Modelos de Replicação 

## Replicação Primary-Replica

Na Replicação Primary-Replica, podemos presumir a existência de um nó primário que recebe todas as operações de escrita e, em seguida, replica essas operações para um ou mais nós secundários, suas replicas. As réplicas geralmente são usadas apenas para leitura, enquanto todas as operações de escrita são gerenciadas pelo nó primário. Essa arquitetura é útil quando se deseja escalar leituras em um sistema, distribuindo-as entre réplicas, mas mantendo a simplicidade no gerenciamento de escrita. 

O nó primário é responsável por garantir a consistência dos dados em todas as réplicas. Essa abordagem pode ser eficiente em cenários de alta demanda de leitura, ou quando fazemos um uso intensivo de [CQRS](/cqrs/), mas cria um ponto único de falha no nó primário. Se o nó primário falhar, um novo primário deve ser promovido de uma das réplicas, o que pode introduzir um tempo de inatividade até que essa atividade seja concluída. 

## Replicação Primary-Primary - Multimaster

A Replicação Primary-Primary, também conhecida como Multi-Master Replication, é uma arquitetura onde múltiplos nós podem atuar simultaneamente como primários, recebendo tanto operações de leitura quanto de escrita. Nessa configuração, qualquer nó pode processar atualizações e as mudanças são replicadas entre todos os nós, permitindo alta disponibilidade e escalabilidade de escrita. 

![Replicacao Multiprimary](/assets/images/system-design/replicacao-multi-primary.png)

Esse modelo reduz o ponto único de falha da replicação Primary-Replica e permite maior flexibilidade na distribuição de carga de trabalho. No entanto, ele também introduz complexidade adicional, especialmente para resolver conflitos de escrita. Quando duas operações de escrita ocorrem em diferentes nós primários simultaneamente, o sistema precisa de uma estratégia para resolver esses conflitos, como basear-se em timestamps de ordem de execução das tarefas, ou ter políticas específicas de resolução de conflitos por conta de [particionamento temporário por falha de rede](/teorema-cap/).

<br>

# Estratégias de Replicacão

Dentro das disciplinas de engenharia, podemos encontrar várias estratégias de replicação, tanto abordagens que se aplicam somente para dados, que é geralmente o foco desse tipo de estratégia devido a importância e a complexidade, tanto quanto para outras abordagens não convencionais como cargas de trabalho completas, domínios de software em cache e etc. O objetivo desse capítulo é exemplificar alguns dos modelos mais utilizados de replicação e explicar suas diferenças, vantagens e desvantagens. 

## Replicação Total e Parcial

A Replicação Total se refere à prática de replicar todos os dados em todos os nós de um sistema. Isso significa que cada nó tem uma cópia completa dos dados. A vantagem da replicação total é que ela maximiza a disponibilidade e resiliência de forma com que qualquer nó possa atender a uma solicitação do cliente caso a escrita seja amplamente permitida. No entanto,como um tradeoff, essa estratégia pode aumentar os custos de armazenamento e a latência de escrita, já que cada nova informação precisa ser replicada e confirmada em todos os nós que compõe um cluster do dado. 

Em contraponto, a Replicação Parcial, por outro lado, distribui apenas uma parte dos dados em cada nó. Assim, cada nó contém apenas uma fração dos dados totais. Esse modelo é eficiente em termos de armazenamento e reduz a latência de escrita, mas aumenta a complexidade na leitura, pois os dados solicitados podem não estar disponíveis localmente e podem exigir comunicação entre nós, fazendo com que o cliente precise fazer queries em mais de um nó, ou deixar para que o sitema de consulta abstraia essa complexidade. Para encontrar o dado entre os nós, é comum implementar algoritmos de [Sharding](/sharding/) como Hashing Consistente. 

## Replicação Sincrona

Na Replicação Síncrona, todas as alterações de todos os dados devem ser replicadas em todos os nós antes que a operação seja considerada bem-sucedida. Isso garante consistência forte entre os nós, ou seja, todos eles terão os mesmos dados em qualquer momento.

![Replicação Sincrona](/assets/images/system-design/replicacao-sincrona.png)

Em uma dimensão onde um cliente precisa salvar uma informação em um cluster de cache, ele envia o dado a ser salvo em uma forma de chave e valor para o endpoint primário de um cluster de cache, que por sua vez se encarrega de distribuir o dado entre todos os nós do mesmo. A solicitação só é finalizada e encerrada para o cliente prosseguir com o restante de suas tarefas quando essa operação é concluída por completo. 

A replicação síncrona tem vantagens em cenários onde a consistência é crítica, como em sistemas de pagamento ou bancos de dados financeiros, onde qualquer discrepância entre os nós pode causar grandes problemas. No entanto, ela pode aumentar a latência, especialmente quando há grandes distâncias entre os nós, ou uma grande quantidade deles. 

## Replicação Assincrona

Na Replicação Assíncrona, as alterações de dados são enviadas um dos nós de um cluster, e replicada para os outros nós de forma eventual, o que significa que a operação pode ser considerada bem-sucedida sem esperar que todas as réplicas tenham sido atualizadas. Isso resulta em maior desempenho nas operações de escrita, pois o sistema não precisa esperar pelas confirmações de todos os nós. Porém pode se aceitar uma inconsistência eventual em uma consulta subsequente, uma vez que o dado possa não ter sido replicado totalmente nos demais nós, e possam existir mais de uma versão do dado existindo ao mesmo tempo até que a replicação termine por completo. 

![Replicação Assincrona](/assets/images/system-design/replicacao-assincrona.png)

A replicação assíncrona é amplamente utilizada em cenários onde a disponibilidade e o desempenho são mais importantes que a consistência imediata, como redes sociais, assets em uma CDN, dados pouco importantes utilizados somente para evitar excesso de acessos a uma origem, clusters de cache e afins. 

## Replicação Semi-Sincrona

A Replicação Semi-Síncrona combina aspectos da replicação síncrona e assíncrona. Neste modelo, pelo menos uma ou um pequeno subconjunto de réplicas deve confirmar a gravação de dados antes que a operação seja considerada bem-sucedida. As demais réplicas podem ser atualizadas de forma assíncrona.

![Replicação Semi-Sincrona](/assets/images/system-design/replicacao-semi-sincrona.png)

Esse tipo de replicação oferece um equilíbrio entre consistência e desempenho. Ele melhora a resiliência, garantindo que os dados sejam gravados em pelo menos um nó de forma síncrona, enquanto mantém a baixa latência ao não exigir que todas as réplicas estejam atualizadas imediatamente. Alguns flavors de bancos de dados, como o MySQL e MariaDB, a escrita é confirmada assim que um nó secundário grava os dados. Outros nós podem receber as atualizações posteriormente de forma assíncrona, garantindo um grau adicional de consistência sem comprometer a performance por completo.

## Replicação por Logs

A Replicação por Logs é uma abordagem em que todas as operações ofetuadas em um sistema são registradas em um log de operações sequenciais, e esse log é então replicado para outros nós do cluster para executarem as mesmas operações. Em vez de replicar o estado completo dos dados, o sistema replica as mudanças, permitindo que as réplicas apliquem essas mudanças localmente e mantenham seus dados consistentes.

![Replicação por Logs](/assets/images/system-design/replicacao-logs.png)

Essa abordagem é vantajosa em cenários onde as alterações são mais frequentes que leituras, ou onde o volume de dados é muito grande, pois apenas as modificações são replicadas, reduzindo a quantidade de dados trafegados entre um ponto a outro de um cluster. Esse tipo de abordagem pode ser encontrado em tecnologias que permitem interoperabilidade entre multiplas regiões de núvens públicas, multiplos datacenters, zonas de recuperação de desastre e afins. 

Tecnologias altamente conhecidas e maduras como o [Apache Kafka ou outras tecnologias de streaming e eventos](/mensageria-eventos-streaming/) usa replicação por logs em sua arquitetura de nós e replicas. Cada tópico em Kafka é composto por múltiplas partições, e as alterações nessas partições são registradas em logs de transações que são replicados entre os brokers, garantindo durabilidade e resiliência. 


<br>

# Arquitetura

<br>

## Replicação de Domínios


<br>

## Replicação de Cargas de Trabalho


### Referências

[Replication Strategies and Partitioning in Cassandra](https://www.baeldung.com/cassandra-replication-partitioning)
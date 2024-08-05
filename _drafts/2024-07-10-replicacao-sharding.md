---
layout: post
image: assets/images/system-design/escalabilidade-capa.png
author: matheus
featured: false
published: true
categories: [ system-design, engineering, cloud ]
title: System Design - Sharding e Hashing Consistente
---

# Definindo Sharding

Sharding, ou Partição, é uma técnica de divisão de grandes conjuntos em várias partes de conjuntos menores. Essas partes são cosideradas um shard, ou uma partição, de um todo. Esse todo, é frequentemente associado a dados por ser a abordagem mais comum de utilizar partições, porém não se limita a esse tópico nas disciplinas de engenharia. 

![Sharding Definição](/assets/images/system-design/sharding-definicao.png)

Usando dados como exemplo, cada shard é um subconjunto do banco de dados original e pode ser armazenado em diferentes servidores ou nós de um sistema distribuído. Esta abordagem permite que os dados sejam distribuídos e gerenciados de maneira eficiente, melhorando a escalabilidade, a performance e a disponibilidade do sistema que outrora agruparia e centralizaria todos os dados em um único ponto. 

<br>

# Escalabilidade e Performance 

A importância do sharding em sistemas distribuídos está principalmente n**a necessidade de lidar com grandes volumes de dados** e garantir que o **sistema possa escalar horizontalmente, um dos pontos mais críticos de escala, que são os databases**.

Ao dividir os dados em múltiplos shards, **cada shard pode ser armazenado e gerenciado em servidores diferentes**. Isso permite que mais capacidade seja adicionada ao sistema sem a necessidade de reestruturar a base de dados original.

Com os dados divididos em shards menores, as **operações de leitura e escrita podem ser distribuídas entre diferentes capacidades computacionais**. Isso **reduz a carga em qualquer servidor individual**, resultando em tempos de resposta mais rápidos e melhor performance geral.

Além disso, **o sharding pode contribuir para a alta disponibilidade do sistema**. Se um servidor que contém **um shard específico falhar, os outros shards ainda estarão disponíveis**, permitindo que o sistema continue a operar com funcionalidades reduzidas, em vez de ocorrer uma falha total.

<br>

# Estratégias e Aplicações de Sharding

## Sharding Keys 

Quando pensamos em uma estratégia de particionamento de dados para resolver problemas de escalabilidade, a primeira pergunta que devemos fazer é: **"Particionar baseado em quê?"**. Definir como vamos dividir os dados de um determinado contexto é o passo mais importante, antes de qualquer escolha de tecnologia. **Ao definir uma dimensão de corte para o particionamento, encontramos nossa sharding key**.

A **sharding key, ou chave de partição, é a chave utilizada como critério para determinar como e em qual partição os dados serão armazenados**. A shard key deve ter alta cardinalidade para **garantir uma distribuição uniforme dos dados e deve ser baseada em campos frequentemente acessados, como datas, identificadores, categorias, etc**.

Sharding keys comuns podem incluir as iniciais de um identificador de cliente, o ID de uma entidade, o hash de um valor comum e categorias. Por exemplo, em um sistema financeiro, **é comum dividir a base de clientes entre Pessoas Físicas e Pessoas Jurídicas**. Instituições bancárias podem realizar **shardings baseados em ranges de agências**. Em sistemas de vendas ou logística, **dividir a base por intervalo de datas em que as transações ocorreram** pode ser uma alternativa de escalabilidade, utilizando sharding keys como meses ou anos. Em sistemas multi-tenant, é possível **particionar baseado no hash de um identificador do tenant**.

Existem várias estratégias e aplicações para definir quais sharding keys escolher para a distribuição de dados. Iremos explorar algumas adiante.

## Sharding por ranges de iniciais

Uma estratégia, não tão efetiva, mas ótima para ilustrar a estratégia de sharding é ilustrar um exemplo de distribuição de uma base de usuários, clientes ou tenants baseado na inicial. Podemos **definir a distribuição dos dados entre intervalos de iniciais das sharding keys**, como por exemplo **utilizando intervalos de A-E para um shard, F-J para outro, K-N, O-R, S-V e W-Z consecutivamente**. 

![Sharding Letras](/assets/images/system-design/sharding-letras.png)

Embora seja o exemplo mais simples de ilustrar uma distribuição de dados entre partições, encontramos um dos problemas que o sharding conceitualmente tende a evitar, **que são as hot-partitions, ou partições quentes, onde teremos um outlier de uso entre os shards**. Para complementar o exemplo, em um caso de distribuição baseada em iniciais de um cliente, **podemos presumir por inferência que existem mais Anas, Brunos, Carlos e Danielas do que Wesleys, Yasmins e Ziraldos**. Nesse caso, em um curto médio prazo teremos um **desbalanceamento de performance** muito grande entre a partição 1 e 6, onde a 1 seria superutilizada enquanto a 6 viveria em sub-utilização.

## Sharding por Ranges de Identificadores

Estabelecer uma estratégia de distribuição onde os dados são divididos baseados em intervalos contínuos de valores da sharding key também é uma estratégia muito comum quando olhamos para o mercado. Uma distribuição sequencial requer um controle maior de governança onde acabamos por ter um fenômeno de "transbordo", pois pode ser traçado um paralelo onde shardings podem estar "cheios" e outros "vazios". 

No mais, a estratégia consiste na ideia em que cada shard contém um intervalo específico de valores, e as consultas são direcionadas ao shard apropriado com base na sharding key. Esta abordagem é particularmente útil quando os dados podem ser ordenados de forma natural, ou não e as consultas frequentemente envolvem intervalos de valores. 

![Sharding Range](/assets/images/system-design/sharding-range.png)

Imagine que temos uma base de 10.000 usuários que foram ordenados de forma sequencial durante a sua criação. Após supostas análises, foi visto que essa base de dados poderia ser particionada em 3 shards, e inclusive suportar a criação de novos usuários. Se levarmos o aspecto sequencial ao pé da letra, teriamos 2 shards "cheios" e um com capacidade ociosa suficiente para suportar o crescimento de usuários da base. 

## Sharding por Ranges de Datas

Utilizar atributos sequenciais é uma das possibilidades quando olhamos para distribuições baseadas em ranges de valores das sharding keys, esse aspecto pode ser reaproveitado por exemplo por ranges de tempo. Dentro deum microserviço de vendas, poderiamos por exemplo definir o sharding por intervalos de datas, em um exemplo mais direto, imagine que temos uma base de dados para comportar as transações que ocorreram dentro de cada ano. A longo prazo teriamos uma base de dados que seria responsável por agrupar todas as transacões do ano. 

![Sharding Ano](/assets/images/system-design/sharding-ano.png)

Nesse sentido poderiamos aplicar uma outra estratégia que normalmente se aplicam em shardings que é ter vários "tiers" de storage dos dados, deixando opções mais caras e performáticas para o ano corrente e ano anterior em tier "hot", ter um tier intermediário "warm" para anos que ainda tem acesso frequente mas sem a mesma intensidade que os anos acessados em meior volume e uma opção de tier mais barata e menos performática em "cold" para armazenar os dados de vendas de anos muito anteriores que são acessados esporádicamente. 
 

## Sharding por Hashing

O Sharding por Hashing é uma técnica de particionamento de dados ou computação onde uma função hash é aplicada sobre a Shard Key e o resultado é utilizado para decidir onde cada dado será armazenado, ou o cliente será roteado. Essa função converte o valor do atributo em um valor de hash que deve resultar em um número inteiro. O valor de hash é então mapeado para um dos shards disponíveis usando uma operação de módulo (`mod`), que retorna o resto da divisão de um número por outro. Por exemplo, se o valor de hash for 15 e houver 3 shards, a operação `15 % 3` resultará em um mod 0, indicando que o registro deve ser armazenado no shard 0. Caso o valor do hash for 10, a operação `10 % 3`, o modulo retornará 1, o que significa que o cliente será alocado no shard 1. 

![Hash function](/assets/images/system-design/sharding-hash.png)

#### Exemplo de Balanceamento por Hash Functions

No exemplo, vamos imaginar um sistema multi-tenant que atende vários cenários de negócio. Foi visto que o identificador do tenant seria a melhor shard key para distribuir os clientes de forma total entre os shards. Nesse caso, para descobrir em qual shard o cliente será alocado, podemos aplicar um algoritmo de sha256 para criar uma hash do valor, e em seguida converter o hash para um inteiro. Com base nesse inteiro, aplicamos a função de modulo pelo número de shards disponíveis e o resultado será o shard no qual o tenant será alocado. 

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"strings"
)

// Calcula o hash SHA-256 do valor do tenant e o converte para um inteiro
func hashTenant(tenant string) int {

	// Converte o valor do tenant para minúsculas
	tenant = strings.ToLower(tenant) 

	// Converte a string tenant para um byte slice e calcula o hash.
	hash := sha256.New()
	hash.Write([]byte(tenant))
	hashBytes := hash.Sum(nil)

	// Converte o hash para um número inteiro
	hashInt := binary.BigEndian.Uint64(hashBytes)
	
	// Converte o valor para um valor positivo caso o hashint venha a ser negativo
	if int(hashInt) < 0 {
		return -int(hashInt)
	}
	return int(hashInt)
}

// Recebe uma string do tenant e o número de shards, retornando o número do shard correspondente
func getShardByTenant(tenant string) int {
	// Número disponível de Shards 
	numShards := 3

	// Calcula o mod do hash baseado no numero de shards
	hashValue := hashTenant(tenant)
	shard := hashValue % numShards
	return shard
}

func main() {
	// Lista de tenants
	tenants := []string{
		"Petshops-Souza",
		"Pizzarias-Carvalho",
		"Mecanica-Dois-Irmaos",
		"Padaria-Estrela-Filial-1",
		"Padaria-Estrela-Filial-2",
		"Padaria-Estrela-Filial-3",
		"Hortifruti-Oba",
		"Acougue-Zona-Leste",
		"Acougue-Zona-Oeste",
		"Acougue-Zona-Norte",
	}

	// Verifica a distribuição dos tenants entre os shards
	for _, tenant := range tenants {
		shard := getShardByTenant(tenant)
		fmt.Printf("Tenant: %s, Shard: %d\n", tenant, shard)
	}
}
```

#### Output
```
Tenant: Petshops-Souza, Shard: 1
Tenant: Pizzarias-Carvalho, Shard: 2
Tenant: Mecanica-Dois-Irmaos, Shard: 1
Tenant: Padaria-Estrela-Filial-1, Shard: 0
Tenant: Padaria-Estrela-Filial-2, Shard: 2
Tenant: Padaria-Estrela-Filial-3, Shard: 1
Tenant: Hortifruti-Oba, Shard: 2
Tenant: Acougue-Zona-Leste, Shard: 2
Tenant: Acougue-Zona-Oeste, Shard: 0
Tenant: Acougue-Zona-Norte, Shard: 1
```

Este esquema de distribuição é simples, intuitivo e funciona bem. Ou seja, até que o número de servidores mude. O que acontece se um dos servidores falhar ou ficar indisponível? As chaves precisam ser redistribuídas para a conta do servidor ausente, é claro. O mesmo se aplica se um ou mais servidores novos forem adicionados ao pool;

## Sharding por Hashing Consistente

Sharding por hashing consistente é uma técnica onde uma função de hash é usada para mapear dados para diferentes shards. Em vez de distribuir os dados uniformemente entre os shards, o hashing consistente distribui os dados de maneira a minimizar o número de movimentos de dados quando os shards são adicionados ou removidos. A função hash é aplicada a partir do valor da Sharding Key, e a partir do algoritmo selecionado, o número do shard é retornado com base no hash da sharding key. 

### Algoritmos de Hashing Consistente



## Sharding Consistente e Gestão de Chaves

![Sharding Key Service](/assets/images/system-design/sharding-hash-consistente-key-service.png)

<br>


<br>

# Problemas Conhecidos

## Hot Partitions

## Balanceamento 

## Extensão do número de shardings

<br>

#### Referencias 

[Sharding pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding)

[Database Sharding](https://www.geeksforgeeks.org/database-sharding-a-system-design-concept/)

[Database Sharding for System Design Interview ](https://dev.to/somadevtoo/database-sharding-for-system-design-interview-1k6b)

[Database Sharding Pattern for Scaling Microservices Database Architecture](https://medium.com/design-microservices-architecture-with-patterns/database-sharding-pattern-for-scaling-microservices-database-architecture-2077a556078)

[Sharding: Architecture Pattern](https://www.linkedin.com/pulse/sharding-architecture-pattern-pratik-pandey/)

[Consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing)

[A Guide to Consistent Hashing](https://www.toptal.com/big-data/consistent-hashing)

[What Is Consistent Hashing?](https://www.baeldung.com/cs/consistent-hashing)

[Shuffle Sharding: Massive and Magical Fault Isolation](https://aws.amazon.com/pt/blogs/architecture/shuffle-sharding-massive-and-magical-fault-isolation/)

[System Design — Consistent Hashing](https://medium.com/must-know-computer-science/system-design-consistent-hashing-f66fa9b75f3f)

[A Crash Course in Database Sharding](https://blog.bytebytego.com/p/a-crash-course-in-database-sharding)
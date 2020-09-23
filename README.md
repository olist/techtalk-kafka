# Techtalk Kafka


Código apresentado durante a TechTalk sobre Kafka.

Abaixo estão listados o passo à passo e comandos utilizados durante a apresentação.

[Link da apresentação](https://www.youtube.com/watch?v=ANQ9CMz82d8) 

### Usando o Docker compose

Você pode subir os serviços utilizando o docker na pasta [twitter-kafka-producer](https://github.com/olist/techtalk-kafka/twitter-kafka-producer)

Depois de clonar o repositório abra a pasta `twitter-kafka-producer` e execute o comando:

```shell script
docker-compose up
```

### Subindo Kafka e RethinkDB manualmente

#### RethinkDB

Instale o RethinkDB via docker


Com o RethinkDB instalado e rodando abra o navegador e acesse `http://localhost:8080` e clique no menu `Data Explorer`.

O link completo é:

```console
http://localhost:8080/#dataexplorer
```

Rode cada linha abaixo, para:

- Criar o banco
```javascript
  r.dbCreate("olist-talk-kafka");
```
- Criar a tabela
```javascript
  r.db("olist-talk-kafka").tableCreate("products");
```
- Inserir um novo produto

```javascript
r.db("olist-talk-kafka").table("products").insert({
  "name": "Jogo De Panelas Antiaderente 04 Peças Verde Classic Panelu",
  "stock": 80,
  "value": "75,90",
});
```
- Listar todos os produtos cadastrados
```javascript
  r.db("olist-talk-kafka").table("products");
```

#### Kafka


Para os exemplos eu utilizamos a imagem [https://hub.docker.com/r/landoop/fast-data-dev](https://hub.docker.com/r/landoop/fast-data-dev). 

Subindo o Kafka

> Certifique que o container do RethinkDB está rodando antes de subir o Kafka

```sh
docker run --rm -p 2181:2181 -p 3030:3030 -p 8081-8083:8081-8083 \
       -p 9581-9585:9581-9585 -p 9092:9092 -e \
       --link rethinkdb:rethinkdb lensesio/fast-data-dev:latest
```

Acesse o terminal do nosso container kafka com o comando 

```sh
docker exec -it fast-data-dev /bin/bash
```

Esse comando irá executar o bash que é o nosso terminal no Linux. A flag `-i` permite mapear a entrada do teclado para o bashs e `-t` reserva o terminal.



### Criando o Connector no Kafka

Para criar o nosso conector iremos utilizar a interface web acessando o endereço `http://localhost:3030`.

1. Clique no link `ENTER` da caixa `connectors`.
2. Clique no botão `NEW`
3. Na coluna de `sources` clique no link `RethinkDB`
4. Copie e Cole o código abaixo na aba `PROPERTIES`

Código para criar o connector `source` do RethinkDB.

```yaml 

name=ReThinkSourceConnector
connector.class=com.datamountaineer.streamreactor.connect.rethink.source.ReThinkSourceConnector
tasks.max=1
connect.rethink.db=olist-talk-kafka
connect.rethink.host=rethinkdb
connect.rethink.port=28015
connect.rethink.kcql=INSERT INTO products SELECT * FROM products

```
Campos importantes:

- `connect.rethink.db` -> Nome do banco de dados criado no RethinkDB
- `connect.rethink.host` -> Nome do `link` que criamos quando subimos o Kafka
- `connect.rethink.port` -> Porta que estamos rodando o RethinkDB
- `connect.rethink.kcql` -> Código para a integração do Kafka connect com o RethinkDB conforme descrito abaixo:

  ```sh
  -- Escreve no topicoA as alterações realizadas na tabelaA
  INSERT INTO tabelaA SELECT * FROM topicoA
  ```

Clique em `CREATE`.

Após esta etapa o Kafka irá começar um rebalanciamento e dentro de alguns minutos o nosso conector já estará funcionando.

O tópico informado no `connect.rethink.kcql` será criado automáticamente.

> Em alguns casos fica travado na página de rebalanciamento.
> Feche a janela e acesse novamente o endereço `http://localhost:3030`

Criando o consumidor:

```sh
kafka-console-consumer --bootstrap-server localhost:9092 --topic products
```

Agora a cada vez que inserimos um produto novo no banco de dados o Kafka Connect por meio do nosso conector irá escrever no tópico `products` e o nosso consumidor irá consumir estes dados.

Super prático, não é?


## Outros exemplos do Kafka

Criando um tópico:

```sh
kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic connect-talk-kafka
```

Criando um produtor:
```sh
kafka-console-consumer --bootstrap-server localhost:9092 --topic connect-talk-kafka

kafka-console-producer --broker-list localhost:9092 --topic connect-talk-kafka

```
No produtor o terminal fica em modo iterativo e tudo que for digitado + tecla `Enter` no produtor será consumido pelo nosso consumidor.


## Exemplo do Kafka com o Twitter 

Crie um tópico para receber os tweets produzidos pelo código `Python`. Este tópico terá 3 partições e uma réplica.

```sh
kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 3 --topic tweets
```

Suba a quantidade de consumidor que quiser com o comando: 

```sh
kafka-console-consumer --bootstrap-server localhost:9092 --topic tweets --group twitter-group
```

Para saber quais consumidores estão consumindo as mensagens execute o comando:

```sh
watch -n 5 kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group twitter-group
```

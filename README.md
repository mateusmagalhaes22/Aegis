# Aegis

Plataforma distribuída de análise de transações financeiras. O Aegis recebe transações, persiste os dados, analisa riscos em tempo real por meio de regras antifraude e atualiza a conta do usuário quando a transação é aprovada.

O projeto é composto por microsserviços Spring Boot independentes, conectados por descoberta de serviços com Eureka e processamento assíncrono com Apache Kafka.

## Visão geral

```text
Cliente
	 |
	 v
API Gateway :8081
	 |
	 +--> Identity MS       (descoberta via Eureka)
	 |
	 +--> Transaction MS    (descoberta via Eureka)
	 |
	 v
Eureka Service Discovery :8761

Transaction MS -- transactions ------------------------> Fraud Engine
Fraud Engine  -- transaction-fraud-analysis ------------> Identity MS
Identity MS   -- transaction-validation ----------------> Transaction MS
```

O fluxo síncrono é usado para autenticação, gerenciamento de usuários e operações HTTP de transações. A validação antifraude e a atualização do saldo acontecem de forma assíncrona por meio de tópicos Kafka.

## Funcionalidades atuais

### Identity MS

O `Aegis_Identity_Ms` é responsável por:

- cadastrar usuários e impedir e-mails duplicados;
- armazenar senhas com BCrypt;
- autenticar por e-mail e senha e emitir tokens JWT HS256;
- consultar, atualizar e excluir usuários;
- criar conta ativa com saldo inicial zero;
- consumir análises antifraude;
- aplicar transações aprovadas ao saldo;
- impedir o processamento duplicado de uma transação;
- publicar o resultado da validação para o Transaction MS.

Rotas:

| Método | Rota | Descrição | Auth |
| --- | --- | --- | --- |
| `POST` | `/auth/register` | Cadastra usuário | Não |
| `POST` | `/auth/login` | Retorna JWT | Não |
| `GET` | `/users` | Lista usuários | JWT |
| `GET` | `/users/{id}` | Busca usuário por UUID | JWT |
| `PUT` | `/users/{id}` | Atualiza usuário | JWT |
| `DELETE` | `/users/{id}` | Exclui usuário | JWT |

### Transaction MS

O `Aegis_Transaction_Ms` cria transações com status `PENDING`, persiste os dados no PostgreSQL, publica novas transações em `transactions`, recebe o resultado pelo tópico `transaction-validation` e atualiza o status para `APPROVED` ou `REJECTED`.

Rotas:

| Método | Rota | Descrição |
| --- | --- | --- |
| `POST` | `/transactions` | Cria transação |
| `GET` | `/transactions` | Lista transações |
| `GET` | `/transactions/{id}` | Busca transação |
| `PUT` | `/transactions/{id}` | Atualiza descrição |
| `DELETE` | `/transactions/{id}` | Exclui transação |

Todas as rotas do serviço exigem JWT.

### Fraud Engine

O `Aegis_Fraud_Engine` consome `transactions`, executa as regras antifraude, salva o resultado e publica a análise em `transaction-fraud-analysis`.

Os resultados são classificados assim:

- `APPROVED`: pontuação igual a zero;
- `SUSPICIOUS`: pontuação maior que zero e menor que `100`;
- `REJECTED`: pontuação maior ou igual a `100`.

Considerando a regra de valor acima da média registrada como `@Bean`, as regras ativas são:

1. **Valor elevado:** acima de `1.000`, `5.000`, `10.000` e `25.000`, atribui respectivamente `20`, `40`, `60` e `80` pontos.
2. **Múltiplas transações:** verifica o mesmo usuário nos últimos dez minutos; uma transação anterior gera `50` pontos, duas ou três geram `80`, e quatro ou mais geram `100`.
3. **Valor acima da média:** compara com a média do usuário nos últimos 90 dias e atribui `60` pontos quando o valor excede três vezes essa média.

As pontuações são somadas, portanto a combinação de regras também pode rejeitar uma transação.

### Service Discovery

O `Aegis_Service_Discovery` executa o Eureka Server na porta `8761`. Ele mantém o registro das instâncias e não se registra como cliente.

### API Gateway

O `Aegis_Api_Gateway` é a entrada HTTP na porta `8081`. Ele descobre serviços pelo Eureka, cria rotas automaticamente, remove o primeiro segmento da URL e usa o Spring Cloud LoadBalancer.

Formato das chamadas:

```text
http://localhost:8081/{service-id}/{rota-do-servico}
```

Os identificadores ficam em minúsculas. Com os nomes atuais, exemplos são:

```text
POST http://localhost:8081/aegisidentity/auth/login
GET  http://localhost:8081/aegisidentity/users
POST http://localhost:8081/aegistransaction/transactions
GET  http://localhost:8081/aegistransaction/transactions/{id}
```

O gateway encaminha a requisição; a autenticação é validada pelo serviço de destino.

## Fluxo de uma transação

1. O cliente cadastra um usuário em `/auth/register`.
2. Faz login em `/auth/login` e recebe um JWT.
3. Envia uma transação com `Authorization: Bearer {token}`.
4. O Transaction MS salva como `PENDING` e publica em `transactions`.
5. O Fraud Engine avalia as regras e publica em `transaction-fraud-analysis`.
6. O Identity MS rejeita transações fraudulentas ou aplica o valor à conta.
7. O Identity MS publica `success` ou `failure` em `transaction-validation`.
8. O Transaction MS atualiza o status final.

## Tópicos Kafka

| Tópico | Produtor | Consumidor | Finalidade |
| --- | --- | --- | --- |
| `transactions` | Transaction MS | Fraud Engine | Solicita análise |
| `transaction-fraud-analysis` | Fraud Engine | Identity MS | Publica análise |
| `transaction-validation` | Identity MS | Transaction MS | Aprova ou rejeita |

O Kafka é esperado em `localhost:9092` por padrão.

## Infraestrutura necessária

- Java 25 para Transaction MS e Fraud Engine;
- Java 21 ou superior para gateway, identidade e discovery;
- PostgreSQL em `localhost:5432`;
- bancos `identity`, `transaction` e `antifraud`;
- usuário PostgreSQL `root` com senha `root`;
- Apache Kafka em `localhost:9092`;
- Eureka em `localhost:8761`.

Os serviços usam `spring.jpa.hibernate.ddl-auto=update`, então as tabelas são atualizadas durante a inicialização.

## Como obter o projeto

```bash
git clone --recurse-submodules https://github.com/mateusmagalhaes22/Aegis.git
cd Aegis
```

Para inicializar submódulos de um clone existente:

```bash
git submodule update --init --recursive
```

Submódulos:

```text
Aegis_Service_Discovery/
Aegis_Api_Gateway/
Aegis_Transaction_Ms/
Aegis_Identity_Ms/
Aegis_Fraud_Engine/
```

## Como executar

Inicie a infraestrutura PostgreSQL e Kafka. Depois abra um terminal para cada serviço e execute nesta ordem.

### 1. Eureka

```bash
cd Aegis_Service_Discovery
./mvnw spring-boot:run
```

Painel: `http://localhost:8761`.

### 2. Identity MS

```bash
cd Aegis_Identity_Ms
./mvnw spring-boot:run
```

### 3. Transaction MS

```bash
cd Aegis_Transaction_Ms
./mvnw spring-boot:run
```

### 4. Fraud Engine

```bash
cd Aegis_Fraud_Engine
./mvnw spring-boot:run
```

### 5. API Gateway

```bash
cd Aegis_Api_Gateway
./mvnw spring-boot:run
```

Os serviços de negócio usam `server.port: 0` e recebem portas aleatórias. Use o gateway quando as instâncias estiverem visíveis no Eureka. No Windows, use `mvnw.cmd` no lugar de `./mvnw`.

## Exemplo de uso

### Cadastro

```bash
curl -X POST http://localhost:8081/aegisidentity/auth/register \
	-H 'Content-Type: application/json' \
	-d '{"name":"Maria Silva","email":"maria@example.com","password":"senha-segura","cpf":"12345678900"}'
```

### Login

```bash
curl -X POST http://localhost:8081/aegisidentity/auth/login \
	-H 'Content-Type: application/json' \
	-d '{"email":"maria@example.com","password":"senha-segura"}'
```

Use o token retornado no cadastro da transação:

```bash
curl -X POST http://localhost:8081/aegistransaction/transactions \
	-H 'Content-Type: application/json' \
	-H 'Authorization: Bearer SEU_TOKEN' \
	-d '{"description":"Compra online","userId":"ID_DO_USUARIO","amount":1500.00}'
```

A criação retorna `PENDING`. Depois do processamento Kafka, consulte a transação para verificar `APPROVED` ou `REJECTED`.

## Configuração por ambiente

| Variável | Padrão | Uso |
| --- | --- | --- |
| `KAFKA_BOOTSTRAP_SERVERS` | `localhost:9092` | Broker Kafka |
| `KAFKA_TRANSACTIONS_TOPIC` | `transactions` | Fluxo de transações |
| `KAFKA_TRANSACTION_VALIDATION_TOPIC` | `transaction-validation` | Resultado da validação |
| `KAFKA_TRANSACTION_FRAUD_ANALYSIS_TOPIC` | `transaction-fraud-analysis` | Resultado antifraude |
| `KAFKA_CONSUMER_GROUP` | definido por serviço | Grupos de consumidores |
| `JWT_SECRET` | valor padrão do projeto | Transaction MS |

Para produção, substitua os segredos padrão por valores externos e fortes. O segredo usado para emitir e validar JWT deve ser compatível entre os serviços.

## Build e testes

Cada submódulo possui Maven Wrapper próprio:

```bash
cd Aegis_Identity_Ms
./mvnw clean test
```

Para testar todos os módulos:

```bash
for service in Aegis_Service_Discovery Aegis_Api_Gateway Aegis_Transaction_Ms Aegis_Identity_Ms Aegis_Fraud_Engine; do
	(cd "$service" && ./mvnw clean test) || exit 1
done
```

Para gerar um JAR:

```bash
./mvnw clean package
java -jar target/*.jar
```

## Limitações e cuidados

- O gateway possui rotas geradas por descoberta, não rotas manuais.
- O segredo JWT padrão não deve ser usado em produção.
- `ddl-auto: update` é adequado para desenvolvimento, não para migrações produtivas.
- Os três tópicos Kafka devem possuir nomes compatíveis em todos os serviços.

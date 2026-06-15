# 🏪 Projeto Final — Arquitetura de Aplicações Web

Plataforma de e-commerce de moda social desenvolvida como projeto final da disciplina de **Arquitetura de Aplicações Web** — Newton Paiva.

O sistema é composto por microsserviços independentes, cada um com responsabilidade bem definida, comunicando-se de forma assíncrona via **Apache Kafka**.

---

## 👥 Equipe

| Nome | Módulo |
|------|--------|
| Gustavo Alves | Category Service |
| Victor Andrey | Auth Service |
| Ana Carolina Muniz | Cart Service |
| Álax Fernando de Freitas Nunes | Inventory Service (Módulo Principal) |
| Ryan Junio Pereira Costa | Email Service/Payment Service |


---

## 🗺️ Visão Geral da Arquitetura

```
Cliente
   │
   ▼
Auth Service ──────────────────────► JWT Token
   │
   ▼
Cart Service ──────► Kafka (order-topic) ──────► Outros Serviços
   │
   ▼
Catalog Service ◄─── Kafka (loja-pedido-criado) ◄─── Serviço Principal
   │
   ├──► grupo-email     → Notification Service (e-mail ao cliente)
   │
   └──► grupo-catalog   → Atualização de estoque no MongoDB
   │
   ▼
Inventory Service ──► PostgreSQL
   │
   ▼
Payment Service ────► Kafka (lojavintage-pedidos) ──► Notification Service
```

---

## 📦 Microsserviços

---

### 1. 🔐 Auth Service

Microsserviço de autenticação responsável pelo cadastro de usuários, login e controle de acesso por perfil.

**Tecnologias:** Java 21 · Spring Boot · Spring Security · JWT · MySQL · Maven

#### Endpoints

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/auth/register` | Cadastro de novo usuário |
| `POST` | `/auth/login` | Login — retorna token JWT |
| `GET` | `/user` | Área do usuário (requer JWT) |
| `GET` | `/admin` | Área administrativa (requer JWT + perfil ADMIN) |

#### Exemplo — Cadastro

```json
POST /auth/register
{
  "nome": "João",
  "email": "joao@email.com",
  "senha": "123456"
}
```

#### Exemplo — Login

```json
POST /auth/login
{
  "email": "joao@email.com",
  "senha": "123456"
}

// Retorna:
{
  "token": "jwt-token"
}
```

#### Banco de Dados

**MySQL** — Tabela `users`

| Campo | Tipo |
|-------|------|
| id | PK |
| nome | String |
| email | String |
| senha | String (hash) |
| role | Enum (USER / ADMIN) |

---

### 2. 🛒 Cart Service

Microsserviço responsável pelo gerenciamento do carrinho de compras, incluindo checkout e publicação de eventos via Kafka.

**Tecnologias:** Java 17 · Spring Boot 3 · Spring Data MongoDB · Apache Kafka · Swagger/OpenAPI · Maven

#### Estrutura do Projeto

```
controller/CartController.java
service/CartService.java
repository/CartRepository.java
model/
  ├── Cart.java
  ├── CartItem.java
  └── Order.java
kafka/
  ├── CartProducer.java
  └── CartConsumer.java
```

#### Endpoints

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/cart` | Criar carrinho |
| `GET` | `/cart/{userId}` | Buscar carrinho do usuário |
| `POST` | `/cart/{userId}/items` | Adicionar item |
| `DELETE` | `/cart/{userId}/items/{productName}` | Remover item |
| `PUT` | `/cart/{userId}/items/{productName}?quantity=3` | Atualizar quantidade |
| `DELETE` | `/cart/{userId}` | Limpar carrinho |
| `POST` | `/cart/{userId}/checkout` | Finalizar compra |

#### Exemplo de documento no MongoDB

```json
{
  "_id": "123",
  "userId": "user01",
  "items": [
    {
      "productName": "Terno Slim Preto",
      "quantity": 2,
      "price": 499.90
    }
  ],
  "total": 999.80
}
```

#### Integração Kafka

| Tópico | Ação |
|--------|------|
| `cart-topic` | Publica informações do carrinho |
| `order-topic` | Publica pedidos finalizados |

O consumer escuta `cart-topic` e exibe usuário, total, produtos, quantidade e preço.

---

### 3. 📦 Catalog Service

Microsserviço responsável pela gestão do catálogo da loja: categorias, produtos disponíveis e vitrines de destaque. Consome eventos Kafka para sincronização de estoque em tempo real.

**Tecnologias:** Java 21 · Spring Boot 3.4.5 · Spring Data MongoDB · Apache Kafka · Spring Cache · Docker

#### Estrutura do Projeto

```
catalog-service/
├── dockerfile
├── docker-compose.yml
├── pom.xml
└── src/main/java/com/example/category/
    ├── model/         (Categoria, Produto, Vitrine)
    ├── repository/    (CategoriaRepository, ProdutoRepository, VitrineRepository)
    ├── service/       (CategoriaService, ProdutoService, VitrineService)
    ├── controller/    (CategoriaController, ProdutoController, VitrineController)
    ├── dto/           (ProdutoDTO, VitrineDTO, PedidoEventDTO)
    └── kafka/         (EstoqueEventConsumer)
```

#### Endpoints — Categorias `/catalog/categorias`

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `GET` | `/catalog/categorias` | Lista categorias ativas |
| `POST` | `/catalog/categorias` | Cria nova categoria |
| `PUT` | `/catalog/categorias/{id}` | Atualiza categoria |
| `DELETE` | `/catalog/categorias/{id}` | Desativa categoria |

#### Endpoints — Produtos `/catalog/produtos`

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `GET` | `/catalog/produtos` | Lista produtos disponíveis |
| `GET` | `/catalog/produtos/categoria/{categoriaId}` | Lista por categoria |
| `GET` | `/catalog/produtos/busca?nome=` | Busca por nome |
| `POST` | `/catalog/produtos` | Cadastra produto |
| `PUT` | `/catalog/produtos/{id}` | Atualiza produto |
| `DELETE` | `/catalog/produtos/{id}` | Remove produto |

#### Endpoints — Vitrines `/catalog/vitrines`

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `GET` | `/catalog/vitrines` | Lista vitrines ativas (com cache) |
| `POST` | `/catalog/vitrines` | Cria vitrine |
| `PUT` | `/catalog/vitrines/{id}` | Atualiza vitrine |
| `DELETE` | `/catalog/vitrines/{id}` | Desativa vitrine e invalida cache |

#### Integração Kafka

O consumer `EstoqueEventConsumer` escuta o tópico `loja-pedido-criado` (group-id `grupo-catalog`). Para cada pedido recebido, atualiza o estoque e o campo `disponivel` de cada produto no MongoDB.

#### Executando

```bash
# Subir tudo com Docker Compose
docker-compose up --build

# Rodar apenas a infraestrutura (para desenvolvimento local)
docker-compose up mongodb kafka zookeeper
./mvnw spring-boot:run
```

> Swagger disponível em: `http://localhost:8080/swagger-ui.html`

---

### 4. 📋 Inventory Service

Microsserviço RESTful responsável pelo controle completo de estoque de roupas sociais masculinas e femininas da plataforma Divenclasse.

**Tecnologias:** Java 17 · Spring Boot 3.3.0 · Spring Data JPA · Hibernate · PostgreSQL 16 · Docker · Swagger/OpenAPI · JUnit 5 · Mockito

#### Endpoints

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/products` | Cadastrar produto |
| `GET` | `/products` | Listar todos (paginado) |
| `GET` | `/products/{id}` | Buscar por ID |
| `GET` | `/products/search?name=` | Buscar por nome |
| `GET` | `/products/category/{c}` | Buscar por categoria |
| `GET` | `/products/size/{s}` | Buscar por tamanho |
| `GET` | `/products/gender/{g}` | Buscar por gênero |
| `GET` | `/products/low-stock` | Produtos com estoque baixo |
| `PUT` | `/products/{id}` | Atualização completa |
| `PATCH` | `/products/{id}/stock` | Atualizar estoque |
| `DELETE` | `/products/{id}` | Excluir produto |

#### Entidade Product

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| id | Long | Auto | Gerado automaticamente |
| nome | String | ✅ | Nome do produto |
| descricao | String | — | Descrição detalhada |
| categoria | Enum | ✅ | Categoria do produto |
| tamanho | String | ✅ | Tamanho |
| cor | String | ✅ | Cor principal |
| preco | BigDecimal | ✅ | Preço (> 0) |
| quantidadeEstoque | Integer | ✅ | Quantidade disponível |
| marca | String | — | Marca |
| genero | Enum | ✅ | MASCULINO / FEMININO / UNISSEX |
| dataCadastro | LocalDateTime | Auto | Imutável após criação |
| dataAtualizacao | LocalDateTime | Auto | Atualizada automaticamente |

#### Categorias disponíveis

```
TERNO · BLAZER · CAMISA_SOCIAL · CALCA_SOCIAL
GRAVATA · SAPATO_SOCIAL · CINTO · COLETE
```

#### Regras de Negócio

- Estoque nunca negativo
- Campo `emEstoque` calculado automaticamente
- Alerta para estoque baixo (≤ 3)
- `dataCadastro` imutável após criação
- Preço obrigatoriamente maior que zero
- Bean Validation em todos os campos obrigatórios
- Paginação em listagens

#### Tratamento de Erros

| Exceção | HTTP |
|---------|------|
| ProductNotFoundException | 404 |
| InsufficientStockException | 422 |
| MethodArgumentNotValidException | 400 |
| Exception | 500 |

```json
// Exemplo de resposta de erro
{
  "timestamp": "2025-05-27T10:30:00",
  "status": 404,
  "error": "Not Found",
  "message": "Produto não encontrado com ID: 5",
  "path": "/products/5"
}
```

#### Testes

```
10 tests passed / 10 total — BUILD SUCCESS
```

Utilizando JUnit 5, Mockito e H2 (banco em memória para testes).

#### Executando

```bash
# Subir containers (PostgreSQL na porta 5432, serviço na porta 8085)
docker compose up --build

# Swagger
http://localhost:8085/swagger-ui.html
```

---

### 5. 💳 Payment Service

Microsserviço responsável pelo processamento de fluxos de pagamento e publicação de eventos para outros serviços reagirem de forma assíncrona.

**Tecnologias:** Java 17 · Spring Boot 3 · Spring Kafka · JUnit 5 · Spring Boot Test

#### Funcionamento

Ao processar um pagamento, o serviço converte os dados do pedido (ID, Nome, E-mail, Produto, Valor e Status) em JSON e publica de forma assíncrona no tópico `lojavintage-pedidos` do Kafka.

#### Teste de Integração

O teste `PaymentServiceApplicationTests` interage diretamente com o ambiente Docker:
- Conecta ao broker Kafka real na porta `9092`
- Dispara um cenário de pagamento **APROVADO** com payload JSON válido
- Garante que o broker receba a mensagem e o Notification Service processe o envio de e-mail de ponta a ponta

---

### 6. 📨 Notification Service

Microsserviço responsável por escutar eventos de pagamento, registrar histórico de transações e disparar e-mails de confirmação via Gmail.

**Tecnologias:** Java 17 · Spring Boot 3 · Spring Kafka · Spring Data MongoDB · Spring Boot Starter Mail (JavaMailSender)

#### Fluxo de Negócio

1. **Consumo de Eventos** — escuta o tópico `lojavintage-pedidos` via Kafka
2. **Persistência** — grava cada pedido no MongoDB (`lojavintage_db`) para auditoria
3. **Disparo de E-mail** — analisa o status do pagamento:
   - `APROVADO` → monta e envia e-mail de confirmação ao cliente via SMTP Gmail
   - Falha / Recusado → registra logs de erro no console

#### Status no Console

```
🟢 STATUS: E-MAIL ENVIADO COM SUCESSO
🔴 STATUS: E-MAIL NÃO ENVIADO (Falha no disparo do servidor)
```

---

## 🔄 Fluxo Completo de uma Compra

```
1. Cliente faz login → Auth Service emite JWT
2. Cliente adiciona produtos → Cart Service gerencia carrinho
3. Cliente finaliza compra → Cart Service publica em order-topic
4. Payment Service processa pagamento → publica em lojavintage-pedidos
5. Notification Service consome evento → salva no MongoDB + envia e-mail
6. Catalog Service consome loja-pedido-criado → atualiza estoque no MongoDB
7. Inventory Service reflete alterações de estoque via PostgreSQL
```

---

## 🛠️ Stack Tecnológica Geral

| Tecnologia | Uso |
|------------|-----|
| Java 17 / 21 | Linguagem principal |
| Spring Boot 3 | Framework base |
| Spring Security + JWT | Autenticação e autorização |
| Apache Kafka | Mensageria assíncrona entre serviços |
| MongoDB | Persistência NoSQL (Cart, Catalog, Notification) |
| PostgreSQL | Persistência relacional (Inventory) |
| MySQL | Persistência relacional (Auth) |
| Docker / Docker Compose | Containerização |
| Swagger / OpenAPI | Documentação das APIs |


## Relatos Pessoais:

# Relato Pessoal – Microserviço de Carrinho de Compras

**Autora:** Ana Carolina Muniz Soares
**Curso:** Análise e Desenvolvimento de Sistemas (ADS)
**Disciplina:** Arquitetura de Dispositivos Móveis e Microserviços
**Professor:** Sândalo

---

## Sobre o Projeto

Neste projeto, participei do desenvolvimento de uma arquitetura baseada em microserviços utilizando Spring Boot, Apache Kafka e MongoDB. Minha principal responsabilidade foi a criação do **Microserviço de Carrinho de Compras**, responsável por armazenar e gerenciar os produtos adicionados pelos usuários.

## Desenvolvimento

Durante a implementação, desenvolvi os endpoints para operações de cadastro, consulta e atualização dos itens do carrinho. Para a persistência dos dados, foi utilizado o **MongoDB**, que ofereceu flexibilidade na modelagem das informações. Além disso, participei da integração com o **Apache Kafka**, permitindo a comunicação assíncrona entre os microserviços por meio de eventos. Essa abordagem tornou a aplicação mais desacoplada, organizada e escalável.

## Desafios e Aprendizados

Um dos principais desafios foi compreender a configuração do Kafka e o fluxo de mensagens entre os serviços. Também foi necessário entender a integração entre o Spring Boot e o MongoDB para garantir a persistência correta dos dados. Com este projeto, aprofundei meus conhecimentos em:

- Arquitetura de Microserviços;
- Apache Kafka;
- MongoDB;
- APIs REST com Spring Boot;
- Docker e Docker Compose.

## Conclusão

A construção do Microserviço de Carrinho de Compras foi uma experiência importante para minha formação acadêmica, permitindo aplicar na prática conceitos modernos de desenvolvimento distribuído e comunicação entre serviços, ampliando minha experiência com tecnologias amplamente utilizadas no mercado.

---

Ana Carolina Muniz Soares
Análise e Desenvolvimento de Sistemas – 2026



# Relato Pessoal – Microserviço de Catálogo (Catalog Service)

**Autor:** Gustavo de Souza Alves
**Curso:** Análise e Desenvolvimento de Sistemas (ADS)
**Disciplina:** Arquitetura de Dispositivos Móveis e Microserviços
**Professor:** Sândalo

---

## Sobre o Projeto

Neste projeto, participei do desenvolvimento de uma arquitetura baseada em microserviços utilizando Spring Boot, Apache Kafka e MongoDB. Minha principal responsabilidade foi a criação do **Microserviço de Catálogo (Catalog Service)**, desenvolvido como uma extensão independente do serviço principal da aplicação. Esse microserviço é responsável por gerenciar as **Categorias**, os **Produtos** disponíveis na loja e as **Vitrines** de destaque, funcionando como a camada de apresentação do catálogo para o cliente final.

## Desenvolvimento

Durante a implementação, desenvolvi os endpoints REST para operações de cadastro, consulta, atualização e desativação dos recursos de Categoria, Produto e Vitrine. Para a persistência dos dados, utilizei o **MongoDB** com banco de dados próprio e isolado (`catalog-db`), seguindo o princípio fundamental de que microserviços não devem compartilhar bases de dados. Implementei também o sistema de **cache com Spring Cache**, aplicado nas vitrines ativas para reduzir consultas desnecessárias ao banco. Outro ponto relevante foi a integração com o **Apache Kafka**, por meio do `EstoqueEventConsumer`, que consome os eventos publicados pelo serviço principal sempre que um pedido é realizado, atualizando automaticamente a disponibilidade dos produtos no catálogo em tempo real.

## Desafios e Aprendizados

Um dos principais desafios foi compreender o funcionamento do Kafka na prática, especialmente a configuração de `group-id` distintos para que o microserviço de catálogo recebesse os mesmos eventos de forma independente do serviço de e-mail. Outro desafio foi a configuração do ambiente com **Docker e Docker Compose**, lidando com conflitos de porta entre os containers e a comunicação interna entre serviços usando os nomes definidos no `docker-compose.yml` no lugar de `localhost`. Com este projeto, aprofundei meus conhecimentos em:

- Arquitetura de Microserviços e isolamento de responsabilidades;
- Apache Kafka e comunicação assíncrona entre serviços;
- MongoDB e modelagem de dados com Spring Data;
- APIs REST com Spring Boot seguindo boas práticas de camadas;
- Cache de dados com Spring Cache (`@Cacheable` e `@CacheEvict`);
- Docker e Docker Compose para containerização de aplicações.

## Conclusão

A construção do Microserviço de Catálogo foi uma experiência fundamental para minha formação acadêmica, pois permitiu aplicar na prática conceitos modernos de desenvolvimento distribuído que são amplamente exigidos pelo mercado. Compreender como serviços independentes se comunicam por eventos, mantêm seus próprios dados e escalam de forma isolada ampliou significativamente minha visão sobre arquitetura de software e me preparou para desafios reais do desenvolvimento profissional.

---

Gustavo de Souza Alves
Análise e Desenvolvimento de Sistemas – 2026
| JUnit 5 + Mockito | Testes unitários e de integração |
| Maven | Gerenciamento de dependências |


# Relato Pessoal – Microserviço de Inventário

**Autor:** Alax Fernando de Freitas Nunes 
**Curso:** Análise e Desenvolvimento de Sistemas  
**Disciplina:** Arquitetura de Dispositivos Móveis e Microserviços  
**Professor:** Sândalo Bessa  

---

## Sobre o Projeto

Neste projeto, participei do desenvolvimento de uma arquitetura baseada em microsserviços utilizando **Spring Boot**, **Apache Kafka**, **PostgreSQL** e outras tecnologias voltadas para aplicações distribuídas. Minha principal responsabilidade foi a criação e desenvolvimento do **Microserviço de Inventory (Estoque)**, responsável por gerenciar o controle de produtos disponíveis, quantidades em estoque e atualização das informações de disponibilidade da loja.

O **Inventory Service** foi projetado para funcionar de forma totalmente independente dos demais componentes da aplicação, possuindo sua própria base de dados e regras de negócio, seguindo os princípios fundamentais da arquitetura de microsserviços. Seu papel é garantir que o estoque permaneça consistente durante todo o fluxo de compras, fornecendo informações atualizadas para os demais serviços da plataforma.

---

## Desenvolvimento

Durante a implementação, desenvolvi os endpoints REST responsáveis pelas operações de cadastro, consulta, atualização e gerenciamento dos produtos armazenados no estoque. O serviço foi estruturado utilizando boas práticas de desenvolvimento com Spring Boot, separando as camadas de **Controller**, **Service**, **Repository** e **Model**, proporcionando maior organização, manutenibilidade e reutilização do código.

Para a persistência dos dados foi utilizado o **PostgreSQL**, mantendo uma base exclusiva para o serviço de estoque, evitando compartilhamento direto de informações com outros microsserviços e respeitando o princípio de autonomia de dados.

Também participei da integração do **Inventory Service** com os demais componentes da arquitetura por meio de eventos, permitindo que alterações realizadas durante o processo de compra atualizem automaticamente a disponibilidade dos produtos e mantenham a sincronização entre catálogo, carrinho e estoque.

Além disso, foram implementadas validações para impedir inconsistências, como:

- Controle de quantidades negativas;
- Atualização automática da disponibilidade dos produtos;
- Validação de estoque antes da confirmação de pedidos;
- Proteção contra alterações inválidas;
- Gerenciamento seguro das operações de entrada e saída de estoque.

---

## Desafios e Aprendizados

Um dos principais desafios encontrados durante o desenvolvimento foi compreender a comunicação entre microsserviços e a sincronização de informações em tempo real. Garantir que o estoque refletisse corretamente as operações realizadas pelos demais serviços exigiu atenção especial à arquitetura orientada a eventos e ao desacoplamento entre aplicações.

Outro desafio importante foi configurar o ambiente utilizando **Docker** e **Docker Compose**, garantindo que todos os serviços e bancos de dados se comunicassem corretamente sem conflitos de portas ou dependências.

Com este projeto, aprofundei meus conhecimentos em:

- Arquitetura de Microsserviços e separação de responsabilidades;
- Desenvolvimento de APIs REST com Spring Boot;
- Persistência de dados utilizando PostgreSQL e Spring Data JPA;
- Comunicação assíncrona entre serviços utilizando Apache Kafka;
- Integração entre microsserviços baseada em eventos;
- Organização de projetos em arquitetura em camadas;
- Conteinerização de aplicações com Docker e Docker Compose;
- Gerenciamento de dependências utilizando Maven;
- Controle de versões com Git e GitHub.

---

## Conclusão

O desenvolvimento do **Microserviço de Inventory** representou uma experiência extremamente enriquecedora para minha formação acadêmica e profissional. A oportunidade de construir um serviço independente, integrado aos demais componentes da aplicação e responsável por uma área crítica do sistema permitiu consolidar conhecimentos importantes sobre desenvolvimento distribuído e arquitetura moderna de software.

Além do aprendizado técnico, o projeto reforçou a importância da colaboração entre equipes, da padronização de interfaces e da comunicação eficiente entre microsserviços para construção de aplicações escaláveis, robustas e de fácil manutenção.

Considero que essa experiência ampliou significativamente minha visão sobre o desenvolvimento de sistemas corporativos e me preparou para enfrentar desafios reais do mercado de tecnologia, especialmente em projetos que utilizam arquiteturas baseadas em microsserviços.

---

## Tecnologias Utilizadas

- Java 17
- Spring Boot
- Spring Data JPA
- PostgreSQL
- Apache Kafka
- Maven
- Docker
- Docker Compose
- APIs REST
- Git
- GitHub

---

## Autor

**Alax Fernando de Freitas Nunes**  
*Análise e Desenvolvimento de Sistemas – 2026*

**Especialização no projeto:**

- 📦 Inventory Service
- 🗄️ PostgreSQL
- ☕ Spring Boot
- 🔄 Apache Kafka
- 🐳 Docker & Docker Compose
- 🌐 APIs REST
- 🏗️ Arquitetura de Microsserviços
- 📚 Maven

# Diagramas do Sistema de Biblioteca

Este documento descreve os três diagramas de modelagem do projeto: o **diagrama de atores** (visão macro do sistema), o **diagrama ER** (modelagem do banco de dados) e o **diagrama de sequência** (fluxo de registro de empréstimo).

---

## 1. Diagrama de Atores e Módulos

### Objetivo

Representar quem interage com o sistema e quais são os grandes módulos funcionais, antes de entrar em detalhes de banco de dados ou API.

### Elementos

**Ator**
- **Bibliotecário/Admin** — único ator do sistema (conforme decidido no levantamento de requisitos). Acessa todos os módulos após autenticação.

**Módulos do sistema**
| Módulo | Responsabilidade |
|---|---|
| Pessoas | Cadastro, edição, listagem e inativação de pessoas (RF01, RF02) |
| Livros | Cadastro, edição, listagem e controle de exemplares (RF03, RF04) |
| Empréstimos | Registro de empréstimo/devolução e cálculo de multa (RF05–RF08, RN01–RN03) |
| Reservas | Fila de espera, prazo de retirada e expiração sob demanda (RF09–RF11, RN04–RN08) |

**Autenticação (JWT)** — camada transversal que protege o acesso a todos os módulos (RS01–RS07). Todo módulo passa por ela antes de processar qualquer requisição do Bibliotecário.

### Por que só um ator

Como definido no levantamento de requisitos, o sistema é de uso interno — não existe perfil de "usuário comum" consultando o acervo publicamente. Isso simplifica a autorização: não há necessidade de `@PreAuthorize` com múltiplas roles, apenas autenticação (logado ou não logado).

### Relação entre módulos

- **Empréstimos** depende de **Pessoas** e **Livros** (chaves estrangeiras).
- **Reservas** depende de **Pessoas** e **Livros** também, e se comunica indiretamente com **Empréstimos**: toda devolução (módulo Empréstimos) verifica a fila de Reservas para o mesmo livro.

---

## 2. Diagrama ER (Entidade-Relacionamento)

### Objetivo

Modelar as tabelas do banco de dados PostgreSQL, seus campos, tipos e relacionamentos, servindo de base direta para as migrations Flyway.

### Entidades

**PESSOA**
| Campo | Tipo | Observação |
|---|---|---|
| id | bigint | PK |
| nome | string | |
| cpf | string | UK — evita cadastro duplicado |
| email | string | |
| telefone | string | |

**LIVRO**
| Campo | Tipo | Observação |
|---|---|---|
| id | bigint | PK |
| titulo | string | |
| autor | string | |
| isbn | string | UK |
| quantidade_total | int | |
| quantidade_disponivel | int | Decrementada no empréstimo, incrementada na devolução |

**EMPRESTIMO**
| Campo | Tipo | Observação |
|---|---|---|
| id | bigint | PK |
| pessoa_id | bigint | FK → Pessoa |
| livro_id | bigint | FK → Livro |
| data_emprestimo | date | |
| data_prevista_devolucao | date | |
| data_devolucao_real | date | Nulo enquanto o empréstimo está ativo |
| valor_multa | decimal | Nunca `double` — evita erro de arredondamento monetário |
| status | string (enum) | ATIVO, ATRASADO, DEVOLVIDO — calculado, nunca setado manualmente (RN09) |

**RESERVA**
| Campo | Tipo | Observação |
|---|---|---|
| id | bigint | PK |
| pessoa_id | bigint | FK → Pessoa |
| livro_id | bigint | FK → Livro |
| data_reserva | timestamp | Define a ordem da fila (RN05) |
| prazo_retirada | timestamp | Preenchido quando o exemplar fica disponível para a pessoa |
| status | string (enum) | AGUARDANDO, DISPONIVEL, RETIRADA, EXPIRADA, CANCELADA |

### Relacionamentos

- **Pessoa 1:N Emprestimo** — uma pessoa pode ter vários empréstimos ao longo do tempo
- **Pessoa 1:N Reserva** — uma pessoa pode ter várias reservas (mas não duplicadas para o mesmo livro — RN08)
- **Livro 1:N Emprestimo** — um livro (via seus exemplares) pode gerar vários empréstimos
- **Livro 1:N Reserva** — um livro pode ter uma fila de várias pessoas aguardando

### Decisões de modelagem

1. **`status` como enum, não como texto livre** — mapeado no JPA com `@Enumerated(EnumType.STRING)`. Isso grava o nome do valor no banco (ex: `"ATIVO"`), não um índice numérico — protege contra corrupção de dados se a ordem do enum mudar no código.
2. **`valor_multa` como `decimal`** — tipos de ponto flutuante (`double`, `float`) têm erro de arredondamento; para valores monetários, `decimal`/`BigDecimal` é obrigatório.
3. **Datas de empréstimo em `date`, datas de reserva em `timestamp`** — o prazo de retirada de uma reserva é medido em horas (ex: 48h), então precisa de granularidade de hora. O ciclo de empréstimo é tratado em dias corridos.
4. **Sem campo de imagem/capa no Livro** — decisão registrada no levantamento de requisitos (ver RF13 nas melhorias futuras).

---

## 3. Diagrama de Sequência — Registro de Empréstimo

### Objetivo

Mostrar a ordem temporal das interações entre as camadas do sistema durante a operação mais rica em regras de negócio: registrar um empréstimo. Enquanto o diagrama ER mostra *o que existe* e o de atores mostra *quem acessa o quê*, o diagrama de sequência mostra *o que acontece, passo a passo, quando o Bibliotecário aperta "confirmar"*.

### Participantes

| Participante | Papel |
|---|---|
| Bibliotecário | Inicia a ação na interface |
| Frontend | Angular — envia a requisição HTTP |
| API | Controller do Spring Boot — recebe a requisição, delega ao Service |
| Service | Camada de regras de negócio — nunca o Controller (RNF03) |
| Banco de dados | PostgreSQL — consultado e atualizado pelo Service via Repository |

### Fluxo

1. O Bibliotecário registra o empréstimo na tela do Angular
2. O Frontend envia `POST /emprestimos` para a API
3. O Controller delega toda a validação ao Service (nunca valida regra de negócio na própria camada de Controller)
4. O Service consulta o banco em duas etapas de validação, **antes** de qualquer escrita:
   - A pessoa tem multa pendente? (RN02) — se sim, interrompe aqui
   - O livro tem exemplar disponível? (RN03) — se não, interrompe e sugere reserva (RF09)
5. Só depois de ambas as validações passarem, o Service grava: decrementa `quantidade_disponivel` do livro e insere o novo registro de `Emprestimo` com status `ATIVO`
6. A resposta sobe de volta pela mesma cadeia até a confirmação aparecer para o Bibliotecário

### Por que essa ordem importa

O ponto crítico do diagrama é que **as duas validações acontecem antes de qualquer escrita no banco**. Se a validação de disponibilidade viesse depois de decrementar a quantidade, um cenário de concorrência (dois empréstimos do mesmo exemplar quase simultâneos) poderia deixar `quantidade_disponivel` negativa. Esse é o tipo de bug que só aparece sob carga — vale a pena, quando for implementar o Service, considerar uma transação (`@Transactional`) envolvendo a leitura e a escrita juntas para reforçar essa garantia.

O fluxo de devolução (não desenhado aqui) segue lógica parecida, mas incluindo a verificação da fila de reserva (RN06) depois de incrementar `quantidade_disponivel` — pode ser o próximo diagrama de sequência quando chegarmos nessa parte da implementação.

---

## 4. Rastreabilidade

Ambos os diagramas devem ser consultados junto com [`requisitos-sistema-biblioteca.md`](requisitos-sistema-biblioteca.md), que contém o detalhamento de cada RF, RN, RNF e RS citado aqui.

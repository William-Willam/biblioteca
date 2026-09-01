# Sistema de Biblioteca — Levantamento de Requisitos

**Stack:** Spring Boot + PostgreSQL (backend) | Angular (frontend)
**Autor:** William
**Data:** Agosto/2026

---

## 1. Visão Geral

Sistema interno de gestão de biblioteca para cadastro de pessoas, livros, controle de empréstimos, devoluções, multas por atraso e fila de reserva de livros indisponíveis.

## 2. Atores

| Ator | Descrição |
|---|---|
| Bibliotecário/Admin | Único perfil do sistema. Acesso completo: cadastros, empréstimos, devoluções, reservas. |

---

## 3. Requisitos Funcionais (RF)

### Pessoa
| ID | Descrição |
|---|---|
| RF01 | Cadastrar, editar, listar e excluir/inativar pessoa |
| RF02 | Impedir exclusão de pessoa com empréstimo ativo (permitir apenas inativação) |

### Livro
| ID | Descrição |
|---|---|
| RF03 | Cadastrar, editar, listar e excluir livro |
| RF04 | Controlar quantidade total x quantidade disponível de exemplares |

### Empréstimo
| ID | Descrição |
|---|---|
| RF05 | Registrar empréstimo (validando disponibilidade e pendências da pessoa) |
| RF06 | Registrar devolução |
| RF07 | Calcular multa automaticamente quando a devolução ocorrer após a data prevista |
| RF08 | Listar empréstimos ativos, atrasados e histórico |

### Reserva
| ID | Descrição |
|---|---|
| RF09 | Permitir reserva quando não houver exemplar disponível (`quantidadeDisponivel == 0`) |
| RF10 | Disponibilizar automaticamente o livro para a próxima pessoa da fila quando houver devolução |
| RF11 | Definir prazo de retirada do livro reservado (ex: 48h) antes de passar para o próximo da fila |

### Multa, Exclusão e Autenticação
| ID | Descrição |
|---|---|
| RF12 | Registrar pagamento de multa via endpoint específico (`PATCH /emprestimos/{id}/pagar-multa`), atualizando o campo `multa_paga` do Empréstimo |
| RF13 | Impedir exclusão de livro com empréstimo ativo ou reserva pendente — permitir apenas inativação (mesmo padrão de RF02 para Pessoa) |
| RF14 | Endpoint de registro do Bibliotecário/Admin, ativo **somente enquanto não existir nenhum usuário cadastrado** (bootstrap de primeiro uso); autodesativa (retorna 403) após o primeiro cadastro |

---

## 4. Regras de Negócio (RN)

| ID | Descrição |
|---|---|
| RN01 | Multa: R$1,00 por dia de atraso, calculada no momento da devolução (`dias_atraso = dataDevolucaoReal - dataPrevistaDevolucao`) |
| RN02 | Pessoa com multa pendente (não paga) não pode realizar novo empréstimo |
| RN03 | Empréstimo só é permitido se `quantidadeDisponivel > 0` **e** a pessoa não tiver pendências |
| RN04 | Se `quantidadeDisponivel == 0`, a pessoa pode entrar na fila de reserva do livro |
| RN05 | Fila de reserva segue ordem de chegada (`dataReserva ASC`) |
| RN06 | Ao ocorrer devolução, o sistema verifica a fila: havendo reserva pendente, reserva automaticamente o exemplar para o primeiro da fila e define prazo de retirada |
| RN07 | Se o prazo de retirada expirar sem retirada, a reserva expira e passa para o próximo da fila |
| RN08 | Uma pessoa não pode ter mais de uma reserva ativa para o mesmo livro |
| RN09 | Status do empréstimo é sempre calculado (nunca armazenado manualmente): `ATIVO`, `ATRASADO` ou `DEVOLVIDO` |
| RN10 | Prazo padrão de empréstimo: 14 dias corridos a partir da data do empréstimo (`dataPrevistaDevolucao = dataEmprestimo + 14`) — política fixa do sistema, não escolhida por empréstimo |
| RN11 | Limite de 3 empréstimos ativos simultâneos por pessoa |
| RN12 | Multa tem teto de R$20,00 por empréstimo, mesmo que os dias de atraso ultrapassem esse valor em cálculo linear |
| RN13 | Pessoa que já está com o único exemplar de um livro pode reservá-lo para si mesma (autorreserva/renovação); nesse caso, o próximo empréstimo desse livro para essa mesma pessoa tem prazo reduzido de 7 dias (em vez dos 14 padrão de RN10) |
| RN14 | Não é permitido reduzir `quantidadeTotal` de um livro abaixo da quantidade atualmente emprestada (`quantidadeTotal - quantidadeDisponivel`) |

**Decisão:** expiração de reservas vencidas será feita **sob demanda** (não haverá job agendado nesta versão). Toda consulta à fila de reservas ou tentativa de novo empréstimo do livro verifica se há reserva `DISPONIVEL` com `prazoRetirada` vencido; se houver, expira na hora e processa a fila. Job agendado (`@Scheduled`) fica registrado como melhoria futura (RF15 no roadmap).

---

## 5. Requisitos Não-Funcionais (RNF)

| ID | Descrição |
|---|---|
| RNF01 | Backend em Spring Boot + PostgreSQL, API REST |
| RNF02 | Frontend em Angular, consumindo a API |
| RNF03 | Validações de negócio isoladas na camada de Service (Controller não contém regra de negócio) |
| RNF04 | Tratamento de erros padronizado via `@ControllerAdvice` + exceptions customizadas (`RegraNegocioException`, `RecursoNaoEncontradoException`) |
| RNF05 | Versionamento de schema do banco via Flyway |

---

## 6. Requisitos de Segurança (RS)

### Backend
| ID | Descrição |
|---|---|
| RS01 | Autenticação via JWT: access token de curta duração (~15min) + refresh token em cookie `httpOnly` |
| RS02 | Senha do bibliotecário armazenada com hash BCrypt, nunca em texto plano |
| RS03 | Todos os endpoints protegidos por padrão, exceto `/auth/login` |
| RS04 | Nenhuma resposta da API expõe senha/hash, mesmo em DTOs de erro |
| RS05 | `application.properties` fora do Git (`.gitignore`), com `application.properties.example` versionado |
| RS06 | Validação de entrada em todos os DTOs (`@Valid`, `@NotBlank`, `@Email`, etc.) |
| RS07 | CORS configurado explicitamente, liberando apenas a origem do Angular (nunca `*`) |

### Frontend (Angular)
| ID | Descrição |
|---|---|
| RS08 | Rotas protegidas por `AuthGuard` — sem token válido, redireciona para login |
| RS09 | `HttpInterceptor` para anexar o access token nas requisições e tratar renovação automática (401 → refresh → repete request) |
| RS10 | Access token armazenado em memória (nunca em `localStorage`), refresh token apenas em cookie `httpOnly` gerenciado pelo backend |
| RS11 | Logout limpa o estado local e invalida o refresh token no backend |

---

## 7. Entidades Principais (visão preliminar)

- **Pessoa** — id, nome, cpf, email, telefone, ativo
- **Livro** — id, título, autor, isbn, quantidadeTotal, quantidadeDisponivel, ativo *(sem campo de imagem/capa nesta versão — decisão registrada abaixo)*
- **Emprestimo** — id, pessoa (FK), livro (FK), dataEmprestimo, dataPrevistaDevolucao, dataDevolucaoReal, valorMulta, multaPaga (boolean, default false), status (calculado)
- **Reserva** — id, pessoa (FK), livro (FK), dataReserva, prazoRetirada, status (AGUARDANDO, DISPONIVEL, RETIRADA, EXPIRADA, CANCELADA)

**Decisão — imagem de capa:** avaliadas 3 opções (BLOB no PostgreSQL, arquivo em disco, cloud storage externo como Cloudinary/S3). Como o deploy será em Railway/Render (filesystem efêmero), a opção viável em produção seria cloud storage externo — porém, **decidido não incluir imagem de capa nesta versão** para manter o foco no core do sistema (empréstimo, devolução, multa, fila). Fica registrado como melhoria futura (RF16 no roadmap).

*Modelagem detalhada (diagrama ER, tipos de dados, constraints) a ser feita na próxima etapa.*

---

## 8. Próximos Passos

1. Modelagem do banco de dados (diagrama ER)
2. Criação das migrations Flyway
3. Implementação do backend (entidades → repositories → services → controllers)
4. Implementação do frontend Angular

## 9. Melhorias Futuras (fora do escopo do MVP)

| ID | Descrição |
|---|---|
| RF15 | Job agendado (`@Scheduled`) para expirar reservas vencidas automaticamente, substituindo a verificação sob demanda |
| RF16 | Upload de imagem de capa do livro, via storage externo (ex: Cloudinary), com URL armazenada no banco |
| RF17 | Renovação de empréstimo (estender prazo sem devolução física do livro) |
| RF18 | Paginação nas listagens (empréstimos, livros, pessoas) — hoje sem tratamento, vale revisar conforme o volume de dados cresce |
| RF19 | Testes automatizados (JUnit) cobrindo as regras de negócio da camada de Service |

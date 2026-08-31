# 📚 Sistema de Biblioteca

Sistema interno de gestão de biblioteca com cadastro de pessoas, livros, controle de empréstimos, devoluções, cálculo automático de multa por atraso e fila de reserva para livros indisponíveis.

Projeto de portfólio desenvolvido por [William-Willam](https://github.com/William-Willam).

## 🚧 Status

Em desenvolvimento — fase de levantamento de requisitos e modelagem concluída. Documentação completa em [`docs/requisitos-sistema-biblioteca.md`](docs/requisitos-sistema-biblioteca.md).

## 🛠️ Stack

**Backend**
- Java 21 + Spring Boot
- Spring Data JPA / Hibernate
- Spring Security (JWT + refresh token em cookie httpOnly)
- PostgreSQL
- Flyway (versionamento de schema)
- Maven

**Frontend**
- Angular
- TypeScript

## ✨ Funcionalidades

- **Pessoas** — cadastro, edição, listagem e inativação
- **Livros** — cadastro, edição, listagem, controle de exemplares (total x disponível)
- **Empréstimos** — registro de empréstimo e devolução, com validação de disponibilidade e pendências
- **Multas** — cálculo automático (R$1,00/dia de atraso), bloqueio de novo empréstimo enquanto houver pendência
- **Reservas** — fila de espera por ordem de chegada quando o livro está indisponível, com prazo de retirada e expiração automática (verificada sob demanda)

## 🔒 Segurança

- Autenticação via JWT (access token de curta duração + refresh token em cookie `httpOnly`)
- Senhas com hash BCrypt
- Rotas do Angular protegidas por `AuthGuard`
- `HttpInterceptor` para renovação automática de token
- CORS restrito à origem do frontend
- Validação de entrada em todos os endpoints (`@Valid`)

## 📁 Estrutura do Backend

```
br.com.william.biblioteca
├── controller
├── service
├── repository
├── model
├── dto
│   ├── request
│   └── response
└── exception
```

## 🚀 Como rodar o projeto

> ⚠️ Seção a ser preenchida conforme o desenvolvimento avança (instruções de setup do banco, variáveis de ambiente, comandos de build/run do backend e frontend).

### Pré-requisitos
- Java 21+
- Node.js e Angular CLI
- PostgreSQL

### Backend
```bash
cd backend
cp src/main/resources/application.properties.example src/main/resources/application.properties
# preencher as credenciais do banco em application.properties
./mvnw spring-boot:run
```

### Frontend
```bash
cd frontend
npm install
ng serve
```

## 🗺️ Roadmap / Melhorias Futuras

- [ ] Job agendado (`@Scheduled`) para expiração automática de reservas vencidas
- [ ] Upload de imagem de capa do livro via storage externo (Cloudinary/S3)

## 📄 Documentação

Levantamento completo de Requisitos Funcionais, Regras de Negócio, Requisitos Não-Funcionais e Requisitos de Segurança disponível em [`docs/requisitos-sistema-biblioteca.md`](docs/requisitos-sistema-biblioteca.md).

## 📝 Licença

Projeto de portfólio pessoal, sem licença específica definida ainda.

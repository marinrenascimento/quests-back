# Rascunho do back — Quests

> **Status: rascunho para discussão (pré-início).** Nada daqui foi implementado ainda. Este documento descreve **como o back deve funcionar** para atender às telas que já existem no front. Ele não substitui o `README.md` deste repositório: quando o grupo aprovar uma decisão daqui, ela vai para a seção "A definir pelo grupo" do README (e para o `CLAUDE.md`, se mudar alguma regra).
>
> Fontes: proposta de stack do grupo, telas e mocks do front (`quests-front`) e as regras de negócio em `quests-front/docs/REGRAS_DE_NEGOCIO.md` (citadas aqui como RN-xxx).

---

## 1. Decisões que precisam ser tomadas antes de começar

| # | Decisão | Por quê | Proposta |
|---|---------|---------|----------|
| D1 | **TypeScript + ES modules** no lugar de CommonJS | O README e o `CLAUDE.md` mandam usar CommonJS (`require`) e o `package.json` está com `"type": "commonjs"`. O Prisma 7 é distribuído como ES module e pede `"type": "module"`. | Adotar TypeScript com ES modules e atualizar o README e o `CLAUDE.md`. |
| D2 | **Postgres no Docker** para todo o grupo | No Windows, o Docker Desktop precisa de WSL2 ou virtualização ligada. | Confirmar que todos conseguem rodar. Se alguém não conseguir, usar um Postgres gratuito hospedado (só para desenvolvimento). |
| D3 | **Versão do Node** | O Prisma 7 exige Node 20.19 ou mais novo. | Fixar uma versão LTS para o grupo (ex.: 22) e registrar no README. |
| D4 | **Corrigir o `.gitignore`** | Ele está em UTF-16, então o Git não o lê. Por isso o `node_modules/` foi commitado, e o `.env` não está ignorado. | Fazer **antes** de existir o `.env` com a senha do banco e a chave do JWT (passos no README, seção "Problema conhecido"). Incluir também a pasta do cliente gerado pelo Prisma. |

---

## 2. Tecnologias

| Camada | Tecnologia | Papel |
|--------|------------|-------|
| Linguagem | Node.js LTS + TypeScript (`strict`) | Mesma linguagem do front. |
| Servidor | Express 5 (já instalado: `^5.3.0`) | Rotas HTTP. Atenção à API da v5 (rotas assíncronas, sintaxe de caminhos). |
| ORM | Prisma 7 + `@prisma/adapter-pg` | Models tipados, migrations e seed. Usa `prisma.config.ts`. |
| Banco | PostgreSQL (Docker) | Relacional e gratuito. Todo o grupo sobe o mesmo banco com `docker compose up`. |
| Validação | Zod | Valida corpo, parâmetros e query antes do controller. |
| Autenticação | JWT (`jsonwebtoken`) + `bcrypt` | Login com token; a senha fica salva só como hash. |
| Apoio | `cors` (já instalado), `dotenv`, `tsx` | O Prisma 7 não lê o `.env` sozinho. O `tsx` roda o TypeScript em desenvolvimento, com reload. |
| Consulta ao banco | DataGrip (gratuito para uso não comercial) | Só para **ver** dados e testar consultas. Mudança de estrutura vai **sempre** por migration. |

> Confirmar as versões na hora de instalar (o Prisma e o Express podem ter versões mais novas).

---

## 3. Estrutura de pastas

```
back/
├── docker-compose.yml       # Postgres para desenvolvimento
├── prisma.config.ts         # caminho do schema, migrations, seed e URL do banco
├── prisma/
│   ├── schema.prisma        # os models (um model = uma tabela)
│   ├── migrations/          # geradas por `npx prisma migrate dev` (vão para o Git)
│   └── seed.ts              # dados iniciais (cópia dos mocks do front)
├── src/
│   ├── server.ts            # sobe o Express em 0.0.0.0:PORT
│   ├── app.ts               # middlewares globais + registro das rotas
│   ├── config/              # variáveis de ambiente (validadas) e cliente do Prisma
│   ├── routes/              # liga URL → middlewares → controller (ex.: deck.routes.ts)
│   ├── controllers/         # lê a requisição, chama o service, devolve a resposta
│   ├── services/            # regras de negócio (RN-xxx) e consultas ao Prisma
│   ├── validators/          # schemas Zod de cada rota
│   ├── middlewares/         # autenticação (JWT), validação, tratamento de erros
│   └── utils/               # funções pequenas (ex.: normalizar texto de busca)
├── docs/
│   └── RASCUNHO_BACKEND.md  # este arquivo
├── .env.example             # variáveis necessárias, sem valores secretos
└── .env                     # valores reais (NUNCA no Git)
```

**Caminho de uma requisição:** route → middleware (autenticação, validação) → controller → service → Prisma → Postgres.

Os models ficam no `schema.prisma`, não numa pasta `src/models/`. Se a disciplina exigir uma pasta `models/` com um arquivo por classe, a troca seria para o Sequelize; o resto da estrutura continua igual.

---

## 4. Ambiente

### 4.1 `.env.example` (proposta)

```
# Banco (o mesmo usuário/senha do docker-compose.yml)
DATABASE_URL="postgresql://quests:quests@localhost:5432/quests"

# Servidor
PORT=3333

# Autenticação
JWT_SECRET="troque-por-um-valor-longo-e-aleatorio"
JWT_EXPIRES_IN="1d"            # sessão sem "Lembrar de Mim"
JWT_REMEMBER_EXPIRES_IN="30d"  # sessão com "Lembrar de Mim" (RN-005)
```

### 4.2 Scripts do `package.json` (proposta)

| Script | Comando | Para quê |
|--------|---------|----------|
| `dev` | `tsx watch src/server.ts` | Desenvolvimento com reload. |
| `build` | `tsc` | Compila para `dist/`. |
| `start` | `node dist/server.js` | Roda a versão compilada. |
| `db:migrate` | `prisma migrate dev` | Cria e aplica migrations. |
| `db:seed` | `prisma db seed` | Popula o banco com os dados iniciais. |
| `db:reset` | `prisma migrate reset` | Apaga e recria o banco (só em desenvolvimento). |

### 4.3 Rede (app no celular)

- O servidor escuta em `0.0.0.0`, para o celular acessar pelo **IP do PC** na mesma Wi-Fi (`http://<IP-do-PC>:3333`). `localhost` não funciona no Expo Go.
- O Windows pode pedir permissão do Firewall para o Node: permitir em redes privadas.
- **CORS** só faz diferença quando o app roda no navegador (Expo web). O app nativo não passa por CORS. Liberar a origem do Expo web em desenvolvimento.

---

## 5. Banco de dados

Ids em `uuid` (texto), que combinam com os ids em string que o front já usa nas rotas. Datas em UTC. Todas as tabelas principais têm `createdAt` (e `updatedAt` quando podem ser editadas).

### 5.1 Fase 1 — substitui os mocks que o front usa hoje

**Conta**

| Model | Campos | Observações |
|-------|--------|-------------|
| `User` | id, name, **username** (único), email (único), passwordHash, avatarId?, backgroundId?, coins, streakDays, lastStreakDate? | O login é por `username` (RN-003). `coins` são as moedas, que também são os pontos do Rank; **não existe XP** (RN-092). `coins` é um cache do histórico de moedas (fase 2). `streakDays` e `lastStreakDate` controlam a sequência de dias (RN-096, regra do "dia" pendente na RN-097). O nível não é guardado: é calculado pelas moedas (seção 5.4). `backgroundId` é o fundo escolhido pelo usuário (RN-100); fica vazio até o nível 14, quando o avatar ainda aparece sem fundo. |

**Catálogo de disciplinas** (usado por cards e questões)

| Model | Campos | Observações |
|-------|--------|-------------|
| `Subject` | id, name (único), createdById? | Disciplina. A busca não diferencia maiúsculas nem acentos (RN-045). |
| `Topic` | id, subjectId, name; único (subjectId, name) | O "Conteúdo" de uma disciplina (RN-046). |

**Decks**

| Model | Campos | Observações |
|-------|--------|-------------|
| `Deck` | id, userId, title, description? | Descrição opcional (RN-022). **O deck não tem disciplina** (RN-029): cada card tem a sua, e um deck pode misturar várias. A quantidade de cards é calculada (RN-021). Não existe deck sem cards: excluir o último card exclui o deck (RN-034). |
| `Flashcard` | id, deckId, topicId, front, back, position, difficulty?, lastReviewedAt?; único (deckId, position) | **Sem título** (RN-052): onde o app mostra "o nome" do card, mostra a frente. A disciplina vem pelo conteúdo (`topicId`), então não fica guardada duas vezes. `position` mantém a ordem (RN-053, RN-061). `difficulty` (`ReviewDifficulty`) é a última dificuldade marcada na revisão, que define quantas vezes o card aparece (RN-170, RN-171). |

**Provas**

| Model | Campos | Observações |
|-------|--------|-------------|
| `Exam` | id, title, ownerId?, institution?, year? | Sem `ownerId`: prova pública (vestibular). A quantidade de questões é calculada (RN-139). `institution` e `year` são para a futura tela de Vestibulares (sem mockup ainda). |
| `Question` | id, topicId, statement, difficulty, createdById? | `difficulty`: FACIL, MEDIO ou DIFICIL (RN-140). `createdById` serve para a regra de quem edita (RN-146). |
| `Alternative` | id, questionId, position (0 a 4 = A a E), text, isCorrect; único (questionId, position) | A letra vem da posição, como no front. Uma correta por questão (garantido no service). **`isCorrect` nunca vai para o app durante a prova.** |
| `ExamQuestion` | examId, questionId, position; chave (examId, questionId); único (examId, position) | `position` é o "QUESTÃO N". A mesma questão pode estar em várias provas. |

**Realização da prova**

| Model | Campos | Observações |
|-------|--------|-------------|
| `ExamAttempt` | id, userId, examId, status, startedAt, endedAt?, correctCount?, score? | `status`: IN_PROGRESS, SUBMITTED ou ABANDONED. Cada realização é uma linha nova. `endedAt` é o momento da entrega ou do abandono: de `startedAt` a `endedAt` é o tempo de estudo que entra nos Relatórios (se o abandono conta, depende da RN-193). |
| `AttemptAnswer` | attemptId, questionId, alternativeId; chave (attemptId, questionId) | Uma resposta por questão (RN-151). **Sem linha = em branco.** O service confere que a alternativa pertence à questão. |

**Home**

| Model | Campos | Observações |
|-------|--------|-------------|
| `RecentAccess` | id, userId, deckId?, examId?, accessedAt; único (userId, deckId) e (userId, examId) | Últimos decks e provas acessados, para os 2 quadrados da Home (RN-112). Um acesso novo ao mesmo item só atualiza `accessedAt`. Exatamente um entre `deckId` e `examId` é preenchido. **Registrado pelo próprio back** ao estudar um deck (`GET /decks/:deckId/review`) e ao fazer uma prova (`POST /exams/:examId/attempts`), não ao só abrir as telas (RN-117). Excluir o deck ou a prova apaga o acesso (cascata). |

**Condicionais** (entram só se o grupo decidir a regra)

| Model | Campos | Quando |
|-------|--------|--------|
| `UserExam` | userId, examId, addedAt; chave (userId, examId) | Se "Minhas Provas" puder incluir vestibulares adicionados pelo usuário. Se forem só as provas criadas por ele, `Exam.ownerId` basta. |
| `AttemptHighlight` | attemptId, questionId, wordIndex, color; chave (attemptId, questionId, wordIndex) | Se retomar a prova (RN-159) incluir as marcações do texto (RN-155). |

### 5.2 Fase 2 — telas e regras ainda em aberto

| Model | Campos | Telas e regras |
|-------|--------|----------------|
| `ReviewSession` | id, userId, deckId?, subjectId?, topicId?, startedAt, endedAt? | Revisão de um deck (`deckId`) ou estudo por disciplina (sem deck, RN-077: `subjectId` e, se escolhido, `topicId`). As duas contam para os Relatórios e para as moedas (RN-089). Início e fim alimentam as "horas de estudo". |
| `CardReview` | sessionId, flashcardId, difficulty?, reviewedAt | Histórico de cada marcação, para os Relatórios. A dificuldade **atual** do card fica em `Flashcard.difficulty` (fase 1). Sem chave única por card: com as repetições (RN-171), o mesmo card pode ser marcado mais de uma vez na sessão. |
| `CoinTransaction` | id, userId, amount, reason, reviewSessionId?, attemptId?, createdAt | Histórico de moedas ganhas (RN-096). É a fonte da verdade de `User.coins`: os dois são atualizados na mesma transação. `reason`: `CoinReason`. |
| `Avatar` | id, imageKey, name, upgraded, requiredLevel (único) | Os 14 avatares de `assets/images/avatares`, cada um com o nível que o libera (RN-094). Sem tabela de desbloqueio: o usuário tem todos os avatares com `requiredLevel` menor ou igual ao nível dele. Escolher um avatar (`User.avatarId`) só é aceito se ele estiver liberado. |
| `Background` | id, imageKey, name, character (único), requiredLevel (único) | Os 7 fundos de `assets/images/cenarios` dos níveis 14 a 20, um para cada personagem (RN-100). Mesma lógica do `Avatar`: sem tabela de desbloqueio, o usuário tem todos os fundos com `requiredLevel` até o nível dele, e escolher um fundo (`User.backgroundId`) só é aceito se ele estiver liberado. Se o fundo só vale com o avatar do mesmo personagem (RN-099), a escolha também confere `character`. |

**Fora do plano por decisão do grupo:**
- **Recuperação de senha:** é feita pelo suporte, por e-mail (RN-008). Não há tabela de token nem rota.
- **Feedback e suporte:** a tela de Feedback orienta o usuário a escrever para o e-mail de suporte, e o "Falar com o Suporte" abre um modal com esse e-mail (RN-210); não há formulário no app. Uma tabela de feedback só entra se isso mudar.

### 5.3 Enums

| Enum | Valores |
|------|---------|
| `QuestionDifficulty` | FACIL, MEDIO, DIFICIL |
| `ReviewDifficulty` | FACIL, MEDIO, DIFICIL, IMPOSSIVEL |
| `AttemptStatus` | IN_PROGRESS, SUBMITTED, ABANDONED |
| `MarkerColor` | BLUE, YELLOW, RED |
| `CoinReason` | REVIEW_COMPLETED, STUDY_COMPLETED, EXAM_CORRECT_ANSWERS, STREAK_DAY |

### 5.4 O que não vira tabela (é calculado)

- Quantidade de cards de um deck e de questões de uma prova: contagem.
- Fila de uma revisão (cards e repetições, RN-171): montada na hora a partir de `Flashcard.difficulty` (seção 7.4).
- Relatórios (RN-190, RN-192): calculados na hora, para a semana ou o mês. Vêm de `ReviewSession` e `ExamAttempt` (horas), de `AttemptAnswer` com o gabarito (`Alternative.isCorrect`) e o conteúdo da questão (acertos e desempenho por matéria) e de `User.streakDays` (sequência). Detalhes na seção 7.7.
- Rank: usuários ordenados por `User.coins` (sem período).
- **Nível** (RN-093): `nível = mínimo(20, moedas ÷ 1.000, arredondado para baixo)`. A faixa sai do nível: 0 a 2 Bronze, 3 a 5 Prata, 6 a 8 Ouro, 9 a 11 Ametista, 12 a 14 Rubi, 15 a 17 Esmeralda, 18 a 20 Diamante. Como os limites são fixos, ficam numa constante do código, e não numa tabela. As moedas são cumulativas e nunca são gastas (RN-097): `User.coins` é o total ganho, e acima de 20.000 o nível fica em 20 enquanto as moedas continuam somando.
- Avatares e fundos liberados: os de `requiredLevel` até o nível atual.

### 5.5 Ligações

```
User 1─N Deck 1─N Flashcard N─1 Topic N─1 Subject
User 1─N ExamAttempt 1─N AttemptAnswer ─> Question, Alternative
Exam N─N Question (via ExamQuestion, com position)
Question 1─N Alternative
Question N─1 Topic
```

---

## 6. Autenticação

- **Cadastro** (`POST /auth/register`): name, username, email e password, todos obrigatórios (RN-006). O e-mail precisa ter formato válido. A senha precisa ter **pelo menos 8 caracteres, com letras e números (RN-009, validada aqui no back)**. `username` e `email` únicos: se já existirem, a resposta é `409`, indicando o campo. O "repetir senha" é conferido só no front. Ao criar a conta, devolve sucesso e o app volta para o login (RN-007).
- **Login** (`POST /auth/login`): username e password (RN-003). O username é comparado sem os espaços das pontas. Usuário ou senha errados devolvem a mesma mensagem genérica (não revelar qual dos dois errou). Sucesso devolve o token e os dados básicos do usuário (RN-004).
- **Token:** JWT no cabeçalho `Authorization: Bearer <token>`. Com "Lembrar de Mim" marcado (o padrão, RN-005), o token dura mais (`JWT_REMEMBER_EXPIRES_IN`) e o app o guarda no aparelho para abrir direto na Home (RN-002). Sem a opção, o token dura menos e não é guardado.
  - No front, guardar o token exige uma lib nova (ex.: `expo-secure-store`, que funciona no Expo Go). Instalar com `npx expo install`.
- **Senha:** só o hash (`bcrypt`) vai para o banco. Nunca devolver `passwordHash` em nenhuma resposta.
- **Recuperação de senha:** não existe no back (RN-008). O app mostra um modal com o e-mail do suporte.

---

## 7. Rotas da API (proposta)

Todas as rotas, exceto as de `/auth`, exigem token. "Dono" = o recurso pertence ao usuário do token; se não pertencer, a resposta é `404` (para não revelar que existe).

### 7.1 Conta e usuário

| Método | Rota | Para quê | Telas e regras |
|--------|------|----------|----------------|
| POST | `/auth/register` | Criar conta | Cadastro (RN-006, RN-007, RN-009) |
| POST | `/auth/login` | Entrar | Login (RN-003, RN-004, RN-005) |
| GET | `/me` | Nome, moedas, nível, faixa, avatar, fundo e sequência de dias | TopBar em todas as telas (RN-090), início do app (RN-002) |
| GET | `/me/recent` | Os 2 últimos decks ou provas estudados (tipo, id, nome). Usuário novo: lista vazia. | Home (RN-112, RN-117) |
| GET | `/me/avatars` | Avatares, com quais estão liberados | Conta, Rank (RN-094) — fase 2 |
| PATCH | `/me/avatar` | Escolher o avatar `{ avatarId }` (só um liberado) | Conta (RN-094) — fase 2 |
| GET | `/me/backgrounds` | Fundos, com quais estão liberados e qual está em uso | Tela de escolha do fundo, a definir (RN-100) — fase 2 |
| PATCH | `/me/background` | Escolher o fundo `{ backgroundId }` (só um liberado) | Tela de escolha do fundo, a definir (RN-100) — fase 2 |

### 7.2 Disciplinas e conteúdos

| Método | Rota | Para quê | Telas e regras |
|--------|------|----------|----------------|
| GET | `/subjects?search=` | Listar e buscar disciplinas | Novo/Editar Flashcard (RN-045) |
| POST | `/subjects` | Criar disciplina | Novo Flashcard e, no futuro, cadastro de questões (RN-045, RN-047) |
| GET | `/subjects/:subjectId/topics?search=` | Conteúdos de uma disciplina | Novo/Editar Flashcard (RN-046) |
| POST | `/subjects/:subjectId/topics` | Criar conteúdo | Novo Flashcard e, no futuro, cadastro de questões (RN-046, RN-047) |

O catálogo de disciplinas e conteúdos é **um só** para cards e questões, com as mesmas regras de busca e criação (RN-047). A criação recusa um nome que já existe (sem diferenciar maiúsculas nem acentos) com `409`.
| GET | `/study/subjects` | Disciplinas e conteúdos **com cards** | Configurar Deck por Disciplina (RN-079) |

### 7.3 Decks e cards

| Método | Rota | Para quê | Telas e regras |
|--------|------|----------|----------------|
| GET | `/decks?cursor=&limit=` | Meus Decks (título, descrição, quantidade de cards) | Meus Decks (RN-020, RN-021) |
| POST | `/decks` | Criar deck **com os cards vinculados**, numa transação só | Cadastro de deck (RN-022, RN-025) |
| GET | `/decks/:deckId` | Dados do deck | Cards do Deck (RN-032: `404` = "Deck Não Encontrado") |
| GET | `/decks/:deckId/cards?cursor=&limit=` | Cards do deck, na ordem, cada um com a própria disciplina e o próprio conteúdo | Cards do Deck (RN-040, RN-055) |
| POST | `/decks/:deckId/cards` | Adicionar card no fim do deck | "Adicionar Card" (RN-053) |
| GET | `/decks/:deckId/cards/:cardId` | Um card | Visualizar Card (RN-063 a RN-067), Editar Flashcard |
| PATCH | `/decks/:deckId/cards/:cardId` | Editar card, mantendo a posição | Editar Flashcard (RN-059 a RN-061) |
| DELETE | `/decks/:deckId/cards/:cardId` | Excluir card. Se era o último, exclui o deck na mesma transação e responde `{ "deckDeleted": true }`, e o app volta para Meus Decks. O aviso antes de excluir é o mesmo de qualquer card. | Cards do Deck (RN-041, RN-034, RN-035) |

### 7.4 Revisão e estudo por disciplina

| Método | Rota | Para quê | Telas e regras |
|--------|------|----------|----------------|
| GET | `/decks/:deckId/review` | Monta a fila da revisão do deck, com as repetições por dificuldade | Revisão (RN-070, RN-171) |
| PUT | `/decks/:deckId/cards/:cardId/difficulty` | Salva a dificuldade marcada `{ difficulty }` no card (vale a última) | Revisão (RN-072, RN-076, RN-170) |
| POST | `/study/subject` | Estudo por disciplina: `{ subjectId, topicId?, quantity }`. Sem `topicId`, usa os cards de todos os conteúdos da disciplina. Sorteia `quantity` cards sem repetir entre os decks do usuário; se houver menos, devolve todos (ex.: 30 pedidos e 14 disponíveis: 14). Não cria deck. | Configurar Deck por Disciplina (RN-077, RN-082 a RN-085) |
| POST | `/review-sessions` e `/review-sessions/:sessionId/finish` | Registrar início e fim da sessão, para os Relatórios | Relatórios — fase 2 |

**Montagem da fila (RN-171, proposta):**
1. Cada card entra 1 vez. Card salvo como Difícil entra mais 2 vezes (3 no total). Card salvo como Impossível entra mais 3 vezes (4 no total).
2. As cópias são distribuídas sem nunca colocar o mesmo card duas vezes seguidas. Uma forma simples: a cada posição, escolher, entre os cards com cópias sobrando e diferentes do anterior, o que tem mais cópias sobrando (empate: a ordem da lista).
3. Quando não houver outro card para intercalar (ex.: no fim só restam cópias do mesmo card), o card pode vir seguido: **nenhuma cópia é descartada**.

Pendentes na RN-172: se a marcação feita durante a revisão já acrescenta cópias na revisão em andamento (o que exigiria o front reorganizar a fila restante, ou pedir ao back uma fila nova); se card sem dificuldade conta como 1 vez; se a ordem base é a da lista ou embaralhada; se a regra vale também para o estudo por disciplina.

**Moedas da revisão (RN-096):** `POST /review-sessions/:sessionId/finish` com `{ completed: true }` (chegou ao "Revisão Concluída!") dá 1 moeda; encerrar antes (`completed: false`) não dá. Vale para deck e estudo por disciplina (RN-089), e toda revisão completa dá moeda, mesmo repetindo o deck no mesmo dia (RN-097).

**Sequência de dias (RN-096, RN-097)** — tratada no service de moedas, na mesma transação de cada moeda ganha por revisão, estudo ou prova:
1. Se `lastStreakDate` já é hoje, nada muda.
2. Senão, o dia é novo e vale para a sequência: se `lastStreakDate` é ontem, `streakDays` soma 1; se é mais antigo (perdeu pelo menos um dia) ou vazio, a sequência recomeça em `streakDays = 1`. Nos dois casos, `lastStreakDate` vira hoje e o usuário ganha 1 moeda de sequência (`STREAK_DAY`), inclusive no primeiro dia (RN-098).
3. A moeda de sequência não conta para validar o dia (só as de revisão, estudo ou prova).
4. Ao **mostrar** a sequência (`/me`, Relatórios), se `lastStreakDate` for anterior a ontem, ela é exibida como 0, mesmo que o banco ainda guarde o valor antigo.

"Hoje" e "ontem" no fuso do Brasil, como nos Relatórios. Se várias revisões no mesmo dia dão moeda, depende da RN-097.

### 7.5 Provas e questões

| Método | Rota | Para quê | Telas e regras |
|--------|------|----------|----------------|
| GET | `/exams?cursor=&limit=` | Minhas Provas (título e quantidade de questões) | Minhas Provas (RN-130, RN-131, RN-133) |
| GET | `/exams/:examId` | Dados da prova | Questões da Prova (RN-142: `404` = "Prova Não Encontrada") |
| GET | `/exams/:examId/questions?cursor=&limit=` | Questões da prova (rótulo, enunciado, dificuldade), na ordem | Questões da Prova (RN-137, RN-138, RN-147) |

### 7.6 Realização da prova

| Método | Rota | Para quê | Telas e regras |
|--------|------|----------|----------------|
| POST | `/exams/:examId/attempts` | Começar: cria a realização e devolve as questões com as alternativas, **sem o gabarito** | Realização da Prova (RN-150) |
| GET | `/attempts/:attemptId` | Retomar uma realização em andamento | Depende da RN-159 |
| PUT | `/attempts/:attemptId/answers/:questionId` | Escolher ou trocar a resposta `{ position }` | RN-151 |
| DELETE | `/attempts/:attemptId/answers/:questionId` | Desmarcar (volta a "em branco") | RN-151 |
| POST | `/attempts/:attemptId/submit` | Entregar: corrige pelo gabarito, grava `correctCount` e `score`, dá 1 moeda por acerto (`EXAM_CORRECT_ANSWERS`) e devolve o resumo | RN-152, RN-096 (refazer a prova dá moedas de novo, RN-097) |
| POST | `/attempts/:attemptId/abandon` | Sair sem entregar | RN-153 |

> Alternativa a avaliar: o app pode enviar todas as respostas de uma vez na entrega (`POST /attempts/:attemptId/submit` com a lista), em vez de uma requisição por toque. Gera menos tráfego, mas só funciona se o grupo **não** quiser retomar a prova (RN-159).

### 7.7 Rank e Relatórios (fase 2)

| Método | Rota | Para quê | Telas e regras |
|--------|------|----------|----------------|
| GET | `/ranking` | Pódio, lista e o cartão do usuário (posição, moedas, nível e faixa), por total de moedas | Rank (RN-091 a RN-093) |
| GET | `/reports?period=semanal\|mensal` | Indicadores do período (abaixo) | Relatórios (RN-114, RN-190, RN-192) |

**Cálculo dos relatórios (RN-190, RN-192)** — o mesmo para a semana e o mês, só muda o período:

| Indicador | Cálculo |
|-----------|---------|
| Horas de estudo | Soma da duração (`startedAt` a `endedAt`) das revisões, dos estudos por disciplina e das provas do período. |
| Questões acertadas | Respostas certas das provas **entregues** no período. |
| Taxa de acerto | Acertadas ÷ respondidas (questões em branco não entram na conta). |
| Sequência | `User.streakDays` (RN-096, RN-097). |
| Horas por dia | As horas de estudo separadas por dia (no semanal, de segunda a domingo). |
| Desempenho por matéria | Para cada disciplina das questões respondidas: acertadas ÷ respondidas. |

Pendentes na RN-193: se só as provas entregues contam; se revisão encerrada antes do fim e prova abandonada contam tempo; o início da semana e do mês; e se o mensal mostra a sequência de dias e as horas por dia (como o semanal) ou continua com a "Sequência de Meses" e o gráfico "Este Mês × Mês Passado". Os dias e meses são calculados no fuso do Brasil, não em UTC.

---

## 8. Padrões da API

- **Validação:** Zod em `validators/`. Erro de validação devolve `400` com o campo, para o front mostrar a mensagem **abaixo do campo**, no padrão que as telas já usam.
- **Formato de erro** (igual em todas as rotas):

  ```json
  { "error": { "code": "VALIDATION_ERROR", "message": "Dados inválidos.", "fields": { "password": "A senha precisa ter pelo menos 8 caracteres, com letras e números." } } }
  ```

  Códigos: `400` validação, `401` sem token ou token inválido, `404` não encontrado (ou não é do usuário), `409` conflito (username ou e-mail já usados), `500` erro inesperado (sem detalhes internos na resposta).
- **Paginação** (as listas do front têm o TODO "carregar mais ao rolar"): `?cursor=<id>&limit=<n>`, com resposta `{ "items": [...], "nextCursor": "<id> | null" }`.
- **Textos** (mensagens de erro e rótulos) em português, como no app.
- **Busca** por "contém", sem diferenciar maiúsculas nem acentos: "arte" encontra "Artes" (RN-045, RN-048, RN-078).

---

## 9. Seed (dados iniciais)

Os mocks estão no repositório do **front**, que é separado; o seed precisa de uma **cópia** dos dados.

- **Usuário de teste:** um usuário com senha conhecida, dono dos decks do seed.
- **Avatares:** os 14 da RN-094, com o nível de cada um (0 a 6 sem upgrade, 7 a 13 com upgrade) e o nome da imagem da RN-095 (ex.: nível 8 `com-upgrade/goblinTerno.png`, nível 13 `com-upgrade/elfoMantoBranco.png`).
- **Fundos:** os 7 da RN-100, com o nível e o personagem de cada um e o nome da imagem em `cenarios/`: 14 Cavaleiro `cenarioCavaleiro.png`, 15 Goblin `cenarioGoblin.png`, 16 Mago `cenarioMago.png`, 17 Fada `cenarioFada.png`, 18 Esqueleto `cenarioEsqueleto.png`, 19 Elfo `cenarioElfo.png` e 20 Rei `cenarioRei.png`. Cada personagem tem um único fundo.
- **Disciplinas e conteúdos:** os do catálogo fictício do front (`constants/mockSubjects.ts`).
- **Decks e cards:** os 15 decks fictícios (`mockDecks.ts`) e os cards-base de cada um (`mockDeckCards.ts`), **sem** as repetições numeradas que o front gera para encher os decks grandes. No mock, a disciplina está no deck; no seed, ela vai para cada card (conteúdo = tema do deck), seguindo a RN-029. O título que os cards do mock têm é descartado: o card não tem título (RN-052).
- **Provas e questões:** cerca de 30 questões únicas (as 21 da lista-base e as 9 do Simulado de Matemática), com 5 alternativas cada, ligadas às 26 provas por `ExamQuestion`. As 1.235 questões do front são repetições geradas ("... (2)") e não precisam virar linhas no banco.
- **Gabarito:** o mock do front não tem gabarito (RN-141). No seed, marcar a alternativa correta só para permitir testar a correção.

---

## 10. Integração com o front

- **URL da API:** proposta de usar a variável `EXPO_PUBLIC_API_URL` (lida pelo Expo) num arquivo `.env.local` do front, com o IP do PC (ex.: `http://192.168.0.15:3333`).
  - **Atenção:** o `.gitignore` do front ignora `.env*.local`, mas **não** ignora `.env`. Usar `.env.local`.
- **Onde trocar os mocks:** os pontos estão marcados no front com `// TODO: substituir pelos dados do back`. Os principais são o `DecksProvider` (`hooks/useDecks.tsx`), `constants/mockExams.ts`, `constants/mockUser.ts`, `mockRanking.ts` e `mockReports.ts`.
- **Deck sem disciplina (RN-029):** o tipo `Deck` do front ainda tem o campo `discipline`, usado no rótulo dos cards e no "Adicionar Card". Ele sai quando o front for ajustado (RN-040, RN-054, RN-055).
- **Contrato:** front e back estão em repositórios separados e não compartilham tipos automaticamente. O formato de cada rota (corpo e resposta) é combinado e registrado antes de implementar.

---

## 11. Ordem sugerida de implementação

1. Decisões D1 a D4 (seção 1) e atualização do README e do `CLAUDE.md`.
2. Esqueleto: `docker-compose.yml`, `prisma.config.ts`, `server.ts`, `app.ts` e uma rota de verificação (`GET /health`).
3. `schema.prisma` da fase 1, a primeira migration e o seed.
4. Autenticação (`/auth/register`, `/auth/login`, `/me`) e a ligação com o login e o cadastro do front.
5. Disciplinas, decks e cards (as telas de decks já estão prontas no front), incluindo a dificuldade salva no card e a fila da revisão com repetições (RN-170, RN-171).
6. Provas, questões e realização da prova.
7. Fase 2: sessões de revisão para os Relatórios, pontos, Rank, avatares e fundos.

---

## 12. Decisões de negócio que mudam o back

Respostas pendentes no `REGRAS_DE_NEGOCIO.md` do front que alteram tabelas ou rotas:

| Regra | Pergunta | O que muda |
|-------|----------|------------|
| RN-047 | Disciplinas e conteúdos criados no app são de todos ou só do usuário? | `Subject.createdById` e quem vê o quê |
| RN-172 | Repetições do card Impossível (N), card sem dificuldade, efeito na revisão em andamento, ordem, sobras, estudo por disciplina | `GET /decks/:deckId/review` e `POST /study/subject` |
| RN-011 | Regras do "Usuário": tamanho, caracteres, maiúsculas, se pode trocar | Validação do cadastro e do login, índice único de `username` |
| RN-012 | Telas citadas na ajuda que não existem (perfil, alterar senha, exportar ou excluir conta, 2FA) | Rotas novas em `/me` |
| RN-036 | Tamanho máximo dos textos; nomes de deck repetidos | Validadores Zod, regras de unicidade |
| RN-037 | Editar ou excluir o deck inteiro | `PATCH` e `DELETE /decks/:deckId` |
| RN-099 | Desempate e tamanho do Rank; escolha do avatar; fundo preso ao avatar do mesmo personagem; fundo novo passa a ser usado ao ser liberado | `GET /ranking`, `PATCH /me/avatar`, `PATCH /me/background` (conferir `Background.character`), atualizar `User.backgroundId` ao subir de nível |
| RN-118 | Acesso recente ao começar ou ao concluir; estudo por disciplina nos recentes | Onde o `RecentAccess` é gravado |
| RN-193 | O que entra nas horas e nas questões dos Relatórios; início da semana e do mês; formato do mensal | `GET /reports` |
| RN-140, RN-141, RN-156 | Quem define a dificuldade, campos da questão, escala das estrelas | `Question`, `Alternative` |
| RN-146 | Quem pode editar ou excluir questões | `createdById` e as regras dos services |
| RN-158 | A prova tem tempo limite? | `ExamAttempt` (prazo) |
| RN-159 | Dá para sair e retomar a prova? | `ExamAttempt` em andamento, `AttemptHighlight`, `GET /attempts/:id` |
| RN-160 | Pode entregar em branco? Pode refazer? | Validação do `submit`, várias realizações |
| "Minhas Provas" | Inclui vestibulares adicionados pelo usuário? | `UserExam` |

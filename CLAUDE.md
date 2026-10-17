# CLAUDE.md — quests (back)

Node.js + Express `^5.3.0` + cors `^2.8.6`, CommonJS. Leia também `README.md`.

## Regras

- O back ainda não tem código: não existe `index.js` (apesar do `"main"`), nem script `start`/`dev`, rotas ou banco.
- Não invente estrutura de pastas, banco, rotas, porta ou autenticação. Esses itens estão em "A definir pelo grupo" no README. Pergunte antes de decidir.
- Use CommonJS (`require` / `module.exports`), não ES modules.
- Express 5: confira a API da versão 5 (não a da v4) antes de escrever rotas/middlewares.
- Segredos só em `.env`, nunca no código nem no Git.
- O app (Expo Go) acessa a API pelo IP do PC na rede local, não por `localhost`.
- Ao tomar uma decisão com o grupo, atualize o README (seção "A definir pelo grupo").
- Nunca commitar `node_modules/` nem `.env`. Commits pequenos e com mensagens claras.
- Problema conhecido: `.gitignore` está em UTF-16 (o Git não lê) e `node_modules/` já foi commitado. Só corrija se o usuário pedir.

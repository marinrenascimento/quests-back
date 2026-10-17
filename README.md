# Quests — Back (API)

## Visão geral

**Quests** é um projeto dividido em duas partes, cada uma no seu próprio repositório:

| Parte | Repositório | Papel |
|-------|-------------|-------|
| **front** | `quests-front` | App mobile em Expo / React Native (telas, navegação, interação). |
| **back** (este) | `quests-back` | API em Node.js + Express que o app chama para buscar e salvar dados. |

O back é responsável pelas regras de negócio, pela persistência dos dados e por expor rotas HTTP para o app.

> **Estado atual:** o back está só no início. Existem apenas o `package.json` com as dependências instaladas. **Ainda não há nenhum arquivo de código** (o `index.js` citado em `"main"` não existe), nem rotas, nem banco de dados.

## Stack e versões

Versões declaradas no `package.json`:

| Tecnologia | Versão | Para quê |
|------------|--------|----------|
| Node.js | não fixada no projeto | Ambiente de execução |
| Express | `^5.3.0` | Servidor HTTP / rotas |
| cors | `^2.8.6` | Permitir que o app faça requisições à API |

Outras informações do `package.json`:

- `"type": "commonjs"`: usa `require(...)` / `module.exports` (não `import`/`export`).
- `"main": "index.js"`: arquivo de entrada esperado (**ainda não criado**).
- Não existe script `start` ou `dev`. O único script é o `test` padrão, que só mostra um erro.

## Como rodar (Windows / PowerShell)

### Pré-requisitos

- [Node.js](https://nodejs.org/) LTS instalado

### Instalar dependências

```powershell
cd caminho\para\quests\back
npm install
```

### Iniciar o servidor

**Ainda não é possível**, porque não existe arquivo de entrada nem script de start. Quando o grupo criar o arquivo de entrada, este README deve ser atualizado com o comando (por exemplo, `node index.js` ou `npm run dev`).

### Acesso pelo app (Expo Go)

Quando a API estiver rodando no seu PC:

- O app roda no celular, então **não consegue acessar `localhost`** do computador. O front deve usar o **IP do PC na rede local** (descubra com `ipconfig` → "Endereço IPv4"), ex.: `http://192.168.0.15:<porta>`.
- O celular e o PC precisam estar **na mesma rede Wi-Fi**.
- Na primeira execução, o Windows pode pedir permissão do Firewall para o Node. **Permita em redes privadas**, senão o celular não consegue se conectar.
- O servidor precisa escutar em todas as interfaces de rede (o padrão do Express já faz isso), e não só em `127.0.0.1`.

## Estrutura de pastas

### Estrutura atual

```
back/
├── .gitignore          # Lista de arquivos ignorados pelo Git (ver aviso abaixo)
├── CLAUDE.md           # Regras resumidas do projeto para o Claude
├── README.md           # Este arquivo
├── package.json        # Dependências (express, cors) e scripts
├── package-lock.json   # Versões exatas instaladas
└── node_modules/       # Dependências instaladas (não deve ir para o Git)
```

A organização do código ainda não foi definida (veja "A definir pelo grupo").

## Padrões e regras

O que já vale hoje:

- **CommonJS:** use `require` e `module.exports`, conforme o `"type": "commonjs"` do `package.json`.
- **Dependências:** instale com `npm install <pacote>` e commite o `package.json` e o `package-lock.json` juntos.
- **Segredos** (senhas, chaves, string de conexão do banco) ficam em arquivo `.env`, **nunca** no código e **nunca** no Git.
- **Não inventar estrutura sozinho:** decisões de pastas, banco e rotas são tomadas pelo grupo e registradas neste README.

## A definir pelo grupo

Nada abaixo foi decidido ainda. Quando o grupo decidir, atualize esta seção:

- [ ] **Arquivo de entrada** (`index.js`? outro nome?) e **porta** do servidor
- [ ] **Scripts** do `package.json` (`start`, `dev` com recarregamento automático?)
- [ ] **Estrutura de pastas** (ex.: separação entre rotas, controllers, serviços, models)
- [ ] **Banco de dados** (qual banco, como conectar, se vai usar ORM)
- [ ] **Rotas da API** (endpoints, formato das requisições/respostas, padrão de nomes)
- [ ] **Autenticação** (como o login do app vai ser validado)
- [ ] **Variáveis de ambiente** (quais existem, se haverá um `.env.example`)
- [ ] **Configuração do CORS**
- [ ] **Versão do Node.js** a ser usada pelo grupo
- [ ] **Testes** (ferramenta e se serão obrigatórios)

## Fluxo de Git

- **Um repositório para o front e outro para o back.**
  - Back: `https://github.com/marinrenascimento/quests-back.git`
  - Front: `https://github.com/marinrenascimento/quests-front.git`
- **Commits pequenos e frequentes**, cada um com uma mudança só.
- **Mensagens claras** dizendo o que mudou, por exemplo:
  - `Cria servidor Express inicial`
  - `Adiciona rota GET /quests`
  - `Configura CORS`
- **Nunca commitar:**
  - `node_modules/`
  - arquivos `.env`
- Antes de commitar, confira o que vai entrar:

  ```powershell
  git status
  git diff
  ```

> **Problema conhecido no `.gitignore`:** o arquivo `back/.gitignore` foi salvo com codificação **UTF-16** (comum quando criado pelo PowerShell com `echo node_modules > .gitignore`). O Git não lê esse formato, então **a pasta `node_modules/` foi commitada** no primeiro commit do back. Além disso, o `.gitignore` não lista `.env`. Para corrigir (combinem antes quem faz):
> 1. Recriar o `.gitignore` em **UTF-8** com as linhas `node_modules/` e `.env`.
> 2. Remover `node_modules` do Git sem apagar do disco: `git rm -r --cached node_modules`.
> 3. Commitar a correção.

## Checklist: antes de criar uma funcionalidade nova

- [ ] Fiz `git pull` para pegar a versão mais recente.
- [ ] A estrutura de pastas, o banco e o padrão de rotas necessários já foram **definidos pelo grupo** (seção "A definir pelo grupo"). Se não, combinem antes.
- [ ] Combinei com quem cuida do front o **formato da rota** (método, URL, corpo e resposta).
- [ ] Usei `require` / `module.exports` (CommonJS).
- [ ] Nenhuma senha/chave está no código. Tudo sensível está no `.env`.
- [ ] Se instalei pacote novo, `package.json` e `package-lock.json` estão no commit.
- [ ] Testei a rota (ex.: pelo navegador, Postman/Insomnia ou pelo app usando o IP do PC).
- [ ] Commits pequenos, com mensagens claras, sem `node_modules` nem `.env`.
- [ ] Atualizei este README se alguma decisão nova foi tomada.

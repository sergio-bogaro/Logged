# Logged — deploy

Repositório de orquestração que sobe o backend ([LoggedApi](https://github.com/sergio-bogaro/LoggedApi))
e o frontend ([LoggedApp](https://github.com/sergio-bogaro/LoggedApp)) juntos via Docker Compose.

Os dois projetos entram como **git submodules**; este repo guarda apenas o `docker-compose.yml`,
o compose do CasaOS, o `Dockerfile` de cada app (dentro dos submodules) e a configuração do Nginx.

## Arquitetura

```
navegador ──:5173──► Nginx ──► serve o dist/ do frontend (LoggedApp)
                        │
                        └──/api,/auth,/uploads,/custom-views──► FastAPI :8000 (LoggedApi)
                                                                     │
                                                         dados persistidos
                                                         ├── logged.db
                                                         └── uploads/
```

O app é **same-origin**: o navegador fala só com a porta `5173`, e o Nginx faz proxy reverso
para a API. Assim **não há CORS**, e funciona a partir de qualquer dispositivo da rede
(`http://<ip-do-servidor>:5173`) sem embutir IP no build.

- Frontend: servido pelo Nginx na porta `5173`.
- API: FastAPI na `8000` (interna; Swagger acessível via `http://<host>:5173/docs`).
- Dados: SQLite (`logged.db`) + `uploads/`.

> A porta `8000` é publicada apenas no compose de desenvolvimento local, para acesso direto à
> API/Swagger. No CasaOS só a `5173` é exposta.

## Pré-requisitos

- Docker com Compose v2 (`docker compose version`)

## Rodar localmente

```bash
# Clone com os dois submodules
git clone --recurse-submodules https://github.com/sergio-bogaro/Logged.git
cd Logged

# Configure a chave do TMDB usada no build do frontend (copie e edite)
cp .env.example .env

# Build + start
docker compose up --build -d
```

Acesse **http://localhost:5173**. Swagger em http://localhost:5173/docs.

```bash
docker compose logs -f      # logs
docker compose down         # para (mantém os dados)
docker compose down -v      # para e apaga banco + uploads
docker compose up --build   # rebuild após mudar código
```

## Instalar no CasaOS (app customizado)

O CasaOS **não constrói imagens** — ele sobe imagens já existentes. Por isso o fluxo é:
**clonar e buildar no servidor** e só então instalar o app no CasaOS.

### 1. No servidor (via SSH), buildar as imagens

```bash
git clone --recurse-submodules https://github.com/sergio-bogaro/Logged.git
cd Logged
docker compose build        # cria logged-api:latest e logged-web:latest
```

Não é preciso subir com `docker compose up`; as imagens ficam prontas para o CasaOS.

> Se você subiu localmente na mesma máquina para testar, rode `docker compose down` antes de
> instalar no CasaOS, para liberar a porta `5173`.

### 2. No CasaOS, instalar o app

1. App Store → **“+”** → **“Install a customized app”**.
2. **Import** → cole o conteúdo de [`casaos/docker-compose.yml`](casaos/docker-compose.yml).
3. Preencha, se quiser a busca de jogos, `IGDB_CLIENT_ID` e `IGDB_CLIENT_SECRET`.
4. Install.

O tile **Logged** aparece no dashboard; clicar nele abre `http://<ip-do-servidor>:5173`.

### Onde ficam os dados

```
/DATA/AppData/logged/
├── logged.db     ← banco SQLite
└── uploads/      ← imagens enviadas
```

Para backup, basta copiar essa pasta.

## Configuração e segredos

| Variável | Onde fica | Obrigatória | Descrição |
|---|---|---|---|
| `VITE_TMDB_API_KEY` | `.env` na raiz (build local) | não | Chave do TMDB, embutida no bundle. Sem ela a busca de filmes fica indisponível. |
| `IGDB_CLIENT_ID` | `LoggedApi/.env` (local) ou UI do CasaOS | não | Credenciais Twitch/IGDB para a busca de jogos. |
| `IGDB_CLIENT_SECRET` | `LoggedApi/.env` (local) ou UI do CasaOS | não | idem. |

Nenhum segredo é versionado. O `env_file` do backend é opcional: se `LoggedApi/.env` não
existir, a API sobe normalmente (só a busca de jogos fica indisponível).

## Atualizar

**Local:** `git pull && git submodule update --init --recursive && docker compose up --build -d`.

**CasaOS:**

```bash
cd Logged
git pull
git submodule update --init --recursive
docker compose build
```

Depois, no CasaOS, **pare e inicie** o app na dashboard para ele recriar os containers com as
novas imagens (se ele insistir na imagem antiga, use “Reinstall”/“Recreate” no menu do app).

## Limpeza de disco

O build do frontend (Node) gera bastante cache. De tempos em tempos:

```bash
docker builder prune -f     # limpa cache de build
docker image prune -f       # remove imagens órfãs
```

## Solução de problemas

| Sintoma | Causa / solução |
|---|---|
| API não responde ao abrir de outro dispositivo | Confirme que está acessando pela **porta 5173** (mesma origem), não pela `8000`. |
| `pull access denied` / erro ao instalar no CasaOS | As imagens locais não existem ou o `pull_policy: never` é ignorado. Rode `docker compose build` no servidor e confirme `docker images` mostrando `logged-api` e `logged-web`. |
| Porta 5173 em uso | Pare o app no CasaOS ou rode `docker compose down` do compose de desenvolvimento na mesma máquina. |
| CasaOS usa a imagem antiga após atualizar | Pare/inicie ou reinstale o app no CasaOS para recriar o container. |

> Alternativa futura: publicar as imagens num registry (ex.: GHCR) e trocar `pull_policy: never`
> por `always` simplificaria as atualizações (sem build no servidor). Não é necessário hoje.

## Estrutura

```
.
├── docker-compose.yml        # desenvolvimento local (build da fonte)
├── casaos/
│   └── docker-compose.yml    # app customizado do CasaOS (imagens locais)
├── .env.example
├── .gitignore
├── .gitmodules
├── LoggedApi/   (submodule — contém Dockerfile e .dockerignore)
└── LoggedApp/   (submodule — contém Dockerfile, nginx.conf e .dockerignore)
```

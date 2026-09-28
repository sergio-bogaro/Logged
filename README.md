# Logged — deploy

Repositório de orquestração que sobe o backend ([LoggedApi](https://github.com/sergio-bogaro/LoggedApi))
e o frontend ([LoggedApp](https://github.com/sergio-bogaro/LoggedApp)) juntos.

Os dois projetos entram como **git submodules**; este repo guarda o(s) `docker-compose`, o
compose de deploy, o `Dockerfile` de cada app (dentro dos submodules) e a configuração do Nginx.

Funciona em **qualquer** dashboard/gerenciador de home server: Docker Compose via SSH, Portainer,
Dockge, Komodo, CasaOS, ou apenas como link em Homarr/Homepage/Dashy.

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
(`http://<ip-do-servidor>:5173`) sem embutir IP no build. A porta `8000` fica interna.

## Pré-requisitos

- Docker com Compose v2 (`docker compose version`) — ou um dashboard/gerenciador que aceite compose.

## Imagens publicadas

| Imagem | Descrição |
|---|---|
| `ghcr.io/sergio-bogaro/logged-api` | Backend FastAPI |
| `ghcr.io/sergio-bogaro/logged-web` | Frontend + Nginx (proxy para a API) |

- Tags: `latest`, versões (`1.0.0`, `1.0`, `1`) e `sha-<curto>`.
- Arquitetura: `linux/amd64`.
- Os pacotes precisam estar **públicos** no GitHub (Package → Settings → Change visibility).

## Opção 1 — Rodar localmente (build da fonte)

Útil para desenvolvimento. Constrói as imagens a partir dos submodules.

```bash
git clone --recurse-submodules https://github.com/sergio-bogaro/Logged.git
cd Logged
cp .env.example .env        # edite o VITE_TMDB_API_KEY (chave do TMDB, embutida no build)
docker compose up --build -d
```

Acesse **http://localhost:5173** (Swagger em `/docs`).

## Opção 2 — Deploy no home server (pull das imagens)

Não precisa de código-fonte nem build: só baixar as imagens do GHCR.

```bash
docker compose -f deploy/docker-compose.yml up -d
```

Variáveis opcionais (defina num `.env` ao lado do compose ou no painel):

| Variável | Padrão | Descrição |
|---|---|---|
| `LOGGED_TAG` | `latest` | Tag da imagem (ex.: `1.0.0` para fixar versão) |
| `LOGGED_WEB_PORT` | `5173` | Porta no host |
| `IGDB_CLIENT_ID` | — | Busca de jogos (opcional) |
| `IGDB_CLIENT_SECRET` | — | Busca de jogos (opcional) |

### Docker Compose (SSH)

```bash
docker compose -f deploy/docker-compose.yml up -d
docker compose -f deploy/docker-compose.yml logs -f
docker compose -f deploy/docker-compose.yml pull   # atualizar
```

### Portainer / Dockge / Komodo

- **Portainer:** Stacks → Add stack → cole o conteúdo de `deploy/docker-compose.yml`
  (ou aponte para o repositório Git com esse caminho) → Deploy.
- **Dockge:** Novo stack → cole o YAML → Deploy.
- **Komodo:** Create Stack → use o compose acima como Stack.

Em todos, defina `IGDB_*` nas variáveis de ambiente do stack, se quiser a busca de jogos.

### Dashboards de links (Homarr / Homepage / Dashy / Heimdall)

Não consomem compose: basta cadastrar um link/apontamento para
`http://<ip-do-servidor>:5173` e usar um ícone qualquer. (A imagem já sobe via compose/gerenciador.)

### CasaOS (app customizado)

1. App Store → **“+” → “Install a customized app” → Import** → cole `deploy/docker-compose.yml`.
2. Ajuste `IGDB_*` na UI, se quiser.
3. Install. O tile **Logged** aparece no dashboard (metadados `x-casaos` do arquivo).

## Configuração e segredos

| Variável | Onde | Descrição |
|---|---|---|
| `VITE_TMDB_API_KEY` | secret do repo **LoggedApp** (CI) ou `.env` na raiz (build local) | Chave do TMDB, embutida no bundle. Sem ela a busca de filmes fica indisponível. |
| `IGDB_CLIENT_ID` / `IGDB_CLIENT_SECRET` | variáveis do stack (deploy) ou `LoggedApi/.env` (local) | Credenciais Twitch/IGDB para a busca de jogos. |

Nenhum segredo é versionado.

## Dados e backup

Por padrão os dados ficam no volume nomeado `logged_data`:

```
logged.db     ← banco SQLite
uploads/      ← imagens enviadas
```

Para dados visíveis no host (backup simples), troque o volume por um bind mount em
`deploy/docker-compose.yml`:

```yaml
    volumes:
      - /caminho/no/host/logged:/data
```

## Atualizar

- Fixe uma versão com `LOGGED_TAG=1.0.0` ou use `latest`.
- Para atualizar: `docker compose -f deploy/docker-compose.yml pull` e depois `up -d`.
- No painel, use “Pull & Redeploy”/“Recreate” do stack.

## Publicar novas imagens (CI)

Os workflows `.github/workflows/docker-publish.yml` (em cada submodule) publicam no GHCR:

- **push em `master`** → `latest` + `sha-<curto>`;
- **tag `v1.2.3`** → `1.2.3`, `1.2`, `1`, `latest`.

Após o primeiro push, marque os pacotes `logged-api` e `logged-web` como **públicos**.

## Limpeza de disco (build local)

```bash
docker builder prune -f
docker image prune -f
```

## Estrutura

```
.
├── docker-compose.yml            # desenvolvimento local (build da fonte)
├── deploy/
│   └── docker-compose.yml        # deploy genérico (pull do GHCR, x-casaos)
├── .env.example
├── .gitignore
├── .gitmodules
├── LoggedApi/   (submodule — Dockerfile + workflow de publicação)
└── LoggedApp/   (submodule — Dockerfile, nginx.conf + workflow de publicação)
```

## Solução de problemas

| Sintoma | Causa / solução |
|---|---|
| API não responde de outro dispositivo | Acesse pela **porta 5173** (mesma origem), não pela `8000`. |
| `manifest unknown` / erro ao subir | Pacote do GHCR privado ou tag inexistente. Torne o pacote público e/ou confira a tag. |
| Porta 5173 em uso | Outro app usa a porta; ajuste `LOGGED_WEB_PORT`. |
| Imagem antiga após atualizar | Rode `pull` no stack/gerenciador e recrie o container. |
| Busca de filmes/jogos indisponível | Falta `VITE_TMDB_API_KEY` (build) ou `IGDB_*` (runtime). |

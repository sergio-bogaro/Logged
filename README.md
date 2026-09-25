# Logged — deploy

Repositório de orquestração que sobe o backend ([LoggedApi](https://github.com/sergio-bogaro/LoggedApi))
e o frontend ([LoggedApp](https://github.com/sergio-bogaro/LoggedApp)) juntos via Docker Compose.

Os dois projetos entram como **git submodules**; este repo guarda apenas o `docker-compose.yml`,
o `Dockerfile` de cada app (dentro dos próprios submodules) e a configuração do Nginx.

## Arquitetura

```
navegador ──:5173──► Nginx (serve o dist/ do LoggedApp)  ──chamadas de API──► :8000 FastAPI (LoggedApi)
                                                                                    │
                                                                        volume logged_data:/data
                                                                        ├── logged.db
                                                                        └── uploads/
```

- O frontend buildado é servido por **Nginx** na porta `5173` (reaproveita o CORS já configurado no backend).
- A API roda em `http://localhost:8000` (Swagger em `/docs`) e é chamada diretamente pelo navegador.
- Banco SQLite e uploads persistem no volume nomeado `logged_data`.

## Pré-requisitos

- Docker com Compose v2 (`docker compose version`)

## Uso

```bash
# Clone com os dois submodules
git clone --recurse-submodules <url-deste-repo> Logged
cd Logged

# Configure a chave do TMDB usada no build do frontend
cp .env.example .env      # depois edite VITE_TMDB_API_KEY

# Suba tudo
docker compose up --build
```

Acesse:

- App: http://localhost:5173
- API: http://localhost:8000
- Swagger: http://localhost:8000/docs

Se você já tinha clonado sem `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

Atualizar os submodules para as versões mais recentes dos repositórios:

```bash
git submodule update --remote --merge
```

## Segredos e configuração

| Variável | Onde fica | Obrigatória | Descrição |
|---|---|---|---|
| `VITE_TMDB_API_KEY` | `.env` (raiz) | não | Chave do TMDB, embutida no build do frontend. Sem ela a busca de filmes fica indisponível. |
| `IGDB_CLIENT_ID` | `LoggedApi/.env` | não | Credenciais Twitch/IGDB para a busca de jogos. |
| `IGDB_CLIENT_SECRET` | `LoggedApi/.env` | não | idem. |

Ambos os arquivos `.env` são ignorados pelo git. O `env_file` do backend é opcional:
se `LoggedApi/.env` não existir, a API sobe normalmente.

## Operação

```bash
docker compose logs -f          # acompanhar logs
docker compose down             # parar (mantém o volume de dados)
docker compose down -v          # parar e apagar banco + uploads
docker compose up --build       # rebuild após mudar código
```

### Reaproveitar dados de um setup local

Por padrão os dados ficam no volume `logged_data`. Para usar um `logged.db` e uma pasta
`uploads/` que já existem em `LoggedApi/`, troque o volume por bind mounts no `docker-compose.yml`:

```yaml
    volumes:
      - ./LoggedApi/logged.db:/data/logged.db
      - ./LoggedApi/uploads:/data/uploads
```

## Estrutura

```
.
├── docker-compose.yml
├── .env.example
├── .gitignore
├── .gitmodules
├── LoggedApi/   (submodule — contém Dockerfile e .dockerignore)
└── LoggedApp/   (submodule — contém Dockerfile, nginx.conf e .dockerignore)
```

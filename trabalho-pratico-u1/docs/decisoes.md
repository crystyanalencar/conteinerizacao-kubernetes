# Decisões de projeto

Como a estrutura em contêineres do `guess_game` foi montada e por quê. Foi feita em camadas: cada
peça entra, é testada sozinha e só então a próxima começa. Para cada camada: o que entrou, por quê,
como foi provado e o que se observou.

## Objetivo

Empacotar o jogo de adivinhação ([fams/guess_game](https://github.com/fams/guess_game)) com Docker
Compose em três papéis:

- **backend** Flask (Python 3.12), em mais de uma instância;
- **Postgres**, com os dados num volume persistente;
- **NGINX**, servindo o frontend React (node 18) e fazendo proxy reverso com balanceamento de carga
  entre as instâncias do backend.

Requisitos transversais: reinício automático de contêiner que falhar, e atualização de qualquer
componente apenas trocando a versão (tag) da imagem.

**Restrição que guia todas as decisões:** o código do jogo não é alterado. Tudo o que precisar de
ajuste se resolve na infraestrutura (Dockerfile, variáveis de ambiente, compose, NGINX).

## Linha de base: código original intacto

O primeiro commit desta pasta é o código de `fams/guess_game` no commit `2c15943` (02/07/2026),
exportado com `git archive` (só os arquivos versionados, sem o histórico nem a pasta `.git` do
original). Todos os commits seguintes são de infraestrutura e documentação.

Para conferir que o fonte não mudou:

```bash
git diff 83decb6 HEAD --stat -- .     # 83decb6 = commit de base; só devem aparecer infraestrutura e docs
```

ou, comparando com o original:

```bash
git clone https://github.com/fams/guess_game.git /tmp/original
diff -rq /tmp/original . -x .git
```

**Por que repositório próprio e não fork:** o repositório reúne os trabalhos da disciplina (a
Unidade 2, de Kubernetes, deve reaproveitar as imagens daqui). O fork daria a comparação com o
original "de graça" no GitHub; o commit de base e o `diff` acima cumprem o mesmo papel.

## Reconhecimento: o que os arquivos do projeto dizem

Antes de escrever qualquer Dockerfile, a leitura dos arquivos do original respondeu o que o backend
precisa para rodar.

| Arquivo | Achado | Consequência |
|---|---|---|
| `start-backend.sh` | Define `FLASK_APP=run.py`, seis variáveis `FLASK_DB_*` e sobe com `flask run --host=0.0.0.0 --port=5000`. | É a receita do contêiner: comando, porta e variáveis. |
| `start-backend.sh` | Fixa `FLASK_DB_HOST="localhost"`. | O script não é usado na imagem: dentro do contêiner, `localhost` é o próprio backend, não o Postgres. Comando vai no `CMD`; variáveis de banco vão para o compose, com o host igual ao nome do serviço do Postgres. Evita também o problema de fim de linha CRLF que o README do original menciona. |
| `run.py` | `from guess import create_app`; `app = create_app()`. | `FLASK_APP=run.py` basta para o `flask run` achar a aplicação. |
| `requirements.txt` | Flask 3.0.3, Flask-Cors, boto3, `psycopg2-binary`, OpenTelemetry (sem versão fixa). | `psycopg2-binary` já vem compilado: `python:3.12-slim` basta, sem compilador. As versões não são alteradas (seria mexer no projeto). |
| `requirements-freeze.txt` | Vazio (0 bytes). | O arquivo que vale é o `requirements.txt`. |
| `guess/__init__.py` | O tipo de banco vem de `FLASK_DB_TYPE`; **sem ela, o padrão é DynamoDB** (AWS). | A variável é obrigatória em todo lugar onde o backend rodar. |
| `guess/__init__.py` | A conexão com o Postgres é aberta na criação da aplicação; se falhar, a aplicação não sobe. | No compose, o backend precisa esperar o banco ficar pronto (`depends_on` com `condition: service_healthy`), e o `restart:` cobre o resto. |
| `guess/__init__.py` | Rota `GET /health` responde `{"status": "ok"}`. | Usada para testar a imagem sozinha e como `healthcheck` do backend. |
| `guess/__init__.py` | A string de conexão do Postgres usa `DB_NAME`, `DB_USER`, `DB_PASSWORD` e `DB_HOST`; `FLASK_DB_PORT` não é lida. | O Postgres fica na porta padrão 5432. |
| `guess/__init__.py` | `print(app.config)` na inicialização. | A configuração inteira, inclusive a senha do banco, aparece no log do backend. Não é corrigido (é código do jogo); fica registrado como limitação. |
| `guess/game_routes.py` | Rotas `POST /create` e `POST /guess/<game_id>`, sem prefixo `/api`. | O NGINX precisa encaminhar essas rotas ao backend; o caminho exato depende de como o frontend chama a API. |
| `.gitignore` | Ignora `Dockerfile-front-local`, `Dockerfile-otel`, `Dockerfile-no-otel` e `docker-compose-local.yaml` (os nomes usados pelo autor), além de `db/*`. | Os arquivos deste trabalho usam os nomes padrão (`Dockerfile`, `docker-compose.yml`), que não estão na lista. Nada de dados em `./db`: o banco usa volume nomeado. |

## Camada 1: imagem do backend

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV FLASK_APP=run.py
EXPOSE 5000
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
```

| Decisão | Por quê |
|---|---|
| `python:3.12-slim` | Versão pedida no enunciado; `slim` é menor e suficiente. |
| `requirements.txt` copiado e instalado **antes** do código | O código muda com frequência e as dependências, raramente. Nessa ordem, mudar o código reaproveita do cache a camada do `pip install`. |
| `--no-cache-dir` | Não guarda o cache do pip dentro da imagem. |
| Só `FLASK_APP` no `ENV` | É fixo da aplicação. As variáveis de banco mudam por ambiente e ficam no compose; senha não fica gravada na imagem. |
| `--host=0.0.0.0` | Com `127.0.0.1`, o Flask só aceitaria conexões de dentro do próprio contêiner, e o NGINX não chegaria nele. |
| `CMD` na forma exec | O Flask vira o PID 1 e recebe o SIGTERM do `docker stop`. Na forma shell, o PID 1 seria o `sh`, que não repassa o sinal, e o contêiner só pararia no SIGKILL após 10 s. |
| `pip` e não `uv` | O `uv pip install --system` lê o mesmo `requirements.txt` e seria mais rápido, mas com poucas dependências o ganho é pequeno e o `pip` é o padrão das aulas. `uv sync` foi descartado porque exigiria `pyproject.toml` e `uv.lock`, mudando a estrutura do projeto. |

**Avisos do build, aceitos:** o pip avisa que roda como root e que há versão nova. O primeiro não se
aplica a contêiner (a imagem só tem esta aplicação, não há Python do sistema a proteger); o segundo
não é seguido para o build não depender da data em que roda. O contêiner também **executa** como
root; rodar com usuário próprio (`USER`) é a melhoria natural de segurança.

**Ambiente:** o pacote `docker.io` do Ubuntu não traz o BuildKit; o `docker build` caía no builder
legado (aviso `DEPRECATED`). Resolvido instalando o pacote `docker-buildx`. As versões recentes do
`docker compose build` também dependem dele.

**Como testar a imagem sozinha:** `guess-backend:1.0` com SQLite, sem Postgres:

```bash
docker run --rm -p 5000:5000 -e FLASK_DB_TYPE=sqlite -e FLASK_DB_PATH=/tmp/game.sqlite guess-backend:1.0
curl -i http://localhost:5000/health
```

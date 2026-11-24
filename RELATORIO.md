# Relatorio de evidencias - Trabalho Pratico Integrador 01-06

Nome: Joao Costa   Data: 24/11/2026
URL do repositorio Git: https://github.com/jdiogocosta92/tp-integrador-devops

## E01 - Ambiente (1 valor)
Comandos e versoes observadas:
- `docker --version` -> `Docker version 29.9.2, build a82e528`
- `docker compose version` -> `Docker Compose version v5.6.1`
- `git --version` -> `git version 2.56.1.windows.1`
- `docker info` -> `Server Version: 29.9.2` | `OSType: linux` | `Containers: 3`

Componentes do projeto:
- API: `app.py` (FastAPI, endpoints /health e /visits)
- Testes: `tests/test_app.py` (pytest)
- Compose: `compose.yaml` (+ overlay de incidente `incidente.yaml`)
- Pipeline: `Jenkinsfile`

Conclusao: o daemon Docker responde e executa containers Linux; Docker Compose V2 e Git estao
disponiveis, pelo que o ambiente cumpre os pre-requisitos do trabalho.

## E02 - Dockerfile e container (3 valores)
Tag da imagem; docker ps; HTTP /health; utilizador efetivo do container:

Comandos:
```
docker build -t natixis-support:1.0 .
docker run -d --name tp-api -p 127.0.0.1:18080:8000 natixis-support:1.0
docker ps
curl.exe http://127.0.0.1:18080/health
docker logs --tail 15 tp-api
docker exec tp-api id
docker rm -f tp-api
```

- Tag: `natixis-support:1.0`
- Portas: host `127.0.0.1:18080` -> container `8000/tcp` (docker ps: `127.0.0.1:18080->8000/tcp`)
- /health: `{"status":"ok","service":"support-api","version":"1.0"}` (HTTP 200)
- Log: `INFO: 172.17.0.1:37466 - "GET /health HTTP/1.1" 200 OK`
- id: `uid=999(appuser) gid=999(appgroup) groups=999(appgroup)`

Conclusao: a imagem instala as dependencias, copia apenas o necessario (.dockerignore exclui .git,
.venv, caches, .env, logs e evidencias) e executa a API com um utilizador nao privilegiado, com /data
gravavel. A API respondeu com HTTP 200 e status "ok" na porta publicada.

## E03 - Compose, DNS, healthcheck e volume (4 valores)
Resultado de docker compose config; health; teste checker -> api; contador antes/depois da recriacao:

Comandos:
```
docker compose config --quiet
docker compose up -d --build
docker compose ps
docker compose --profile diagnostico run --rm checker -fsS http://api:8000/health
curl.exe http://127.0.0.1:18080/visits
curl.exe -X POST http://127.0.0.1:18080/visits
curl.exe http://127.0.0.1:18080/visits
docker compose up -d --force-recreate
docker compose ps
curl.exe http://127.0.0.1:18080/visits
```

- `config --quiet`: sem output (configuracao valida)
- Recursos criados: rede `natixis-integrador_default`, volume `natixis-integrador_app-data`
- docker compose ps: `natixis-integrador-api-1 ... Up 5 seconds (healthy) 127.0.0.1:18080->8000/tcp`
- checker -> api: `Container natixis-integrador-api-1 Healthy` seguido de
  `{"status":"ok","service":"support-api-integrador","version":"1.0"}`
- Contador: GET `{"visits":0}` -> POST `{"visits":1}` -> GET `{"visits":1}`
  -> apos `--force-recreate`: `{"visits":1}` (container novo, novamente healthy)

Conclusao: o healthcheck em Python/urllib marca a api como healthy e o checker so arranca apos
service_healthy, resolvendo o servico pelo nome "api" no DNS interno do Compose. O contador nao se
perdeu porque /data/visitas.txt esta no volume nomeado app-data, que existe independentemente do
container e e reutilizado quando este e recriado.

## E04 - Incidente Docker (3 valores)
Sintoma, comandos de investigacao, causa-raiz, alteracao realizada, prova da recuperacao e contador preservado:

**1. Sintoma:** apos aplicar `docker compose -f compose.yaml -f incidente.yaml up -d --force-recreate`,
o servico api ficou `Up About a minute (unhealthy)` e `curl.exe http://127.0.0.1:18080/health`
devolveu `curl: (52) Empty reply from server`.

**2. Comandos de diagnostico:**
- `docker compose -f compose.yaml -f incidente.yaml ps` -> unhealthy, `127.0.0.1:18080->8000/tcp`
- `docker compose -f compose.yaml -f incidente.yaml logs --tail 30 api` -> `Uvicorn running on http://0.0.0.0:8001`
- `docker compose -f compose.yaml -f incidente.yaml config` -> command com `--port "8001"`; ports `target: 8000`;
  healthcheck em `http://127.0.0.1:8000/health`
- `docker inspect --format "{{json .State.Health}}" natixis-integrador-api-1` -> `"Status":"unhealthy","FailingStreak":10`,
  `ConnectionRefusedError: [Errno 111] Connection refused`
- `docker compose -f compose.yaml -f incidente.yaml exec api python -c "...urlopen('http://127.0.0.1:8001/health')..."`
  -> `{"status":"ok","service":"support-api-integrador","version":"1.0"}`

**3. Causa-raiz:** o overlay incidente.yaml altera o command do servico api para o Uvicorn escutar na
porta 8001, mas o mapeamento (18080->8000) e o healthcheck continuam a apontar para 8000, onde nenhum
processo escuta. A aplicacao esta funcional (responde em 8001 dentro do container); a falha e apenas
de configuracao.

**4. Correcao:** regresso a configuracao base (sem o overlay), recriando apenas o servico api e
mantendo o volume: `docker compose up -d --force-recreate api`

**5. Validacao:**
- `docker compose ps` -> `natixis-integrador-api-1 ... Up 2 minutes (healthy) 127.0.0.1:18080->8000/tcp`
- `curl.exe http://127.0.0.1:18080/health` -> `{"status":"ok","service":"support-api-integrador","version":"1.0"}`
- `curl.exe http://127.0.0.1:18080/visits` -> `{"visits":1}` (igual ao valor anterior ao incidente)

Conclusao: servico reposto e contador preservado, pois o volume app-data nao foi removido.

## E05 - Git (3 valores)
Branch, hashes de commits, merge e tag v1.0.0:

Comandos:
```
git init
git check-ignore -v .venv/x .env __pycache__/x.pyc .pytest_cache/x
git add .
git commit -m "feat: ambiente integrador funcional"
git branch -M main
git switch -c fix/health-info
# app.py: campo "team": "support" em /health; tests/test_app.py: assert response.json()["team"] == "support"
git add app.py tests/test_app.py
git commit -m "fix: acrescentar identificacao no health"
git switch main
git merge fix/health-info
git tag -a v1.0.0 -m "Release integrador 1.0.0"
git log --oneline --graph --decorate -6
git remote add origin https://github.com/jdiogocosta92/tp-integrador-devops.git
git push -u origin main
git push origin v1.0.0
```

- .gitignore: `git check-ignore -v` confirmou `.venv/` (linha 1), `__pycache__/` (linha 2),
  `.pytest_cache/` (linha 3) e `.env` (linha 5) como ignorados
- Branch principal: `main` (criada como master por `git init` e renomeada com `git branch -M main`)
- Branch de correcao: `fix/health-info`
- Commit 1: `fef0757` feat: ambiente integrador funcional
- Commit 2: `d132023` fix: acrescentar identificacao no health
- Testes apos a alteracao (container python:3.13-slim): `2 passed, 1 warning in 0.60s`
- Merge: `Updating fef0757..d132023` / `Fast-forward` (app.py, tests/test_app.py)
- git log:
  ```
  * d132023 (HEAD -> main, tag: v1.0.0, fix/health-info) fix: acrescentar identificacao no health
  * fef0757 feat: ambiente integrador funcional
  ```
- Publicacao: `* [new branch] main -> main` e `* [new tag] v1.0.0 -> v1.0.0` em
  https://github.com/jdiogocosta92/tp-integrador-devops (remoto sem credenciais no URL)

Conclusao: a correcao foi desenvolvida na branch fix/health-info, validada pelos testes e integrada na main
por fast-forward; a release v1.0.0 (tag anotada) aponta para d132023 e esta publicada no remoto,
garantindo rastreabilidade entre codigo, teste e versao.

## E06 - Jenkins SUCCESS (4 valores)
URL/numero do build, etapas e identificador da imagem construida:

## E07 - Jenkins FAILURE e recuperacao (2 valores)
Build FAILURE, etapa que falhou e excerto relevante do log; build SUCCESS apos correcao:

## Verificacao final
[ ] Dockerfile [ ] compose.yaml [ ] app.py [ ] tests/ [ ] Jenkinsfile
[ ] Historico Git [ ] Tag v1.0.0 [ ] Evidencias E01-E07

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

- Job: `tp-integrador-devops` (Pipeline script from SCM, Git
  https://github.com/jdiogocosta92/tp-integrador-devops.git, branch `*/main`, Script Path `Jenkinsfile`)
- Build SUCCESS: **#2** - http://localhost:8080/job/tp-integrador-devops/2/
- Etapas: Checkout -> Tests -> Docker Build, todas concluidas com sucesso (`Finished: SUCCESS`)
- Tests: `.venv/bin/python -m pytest -q` -> `2 passed`
- Docker Build:
  ```
  Successfully tagged natixis-support:ci-2
  + docker image inspect natixis-support:ci-2 --format {{.Id}}
  sha256:c768d275923e90d1f518ca2f831d68b31d3a233e3e99a33e65bdad231284bb7f
  ```
- Imagem construida: `natixis-support:ci-2`, Id `sha256:c768d275923e90d1f518ca2f831d68b31d3a233e3e99a33e65bdad231284bb7f`

Nota (build #1 - FAILURE de ambiente, nao relacionada com o codigo): Checkout e Tests passaram (`2 passed`),
mas Docker Build falhou com `docker: not found` (`script returned exit code 127`). Diagnostico: o agente
ci-python-docker tinha `DOCKER_HOST=tcp://docker-engine:2375` mas nao tinha a CLI `docker`. Correcao no
Dockerfile.agent do laboratorio (`COPY --from=docker:28-cli /usr/local/bin/docker /usr/local/bin/docker`) e
`docker compose up -d --build ci-agent`; validado com `docker exec jenkins-ci-agent docker version`
(Client 28.5.2 / Server 28.5.2).

Conclusao: a pipeline obtem o codigo do Git, executa os testes automatizados num ambiente virtual isolado e
constroi uma imagem com tag unica por execucao (ci-${BUILD_NUMBER}), confirmada por docker image inspect.
O build #2 terminou em SUCCESS nas tres etapas.

## E07 - Jenkins FAILURE e recuperacao (2 valores)
Build FAILURE, etapa que falhou e excerto relevante do log; build SUCCESS apos correcao:

Comandos:
```
# tests/test_app.py (test_health): assert response.json()["status"] == "offline"   (alteracao temporaria)
git add tests/test_app.py
git commit -m "test: falha controlada - esperar status offline"
git push
# Jenkins: Build Now -> #3
# tests/test_app.py: reposto assert response.json()["status"] == "ok"
git add tests/test_app.py
git commit -m "test: repor assercao status ok"
git push
# Jenkins: Build Now -> #4
```

**Build FAILURE controlado: #3** - http://localhost:8080/job/tp-integrador-devops/3/ (commit `c31b19a`)
- Etapa que falhou: **Tests** (Checkout passou; Docker Build nao avancou)
- Excerto do log:
  ```
  FAILED tests/test_app.py::test_health - AssertionError: assert 'ok' == 'offline'
    - offline
    + ok
  1 failed, 1 passed, 1 warning in 0.51s
  [Pipeline] { (Docker Build)
  Stage "Docker Build" skipped due to earlier failure(s)
  ERROR: script returned exit code 1
  Finished: FAILURE
  ```

**Build SUCCESS apos correcao: #4** - http://localhost:8080/job/tp-integrador-devops/4/ (commit `0156215`)
- Checkout, Tests e Docker Build concluidos; `Successfully tagged natixis-support:ci-4`
- `docker image inspect natixis-support:ci-4 --format {{.Id}}` ->
  `sha256:c768d275923e90d1f518ca2f831d68b31d3a233e3e99a33e65bdad231284bb7f`
- `Finished: SUCCESS`

Historico Git (commits de demonstracao mantidos):
```
0156215 (HEAD -> main, origin/main) test: repor assercao status ok
c31b19a test: falha controlada - esperar status offline
9dd6b4f docs: evidencias E06 (build Jenkins SUCCESS)
29ff9d3 docs: relatorio com evidencias E01-E05
901ff41 ci: pipeline Jenkins com testes e build da imagem
d132023 (tag: v1.0.0, fix/health-info) fix: acrescentar identificacao no health
fef0757 feat: ambiente integrador funcional
```

Conclusao: a assercao alterada falhou porque a API devolve status="ok"; o pytest terminou com codigo 1, a
etapa Tests ficou em FAILURE e o Jenkins nao executou Docker Build, impedindo a criacao de uma imagem a partir
de codigo nao validado. Reposta a assercao, o build #4 voltou a SUCCESS. O Id da imagem e igual ao do build #2
porque app.py e requirements.txt (os unicos ficheiros copiados para a imagem) nao mudaram, pelo que o Docker
reutilizou todas as camadas em cache e apenas acrescentou a tag ci-4. Os builds #1 a #4 e os commits de
demonstracao foram mantidos.

## Verificacao final
[x] Dockerfile [x] compose.yaml [x] app.py [x] tests/ [x] Jenkinsfile
[x] Historico Git [x] Tag v1.0.0 [x] Evidencias E01-E07

# Relatorio de evidencias - Trabalho Pratico Integrador 01-06

Nome: Joao Costa   Data: 24/11/2026
URL do repositorio Git: <a preencher no Exercicio 04>

## E01 - Ambiente (1 valor)
Comandos e versoes observadas:
- `docker --version` -> <colar linha>
- `docker compose version` -> <colar linha>
- `git --version` -> <colar linha>
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

## E06 - Jenkins SUCCESS (4 valores)
URL/numero do build, etapas e identificador da imagem construida:

## E07 - Jenkins FAILURE e recuperacao (2 valores)
Build FAILURE, etapa que falhou e excerto relevante do log; build SUCCESS apos correcao:

## Verificacao final
[ ] Dockerfile [ ] compose.yaml [ ] app.py [ ] tests/ [ ] Jenkinsfile
[ ] Historico Git [ ] Tag v1.0.0 [ ] Evidencias E01-E07

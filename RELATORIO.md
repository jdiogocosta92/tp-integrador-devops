# Relatório de evidências - Trabalho Prático

Nome: João Costa   Data: 24/11/2026
URL do repositório Git: https://github.com/jdiogocosta92/tp-integrador-devops

# EXERCÍCIO 00 — PREPARAÇÃO E VALIDAÇÃO DO AMBIENTE
```
docker --version        -> Docker version 29.9.2, build a82e528
docker compose version  -> Docker Compose version v5.6.1
git --version           -> git version 2.56.1.windows.1
docker info             -> Server Version: 29.9.2 | OSType: linux | Containers: 3
```
Componentes do projeto: a API está no app.py, os testes em tests/test_app.py, o Compose no compose.yaml (mais o incidente.yaml) e a pipeline no Jenkinsfile.

Conclusão: o Docker está a responder e a correr containers Linux. Tenho o Compose V2 e o Git instalados.

# EXERCÍCIO 01 — DOCKERFILE E EXECUÇÃO DE UM CONTAINER
```
docker build -t natixis-support:1.0 .
docker run -d --name tp-api -p 127.0.0.1:18080:8000 natixis-support:1.0
docker ps                               -> 127.0.0.1:18080->8000/tcp   tp-api
curl.exe http://127.0.0.1:18080/health  -> {"status":"ok","service":"support-api","version":"1.0"}
docker logs --tail 15 tp-api            -> "GET /health HTTP/1.1" 200 OK
docker exec tp-api id                   -> uid=999(appuser) gid=999(appgroup) groups=999(appgroup)
docker rm -f tp-api
```
Tag: natixis-support:1.0. Porta 18080 no host e 8000 no container.

Conclusão: a imagem foi criada e a API respondeu com 200 e status "ok". O container não corre como root, corre com o utilizador appuser.

# EXERCÍCIO 02 — DOCKER COMPOSE, DNS E PERSISTÊNCIA
```
docker compose config --quiet   -> sem erros
docker compose up -d --build
docker compose ps               -> natixis-integrador-api-1   Up 5 seconds (healthy)
docker compose --profile diagnostico run --rm checker -fsS http://api:8000/health
   -> Container natixis-integrador-api-1 Healthy
   -> {"status":"ok","service":"support-api-integrador","version":"1.0"}
curl.exe http://127.0.0.1:18080/visits           -> {"visits":0}
curl.exe -X POST http://127.0.0.1:18080/visits   -> {"visits":1}
curl.exe http://127.0.0.1:18080/visits           -> {"visits":1}
docker compose up -d --force-recreate
docker compose ps                                -> Up 6 seconds (healthy)
curl.exe http://127.0.0.1:18080/visits           -> {"visits":1}
```
Conclusão: o checker só arrancou depois de a api estar healthy e chegou-lhe pelo nome "api". O contador ficou em 1 depois de recriar o container porque o valor está guardado no volume app-data e não dentro do container.

# EXERCÍCIO 03 — TROUBLESHOOTING E RECUPERAÇÃO DO SERVIÇO
Sintoma: depois de aplicar o incidente.yaml a api ficou unhealthy e o curl ao /health deu "curl: (52) Empty reply from server".

Diagnóstico:
```
docker compose -f compose.yaml -f incidente.yaml ps
   -> Up About a minute (unhealthy)   127.0.0.1:18080->8000/tcp
docker compose -f compose.yaml -f incidente.yaml logs --tail 30 api
   -> Uvicorn running on http://0.0.0.0:8001
docker compose -f compose.yaml -f incidente.yaml config
   -> command com --port "8001", ports com target: 8000, healthcheck em 127.0.0.1:8000
docker inspect --format "{{json .State.Health}}" natixis-integrador-api-1
   -> "Status":"unhealthy","FailingStreak":10 ... Connection refused
docker compose -f compose.yaml -f incidente.yaml exec api python -c "import urllib.request as u; print(u.urlopen('http://127.0.0.1:8001/health').read())"
   -> {"status":"ok","service":"support-api-integrador","version":"1.0"}
```
Causa: o incidente.yaml põe a API a escutar na porta 8001, mas o mapeamento de portas e o healthcheck continuam a usar a 8000. A aplicação estava bem (respondia na 8001), o problema era só a porta.

Correção: voltei à configuração base, sem o incidente.yaml, e recriei só a api sem apagar o volume.
```
docker compose up -d --force-recreate api
```
Validação:
```
docker compose ps                        -> Up 2 minutes (healthy)
curl.exe http://127.0.0.1:18080/health   -> {"status":"ok","service":"support-api-integrador","version":"1.0"}
curl.exe http://127.0.0.1:18080/visits   -> {"visits":1}
```
O contador ficou com o mesmo valor que tinha antes do incidente (1).

# EXERCÍCIO 04 — GIT, BRANCH, MERGE E RELEASE
```
git init
git check-ignore -v .venv/x .env __pycache__/x.pyc .pytest_cache/x   -> os 4 ficam ignorados pelo .gitignore
git add .
git commit -m "feat: ambiente integrador funcional"        -> fef0757
git branch -M main
git switch -c fix/health-info
   (app.py: "team": "support" no /health | teste: assert response.json()["team"] == "support")
   (testes corridos num container python:3.13-slim -> 2 passed)
git add app.py tests/test_app.py
git commit -m "fix: acrescentar identificacao no health"   -> d132023
git switch main
git merge fix/health-info                                  -> Updating fef0757..d132023, Fast-forward
git tag -a v1.0.0 -m "Release integrador 1.0.0"
git push -u origin main
git push origin v1.0.0
```
```
* d132023 (HEAD -> main, tag: v1.0.0, fix/health-info) fix: acrescentar identificacao no health
* fef0757 feat: ambiente integrador funcional
```
Conclusão: fiz a alteração na branch fix/health-info, confirmei que os testes passavam e só depois fiz o merge na main. A main e a tag v1.0.0 estão publicadas no GitHub.

# EXERCÍCIO 05 — JENKINS: PIPELINE DE INTEGRAÇÃO CONTÍNUA
Job tp-integrador-devops (Pipeline script from SCM, branch main, Jenkinsfile).

Build #2 - http://localhost:8080/job/tp-integrador-devops/2/ - SUCCESS nas 3 etapas (Checkout, Tests e Docker Build)
```
2 passed
Successfully tagged natixis-support:ci-2
+ docker image inspect natixis-support:ci-2 --format {{.Id}}
sha256:c768d275923e90d1f518ca2f831d68b31d3a233e3e99a33e65bdad231284bb7f
Finished: SUCCESS
```
Nota: o build #1 falhou na etapa Docker Build com "docker: not found", porque o agente Jenkins não tinha o comando docker instalado. Acrescentei a CLI do Docker ao Dockerfile do agente (na pasta do laboratório Jenkins), reconstruí o agente e o build #2 já passou.

Conclusão: a pipeline vai buscar o código ao GitHub, corre os testes e constrói a imagem com o número do build na tag.

# EXERCÍCIO 06 — FALHA CONTROLADA DA PIPELINE E RECUPERAÇÃO
Mudei o teste para esperar status "offline" (commit c31b19a) e corri o build #3.

Build #3 - http://localhost:8080/job/tp-integrador-devops/3/ - FAILURE na etapa Tests
```
FAILED tests/test_app.py::test_health - AssertionError: assert 'ok' == 'offline'
1 failed, 1 passed, 1 warning in 0.51s
Stage "Docker Build" skipped due to earlier failure(s)
Finished: FAILURE
```
Voltei a pôr "ok" no teste (commit 0156215) e corri o build #4.

Build #4 - http://localhost:8080/job/tp-integrador-devops/4/ - SUCCESS
```
Successfully tagged natixis-support:ci-4
sha256:c768d275923e90d1f518ca2f831d68b31d3a233e3e99a33e65bdad231284bb7f
Finished: SUCCESS
```
Conclusão: com o teste a falhar, a pipeline parou nos Tests e não construiu a imagem. Depois de corrigir voltou a passar. A imagem do #4 tem o mesmo ID da do #2 porque só mexi nos testes, que não entram na imagem.

Captura do Stage View: evidencias/Stage View.png
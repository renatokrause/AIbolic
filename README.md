# Lineage API + Ollama + MLflow

## Visão geral

Este documento registra a arquitetura final, as decisões técnicas, os ajustes realizados, os testes executados e os procedimentos operacionais relacionados à solução de lineage com uso de LLM.

O trabalho teve como objetivo principal desacoplar o processamento de IA do servidor `rodpro19`, migrando o Ollama e os modelos para um servidor físico com GPU NVIDIA A100, mantendo o `lineage-api` no ambiente original e preservando a integração com MLflow.

Ao final do trabalho, a solução foi testada ponta a ponta e considerada pronta para uso e continuidade do desenvolvimento pela equipe.

---

# 1. Arquitetura final

A arquitetura resultante ficou organizada da seguinte forma:

```text
                         rodpro19
                            |
                    +----------------+
                    |  lineage-api   |
                    |     :8000      |
                    +----------------+
                      |            |
                      |            |
                      v            v
                +-----------+   +---------------------+
                |  MLflow   |   | Ollama remoto      |
                |   :5000   |   | 172.26.251.91:11434|
                +-----------+   +---------------------+
                      |                    |
                      |                    v
               lineage_prod            cdswpro01
                                           |
                                      qwen3:14b
                                           |
                                   NVIDIA A100 80 GB
```

Fluxo principal:

```text
Usuário / Aplicação
        |
        v
lineage-api
        |
        +--> SQLGlot
        |
        +--> Parser Agentic
                    |
                    v
                  Ollama
                    |
                    v
                 qwen3:14b
                    |
                    v
              NVIDIA A100
```

Em paralelo:

```text
lineage-api
    |
    v
MLflow
    |
    v
lineage_prod
```

---

# 2. Servidores envolvidos

## rodpro19

Responsabilidades atuais:

- execução do `lineage-api`;
- execução do MLflow;
- armazenamento da configuração da aplicação;
- armazenamento persistente da base de autenticação do MLflow OIDC;
- comunicação com o Ollama remoto.

Principais portas:

```text
8000/tcp   Lineage API
5000/tcp   MLflow
```

O Ollama não deve mais ser executado neste servidor.

---

## cdswpro01

Responsabilidades atuais:

- execução do Ollama;
- armazenamento dos modelos;
- execução de inferência na GPU;
- compartilhamento da NVIDIA A100 com outros workloads existentes no servidor.

Endereço utilizado pelo `lineage-api`:

```text
http://172.26.251.91:11434
```

Hardware relevante:

```text
GPU: NVIDIA A100 80 GB PCIe
Driver NVIDIA: 580.178.04
CUDA suportado pelo driver: 13.0
MIG: desabilitado
```

---

# 3. Ollama

## Container

O Ollama é executado no `cdswpro01` utilizando Podman.

Nome do container:

```text
ollama-sefaz
```

Imagem:

```text
docker.io/library/ollama-sefaz:1.0.3
```

Versão observada do Ollama:

```text
0.20.3
```

O container utiliza diretamente o runtime NVIDIA:

```text
/usr/bin/nvidia-container-runtime
```

Foi necessário utilizar esta abordagem devido a incompatibilidade observada entre a versão do Podman/CDI existente no servidor e a sintaxe:

```text
--device nvidia.com/gpu=all
```

A configuração funcional utilizada foi baseada em:

```text
--runtime=/usr/bin/nvidia-container-runtime
--security-opt=label=disable
NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=all
```

---

# 4. Persistência dos modelos

Os modelos são armazenados no host em:

```text
/data/03/ollama/data
```

Esse caminho é montado dentro do container como:

```text
/data/ollama_models_v2
```

Variável utilizada:

```text
OLLAMA_MODELS=/data/ollama_models_v2/models
```

Isso garante que os modelos não sejam perdidos em recriações do container.

---

# 5. Modelos disponíveis

Durante a migração foram validados os seguintes modelos:

```text
qwen3:14b
llama3.1:8b
command-r7b:7b
```

Modelo utilizado atualmente pelo `lineage-api`:

```text
qwen3:14b
```

Características observadas:

```text
Família: qwen3
Parâmetros: 14.8B
Formato: GGUF
Quantização: Q4_K_M
```

---

# 6. Configuração do Qwen

O parser agentic e o judge utilizam:

```text
PARSER_AGENT_MODEL=qwen3:14b
JUDGE_AGENT_MODEL=qwen3:14b
```

Contexto configurado:

```text
PARSER_AGENT_NUM_CTX=32768
JUDGE_AGENT_NUM_CTX=32768
```

Durante o teste real do fluxo agentic foi observado:

```text
Modelo: qwen3:14b
Processor: 100% GPU
Context: 32768
VRAM utilizada: aproximadamente 14 GB
```

A GPU disponível possui 80 GB, deixando margem significativa para compartilhamento com outros workloads.

---

# 7. Compartilhamento da GPU

O Ollama utiliza a mesma NVIDIA A100 que também pode ser utilizada pelos workloads do ambiente CML/RKE2.

Foi deliberadamente adotado, nesta etapa, um modelo de compartilhamento flexível.

Isso significa que:

- o Ollama roda diretamente no host via Podman;
- o Kubernetes/RKE2 também enxerga a GPU;
- não existe, neste momento, coordenação formal de VRAM entre o workload host e os workloads Kubernetes;
- a utilização deve ser acompanhada operacionalmente.

Essa abordagem foi considerada adequada para esta fase de PoC/desenvolvimento.

Para ambientes de maior criticidade ou concorrência, deverá ser considerada posteriormente uma estratégia de isolamento, quotas, MIG ou gerenciamento centralizado de aceleradores.

---

# 8. Inicialização automática do Ollama

O container foi integrado ao systemd.

Serviço:

```text
container-ollama-sefaz.service
```

Verificar se está habilitado:

```bash
systemctl is-enabled container-ollama-sefaz.service
```

Verificar estado:

```bash
systemctl is-active container-ollama-sefaz.service
```

Consultar detalhes:

```bash
systemctl status container-ollama-sefaz.service
```

Reiniciar:

```bash
systemctl restart container-ollama-sefaz.service
```

Durante a implantação foram testados:

```text
systemctl stop
systemctl start
```

e o container voltou normalmente.

O serviço encontra-se habilitado para inicialização automática junto com o sistema operacional.

Observação:

`podman generate systemd` informou que a abordagem é considerada legada em versões mais recentes do Podman e recomenda Quadlets.

Neste momento, entretanto, a configuração existente está funcional e foi mantida.

Uma futura manutenção pode migrar o serviço para Quadlet.

---

# 9. Lineage API

Servidor:

```text
rodpro19
```

Container:

```text
lineage-api
```

Imagem:

```text
lineage-api:1.0.3
```

Porta:

```text
8000
```

Projeto:

```text
/root/projects/sefaz-llm
```

Arquivo principal de Compose:

```text
/root/projects/sefaz-llm/docker-compose.yml
```

---

# 10. Docker Compose final

Após a migração, o serviço Ollama foi removido do Compose do `rodpro19`.

O Compose deve conter apenas:

```text
api
```

Validação:

```bash
cd /root/projects/sefaz-llm
docker compose config --services
```

Resultado esperado:

```text
api
```

Isso evita que um comando como:

```bash
docker compose up -d
```

tente recriar inadvertidamente um Ollama local.

---

# 11. Configuração do Ollama no Lineage API

As seguintes URLs apontam para o Ollama remoto:

```text
OLLAMA_BASE_URL=http://172.26.251.91:11434

PARSER_AGENT_BASE_URL=http://172.26.251.91:11434

JUDGE_AGENT_BASE_URL=http://172.26.251.91:11434
```

Modelo:

```text
qwen3:14b
```

Configuração relevante:

```text
PARSER_AGENT_TEMPERATURE=0.7
PARSER_AGENT_MAX_TOKENS=8192
PARSER_AGENT_NUM_CTX=32768
PARSER_AGENT_REPEAT_PENALTY=1.1

JUDGE_AGENT_TEMPERATURE=0.2
JUDGE_AGENT_MAX_TOKENS=8192
JUDGE_AGENT_NUM_CTX=32768
JUDGE_AGENT_REPEAT_PENALTY=1.1

RETRY_NODE_MAX_RETRIES=2
```

---

# 12. Health check

Endpoint:

```text
GET /health
```

Teste:

```bash
curl -fsS http://127.0.0.1:8000/health && echo
```

Resposta esperada:

```json
{
  "status": "ok",
  "message": "API está funcionando normalmente. Ollama: ok"
}
```

Esse health check valida:

- funcionamento do `lineage-api`;
- conectividade da aplicação com o Ollama remoto.

---

# 13. Endpoint de processamento

Endpoint:

```text
POST /query-parse
```

Payload:

```json
{
  "sql_query": "SELECT ..."
}
```

Exemplo:

```bash
curl -s \
  http://127.0.0.1:8000/query-parse \
  -H 'Content-Type: application/json' \
  -d '{
    "sql_query": "SELECT c.id, c.nome FROM clientes c"
  }' \
  | python3 -m json.tool
```

---

# 14. Estratégia de parsing

O `lineage-api` possui dois caminhos principais:

```text
SQLGlot
Agentic
```

O router utiliza heurísticas baseadas na complexidade da SQL.

Exemplos de critérios para encaminhamento ao agente:

```text
mais de 2 CTEs
mais de 3 subqueries
mais de 5 JOINs
expressões complexas combinadas com CTEs/subqueries
UPDATE auto-referencial
UPDATE com subquery no FROM
erro de parsing pelo SQLGlot
```

Queries simples são normalmente processadas diretamente pelo SQLGlot.

Queries mais complexas são enviadas ao parser agentic utilizando o Qwen.

---

# 15. Teste do parser Agentic

Foi utilizado um teste com três CTEs para garantir o encaminhamento para o agente.

Payload de teste:

```bash
cat >/tmp/agentic-test.json <<'EOF'
{
  "sql_query": "WITH vendas AS (SELECT cliente_id, SUM(valor) AS total FROM pedidos GROUP BY cliente_id), clientes_ativos AS (SELECT id, nome FROM clientes WHERE ativo = true), consolidado AS (SELECT c.id, c.nome, v.total FROM clientes_ativos c JOIN vendas v ON v.cliente_id = c.id) SELECT id, nome, total FROM consolidado WHERE total > 1000"
}
EOF
```

Execução:

```bash
time curl -s \
  http://127.0.0.1:8000/query-parse \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/agentic-test.json \
  > /tmp/agentic-result.json
```

Resumo:

```bash
python3 - <<'PY'
import json

with open("/tmp/agentic-result.json") as f:
    x = json.load(f)

print("success:", x.get("success"))
print("parser_used:", x.get("parser_used"))
print("error_message:", x.get("error_message"))
print("warnings:", x.get("warnings"))
PY
```

Resultado validado:

```text
success: True
parser_used: agentic
error_message: None
warnings: []
```

Esse teste comprovou o fluxo completo:

```text
lineage-api
   |
   v
router
   |
   v
parser agentic
   |
   v
Ollama
   |
   v
qwen3:14b
   |
   v
NVIDIA A100
```

---

# 16. MLflow

Servidor utilizado pela aplicação:

```text
http://172.26.252.29:5000
```

Experimento:

```text
lineage_prod
```

ID observado:

```text
4
```

Configuração da API:

```text
MLFLOW_TRACKING_URI=http://172.26.252.29:5000
MLFLOW_EXPERIMENT_NAME=lineage_prod
```

---

# 17. MLflow OIDC Authentication

O MLflow utiliza:

```text
mlflow-oidc-auth
```

Versão observada:

```text
7.3.1
```

Durante a migração foi identificado um problema importante.

O banco de autenticação OIDC estava localizado originalmente em:

```text
/mlflow/auth.db
```

dentro do filesystem do container.

Como esse arquivo não era persistente, toda recriação do container apagava:

- contas locais;
- service accounts;
- tokens;
- permissões associadas.

---

# 18. Persistência do auth.db

Foi criado o diretório:

```text
/data/mlflow-auth
```

Banco persistente:

```text
/data/mlflow-auth/auth.db
```

No Compose do MLflow foi adicionado:

```yaml
volumes:
  - /data/mlflow:/data/mlflow
  - /data/mlflow-auth/auth.db:/mlflow/auth.db
```

A persistência foi validada com:

```bash
docker compose up -d --force-recreate mlflow
```

Após a recriação, as contas continuaram existentes.

---

# 19. Allowed Hosts do MLflow

Também foi necessário ajustar a configuração de hosts permitidos.

Foram incluídos:

```text
mlflow.sefaz.rs.gov.br
mlflow.sefaz.rs.gov.br:443
localhost
localhost:5000
127.0.0.1
127.0.0.1:5000
172.26.252.29
172.26.252.29:5000
```

Variáveis:

```text
MLFLOW_ALLOWED_HOSTS
MLFLOW_SERVER_ALLOWED_HOSTS
```

Antes desse ajuste, chamadas internas autenticadas podiam retornar:

```text
403 Invalid Host header
```

---

# 20. Service Account do Lineage API

Foi criada uma service account específica:

```text
lineage-api@sefaz.rs.gov.br
```

Características finais:

```text
is_service_account=True
is_admin=False
```

Ela não possui privilégios administrativos globais.

---

# 21. Permissão da Service Account

A service account recebeu explicitamente:

```text
EDIT
```

sobre:

```text
experiment_id=4
lineage_prod
```

A permissão `EDIT` foi escolhida porque permite:

- leitura do experimento;
- criação e atualização de runs;
- registro de métricas;
- registro das informações necessárias pela aplicação.

Ela não permite administrar permissões, função reservada ao nível `MANAGE`.

---

# 22. Credenciais do MLflow

As credenciais da service account são utilizadas pelo `lineage-api` através das variáveis:

```text
MLFLOW_TRACKING_USERNAME
MLFLOW_TRACKING_PASSWORD
```

O token local utilizado durante a configuração foi armazenado em:

```text
/root/projects/sefaz-llm/.mlflow-lineage-token
```

Esse arquivo deve possuir permissão restrita:

```bash
chmod 600 /root/projects/sefaz-llm/.mlflow-lineage-token
```

O token nunca deve ser incluído em:

- documentação;
- Git;
- e-mails;
- chats;
- prints;
- scripts públicos.

---

# 23. Arquivo .env

Arquivo:

```text
/root/projects/sefaz-llm/.env
```

Deve conter:

```text
MLFLOW_TRACKING_USERNAME=lineage-api@sefaz.rs.gov.br
MLFLOW_TRACKING_PASSWORD=<TOKEN>
```

O arquivo também deve ter permissões restritas:

```bash
chmod 600 /root/projects/sefaz-llm/.env
```

Não versionar `.env`.

Recomendação de `.gitignore`:

```gitignore
.env
.mlflow-lineage-token
*.token
*.secret
```

---

# 24. Teste do MLflow

Teste autenticado:

```bash
cd /root/projects/sefaz-llm

TOKEN="$(cat .mlflow-lineage-token)"

curl -s -o /tmp/mlflow-check.json \
  -w 'HTTP=%{http_code}\n' \
  -u "lineage-api@sefaz.rs.gov.br:${TOKEN}" \
  "http://172.26.252.29:5000/api/2.0/mlflow/experiments/get-by-name?experiment_name=lineage_prod"

unset TOKEN

python3 -m json.tool /tmp/mlflow-check.json
```

Resultado esperado:

```text
HTTP=200
```

---

# 25. Teste rápido diário

No `rodpro19`, o teste mais simples é:

```bash
curl -fsS http://127.0.0.1:8000/health && echo
```

Resultado esperado:

```text
status: ok
Ollama: ok
```

Esse é o comando recomendado para uma verificação rápida.

---

# 26. Teste completo no rodpro19

```bash
cd /root/projects/sefaz-llm

echo "===== LINEAGE API ====="
docker ps --filter name=lineage-api

echo
echo "===== HEALTH ====="
curl -fsS http://127.0.0.1:8000/health && echo

echo
echo "===== MLFLOW ====="
TOKEN="$(cat .mlflow-lineage-token)"

curl -s -o /tmp/mlflow-check.json \
  -w 'HTTP=%{http_code}\n' \
  -u "lineage-api@sefaz.rs.gov.br:${TOKEN}" \
  "http://172.26.252.29:5000/api/2.0/mlflow/experiments/get-by-name?experiment_name=lineage_prod"

unset TOKEN

python3 -m json.tool /tmp/mlflow-check.json

echo
echo "===== TESTE AGENTIC ====="

curl -s \
  http://127.0.0.1:8000/query-parse \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/agentic-test.json \
  > /tmp/agentic-check.json

python3 - <<'PY'
import json

x = json.load(open("/tmp/agentic-check.json"))

print("success:", x.get("success"))
print("parser_used:", x.get("parser_used"))
print("error_message:", x.get("error_message"))
print("warnings:", x.get("warnings"))
PY
```

Resultado esperado:

```text
lineage-api: Up

health:
status = ok
Ollama = ok

MLflow:
HTTP=200

Agentic:
success=True
parser_used=agentic
error_message=None
warnings=[]
```

---

# 27. Teste completo no cdswpro01

```bash
echo "===== OLLAMA SYSTEMD ====="

systemctl is-enabled container-ollama-sefaz.service
systemctl is-active container-ollama-sefaz.service

echo
echo "===== CONTAINER ====="

podman ps --filter name=ollama-sefaz

echo
echo "===== MODELOS DISPONIVEIS ====="

podman exec ollama-sefaz ollama list

echo
echo "===== MODELOS CARREGADOS ====="

podman exec ollama-sefaz ollama ps

echo
echo "===== GPU ====="

nvidia-smi
```

Resultados esperados:

```text
systemd:
enabled
active

container:
ollama-sefaz Up

GPU:
NVIDIA A100 80GB
```

Durante processamento agentic:

```text
qwen3:14b
100% GPU
context 32768
```

---

# 28. Monitoramento da GPU

Para observar a GPU em tempo real:

```bash
watch -n 1 nvidia-smi
```

Para observar modelos carregados:

```bash
watch -n 1 'podman exec ollama-sefaz ollama ps'
```

Por padrão, o Ollama descarrega o modelo da memória depois de alguns minutos sem utilização.

Isso é desejável neste momento para permitir que a GPU continue disponível para outros workloads.

---

# 29. Reinício do Lineage API

```bash
cd /root/projects/sefaz-llm

docker compose up -d --force-recreate api
```

A aplicação pode demorar alguns segundos para concluir o startup.

Por isso, após recriação, utilizar:

```bash
for i in $(seq 1 12); do
  echo "Tentativa $i"

  if curl -fsS http://127.0.0.1:8000/health; then
    echo
    break
  fi

  sleep 5
done
```

Durante os testes foi observado `Connection reset by peer` nos primeiros segundos após recriação.

Isso ocorreu apenas porque o container já estava iniciado enquanto o Uvicorn ainda concluía o startup.

---

# 30. Logs

## Lineage API

```bash
docker logs --tail=200 lineage-api
```

Acompanhar em tempo real:

```bash
docker logs -f lineage-api
```

---

## MLflow

```bash
docker logs --tail=200 mlflow_server
```

Em tempo real:

```bash
docker logs -f mlflow_server
```

---

## Ollama

```bash
podman logs --tail=200 ollama-sefaz
```

Em tempo real:

```bash
podman logs -f ollama-sefaz
```

Ou via systemd:

```bash
journalctl -u container-ollama-sefaz.service
```

Em tempo real:

```bash
journalctl -fu container-ollama-sefaz.service
```

---

# 31. Comandos úteis do Ollama

Listar modelos:

```bash
podman exec ollama-sefaz ollama list
```

Ver modelos carregados:

```bash
podman exec ollama-sefaz ollama ps
```

Testar API:

```bash
curl http://172.26.251.91:11434/api/tags
```

Testar inferência diretamente:

```bash
curl http://172.26.251.91:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3:14b",
    "prompt": "Responda apenas: Ollama funcionando.",
    "stream": false
  }'
```

---

# 32. Segurança

## Ollama

O Ollama não possui autenticação própria nessa configuração.

A porta:

```text
172.26.251.91:11434
```

deve ser considerada uma interface interna.

Não deve ser exposta diretamente a redes não confiáveis.

Para evolução da arquitetura corporativa, o acesso aos modelos deverá preferencialmente ocorrer através de:

```text
LLM Gateway
```

com:

- autenticação;
- autorização;
- rate limiting;
- quotas;
- observabilidade;
- políticas;
- guardrails;
- auditoria.

---

# 33. MLflow

O acesso do `lineage-api` ao MLflow utiliza uma conta específica e de privilégio mínimo.

Configuração final:

```text
Service Account: lineage-api@sefaz.rs.gov.br
Admin: não
Experimento: lineage_prod
Permission: EDIT
```

Não voltar a utilizar conta administrativa para a aplicação.

---

# 34. Arquivos sensíveis

Nunca versionar ou compartilhar:

```text
.env
.mlflow-lineage-token
tokens
senhas
client secrets
API keys
```

Durante o diagnóstico e inventário do ambiente foram encontrados arquivos contendo credenciais e secrets.

Esses arquivos devem ser tratados como sensíveis.

Caso algum deles tenha sido compartilhado fora de ambiente controlado, recomenda-se rotação das respectivas credenciais.

---

# 35. Containers antigos

O serviço Ollama foi removido do Compose do `rodpro19`.

Durante a limpeza ainda existia um container histórico parado:

```text
ollama-sefaz-container
```

Esse container não faz parte da arquitetura atual.

Pode ser removido quando conveniente:

```bash
docker rm ollama-sefaz-container
```

Antes de qualquer remoção adicional, validar sempre:

```bash
docker ps -a
```

---

# 36. Backup das configurações

Antes das alterações foram criados backups dos arquivos de Compose.

É recomendável manter essa prática antes de futuras mudanças:

```bash
cp -a docker-compose.yml \
  docker-compose.yml.bkp-$(date +%Y%m%d-%H%M%S)
```

Para o MLflow:

```bash
cp -a /opt/mlflow-docker/docker-compose.yaml \
  /opt/mlflow-docker/docker-compose.yaml.bkp-$(date +%Y%m%d-%H%M%S)
```

---

# 37. Procedimento básico de diagnóstico

Caso a aplicação apresente problema, seguir preferencialmente esta sequência.

## 1. Verificar Lineage API

```bash
docker ps --filter name=lineage-api
```

```bash
curl -fsS http://127.0.0.1:8000/health
```

---

## 2. Verificar logs

```bash
docker logs --tail=200 lineage-api
```

---

## 3. Verificar Ollama remotamente

```bash
curl http://172.26.251.91:11434/api/tags
```

---

## 4. Verificar Ollama no cdswpro01

```bash
systemctl status container-ollama-sefaz.service
```

```bash
podman ps --filter name=ollama-sefaz
```

---

## 5. Verificar GPU

```bash
nvidia-smi
```

---

## 6. Verificar MLflow

```bash
docker ps --filter name=mlflow
```

```bash
docker logs --tail=200 mlflow_server
```

---

## 7. Testar autenticação MLflow

```bash
cd /root/projects/sefaz-llm

TOKEN="$(cat .mlflow-lineage-token)"

curl -s -o /tmp/mlflow-test.json \
  -w 'HTTP=%{http_code}\n' \
  -u "lineage-api@sefaz.rs.gov.br:${TOKEN}" \
  "http://172.26.252.29:5000/api/2.0/mlflow/experiments/get-by-name?experiment_name=lineage_prod"

unset TOKEN
```

Esperado:

```text
HTTP=200
```

---

# 38. Situações conhecidas

## 404 em /

É normal:

```bash
curl http://127.0.0.1:8000/
```

retornar:

```text
404 Not Found
```

Não existe endpoint raiz.

Utilizar:

```text
/health
/query-parse
/openapi.json
```

---

## Connection reset após recreate

Pode ocorrer durante os primeiros segundos após:

```bash
docker compose up -d --force-recreate api
```

Aguardar o startup ou utilizar o loop de health check.

---

## Modelo não aparece no ollama ps

Isso não significa falha.

`ollama ps` mostra apenas modelos carregados naquele momento.

Utilizar:

```bash
podman exec ollama-sefaz ollama list
```

para listar todos os modelos instalados.

---

## VRAM cai após alguns minutos

Comportamento normal.

Ollama descarrega o modelo após determinado período de inatividade.

Isso libera VRAM para os demais workloads.

---

# 39. Testes realizados antes da liberação

Antes da liberação para a equipe foram executados e aprovados:

```text
Ollama API                      OK
Qwen3:14b                       OK
Inferência direta               OK
NVIDIA A100                     OK
GPU 100% utilizada pelo Qwen    OK
Contexto 32768                  OK
Lineage API startup             OK
Lineage API health              OK
Lineage -> Ollama               OK
SQLGlot                         OK
Parser Agentic                  OK
Agentic -> Qwen                 OK
Agentic -> A100                 OK
MLflow                          OK
MLflow authentication           OK
MLflow allowed hosts            OK
MLflow auth.db persistence      OK
Service Account                 OK
Privilégio mínimo EDIT          OK
Service Account sem admin       OK
Compose sem Ollama local        OK
Recreate Lineage API            OK
Recreate MLflow                 OK
Systemd Ollama                  OK
```

---

# 40. Resultado final do teste Agentic

No teste final:

```text
success: True
parser_used: agentic
error_message: None
warnings: []
table_entities: 2
```

Tempo observado em uma das execuções finais:

```text
aproximadamente 56 segundos
```

Em execução anterior, com carga inicial do modelo:

```text
aproximadamente 1 minuto e 50 segundos
```

Essa diferença é esperada devido ao carregamento do modelo e estado da GPU.

---

# 41. Estado final do ambiente

## rodpro19

```text
lineage-api
    status: ativo

MLflow
    status: ativo

Ollama local
    não utilizado
```

---

## cdswpro01

```text
Ollama
    status: ativo

qwen3:14b
    disponível

NVIDIA A100 80GB
    funcional

systemd
    Ollama habilitado para inicialização automática
```

---

# 42. Considerações para evolução futura

Este trabalho encerra a migração e estabilização atual.

Possíveis evoluções futuras:

- colocar o acesso ao Ollama atrás do LLM Gateway corporativo;
- substituir IP fixo por DNS/FQDN interno;
- adicionar TLS entre componentes;
- usar gerenciamento centralizado de secrets;
- migrar o serviço Podman/systemd para Quadlet;
- implementar observabilidade específica de GPU;
- controlar concorrência e consumo de VRAM;
- avaliar MIG;
- avaliar compartilhamento e scheduling corporativo das GPUs;
- criar dashboards de inferência;
- criar métricas de latência/token;
- implementar quotas;
- implementar rate limiting;
- adicionar guardrails;
- integrar RBAC corporativo;
- monitorar disponibilidade do modelo;
- criar testes automatizados de health;
- criar pipeline CI/CD da aplicação;
- criar alertas de falha do `lineage-api`, MLflow e Ollama.

Essas melhorias não são bloqueantes para o ambiente atual.

---

# 43. Resumo executivo

O trabalho resultou na separação entre aplicação e infraestrutura de inferência.

Antes:

```text
rodpro19
├── lineage-api
└── Ollama/GPU
```

Depois:

```text
rodpro19
└── lineage-api
       |
       +--> MLflow
       |
       +--> Ollama remoto

cdswpro01
└── Ollama
      |
      └── qwen3:14b
             |
             └── NVIDIA A100 80GB
```

Benefícios obtidos:

- melhor aproveitamento da GPU;
- desacoplamento da aplicação;
- possibilidade de compartilhamento do acelerador;
- persistência adequada dos modelos;
- persistência da autenticação MLflow;
- autenticação de aplicação com service account;
- princípio de menor privilégio;
- simplificação do Compose;
- eliminação da dependência do Ollama local;
- capacidade de evolução independente entre aplicação e infraestrutura;
- preparação para futura integração com uma plataforma corporativa de IA.

---

# 44. Status

```text
STATUS FINAL: LIBERADO PARA USO E DESENVOLVIMENTO
```

Data da conclusão:

```text
10/09/2026
```

A solução foi testada ponta a ponta após todas as alterações finais.
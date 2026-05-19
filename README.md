# Evaluation Service ⚡

Serviço de avaliação de feature flags do **ToggleMaster**. Este é o serviço crítico de "hot path" (caminho crítico) onde clientes finais fazem requisições para avaliar se uma feature flag está ativa para um usuário específico.

## 🎯 Descrição do Serviço

O Evaluation Service é otimizado para alta velocidade e baixa latência. Ele:

1. Recebe requisições de avaliação de flags (`/evaluate?user_id=...&flag_name=...`)
2. Busca regras da flag em cache Redis (alta velocidade)
3. Em caso de cache miss: busca definição no Flag Service e regras no Targeting Service
4. Executa a lógica de avaliação (ex: usuário está nos 50%?)
5. Retorna `true` ou `false` imediatamente ao cliente
6. Envia eventos assincronamente para AWS SQS (para análise posterior)

**Crítico para:** Qualquer cliente que precisa saber se uma flag está ativa para um usuário específico em tempo real.

## 📦 Stack Técnico

- **Linguagem:** Go 1.21+
- **Framework:** Gin Web Framework
- **Cache:** Redis
- **Filas:** AWS SQS
- **Dependências principais:** go-redis/redis, aws-sdk-go, gin-gonic/gin

## 🚀 Como Usar

### Pré-requisitos Locais

- Go 1.21 ou superior
- Redis 6+ (instalado ou via Docker)
- Conta AWS com permissões para SQS
- Os seguintes serviços rodando:
  - Auth Service (porta 8001)
  - Flag Service (porta 8002)
  - Targeting Service (porta 8003)

### Setup Local

#### 1. Clone e Navegue para o Diretório
```bash
cd Evaluation-Service
```

#### 2. Crie uma Chave de API de Serviço

Este serviço precisa se autenticar nos outros serviços. Use o Auth Service para criar uma chave:

```bash
curl -X POST http://localhost:8001/admin/keys \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer admin-secreto-123" \
  -d '{"name": "evaluation-service"}'

# Resposta esperada:
# {
#   "key": "tm_key_a1b2c3d4e5f6g7h8...",
#   "message": "Guarde esta chave com segurança!"
# }
```

Guarde a chave retornada para usar no `.env`.

#### 3. Configure as Variáveis de Ambiente
Crie um arquivo `.env` na raiz do serviço:

```env
# Serviço
PORT=8004

# Redis - Cache das regras de flags
REDIS_URL=redis://localhost:6379
REDIS_DB=0
CACHE_TTL_SECONDS=300

# URLs dos serviços dependentes
FLAG_SERVICE_URL=http://localhost:8002
TARGETING_SERVICE_URL=http://localhost:8003

# Chave de API para autenticação nos serviços
SERVICE_API_KEY=tm_key_a1b2c3d4e5f6g7h8...

# AWS SQS - Para enviar eventos
AWS_REGION=us-east-1
AWS_SQS_URL=https://sqs.us-east-1.amazonaws.com/123456789012/togglemaster-events

# Configurações opcionais
ENVIRONMENT=development
LOG_LEVEL=INFO
```

#### 4. Instale as Dependências
```bash
go mod tidy
```

#### 5. Inicie Redis (se não estiver rodando)
```bash
# Via Docker
docker run -d -p 6379:6379 redis:7-alpine

# Ou se estiver instalado localmente
redis-server
```

#### 6. Inicie o Serviço
```bash
go run .
```

O servidor estará disponível em `http://localhost:8004`.

### Testando Localmente

#### Health Check
```bash
curl http://localhost:8004/health
# Resposta esperada: {"status":"ok"}
```

#### Avaliar uma Flag
```bash
curl "http://localhost:8004/evaluate?user_id=user-123&flag_name=enable-new-dashboard"

# Resposta esperada:
# {"flag_name":"enable-new-dashboard","user_id":"user-123","enabled":true}
```

#### Testar com Diferentes Usuários
```bash
# Usuário 1
curl "http://localhost:8004/evaluate?user_id=user-001&flag_name=enable-new-dashboard"

# Usuário 2
curl "http://localhost:8004/evaluate?user_id=user-002&flag_name=enable-new-dashboard"

# Mesmo usuário com flag diferente
curl "http://localhost:8004/evaluate?user_id=user-001&flag_name=beta-feature"
```

#### Monitorar Cache
```bash
# Conectar ao Redis
redis-cli

# Ver todas as chaves em cache
KEYS *

# Ver uma chave específica
GET "flag:enable-new-dashboard"
```

## 🔧 Variáveis de Ambiente

### Obrigatórias
| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `REDIS_URL` | URL de conexão com Redis | `redis://localhost:6379` |
| `FLAG_SERVICE_URL` | URL do Flag Service | `http://localhost:8002` |
| `TARGETING_SERVICE_URL` | URL do Targeting Service | `http://localhost:8003` |
| `SERVICE_API_KEY` | Chave de API do serviço | `tm_key_...` |

### AWS SQS (Recomendadas)
| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `AWS_REGION` | Região AWS | `us-east-1` |
| `AWS_SQS_URL` | URL da fila SQS | `https://sqs.us-east-1.amazonaws.com/123456789012/togglemaster-events` |

### Opcionais
| Variável | Descrição | Padrão |
|----------|-----------|--------|
| `PORT` | Porta do servidor | `8004` |
| `REDIS_DB` | Database Redis | `0` |
| `CACHE_TTL_SECONDS` | TTL do cache em segundos | `300` |
| `ENVIRONMENT` | Ambiente (development/production) | `development` |
| `LOG_LEVEL` | Nível de log | `INFO` |

## 🔐 GitHub Secrets Necessários

Configure os seguintes secrets no GitHub para CI/CD:

```yaml
SERVICE_API_KEY
  Descrição: Chave de API do Evaluation Service
  Valor: <sua-chave-gerada>

REDIS_URL
  Descrição: URL de conexão Redis
  Valor: redis://localhost:6379

FLAG_SERVICE_URL
  Descrição: URL do Flag Service
  Valor: http://flag-service:8002

TARGETING_SERVICE_URL
  Descrição: URL do Targeting Service
  Valor: http://targeting-service:8003

AWS_REGION
  Descrição: Região AWS
  Valor: us-east-1

AWS_SQS_URL
  Descrição: URL da fila SQS
  Valor: https://sqs.us-east-1.amazonaws.com/123456789012/togglemaster-events

AWS_ACCESS_KEY_ID
  Descrição: AWS Access Key ID
  Valor: <sua-access-key>

AWS_SECRET_ACCESS_KEY
  Descrição: AWS Secret Access Key
  Valor: <sua-secret-key>

DOCKERHUB_USERNAME
  Descrição: Docker Hub username
  Valor: <seu-username>

DOCKERHUB_TOKEN
  Descrição: Docker Hub personal access token
  Valor: <seu-token>

REGISTRY_URL
  Descrição: URL do registry de container
  Valor: docker.io

SONAR_TOKEN
  Descrição: Token SonarQube
  Valor: <seu-token>
```

## 📊 Endpoints da API

| Método | Endpoint | Requer Auth | Descrição |
|--------|----------|------------|-----------|
| GET | `/health` | Não | Verifica saúde do serviço |
| GET | `/evaluate` | Não | Avalia flag para um usuário |
| GET | `/cache/stats` | Não | Estatísticas de cache (desenvolvimento) |

### Parâmetros de Query

**`/evaluate` requer:**
- `user_id` (string): ID único do usuário
- `flag_name` (string): Nome da flag a avaliar

## ⚡ Otimizações de Performance

### Cache em Redis
- Flags são cacheadas em Redis com TTL configurável
- Chave de cache: `flag:{flag_name}`
- TTL padrão: 300 segundos

### Estratégia de Cache Miss
1. Busca em Flag Service
2. Busca em Targeting Service
3. Valida resultado
4. Armazena em Redis
5. Retorna ao cliente

### Async Event Publishing
- Eventos são enviados assincronamente ao SQS
- Não bloqueia a resposta ao cliente
- Falhas de envio não afetam a avaliação

## 🔄 Fluxo de Avaliação

```
1. Cliente requisita: GET /evaluate?user_id=X&flag_name=Y
2. Evaluation Service:
   a. Verifica cache Redis → Se hit, retorna
   b. Se miss:
      - Busca definição em Flag Service
      - Busca regras em Targeting Service
      - Calcula resultado
      - Armazena em cache
   c. Envia evento async para SQS
3. Retorna resultado ao cliente
```

## 🐛 Troubleshooting

### Problema: "redis: connection refused"
**Solução:** Inicie Redis
```bash
docker run -d -p 6379:6379 redis:7-alpine
# ou
redis-server
```

### Problema: "connection refused" para Flag/Targeting Service
**Solução:** Verifique se os serviços estão rodando
```bash
curl http://localhost:8002/health  # Flag Service
curl http://localhost:8003/health  # Targeting Service
```

### Problema: Cache hits muito baixo
**Solução:** Aumente o TTL do cache em `CACHE_TTL_SECONDS`

### Problema: "Invalid API key"
**Solução:** Regenere a chave de API via Auth Service

## 📈 Monitoramento

### Métricas importantes
- **Latência de avaliação:** Deve ser < 10ms (com cache)
- **Taxa de cache hit:** Deve ser > 80%
- **Eventos enviados ao SQS:** Deve corresponder ao número de avaliações

### Logs
```bash
# Ver logs em tempo real
tail -f logs/evaluation-service.log

# Buscar erros
grep "ERROR" logs/evaluation-service.log
```

## 📚 Recursos Adicionais

- [Go Documentation](https://golang.org/doc/)
- [Redis Documentation](https://redis.io/documentation)
- [AWS SQS Documentation](https://docs.aws.amazon.com/sqs/)
- [ToggleMaster Architecture](../README.md)

## 👥 Suporte

Para dúvidas ou problemas, abra uma issue no repositório principal ou entre em contato com o time DevOps.

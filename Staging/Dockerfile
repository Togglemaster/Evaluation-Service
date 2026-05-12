# Etapa 1: Build do Go
FROM golang:1.25-alpine AS builder

WORKDIR /app

# Copia arquivos de dependências
COPY go.mod go.sum ./
RUN go mod download

# Copia todo o código
COPY . .

# 4. Instala o orchestrion (necessário para o comando abaixo funcionar)
RUN go install github.com/DataDog/orchestrion@latest

# 5. Fixa a versão do orchestrion no go.mod (necessário a partir da v1.x)
RUN orchestrion pin

# 6. Compila a aplicação
# O 'go build' agora usará o go.mod e o go.sum criados pelo 'tidy'
#RUN CGO_ENABLED=0 go build -o auth-service -ldflags "-s -w" .
RUN CGO_ENABLED=0 orchestrion go build -o evaluation-service -ldflags "-s -w" .

# Etapa 2: Container final mais leve
FROM alpine:3.23
RUN apk update && apk upgrade							 

WORKDIR /app

# Copia o binário e o .env
COPY --from=builder /app/evaluation-service .

EXPOSE 8004

# Comando padrão ao iniciar o container
CMD ["./evaluation-service"]

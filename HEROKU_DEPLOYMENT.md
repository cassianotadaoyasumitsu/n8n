# 🚀 Guia de Deploy do n8n no Heroku

Este guia detalha o processo completo de deploy do n8n no Heroku usando Docker e PostgreSQL.

## 📋 Pré-requisitos

- Conta no [Heroku](https://signup.heroku.com/)
- [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli) instalado
- [Git](https://git-scm.com/) instalado
- [Docker](https://www.docker.com/) instalado (para testes locais)

## 💰 Estimativa de Custos

| Recurso | Plano | Custo Mensal |
|---------|-------|--------------|
| Dyno | Eco | $5.00 |
| PostgreSQL | Mini | $5.00 |
| **Total** | | **~$10.00/mês** |

> ⚠️ **Nota sobre Eco Dyno**: O dyno Eco hiberna após 30 minutos de inatividade. Para uso em produção 24/7, considere o plano Basic ($7/mês).

## 🏗️ Arquitetura

```
Heroku Application
├── Web Dyno (n8n)
│   ├── Docker Container
│   ├── PORT: Dinâmico (definido pelo Heroku)
│   └── Protocolo: HTTPS
└── PostgreSQL Database
    ├── Addon: Heroku Postgres
    └── Conexão: DATABASE_URL (automática)
```

## 📦 Estrutura de Arquivos

O repositório já contém os arquivos necessários:

```
n8n/
├── heroku.yml                    # Configuração de build Docker para Heroku
├── docker-compose.heroku.yml     # Para testes locais (simula ambiente Heroku)
├── env.heroku.example           # Template de variáveis de ambiente
├── docker/
│   └── images/
│       └── n8n/
│           ├── Dockerfile        # Build da imagem Docker
│           └── docker-entrypoint.sh  # Script de inicialização (com suporte a PORT)
└── .dockerignore                # Otimização do build
```

## 🚀 Passo a Passo - Deploy Inicial

### 1. Fazer Login no Heroku

```bash
heroku login
```

### 2. Criar Aplicação no Heroku

```bash
# Substitua 'my-n8n-app' pelo nome desejado (deve ser único)
heroku create my-n8n-app
```

### 3. Adicionar PostgreSQL

```bash
# Adiciona PostgreSQL Mini ($5/mês)
heroku addons:create heroku-postgresql:mini -a my-n8n-app

# Aguarde alguns segundos e verifique
heroku addons:info postgresql -a my-n8n-app
```

### 4. Configurar Stack para Container

```bash
heroku stack:set container -a my-n8n-app
```

### 5. Configurar Variáveis de Ambiente Obrigatórias

```bash
# Gerar chave de criptografia (NUNCA altere após o primeiro deploy!)
ENCRYPTION_KEY=$(openssl rand -hex 32)

# Configurar variáveis essenciais
heroku config:set \
  N8N_ENCRYPTION_KEY="$ENCRYPTION_KEY" \
  N8N_HOST="my-n8n-app.herokuapp.com" \
  N8N_PROTOCOL="https" \
  WEBHOOK_URL="https://my-n8n-app.herokuapp.com/" \
  DB_TYPE="postgresdb" \
  NODE_ENV="production" \
  N8N_LISTEN_ADDRESS="0.0.0.0" \
  -a my-n8n-app
```

> ⚠️ **IMPORTANTE**: Substitua `my-n8n-app` pelo nome real da sua aplicação Heroku!

### 6. Configurar Variáveis Opcionais (Recomendadas)

```bash
heroku config:set \
  N8N_DIAGNOSTICS_ENABLED="false" \
  N8N_VERSION_NOTIFICATIONS_ENABLED="false" \
  GENERIC_TIMEZONE="America/Sao_Paulo" \
  -a my-n8n-app
```

### 7. Build e Deploy

Antes de fazer o deploy, é necessário compilar o projeto:

```bash
# No diretório raiz do n8n
pnpm build:deploy

# Fazer commit dos arquivos compilados (se necessário)
git add .
git commit -m "feat: prepare for Heroku deployment"
```

Agora faça o deploy:

```bash
# Adicionar remote do Heroku (se ainda não foi adicionado)
heroku git:remote -a my-n8n-app

# Push para Heroku (trigger do build Docker)
git push heroku master
```

> **Nota**: O build pode levar de 10-20 minutos na primeira vez.

### 8. Verificar Status

```bash
# Ver logs em tempo real
heroku logs --tail -a my-n8n-app

# Verificar status do dyno
heroku ps -a my-n8n-app

# Abrir aplicação no navegador
heroku open -a my-n8n-app
```

### 9. Configurar Primeiro Usuário

1. Acesse: `https://my-n8n-app.herokuapp.com`
2. Crie sua conta de administrador
3. Configure seus workflows

## 🔧 Configurações Avançadas

### Configurar Database Manual (Opcional)

Se preferir não usar `DATABASE_URL`, configure manualmente:

```bash
# Obter credenciais do PostgreSQL
heroku pg:credentials:url postgresql -a my-n8n-app

# Configurar individualmente
heroku config:set \
  DB_POSTGRESDB_DATABASE="seu_database" \
  DB_POSTGRESDB_HOST="host.amazonaws.com" \
  DB_POSTGRESDB_PORT="5432" \
  DB_POSTGRESDB_USER="seu_user" \
  DB_POSTGRESDB_PASSWORD="sua_senha" \
  DB_POSTGRESDB_SSL_ENABLED="true" \
  DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED="false" \
  -a my-n8n-app
```

### Configurar Email (SMTP)

Para notificações e convites de usuário:

```bash
heroku config:set \
  N8N_EMAIL_MODE="smtp" \
  N8N_SMTP_HOST="smtp.gmail.com" \
  N8N_SMTP_PORT="465" \
  N8N_SMTP_USER="seu-email@gmail.com" \
  N8N_SMTP_PASS="sua-senha-app" \
  N8N_SMTP_SENDER="seu-email@gmail.com" \
  N8N_SMTP_SSL="true" \
  -a my-n8n-app
```

### Aumentar Timeout de Execução

```bash
heroku config:set \
  EXECUTIONS_TIMEOUT="600" \
  EXECUTIONS_DATA_SAVE_ON_ERROR="all" \
  EXECUTIONS_DATA_SAVE_ON_SUCCESS="all" \
  -a my-n8n-app
```

### Configurar Logging

```bash
heroku config:set \
  N8N_LOG_LEVEL="info" \
  N8N_LOG_OUTPUT="console" \
  -a my-n8n-app
```

## 🧪 Testes Locais (Antes do Deploy)

Use o `docker-compose.heroku.yml` para testar localmente:

```bash
# No diretório raiz do n8n
docker-compose -f docker-compose.heroku.yml up --build

# Acesse: http://localhost:5678
```

Para parar:

```bash
docker-compose -f docker-compose.heroku.yml down
```

## 🔄 Atualizações e Manutenção

### Atualizar n8n

```bash
# Fazer pull das últimas mudanças
git pull origin master

# Build novamente
pnpm build:deploy

# Commit e push
git add .
git commit -m "chore: update n8n version"
git push heroku master
```

### Ver Logs

```bash
# Logs em tempo real
heroku logs --tail -a my-n8n-app

# Últimas 200 linhas
heroku logs -n 200 -a my-n8n-app

# Filtrar por erro
heroku logs --tail -a my-n8n-app | grep ERROR
```

### Reiniciar Aplicação

```bash
heroku restart -a my-n8n-app
```

### Backup do Database

```bash
# Criar backup manual
heroku pg:backups:capture -a my-n8n-app

# Listar backups
heroku pg:backups -a my-n8n-app

# Download do backup
heroku pg:backups:download -a my-n8n-app
```

### Escalar Dynos

```bash
# Mudar para Basic (nunca hiberna)
heroku dyno:type web=basic -a my-n8n-app

# Aumentar para Standard
heroku dyno:type web=standard-1x -a my-n8n-app
```

## 🐛 Troubleshooting

### Erro: "Application Error" ou "H10"

**Causa**: Aplicação não conseguiu fazer bind na porta

**Solução**:
```bash
# Verificar se PORT está sendo usado corretamente
heroku logs --tail -a my-n8n-app

# Verificar entrypoint
cat docker/images/n8n/docker-entrypoint.sh
```

### Erro: Database Connection

**Causa**: Credenciais do PostgreSQL incorretas

**Solução**:
```bash
# Verificar DATABASE_URL
heroku config:get DATABASE_URL -a my-n8n-app

# Verificar DB_TYPE
heroku config:get DB_TYPE -a my-n8n-app

# Reconfigurar se necessário
heroku config:set DB_TYPE="postgresdb" -a my-n8n-app
```

### Erro: "Credentials could not be decrypted"

**Causa**: N8N_ENCRYPTION_KEY foi alterada

**Solução**:
```bash
# NUNCA mude a encryption key após criar credenciais
# Se perdeu a key, precisará recriar todas as credenciais

# Ver key atual
heroku config:get N8N_ENCRYPTION_KEY -a my-n8n-app
```

### Dyno Hibernando (Eco)

**Causa**: Eco dynos hibernam após 30 min de inatividade

**Soluções**:
1. Upgrade para Basic: `heroku dyno:type web=basic -a my-n8n-app`
2. Usar serviço de ping (ex: UptimeRobot) para manter ativo
3. Configurar workflow interno para se auto-pingar

### Build Timeout

**Causa**: Build do monorepo leva muito tempo

**Solução**:
```bash
# Build localmente e depois push
pnpm build:deploy
git add .
git commit -m "chore: pre-compiled build"
git push heroku master
```

### Limite de Memória (R14)

**Causa**: Dyno Eco tem 512MB RAM

**Solução**:
```bash
# Verificar uso de memória
heroku logs --tail -a my-n8n-app | grep "R14"

# Upgrade para Standard (1GB RAM)
heroku dyno:type web=standard-1x -a my-n8n-app
```

## 📊 Monitoramento

### Métricas Básicas

```bash
# Ver métricas do dyno
heroku ps -a my-n8n-app

# Ver métricas do database
heroku pg:info -a my-n8n-app

# Ver uso de conexões
heroku pg:ps -a my-n8n-app
```

### Configurar Alertas

No dashboard do Heroku:
1. Acesse sua aplicação
2. Vá em "Metrics" > "Alerts"
3. Configure alertas para:
   - Alta utilização de CPU
   - Uso de memória
   - Erros de aplicação
   - Downtime

## 🔒 Segurança

### SSL/TLS

Heroku fornece SSL automático:
- URLs `*.herokuapp.com` têm SSL gratuito
- Para domínio customizado, use ACM (Automated Certificate Management)

### Domínio Customizado

```bash
# Adicionar domínio
heroku domains:add www.meu-n8n.com -a my-n8n-app

# Ver configuração DNS necessária
heroku domains -a my-n8n-app

# Configurar no seu DNS provider
# Adicione um CNAME apontando para o DNS target fornecido pelo Heroku
```

### Variáveis Sensíveis

```bash
# NUNCA commite variáveis sensíveis no código
# Use sempre heroku config:set

# Ver todas as variáveis
heroku config -a my-n8n-app

# Remover variável
heroku config:unset VARIAVEL -a my-n8n-app
```

## 📚 Referências

- [n8n Documentation](https://docs.n8n.io/)
- [Heroku Container Registry](https://devcenter.heroku.com/articles/container-registry-and-runtime)
- [Heroku PostgreSQL](https://devcenter.heroku.com/articles/heroku-postgresql)
- [n8n Environment Variables](https://docs.n8n.io/hosting/environment-variables/)

## 🆘 Suporte

- **n8n Community**: https://community.n8n.io/
- **Heroku Support**: https://help.heroku.com/
- **Issues**: Abra uma issue no repositório do n8n

## 📝 Checklist de Deploy

Antes de fazer deploy em produção:

- [ ] Build do projeto completo (`pnpm build:deploy`)
- [ ] `N8N_ENCRYPTION_KEY` configurada (e salva em local seguro)
- [ ] `N8N_HOST` configurado corretamente
- [ ] `WEBHOOK_URL` configurado corretamente
- [ ] PostgreSQL addon adicionado
- [ ] `DB_TYPE=postgresdb` configurado
- [ ] Testado localmente com `docker-compose.heroku.yml`
- [ ] Backup do database configurado
- [ ] Monitoring/alertas configurados
- [ ] SSL verificado (https funcionando)
- [ ] Primeiro usuário admin criado
- [ ] Documentação de variáveis de ambiente salva

---

**Boa sorte com seu deploy! 🚀**


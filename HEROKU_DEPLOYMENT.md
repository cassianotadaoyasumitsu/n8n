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
├── Dockerfile.heroku            # Dockerfile específico para Heroku (compila tudo)
├── docker-compose.heroku.yml     # Para testes locais (simula ambiente Heroku)
├── env.heroku.example           # Template de variáveis de ambiente
└── .dockerignore                # Otimização do build
```

> **Nota**: Usamos `Dockerfile.heroku` ao invés do Dockerfile original porque o n8n usa um build system complexo com monorepo pnpm. O Dockerfile.heroku faz todo o build dentro do container.

## 🚀 Passo a Passo - Deploy Inicial

### 1. Fazer Login no Heroku

```bash
heroku login
```

### 2. Criar Aplicação no Heroku

```bash
# Criar aplicação (escolha um nome único)
heroku create u-innova-n8n --region us

# O Heroku irá mostrar a URL: https://u-innova-n8n.herokuapp.com
```

### 3. Adicionar PostgreSQL

```bash
# Adiciona PostgreSQL Mini ($5/mês)
heroku addons:create heroku-postgresql:mini -a u-innova-n8n

# Aguarde alguns segundos e verifique
heroku addons:info postgresql -a u-innova-n8n
```

### 4. Configurar Stack para Container

```bash
heroku stack:set container -a u-innova-n8n
```

### 5. Configurar Variáveis de Ambiente Obrigatórias

```bash
# Gerar chave de criptografia (NUNCA altere após o primeiro deploy!)
ENCRYPTION_KEY=$(openssl rand -hex 32)

# Configurar variáveis essenciais
heroku config:set \
  N8N_ENCRYPTION_KEY="$ENCRYPTION_KEY" \
  N8N_HOST="u-innova-n8n.herokuapp.com" \
  N8N_PROTOCOL="https" \
  WEBHOOK_URL="https://u-innova-n8n.herokuapp.com/" \
  DB_TYPE="postgresdb" \
  NODE_ENV="production" \
  N8N_LISTEN_ADDRESS="0.0.0.0" \
  -a u-innova-n8n
```

> 💾 **IMPORTANTE**: Salve o `$ENCRYPTION_KEY` em um local seguro! Você precisará dele se recriar a aplicação.

### 6. Configurar Variáveis Opcionais (Recomendadas)

```bash
heroku config:set \
  N8N_DIAGNOSTICS_ENABLED="false" \
  N8N_VERSION_NOTIFICATIONS_ENABLED="false" \
  GENERIC_TIMEZONE="America/Sao_Paulo" \
  -a u-innova-n8n
```

### 7. Deploy

**Não é necessário compilar localmente!** O `Dockerfile.heroku` faz todo o build dentro do container.

```bash
# Adicionar remote do Heroku (se ainda não foi adicionado automaticamente)
heroku git:remote -a u-innova-n8n

# Fazer commit das mudanças (se houver)
git add heroku.yml Dockerfile.heroku
git commit -m "feat: add Heroku deployment configuration"

# Push para Heroku (trigger do build Docker)
git push heroku master
```

> **Nota**: O primeiro build pode levar de **15-25 minutos** pois compila todo o monorepo pnpm dentro do container.

### 8. Verificar Status e Logs

```bash
# Ver logs em tempo real (acompanhe o build)
heroku logs --tail -a u-innova-n8n

# Verificar status do dyno
heroku ps -a u-innova-n8n

# Abrir aplicação no navegador
heroku open -a u-innova-n8n
```

### 9. Configurar Primeiro Usuário

1. Acesse: `https://u-innova-n8n.herokuapp.com`
2. Crie sua conta de administrador
3. Configure seus workflows

🎉 **Pronto! Seu n8n está rodando no Heroku!**

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

### Erro: "COPY failed: file not found... stat compiled"

**Causa**: O Dockerfile original do n8n (`docker/images/n8n/Dockerfile`) espera arquivos já compilados na pasta `./compiled`

**Solução**: ✅ **Já resolvido!** Use o `Dockerfile.heroku` que compila tudo internamente. Certifique-se de que o `heroku.yml` está apontando para `Dockerfile.heroku`:

```yaml
build:
  docker:
    web: Dockerfile.heroku
```

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


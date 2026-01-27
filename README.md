# QueryBase Web - Painel Administrativo

Painel administrativo em Laravel para gerenciar queries SQL do sistema QueryBase.

## Arquitetura

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Power BI /    │────▶│   API Golang    │────▶│     Oracle      │
│    Analistas    │     │   (porta 8080)  │     │   (Produção)    │
└─────────────────┘     └────────┬────────┘     └─────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │  Redis   │ │ Postgres │ │  Laravel │
              │ (Cache)  │ │(Metadata)│ │ (Admin)  │
              └──────────┘ └──────────┘ └──────────┘
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │  Desenvolvedores │
                                    │   cadastram     │
                                    │    queries      │
                                    └─────────────────┘
```

## Stack

- **PHP 8.3** + **Laravel 12**
- **PostgreSQL 16** (banco de metadados compartilhado com API Go)
- **Redis 7** (cache compartilhado com API Go)
- **Tailwind CSS** (via CDN)
- **Alpine.js** (interatividade)

## Estrutura do Projeto

```
querybase-web/
├── app/
│   ├── Http/Controllers/
│   │   ├── DashboardController.php   # Página inicial com métricas
│   │   ├── QueryController.php       # CRUD de queries SQL
│   │   └── DatasourceController.php  # CRUD de fontes de dados
│   └── Models/
│       ├── Datasource.php            # Configurações de conexão
│       ├── Query.php                 # Queries SQL cadastradas
│       ├── QueryParameter.php        # Parâmetros das queries
│       └── QueryExecution.php        # Logs de execução (read-only)
├── database/migrations/              # Schema compatível com API Go
├── resources/views/
│   ├── layouts/app.blade.php         # Layout principal
│   ├── components/                   # Componentes reutilizáveis
│   ├── dashboard.blade.php           # Dashboard
│   ├── queries/                      # Views de queries
│   └── datasources/                  # Views de datasources
├── routes/web.php                    # Rotas da aplicação
├── docker-compose.yml                # Docker para dev local
└── Dockerfile                        # Build da imagem
```

## Models

### Datasource
Representa uma conexão com banco de dados externo (Oracle, PostgreSQL, MySQL).

| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | UUID | Chave primária |
| slug | string | Identificador único para URLs |
| name | string | Nome de exibição |
| driver | string | oracle, postgres, mysql |
| host, port | string | Endereço do servidor |
| database_name | string | Nome do banco/service |
| username, password | string | Credenciais (senha nunca exposta) |
| max_open_conns | int | Pool de conexões ativas |
| max_idle_conns | int | Pool de conexões em espera |
| is_active | bool | Flag de ativo/inativo |

### Query
Representa uma query SQL cadastrada no sistema.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | UUID | Chave primária |
| slug | string | Usado no endpoint: /api/query/{slug} |
| name | string | Nome de exibição |
| description | text | Documentação da query |
| sql_query | text | SQL com placeholders :1, :2, etc |
| datasource_id | UUID | FK para datasource (opcional) |
| cache_ttl | int | Tempo de cache em segundos |
| timeout_seconds | int | Timeout de execução |
| is_active | bool | Flag de ativo/inativo |
| created_by, updated_by | string | Auditoria |

### QueryParameter
Define os parâmetros que uma query aceita.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | UUID | Chave primária |
| query_id | UUID | FK para query |
| name | string | Nome do parâmetro (usado na URL) |
| param_type | string | string, integer, number, date, datetime, boolean |
| is_required | bool | Obrigatoriedade |
| default_value | string | Valor padrão |
| position | int | Posição no SQL (:1, :2, :3...) |
| validations | JSONB | Regras de validação customizadas |

### QueryExecution
Log de execuções (populado pela API Go, read-only no Laravel).

| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | UUID | Chave primária |
| query_id | UUID | FK para query |
| query_slug | string | Backup do slug |
| executed_at | timestamp | Quando executou |
| duration_ms | int | Tempo de execução |
| cache_hit | bool | Se veio do cache |
| row_count | int | Linhas retornadas |
| parameters | JSONB | Parâmetros usados |
| error | text | Mensagem de erro (se houver) |
| client_ip | string | IP do cliente |
| user_agent | string | User agent |

## Rotas

### Dashboard
- `GET /` - Redireciona para /dashboard
- `GET /dashboard` - Página inicial com métricas

### Queries
- `GET /queries` - Lista com filtros e paginação
- `GET /queries/create` - Formulário de criação
- `POST /queries` - Salvar nova query
- `GET /queries/{id}` - Detalhes da query
- `GET /queries/{id}/edit` - Formulário de edição
- `PUT /queries/{id}` - Atualizar query
- `DELETE /queries/{id}` - Deletar query
- `POST /queries/{id}/duplicate` - Duplicar query
- `POST /queries/{id}/toggle` - Ativar/desativar

### Datasources
- `GET /datasources` - Lista
- `GET /datasources/create` - Formulário de criação
- `POST /datasources` - Salvar novo
- `GET /datasources/{id}` - Detalhes
- `GET /datasources/{id}/edit` - Formulário de edição
- `PUT /datasources/{id}` - Atualizar
- `DELETE /datasources/{id}` - Deletar
- `POST /datasources/{id}/toggle` - Ativar/desativar
- `POST /datasources/{id}/test-connection` - Testar conexão

## Instalação

### Desenvolvimento Local (sem Docker)

```bash
# Clone o repositório
cd querybase-web

# Instale dependências
composer install

# Configure o ambiente
cp .env.example .env
php artisan key:generate

# Configure o banco no .env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=querybase_metadata
DB_USERNAME=querybase
DB_PASSWORD=querybase123

# Execute migrations (se banco estiver vazio)
php artisan migrate

# Inicie o servidor
php artisan serve
```

### Com Docker

```bash
# Na raiz do projeto querybase-system
docker-compose up -d

# Ou apenas o projeto web
cd querybase-web
docker-compose up -d
```

Acesse:
- **Web (Laravel)**: http://localhost:8000
- **API (Golang)**: http://localhost:8080

## Fluxo de Uso

1. **Desenvolvedor** acessa o painel web (localhost:8000)
2. **Cadastra um Datasource** (configuração do Oracle)
3. **Cadastra uma Query** com SQL e parâmetros
4. **Power BI** consome o endpoint gerado (localhost:8080/api/query/slug)
5. **API Go** executa a query no Oracle com cache no Redis
6. **Dashboard** exibe métricas de execução

## Componentes Blade

### Layout Principal
`resources/views/layouts/app.blade.php`
- Sidebar com navegação
- Flash messages
- Slots para título, subtitle e actions

### Componentes Reutilizáveis
- `<x-card>` - Card com título opcional
- `<x-stat-card>` - Card de estatística
- `<x-badge>` - Badge colorido
- `<x-form.input>` - Input com label e validação
- `<x-form.select>` - Select com opções
- `<x-form.textarea>` - Textarea
- `<x-form.checkbox>` - Checkbox

## Banco de Dados

O Laravel compartilha o banco PostgreSQL com a API Golang. As tabelas são:

- `datasources` - Configurações de conexão
- `queries` - Queries SQL cadastradas
- `query_parameters` - Parâmetros das queries
- `query_executions` - Log de execuções

A view `vw_queries_with_params` agrega queries com parâmetros em JSON para a API Go.

## Decisões Técnicas

### Por que compartilhar o banco com a API Go?
- Simplicidade: ambos sistemas manipulam os mesmos dados
- Consistência: não há sincronização entre bancos
- Performance: sem overhead de comunicação entre sistemas

### Por que Tailwind CSS via CDN?
- MVP rápido: não precisa de build pipeline (npm, Vite)
- Simplicidade: fácil de entender e modificar
- Produção: pode migrar para build otimizado depois

### Por que Alpine.js?
- Leve: 15kb vs 30kb+ de Vue/React
- Integrado: funciona direto no Blade
- Simples: curva de aprendizado mínima

### Por que não usar Livewire?
- Complexidade adicional
- Para este MVP, formulários tradicionais são suficientes
- Pode ser adicionado depois se necessário

## Próximos Passos

- [ ] Autenticação de usuários
- [ ] Permissões por role (admin, dev, viewer)
- [ ] Histórico de alterações (audit log)
- [ ] Teste de queries no painel
- [ ] Export de queries para JSON/YAML
- [ ] Documentação automática de API (Swagger)
- [ ] Gráficos de performance (Chart.js)
# How to run EnerSight release 1

## Pré-requisitos

- Docker 24+
- Docker Compose v2
- Java 21
- Maven 3.9+
- Python 3.12
- GDAL / ogr2ogr instalado
- Projeto clonado localmente

---

## Passo 0 - Limpeza (opcional)

> Atenção: os comandos abaixo removem todos os containers e volumes do host local.

### Parar containers
```bash
docker stop $(docker ps -aq)
```

### Remover containers
```bash
docker rm $(docker ps -aq)
```

### Remover volumes
```bash
docker volume rm $(docker volume ls -q)
```

### Quick clean
```bash
docker system prune -a --volumes -f
```

---

## Passo 1 - Subir banco

Estando na raiz do projeto.

```bash
docker compose -f backend/docker/docker-compose.yaml up -d enersight-db --build --remove-orphans
```

### Validar banco
```bash
docker ps
docker logs enersight-db
```

---

## Passo 2 - Subir API

Estando em `EnerSight/backend/enersight-api`

### Limpar build
```bash
mvn clean
```

### Rodar API
```bash
mvn spring-boot:run
```

---

## Passo 3 - Worker

Estando em `EnerSight/backend/enersight-worker`

### Criar venv
```bash
python -m venv .venv
```

### Ativar venv
```bash
source .venv/bin/activate
```

### Instalar dependências
```bash
pip install -r requirements.txt
```

---

## Passo 4 - Importar GDB

Ajustar caminho físico do arquivo `.gdb` no módulo `loader.py`.

Executar:

```bash
python -m app.ingestion.loader
```

### Validar carga
```sql
SELECT COUNT(*) FROM enel_sp_ssdmt;
```

---

## Passo 5 - Importar indicadores

Ajustar caminhos CSV em `import_man_ind.py`.

Executar:

```bash
python -m app.ingestion.import_man_ind
```

### Validar carga
```sql
SELECT COUNT(*) FROM indicador_qualidade;
```

---

## Passo 6 - Testar API

### Com ano

```bash
curl -G "http://localhost:8080/api/geo" \
  --data-urlencode "year=2025" \
  --data-urlencode "minx=-46.7" \
  --data-urlencode "miny=-23.6" \
  --data-urlencode "maxx=-46.5" \
  --data-urlencode "maxy=-23.4"
```

### Sem ano

```bash
curl -G "http://localhost:8080/api/geo" \
  --data-urlencode "minx=-46.7" \
  --data-urlencode "miny=-23.6" \
  --data-urlencode "maxx=-46.5" \
  --data-urlencode "maxy=-23.4"
```

---

## Resultado esperado

- API em `http://localhost:8080`
- endpoint `/api/geo` respondendo
- banco populado
- geometrias carregadas
- indicadores disponíveis

---

## Troubleshooting

### Logs banco
```bash
docker logs enersight-db
```

### Logs API
Consultar console do Spring Boot

### Reset completo
Reexecutar Passo 0

# Roadmap de implementação para o Injestor de dados GDB - ANEEL
## Fase 0 — Base
* Subir PostgreSQL + PostGIS (Docker)
* Criar schemas: staging, core
* Habilitar extensão PostGIS

## Fase 1 — Container do ingestor
* Criar Dockerfile (Python + GDAL)
* Implementar comando ogr2ogr → staging
* Montar volume com arquivos GDB

## Fase 2 — Paralelismo
* Implementar processamento paralelo (4–8 workers)
* Processar múltiplos GDBs simultaneamente
* Adicionar retry básico

## Fase 3 — Controle incremental
* Criar tabela ingestion_control
* Calcular hash dos arquivos
* Ignorar arquivos não modificados

## Fase 4 — Pipeline SQL (staging → core)
* Validar geometrias (ST_IsValid)
* Padronizar SRID (ST_Transform)
* Mapear colunas
* Implementar UPSERT

## Fase 5 — Performance
* Usar PG_USE_COPY (bulk insert)
* Criar índices espaciais (GiST) após ingestão
* Ajustar número de workers conforme hardware

## Fase 6 — Orquestração
* Automatizar execução (cron ou pipeline)
> Fluxo: detectar → ingerir → transformar → logar

## Fase 7 — Observabilidade
* Criar ingestion_log
* Log de erros e tempo por arquivo
* Monitorar falhas e throughput

## Arquitetura final
* Container ingestor (Python + GDAL)
* Container banco (PostgreSQL + PostGIS)
* Pipeline: GDB → staging → transformação SQL → core → índices
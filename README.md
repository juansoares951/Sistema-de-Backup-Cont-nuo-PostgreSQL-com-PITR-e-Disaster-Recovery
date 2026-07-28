# Sistema-de-Backup-Cont-nuo-PostgreSQL-com-PITR-e-Disaster-Recovery
Implementação de uma solução completa de backup contínuo para PostgreSQL utilizando WAL-G, Backblaze B2, Zabbix e Grafana. O projeto contempla backups FULL automatizados, arquivamento contínuo de WAL, recuperação Point-in-Time Recovery (PITR), políticas de retenção, monitoramento preventivo e alertas



# Instalação rápida

## Requisitos

- Ubuntu 22.04/24.04
- PostgreSQL 17
- WAL-G
- Conta Backblaze B2
- Zabbix Agent 2
- Grafana

## Fluxo

1. Instalar PostgreSQL
2. Instalar WAL-G
3. Configurar Backblaze B2
4. Ativar archive_command
5. Criar scripts
6. Configurar cron
7. Configurar monitoramento
8. Executar teste de restore

========================================================
1. VISÃO GERAL DA ARQUITETURA
========================================================

Este sistema implementa uma solução completa de backup contínuo para PostgreSQL 17 com suporte a Point-in-Time Recovery (PITR), utilizando WAL-G como engine de backup e Backblaze B2 como storage externo.


flowchart TD

A[PostgreSQL 17 VPS]
A --> B[pg_wal]
B --> C[archive_command]
C --> D[WAL-G]
D --> E[Backblaze B2]

E --> F[Backup FULL]
E --> G[WAL Archive]

F --> H[PITR Restore]
G --> H






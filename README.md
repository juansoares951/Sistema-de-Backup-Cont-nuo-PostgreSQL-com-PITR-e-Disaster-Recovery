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



Arquitetura:

PostgreSQL 17 (VPS)
        |
        | WAL (Write-Ahead Logs)
        v
WAL-G (backup engine / archive_command)
        |
        | protocolo S3 ( Blackbaze B2)
        v
Backblaze B2 (bucket: backupExsis)

========================
2. OBJETIVOS DO SISTEMA
========================

- Garantir backup contínuo do banco de dados
- Permitir recuperação total (disaster recovery)
- Permitir recuperação em qualquer ponto no tempo (PITR)
- Armazenamento externo seguro (offsite)
- Evitar perda de dados em falhas críticas
- Automação completa sem intervenção manual
- Controle de retenção de backups

========================
3. ESTRUTURA DO POSTGRESQL
========================

Diretório principal:

/var/lib/postgresql/17/main

Contém:
- dados do banco (tablespaces)
- índices
- catálogo interno
- configurações locais
- diretório WAL (pg_wal)

Acesso:

sudo -u postgres ls /var/lib/postgresql/17/main

Baixar WAL-G 

![Download WAL-G](5_baixarWalG.png)







# Sistema de Backup Continuo PostgreSQL com PITR e Disaster-Recovery
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

Este sistema implementa uma solução completa de backup contínuo para PostgreSQL 17 com suporte a Point-in-Time Recovery (PITR), utilizando WAL-G como engine de backup e Backblaze B2 como storage externo.



Arquitetura:

PostgreSQL 17 (VPS)
        |
        | WAL (Write-Ahead Logs)
        v
WAL-G (backup engine / archive_command)
        |
        | protocolo S3  
        v
Backblaze B2 (bucket: backupExsis)


2. OBJETIVOS DO SISTEMA

- Garantir backup contínuo do banco de dados
- Permitir recuperação total (disaster recovery)
- Permitir recuperação em qualquer ponto no tempo (PITR)
- Armazenamento externo seguro (offsite)
- Evitar perda de dados em falhas críticas
- Automação completa sem intervenção manual
- Controle de retenção de backups


3. ESTRUTURA DO POSTGRESQL


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

## Baixar WAL-G
Processo de instalação do WAL-G utilizado no ambiente PostgreSQL 17.
<img width="805" height="54" alt="5_baixarWalG" src="https://github.com/user-attachments/assets/ebc3e484-6b53-4154-8d8b-75a4560b1ea8" />



4. CONFIGURAÇÃO DO POSTGRESQL


Arquivo:

sudo nano /etc/postgresql/17/main/postgresql.conf

Parâmetros críticos:

wal_level = replica
archive_mode = on
archive_timeout = 60s

archive_command = 'bash -c "set -a; source /etc/wal-g/env; set +a; wal-g wal-push %p"'

<img width="957" height="170" alt="15_adicionarCONF" src="https://github.com/user-attachments/assets/2c9d1e92-367c-49f0-8003-8b3812187b24" />



Função:

- habilita geração de WAL necessário para PITR
- ativa arquivamento automático
- envia WAL diretamente ao WAL-G



5. CONFIGURAÇÃO DO WAL-G


Arquivo:

sudo nano /etc/wal-g/env

Contém:

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_ENDPOINT
- WALG_S3_PREFIX
- WALG_COMPRESSION_METHOD

<img width="821" height="509" alt="image" src="https://github.com/user-attachments/assets/1c8498cc-61d7-4910-b3cc-859ffbea7c15" />


Verificação:

sudo cat /etc/wal-g/env


Função:

- autenticação com Backblaze B2
- definição do bucket de armazenamento
- parâmetros de compressão
- endpoint S3 compatível

  
6. BACKBLAZE B2 STORAGE


Bucket utilizado:

backupExsis

Contém:
- backups FULL (base_)
- WAL contínuo
- histórico de versões
- dados de PITR

<img width="1319" height="640" alt="image" src="https://github.com/user-attachments/assets/8963222e-b8be-4e0e-872f-6cb4dd603115" />


Comando de verificação dos backups FULL no storage:

sudo -u postgres bash -c 'set -a; source /etc/wal-g/env; set +a; wal-g backup-list'


Onde os backups FULL ficam armazenados:

- Os backups FULL NÃO ficam no servidor local
- Eles são armazenados exclusivamente no Backblaze B2
- Cada backup aparece como:

  base_XXXXXXXXXXXXXXXXXXXX

- Esses backups são usados como ponto inicial para restauração (PITR)

7. TIPOS DE BACKUP


7.1 BACKUP FULL (BASE)

Função:
- snapshot completo do banco
- base inicial para recuperação

Execução:

sudo /usr/local/bin/wal-g-full-backup.sh

<img width="1129" height="720" alt="image" src="https://github.com/user-attachments/assets/1a48bf8e-4886-4f2e-9f3f-e4debdb96fea" />

Agendamento:
03:00 diariamente


7.2 WAL (CONTÍNUO)

Função:
- captura todas as transações do banco
- permite recuperação ponto a ponto

Execução:
automática via archive_command

<img width="816" height="324" alt="logs em tempo real" src="https://github.com/user-attachments/assets/a1627263-1da4-4a06-b9e3-71bf924552ec" />



8. POLÍTICA DE RETENÇÃO


Configuração atual:

- manter apenas 4 backups FULL mais recentes
- remover backups antigos automaticamente
- preservar WALs necessários para restauração

Comando:

wal-g delete retain FULL 4 --confirm


Comportamento:

- novos backups FULL substituem os antigos
- WAL-G garante consistência entre FULL + WAL
- impede crescimento infinito do storage

- 

9. SCRIPTS IMPLEMENTADOS


9.1 BACKUP FULL AUTOMÁTICO

Arquivo:

sudo nano /usr/local/bin/wal-g-full-backup.sh

Função:
- executa backup completo do PostgreSQL
- envia para Backblaze B2
- gera logs de execução
  
<img width="849" height="550" alt="29_scriptBackupFullAutomatico" src="https://github.com/user-attachments/assets/d1a5c92d-ce13-4e51-b121-bbbe4195b1fe" />


9.2 LIMPEZA AUTOMÁTICA

Arquivo:

sudo nano /usr/local/bin/wal-g-cleanup.sh

Função:
- remove backups FULL antigos
- mantém apenas 4 backups
- executa retenção segura
<img width="851" height="550" alt="25_scriptLimpezaAtutomaticaBackupsAntigos" src="https://github.com/user-attachments/assets/62bcfce0-c9ab-4677-861a-334d8f21b7d6" />


9.3 MONITORAMENTO DO ARCHIVE

Arquivo:

sudo nano /usr/local/bin/wal-archive-monitor.sh

Função:
- verifica falhas no archive_command
- monitora pg_stat_archiver
- detecta atraso de WAL
- gera logs
- envia alertas por e-mail

  <img width="845" height="541" alt="32_scriptMonitoramentoLimitePgWal" src="https://github.com/user-attachments/assets/1c157327-aae1-49bf-a5ff-262012ed8afb" />


9.4 MONITORAMENTO DE CRESCIMENTO DO pg_wal (NOVO)


Arquivo:

sudo nano /usr/local/bin/wal-wal-growth-monitor.sh


Função:
- monitora tamanho absoluto do diretório pg_wal
- detecta crescimento anormal entre execuções
- evita falhas silenciosas no archive_command
- protege contra risco de disco cheio

<img width="825" height="506" alt="image" src="https://github.com/user-attachments/assets/cc628513-2615-4d1f-8281-51ab71f4d95e" />


Regras implementadas:

 Limite absoluto de crescimento:
- alerta por email se pg_wal ultrapassar 500MB

 Detecção de crescimento rápido:
- compara tamanho atual vs execução anterior
- detecta crescimento anormal em curto período

 Estado persistente:
- salva histórico em /var/tmp/wal_size.state


Execução automática (cron):

*/5 * * * * /usr/local/bin/wal-wal-growth-monitor.sh


Logs:

/var/log/wal-growth-monitor.log


Impacto:

- evita saturação de disco
- detecta falha de arquivamento do WAL-G
- previne indisponibilidade do PostgreSQL

9.5 HEALTH CHECK AUTOMÁTICO DO BACKUP (NOVO)

Arquivo:
sudo nano /usr/local/bin/wal-backup-healthcheck.sh

<img width="819" height="504" alt="image" src="https://github.com/user-attachments/assets/866d206a-418e-43b7-bb7b-b5015788248c" />


Função:
- valida existência de backups FULL no B2
- verifica conectividade com WAL-G (Backblaze B2)
- checa integridade da cadeia WAL
- valida status do PostgreSQL archiver
- monitora uso de disco do sistema
- monitora crescimento do diretório pg_wal
- gera relatório automático em log local

Execução manual:
sudo /usr/local/bin/wal-backup-healthcheck.sh

Agendamento recomendado (cron):
0 * * * * /usr/local/bin/wal-backup-healthcheck.sh

Logs gerados:
/var/log/wal-backup-healthcheck.log

Status validado:

O healthcheck valida automaticamente:

- backups FULL existentes no Backblaze B2
- conexão com WAL-G e storage remoto
- status do WAL archiver (PostgreSQL)
- uso de disco do servidor
- tamanho do pg_wal (proteção contra crescimento anormal)

Observação operacional:

Este healthcheck é read-only (não invasivo), ou seja:

- não altera banco de dados
- não interrompe PostgreSQL
- não interfere no WAL-G
- apenas coleta métricas e valida estados



10. CRON JOBS (AUTOMAÇÃO)


10.1 LIMPEZA DE BACKUP

Executa diariamente:

02:00

0 2 * * * /usr/local/bin/wal-g-cleanup.sh >> /var/log/wal-g-cleanup.log 2>&1

<img width="851" height="550" alt="25_scriptLimpezaAtutomaticaBackupsAntigos" src="https://github.com/user-attachments/assets/5bccf5b2-a968-4a4d-a12d-f05fbc342d77" />



10.2 BACKUP FULL

Executa diariamente:

03:00

0 3 * * * /usr/local/bin/wal-g-full-backup.sh >> /var/log/wal-g-full-backup.log 2>&1

<img width="849" height="550" alt="29_scriptBackupFullAutomatico" src="https://github.com/user-attachments/assets/4120038a-8ef9-4eab-a8cc-f872797f929a" />


10.3 MONITORAMENTO

Executa a cada 5 minutos:

*/5 * * * * /usr/local/bin/wal-archive-monitor.sh



11. MONITORAMENTO DO SISTEMA


Ver status do archiver:

sudo -u postgres psql -c "SELECT * FROM pg_stat_archiver;"


Ver cadeia WAL:

sudo -u postgres bash -c 'set -a; source /etc/wal-g/env; set +a; wal-g wal-show'


Ver backups:

sudo -u postgres bash -c 'set -a; source /etc/wal-g/env; set +a; wal-g backup-list'


Ver falhas:

sudo -u postgres psql -t -c "SELECT failed_count FROM pg_stat_archiver;"




12. LOGS DO SISTEMA


Backup FULL:

/var/log/wal-g-full-backup.log

Limpeza:

/var/log/wal-g-cleanup.log

Monitoramento:

/var/log/wal-archive-monitor.log


13. FLUXO COMPLETO DO SISTEMA


1. PostgreSQL gera transações
        |
2. WAL é criado em pg_wal
        |
3. archive_command envia WAL para WAL-G
        |
4. WAL-G envia para Backblaze B2
        |
5. Backup FULL é gerado diariamente às 03:00 e depende do fluxo contínuo de WAL para consistência e PITR
        |
6. Limpeza remove backups antigos às 02:00
        |
7. PITR combina FULL + WAL para recuperação total


14. SEGURANÇA


- armazenamento externo (offsite)
- credenciais isoladas em /etc/wal-g/env
- acesso via S3 compatível
- dados versionados no B2
- WAL contínuo para Point-in-Time Recovery (PITR)
- retenção controlada de backups FULL
- proteção contra perda total do servidor


15. CENÁRIOS OPERACIONAIS E MITIGAÇÃO


- Falha no WAL-G
  → Mitigado por monitoramento ativo + alertas por e-mail + logs automáticos

- Crescimento do pg_wal
  → Mitigado por archive_command ativo + verificação contínua via monitoramento

- Backup FULL indisponível
  → Mitigado por execução automática diária às 03:00

- Falha no archive_command
  → Detectado automaticamente pelo script de monitoramento e alertado em tempo real

- Perda de storage ou corrupção parcial
  → Mitigado por retenção de múltiplos backups FULL (4 versões) + WAL contínuo

16. MONITORAMENTO COM ZABBIX + GRAFANA


Objetivo:

Monitorar continuamente a saúde do arquivamento WAL do PostgreSQL, permitindo visualizar:

- quantidade de WALs arquivados
- falhas de arquivamento
- tempo desde o último WAL enviado
- utilização de CPU
- utilização de memória
- utilização de disco

<img width="1374" height="441" alt="37_adicionadoZabbix" src="https://github.com/user-attachments/assets/06e3bac7-146d-4294-9f72-cdc080a75e68" />



16.1 SCRIPT DE COLETA


Arquivo:

/etc/zabbix/scripts/postgres_wal_metrics.sh

Função:

Coleta informações da view pg_stat_archiver e retorna uma única linha no formato:

archived_count|failed_count|delay_seconds

Exemplo:

966|0|42

Onde:

Campo 1
Quantidade total de WALs arquivados com sucesso.

Campo 2
Quantidade total de falhas de arquivamento.

Campo 3
Tempo (em segundos) desde o último WAL arquivado.

Execução manual:

sudo bash /etc/zabbix/scripts/postgres_wal_metrics.sh



16.2 CONFIGURAÇÃO DO ZABBIX AGENT



Arquivo:

/etc/zabbix/zabbix_agent2.conf

UserParameter configurado:

UserParameter=postgres.wal.metrics,/etc/zabbix/scripts/postgres_wal_metrics.sh

Após alterações:

sudo systemctl restart zabbix-agent2


16.3 ITEM MESTRE (MASTER ITEM)


Host:

PostgreSQL-WAL-G-VPS

Nome:

PostgreSQL WAL Metrics

Tipo:

Agente Zabbix

Chave:

postgres.wal.metrics

Tipo de informação:

Texto

Intervalo de coleta:

30 segundos

Este item retorna uma única string contendo todas as métricas do WAL.


16.4 ITENS DEPENDENTES


A partir do item mestre "PostgreSQL WAL Metrics", o Zabbix cria três itens dependentes responsáveis por separar as métricas retornadas pelo script.

Métrica 1

Nome:
WAL Archived

Chave:
postgres.wal.archived

Origem:
Campo 1 da saída

Exemplo:
966

Descrição:
Quantidade total de segmentos WAL arquivados com sucesso.




Métrica 2

Nome:
WAL Failed

Chave:
postgres.wal.failed

Origem:
Campo 2 da saída

Exemplo:
0

Descrição:
Quantidade total de falhas de arquivamento registradas pelo PostgreSQL.




Métrica 3

Nome:
WAL Delay

Chave:
postgres.wal.delay

Origem:
Campo 3 da saída

Exemplo:
42

Descrição:
Tempo, em segundos, desde o último WAL arquivado.

<img width="900" height="472" alt="41_LogColetaZabix" src="https://github.com/user-attachments/assets/171dbe1e-103b-4148-9292-659e5aa9f00d" />



16.5 GRAFANA


O Grafana foi integrado ao Zabbix utilizando o plugin oficial do Zabbix.

No dashboard foram adicionados os painéis de infraestrutura:

- CPU
- Memória
- Disco

Além disso, foram disponibilizadas métricas específicas do PostgreSQL:

- WAL Archived
- WAL Failed
- WAL Delay

Essas métricas são obtidas diretamente dos itens dependentes do Zabbix.

<img width="1437" height="466" alt="40_criadoDashboardGrafana" src="https://github.com/user-attachments/assets/8a8251b0-ad94-43e5-b24f-e15b16271e56" />



16.6 BENEFÍCIOS DO MONITORAMENTO


O monitoramento permite identificar rapidamente:

- interrupção do envio de WALs
- aumento do tempo entre arquivamentos
- falhas no archive_command
- indisponibilidade do WAL-G
- problemas de comunicação com o Backblaze B2
- degradação do sistema operacional (CPU, RAM e Disco)


17.PROCESSO DE RESTAURAÇÃO E VALIDAÇÃO DE BACKUP FULL + PREPARAÇÃO PARA PITR


Esta seção documenta o teste real de recuperação realizado utilizando
um backup armazenado no Backblaze B2 através do WAL-G.

O objetivo do teste foi validar:

- acesso ao storage remoto Backblaze B2
- disponibilidade dos backups FULL
- integridade do backup armazenado
- download e extração através do WAL-G
- recuperação do cluster PostgreSQL 17
- funcionamento do processo PITR utilizando WAL



17.1 VALIDAÇÃO DOS BACKUPS DISPONÍVEIS NO BACKBLAZE B2


Antes do processo de restauração foi realizada a consulta dos backups
disponíveis no storage remoto.


Comando executado:


sudo -u postgres bash -c '
set -a
source /etc/wal-g/env
set +a
wal-g backup-list
'


Resultado obtido:


backup_name                          modified

base_000000010000000E0000002F        2026-07-02T13:42:57Z

base_000000010000000E000000AF        2026-07-02T17:31:27Z


Validação realizada:


Foi confirmado que existem backups FULL disponíveis no Backblaze B2.

Os backups possuem o padrão:


base_XXXXXXXXXXXXXXXXXXXX


Estes backups representam os pontos base utilizados para recuperação
completa do PostgreSQL.



17.2 VALIDAÇÃO DETALHADA DO BACKUP ESCOLHIDO


Foi selecionado o backup:


base_000000010000000E000000AF


Comando executado:


sudo -u postgres bash -c '
set -a
source /etc/wal-g/env
set +a
wal-g backup-list --detail
'


Resultado obtido:


backup_name:

base_000000010000000E000000AF


Start time:

2026-07-02T17:28:48Z


Finish time:

2026-07-02T17:31:24Z


PostgreSQL version:

170009


Data directory original:

/var/lib/postgresql/17/main


Start LSN:

E/AF000028


Finish LSN:

E/B0000050



Validação realizada:


O backup possui informações completas de controle:

- horário de criação
- versão PostgreSQL compatível
- diretório original
- posição inicial do WAL
- posição final do WAL


O backup foi considerado apto para restauração.



17.3 VALIDAÇÃO DA CADEIA WAL


Antes da restauração foi validada a existência dos WALs necessários
para recuperação.


Comando executado:


sudo -u postgres bash -c '
set -a
source /etc/wal-g/env
set +a
wal-g wal-show
'


Resultado obtido:


Timeline:

1


Status:

OK


Backups count:

2


Segmentos WAL disponíveis:

4601


Validação realizada:


A cadeia WAL estava disponível no storage remoto permitindo
recuperação Point-In-Time Recovery (PITR).



17.4 RESTAURAÇÃO DO BACKUP FULL


O cluster PostgreSQL foi preparado para receber o backup restaurado.


Diretório utilizado:


/var/lib/postgresql/17/main



Comando executado:


sudo -u postgres bash -c '
set -a
source /etc/wal-g/env
set +a
wal-g backup-fetch \
/var/lib/postgresql/17/main \
base_000000010000000E000000AF
'


Resultado obtido:


INFO: Finished extraction of part_001.tar.lz4

INFO: Finished extraction of part_002.tar.lz4

INFO: Finished extraction of part_003.tar.lz4

INFO: Finished extraction of part_004.tar.lz4

INFO: Finished extraction of part_005.tar.lz4

INFO: Finished extraction of part_006.tar.lz4

INFO: Finished extraction of part_007.tar.lz4

INFO: Finished extraction of part_008.tar.lz4

INFO: Finished extraction of part_009.tar.lz4

INFO: Finished extraction of part_010.tar.lz4

INFO: Finished extraction of pg_control.tar.lz4

INFO: Finished extraction of backup_label.tar.lz4


Backup extraction complete.


Validação realizada:


A extração foi concluída com sucesso.

O diretório PostgreSQL foi restaurado contendo:


- base/
- global/
- pg_wal/
- pg_xact/
- pg_multixact/
- backup_label
- pg_control



17.5 VALIDAÇÃO DOS ARQUIVOS RESTAURADOS



Comando executado:


sudo ls -la /var/lib/postgresql/17/main



Resultado obtido:


drwx------ postgres postgres base

drwx------ postgres postgres global

drwx------ postgres postgres pg_wal

-rw------- postgres postgres backup_label

-rw------- postgres postgres PG_VERSION


Validação realizada:


A estrutura física do cluster PostgreSQL foi restaurada corretamente.



17.6 CONFIGURAÇÃO DO MODO RECOVERY



Foi criado o arquivo:


/var/lib/postgresql/17/main/recovery.signal



Comandos executados:


sudo touch /var/lib/postgresql/17/main/recovery.signal


sudo chown postgres:postgres \
/var/lib/postgresql/17/main/recovery.signal


sudo chmod 600 \
/var/lib/postgresql/17/main/recovery.signal



Validação realizada:


O PostgreSQL foi preparado para iniciar em modo recovery,
permitindo buscar WALs adicionais através do WAL-G.



17.7 RESTAURAÇÃO DOS WALs DURANTE O RECOVERY



Durante o processo de inicialização o PostgreSQL utiliza o WAL-G
para localizar e aplicar os segmentos WAL armazenados no Backblaze B2.


Fluxo validado:


Backup FULL restaurado
          |
PostgreSQL inicia recovery
          |
Solicita WAL necessário
          |
WAL-G consulta Backblaze B2
          |
Segmentos WAL aplicados
          |
Banco retorna ao último ponto consistente



17.8 COMANDOS UTILIZADOS PARA RECUPERAÇÃO EM DESASTRE



Parar PostgreSQL:


sudo systemctl stop postgresql



Limpar diretório:


sudo rm -rf /var/lib/postgresql/17/main/*



Restaurar backup:


sudo -u postgres bash -c '
set -a
source /etc/wal-g/env
set +a
wal-g backup-fetch \
/var/lib/postgresql/17/main \
BACKUP_NAME
'


Criar recovery:


sudo touch /var/lib/postgresql/17/main/recovery.signal



Corrigir permissões:


sudo chown -R postgres:postgres \
/var/lib/postgresql/17/main



Iniciar cluster:


sudo pg_ctlcluster 17 main start




17.9 VALIDAÇÃO FINAL DO RESTORE



Comandos de validação:


Ver cluster:


pg_lsclusters



Ver serviço:


sudo systemctl status postgresql



Acessar banco:


sudo -u postgres psql



Ver último WAL aplicado:


SELECT pg_last_wal_replay_lsn();



Validação realizada:


O processo de recuperação deve confirmar:

- cluster PostgreSQL iniciado
- diretório restaurado utilizado
- WAL aplicado corretamente
- bancos disponíveis para acesso



17.10 TESTE REAL EXECUTADO


Backup utilizado:

base_000000010000000E000000AF


Resultado obtido:

 Backup localizado no Backblaze B2

 Backup consultado pelo WAL-G

 Download realizado com sucesso

 Arquivos extraídos corretamente

 backup_label restaurado

 pg_control restaurado

 Estrutura PostgreSQL recuperada

 <img width="1196" height="755" alt="image" src="https://github.com/user-attachments/assets/9422dbec-2536-49a9-9432-34914c87cbf5" />





18. RESUMO FINAL


Sistema completo de backup empresarial:

O ambiente encontra-se totalmente automatizado.

Atualmente o sistema possui:

 PostgreSQL 17
 WAL-G
 Backblaze B2
 Backup FULL diário
 WAL contínuo
 Retenção automática
 PITR
 Monitoramento Zabbix
 Dashboards Grafana
 Métricas de CPU
 Métricas de Memória
 Métricas de Disco
 Métricas de WAL
 Logs locais
 Health Check
 Monitoramento do crescimento do pg_wal
 Restore e backup validados em maquina de servidor local


O ambiente está preparado para operação contínua, recuperação de desastres (Disaster Recovery) e restauração Point-in-Time Recovery (PITR).











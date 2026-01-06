# Documentação Completa do SQL Server
*Algumas tarefas que compôe o dia a dia de um DBA*

---

## Índice

1. [Gerenciamento de Acessos](#gerenciamento-de-acessos)
2. [Backup e Restore](#backup-e-restore)
3. [SQL Server Agent](#sql-server-agent)
4. [Tuning e Otimização](#tuning-e-otimização)
5. [Estatísticas no SQL Server](#estatísticas-no-sql-server)
6. [Seek vs Scan](#seek-vs-scan)
7. [Query Store](#query-store)
8. [Sistema de Alertas](#sistema-de-alertas)

---

## Gerenciamento de Acessos

### O que são Acessos no SQL Server?

O SQL Server utiliza um sistema de segurança em camadas que controla o acesso aos dados através de:
- **Logins**: Autenticação no nível do servidor
- **Usuários**: Autenticação no nível do banco de dados
- **Roles**: Grupos de permissões predefinidas
- **Permissões**: Direitos específicos sobre objetos

### Tipos de Autenticação

#### 1. Autenticação Windows (Recomendada)
```sql
-- Criar login usando autenticação Windows
CREATE LOGIN [DOMAIN\Usuario] FROM WINDOWS;

-- Criar usuário no banco de dados
USE MinhaBaseDeDados;
CREATE USER [DOMAIN\Usuario] FOR LOGIN [DOMAIN\Usuario];
```

#### 2. Autenticação SQL Server
```sql
-- Criar login SQL Server
CREATE LOGIN MeuUsuario WITH PASSWORD = 'SenhaExemplo';

-- Criar usuário no banco
USE MinhaBaseDeDados;
CREATE USER MeuUsuario FOR LOGIN MeuUsuario;
```

### Roles Principais

#### Server Roles (Nível Servidor)
- **sysadmin**: Acesso total ao servidor - ideal apenas para DBA
- **serveradmin**: Configuração do servidor
- **securityadmin**: Gerenciamento de logins
- **processadmin**: Gerenciamento de processos
- **dbcreator**: Criação e alteração de bancos
- **diskadmin**: Gerenciamento de arquivos de disco
- **bulkadmin**: Operações BULK INSERT

#### Database Roles (Nível Banco)
- **db_owner**: Proprietário do banco
- **db_datareader**: Leitura de todas as tabelas
- **db_datawriter**: Escrita em todas as tabelas
- **db_ddladmin**: Comandos DDL (CREATE, ALTER, DROP)
- **db_securityadmin**: Gerenciamento de roles e membros

### Exemplos Práticos

```sql
-- Conceder acesso de leitura
ALTER ROLE db_datareader ADD MEMBER MeuUsuario;

-- Conceder permissões específicas
GRANT SELECT, INSERT ON MinhaTabela TO MeuUsuario;

-- Criar role customizada
CREATE ROLE RelatoriosRole;
GRANT SELECT ON VIEW_Vendas TO RelatoriosRole;
ALTER ROLE RelatoriosRole ADD MEMBER MeuUsuario;

-- Verificar permissões
SELECT 
    p.principal_id,
    p.name AS principal_name,
    p.type_desc,
    pe.permission_name,
    pe.state_desc,
    s.name AS securable_name
FROM sys.database_principals p
INNER JOIN sys.database_permissions pe ON p.principal_id = pe.grantee_principal_id
INNER JOIN sys.objects s ON pe.major_id = s.object_id;
```

---

## Backup e Restore

### Importância do Backup

O backup é fundamental para:
- **Proteção contra perda de dados**
- **Recuperação de desastres**
- **Compliance e auditoria**
- **Migração de dados**
- **Desenvolvimento e testes**

### Tipos de Backup

#### 1. Full Backup (Completo)
```sql
-- Backup completo
BACKUP DATABASE MinhaBaseDeDados 
TO DISK = 'C:\Backup\MinhaBase_Full.bak'
WITH COMPRESSION, CHECKSUM, INIT;
```

#### 2. Differential Backup (Diferencial)
```sql
-- Backup diferencial
BACKUP DATABASE MinhaBaseDeDados 
TO DISK = 'C:\Backup\MinhaBase_Diff.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM, INIT;
```

#### 3. Transaction Log Backup
```sql
-- Backup do log de transação
BACKUP LOG MinhaBaseDeDados 
TO DISK = 'C:\Backup\MinhaBase_Log.trn'
WITH COMPRESSION, CHECKSUM, INIT;
```

### Estratégias de Backup

#### Modelo Full Recovery
```sql
-- Configurar modelo de recuperação
ALTER DATABASE MinhaBaseDeDados SET RECOVERY FULL;

-- Estratégia recomendada:
-- Full backup: Semanal
-- Differential backup: Diário
-- Log backup: A cada 15 minutos
```

#### Plano de Manutenção Exemplo
```sql
-- Criar job para backup automático
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'Backup Diário Full',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Backup Diário Full',
    @step_name = N'Backup Database',
    @command = N'BACKUP DATABASE MinhaBaseDeDados 
                 TO DISK = ''C:\Backup\MinhaBase_Full.bak''
                 WITH COMPRESSION, CHECKSUM, INIT;';

EXEC dbo.sp_add_schedule
    @schedule_name = N'Diário 02:00',
    @freq_type = 4,
    @freq_interval = 1,
    @active_start_time = 020000;

EXEC dbo.sp_attach_schedule
    @job_name = N'Backup Diário Full',
    @schedule_name = N'Diário 02:00';

EXEC dbo.sp_add_jobserver
    @job_name = N'Backup Diário Full';
```

### Restore

#### Restore Completo
```sql
-- Restore com replace
RESTORE DATABASE MinhaBaseDeDados 
FROM DISK = 'C:\Backup\MinhaBase_Full.bak'
WITH REPLACE, CHECKDB;
```

#### Restore Point-in-Time
```sql
-- Restore até um ponto específico
RESTORE DATABASE MinhaBaseDeDados 
FROM DISK = 'C:\Backup\MinhaBase_Full.bak'
WITH NORECOVERY;

RESTORE DATABASE MinhaBaseDeDados 
FROM DISK = 'C:\Backup\MinhaBase_Diff.bak'
WITH NORECOVERY;

RESTORE LOG MinhaBaseDeDados 
FROM DISK = 'C:\Backup\MinhaBase_Log.trn'
WITH STOPAT = '2026-01-06 10:30:00';
```

---

## SQL Server Agent

### O que é o SQL Server Agent?

O SQL Server Agent é um serviço do Windows que executa tarefas administrativas agendadas, chamadas de **jobs**. É essencial para:

- **Automação de backups**
- **Manutenção de índices**
- **Execução de relatórios**
- **Monitoramento proativo**
- **Limpeza de logs**

### Componentes Principais

#### 1. Jobs (Trabalhos)
Conjuntos de tarefas que podem ser executadas em horários específicos

#### 2. Schedules (Agendamentos)
Definem quando os jobs devem ser executados

#### 3. Alerts (Alertas)
Respondem a eventos específicos do SQL Server

#### 4. Operators (Operadores)
Pessoas que recebem notificações

### Criando Jobs

#### Job Básico de Backup
```sql
USE msdb;
GO

-- 1. Criar o job
EXEC dbo.sp_add_job
    @job_name = N'Manutenção Diária',
    @enabled = 1,
    @description = N'Job de manutenção diária com backup e reorganização de índices';

-- 2. Adicionar step de backup
EXEC dbo.sp_add_jobstep
    @job_name = N'Manutenção Diária',
    @step_name = N'Backup Full',
    @step_id = 1,
    @command = N'
        BACKUP DATABASE MinhaBaseDeDados 
        TO DISK = N''C:\Backup\MinhaBase_$(ESCAPE_SQUOTE(STRTDT))_$(ESCAPE_SQUOTE(STRTTM)).bak''
        WITH COMPRESSION, CHECKSUM, INIT;
        
        PRINT ''Backup concluído com sucesso'';
    ',
    @database_name = N'MinhaBaseDeDados',
    @on_success_action = 3, -- Go to next step
    @on_fail_action = 2;    -- Quit with failure

-- 3. Adicionar step de manutenção de índices
EXEC dbo.sp_add_jobstep
    @job_name = N'Manutenção Diária',
    @step_name = N'Reorganizar Índices',
    @step_id = 2,
    @command = N'
        -- Reorganizar índices fragmentados
        DECLARE @sql NVARCHAR(MAX) = '''';
        
        SELECT @sql = @sql + 
            ''ALTER INDEX '' + i.name + 
            '' ON '' + SCHEMA_NAME(t.schema_id) + ''.'' + t.name + 
            '' REORGANIZE;'' + CHAR(13)
        FROM sys.indexes i
        INNER JOIN sys.tables t ON i.object_id = t.object_id
        INNER JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, ''LIMITED'') ps 
            ON i.object_id = ps.object_id AND i.index_id = ps.index_id
        WHERE ps.avg_fragmentation_in_percent > 10 
        AND ps.avg_fragmentation_in_percent <= 30
        AND i.index_id > 0;
        
        EXEC sp_executesql @sql;
        
        PRINT ''Reorganização de índices concluída'';
    ',
    @database_name = N'MinhaBaseDeDados';

-- 4. Criar schedule
EXEC dbo.sp_add_schedule
    @schedule_name = N'Diário 03:00',
    @freq_type = 4,        -- Daily
    @freq_interval = 1,    -- Every day
    @active_start_time = 030000; -- 03:00:00

-- 5. Anexar schedule ao job
EXEC dbo.sp_attach_schedule
    @job_name = N'Manutenção Diária',
    @schedule_name = N'Diário 03:00';

-- 6. Adicionar job ao servidor
EXEC dbo.sp_add_jobserver
    @job_name = N'Manutenção Diária';
```

### Monitoramento de Jobs

```sql
-- Verificar status dos jobs
SELECT 
    j.name AS JobName,
    j.enabled,
    ja.start_execution_date AS LastRun,
    ja.stop_execution_date AS LastStop,
    CASE ja.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Canceled'
        WHEN 4 THEN 'Running'
    END AS LastRunStatus
FROM msdb.dbo.sysjobs j
LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id
WHERE ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.sysjobactivity);

-- Histórico detalhado de execução
SELECT 
    j.name AS JobName,
    h.step_name,
    h.run_date,
    h.run_time,
    h.run_duration,
    CASE h.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Canceled'
    END AS Status,
    h.message
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
WHERE h.run_date >= CONVERT(int, CONVERT(varchar, GETDATE(), 112)) -- Today
ORDER BY h.run_date DESC, h.run_time DESC;
```

---

## Tuning e Otimização

### O que é Tuning?

Tuning é o processo de otimização do desempenho do banco de dados através de:
- **Análise de consultas lentas**
- **Otimização de índices**
- **Ajuste de configurações**
- **Melhoria da arquitetura**

### Identificando Problemas de Performance

#### 1. Consultas mais custosas
```sql
-- Top 10 consultas mais custosas
SELECT TOP 10
    qs.total_worker_time / qs.execution_count AS AvgCPUTime,
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS AvgElapsedTime,
    qs.total_logical_reads / qs.execution_count AS AvgLogicalReads,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS QueryText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time / qs.execution_count DESC;
```

#### 2. Processos bloqueados
```sql
-- Identificar bloqueios
SELECT 
    blocking.session_id AS BlockingSessionId,
    blocked.session_id AS BlockedSessionId,
    blocking_sql.text AS BlockingQuery,
    blocked_sql.text AS BlockedQuery,
    blocking.wait_type,
    blocking.wait_time,
    blocking.last_wait_type
FROM sys.dm_exec_requests blocking
INNER JOIN sys.dm_exec_requests blocked ON blocking.session_id = blocked.blocking_session_id
CROSS APPLY sys.dm_exec_sql_text(blocking.sql_handle) blocking_sql
CROSS APPLY sys.dm_exec_sql_text(blocked.sql_handle) blocked_sql;
```

### Dicas de Tuning

#### 1. Quando realizar Tuning
- **Performance degradada** (consultas lentas)
- **Alto consumo de CPU** (>80% consistente)
- **Muitos bloqueios** (deadlocks frequentes)
- **I/O excessivo** (reads/writes altos)
- **Memória insuficiente** (page life expectancy baixo)
- **Crescimento descontrolado** (log/data files)

#### 2. Como realizar Tuning

##### Análise de Waits
```sql
-- Top wait statistics
SELECT 
    wait_type,
    wait_time_ms,
    max_wait_time_ms,
    signal_wait_time_ms,
    wait_time_ms - signal_wait_time_ms AS resource_wait_time_ms,
    waiting_tasks_count,
    wait_time_ms / waiting_tasks_count AS avg_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE waiting_tasks_count > 0
    AND wait_type NOT IN (
        'CLR_SEMAPHORE', 'LAZYWRITER_SLEEP', 'RESOURCE_QUEUE',
        'SLEEP_TASK', 'SLEEP_SYSTEMTASK', 'SQLTRACE_BUFFER_FLUSH', 'WAITFOR'
    )
ORDER BY wait_time_ms DESC;
```

##### Análise de I/O
```sql
-- I/O Statistics por arquivo
SELECT 
    mf.database_id,
    DB_NAME(mf.database_id) AS database_name,
    mf.file_id,
    mf.name AS logical_name,
    mf.physical_name,
    ios.num_of_reads,
    ios.num_of_writes,
    ios.num_of_bytes_read,
    ios.num_of_bytes_written,
    ios.io_stall_read_ms,
    ios.io_stall_write_ms,
    CASE 
        WHEN ios.num_of_reads > 0 
        THEN ios.io_stall_read_ms / ios.num_of_reads 
        ELSE 0 
    END AS avg_read_stall_ms,
    CASE 
        WHEN ios.num_of_writes > 0 
        THEN ios.io_stall_write_ms / ios.num_of_writes 
        ELSE 0 
    END AS avg_write_stall_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) ios
INNER JOIN sys.master_files mf ON ios.database_id = mf.database_id 
    AND ios.file_id = mf.file_id
ORDER BY ios.io_stall_read_ms + ios.io_stall_write_ms DESC;
```

---

## Estatísticas no SQL Server

### O que são Estatísticas?

Estatísticas são **metadados sobre a distribuição de dados** nas colunas e índices que o otimizador de consultas usa para:
- **Estimar cardinalidade** (número de linhas)
- **Escolher planos de execução** otimizados
- **Decidir sobre joins** e operações
- **Selecionar índices** apropriados

### Como o SQL Server usa Estatísticas

```sql
-- Visualizar estatísticas existentes
SELECT 
    s.name AS statistics_name,
    c.name AS column_name,
    s.stats_id,
    s.has_filter,
    s.filter_definition,
    s.is_temporary,
    STATS_DATE(s.object_id, s.stats_id) AS last_updated
FROM sys.stats s
INNER JOIN sys.stats_columns sc ON s.object_id = sc.object_id 
    AND s.stats_id = sc.stats_id
INNER JOIN sys.columns c ON sc.object_id = c.object_id 
    AND sc.column_id = c.column_id
WHERE s.object_id = OBJECT_ID('MinhaTabela')
ORDER BY s.name, sc.stats_column_id;
```

### Manutenção de Estatísticas

#### Atualização Automática
```sql
-- Verificar configurações auto stats
SELECT 
    name,
    is_auto_create_stats_on,
    is_auto_update_stats_on,
    is_auto_update_stats_async_on
FROM sys.databases
WHERE name = 'MinhaBaseDeDados';

-- Configurar auto stats
ALTER DATABASE MinhaBaseDeDados SET AUTO_CREATE_STATISTICS ON;
ALTER DATABASE MinhaBaseDeDados SET AUTO_UPDATE_STATISTICS ON;
ALTER DATABASE MinhaBaseDeDados SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

#### Atualização Manual
```sql
-- Atualizar todas as estatísticas de uma tabela
UPDATE STATISTICS MinhaTabela;

-- Atualizar estatística específica
UPDATE STATISTICS MinhaTabela IX_MinhaTabela_Coluna WITH FULLSCAN;

-- Atualizar todas as estatísticas do banco
EXEC sp_updatestats;

-- Job para atualização de estatísticas
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'Atualizar Estatísticas',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Atualizar Estatísticas',
    @step_name = N'Update Stats',
    @command = N'
        -- Atualizar apenas estatísticas desatualizadas
        DECLARE @sql NVARCHAR(MAX) = '''';
        
        SELECT @sql = @sql + 
            ''UPDATE STATISTICS '' + 
            QUOTENAME(SCHEMA_NAME(t.schema_id)) + ''.'' + 
            QUOTENAME(t.name) + '' '' + 
            QUOTENAME(s.name) + '' WITH SAMPLE 25 PERCENT;'' + CHAR(13)
        FROM sys.stats s
        INNER JOIN sys.tables t ON s.object_id = t.object_id
        WHERE STATS_DATE(s.object_id, s.stats_id) < DATEADD(day, -7, GETDATE())
           OR STATS_DATE(s.object_id, s.stats_id) IS NULL;
        
        EXEC sp_executesql @sql;
    ';

-- Schedule semanal
EXEC dbo.sp_add_schedule
    @schedule_name = N'Semanal Domingo',
    @freq_type = 8,        -- Weekly
    @freq_interval = 1,    -- Sunday
    @active_start_time = 020000;

EXEC dbo.sp_attach_schedule
    @job_name = N'Atualizar Estatísticas',
    @schedule_name = N'Semanal Domingo';

EXEC dbo.sp_add_jobserver
    @job_name = N'Atualizar Estatísticas';
```

---

## Seek vs Scan

### Index Seek (Busca)

**Seek** é uma operação **eficiente** onde o SQL Server:
- Usa o índice para **localizar diretamente** as linhas
- **Navega através da estrutura** B-Tree
- Acessa apenas **dados necessários**
- **Menor I/O** e tempo de execução

```sql
-- Exemplo que gera INDEX SEEK
SELECT * 
FROM Produtos 
WHERE ProdutoID = 123;  -- Usando chave primária

-- Seek com range
SELECT * 
FROM Vendas 
WHERE DataVenda BETWEEN '2026-01-01' AND '2026-01-31'
    AND VendedorID = 5;  -- Índice composto otimizado
```

### Index Scan (Varredura)

**Scan** é uma operação **custosa** onde o SQL Server:
- **Examina todas as páginas** do índice/tabela
- Não pode usar a estrutura de índice eficientemente
- **Alto I/O** e tempo de execução
- Geralmente indica **falta de índices** adequados

```sql
-- Exemplo que gera INDEX SCAN (problema!)
SELECT * 
FROM Produtos 
WHERE UPPER(NomeProduto) LIKE '%NOTEBOOK%';  -- Função na coluna

-- Table Scan ainda pior
SELECT * 
FROM VendasHistorico 
WHERE YEAR(DataVenda) = 2025;  -- Função na coluna indexada
```

### Identificando Seek vs Scan

```sql
-- Query para identificar scans custosos
SELECT 
    cp.objtype,
    cp.cacheobjtype,
    cp.usecounts,
    cp.size_in_bytes,
    qs.execution_count,
    qs.total_logical_reads,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS QueryText
FROM sys.dm_exec_cached_plans cp
INNER JOIN sys.dm_exec_query_stats qs ON cp.plan_handle = qs.plan_handle
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.total_logical_reads > 1000  -- Queries com muitas leituras
ORDER BY qs.total_logical_reads DESC;
```

### Convertendo Scan para Seek

#### Problema: Function em coluna indexada
```sql
-- ❌ PROBLEMA - Gera SCAN
SELECT * FROM Vendas 
WHERE YEAR(DataVenda) = 2025;

-- ✅ SOLUÇÃO - Gera SEEK  
SELECT * FROM Vendas 
WHERE DataVenda >= '2025-01-01' 
    AND DataVenda < '2026-01-01';
```

#### Problema: LIKE com wildcard no início
```sql
-- ❌ PROBLEMA - Gera SCAN
SELECT * FROM Produtos 
WHERE NomeProduto LIKE '%Notebook%';

-- ✅ SOLUÇÃO - Full Text Search ou índice invertido
CREATE FULLTEXT INDEX ON Produtos(NomeProduto);

SELECT * FROM Produtos 
WHERE CONTAINS(NomeProduto, 'Notebook');
```

#### Criar índices apropriados
```sql
-- Análise de índices em falta
SELECT 
    mid.statement AS ObjectName,
    mid.column_id,
    mid.column_name,
    mid.column_usage,
    migs.user_seeks,
    migs.user_scans,
    migs.last_user_seek,
    migs.avg_total_user_cost,
    migs.avg_user_impact
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 50  -- Impacto > 50%
ORDER BY migs.avg_user_impact DESC;

-- Script para criar índice sugerido
SELECT 
    'CREATE INDEX IX_' + 
    REPLACE(REPLACE(REPLACE(OBJECT_NAME(mid.object_id), '[', ''), ']', ''), '.', '_') + 
    '_Suggested ON ' + mid.statement + 
    ' (' + ISNULL(mid.equality_columns, '') + 
    CASE WHEN mid.inequality_columns IS NOT NULL 
         THEN ', ' + mid.inequality_columns 
         ELSE '' END + ')' +
    CASE WHEN mid.included_columns IS NOT NULL 
         THEN ' INCLUDE (' + mid.included_columns + ')' 
         ELSE '' END AS CreateIndexScript
FROM sys.dm_db_missing_index_details mid
INNER JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE migs.avg_user_impact > 50
ORDER BY migs.avg_user_impact DESC;
```

---

## Query Store

### O que é o Query Store?

Query Store é uma funcionalidade que **captura automaticamente** o histórico de:
- **Queries executadas**
- **Planos de execução**  
- **Estatísticas de runtime**
- **Métricas de performance**

É como uma "caixa preta" para troubleshooting de performance.

### Habilitando o Query Store

```sql
-- Habilitar Query Store
ALTER DATABASE MinhaBaseDeDados 
SET QUERY_STORE = ON (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    MAX_STORAGE_SIZE_MB = 1024,
    INTERVAL_LENGTH_MINUTES = 60,
    SIZE_BASED_CLEANUP_MODE = AUTO,
    QUERY_CAPTURE_MODE = AUTO
);

-- Verificar configuração
SELECT 
    desired_state_desc,
    actual_state_desc,
    readonly_reason,
    current_storage_size_mb,
    max_storage_size_mb,
    flush_interval_seconds,
    interval_length_minutes,
    stale_query_threshold_days,
    size_based_cleanup_mode_desc,
    query_capture_mode_desc
FROM sys.database_query_store_options;
```

### Utilizando o Query Store

#### 1. Identificar Queries Regressivas
```sql
-- Queries com degradação de performance
SELECT 
    qsq.query_id,
    qst.query_sql_text,
    qsp.plan_id,
    qsrs.runtime_stats_interval_id,
    qsrs.execution_count,
    qsrs.avg_duration,
    qsrs.avg_logical_io_reads,
    qsrs.avg_cpu_time,
    qsrs.stdev_duration
FROM sys.query_store_query qsq
INNER JOIN sys.query_store_query_text qst ON qsq.query_text_id = qst.query_text_id
INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
INNER JOIN sys.query_store_runtime_stats qsrs ON qsp.plan_id = qsrs.plan_id
WHERE qsrs.last_execution_time >= DATEADD(day, -7, GETDATE())
    AND qsrs.avg_duration > 10000  -- > 10 segundos
ORDER BY qsrs.avg_duration DESC;
```

#### 2. Comparar Planos de Execução
```sql
-- Comparar diferentes planos para a mesma query
WITH PlanComparison AS (
    SELECT 
        qsq.query_id,
        qsp.plan_id,
        qsrs.avg_duration,
        qsrs.avg_logical_io_reads,
        qsrs.execution_count,
        ROW_NUMBER() OVER (PARTITION BY qsq.query_id ORDER BY qsrs.avg_duration) as rn
    FROM sys.query_store_query qsq
    INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
    INNER JOIN sys.query_store_runtime_stats qsrs ON qsp.plan_id = qsrs.plan_id
)
SELECT 
    pc1.query_id,
    pc1.plan_id AS best_plan_id,
    pc1.avg_duration AS best_avg_duration,
    pc2.plan_id AS worst_plan_id,  
    pc2.avg_duration AS worst_avg_duration,
    ((pc2.avg_duration - pc1.avg_duration) * 100.0 / pc1.avg_duration) AS performance_difference_pct
FROM PlanComparison pc1
INNER JOIN PlanComparison pc2 ON pc1.query_id = pc2.query_id
WHERE pc1.rn = 1 AND pc2.rn = (SELECT MAX(rn) FROM PlanComparison WHERE query_id = pc1.query_id)
    AND pc2.avg_duration > pc1.avg_duration * 1.5  -- 50% pior
ORDER BY performance_difference_pct DESC;
```

#### 3. Forçar Planos de Execução
```sql
-- Forçar um plano específico (Plan Forcing)
EXEC sp_query_store_force_plan 
    @query_id = 12345, 
    @plan_id = 67890;

-- Remover força de plano
EXEC sp_query_store_unforce_plan 
    @query_id = 12345, 
    @plan_id = 67890;

-- Ver planos forçados
SELECT 
    qsq.query_id,
    qst.query_sql_text,
    qsp.plan_id,
    qsp.is_forced_plan,
    qsp.force_failure_count,
    qsp.last_force_failure_reason_desc
FROM sys.query_store_query qsq
INNER JOIN sys.query_store_query_text qst ON qsq.query_text_id = qst.query_text_id
INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
WHERE qsp.is_forced_plan = 1;
```

### Manutenção do Query Store

```sql
-- Limpeza manual
ALTER DATABASE MinhaBaseDeDados SET QUERY_STORE CLEAR;

-- Job de manutenção do Query Store
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'Query Store Maintenance',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Query Store Maintenance',
    @step_name = N'Cleanup Old Data',
    @command = N'
        -- Limpar dados antigos do Query Store
        DECLARE @retention_days INT = 30;
        
        -- Remover runtime stats antigas
        DELETE FROM sys.query_store_runtime_stats 
        WHERE last_execution_time < DATEADD(day, -@retention_days, GETDATE());
        
        -- Verificar espaço usado
        DECLARE @current_size_mb INT, @max_size_mb INT;
        SELECT 
            @current_size_mb = current_storage_size_mb,
            @max_size_mb = max_storage_size_mb
        FROM sys.database_query_store_options;
        
        IF (@current_size_mb > @max_size_mb * 0.8)
        BEGIN
            -- Se usando mais de 80% do espaço, fazer limpeza agressiva
            EXEC sp_query_store_flush_db;
        END
    ';

-- Schedule semanal
EXEC dbo.sp_add_schedule
    @schedule_name = N'Weekly Sunday 01:00',
    @freq_type = 8,
    @freq_interval = 1,
    @active_start_time = 010000;

EXEC dbo.sp_attach_schedule
    @job_name = N'Query Store Maintenance',
    @schedule_name = N'Weekly Sunday 01:00';

EXEC dbo.sp_add_jobserver
    @job_name = N'Query Store Maintenance';
```

---

## Sistema de Alertas

### Configuração de Operadores

```sql
-- Criar operador para receber alertas
USE msdb;
GO

EXEC dbo.sp_add_operator
    @name = N'DBA Team',
    @enabled = 1,
    @email_address = N'dba@empresa.com',
    @pager_address = N'',
    @weekday_pager_start_time = 080000,
    @weekday_pager_end_time = 180000;
```

### 1. Alerta para Processos Bloqueados

```sql
-- Alerta para bloqueios prolongados
EXEC dbo.sp_add_alert
    @name = N'Blocked Processes',
    @message_id = 1205,  -- Deadlock detected
    @severity = 0,
    @notification_message = N'Deadlock detectado no SQL Server. Verificar processos bloqueados.',
    @response_message = N'Investigar bloqueios imediatamente.',
    @include_event_description_in = 1;

-- Notificar operador
EXEC dbo.sp_add_notification
    @alert_name = N'Blocked Processes',
    @operator_name = N'DBA Team',
    @notification_method = 1; -- Email

-- Job customizado para monitorar bloqueios
EXEC dbo.sp_add_job
    @job_name = N'Monitor Blocked Processes',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Monitor Blocked Processes',
    @step_name = N'Check for Long Running Blocks',
    @command = N'
        DECLARE @blocked_count INT;
        
        SELECT @blocked_count = COUNT(*)
        FROM sys.dm_exec_requests r1
        WHERE r1.blocking_session_id <> 0
            AND EXISTS (
                SELECT 1 FROM sys.dm_exec_requests r2 
                WHERE r2.session_id = r1.blocking_session_id
                AND DATEDIFF(minute, r2.start_time, GETDATE()) > 5
            );
        
        IF @blocked_count > 0
        BEGIN
            DECLARE @msg NVARCHAR(500);
            SET @msg = ''ALERTA: '' + CAST(@blocked_count AS VARCHAR) + 
                      '' processos bloqueados há mais de 5 minutos'';
            
            EXEC msdb.dbo.sp_send_dbmail
                @recipients = ''dba@empresa.com'',
                @subject = ''Processos Bloqueados Detectados'',
                @body = @msg,
                @importance = ''High'';
        END
    ';

-- Schedule a cada 2 minutos
EXEC dbo.sp_add_schedule
    @schedule_name = N'Every 2 Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 2;

EXEC dbo.sp_attach_schedule
    @job_name = N'Monitor Blocked Processes',
    @schedule_name = N'Every 2 Minutes';

EXEC dbo.sp_add_jobserver
    @job_name = N'Monitor Blocked Processes';
```

### 2. Alerta de Status do Database

```sql
-- Alerta para mudanças de estado do banco
EXEC dbo.sp_add_alert
    @name = N'Database State Change',
    @message_id = 5069,  -- Database state change
    @severity = 0,
    @notification_message = N'Estado do banco de dados foi alterado.',
    @include_event_description_in = 1;

EXEC dbo.sp_add_notification
    @alert_name = N'Database State Change',
    @operator_name = N'DBA Team',
    @notification_method = 1;

-- Job para monitorar status dos bancos críticos
EXEC dbo.sp_add_job
    @job_name = N'Database Status Monitor',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Database Status Monitor',
    @step_name = N'Check Critical Databases',
    @command = N'
        DECLARE @db_issues TABLE (
            DatabaseName NVARCHAR(128),
            State NVARCHAR(60),
            Issue NVARCHAR(500)
        );
        
        -- Verificar bancos não online
        INSERT INTO @db_issues
        SELECT 
            name,
            state_desc,
            ''Database não está ONLINE''
        FROM sys.databases
        WHERE name NOT IN (''tempdb'', ''model'', ''msdb'')
            AND state_desc <> ''ONLINE'';
        
        -- Verificar bancos em modo de emergência
        INSERT INTO @db_issues
        SELECT 
            name,
            state_desc,
            ''Database em modo EMERGENCY''
        FROM sys.databases
        WHERE user_access_desc = ''EMERGENCY'';
        
        -- Verificar bancos suspeitos
        INSERT INTO @db_issues
        SELECT 
            name,
            state_desc,
            ''Database marcado como SUSPECT''
        FROM sys.databases
        WHERE state_desc = ''SUSPECT'';
        
        IF EXISTS (SELECT 1 FROM @db_issues)
        BEGIN
            DECLARE @body NVARCHAR(MAX) = ''ALERTA: Problemas detectados nos bancos de dados:'' + CHAR(13) + CHAR(10);
            
            SELECT @body = @body + 
                ''- '' + DatabaseName + '': '' + Issue + '' (Estado: '' + State + '')'' + CHAR(13) + CHAR(10)
            FROM @db_issues;
            
            EXEC msdb.dbo.sp_send_dbmail
                @recipients = ''dba@empresa.com'',
                @subject = ''CRÍTICO: Problemas no Status dos Databases'',
                @body = @body,
                @importance = ''High'';
        END
    ';

-- Schedule a cada 5 minutos
EXEC dbo.sp_add_schedule
    @schedule_name = N'Every 5 Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 5;

EXEC dbo.sp_attach_schedule
    @job_name = N'Database Status Monitor',
    @schedule_name = N'Every 5 Minutes';

EXEC dbo.sp_add_jobserver
    @job_name = N'Database Status Monitor';
```

### 3. Alerta para Falha em Jobs

```sql
-- Alerta automático para falhas em jobs
EXEC dbo.sp_add_alert
    @name = N'Job Failed',
    @message_id = 50000,
    @severity = 16,
    @notification_message = N'Job crítico falhou. Verificar imediatamente.',
    @include_event_description_in = 1;

EXEC dbo.sp_add_notification
    @alert_name = N'Job Failed',
    @operator_name = N'DBA Team',
    @notification_method = 1;

-- Job para monitorar falhas em jobs críticos
EXEC dbo.sp_add_job
    @job_name = N'Job Failure Monitor',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Job Failure Monitor',
    @step_name = N'Check Failed Jobs',
    @command = N'
        DECLARE @failed_jobs TABLE (
            JobName NVARCHAR(128),
            LastRunDate INT,
            LastRunTime INT,
            RunStatus INT,
            Message NVARCHAR(4000)
        );
        
        -- Jobs críticos que falharam nas últimas 24 horas
        INSERT INTO @failed_jobs
        SELECT 
            j.name,
            h.run_date,
            h.run_time,
            h.run_status,
            h.message
        FROM msdb.dbo.sysjobs j
        INNER JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
        WHERE h.run_status = 0  -- Failed
            AND h.run_date >= CONVERT(int, CONVERT(varchar, DATEADD(day, -1, GETDATE()), 112))
            AND j.name IN (
                ''Backup Diário Full'',
                ''Manutenção Diária'',
                ''Atualizar Estatísticas''
            )
            AND h.step_id = 0;  -- Job outcome, not individual step
        
        IF EXISTS (SELECT 1 FROM @failed_jobs)
        BEGIN
            DECLARE @body NVARCHAR(MAX) = ''ALERTA: Jobs críticos falharam:'' + CHAR(13) + CHAR(10);
            
            SELECT @body = @body + 
                ''- '' + JobName + 
                '' (Data: '' + CAST(LastRunDate AS VARCHAR) + 
                '', Status: '' + CASE RunStatus 
                    WHEN 0 THEN ''Falhou''
                    WHEN 2 THEN ''Retry''  
                    WHEN 3 THEN ''Cancelado''
                END + '')'' + CHAR(13) + CHAR(10) +
                ''  Mensagem: '' + LEFT(Message, 200) + CHAR(13) + CHAR(10) + CHAR(13) + CHAR(10)
            FROM @failed_jobs;
            
            EXEC msdb.dbo.sp_send_dbmail
                @recipients = ''dba@empresa.com'',
                @subject = ''CRÍTICO: Falha em Jobs Críticos'',
                @body = @body,
                @importance = ''High'';
        END
    ';

-- Schedule a cada 30 minutos
EXEC dbo.sp_add_schedule
    @schedule_name = N'Every 30 Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 30;

EXEC dbo.sp_attach_schedule
    @job_name = N'Job Failure Monitor',
    @schedule_name = N'Every 30 Minutes';

EXEC dbo.sp_add_jobserver
    @job_name = N'Job Failure Monitor';
```

### 4. Alertas de Severidade

```sql
-- Alertas para diferentes níveis de severidade
EXEC dbo.sp_add_alert
    @name = N'Severity 016 - User Error',
    @severity = 16,
    @notification_message = N'Erro de usuário detectado (Severidade 16).',
    @include_event_description_in = 1;

EXEC dbo.sp_add_alert
    @name = N'Severity 017 - Insufficient Resources',
    @severity = 17,  
    @notification_message = N'Recursos insuficientes (Severidade 17).',
    @include_event_description_in = 1;

EXEC dbo.sp_add_alert
    @name = N'Severity 018 - System Error',
    @severity = 18,
    @notification_message = N'Erro de sistema não fatal (Severidade 18).',
    @include_event_description_in = 1;

EXEC dbo.sp_add_alert
    @name = N'Severity 019 - Fatal Error',
    @severity = 19,
    @notification_message = N'CRÍTICO: Erro fatal no SQL Server (Severidade 19).',
    @include_event_description_in = 1;

EXEC dbo.sp_add_alert
    @name = N'Severity 020 - System Problem',
    @severity = 20,
    @notification_message = N'CRÍTICO: Problema grave no sistema (Severidade 20).',
    @include_event_description_in = 1;

-- Notificar operador para alertas críticos
EXEC dbo.sp_add_notification
    @alert_name = N'Severity 019 - Fatal Error',
    @operator_name = N'DBA Team',
    @notification_method = 1;

EXEC dbo.sp_add_notification
    @alert_name = N'Severity 020 - System Problem',
    @operator_name = N'DBA Team',
    @notification_method = 1;
```

### 5. Alerta para Banco Corrompido

```sql
-- Alerta específico para corrupção
EXEC dbo.sp_add_alert
    @name = N'Database Corruption',
    @message_id = 824,  -- I/O error
    @severity = 0,
    @notification_message = N'CRÍTICO: Possível corrupção detectada no banco de dados.',
    @include_event_description_in = 1;

EXEC dbo.sp_add_notification
    @alert_name = N'Database Corruption',
    @operator_name = N'DBA Team',
    @notification_method = 1;

-- Job para verificação de integridade
EXEC dbo.sp_add_job
    @job_name = N'Database Integrity Check',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Database Integrity Check',
    @step_name = N'DBCC CHECKDB All Databases',
    @command = N'
        DECLARE @db_name NVARCHAR(128);
        DECLARE @sql NVARCHAR(MAX);
        DECLARE @corruption_found BIT = 0;
        
        DECLARE db_cursor CURSOR FOR
        SELECT name FROM sys.databases
        WHERE state_desc = ''ONLINE''
            AND name NOT IN (''tempdb'', ''model'');
        
        OPEN db_cursor;
        FETCH NEXT FROM db_cursor INTO @db_name;
        
        WHILE @@FETCH_STATUS = 0
        BEGIN
            BEGIN TRY
                SET @sql = ''DBCC CHECKDB ('''''' + @db_name + '''''') WITH NO_INFOMSGS'';
                EXEC sp_executesql @sql;
                PRINT ''DBCC CHECKDB completed for '' + @db_name + '' - No errors found'';
            END TRY
            BEGIN CATCH
                SET @corruption_found = 1;
                
                DECLARE @error_message NVARCHAR(4000) = 
                    ''CRÍTICO: Corrupção detectada no banco '' + @db_name + 
                    CHAR(13) + CHAR(10) + ''Erro: '' + ERROR_MESSAGE();
                
                EXEC msdb.dbo.sp_send_dbmail
                    @recipients = ''dba@empresa.com'',
                    @subject = ''CRÍTICO: Corrupção Detectada'',
                    @body = @error_message,
                    @importance = ''High'';
                    
                PRINT @error_message;
            END CATCH
            
            FETCH NEXT FROM db_cursor INTO @db_name;
        END
        
        CLOSE db_cursor;
        DEALLOCATE db_cursor;
        
        IF @corruption_found = 0
        BEGIN
            PRINT ''Verificação de integridade concluída - Todos os bancos estão íntegros'';
        END
    ';

-- Schedule semanal (domingo de madrugada)
EXEC dbo.sp_add_schedule
    @schedule_name = N'Weekly Sunday 04:00',
    @freq_type = 8,
    @freq_interval = 1,
    @active_start_time = 040000;

EXEC dbo.sp_attach_schedule
    @job_name = N'Database Integrity Check',
    @schedule_name = N'Weekly Sunday 04:00';

EXEC dbo.sp_add_jobserver
    @job_name = N'Database Integrity Check';
```

### Configuração do Database Mail

```sql
-- Configurar Database Mail para envio de alertas
EXEC sp_configure 'Database Mail XPs', 1;
RECONFIGURE;

-- Criar profile de email
EXEC msdb.dbo.sysmail_add_profile_sp
    @profile_name = 'DBA Profile',
    @description = 'Profile para alertas do DBA';

-- Criar conta de email  
EXEC msdb.dbo.sysmail_add_account_sp
    @account_name = 'DBA Account',
    @description = 'Conta para alertas DBA',
    @email_address = 'sqlserver@empresa.com',
    @display_name = 'SQL Server DBA',
    @mailserver_name = 'smtp.empresa.com',
    @port = 587,
    @enable_ssl = 1,
    @username = 'sqlserver@empresa.com',
    @password = 'SenhaDoEmail';

-- Associar conta ao profile
EXEC msdb.dbo.sysmail_add_profileaccount_sp
    @profile_name = 'DBA Profile',
    @account_name = 'DBA Account',
    @sequence_number = 1;

-- Definir profile padrão público
EXEC msdb.dbo.sysmail_add_principalprofile_sp
    @profile_name = 'DBA Profile',
    @principal_name = 'public',
    @is_default = 1;

-- Testar envio de email
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'DBA Profile',
    @recipients = 'dba@empresa.com',
    @subject = 'Teste Database Mail',
    @body = 'Este é um teste do sistema de alertas do SQL Server.';
```

---

## Scripts de Monitoramento Úteis

### Monitoramento Geral de Performance

```sql
-- Dashboard de monitoramento
SELECT 
    'CPU Usage' AS Metric,
    CAST(100 - AVG(record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')) AS VARCHAR) + '%' AS Value
FROM (
    SELECT TOP 30 CONVERT(xml, record) AS record
    FROM sys.dm_os_ring_buffers
    WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
    ORDER BY timestamp DESC
) AS cpu_records

UNION ALL

SELECT 
    'Memory Usage',
    CAST((total_physical_memory_kb - available_physical_memory_kb) * 100 / total_physical_memory_kb AS VARCHAR) + '%'
FROM sys.dm_os_sys_memory

UNION ALL

SELECT 
    'Buffer Pool Hit Ratio',
    CAST(
        (SELECT cntr_value FROM sys.dm_os_performance_counters 
         WHERE counter_name = 'Buffer cache hit ratio' AND instance_name = '') * 100.0 /
        (SELECT cntr_value FROM sys.dm_os_performance_counters 
         WHERE counter_name = 'Buffer cache hit ratio base' AND instance_name = '')
    AS VARCHAR) + '%'

UNION ALL

SELECT 
    'Page Life Expectancy',
    CAST((SELECT cntr_value FROM sys.dm_os_performance_counters 
          WHERE counter_name = 'Page life expectancy') AS VARCHAR) + ' seconds';
```

### Limpeza e Manutenção Automatizada

```sql
-- Job de limpeza geral
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'Maintenance Cleanup',
    @enabled = 1;

EXEC dbo.sp_add_jobstep
    @job_name = N'Maintenance Cleanup',
    @step_name = N'Cleanup Tasks',
    @command = N'
        -- Limpar backup history antigo
        EXEC sp_delete_backuphistory @oldest_date = ''2025-12-01'';
        
        -- Limpar job history antigo  
        EXEC sp_purge_jobhistory @oldest_date = ''2025-12-01'';
        
        -- Limpar Database Mail log
        EXEC msdb.dbo.sysmail_delete_log_sp @logged_before = ''2025-12-01'';
        
        -- Encolher log do msdb se necessário
        USE msdb;
        IF (SELECT size*8/1024 FROM sys.database_files WHERE name = ''MSDBLog'') > 500
        BEGIN
            DBCC SHRINKFILE(MSDBLog, 100);
        END
        
        -- Atualizar estatísticas do msdb
        EXEC sp_updatestats;
        
        PRINT ''Limpeza de manutenção concluída'';
    ';

-- Schedule mensal
EXEC dbo.sp_add_schedule
    @schedule_name = N'Monthly First Sunday',
    @freq_type = 16,
    @freq_interval = 1,
    @active_start_time = 050000;

EXEC dbo.sp_attach_schedule
    @job_name = N'Maintenance Cleanup',
    @schedule_name = N'Monthly First Sunday';

EXEC dbo.sp_add_jobserver
    @job_name = N'Maintenance Cleanup';
```

---

## Conclusão

Documentação com alguns dos principais aspectos da administração do SQL Server

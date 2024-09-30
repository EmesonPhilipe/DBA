## **1. Checagens do Sistema Operacional (S.O.)**

1.1. **Verificar a distribuição Linux**  
   - Comando:  
     ```bash
     cat /etc/redhat-release
     ```

1.2. **Checar arquitetura 32 ou 64 bits**  
   - Comando:  
     ```bash
     uname -a
     ```

1.3. **Verificar a quantidade de memória**  
   - Comando:  
     ```bash
     free -m
     ```
   - Observação: O sistema deve ter, preferencialmente, 8GB ou mais de RAM.

## 2 - Checagens de Banco de Origem

### 2.1 - Checar as extensões instaladas no PostgreSQL atual
```sql
postgres=# \dx
```
ou
```sql
postgres=# select * from pg_extension;
```

**Exemplo de Saída**:
```text
 extname | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition
---------+----------+--------------+----------------+------------+-----------+--------------
 plpgsql |       10 |           11 | f              | 1.0        |           |
(1 row)
```

### 2.2 - Checar o locale e encoding do banco de dados de origem
```sql
psql> \l
```

### 2.3 - Ajustes do HOME do usuário postgres
```bash
mkdir /home/postgres
chown postgres.postgres /home/postgres
```

#### Ajuste no arquivo `/etc/passwd`:
```bash
vi /etc/passwd
```
Trocar o diretório HOME (penúltimo parâmetro), para `/home/postgres`:
```text
postgres:x:1001:1001:PostgreSQL:/home/postgres:/bin/bash
```

#### Ajustar `.bash_profile` do usuário `postgres`:
```bash
export PGHOME=/usr/pgsql-12
export PGDATA=/opt/PostgreSQL/12.20
export PGDATABASE=postgres
export PATH=$PGHOME/bin:$PATH
export PGUSER=postgres
export PGPORT=5432
```

Testar se as variáveis foram aplicadas corretamente:
```bash
echo $PGHOME
echo $PGDATA
```

## 3 - Instalar/Configurar o repositório PostgreSQL 12.x para 64 bits

### Versão 6.x
```bash
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-6-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### Versão 7.x
```bash
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### Versão 8.x
```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### Versão 9.x
```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

## 4 - Instalar a versão do PostgreSQL

### Versões 6.x e 7.x
```bash
yum -y install postgresql12-server
yum -y install postgresql12-contrib
```

### Versões 8.x e 9.x
```bash
dnf -y install postgresql12-server
dnf -y install postgresql12-contrib
```

**Obs:** Por padrão, o PostgreSQL será instalado na pasta `/usr/pgsql-12/`, que será o PGHOME.

## 5 - Inicializar o banco apontando para um novo PGDATA

### 5.1 - Criar a pasta do PGDATA
```bash
mkdir /opt/PostgreSQL/12.20
chown postgres.postgres /opt/PostgreSQL/12.20
```

### 5.2 - Inicializar a instância PostgreSQL 12
```bash
/usr/pgsql-12/bin/initdb -D /opt/PostgreSQL/12.20
```
**Obs:** Prestar atenção nos parâmetros de locale conforme o banco de origem.

## 6 - Ajustes na instância criada

### 6.1 - Ajustar configurações no novo PGDATA
Editar o arquivo `postgresql.conf`:
```bash
vi /opt/PostgreSQL/12.20/postgresql.conf
```
Alterar a porta, por exemplo, para 5440.

### 6.2 - Adicionar extensões de acordo com o passo 2.1

#### 6.2.1 - Iniciar o novo cluster
```bash
/usr/pgsql-12/bin/pg_ctl start -D /opt/PostgreSQL/12.20
```

#### 6.2.2 - Criar a extensão (exemplo adminpack):
```sql
psql> create extension adminpack;
```

#### 6.2.3 - Confirmar que a extensão foi instalada:
```sql
psql> \dx
```

#### 6.2.4 - Parar o novo cluster:
```bash
/usr/pgsql-12/bin/pg_ctl stop -D /opt/PostgreSQL/12.20
```

## 7 - Realizar o upgrade

Durante o processo, o `pg_upgrade` levantará os dois bancos para realizar a migração.

### 7.1 - Rodar em modo de checagem (opção `-c`):
```bash
/usr/pgsql-12/bin/pg_upgrade -b /opt/PostgreSQL/9.4/bin/ -B /usr/pgsql-12/bin/ -d /opt/PostgreSQL/9.4/data/ -D /opt/PostgreSQL/12.20/ -p 5432 -P 5440 -r -k -c
```

### 7.2 - Parar o banco de dados de origem e fazer backup

#### 7.2.1 - Parar o banco de origem:
```bash
systemctl stop postgresql9.4@service
```

#### 7.2.2 - Criar snapshot:
```bash
lvcreate -s -n lv_snap_db -L 20g /dev/rhel/opt
```

### 7.3 - Rodar a execução definitiva:
```bash
/usr/pgsql-12/bin/pg_upgrade -b /opt/PostgreSQL/9.4/bin/ -B /usr/pgsql-12/bin/ -d /opt/PostgreSQL/9.4/data/ -D /opt/PostgreSQL/12.20/ -p 5432 -P 5440 -r -k
```

## 8 - Ajustar configurações no novo `postgresql.conf`

### 8.1 - Exemplo de ajustes:
```bash
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 2GB
work_mem = 8MB
maintenance_work_mem = 256MB
max_wal_size = 4GB
min_wal_size = 256MB
log_directory = 'log'
log_rotation_size = 256MB
datestyle = 'iso, dmy'
```

### 8.2 - Copiar o `pg_hba.conf` do banco antigo para o novo:
```bash
cp /opt/PostgreSQL/9.4/data/pg_hba.conf /opt/PostgreSQL/12.20/
```

### 8.3 - Iniciar o novo cluster:
```bash
/usr/pgsql-12/bin/pg_ctl start -D /opt/PostgreSQL/12.20
```

## 9 - Verificar passos pós-migração

### 9.1 - Rodar o script de `analyze`:
```bash
nohup ./analyze_new_cluster.sh > ./analyze_new_cluster.out &
```

### 9.2 - Rodar o script de atualização de extensões, se necessário:
```bash
psql> ALTER EXTENSION "adminpack" UPDATE;
```

## 10 - Reiniciar o PostgreSQL
```bash
pg_ctl -D /usr/pgsql-12/data/ -l logfile stop
pg_ctl -D /usr/pgsql-12/data/ -l logfile start
```

## 11 - Executar o REINDEX em tabelas específicas

### 11.1 - Criar o script `reindex_tabs.sql` com o conteúdo:
```sql
REINDEX table concurrently dicomimages;
REINDEX table concurrently statusseries;
REINDEX table concurrently dicomstudies;
```

### 11.2 - Executar o script:
```bash
nohup psql -d pacs_aurora -f reindex_tabs.sql > reindex_tabs.out &
```
```

Esse arquivo está pronto para ser colado no seu repositório GitHub.

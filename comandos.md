# Comandos executados — Incidente A

## 1. Verificação do ambiente

Verificação da versão do Docker:

```cmd
docker --version
docker compose version
```

Verificação dos containers:

```cmd
docker compose ps
```

## 2. Criação da tabela e preparação dos dados

Criação da tabela clientes:

```sql
CREATE TABLE clientes (
    id serial PRIMARY KEY,
    nome text,
    email text
);
```

Inserção dos 1.000 clientes iniciais:

```sql
INSERT INTO clientes (nome, email)
SELECT 'cliente ' || g, 'c' || g || '@loja.com'
FROM generate_series(1, 1000) g;
```

Verificação da quantidade de registros:

```cmd
docker compose exec primary psql -U postgres -c "SELECT count(*) FROM clientes;"
```

## 3. Criação do backup

Comando utilizado:

```cmd
docker compose exec primary pg_dump -U postgres -t clientes -f /tmp/backup_clientes.sql postgres
```

Verificação do arquivo de backup:

```cmd
docker compose exec primary ls -lh /tmp/backup_clientes.sql
```

O arquivo foi armazenado dentro do container primary, no caminho `/tmp/backup_clientes.sql`.

## 4. Inserção de novos registros após o backup

Inserção de 50 clientes:

```cmd
docker compose exec primary psql -U postgres -c "INSERT INTO clientes (nome, email) SELECT 'novo ' || g, 'n' || g || '@loja.com' FROM generate_series(1, 50) g;"
```

Verificação:

```cmd
docker compose exec primary psql -U postgres -c "SELECT count(*) FROM clientes;"
```

Resultado: 1.050 registros.

## 5. Simulação do incidente

Registro do horário T0:

```cmd
powershell -NoProfile -Command "Get-Date -Format 'HH:mm:ss.fff'"
```

T0 registrado: 11:36:28.554.

Simulação da exclusão indevida:

```cmd
docker compose exec primary psql -U postgres -c "DELETE FROM clientes;"
```

Resultado: `DELETE 1050`.

Consulta à réplica:

```cmd
docker compose exec replica psql -U postgres -c "SELECT count(*) FROM clientes;"
```

Resultado: 0 registros.

## 6. Restauração do backup

Remoção da tabela vazia:

```cmd
docker compose exec primary psql -U postgres -c "DROP TABLE clientes;"
```

Restauração:

```cmd
docker compose exec primary psql -U postgres -f /tmp/backup_clientes.sql
```

Resultado: `COPY 1000`.

Verificação dos registros recuperados:

```cmd
docker compose exec primary psql -U postgres -c "SELECT count(*) FROM clientes;"
```

Resultado: 1.000 registros.

Registro do horário T1:

```cmd
powershell -NoProfile -Command "Get-Date -Format 'HH:mm:ss.fff'"
```

T1 registrado: 11:46:13.685.

## 7. Criação do usuário com permissões limitadas

Criação do usuário da aplicação:

```sql
CREATE ROLE app LOGIN PASSWORD 'app';
```

Concessão das permissões de consulta, inserção e atualização:

```sql
GRANT SELECT, INSERT, UPDATE ON clientes TO app;
```

Permissão de utilização da sequência:

```sql
GRANT USAGE ON SEQUENCE clientes_id_seq TO app;
```

## 8. Testes de permissões do usuário app

Consulta dos registros:

```cmd
docker compose exec -e PGPASSWORD=app primary psql -h 127.0.0.1 -U app -d postgres -c "SELECT count(*) FROM clientes;"
```

Resultado: consulta permitida, 1.000 registros.

Teste de inserção:

```cmd
docker compose exec -e PGPASSWORD=app primary psql -h 127.0.0.1 -U app -d postgres -c "INSERT INTO clientes (nome, email) VALUES ('teste app', 'app@loja.com');"
```

Resultado: `INSERT 0 1`.

Teste de exclusão de registros:

```cmd
docker compose exec -e PGPASSWORD=app primary psql -h 127.0.0.1 -U app -d postgres -c "DELETE FROM clientes;"
```

Resultado esperado e obtido:

```text
ERROR: permission denied for table clientes
```

Teste de exclusão da tabela:

```cmd
docker compose exec -e PGPASSWORD=app primary psql -h 127.0.0.1 -U app -d postgres -c "DROP TABLE clientes;"
```

Resultado esperado e obtido:

```text
ERROR: must be owner of table clientes
```

## 9. Verificação final

Consulta final da quantidade de clientes:

```cmd
docker compose exec primary psql -U postgres -c "SELECT count(*) FROM clientes;"
```

Resultado: 1.001 registros.

A quantidade final corresponde aos 1.000 registros recuperados do backup mais o registro inserido pelo usuário app.

---

## 10. Resumo dos resultados

- Registros antes do incidente: 1.050.
- Registros excluídos no incidente: 1.050.
- Registros recuperados pelo backup: 1.000.
- Registros perdidos: 50.
- Registro inserido pelo usuário app após a recuperação: 1.
- Total final: 1.001 registros.
- T0: 11:36:28.554.
- T1: 11:46:13.685.
- RTO: 9 minutos, 45 segundos e 131 milissegundos.
- RPO observado: 50 registros perdidos.

**Observação:** os comandos foram executados no ambiente Docker com PostgreSQL 16, utilizando os serviços primary e replica.
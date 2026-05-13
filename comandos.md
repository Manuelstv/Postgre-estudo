# Guia de Comandos PostgreSQL

Referência rápida dos comandos mais importantes do PostgreSQL, organizados por categoria.

---

## Sumário

1. [Banco de Dados e Tabelas (DDL)](#1-banco-de-dados-e-tabelas-ddl)
2. [Manipulação de Dados (CRUD)](#2-manipulação-de-dados-crud)
3. [Consultas (SELECT)](#3-consultas-select)
4. [Joins](#4-joins)
5. [Administração](#5-administração)
6. [Comandos do psql (terminal)](#6-comandos-do-psql-terminal)

---

## 1. Banco de Dados e Tabelas (DDL)

### Criar banco de dados

```sql
CREATE DATABASE meu_banco;
```

### Criar tabela

```sql
CREATE TABLE usuarios (
  id        SERIAL PRIMARY KEY,
  nome      VARCHAR(100) NOT NULL,
  email     VARCHAR(150) UNIQUE NOT NULL,
  criado_em TIMESTAMP DEFAULT NOW()
);
```

### Alterar tabela

```sql
-- Adicionar coluna
ALTER TABLE usuarios ADD COLUMN idade INT;

-- Remover coluna
ALTER TABLE usuarios DROP COLUMN idade;

-- Renomear coluna
ALTER TABLE usuarios RENAME COLUMN nome TO nome_completo;

-- Renomear tabela
ALTER TABLE usuarios RENAME TO clientes;
```

### Apagar tabela ou banco

```sql
DROP TABLE usuarios;
DROP DATABASE meu_banco;
```

> ⚠️ **Atenção:** sem WHERE, apaga tudo. Não tem volta.

### Tipos de dados mais comuns

| Tipo | Uso |
|---|---|
| `SERIAL` | Inteiro auto-incremento (ideal para IDs) |
| `INT` / `BIGINT` | Números inteiros |
| `VARCHAR(n)` | Texto até `n` caracteres |
| `TEXT` | Texto sem limite |
| `BOOLEAN` | true/false |
| `DATE` | Apenas data |
| `TIMESTAMP` | Data e hora |
| `NUMERIC(p,s)` | Decimais precisos (valores monetários) |
| `JSON` / `JSONB` | Dados em formato JSON |

---

## 2. Manipulação de Dados (CRUD)

### Inserir (CREATE)

```sql
-- Um registro
INSERT INTO usuarios (nome, email)
VALUES ('Ana Silva', 'ana@email.com');

-- Vários de uma vez
INSERT INTO usuarios (nome, email) VALUES
  ('João', 'joao@email.com'),
  ('Maria', 'maria@email.com');

-- Retornar o ID gerado
INSERT INTO usuarios (nome, email)
VALUES ('Pedro', 'pedro@email.com')
RETURNING id;
```

### Atualizar (UPDATE)

```sql
UPDATE usuarios
SET nome = 'Ana Costa'
WHERE id = 1;
```

> ⚠️ Sempre use `WHERE` no UPDATE, senão atualiza TODOS os registros.

### Deletar (DELETE)

```sql
DELETE FROM usuarios
WHERE id = 1;

-- Apagar todos os registros (mantém estrutura da tabela)
TRUNCATE TABLE usuarios;
```

### Upsert (inserir ou atualizar)

Exclusivo do Postgres: insere se não existe, atualiza se já existe.

```sql
INSERT INTO usuarios (email, nome)
VALUES ('ana@email.com', 'Ana Nova')
ON CONFLICT (email)
DO UPDATE SET nome = EXCLUDED.nome;
```

---

## 3. Consultas (SELECT)

### Básico

```sql
-- Tudo
SELECT * FROM usuarios;

-- Colunas específicas
SELECT nome, email FROM usuarios;

-- Com filtro
SELECT * FROM usuarios WHERE nome = 'Ana';

-- Com ordenação e limite
SELECT * FROM usuarios
ORDER BY criado_em DESC
LIMIT 10;
```

### Filtros avançados

```sql
-- LIKE (busca parcial, % = qualquer texto)
SELECT * FROM usuarios WHERE nome LIKE 'An%';

-- ILIKE (LIKE sem diferenciar maiúsculas/minúsculas)
SELECT * FROM usuarios WHERE nome ILIKE 'an%';

-- IN (lista de valores)
SELECT * FROM usuarios WHERE id IN (1, 2, 3);

-- BETWEEN (intervalo)
SELECT * FROM pedidos
WHERE criado_em BETWEEN '2024-01-01' AND '2024-12-31';

-- IS NULL / IS NOT NULL
SELECT * FROM usuarios WHERE telefone IS NULL;

-- Combinar com AND / OR
SELECT * FROM usuarios
WHERE idade > 18 AND (cidade = 'SP' OR cidade = 'RJ');
```

### Agregações

```sql
SELECT
  COUNT(*)   AS total,
  AVG(preco) AS preco_medio,
  SUM(preco) AS total_vendas,
  MAX(preco) AS maior,
  MIN(preco) AS menor
FROM pedidos;
```

### GROUP BY e HAVING

```sql
-- Agrupar e contar
SELECT cidade, COUNT(*) AS qtd
FROM usuarios
GROUP BY cidade
HAVING COUNT(*) > 5
ORDER BY qtd DESC;
```

> 💡 `WHERE` filtra antes do agrupamento. `HAVING` filtra depois.

### DISTINCT (remover duplicados)

```sql
SELECT DISTINCT cidade FROM usuarios;
```

---

## 4. Joins

### INNER JOIN

Retorna só os registros com correspondência nas duas tabelas.

```sql
SELECT u.nome, p.produto, p.preco
FROM usuarios u
INNER JOIN pedidos p ON u.id = p.usuario_id;
```

### LEFT JOIN

Retorna todos da tabela da esquerda, mesmo sem correspondência (preenche com NULL).

```sql
SELECT u.nome, p.produto
FROM usuarios u
LEFT JOIN pedidos p ON u.id = p.usuario_id;
```

### RIGHT JOIN

Espelho do LEFT — todos da direita.

```sql
SELECT u.nome, p.produto
FROM usuarios u
RIGHT JOIN pedidos p ON u.id = p.usuario_id;
```

### FULL OUTER JOIN

Todos os registros de ambas as tabelas.

```sql
SELECT u.nome, p.produto
FROM usuarios u
FULL OUTER JOIN pedidos p ON u.id = p.usuario_id;
```

### Resumo visual

| Tipo de JOIN | O que retorna |
|---|---|
| `INNER` | Só o que tem nos dois lados |
| `LEFT` | Tudo da esquerda + correspondências |
| `RIGHT` | Tudo da direita + correspondências |
| `FULL` | Tudo de ambos os lados |

---

## 5. Administração

### Usuários e permissões

```sql
-- Criar usuário
CREATE USER dev_user WITH PASSWORD 'senha123';

-- Dar permissão de leitura em todas as tabelas
GRANT SELECT ON ALL TABLES IN SCHEMA public TO dev_user;

-- Dar todas as permissões em um banco
GRANT ALL PRIVILEGES ON DATABASE meu_banco TO dev_user;

-- Revogar permissão
REVOKE SELECT ON usuarios FROM dev_user;

-- Apagar usuário
DROP USER dev_user;
```

### Índices

Aceleram queries em colunas muito consultadas.

```sql
-- Criar índice
CREATE INDEX idx_usuarios_email ON usuarios(email);

-- Índice único
CREATE UNIQUE INDEX idx_email_unico ON usuarios(email);

-- Listar índices de uma tabela
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'usuarios';

-- Remover índice
DROP INDEX idx_usuarios_email;
```

> 💡 Índices aceleram leituras mas tornam escritas (INSERT/UPDATE) mais lentas. Use com cautela.

### Transações

Garantem que um conjunto de operações acontece tudo ou nada (atomicidade).

```sql
BEGIN;
  UPDATE contas SET saldo = saldo - 100 WHERE id = 1;
  UPDATE contas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;

-- Se algo der errado antes do COMMIT:
ROLLBACK;
```

### Backup e restore (via terminal)

```bash
# Backup
pg_dump -U postgres meu_banco > backup.sql

# Restore
psql -U postgres meu_banco < backup.sql
```

---

## 6. Comandos do psql (terminal)

### Conectar

```bash
psql -U postgres -d meu_banco
```

### Comandos dentro do psql

| Comando | O que faz |
|---|---|
| `\l` | Listar bancos de dados |
| `\c meu_banco` | Conectar a um banco |
| `\dt` | Listar tabelas |
| `\d usuarios` | Descrever estrutura de uma tabela |
| `\du` | Listar usuários/roles |
| `\df` | Listar funções |
| `\dn` | Listar schemas |
| `\timing` | Medir tempo das queries |
| `\e` | Abrir editor externo |
| `\i arquivo.sql` | Executar arquivo SQL |
| `\copy tabela TO 'saida.csv' CSV HEADER` | Exportar para CSV |
| `\?` | Ajuda dos comandos `\` |
| `\h` | Ajuda do SQL |
| `\q` | Sair |

---

## Dicas finais

- **Sempre teste UPDATE/DELETE com SELECT antes.** Roda primeiro `SELECT * FROM tabela WHERE ...` pra ver o que vai ser afetado.
- **Use transações** para operações críticas que envolvem várias tabelas.
- **Comente seu SQL** com `--` (linha) ou `/* ... */` (bloco).
- **`EXPLAIN ANALYZE`** antes de uma query mostra o plano de execução — essencial para otimização.
- **Backup antes de migrations grandes.** Sempre.

```sql
-- Exemplo de EXPLAIN
EXPLAIN ANALYZE
SELECT * FROM usuarios WHERE email = 'ana@email.com';
```

---

*Documento de referência rápida do PostgreSQL.*

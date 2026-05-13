# Guia de Comandos PostgreSQL (com pgAdmin)

Referência rápida dos comandos mais importantes do PostgreSQL, focada em quem usa o **pgAdmin** como cliente.

---

## Sumário

1. [Como rodar comandos no pgAdmin](#1-como-rodar-comandos-no-pgadmin)
2. [⭐ COMANDOS ESSENCIAIS](#2--comandos-essenciais)
   - [2.1 SELECT](#21--select)
   - [2.2 JOIN](#22--join)
   - [2.3 VIEW](#23--view)
   - [2.4 MATERIALIZED VIEW](#24--materialized-view)
   - [2.5 FUNCTION](#25--function)
   - [2.6 PROCEDURE](#26--procedure)
   - [2.7 TRIGGER](#27--trigger)
   - [2.8 CRON (pg_cron)](#28--cron-pg_cron)
   - [2.9 TRANSACTIONS](#29--transactions)
3. [Banco de Dados e Tabelas (DDL)](#3-banco-de-dados-e-tabelas-ddl)
4. [Manipulação de Dados (CRUD)](#4-manipulação-de-dados-crud)
5. [Administração](#5-administração)
6. [Dicas do pgAdmin](#6-dicas-do-pgadmin)

---

## 1. Como rodar comandos no pgAdmin

1. No painel esquerdo, expanda `Servers → seu_servidor → Databases`
2. **Clique com botão direito no banco** que você quer usar
3. Escolha **`Query Tool`** (ou pressione `Alt + Shift + Q`)
4. Cole/escreva o SQL no editor
5. Aperte **F5** ou clique no botão ▶ (Play) para executar

> 💡 O nome da aba mostra `banco/usuário@servidor` — sempre confira se está no banco certo antes de rodar.

---

# 2. ⭐ COMANDOS ESSENCIAIS

> Esta é a seção mais importante do guia. Os 9 comandos abaixo são o **coração do PostgreSQL** no dia a dia.

---

## 2.1 ⭐ SELECT

> **O comando mais usado de todos.** Lê dados das tabelas.

### Estrutura completa

```sql
SELECT  coluna1, coluna2, ...
FROM    tabela
WHERE   condição
GROUP BY coluna
HAVING  condição_de_grupo
ORDER BY coluna
LIMIT   n
OFFSET  n;
```

### Exemplos do básico ao avançado

```sql
-- 1) Tudo
SELECT * FROM usuarios;

-- 2) Colunas específicas com alias
SELECT nome AS nome_completo, email FROM usuarios;

-- 3) Filtros (WHERE)
SELECT * FROM usuarios WHERE idade >= 18 AND cidade = 'SP';

-- 4) Busca por texto
SELECT * FROM usuarios WHERE nome ILIKE '%silva%';

-- 5) Ordenação + paginação
SELECT * FROM usuarios
ORDER BY criado_em DESC
LIMIT 10 OFFSET 20;  -- pula 20, traz 10

-- 6) Agregações
SELECT
  cidade,
  COUNT(*)   AS qtd,
  AVG(idade) AS media_idade
FROM usuarios
GROUP BY cidade
HAVING COUNT(*) > 5
ORDER BY qtd DESC;

-- 7) Subquery
SELECT * FROM usuarios
WHERE id IN (SELECT usuario_id FROM pedidos WHERE preco > 1000);

-- 8) CTE (Common Table Expression) — query "nomeada"
WITH top_clientes AS (
  SELECT usuario_id, SUM(preco) AS total
  FROM pedidos
  GROUP BY usuario_id
)
SELECT u.nome, t.total
FROM top_clientes t
JOIN usuarios u ON u.id = t.usuario_id
ORDER BY t.total DESC
LIMIT 5;
```

> 💡 **Dica:** sempre rode `SELECT` antes de qualquer `UPDATE` ou `DELETE` para ver o que vai ser afetado.

---

## 2.2 ⭐ JOIN

> **Combina dados de duas ou mais tabelas.** Essencial em qualquer banco relacional.

### Tipos de JOIN

| Tipo | O que retorna |
|---|---|
| `INNER JOIN` | Só o que tem nos dois lados |
| `LEFT JOIN` | Tudo da esquerda + correspondências |
| `RIGHT JOIN` | Tudo da direita + correspondências |
| `FULL OUTER JOIN` | Tudo de ambos os lados |
| `CROSS JOIN` | Produto cartesiano (todas combinações) |

### Exemplos

```sql
-- INNER JOIN
SELECT u.nome, p.produto, p.preco
FROM usuarios u
INNER JOIN pedidos p ON u.id = p.usuario_id;

-- LEFT JOIN (traz usuários mesmo sem pedidos)
SELECT u.nome, p.produto
FROM usuarios u
LEFT JOIN pedidos p ON u.id = p.usuario_id;

-- Múltiplos JOINs
SELECT u.nome, p.produto, c.nome AS categoria
FROM usuarios u
INNER JOIN pedidos p   ON u.id = p.usuario_id
INNER JOIN categorias c ON p.categoria_id = c.id;

-- SELF JOIN (tabela com ela mesma) — útil para hierarquias
SELECT
  f.nome       AS funcionario,
  g.nome       AS gerente
FROM funcionarios f
LEFT JOIN funcionarios g ON f.gerente_id = g.id;
```

> ⚠️ **Cuidado:** sem condição `ON`, um JOIN vira CROSS JOIN e pode retornar milhões de linhas inesperadas.

---

## 2.3 ⭐ VIEW

> **Uma "tabela virtual"** baseada em uma query salva. Não armazena dados — toda vez que você consulta, a query roda de novo.

### Quando usar?

- Simplificar queries complexas (você consulta a view como se fosse uma tabela)
- Esconder colunas sensíveis de usuários específicos
- Padronizar relatórios

### Sintaxe

```sql
-- Criar view
CREATE VIEW usuarios_ativos AS
SELECT id, nome, email
FROM usuarios
WHERE ativo = true;

-- Usar a view (igual uma tabela)
SELECT * FROM usuarios_ativos WHERE nome ILIKE 'an%';

-- Atualizar definição
CREATE OR REPLACE VIEW usuarios_ativos AS
SELECT id, nome, email, criado_em
FROM usuarios
WHERE ativo = true AND deletado_em IS NULL;

-- Remover
DROP VIEW usuarios_ativos;
```

### Exemplo prático: view de relatório

```sql
CREATE VIEW vendas_por_mes AS
SELECT
  DATE_TRUNC('month', criado_em) AS mes,
  COUNT(*)                       AS total_pedidos,
  SUM(preco)                     AS receita
FROM pedidos
GROUP BY DATE_TRUNC('month', criado_em);

-- Agora qualquer um consulta o relatório com:
SELECT * FROM vendas_por_mes ORDER BY mes DESC;
```

> 💡 Views são **leves** — não ocupam espaço, só guardam a query.

---

## 2.4 ⭐ MATERIALIZED VIEW

> **Uma view que cacheia o resultado em disco.** Não executa a query toda hora — fica salva como uma tabela.

### View vs Materialized View

| | VIEW | MATERIALIZED VIEW |
|---|---|---|
| **Armazena dados?** | Não | Sim (em disco) |
| **Velocidade de leitura** | Mais lenta (roda query) | Muito rápida |
| **Sempre atualizada?** | Sim | Não — precisa atualizar manualmente |
| **Uso ideal** | Dados que mudam sempre | Dados pesados que mudam pouco |

### Sintaxe

```sql
-- Criar
CREATE MATERIALIZED VIEW relatorio_vendas AS
SELECT
  u.cidade,
  COUNT(p.id)  AS total_pedidos,
  SUM(p.preco) AS receita
FROM usuarios u
LEFT JOIN pedidos p ON u.id = p.usuario_id
GROUP BY u.cidade;

-- Consultar (super rápido)
SELECT * FROM relatorio_vendas;

-- Atualizar os dados (rodar a query de novo e regravar)
REFRESH MATERIALIZED VIEW relatorio_vendas;

-- Atualizar sem travar leituras (precisa de índice único)
REFRESH MATERIALIZED VIEW CONCURRENTLY relatorio_vendas;

-- Remover
DROP MATERIALIZED VIEW relatorio_vendas;
```

> 💡 **Quando usar:** dashboards e relatórios pesados que rodam várias vezes mas só precisam estar atualizados de hora em hora (combine com `CRON`, veja seção 2.8).

---

## 2.5 ⭐ FUNCTION

> **Um bloco de código reutilizável** que recebe parâmetros e retorna um valor (ou tabela).

### Sintaxe básica

```sql
CREATE OR REPLACE FUNCTION nome_funcao(parametros)
RETURNS tipo_retorno
LANGUAGE plpgsql
AS $$
BEGIN
  -- corpo da função
  RETURN algum_valor;
END;
$$;
```

### Exemplos

```sql
-- 1) Função simples: calcular idade
CREATE OR REPLACE FUNCTION calcular_idade(data_nasc DATE)
RETURNS INT
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN EXTRACT(YEAR FROM AGE(data_nasc));
END;
$$;

-- Usar:
SELECT nome, calcular_idade(data_nascimento) FROM usuarios;

-- 2) Função que retorna várias colunas (uma "linha")
CREATE OR REPLACE FUNCTION buscar_usuario(p_id INT)
RETURNS TABLE(nome VARCHAR, email VARCHAR)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT u.nome, u.email
  FROM usuarios u
  WHERE u.id = p_id;
END;
$$;

-- Usar:
SELECT * FROM buscar_usuario(1);

-- 3) Função com lógica condicional
CREATE OR REPLACE FUNCTION classificar_cliente(total NUMERIC)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
  IF total >= 10000 THEN RETURN 'VIP';
  ELSIF total >= 1000 THEN RETURN 'Premium';
  ELSE RETURN 'Regular';
  END IF;
END;
$$;

-- Remover função
DROP FUNCTION calcular_idade(DATE);
```

> 💡 **Functions** podem ser usadas dentro de SELECT, WHERE, em outras functions, triggers, etc.

---

## 2.6 ⭐ PROCEDURE

> **Similar à FUNCTION, mas para executar ações** (não retorna valor diretamente). Suporta `COMMIT` e `ROLLBACK` dentro do código — uma function não pode.

### Function vs Procedure

| | FUNCTION | PROCEDURE |
|---|---|---|
| **Retorna valor?** | Sim | Não (usa parâmetros OUT) |
| **Pode fazer COMMIT?** | Não | Sim |
| **Como chamar?** | `SELECT funcao()` | `CALL procedure()` |
| **Uso ideal** | Cálculos, transformações | Operações complexas, batches |

### Sintaxe

```sql
CREATE OR REPLACE PROCEDURE nome_procedure(parametros)
LANGUAGE plpgsql
AS $$
BEGIN
  -- corpo
END;
$$;

-- Chamar:
CALL nome_procedure(parametros);
```

### Exemplo prático: transferência bancária

```sql
CREATE OR REPLACE PROCEDURE transferir(
  origem  INT,
  destino INT,
  valor   NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
  UPDATE contas SET saldo = saldo - valor WHERE id = origem;
  UPDATE contas SET saldo = saldo + valor WHERE id = destino;
  COMMIT;  -- procedures podem dar COMMIT, functions não
END;
$$;

-- Executar:
CALL transferir(1, 2, 100.00);

-- Remover:
DROP PROCEDURE transferir(INT, INT, NUMERIC);
```

> 💡 Procedures foram adicionadas no PostgreSQL **11+** — antes disso, tudo era function.

---

## 2.7 ⭐ TRIGGER

> **Código que dispara automaticamente** quando algo acontece em uma tabela (INSERT, UPDATE, DELETE).

### Quando usar?

- Auditoria (registrar quem mudou o quê)
- Manter campos calculados atualizados
- Validações complexas
- Sincronizar tabelas relacionadas

### Anatomia de um trigger

1. Criar uma **function** que será chamada
2. Criar o **trigger** ligando a function a um evento da tabela

### Exemplo: atualizar campo `atualizado_em` automaticamente

```sql
-- Passo 1: criar a função
CREATE OR REPLACE FUNCTION set_atualizado_em()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.atualizado_em = NOW();
  RETURN NEW;
END;
$$;

-- Passo 2: criar o trigger
CREATE TRIGGER trg_atualizar_usuario
BEFORE UPDATE ON usuarios
FOR EACH ROW
EXECUTE FUNCTION set_atualizado_em();
```

Agora, toda vez que rodar `UPDATE usuarios SET ...`, o campo `atualizado_em` é preenchido sozinho.

### Exemplo: auditoria de exclusões

```sql
CREATE TABLE log_exclusoes (
  id SERIAL PRIMARY KEY,
  tabela TEXT,
  registro_id INT,
  apagado_em TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION logar_exclusao()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
  INSERT INTO log_exclusoes (tabela, registro_id)
  VALUES (TG_TABLE_NAME, OLD.id);
  RETURN OLD;
END;
$$;

CREATE TRIGGER trg_log_delete_usuarios
AFTER DELETE ON usuarios
FOR EACH ROW
EXECUTE FUNCTION logar_exclusao();
```

### Tipos de trigger

| Quando dispara | Palavras-chave |
|---|---|
| Antes da operação | `BEFORE INSERT / UPDATE / DELETE` |
| Depois da operação | `AFTER INSERT / UPDATE / DELETE` |
| Em vez da operação | `INSTEAD OF` (só para views) |
| Por linha | `FOR EACH ROW` |
| Por comando | `FOR EACH STATEMENT` |

```sql
-- Remover trigger
DROP TRIGGER trg_log_delete_usuarios ON usuarios;
```

> ⚠️ Triggers facilitam muito, mas podem deixar o banco lento e o comportamento "mágico" — use com moderação e documente bem.

---

## 2.8 ⭐ CRON (pg_cron)

> **Agenda execução de SQL** em horários específicos, direto no Postgres. Útil para refresh de materialized views, limpezas, relatórios.

> ⚠️ `pg_cron` é uma **extensão** — precisa ser instalada e habilitada. Vem por padrão em serviços gerenciados (AWS RDS, Supabase, Neon), mas em instalações locais pode precisar de configuração extra.

### Habilitar a extensão

```sql
CREATE EXTENSION pg_cron;
```

### Sintaxe básica

```sql
SELECT cron.schedule('nome_do_job', 'cron_expression', 'comando SQL');
```

### Exemplos

```sql
-- 1) Atualizar uma materialized view toda hora
SELECT cron.schedule(
  'refresh-relatorio',
  '0 * * * *',                              -- toda hora cheia
  'REFRESH MATERIALIZED VIEW relatorio_vendas'
);

-- 2) Limpar logs antigos toda madrugada (3h da manhã)
SELECT cron.schedule(
  'limpar-logs',
  '0 3 * * *',
  $$DELETE FROM logs WHERE criado_em < NOW() - INTERVAL '30 days'$$
);

-- 3) Executar uma procedure de fechamento mensal (todo dia 1, 4h)
SELECT cron.schedule(
  'fechamento-mensal',
  '0 4 1 * *',
  'CALL fechar_mes()'
);
```

### Formato cron (recap)

```
*  *  *  *  *
│  │  │  │  │
│  │  │  │  └── dia da semana (0-6, 0 = domingo)
│  │  │  └───── mês (1-12)
│  │  └──────── dia do mês (1-31)
│  └─────────── hora (0-23)
└────────────── minuto (0-59)
```

### Gerenciar jobs

```sql
-- Listar todos os jobs agendados
SELECT * FROM cron.job;

-- Ver histórico de execuções
SELECT * FROM cron.job_run_details
ORDER BY start_time DESC LIMIT 10;

-- Remover um job
SELECT cron.unschedule('refresh-relatorio');
```

> 💡 **Combo poderoso:** Materialized View + pg_cron = dashboards rápidos sem complicação.

---

## 2.9 ⭐ TRANSACTIONS

> **Garante que um conjunto de operações acontece "tudo ou nada".** Princípio fundamental: **atomicidade**.

### Sintaxe básica

```sql
BEGIN;
  -- várias operações
  UPDATE contas SET saldo = saldo - 100 WHERE id = 1;
  UPDATE contas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;  -- confirma tudo

-- Se algo der errado antes do COMMIT:
ROLLBACK;  -- desfaz tudo
```

### Exemplo com tratamento de erro

```sql
BEGIN;

  UPDATE contas SET saldo = saldo - 500 WHERE id = 1;

  -- Verificar se saldo ficou negativo
  -- (em código real, faria isso na aplicação)
  SELECT saldo FROM contas WHERE id = 1;

  -- Se OK:
  UPDATE contas SET saldo = saldo + 500 WHERE id = 2;
  COMMIT;

  -- Se deu ruim, em vez de COMMIT:
  -- ROLLBACK;
```

### SAVEPOINT (rollback parcial)

```sql
BEGIN;

  INSERT INTO usuarios (nome) VALUES ('Ana');
  SAVEPOINT depois_ana;

  INSERT INTO usuarios (nome) VALUES ('João');
  SAVEPOINT depois_joao;

  INSERT INTO usuarios (nome) VALUES ('Erro!');

  -- Volta só até depois do João, mantém Ana e João
  ROLLBACK TO SAVEPOINT depois_joao;

COMMIT;
-- Resultado final: Ana e João salvos, "Erro!" descartado
```

### Níveis de isolamento

Controlam como transações simultâneas se enxergam.

```sql
-- Definir antes de iniciar
BEGIN ISOLATION LEVEL SERIALIZABLE;
  -- ...
COMMIT;
```

| Nível | Descrição |
|---|---|
| `READ UNCOMMITTED` | (Postgres trata como READ COMMITTED) |
| `READ COMMITTED` | **Padrão** — vê dados já commitados |
| `REPEATABLE READ` | Vê snapshot consistente durante toda a transação |
| `SERIALIZABLE` | Como se transações rodassem em sequência (mais seguro, mais lento) |

### As propriedades ACID

| Letra | Significa | O que garante |
|---|---|---|
| **A** | Atomicidade | Tudo ou nada |
| **C** | Consistência | Banco sempre em estado válido |
| **I** | Isolamento | Transações não interferem entre si |
| **D** | Durabilidade | Dado commitado não some |

> ⚠️ **Boa prática:** sempre use transações em operações que envolvem várias tabelas ou várias linhas relacionadas.

---

# 3. Banco de Dados e Tabelas (DDL)

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
ALTER TABLE usuarios ADD COLUMN idade INT;
ALTER TABLE usuarios DROP COLUMN idade;
ALTER TABLE usuarios RENAME COLUMN nome TO nome_completo;
ALTER TABLE usuarios RENAME TO clientes;
```

### Apagar

```sql
DROP TABLE usuarios;
DROP DATABASE meu_banco;
```

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

# 4. Manipulação de Dados (CRUD)

### Inserir

```sql
INSERT INTO usuarios (nome, email) VALUES ('Ana', 'ana@email.com');

INSERT INTO usuarios (nome, email) VALUES
  ('João', 'joao@email.com'),
  ('Maria', 'maria@email.com');

-- Retornar ID gerado
INSERT INTO usuarios (nome) VALUES ('Pedro') RETURNING id;
```

### Atualizar

```sql
UPDATE usuarios SET nome = 'Ana Costa' WHERE id = 1;
```

### Deletar

```sql
DELETE FROM usuarios WHERE id = 1;
TRUNCATE TABLE usuarios;  -- apaga tudo, mantém estrutura
```

### Upsert (inserir ou atualizar)

```sql
INSERT INTO usuarios (email, nome)
VALUES ('ana@email.com', 'Ana Nova')
ON CONFLICT (email)
DO UPDATE SET nome = EXCLUDED.nome;
```

---

# 5. Administração

### Usuários e permissões

```sql
CREATE USER dev_user WITH PASSWORD 'senha123';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO dev_user;
GRANT ALL PRIVILEGES ON DATABASE meu_banco TO dev_user;
REVOKE SELECT ON usuarios FROM dev_user;
DROP USER dev_user;
```

### Índices

```sql
CREATE INDEX idx_usuarios_email ON usuarios(email);
CREATE UNIQUE INDEX idx_email_unico ON usuarios(email);
DROP INDEX idx_usuarios_email;

-- Listar índices
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'usuarios';
```

> 💡 Índices aceleram leituras mas tornam escritas mais lentas. Use com critério.

---

# 6. Dicas do pgAdmin

### Atalhos do Query Tool

| Atalho | O que faz |
|---|---|
| `F5` | Executar a query |
| `F7` | Executar com `EXPLAIN` |
| `F8` | Executar com `EXPLAIN ANALYZE` |
| `Ctrl + Espaço` | Autocompletar |
| `Ctrl + /` | Comentar/descomentar linha |
| `Alt + Shift + Q` | Abrir novo Query Tool |
| `Ctrl + S` | Salvar a query como `.sql` |

### Navegação visual

| Para ver | Onde clicar |
|---|---|
| Bancos | Expandir `Databases` |
| Tabelas | `Schemas → public → Tables` |
| Views | `Schemas → public → Views` |
| Materialized Views | `Schemas → public → Materialized Views` |
| Functions | `Schemas → public → Functions` |
| Procedures | `Schemas → public → Procedures` |
| Triggers | Dentro da tabela → `Triggers` |
| Estrutura de uma tabela | Botão direito → `Properties` |
| Dados de uma tabela | Botão direito → `View/Edit Data → All Rows` |
| Atualizar painel | Botão direito → `Refresh` |

### Backup e restore

- **Backup**: botão direito no banco → `Backup...`
- **Restore**: botão direito no banco → `Restore...`

---

## Dicas gerais finais

- **Sempre teste UPDATE/DELETE com SELECT antes.**
- **Use transações** para operações críticas com várias tabelas.
- **Comente seu SQL** com `--` (linha) ou `/* ... */` (bloco).
- **Faça backup antes de migrations grandes.**
- **pgAdmin não atualiza sozinho** — use `Refresh` após mudanças via SQL.
- **`EXPLAIN ANALYZE`** antes de uma query mostra o plano de execução — essencial para otimização.

```sql
EXPLAIN ANALYZE
SELECT * FROM usuarios WHERE email = 'ana@email.com';
```

---

*Documento de referência rápida do PostgreSQL para usuários do pgAdmin.*

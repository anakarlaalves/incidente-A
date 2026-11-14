# Incidente A — Exclusão acidental em banco de dados replicado

## 1. Identificação

**Disciplina:** DBaaS — Alta Disponibilidade e Réplicas  
**Cenário:** A — Exclusão indevida de dados em banco replicado  
**Tecnologia:** Docker, Docker Compose e PostgreSQL 16  
**Ambiente:** Banco principal (`primary`) e banco réplica (`replica`).

## 2. Resumo do incidente

O cenário simula um incidente de segurança em que uma senha de acesso ao serviço de banco de dados foi exposta em um repositório público. Um invasor, utilizando uma conta com privilégios excessivos, executou uma operação de exclusão na tabela `clientes`.

A operação removeu todos os registros do banco principal. Como o ambiente possuía replicação entre os bancos, a exclusão também foi propagada para a réplica.

O incidente demonstrou que a replicação não substitui uma estratégia de backup, pois alterações indevidas também podem ser replicadas.

## 3. Preparação do ambiente

Foi criado um ambiente com dois serviços PostgreSQL utilizando Docker Compose:

- **Primary:** banco de dados principal, responsável pelas operações de escrita.
- **Replica:** banco secundário, que recebe as alterações do banco principal por replicação.

Foi criada a tabela `clientes`, contendo os campos `id`, `nome` e `email`.

Inicialmente, foram inseridos 1.000 clientes. Em seguida, foi realizado um backup da tabela.

Após o backup, foram inseridos mais 50 clientes, totalizando 1.050 registros antes do incidente.

## 4. Linha do tempo do incidente

| Etapa | Descrição | Resultado |
|---|---|---|
| Preparação | Inserção dos clientes iniciais | 1.000 registros |
| Backup | Cópia da tabela clientes | 1.000 registros salvos |
| Novos cadastros | Inserção de mais 50 clientes | 1.050 registros |
| T0 — 11:36:28.554 | Início da simulação do ataque | Exclusão de 1.050 registros |
| Verificação da réplica | Consulta após o ataque | 0 registros |
| Restauração | Recuperação pelo backup | 1.000 registros |
| T1 — 11:46:13.685 | Recuperação validada | 1.000 registros |
| Teste de segurança | Inserção pelo usuário app | 1 registro inserido |
| Conferência final | Consulta ao banco principal | 1.001 registros |

## 5. Cálculo do RTO e RPO

### RTO — Recovery Time Objective

O RTO representa o tempo necessário para recuperar o serviço após o incidente.

- T0: 11:36:28.554
- T1: 11:46:13.685

**Cálculo:**

RTO = T1 − T0

RTO = 11:46:13.685 − 11:36:28.554

**RTO = 9 minutos, 45 segundos e 131 milissegundos.**

Esse foi o tempo observado entre o registro do início da simulação e a validação da recuperação dos dados.

### RPO — Recovery Point Objective

O RPO representa o ponto de recuperação dos dados e a quantidade ou intervalo de informações que pode ser perdido em um incidente.

Neste experimento:

- Registros existentes no momento do ataque: 1.050.
- Registros salvos no backup: 1.000.
- Registros perdidos: 50.

**Cálculo:**

RPO observado em registros = 1.050 − 1.000

**Resultado: 50 registros perdidos.**

A perda ocorreu porque os 50 clientes foram cadastrados depois da realização do backup e não estavam incluídos no arquivo de recuperação.

Observação: os 50 registros representam a perda efetivamente observada no exercício. Não foi calculado o intervalo exato de tempo entre o backup e o incidente.

## 6. Causa raiz

A causa raiz do incidente foi a exposição de uma credencial de acesso ao banco de dados, associada à utilização de uma conta com permissões excessivas.

O usuário utilizado na simulação do comando destrutivo foi o administrador `postgres`, para reproduzir a exclusão indevida.

A operação executada foi:

```sql
DELETE FROM clientes;
```

O comando removeu todos os registros da tabela.

Como a replicação estava ativa, a operação também foi aplicada à réplica, que passou a apresentar zero registros.

O problema evidenciou duas falhas importantes:

1. Credenciais expostas podem permitir acesso indevido ao banco.
2. Usuários com privilégios administrativos ou excessivos podem executar operações destrutivas.

## 7. Correção e validação

Para recuperar os dados, foi utilizado o backup realizado antes da inserção dos últimos 50 clientes.

Primeiro, a tabela vazia foi removida:

```sql
DROP TABLE clientes;
```

Em seguida, foi realizada a restauração:

```cmd
docker compose exec primary psql -U postgres -f /tmp/backup_clientes.sql
```

O resultado apresentou:

```text
COPY 1000
```

A consulta posterior confirmou a recuperação de 1.000 registros.

A réplica também voltou a apresentar os 1.000 registros após a restauração ser replicada.

Posteriormente, foi inserido mais um cliente utilizando o usuário `app`, totalizando 1.001 registros no banco principal.

## 8. Comparação com serviço gerenciado — Amazon RDS

Como exemplo de serviço gerenciado, foi considerado o Amazon RDS para PostgreSQL.

O Amazon RDS oferece recursos de backup automatizado e recuperação para um ponto específico no tempo, conhecida como PITR (Point-in-Time Recovery).

Com esse recurso, é possível utilizar backups e logs de transações para recuperar o banco até um momento anterior a uma exclusão indevida, respeitando o período de retenção configurado.

Dessa forma, em vez de recuperar somente o último backup completo, seria possível selecionar um instante anterior ao comando destrutivo e recuperar também as transações realizadas depois do backup.

Outro recurso oferecido é o Multi-AZ, que mantém uma instância standby em outra zona de disponibilidade para melhorar a disponibilidade do serviço em caso de falhas de infraestrutura.

É importante diferenciar os recursos:

- **PITR:** permite recuperar os dados para um momento anterior ao incidente.
- **Multi-AZ:** melhora a disponibilidade e permite failover em caso de falhas da instância ou da zona de disponibilidade.

O Multi-AZ, isoladamente, não impede que uma exclusão lógica seja replicada. Para esse tipo de incidente, a proteção por backup e PITR é fundamental.

Fonte: documentação oficial da AWS — https://docs.aws.amazon.com/AmazonRDS/latest/gettingstartedguide/managing-backup-restore.html

## 9. Estratégias de prevenção

Para reduzir a possibilidade de novos incidentes, recomenda-se:

- Não armazenar senhas e credenciais em repositórios públicos.
- Utilizar variáveis de ambiente ou gerenciadores de segredos.
- Aplicar o princípio do menor privilégio aos usuários do banco.
- Evitar utilizar o usuário administrador nas operações da aplicação.
- Realizar backups automatizados e testar periodicamente a restauração.
- Manter logs de auditoria para identificar operações suspeitas.
- Monitorar acessos e operações de exclusão.
- Configurar retenção adequada dos backups e dos logs de transação.
- Utilizar recuperação para um ponto no tempo (PITR), quando disponível.

### Frequência de backup para RPO de 15 minutos

Para buscar um RPO máximo de 15 minutos utilizando backups periódicos, os backups devem ser realizados em intervalos de, no máximo, 15 minutos.

Uma estratégia mais segura seria realizar backups a cada 10 minutos, reduzindo o risco de ultrapassar o limite devido a atrasos operacionais.

Outra alternativa é utilizar backup contínuo dos logs de transação e PITR, permitindo recuperar o banco em um ponto específico dentro da janela de retenção.

## 10. Respostas às questões do cenário

**1. Por que a réplica não protegeu os dados?**

Porque a réplica recebe as alterações realizadas no banco principal, inclusive operações destrutivas. Como o comando DELETE foi executado no banco principal, a exclusão também foi propagada para a réplica.

**2. Qual recurso de um serviço gerenciado ajuda a reduzir o RPO?**

O recurso PITR (Point-in-Time Recovery), associado aos backups automatizados e à retenção dos logs de transação, permite recuperar o banco em um ponto anterior ao incidente.

**3. Qual frequência de backup permite obter RPO de 15 minutos?**

Backups realizados a cada 15 minutos, no máximo. Para maior segurança, recomenda-se uma frequência menor, como a cada 10 minutos, ou o uso de recuperação contínua por logs de transação.

## 11. Conclusão e aprendizados

O experimento demonstrou que a replicação, apesar de importante para disponibilidade e continuidade do serviço, não substitui backups independentes.

Também foi possível observar a importância de manter cópias de segurança atualizadas, testar os procedimentos de restauração e controlar as permissões de acesso ao banco de dados.

A criação do usuário `app` demonstrou a aplicação do princípio do menor privilégio. Esse usuário conseguiu consultar e inserir registros, mas teve suas tentativas de exclusão de dados e remoção da tabela bloqueadas pelo PostgreSQL.

Com isso, foi possível compreender na prática os conceitos de replicação, backup, restauração, RTO, RPO, controle de acesso e prevenção de incidentes em bancos de dados.

---

**Repositório:** incidente-A  
**Arquivos complementares:** `comandos.md` e pasta `evidencias/`.
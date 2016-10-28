
----BUILDING A CLUSTER-------
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS bdr;
CREATE DATABASE test;


psql -U postgres -d test
SELECT bdr.bdr_group_create(
     local_node_name := 'postgresmmm_bdr1_1',
     node_external_dsn := 'host=postgresmmm_bdr1_1 port=5432 dbname=test');

CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS bdr;

SELECT bdr.bdr_group_join(
  local_node_name := 'postgresmmm_bdr2_1',
  node_external_dsn := 'host=postgresmmm_bdr2_1 port=5432 dbname=test',
  join_using_dsn := 'host=postgresmmm_bdr1_1 port=5432 dbname=test'
);

SELECT bdr.bdr_group_join(
  local_node_name := 'postgresmmm_bdr3_1',
  node_external_dsn := 'host=postgresmmm_bdr3_1 port=5432 dbname=test',
  join_using_dsn := 'host=postgresmmm_bdr1_1 port=5432 dbname=test'
);

SELECT bdr.bdr_group_join(
  local_node_name := 'postgresmmm_bdr4_1',
  node_external_dsn := 'host=postgresmmm_bdr4_1 port=5432 dbname=test',
  join_using_dsn := 'host=postgresmmm_bdr1_1 port=5432 dbname=test'
);

SELECT bdr.bdr_group_join(
  local_node_name := 'postgresmmm_bdr5_1',
  node_external_dsn := 'host=postgresmmm_bdr5_1 port=5432 dbname=test',
  join_using_dsn := 'host=postgresmmm_bdr1_1 port=5432 dbname=test'
);

CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);

CREATE TABLE DEPARTMENT(
   ID INT PRIMARY KEY      NOT NULL,
   DEPT           CHAR(50) NOT NULL,
   EMP_ID         INT      NOT NULL
);

---DURABILITY?-----
docker stop postgresmmm_bdr1_1;

deathowl@Lazarus ‹ master ● › : ~/hwx/postgres-mmm
[1] % docker-compose ps
       Name                     Command               State            Ports
-------------------------------------------------------------------------------------
postgresmmm_bdr1_1   /docker-entrypoint.sh postgres   Exit 0
postgresmmm_bdr2_1   /docker-entrypoint.sh postgres   Up       0.0.0.0:5002->5432/tcp
postgresmmm_bdr3_1   /docker-entrypoint.sh postgres   Up       0.0.0.0:5003->5432/tcp
postgresmmm_bdr4_1   /docker-entrypoint.sh postgres   Up       0.0.0.0:5004->5432/tcp
postgresmmm_bdr5_1   /docker-entrypoint.sh postgres   Up       0.0.0.0:5005->5432/tcp


CREATE TABLE DEPARTMENT_X(
   ID INT PRIMARY KEY      NOT NULL,
   DEPT           CHAR(50) NOT NULL,
   EMP_ID         INT      NOT NULL
);

FAILS: Ends up in a deadlock since not all nodes are available in the cluster. Creating/altering tables fails in this case, but:
docker exec -it postgresmmm_bdr5_1 psql -U postgres -d test -c "insert into company values(2,'b', 12, 'lofasz', 123);"
INSERT 0 1

docker exec -it postgresmmm_bdr3_1 psql -U postgres -d test -c "insert into company values(2,'b', 12, 'lofasz', 123);"
ERROR:  duplicate key value violates unique constraint "company_pkey"
DETAIL:  Key (id)=(2) already exists.

YAY. Adding rows works, and gets nicely replicated along the cluster.
After restarting /removing the failing node

deathowl@Lazarus ‹ master ● › : ~/hwx/postgres-mmm
[1] % docker exec -it postgresmmm_bdr1_1 psql -U postgres -d test -c "insert into company values(2,'b', 12, 'lofasz', 123);"
ERROR:  duplicate key value violates unique constraint "company_pkey"
DETAIL:  Key (id)=(2) already exists.

deathowl@Lazarus ‹ master ● › : ~/hwx/postgres-mmm
[1] % docker exec -it postgresmmm_bdr1_1 psql -U postgres -d test -c "\dt;"
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | company    | table | postgres
 public | department | table | postgres
(2 rows)


deathowl@Lazarus ‹ master ● › : ~/hwx/postgres-mmm
[0] % docker exec -it postgresmmm_bdr1_1 psql -U postgres -d test -c "\dt;"


docker exec -it postgresmmm_bdr1_1 psql -U postgres -d test -c "CREATE TABLE DEPARTMENT_X(
   ID INT PRIMARY KEY      NOT NULL,
   DEPT           CHAR(50) NOT NULL,
   EMP_ID         INT      NOT NULL
);"
CREATE TABLE



Restarting the whole cluster:
deathowl@Lazarus ‹ master ● › : ~/hwx/postgres-mmm
[0] % docker-compose stop
Stopping postgresmmm_bdr5_1 ... done
Stopping postgresmmm_bdr3_1 ... done
Stopping postgresmmm_bdr2_1 ... done
Stopping postgresmmm_bdr1_1 ... done
Stopping postgresmmm_bdr4_1 ... done

deathowl@Lazarus ‹ master ● › : ~/hwx/postgres-mmm
[0] % docker-compose up
Starting postgresmmm_bdr1_1
Starting postgresmmm_bdr2_1
Starting postgresmmm_bdr3_1
Starting postgresmmm_bdr4_1
Starting postgresmmm_bdr5_1
.
.
.
.

docker exec -it postgresmmm_bdr1_1 psql -U postgres -d test -c " SELECT * FROM bdr.bdr_nodes;"

    node_sysid      | node_timeline | node_dboid | node_status |     node_name      |                node_local_dsn                 |              node_init_from_dsn
---------------------+---------------+------------+-------------+--------------------+-----------------------------------------------+-----------------------------------------------
6346519029629820947 |             1 |      17159 | r           | postgresmmm_bdr1_1 | host=postgresmmm_bdr1_1 port=5432 dbname=test |
6346519030967701524 |             1 |      16385 | r           | postgresmmm_bdr2_1 | host=postgresmmm_bdr2_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
6346519032075821076 |             1 |      16385 | r           | postgresmmm_bdr3_1 | host=postgresmmm_bdr3_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
6346519028322619412 |             1 |      16385 | r           | postgresmmm_bdr4_1 | host=postgresmmm_bdr4_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
6346519037869105172 |             1 |      16385 | r           | postgresmmm_bdr5_1 | host=postgresmmm_bdr5_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
(5 rows)

Cluster works just fine. :)


---- REMOVING DEAD NODES -----

docker stop postgresmmm_bdr1_1


test=# select * from bdr.bdr_nodes;
     node_sysid      | node_timeline | node_dboid | node_status |     node_name      |                node_local_dsn                 |              node_init_from_dsn
---------------------+---------------+------------+-------------+--------------------+-----------------------------------------------+-----------------------------------------------
 6346519029629820947 |             1 |      17159 | r           | postgresmmm_bdr1_1 | host=postgresmmm_bdr1_1 port=5432 dbname=test |
 6346519030967701524 |             1 |      16385 | r           | postgresmmm_bdr2_1 | host=postgresmmm_bdr2_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
 6346519032075821076 |             1 |      16385 | r           | postgresmmm_bdr3_1 | host=postgresmmm_bdr3_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
 6346519028322619412 |             1 |      16385 | r           | postgresmmm_bdr4_1 | host=postgresmmm_bdr4_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
 6346519037869105172 |             1 |      16385 | r           | postgresmmm_bdr5_1 | host=postgresmmm_bdr5_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
(5 rows)

test=# SELECT bdr.bdr_part_by_node_names(ARRAY['postgresmmm_bdr1_1']);
 bdr_part_by_node_names
------------------------

(1 row)

test=# select * from bdr.bdr_nodes;
     node_sysid      | node_timeline | node_dboid | node_status |     node_name      |                node_local_dsn                 |              node_init_from_dsn
---------------------+---------------+------------+-------------+--------------------+-----------------------------------------------+-----------------------------------------------
 6346519030967701524 |             1 |      16385 | r           | postgresmmm_bdr2_1 | host=postgresmmm_bdr2_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
 6346519032075821076 |             1 |      16385 | r           | postgresmmm_bdr3_1 | host=postgresmmm_bdr3_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
 6346519028322619412 |             1 |      16385 | r           | postgresmmm_bdr4_1 | host=postgresmmm_bdr4_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
 6346519037869105172 |             1 |      16385 | r           | postgresmmm_bdr5_1 | host=postgresmmm_bdr5_1 port=5432 dbname=test | host=postgresmmm_bdr1_1 port=5432 dbname=test
 6346519029629820947 |             1 |      17159 | k           | postgresmmm_bdr1_1 | host=postgresmmm_bdr1_1 port=5432 dbname=test |
(5 rows)


LIfe goes on as usual:
deathowl@Lazarus ‹ master › : ~
[0] % docker exec -it postgresmmm_bdr3_1 psql -U postgres -d test -c "CREATE TABLE DEPARTMENT_ZZZ(
   ID INT PRIMARY KEY      NOT NULL,
   DEPT           CHAR(50) NOT NULL,
   EMP_ID         INT      NOT NULL
);"
CREATE TABLE

deathowl@Lazarus ‹ master › : ~
[0] % docker exec -it postgresmmm_bdr2_1 psql -U postgres -d test -c "\dt;"

             List of relations
 Schema |      Name      | Type  |  Owner
--------+----------------+-------+----------
 public | company        | table | postgres
 public | department     | table | postgres
 public | department_x   | table | postgres
 public | department_zzz | table | postgres
(4 rows)

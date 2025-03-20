# trino-on-fhir

Demo for querying FHIR resources encoded as Delta Lake tables via Pathling using Trino as a query engine.

## Prerequisites

- Java 17 or later
- curl
- docker or podman with the compose plugin
- 16GiB RAM (Trino is limited to 8GB, see [jvm.config](config/trino/etc/jvm.config))
- 4 CPUs

## Setup

Download synthea & trino cli

```sh
curl -LO https://github.com/synthetichealth/synthea/releases/download/master-branch-latest/synthea-with-dependencies.jar
curl -L -o trino-cli.jar https://repo1.maven.org/maven2/io/trino/trino-cli/472/trino-cli-472-executable.jar
```

Generate sample resources:

```sh
java -jar synthea-with-dependencies.jar -s 20250306 -cs 20250306 -r 20250306 -p 1000 -c config/synthea.properties --exporter.baseDirectory="./synthea/output/bulk" --exporter.fhir.bulk_data="true"
```

Start Trino and kick-off the FHIR resource import

```sh
# this automatically starts a Pathling server and kicks-off the resource $import.
docker compose up warehousekeeper --attach-dependencies --abort-on-container-failure
# after the import is done, we no longer need the Pathling server itself
docker compose stop pathling
```

Run queries

```sh
java -jar trino-cli.jar http://localhost:8080 -f sql/table-counts.sql
java -jar trino-cli.jar http://localhost:8080 -f sql/observations-by-code.sql
java -jar trino-cli.jar http://localhost:8080 -f sql/hemoglobin-count.sql
java -jar trino-cli.jar http://localhost:8080 -f sql/diabetes-extract.sql
```

Or go interactive:

```sh
java -jar trino-cli.jar http://localhost:8080
```

The schema and catalog containing the resources is called: `fhir.default`.

```sh
$ java -jar trino-cli.jar http://localhost:8080
trino> SHOW CATALOGS;
 Catalog
---------
 fhir
 system
(2 rows)

Query 20250308_122142_00001_cqmbk, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.25 [0 rows, 0B] [0 rows/s, 0B/s]

trino> SHOW SCHEMAS FROM fhir;
       Schema
--------------------
 default
 information_schema
(2 rows)

Query 20250308_122152_00002_cqmbk, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.17 [2 rows, 35B] [11 rows/s, 202B/s]

trino> SHOW TABLES FROM fhir.default;
    Table
-------------
 condition
 encounter
 observation
 patient
(4 rows)

Query 20250308_122210_00004_cqmbk, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.22 [4 rows, 104B] [18 rows/s, 479B/s]
```

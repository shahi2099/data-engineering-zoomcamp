# Module 1 Homework: Docker & SQL

## Question 1. Understanding Docker images

Run docker with the `python:3.13` image. Use an entrypoint `bash` to interact with the container.

What's the version of `pip` in the image?

- 25.3
- 24.3.1
- 24.2.1
- 23.3.1

```
25.3 
```

```bash
docker run -it --rm --entrypoint=bash python:3.13
pip --version
pip 25.3 from /usr/local/lib/python3.13/site-packages/pip (python 3.13)
```


## Question 2. Understanding Docker networking and docker-compose

Given the following `docker-compose.yaml`, what is the `hostname` and `port` that pgadmin should use to connect to the postgres database?

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```

- postgres:5433
- localhost:5432
- db:5433
- postgres:5432
- db:5432

If multiple answers are correct, select any 

```
db:5433
```

## Prepare the Data

Download the green taxi trips data for November 2025:

```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet
```

You will also need the dataset with zones:

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```

```bash
uv init --python=3.13 
uv which python
uv add pandas pyarrow sqlalchemy tqdm psycopg
uv add --dev jupyter pgcli
```

## Question 3. Counting short trips

For the trips in November 2025 (lpep_pickup_datetime between '2025-11-01' and '2025-12-01', exclusive of the upper bound), how many trips had a `trip_distance` of less than or equal to 1 mile?

- 7,853
- 8,007
- 8,254
- 8,421

```
8,007
```

```sql
SELECT
	COUNT(*)
FROM
	PUBLIC.GREEN_TAXI_DATA G
WHERE
	G.LPEP_PICKUP_DATETIME BETWEEN '2025-11-01' AND '2025-12-01'
	AND G.TRIP_DISTANCE <= 1;
```

## Question 4. Longest trip for each day

Which was the pick up day with the longest trip distance? Only consider trips with `trip_distance` less than 100 miles (to exclude data errors).

Use the pick up time for your calculations.

- 2025-11-14
- 2025-11-20
- 2025-11-23
- 2025-11-25

```
2025-11-14
```

```sql
SELECT
	G.LPEP_PICKUP_DATETIME,
	G.TRIP_DISTANCE,
	ROW_NUMBER() OVER (
		ORDER BY
			G.TRIP_DISTANCE DESC
	) AS ROW_NUM
FROM
	PUBLIC.GREEN_TAXI_DATA G
WHERE
	G.LPEP_PICKUP_DATETIME BETWEEN '2025-11-01' AND '2025-12-01'
	AND G.TRIP_DISTANCE < 100
```

## Question 5. Biggest pickup zone

Which was the pickup zone with the largest `total_amount` (sum of all trips) on November 18th, 2025?

- East Harlem North
- East Harlem South
- Morningside Heights
- Forest Hills

```
East Harlem North
```

```sql
WITH
	CTE_ZONE_SUM AS (
		SELECT
			TZ."Zone",
			SUM(G.TOTAL_AMOUNT) AS TOTAL_AMOUNT
		FROM
			PUBLIC.GREEN_TAXI_DATA G
			JOIN PUBLIC.TAXI_ZONE_LOOKUP TZ ON G."PULocationID" = TZ."LocationID"
		WHERE
			G.LPEP_PICKUP_DATETIME >= '2025-11-18'
			AND G.LPEP_PICKUP_DATETIME < '2025-11-19'
		GROUP BY
			TZ."Zone"
	),
	CTE_ORDERED AS (
		SELECT
			"Zone",
			TOTAL_AMOUNT,
			ROW_NUMBER() OVER (
				ORDER BY
					TOTAL_AMOUNT DESC
			) AS ROW_NUM
		FROM
			CTE_ZONE_SUM
	)
SELECT
	*
FROM
	CTE_ORDERED
WHERE
	ROW_NUM = 1;

```
## Question 6. Largest tip

For the passengers picked up in the zone named "East Harlem North" in November 2025, which was the drop off zone that had the largest tip?

Note: it's `tip` , not `trip`. We need the name of the zone, not the ID.

- JFK Airport
- Yorkville West
- East Harlem North
- LaGuardia Airport

```
Yorkville West
```

```sql

WITH
	CTE_ZONES AS (
		SELECT
			DO_TZ."Zone",
			MAX(G.TIP_AMOUNT) AS TIP_AMOUNT
		FROM
			PUBLIC.GREEN_TAXI_DATA G
			JOIN PUBLIC.TAXI_ZONE_LOOKUP PU_TZ ON G."PULocationID" = PU_TZ."LocationID"
			AND PU_TZ."Zone" = 'East Harlem North'
			JOIN PUBLIC.TAXI_ZONE_LOOKUP DO_TZ ON G."DOLocationID" = DO_TZ."LocationID"
		WHERE
			G.LPEP_PICKUP_DATETIME >= '2025-11-01'
			AND G.LPEP_PICKUP_DATETIME < '2025-12-01'
		GROUP BY
			DO_TZ."Zone"
	)
SELECT
	"Zone",
	TIP_AMOUNT
FROM
	CTE_ZONES
ORDER BY
	TIP_AMOUNT DESC
LIMIT
	1;
```

## Question 7. Terraform Workflow

Which of the following sequences, respectively, describes the workflow for:
1. Downloading the provider plugins and setting up backend,
2. Generating proposed changes and auto-executing the plan
3. Remove all resources managed by terraform`

Answers:
- terraform import, terraform apply -y, terraform destroy
- teraform init, terraform plan -auto-apply, terraform rm
- terraform init, terraform run -auto-approve, terraform destroy
- terraform init, terraform apply -auto-approve, terraform destroy
- terraform import, terraform apply -y, terraform rm

```
terraform init
terraform apply -auto-approve
terraform destroy
```




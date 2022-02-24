# Supabase

## Postgres container stuff

### Import using CSV

```shell
docker run --rm -v $PWD:/app --net=host supabase/postgres psql 'postgresql://postgres:postgres@localhost:54322/postgres' -c "COPY stars(id, created_at, full_name, stargazers_count, open_issues_count, forks_count) FROM 'stars.csv' DELIMITER ',' CSV HEADER;"
```

### Pg_dump the classic way

```shell
docker exec -i postgres pg_dump 'postgresql://postgres:postgres@localhost:54322:5432/postgres' > migrate.sql
```

### Extend seeding system with shell

```shell
docker exec -it "supabase_db_$(jq -r '.projectId' supabase/config.json)" bash -c $(cat supabase/seed.sh)
```

## SQL studio stuff

### Clean migrations only

```sql
delete from supabase_migrations.schema_migrations;
drop table public.recommendations;
drop table public.stars;
drop table public.user_stars;
drop table public.users;
drop table public.votes;
```
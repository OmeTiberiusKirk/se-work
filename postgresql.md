## Manual Repository Configuration

Import the repository key :

```
sudo apt install curl ca-certificates && \
sudo install -d /usr/share/postgresql-common/pgdg && \
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
--fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
```

Install PostgreSQL:

```
sudo apt update && \
sudo apt install postgresql-18
```

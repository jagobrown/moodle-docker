# moodle-docker: notes for local development

## Some initial checks when returning to develop & running containers

```bash

# Check env vars
printenv

# Set below if absent
export MOODLE_DOCKER_WWWROOT=./MOODLE_LTS
export MOODLE_DOCKER_DB=pgsql

# Check compose version
docker compose version

```

## Docker-compose commands

```
bin/moodle-docker-compose up -d
bin/moodle-docker-compose stop

bin/moodle-docker-compose start
bin/moodle-docker-compose stop

# Remove the postgres volume (this will DELETE all database data)
# or whatever your volume name is if different
docker volume rm moodle-docker_pgsql-data   



```
# Quickstart

The scripts from this quickstart guide set up a local environment consisting of
two Trino servers and a PostgreSQL database running in Docker, and a Trino
Gateway server running in the host operating system.

## Start Trino Gateway server

The following script starts a Trino Gateway server using the
[Quickstart configuration](quickstart-config.yaml) at http://localhost:8080.
It also starts a dockerized PostgreSQL database at localhost:5432.

To start the server, copy the script below to a temporary directory
under the project root folder, and run it at the temporary directory.

It  copies the following, necessary files to current directory:

- gateway-ha.jar
- gateway-ha-persistence-postgres.sql
- quickstart-config.yaml

```shell
#!/usr/bin/env sh

VERSION=13
BASE_URL="https://repo1.maven.org/maven2/io/trino/gateway/gateway-ha"

# Copy necessary files
copy_files() {
    if [[ ! -f "gateway-ha.jar" ]]; then
        echo "Fetching gateway-ha.jar version $VERSION"
        curl -O "$BASE_URL/$VERSION/gateway-ha-$VERSION-jar-with-dependencies.jar"
        mv "gateway-ha-$VERSION-jar-with-dependencies.jar" gateway-ha.jar
    fi

    [[ ! -f "quickstart-config.yaml" ]] && cp ../docs/quickstart-config.yaml .
    [[ ! -f "gateway-ha-persistence-postgres.sql" ]] && cp ../gateway-ha/src/main/resources/gateway-ha-persistence-postgres.sql .
}

# Start PostgreSQL database if not running
start_postgres_db() {
    if ! docker ps --format '{{.Names}}' | grep -q '^local-postgres$'; then
        echo "Starting PostgreSQL database container"
        PGPASSWORD=mysecretpassword
        docker run -v "$(pwd)"/gateway-ha-persistence-postgres.sql:/tmp/gateway-ha-persistence-postgres.sql \
            --name local-postgres -p 5432:5432 -e POSTGRES_PASSWORD=$PGPASSWORD -d postgres:latest
        sleep 5
        docker exec local-postgres psql -U postgres -h localhost -c 'CREATE DATABASE gateway'
        docker exec local-postgres psql -U postgres -h localhost -d gateway -f /tmp/gateway-ha-persistence-postgres.sql
    fi
}

# Main execution flow
copy_files
start_postgres_db

# Start Trino Gateway server
echo "Starting Trino Gateway server..."
java -Xmx1g -jar ./gateway-ha.jar ./quickstart-config.yaml
```

You can clean up by running

```shell
docker kill local-postgres && docker rm local-postgres
kill -5 $(jps | grep gateway-ha.jar | cut -d' ' -f1)
```

## Add Trino backends

This following script starts two dockerized Trino servers at
http://localhost:8081 and http://localhost:8082. It then adds them as backends
to the Trino Gateway server started by the preceding script.

```shell
#!/usr/bin/env sh

# Start a pair of Trino servers on different ports
docker run --name trino1 -d -p 8081:8080 \
    -e JAVA_TOOL_OPTIONS="-Dhttp-server.process-forwarded=true" trinodb/trino

docker run --name trino2 -d -p 8082:8080 \
    -e JAVA_TOOL_OPTIONS="-Dhttp-server.process-forwarded=true" trinodb/trino

# Add the Trino servers as Gateway backends
add_backend() {
    local name=$1
    local proxy_to=$2

    curl -H "Content-Type: application/json" -X POST \
        localhost:8080/gateway/backend/modify/add \
        -d "{
              \"name\": \"$name\",
              \"proxyTo\": \"$proxy_to\",
              \"active\": true,
              \"routingGroup\": \"adhoc\"
            }"
}

# Adding Trino servers as backends
add_backend "trino1" "http://localhost:8081"
add_backend "trino2" "http://localhost:8082"                                                                                       }'
```

You can clean up by running

```shell
docker kill trino1 && docker rm trino1
docker kill trino2 && docker rm trino2
```

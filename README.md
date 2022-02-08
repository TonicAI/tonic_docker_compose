# Tonic Docker Compose
Deploying Tonic via `docker-compose` is a relatively straightforward process. This repository contains an example/template docker-compose.yaml which can be used for this.

Project structure:
```
.
├── .template.env
├── docker-compose.yaml
└── README.md
```

[_docker-compose.yaml_](docker-compose.yaml)
``` yaml
services:
    tonic_web_server:
        image: quay.io/tonicai/tonic_web_server:latest
        ...
    tonic_worker:
        image: quay.io/tonicai/tonic_worker:latest
        ...
    tonic_pii_detection:
        image: quay.io/tonicai/tonic_pii_detection:latest
        ...
    # OPTIONAL: This container must be deployed to support Webhooks and Email notifications.
    tonic_notifications:
        image: quay.io/tonicai/tonic_notifications:latest
        ...
    # OPTIONAL: This container must be deployed to use the Smart Linking generator.
    tonic_pyml_service:
        image: quay.io/tonicai/tonic_pyml_service:latest
        ...
    # OPTIONAL: This container must be deployed if you are using an Oracle source and destination database.
    tonic_oracle_helper:
        image: quay.io/tonicai/tonic_oracle_helper_19:latest
        ...
    # OPTIONAL: It is recommended to use a standalone or managed Postgres database (e.g. AWS RDS for PostgreSQL, GCP Cloud SQL for PostgreSQL, Azure Database for PostgreSQL) for Tonic's application database. These provide many benefits such as automated backups and patches/upgrades. 
    but a containerized PostgreSQL database can be used.
    tonic_db:
        image: postgres:14
        ...
```

## Configuration

### .env
Before deploying this setup, you need to rename [.template.env](.template.env) to `.env` and configure the following values, or set the environment variables directly within docker-compose.yaml. Note that several variables are shared by multiple, or all, containers.

### Tonic License

- `TONIC_LICENSE`: This value will be provided by Tonic.

### Environment Name

- `ENVIRONMENT_NAME`: E.g. "my-company-name", or if deploying multiple Tonic instances, "my-company-name-dev" or "my-company-name-prod to differentiate instances.

### Version

- `VERSION_TAG`: "latest" or a specific version tag. Tonic's tag convention is just the release number, e.g. "123". Release notes are available at [doc.tonic.ai](https://docs.tonic.ai/app/release-notes).

### Application Database
The connection details for the Postgres metadata/application database which holds Tonic's state (user accounts, workspaces, etc.).

- `TONIC_DB_HOST`
- `TONIC_DB_PORT`
- `TONIC_DB_DATABASE`
- `TONIC_DB_USERNAME`
- `TONIC_DB_PASSWORD`

### Log Collection
Tonic never collects your sensitive data. Enabling this option securely and safely shares logs with Tonic's engineering team. We recommend that you enable this option. See: https://docs.tonic.ai/app/sharing-logs-with-tonic

- `ENABLE_LOG_COLLECTION`: "false" (default) or "true"

### Consistency Seed
This value is used to support [Consistency](https://docs.tonic.ai/app/concepts/consistency) functionality across data generations.

- `TONIC_STATISTICS_SEED`: Any signed 32-bit integer, i.e. between -2147483648 and 2147483647

### Memory Limits
The example docker-compose.yaml file includes memory limits per container. These are a baseline recommendation assuming a host with 16gb of memory dedicated to Tonic. In some cases it may be necessary to modify these limits and increase the total memory to more than 16gb.

## Deploy
To run Tonic, execute the `docker-compose up -d` command from within the directory containing your docker-compose.yaml file.

``` shell
$ docker-compose up -d
Creating tonic_worker        ... done
Creating tonic_pii_detection ... done
Creating tonic_pyml_service  ... done
Creating tonic_notifications ... done
Creating tonic_web_server    ... done
```


## Validate the deployment

Use `docker ps` to check that containers are running:
```
$ docker ps
CONTAINER ID   IMAGE                                           COMMAND                  CREATED         STATUS                   PORTS                                         NAMES
80e549224dd5   quay.io/tonicai/tonic_worker:latest             "/bin/sh -c 'bash st…"   3 minutes ago   Up 3 minutes             0.0.0.0:8080->80/tcp, 0.0.0.0:4433->443/tcp   tonic_worker
b30fb352247d   quay.io/tonicai/tonic_pyml_service:latest       "/bin/sh -c 'bash st…"   3 minutes ago   Up 3 minutes             0.0.0.0:7700->7700/tcp                        tonic_pyml_service
99b2a09d96ef   quay.io/tonicai/tonic_pii_detection:latest      "/bin/sh -c 'bash st…"   3 minutes ago   Up 3 minutes             0.0.0.0:7687->7687/tcp                        tonic_pii_detection
f1379937bed8   quay.io/tonicai/tonic_web_server:latest         "/bin/sh -c 'bash st…"   3 minutes ago   Up 3 minutes             0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp      tonic_web_server
932379479ce9   quay.io/tonicai/tonic_notifications:latest      "/bin/sh -c 'sh star…"   3 minutes ago   Up 3 minutes             0.0.0.0:7000->80/tcp, 0.0.0.0:7001->443/tcp   tonic_notifications
```

The Tonic UI will be accessible in your browser on port 80 or 443. I.e. from the server at https://localhost:443 or http://localhost, or via the server IP/hostname or domain you are routing to the server.

Tonic may take a few minutes to fully startup. You can validate that it has fully started up and is in a healthy state by running `docker logs tonic_web_server | grep listening` and checking for the following output.

``` shell
$ docker logs tonic_web_server | grep listening
[2022-02-08T16:23:15+00:00 INF Microsoft.Hosting.Lifetime] Now listening on: http://0.0.0.0:80
[2022-02-08T16:23:15+00:00 INF Microsoft.Hosting.Lifetime] Now listening on: https://0.0.0.0:443
```

### If the Tonic UI does not load
1. Check that Tonic is successfully connecting to the application database.
Run `docker logs tonic_web_server | grep "Failed to connect"`. If you see a `Failed to connect to db during startup.  Retrying in 5 seconds...` message like below, Tonic is not able to connect to your Postgres application database. Please verify the network path between Tonic and the database as well as the connection parameters.

``` shell
$ docker logs tonic_web_server | grep "Failed to connect"
[2022-02-08T16:18:26+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
[2022-02-08T16:18:31+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
[2022-02-08T16:18:36+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
[2022-02-08T16:18:41+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
```

2. If Tonic appears to be running and in a healthy state but you are unable to load the UI, verify that Tonic is reachable. Common issues may be a requirement to be on a VPN or firewall rules preventing access.
# Setting Up the AAS Registry
We provide example Set-Ups to get you started with the BaSyx Go Components on our [GitHub Repository](https://github.com/eclipse-basyx/basyx-go-components/tree/main/examples).
But if you need to configure the service yourself, this page will guide you through.

## Using Docker Compose
The easiest way to use and set-up the AAS Registry is Docker Compose.

The minimal configuration includes three services:

1. PostgreSQL
2. BaSyx Configuration (Go)
3. BaSyx AAS Registry (Go)

```yaml
services:
  postgres:
    image: postgres:18
    container_name: postgres_basyx
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: basyxTestDB
    command: ["postgres", "-c", "listen_addresses=*"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d basyxTestDB"]
      interval: 10s
      timeout: 5s
      retries: 5

  basyx_configuration:
    container_name: basyx_configuration
    image: eclipsebasyx/basyxconfigurationservice-go:SNAPSHOT
    pull_policy: always
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin123
      - POSTGRES_DBNAME=basyxTestDB
      - POSTGRES_MAXOPENCONNECTIONS=50
      - POSTGRES_MAXIDLECONNECTIONS=25
      - POSTGRES_CONNMAXLIFETIMEMINUTES=5
      - POSTGRES_CONNMAXIDLETIMEMINUTES=0
    depends_on:
      postgres:
        condition: service_healthy

  aas_registry:
    image: eclipsebasyx/aasregistry-go:SNAPSHOT
    pull_policy: always
    container_name: aas_registry
    environment:
      - SERVER_PORT=8082
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin123
      - POSTGRES_DBNAME=basyxTestDB
    ports:
      - "8082:8082"
    depends_on:
      basyx_configuration:
        condition: service_completed_successfully
```
*docker-compose.yml including PostgreSQL 18, the BaSyx Go Configuration Service, and BaSyx Go AAS Registry*

### Access Rules and Trustlist Files (Secured Setup)

For general handling of OIDC trustlist and ABAC access-rules files (config keys, env vars, startup behavior), see [Security Configuration Files (Common)](../common/configuration#security-files).

For this component in Docker Compose, mount the security files into the container and configure `ABAC_ENABLED=true`, `ABAC_MODELPATH`, and `OIDC_TRUSTLISTPATH` if you enable ABAC.

## Using BaSyx Go Components without Docker
If you need to run the AAS Registry without Docker, build the binary from source for your target platform.

```{warning}
We recommend using the Docker Images for production use-cases, as they are pre-configured and optimized for production environments.
```

### Prerequisites
- [Go](https://go.dev/dl/) (Use at least the version specified by the `go` directive in the repository's [`go.mod`](https://github.com/eclipse-basyx/basyx-go-components/blob/main/go.mod)).
- [Git](https://git-scm.com/)

### Cloning the Repository
```bash
git clone https://github.com/eclipse-basyx/basyx-go-components
```

### Building the Binary

Change to the AAS Registry service directory:
```bash
cd basyx-go-components/cmd/aasregistryservice
```

#### Linux / macOS

Build the executable with:
```bash
go build -o aasregistryservice
```

#### Windows

Build the executable with the `.exe` extension:
```powershell
go build -o aasregistryservice.exe
```

### Running the Service
Before running the service, ensure PostgreSQL is available and that the BaSyx database schema has already been initialized by the [BaSyx Configuration Service](../configuration_service/index). Configure the PostgreSQL connection through environment variables or the provided `config.yaml`.

#### Linux / macOS

Run the service with:
```bash
./aasregistryservice -config ./config.yaml
```

#### Windows PowerShell

Run the service with:
```powershell
.\aasregistryservice.exe -config .\config.yaml
```

The AAS Registry does not initialize the database schema itself. Database initialization and migrations are handled by the BaSyx Configuration Service.

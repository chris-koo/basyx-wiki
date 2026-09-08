# Setting Up the Company Lookup
We provide example Set-Ups to get you started with the BaSyx Go Components on our [GitHub Repository](https://github.com/eclipse-basyx/basyx-go-components/tree/main/examples/BaSyxCompanyLookup).
But if you need to configure the service yourself, this page will guide you through.

## Using Docker Compose
The easiest way to use and set-up the Company Lookup is Docker Compose.

The minimal configuration includes three services:

1. PostgreSQL
2. BaSyx Configuration Service (Go), which initializes the database
3. BaSyx Company Lookup (Go)

```yaml
services:
  postgres:
    image: postgres:18
    container_name: postgres_basyx_company_lookup
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: basyxCompanyLookupDB
    command: ["postgres", "-c", "listen_addresses=*"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d basyxCompanyLookupDB"]
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
      - POSTGRES_DBNAME=basyxCompanyLookupDB
      - POSTGRES_MAXOPENCONNECTIONS=50
      - POSTGRES_MAXIDLECONNECTIONS=25
      - POSTGRES_CONNMAXLIFETIMEMINUTES=5
      - POSTGRES_CONNMAXIDLETIMEMINUTES=0
    depends_on:
      postgres:
        condition: service_healthy

  company-lookup:
    container_name: company-lookup
    image: eclipsebasyx/companylookup-go:SNAPSHOT
    pull_policy: always
    ports:
      - 5080:5080
    environment:
      - SERVER_PORT=5080
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin123
      - POSTGRES_DBNAME=basyxCompanyLookupDB
      - POSTGRES_MAXOPENCONNECTIONS=50
      - POSTGRES_MAXIDLECONNECTIONS=25
      - POSTGRES_CONNMAXLIFETIMEMINUTES=5
      - POSTGRES_CONNMAXIDLETIMEMINUTES=0
    depends_on:
      basyx_configuration:
        condition: service_completed_successfully
```
*docker-compose.yml including PostgreSQL 18, the BaSyx Configuration Service, and BaSyx Go Company Lookup*

If you need advanced configuration options, see [General Configuration](../common/configuration).

## Using BaSyx Go Components without Docker
If you need to run the Company Lookup without Docker, build the binary from source for your target platform.

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

Change to the Company Lookup service directory:
```bash
cd basyx-go-components/cmd/companylookupservice
```

#### Linux / macOS

Build the executable with:
```bash
go build -o companylookupservice
```

#### Windows

Build the executable with the `.exe` extension:
```powershell
go build -o companylookupservice.exe
```

### Running the Service
Before running the service, ensure PostgreSQL is available and that the BaSyx database schema has already been initialized by the [BaSyx Configuration Service](../configuration_service/index). Configure the PostgreSQL connection through environment variables or the provided `config.yaml`.

#### Linux / macOS

Run the service with:
```bash
./companylookupservice -config ./config.yaml
```

#### Windows PowerShell

Run the service with:
```powershell
.\companylookupservice.exe -config .\config.yaml
```

The Company Lookup does not initialize the database schema itself. Database initialization and migrations are handled by the BaSyx Configuration Service.

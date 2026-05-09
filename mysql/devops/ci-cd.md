# CI/CD for MySQL — Docker, Testing, and Pipeline Integration

## Overview

Integrating MySQL into CI/CD pipelines ensures that schema changes, queries, and application code are validated automatically before reaching production. A well-designed database CI pipeline catches breaking schema changes, slow queries, and data integrity issues early in the development cycle.

The foundation is a reproducible MySQL test environment, typically built with Docker containers that can be spun up in seconds, initialized with schema and seed data, exercised by integration tests, and torn down cleanly. This approach eliminates "works on my machine" problems and gives every developer and CI runner an identical database.

This skill covers Docker image selection, compose configurations, test database initialization, integration testing patterns, and pipeline definitions for GitHub Actions and Jenkins.

## Key Concepts

- **Ephemeral database**: A MySQL instance created fresh for each test run, ensuring test isolation and reproducibility.
- **Schema initialization**: Applying DDL scripts to a fresh MySQL container via entrypoint scripts or migration tools.
- **Service container**: A sidecar container in CI that provides MySQL to the test runner (GitHub Actions services, Jenkins docker agent).
- **Health check**: A readiness probe ensuring MySQL is accepting connections before tests begin.
- **Fixture data**: Seed/reference data loaded before tests to provide a known baseline state.
- **mysqltest framework**: MySQL's own test framework using `.test` and `.result` files for regression testing of SQL behavior.

## Docker MySQL Images

### Official MySQL image

```dockerfile
# docker-compose.yml for development
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: testdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    ports:
      - "3306:3306"
    volumes:
      - ./docker/init:/docker-entrypoint-initdb.d
      - mysql_data:/var/lib/mysql
    command: >
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --innodb-buffer-pool-size=256M
      --max-connections=100
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpass"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  mysql_data:
```

### Percona Server image

```yaml
services:
  mysql:
    image: percona/percona-server:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: testdb
    command: >
      --innodb-buffer-pool-size=256M
      --log-bin=binlog
      --server-id=1
      --gtid-mode=ON
      --enforce-gtid-consistency=ON
```

### Initialization scripts

Files placed in `/docker-entrypoint-initdb.d/` are executed in alphabetical order on first startup.

```
docker/init/
  01-schema.sql        # CREATE TABLE statements
  02-indexes.sql       # Secondary indexes
  03-seed-data.sql     # Reference/lookup data
  04-test-fixtures.sql # Test-specific data
  05-grants.sql        # User permissions
```

```sql
-- docker/init/01-schema.sql
CREATE TABLE IF NOT EXISTS users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS orders (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    total DECIMAL(10,2) NOT NULL,
    status ENUM('pending','processing','shipped','delivered') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Multi-Environment Compose

```yaml
# docker-compose.test.yml — optimized for fast CI
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: test
      MYSQL_DATABASE: test_db
    tmpfs:
      - /var/lib/mysql       # RAM-backed storage for speed
    command: >
      --innodb-flush-log-at-trx-commit=0
      --innodb-flush-method=nosync
      --innodb-doublewrite=OFF
      --sync-binlog=0
      --skip-log-bin
      --innodb-buffer-pool-size=512M
      --max-connections=200
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-ptest"]
      interval: 2s
      timeout: 2s
      retries: 15

  app:
    build: .
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_NAME: test_db
      DB_USER: root
      DB_PASSWORD: test
```

## GitHub Actions Pipeline

```yaml
name: Database CI
on:
  pull_request:
    paths:
      - 'migrations/**'
      - 'src/**'

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: ci_password
          MYSQL_DATABASE: ci_db
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h localhost -u root -pci_password"
          --health-interval=5s
          --health-timeout=3s
          --health-retries=10

    steps:
      - uses: actions/checkout@v4

      - name: Wait for MySQL
        run: |
          until mysqladmin ping -h 127.0.0.1 -u root -pci_password --silent; do
            sleep 1
          done

      - name: Apply migrations
        run: |
          flyway -url=jdbc:mysql://127.0.0.1:3306/ci_db \
                 -user=root -password=ci_password \
                 -locations=filesystem:./migrations \
                 migrate

      - name: Run schema validation
        run: |
          mysql -h 127.0.0.1 -u root -pci_password ci_db < tests/validate_schema.sql

      - name: Run integration tests
        env:
          DATABASE_URL: mysql://root:ci_password@127.0.0.1:3306/ci_db
        run: |
          npm test -- --grep "database"

      - name: Check for slow queries in migration
        run: |
          mysql -h 127.0.0.1 -u root -pci_password -e \
            "SELECT * FROM performance_schema.events_statements_summary_by_digest
             WHERE AVG_TIMER_WAIT > 1000000000000
             ORDER BY AVG_TIMER_WAIT DESC LIMIT 10;"
```

## Jenkins Pipeline

```groovy
pipeline {
    agent any
    environment {
        MYSQL_ROOT_PASSWORD = credentials('mysql-test-password')
    }
    stages {
        stage('Start MySQL') {
            steps {
                sh '''
                docker run -d --name mysql-ci \
                  -e MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
                  -e MYSQL_DATABASE=ci_db \
                  -p 3307:3306 \
                  --tmpfs /var/lib/mysql \
                  mysql:8.0 \
                  --innodb-flush-log-at-trx-commit=0 \
                  --skip-log-bin

                # Wait for readiness
                for i in $(seq 1 30); do
                  docker exec mysql-ci mysqladmin ping -u root -p${MYSQL_ROOT_PASSWORD} && break
                  sleep 2
                done
                '''
            }
        }
        stage('Migrate') {
            steps {
                sh '''
                flyway -url=jdbc:mysql://localhost:3307/ci_db \
                       -user=root -password=${MYSQL_ROOT_PASSWORD} \
                       migrate
                '''
            }
        }
        stage('Test') {
            steps {
                sh 'DATABASE_URL=mysql://root:${MYSQL_ROOT_PASSWORD}@localhost:3307/ci_db ./run_tests.sh'
            }
        }
    }
    post {
        always {
            sh 'docker rm -f mysql-ci || true'
        }
    }
}
```

## Integration Testing Patterns

### Transaction-per-test isolation

```python
import pytest
import pymysql

@pytest.fixture
def db_connection():
    conn = pymysql.connect(
        host='127.0.0.1', port=3306,
        user='root', password='test',
        database='test_db', autocommit=False
    )
    yield conn
    conn.rollback()   # Undo all changes from the test
    conn.close()

def test_insert_user(db_connection):
    cursor = db_connection.cursor()
    cursor.execute(
        "INSERT INTO users (email, name) VALUES (%s, %s)",
        ('test@example.com', 'Test User')
    )
    cursor.execute("SELECT COUNT(*) FROM users WHERE email = %s", ('test@example.com',))
    assert cursor.fetchone()[0] == 1
```

### Schema diff validation

```bash
#!/bin/bash
# Compare expected schema against actual after migration
mysqldump -h 127.0.0.1 -u root -ptest --no-data test_db > /tmp/actual_schema.sql
diff <(grep -v "^--" expected_schema.sql | grep -v "^$") \
     <(grep -v "^--" /tmp/actual_schema.sql | grep -v "^$")
if [ $? -ne 0 ]; then
  echo "FAIL: Schema drift detected"
  exit 1
fi
```

### mysqltest framework

```
# tests/test_users.test
--source include/have_innodb.inc

CREATE TABLE t1 (id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(50));
INSERT INTO t1 (name) VALUES ('Alice'), ('Bob');
SELECT * FROM t1 ORDER BY id;
DROP TABLE t1;

# tests/test_users.result (expected output)
id	name
1	Alice
2	Bob
```

```bash
# Run with mysqltest
mysqltest --user=root --password=test --database=test_db < tests/test_users.test
```

## Best Practices

- Use `tmpfs` for MySQL data directory in CI to dramatically speed up test runs (3-5x faster).
- Disable durability settings in test environments (`innodb_flush_log_at_trx_commit=0`, `sync_binlog=0`, `skip-log-bin`).
- Pin Docker image tags to specific versions (e.g., `mysql:8.0.36`) to avoid surprise behavior changes.
- Use health checks with `depends_on: condition: service_healthy` to prevent race conditions.
- Run migration validation as a separate CI step before application tests.
- Cache Docker images in CI to reduce pull times.
- Use separate databases per parallel test suite to avoid cross-contamination.
- Include a schema snapshot check in CI to detect uncommitted schema drift.
- Mask database passwords in CI logs using secrets/credentials management.
- Test both upgrade (forward migration) and downgrade (rollback) paths.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not waiting for MySQL readiness before running tests | Tests fail with "Connection refused" errors intermittently | Use health checks or retry loops before test execution |
| Using `MYSQL_ALLOW_EMPTY_PASSWORD` in CI | Security scanners flag the pipeline; bad habit leaks to staging configs | Always set `MYSQL_ROOT_PASSWORD` even in ephemeral test containers |
| Sharing a single MySQL instance across parallel test jobs | Data collisions, flaky tests, non-deterministic failures | Use unique database names per job or separate containers |
| Running CI MySQL with production durability settings | CI runs take 5-10x longer than necessary | Disable `innodb_doublewrite`, `sync_binlog`, use `tmpfs` |
| Not testing rollback migrations | Broken rollback discovered during production incident recovery | Run `flyway undo` or `liquibase rollback` as a CI step |

## MySQL Version Notes

- **5.7**: Use `mysql:5.7` image. Default auth plugin is `mysql_native_password`. No `caching_sha2_password` issues with older clients.
- **8.0**: Default auth plugin changed to `caching_sha2_password` in 8.0.4+. Use `--default-authentication-plugin=mysql_native_password` for compatibility with older drivers. Supports `--partial-revokes` for fine-grained CI user permissions.
- **8.4/9.x**: `mysql_native_password` plugin removed by default. CI configurations must use `caching_sha2_password` or `mysql_clear_password`. Docker images may require updated client libraries.

## Sources

- [Official MySQL Docker Image](https://hub.docker.com/_/mysql)
- [Percona Server Docker Image](https://hub.docker.com/r/percona/percona-server)
- [GitHub Actions - Creating MySQL Service Containers](https://docs.github.com/en/actions/using-containerized-services/creating-mysql-service-containers)
- [MySQL Test Framework](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_TEST_RUN.html)
- [Flyway Docker Integration](https://documentation.red-gate.com/fd/docker-184127599.html)

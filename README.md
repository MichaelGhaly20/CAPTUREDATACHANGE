## CDC with Debezium, Kafka, Postgres, Docker

### Overview

This Python script generates simulated financial transactions and inserts them into a PostgreSQL database. It's particularly useful for setting up a test environment for Change Data Capture (CDC) with Debezium. The script utilizes the faker library to create realistic, yet fictitious, transaction data and inserts it into a PostgreSQL table.

### System Architecture
![image](https://github.com/MichaelGhaly20/CAPTUREDATACHANGE/assets/59583421/0782faaa-81b5-4b1d-882d-d92560151336)


### Prerequisites

Before running this script, ensure you have the following installed:

- Python 3.9+
- `psycopg2` library for Python
- `faker` library for Python
- PostgreSQL server running locally or accessible remotely
- Docker and Docker Compose installed on your machine
- Basic understanding of Docker, Kafka, and Postgres

### Installation

**Install Required Python Libraries:**

You can install the required libraries using pip:

```
pip install psycopg2-binary faker
```

### Services in the Compose File

- Zookeeper: A centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.
- Kafka Broker: A distributed streaming platform used for handling real-time data feeds.
- Confluent Control Center: A web-based tool for managing and monitoring Apache Kafka.
- Debezium: An open-source distributed platform for change data capture.
- Debezium UI: A user interface for managing and monitoring Debezium connectors.
- Postgres: An open-source relational database.

### Getting Started

1. **Clone the Repository:** Ensure you have this Docker Compose file in your local system. If it's part of a repository, clone the repository to your local machine.

2. **Navigate to the Directory:** Open a terminal and navigate to the directory containing the Docker Compose file.

3. **Run Docker Compose:** Execute the following command to start all services defined in the Docker Compose file:

   ```
   docker-compose up -d
   ```

   This command will download the necessary Docker images, create containers, and start the services in detached mode.

4. **Verify the Services:** Check if all the services are up and running:

   ```
   docker-compose ps
   ```

   You should see all services listed as 'running'.

### Accessing the Services

- Kafka Control Center is accessible at [http://localhost:9021](http://localhost:9021).
- Debezium UI is accessible at [http://localhost:8080](http://localhost:8080).
- Postgres is accessible on the default port 5432.

### Shutting Down

To stop and remove the containers, networks, and volumes, run:

```
docker-compose down
```

### Customization

You can modify the Docker Compose file to suit your needs. For example, you might want to persist data in Postgres by adding a volume for the Postgres service.

**Note:** This setup is intended for development and testing purposes. For production environments, consider additional factors like security, scalability, and data persistence.

### Updating Debezium Connector Configuration

To update the existing Debezium connector configuration, execute the following Docker command:

```bash
docker exec -i debezium curl -X PUT -H 'Content-Type: application/json' \
  localhost:8083/connectors/postgres-fin-connector/config --data @- <<EOF
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "postgres",
  "database.port": "5432",
  "database.user": "postgres",
  "database.password": "postgres",
  "database.dbname": "financial_db",
  "plugin.name": "pgoutput",
  "decimal.handling.mode": "string",
  "topic.prefix": "cdc"
}
EOF
```

This command updates the configuration of the existing connector named "postgres-fin-connector". Adjust the JSON data as needed for your specific configuration requirements.


Sure! Here's the continuation of the README.md with the addition of the trigger creation:

### Creating a Trigger Function

To automatically record changes to rows in a table, you can create a trigger function. Here's an example of a trigger function that sets the `modified_by` column to the current user and the `modified_at` column to the current timestamp:

```sql
CREATE OR REPLACE FUNCTION record_change_user()
RETURNS TRIGGER AS $$
BEGIN
  NEW.modified_by := current_user;
  NEW.modified_at := current_timestamp;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

This function can be attached to a table's trigger to record changes made to the table's rows.

### Attaching the Trigger to a Table

Once the trigger function is created, you can attach it to a table's trigger. Here's an example of attaching the `record_change_user()` function to a trigger named `trigger_record_user_update` on a table named `transactions`:

```sql
CREATE TRIGGER trigger_record_user_update
BEFORE UPDATE ON transactions
FOR EACH ROW EXECUTE FUNCTION record_change_user();
```

This trigger will execute the `record_change_user()` function before each update operation on the `transactions` table, automatically updating the `modified_by` and `modified_at` columns.


### Creating a Trigger Function

To automatically record changes to specific columns in a table, you can create a trigger function. Here's an example of a trigger function `record_change_columns()` that captures changes to the `amount` column and updates a `change_info` column with details:

```sql
CREATE OR REPLACE FUNCTION record_change_columns()
RETURNS TRIGGER AS $$
DECLARE
  change_details JSONB;
BEGIN
  change_details := '{}'::JSONB; -- empty JSON object

  -- Check if the amount column has changed
  IF NEW.amount IS DISTINCT FROM OLD.amount THEN
    change_details := jsonb_insert(change_details, '{amount}', jsonb_build_object('old', OLD.amount, 'new', NEW.amount));
  END IF;

  -- Adding the user and timestamp
  change_details := change_details || jsonb_build_object('modified_by', current_user, 'modified_at', now());

  -- Update the change_info column
  NEW.change_info := change_details;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

This function captures changes to the `amount` column and updates a `change_info` column with details about the change, including the old and new values of the `amount` column, the user who made the change, and the timestamp of the change.


#### Attaching the Trigger to a Table

Once the trigger function is created, you can attach it to a table's trigger. Here's an example of attaching the `record_change_columns()` function to a trigger named `trigger_record_change_info` on a table named `transactions`:

```sql
CREATE TRIGGER trigger_record_change_info
BEFORE UPDATE ON transactions
FOR EACH ROW EXECUTE FUNCTION record_change_columns();
```

This trigger will execute the `record_change_columns()` function before each update operation on the `transactions` table, automatically updating the `change_info` column with details about the changes made to the `amount` column.


#### cdc.public.transactions Message: 

```json
{
  "schema": {
    "type": "struct",
    "fields": [<>],
    "optional": false,
    "name": "cdc.public.transactions.Envelope",
    "version": 1
  },
  "payload": {
    "before": {
      "transaction_id": "d8f43e1b-b455-4d06-b584-abde67fb2fbc",
      "user_id": "ysmith",
      "timestamp": 1712661619000000,
      "amount": "663.05",
      "currency": "USD",
      "city": "Dunnchester",
      "country": "Chad",
      "merchant_name": "Jones, King and Jimenez",
      "payment_method": "credit_card",
      "ip_address": "16.221.100.144",
      "voucher_code": "",
      "affiliateid": "a28ff411-757d-4650-93bd-e7e64843ded4",
      "modified_by": "postgres",
      "modified_at": 1712756202812726,
      "change_info": null
    },
    "after": {
      "transaction_id": "d8f43e1b-b455-4d06-b584-abde67fb2fbc",
      "user_id": "ysmith",
      "timestamp": 1712661619000000,
      "amount": "1000",
      "currency": "USD",
      "city": "Dunnchester",
      "country": "Chad",
      "merchant_name": "Jones, King and Jimenez",
      "payment_method": "credit_card",
      "ip_address": "16.221.100.144",
      "voucher_code": "",
      "affiliateid": "a28ff411-757d-4650-93bd-e7e64843ded4",
      "modified_by": "postgres",
      "modified_at": 1712756202812726,
      "change_info": "{\"amount\": {\"new\": 1000, \"old\": 663.05}, \"modified_at\": \"2024-04-10T14:03:26.387745+00:00\", \"modified_by\": \"postgres\"}"
    },
    "source": {
      "version": "2.5.0.Final",
      "connector": "postgresql",
      "name": "cdc",
      "ts_ms": 1712757806395,
      "snapshot": "false",
      "db": "financial_db",
      "sequence": "[\"26959464\",\"26962024\"]",
      "schema": "public",
      "table": "transactions",
      "txId": 767,
      "lsn": 26962024,
      "xmin": null
    },
    "op": "u",
    "ts_ms": 1712757806489,
    "transaction": null
  }
}
```

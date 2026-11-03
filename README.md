# Change Data Capture with Debezium and Kafka

Every insert, update and delete in a MySQL table is captured from the binary log by Debezium, published to Kafka, and shown in a small Flask app as it happens. The whole stack runs from one Docker Compose file.

I built this to understand log-based CDC end to end: how row changes leave the database without touching the application, what the events look like on the topic, and what a consumer has to handle.

## How it works

```
MySQL (binlog, ROW format)
   -> Debezium MySQL connector (Kafka Connect)
   -> Kafka topic demo_server.demo_db.users
   -> Flask consumer: live table of changes, filters and a JSON API
```

Each event carries the operation (`c` create, `u` update, `d` delete, `r` snapshot read), the row before and after the change, and a timestamp. For deletes the consumer shows the `before` image, otherwise the `after` image.

## Services

| Service | Port | What it is for |
| --- | --- | --- |
| MySQL 8.0 | 3307 | Source database with GTID and row-based binlog enabled |
| Zookeeper | 2181 | Coordination for Kafka |
| Kafka | 9092 | Event broker |
| Debezium Connect | 8083 | Runs the MySQL connector |
| Kafdrop | 9000 | Browse topics and messages |
| Debezium UI | 8080 | Check connector status |
| Flask consumer | 5000 | Live view of the changes, plus `/api/events` |

## Run it

Requires Docker.

1. Start the stack:
   ```bash
   docker-compose up -d
   ```
2. Register the connector:
   ```bash
   curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d @connector-config.json
   curl http://localhost:8083/connectors/demo-mysql-connector/status
   ```
3. Create the sample table and give the demo user the replication rights Debezium needs:
   ```bash
   docker exec -it mysql_db mysql -u demo -pdemo -D demo_db -e "CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(255), email VARCHAR(255));"
   docker exec -it mysql_db mysql -uroot -pmysql -e "GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'demo'@'%'; FLUSH PRIVILEGES;"
   ```
4. Make some changes. `python_test.py` runs an insert, an update and a delete with short pauses so you can watch each one arrive:
   ```bash
   pip install mysql-connector-python
   python python_test.py
   ```
5. Open http://localhost:5000 to see the changes, or http://localhost:9000 to look at the raw messages in Kafdrop.

## Files

- `docker-compose.yml`: the seven services above
- `connector-config.json`: Debezium connector for the `demo_db.users` table (`connector-config-all.json` captures every database on the server, under the `demo_server_all` prefix)
- `consumer/`: Flask app that reads the topic and renders the events
- `python_test.py`: insert, update and delete script for testing

The credentials in this repo are local demo values for the containers only.

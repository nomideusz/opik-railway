# Opik's trace store: stock ClickHouse at the version upstream compose pins,
# plus Opik's cluster config with an embedded Keeper (replaces ZooKeeper) and
# one config file that keeps it small (logs to the console, ClickHouse's own
# metric tables off).
FROM clickhouse/clickhouse-server:26.3.16.16-alpine

COPY opik.xml /etc/clickhouse-server/config.d/opik.xml
COPY railway.xml /etc/clickhouse-server/config.d/railway.xml
# The stock file listens on :: and 0.0.0.0; on Railway :: is already
# dual-stack, so the second bind fails with "Address already in use" on every
# boot. Overwritten, not deleted: Railway resurrects files removed in a later layer.
COPY listen.xml /etc/clickhouse-server/config.d/docker_related_config.xml
COPY users.xml /etc/clickhouse-server/users.d/opik.xml

ENV CLICKHOUSE_DB=opik \
    CLICKHOUSE_USER=opik \
    CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1

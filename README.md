# Confluent Real-Time Context Engine Demo


- [Confluent Real-Time Context Engine Demo](#confluent-real-time-context-engine-demo)
  - [Prerequisites](#prerequisites)
  - [Configure Your Environment](#configure-your-environment)
  - [Create or Select Cloud Resources](#create-or-select-cloud-resources)
  - [Create the Topic and Register Its Schema](#create-the-topic-and-register-its-schema)
  - [Produce Avro Records](#produce-avro-records)
  - [Enable the Real-Time Context Engine](#enable-the-real-time-context-engine)
  - [Connect Claude Code](#connect-claude-code)
  - [Check Freshness](#check-freshness)
  - [Query Limitations](#query-limitations)


This demo creates an Avro-backed `orders` topic in Confluent Cloud, enables the Real-Time Context Engine (RTCE), and queries the topic from Claude Code through MCP.

## Prerequisites

- A Confluent Cloud account with an AWS environment and a Basic Kafka cluster, or permission to create them.
- Confluent CLI installed and authenticated.
- `kafka-avro-console-producer` available on your `PATH` (from Confluent Platform tools).
- Claude Code installed and authenticated.
- Schema Registry enabled for the environment.

## Configure Your Environment

Copy the sample file and replace every placeholder with your own Confluent Cloud values:

```sh
cp .env.sample .env
```

Use separate credentials for the different services:

- `API_KEY` and `API_SECRET`: a Global API key for the RTCE MCP endpoint. Create it from the Confluent Cloud Console under **Administration > API keys**, selecting **Global**. RTCE's get-started guide uses a Global API key for MCP authentication.
- `KAFKA_API_KEY` and `KAFKA_API_SECRET`: an API key scoped to your Kafka cluster. Basic clusters require a resource-scoped key for Kafka access.
- `SCHEMA_REGISTRY_API_KEY` and `SCHEMA_REGISTRY_API_SECRET`: an API key scoped to your environment's Schema Registry. Use a resource-scoped key for a public Schema Registry endpoint.
- `BOOTSTRAP_SERVER` and `SCHEMA_REGISTRY_URL`: the endpoints for the Kafka cluster and Schema Registry, respectively.
- `ORG_ID`, `ENV_ID`, `LKC_ID`, `AWS_REGION`, `TOPIC`, and `MCP_ENDPOINT`: the organization, environment, cluster, AWS region, topic, and RTCE endpoint values for your setup.

Load your local values into the current zsh session:

```sh
set -a
source .env
set +a
```

## Create or Select Cloud Resources

Log in and select your environment. Replace the sample values in `.env` before running commands that use them.

```sh
confluent login
confluent environment list
confluent environment use "$ENV_ID"
```

If you need a Basic Kafka cluster, create one in the selected environment:

```sh
confluent kafka cluster create \
	orders-demo \
	--type basic \
	--cloud aws \
	--region "$AWS_REGION"
```

After creation, update `.env` with the new cluster ID, bootstrap server, and resource-scoped Kafka API key and secret, then load it again. Make sure Schema Registry is enabled in the same environment and set its URL and resource-scoped API credentials in `.env` as well.

## Create the Topic and Register Its Schema

Create the topic:

```sh
confluent kafka topic create "$TOPIC" --cluster "$LKC_ID"
```

Register the value schema under the conventional `<topic>-value` subject:

```sh
confluent schema-registry schema create \
	--subject "$TOPIC-value" \
	--schema schema.avsc \
	--type AVRO
```

The schema is in [schema.avsc](schema.avsc).

## Produce Avro Records

Start the producer:

```sh
kafka-avro-console-producer \
	--bootstrap-server "$BOOTSTRAP_SERVER" \
	--topic "$TOPIC" \
	--command-property security.protocol=SASL_SSL \
	--command-property sasl.mechanism=PLAIN \
	--command-property sasl.jaas.config="org.apache.kafka.common.security.plain.PlainLoginModule required username=\"$KAFKA_API_KEY\" password=\"$KAFKA_API_SECRET\";" \
	--reader-property schema.registry.url="$SCHEMA_REGISTRY_URL" \
	--reader-property basic.auth.credentials.source=USER_INFO \
	--reader-property basic.auth.user.info="$SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET" \
	--reader-property value.schema="$(cat schema.avsc)"
```

Paste each record on its own line, then press `Ctrl+D` to finish:

```json
{"order_id":"o-1001","customer_id":"c-001","status":"SHIPPED","total":125.50}
{"order_id":"o-1002","customer_id":"c-002","status":"PENDING","total":89.00}
{"order_id":"o-1003","customer_id":"c-001","status":"SHIPPED","total":240.75}
```

## Enable the Real-Time Context Engine

In the Confluent Cloud Console:

1. Select **Environments**, then your environment and Kafka cluster.
2. Open **Topics** and find your topic.
3. Scroll to the **Context engine** column and open the topic's details.
4. Turn on the Real-Time Context Engine and wait for its status to become **On**.
5. Copy the MCP endpoint from the topic details and set `MCP_ENDPOINT` in `.env`.

The endpoint includes your organization, environment, cluster, and region. After updating `.env`, reload it in the terminal with the `source .env` commands above.

## Connect Claude Code

Create a Base64 token from the Global API key used for RTCE, then register the MCP server with Claude Code:

```sh
RTCE_TOKEN="$(printf '%s' "$API_KEY:$API_SECRET" | base64 | tr -d '\n')"

claude mcp add --transport http \
	confluent-rtce \
	"$MCP_ENDPOINT" \
	--header "Authorization: Basic $RTCE_TOKEN"
```

Start Claude Code:

```sh
claude
```

The RTCE MCP server provides `listTopics`, `getMetadata`, and `queryData` tools. Try these prompts in Claude Code:

```text
Use the Confluent Real-Time Context Engine listTopics tool. List the available topics and confirm that orders is present.

Use getMetadata for the orders topic. Return the columns, data types, and primary-key information.

Use queryData on orders. Return order_id, customer_id, status, and total where status equals SHIPPED. Order by total descending and return at most 10 rows. Set max_result_rows to 10.

Use queryData on orders. Return orders where total is between 100 and 300. Return order_id and total, ordered by total descending. Set max_result_rows to 10.
```

## Check Freshness

Run the producer again and publish a new record:

```json
{"order_id":"o-1004","customer_id":"c-003","status":"SHIPPED","total":310.00}
```

Then ask Claude Code:

```text
Use queryData on orders to find order_id o-1004 and return the complete row.
```

## Query Limitations

Per the current demo notes, RTCE `queryData` does not support aggregates, `GROUP BY`, joins, subqueries, set operations, DDL, or DML. Avoid aggregate-count prompts in this demo.

For later CLI experiment (not yet available).

```sh
confluent lightning query \
	--cluster "$LKC_ID" \
	--sql "SELECT ORDER_ID, STATUS, TOTAL FROM $TOPIC WHERE STATUS = 'SHIPPED' ORDER BY TOTAL DESC LIMIT 10"
```

For the supported MCP workflow, see the [RTCE get-started guide](https://docs.confluent.io/cloud/current/ai/real-time-context-engine/get-started.html) and [query documentation](https://docs.confluent.io/cloud/current/ai/real-time-context-engine/query-data.html).

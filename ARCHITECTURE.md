# Architecture Overview

This document provides an overview of the architecture of the project, outlining its main components, their interactions, and the overall design principles.

## Components
1. **Data Ingestion Layer**. Responsible for collecting data from various sources, including marketplaces and social media platforms.
  - **RabbitMQ**: A message broker that manages task queues for data collection requests.
  - **Kafka**: A distributed streaming platform used for handling real-time data feeds from crawlers. Any result related topic automatically sinks to ClickHouse for storage and further analysis.
  - **TaskManager**: Orchestrates the scheduling and execution of data collection tasks.
    * registers task in RabbitMQ
    * starts CrawlerCluster nodes to execute tasks
    * monitors task progress and handles retries/failures
    * listens extracted results of tasks and registers new tasks if needed
  - **CrawlerCluster**: A distributed system of crawler nodes that perform the actual data extraction
    * each node has a headless browser instance to navigate and scrape web pages
    * depending on the task type, different scraping strategies are employed (e.g., DOM parsing, API calls)
    * pushes raw data to Kafka for further processing (ETL layer is separately implemented due to has an advantage of scalability and fault-tolerance)
    * scales horizontally by adding more nodes to handle increased load
2. **ETL Layer**. Processes raw data from the ingestion layer, transforming it into structured formats suitable for analysis.
  - **Redis**: Used for caching canonical data and dictionaries (stop words, standardize patterns) to speed up processing.
  - **DataProcessor**: A set of services that perform data cleaning, normalization, and enrichment.
    * consume raw data from Kafka topics
    * extract data required by task types
    * clean and normalize data (e.g., remove duplicates, standardize formats)
    * enrich data with additional metadata (e.g., timestamps, source identifiers)
    * enrich data with embeddings using pre-trained models
    * commit processed data to Kafka topics for storage and analysis
    * scales horizontally to handle large volumes of data
3. **Monitoring & Alerting**. Ensures system health and performance.
  - **Prometheus**: Collects metrics from various components for monitoring.
  - **Grafana**: Visualizes metrics and provides dashboards for real-time monitoring.
  - **Alertmanager**: Sends alerts based on predefined thresholds to notify the team of any issues.
  - **Sentry**: Monitors application errors and exceptions, providing insights into system reliability.

---

## Design Principles
- **Scalability**: The architecture is designed to scale horizontally, allowing for the addition of more nodes in the CrawlerCluster and DataProcessor components as demand increases.
- **Modularity**: Each component is designed to be independent, allowing for easier maintenance and updates without affecting the entire system.
- **Fault Tolerance**: The use of message brokers like RabbitMQ and Kafka ensures that data is not lost in case of component failures, and tasks can be retried as needed.

---

## RabbitMQ consumer behaviour

This project uses RabbitMQ for task delivery and a worker pattern where each crawler node consumes tasks one-by-one and only acknowledges (acks) a message after the task has completed successfully and downstream durable side-effects are confirmed. The following section documents the recommended state model, delivery guarantees, retry behaviour, and operational guidance so the architecture and TaskManager are aligned with that runtime pattern.

### RabbitMQ consumer settings and guarantees
- Use durable queues and persistent messages (delivery_mode=2) so messages survive broker restarts.
- Use manual acknowledgements: workers must call `basic.ack` only after all durable downstream side-effects (DB writes, Kafka publish with confirms, files stored) are successful.
- Set `prefetch_count=1` (basic.qos) on worker channels to ensure each worker receives and processes one message at a time.
- Use publisher confirms on the TaskManager/producer side to ensure the task message is durably stored.
- Unacked messages are requeued by RabbitMQ if the consumer connection/channel closes; design for at-least-once delivery and deduplicate downstream side-effects.

---

## TaskManager
The TaskManager is responsible for orchestrating the scheduling and execution of data collection tasks. It interacts with RabbitMQ to register tasks, starts CrawlerCluster nodes to execute them, monitors progress, and handles retries or failures.

### Task Lifecycle
1. **Task Registration**: The TaskManager registers a new task by publishing a message to a RabbitMQ queue.
2. **Task Execution**: The TaskManager monitors the CrawlerCluster status and if there are available nodes, it starts a CrawlerCluster node to pick up and execute the task.
3. **Progress Monitoring**: The TaskManager listens for task progress updates from the CrawlerCluster nodes.
4. **Completion Handling**: Upon task completion, the TaskManager verifies the results and acknowledges the task in RabbitMQ.
5. **Failure Handling**: If a task fails, the TaskManager handles retries based on predefined policies and re-registers the task if necessary.

---

## CrawlerCluster
The CrawlerCluster is a distributed system of crawler nodes that perform the actual data extraction. Each node is equipped with a headless browser instance to navigate and scrape web pages.

### Crawler Node Architecture

Crawler Node is a dockerized service that includes the following components:
1. **Headless Browser**: Each crawler node runs a headless browser (e.g., LightPanda) to load and interact with web pages.
2. **Node.js Application**: The main application logic is implemented in Node.js, which handles task consumption, and web scraping
3. **Task Status**: disposes task status updates to RabbitMQ so TaskManager can monitor progress.
4. **Data Extraction**: Depending on the task type, different scraping strategies are employed (e.g., DOM parsing, API calls) to extract the required data.
5. **Data Publishing**: The extracted raw data is pushed to Kafka for further processing in the ETL layer.
6. **Scalability**: The CrawlerCluster can scale horizontally by adding more nodes to handle increased load.
7. **Error Handling**: Each crawler node implements error handling mechanisms to manage issues such as timeouts, navigation errors, and data extraction failures.
8. **Logging & Monitoring**: Crawler nodes log their activities and expose metrics for monitoring via Prometheus.
9. **Configuration Management**: Crawler nodes can be configured dynamically based on task requirements, allowing for flexibility in scraping strategies and resource allocation. Common configurations are passed via etcd or environment variables.

---

## DataProcessor
The DataProcessor is a set of services that perform data cleaning, normalization, and enrichment on the raw data collected by the CrawlerCluster.

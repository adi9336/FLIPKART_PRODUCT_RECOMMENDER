# Flipkart Product Recommender

An LLM-powered Flipkart product review chatbot built with Flask, LangChain, Astra DB, Groq, Docker, Prometheus, and Grafana.

The application answers product-related questions using Flipkart review data. It converts product reviews into LangChain documents, stores/retrieves embeddings from Astra DB, generates responses through Groq, and exposes Prometheus metrics for monitoring.

## What This Project Implements

- Flask web chatbot UI for asking product questions.
- Retrieval-Augmented Generation (RAG) pipeline with LangChain.
- Review ingestion from `data/flipkart_product_review.csv`.
- HuggingFace embedding model: `BAAI/bge-base-en-v1.5`.
- Astra DB vector store collection: `flipkart_database`.
- Groq chat model: `llama-3.1-8b-instant`.
- Conversational memory using LangChain chat history.
- Prometheus metrics endpoint at `/metrics`.
- Docker containerization with `Dockerfile`.
- Kubernetes deployment for the Flask app.
- Kubernetes monitoring stack with Prometheus and Grafana manifests.

## Project Structure

```text
.
|-- app.py                              # Flask app, chatbot routes, metrics endpoint
|-- Dockerfile                          # Docker image definition
|-- flask-deployment.yaml               # Kubernetes deployment/service for Flask app
|-- requirements.txt                    # Python dependencies
|-- setup.py                            # Editable package setup
|-- data/
|   `-- flipkart_product_review.csv     # Product review dataset
|-- flipkart/
|   |-- config.py                       # Environment-based app configuration
|   |-- data_converter.py               # CSV to LangChain documents
|   |-- data_ingestion.py               # Astra DB vector store ingestion
|   `-- rag_chain.py                    # History-aware RAG chain
|-- prometheus/
|   |-- prometheus-configmap.yaml       # Prometheus scrape config
|   `-- prometheus-deployment.yaml      # Prometheus deployment/service
|-- grafana/
|   `-- grafana-deployment.yaml         # Grafana deployment/service
|-- templates/
|   `-- index.html                      # Chat UI
`-- static/
    `-- style.css                       # Chat UI styling
```

## Environment Variables

Create a `.env` file locally or configure these values as Kubernetes secrets:

```env
ASTRA_DB_API_ENDPOINT=your_astra_db_api_endpoint
ASTRA_DB_APPLICATION_TOKEN=your_astra_db_application_token
ASTRA_DB_KEYSPACE=your_astra_db_keyspace
GROQ_API_KEY=your_groq_api_key
```

The Flask Kubernetes manifest expects a secret named `llmops-secrets`.

Example:

```bash
kubectl create secret generic llmops-secrets \
  --from-literal=ASTRA_DB_API_ENDPOINT="your_astra_db_api_endpoint" \
  --from-literal=ASTRA_DB_APPLICATION_TOKEN="your_astra_db_application_token" \
  --from-literal=ASTRA_DB_KEYSPACE="your_astra_db_keyspace" \
  --from-literal=GROQ_API_KEY="your_groq_api_key"
```

## Run Locally

Install dependencies:

```bash
pip install -e .
```

Start the Flask app:

```bash
python app.py
```

Open:

```text
http://localhost:5000
```

Metrics endpoint:

```text
http://localhost:5000/metrics
```

## Docker

Build the Docker image:

```bash
docker build -t flask-app:latest .
```

Run the container:

```bash
docker run --env-file .env -p 5000:5000 flask-app:latest
```

Open the chatbot:

```text
http://localhost:5000
```

## Kubernetes Deployment

Build the image before applying the Flask manifest. For local Kubernetes clusters such as Minikube, make sure the image is available inside the cluster.

Apply the Flask app deployment:

```bash
kubectl apply -f flask-deployment.yaml
```

Check pods and services:

```bash
kubectl get pods
kubectl get svc
```

The Flask service is configured as a `LoadBalancer` service that forwards port `80` to container port `5000`.

## Prometheus Monitoring

The Flask app exposes metrics at:

```text
/metrics
```

The current metric implemented is:

```text
http_requests_total
```

Prometheus configuration lives in:

```text
prometheus/prometheus-configmap.yaml
```

Create the monitoring namespace:

```bash
kubectl create namespace monitoring
```

Deploy Prometheus:

```bash
kubectl apply -f prometheus/prometheus-configmap.yaml
kubectl apply -f prometheus/prometheus-deployment.yaml
```

Access Prometheus through the configured NodePort:

```text
http://<node-ip>:32001
```

In Prometheus, query:

```promql
http_requests_total
```

Note: `prometheus/prometheus-configmap.yaml` currently scrapes the Flask app from a hardcoded target:

```text
35.255.171.200:5000
```

Update this target if your Flask app runs at a different IP, port, or Kubernetes service address.

## Grafana Dashboard

Deploy Grafana:

```bash
kubectl apply -f grafana/grafana-deployment.yaml
```

Access Grafana through the configured NodePort:

```text
http://<node-ip>:32000
```

Default Grafana login:

```text
username: admin
password: admin
```

Add Prometheus as a Grafana data source:

```text
http://prometheus-service.monitoring.svc.cluster.local:9090
```

Then create a dashboard panel using this PromQL query:

```promql
http_requests_total
```

## Application Flow

1. User opens the Flask chatbot UI at `/`.
2. User submits a question through the chat input.
3. Flask sends the question to the LangChain RAG chain.
4. The RAG chain retrieves relevant Flipkart review context from Astra DB.
5. Groq generates a concise product-related answer.
6. The response is returned to the web UI.
7. Prometheus scrapes `/metrics`.
8. Grafana visualizes Prometheus metrics.

## Useful Commands

```bash
kubectl get pods
kubectl get svc
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl logs deployment/flask-app
kubectl logs deployment/prometheus -n monitoring
kubectl logs deployment/grafana -n monitoring
```

## Notes

- Keep `.env` out of version control because it contains API keys and tokens.
- If running in Kubernetes, configure secrets before starting the Flask deployment.
- If Prometheus does not show Flask metrics, verify that the Flask app is reachable from the Prometheus pod and update the scrape target.

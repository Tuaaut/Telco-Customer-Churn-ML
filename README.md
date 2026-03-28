# Telco Customer Churn ML
End-to-End ML Project with Real Deployment in AWS

## Purpose

Build and ship a full machine-learning solution for predicting customer churn in a telecom setting—from data prep and modeling to an API + web UI deployed on AWS.

### Problem solved & benefits

- Faster decisions: Predicts which customers are likely to churn so teams can act before they leave.
- Operationalized ML: Model is accessible via a REST API and a simple UI; anyone can test it without notebooks.
- Repeatable delivery: CI/CD + containers mean every change can be rebuilt, tested, and redeployed in a consistent way.
- Traceable experiments: MLflow tracks runs, metrics, and artifacts for reproducibility and auditing.

### What I built

- Data & Modeling: Feature engineering + XGBoost classifier; experiments logged to MLflow.
- Model tracking: Runs, metrics, and the serialized model logged under a named MLflow experiment.
- Inference service: FastAPI app exposing /predict (POST) and a root health check /.
- Web UI: Gradio interface mounted at /ui for quick, shareable manual testing.
- Containerization: Docker image with uvicorn entrypoint (src.app.main:app) listening on port 8000.
- CI/CD: GitHub Actions builds the image and pushes to Docker Hub; optionally triggers an ECS service update.
- Orchestration: AWS ECS Fargate runs the container (serverless).
- Networking: Application Load Balancer (ALB) on HTTP:80 forwarding to a Target Group (IP targets on HTTP:8000).
- Security: Security groups scoped to allow ALB inbound 80 from the internet, and task inbound 8000 from the ALB SG.
- Observability: CloudWatch Logs for container stdout/stderr and ECS service events.

### Folder Structure

```
Telco-Customer-Churn-ML/
├── notebooks/          # Exploratory data analysis (EDA.ipynb)
├── scripts/            # Pipeline and test scripts
│   ├── run_pipeline.py                        # Main ML training pipeline
│   ├── prepare_processed_data.py              # Data preparation helper
│   ├── test_pipeline_phase1_data_features.py  # Tests for data & feature engineering
│   ├── test_pipeline_phase2_modeling.py       # Tests for model training & evaluation
│   └── test_fastapi.py                        # Tests for FastAPI endpoints
└── src/                # Application source code
    ├── app/            # FastAPI + Gradio application entry points
    │   ├── main.py     # Primary app (FastAPI + Gradio mounted at /ui)
    │   └── app.py      # Alternative app entry point
    ├── data/           # Data loading and preprocessing
    │   ├── load_data.py
    │   └── preprocess.py
    ├── features/       # Feature engineering
    │   └── build_features.py
    ├── models/         # Model training, tuning, and evaluation
    │   ├── train.py
    │   ├── tune.py
    │   └── evaluate.py
    ├── serving/        # Model inference and serving artifacts
    │   ├── inference.py
    │   └── model/      # Local MLflow run copies for development
    └── utils/          # Shared utilities and data validation
        ├── utils.py
        └── validate_data.py
```

### Deployment flow (high-level)

- Push to main → GitHub Actions builds the Docker image and pushes it to Docker Hub.
- ECS service is updated (manually or via the workflow) to force a new deployment.
- ALB health checks hit / on port 8000; once healthy, traffic is routed to the new task.
- Users call POST /predict or open the Gradio UI at /ui via the ALB DNS.

### Roadblocks & how we solved them

Unhealthy targets behind ALB

- Cause: App didn't respond at the health-check path; listener/target port mismatches.
- Fixes: Added GET / health endpoint; confirmed ALB listener on 80 forwards to TG on 8000; TG health check path set to /.

Module import error in container (ModuleNotFoundError: serving)

- Cause: Python path in the image didn't include src/.
- Fixes: Set PYTHONPATH=/app/src in the Dockerfile; corrected uvicorn app path to src.app.main:app.

ALB DNS timing out

- Cause: Security group rules not aligned with traffic flow.
- Fixes: ALB SG allows inbound 80 from 0.0.0.0/0; task SG allows inbound 8000 from the ALB SG; outbound open.

ECS redeploy not picking up the new image

- Cause: Service still running previous task definition.
- Fixes: Force new deployment (CLI or console) after pushing the new image; optional step added to CI.

Gradio UI error ("No runs found in experiment")

- Cause: Inference/UI expected an MLflow-logged model but couldn't resolve a run.
- Fixes: Standardized MLflow experiment name and model logging in training; inference loads the logged model consistently (and a local path for dev).

Local testing vs. prod paths

- Cause: MLflow artifact URIs differ locally vs. in container.
- Fixes: For local dev, load via direct ./mlruns/.../artifacts/model; in prod, container loads the packaged model path used at build time.

# AI Sales Prediction

Sales forecasting for 86 products in 13 categories. Python (FastAPI) backend with a machine-learning model, Postgres database, and a 3D-styled frontend: info page, dashboard, and one animated page per category (no photos).

```
ai-sales-prediction/
├── backend/            FastAPI + SQLAlchemy + scikit-learn
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app/main.py     API endpoints, agent, retraining
│   ├── app/ml.py       model (HistGradientBoostingRegressor)
│   ├── app/db.py       tables + first-run seeding
│   └── data/products.csv   the 86-product catalog
├── frontend/           single-page site served by nginx
│   ├── Dockerfile
│   ├── nginx.conf      proxies /api to the backend
│   └── index.html
├── db/schema.sql       reference schema (created automatically)
├── docker-compose.yml  local run: db + backend + frontend
└── deploy/aws/         ECR push script + ECS Fargate task definition
```

## Run locally

```bash
cp .env.example .env        # change the password
docker compose up --build
```

Open http://localhost:8080. The first start creates the tables and loads the catalog.

## How the forecasting works

- The catalog gives each product a typical monthly volume, a price, and a seasonal pattern (festive peak Oct to Dec) and region index.
- The model is trained on the catalog with simulated variation in price, discount, ad spend, month and region. It is not trained on real history until you add some.
- Save real monthly sales (dashboard, "Your sales data": one entry, or a CSV with `product,month,units` where month looks like `2026-03`). The product's volume then comes from your real numbers and the model retrains in the background.
- The more real months you save, the closer the forecasts get. Targets stay at the sample numbers.

## API

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/health` | liveness check |
| GET | `/api/products` | catalog with current base volumes |
| POST | `/api/sales` | save one month `{product, month, units}` |
| POST | `/api/sales/bulk` | save many rows `{rows:[...]}` |
| DELETE | `/api/sales/{product}` | clear a product's saved sales |
| POST | `/api/predict` | `{product, month 0-11, region, price, discount, ad_spend}` |
| POST | `/api/agent` | `{q, price?, region}` forecast for any product name |

Interactive docs: `/docs` on the backend (port 8000, not exposed by compose by default).

## Deploy on AWS (ECS Fargate + RDS Postgres)

1. **Database.** Create an RDS PostgreSQL instance in private subnets. Put the connection string in Secrets Manager as `sales/database-url`, in the form `postgresql://USER:PASSWORD@HOST:5432/DBNAME`.
2. **Images.** `AWS_REGION=ap-south-1 AWS_ACCOUNT_ID=<id> ./deploy/aws/deploy.sh` pushes `sales-backend` and `sales-frontend` to ECR.
3. **Task.** Replace the `<ACCOUNT_ID>` and `<REGION>` placeholders in `deploy/aws/ecs-taskdef.json`, create the log group `/ecs/sales-prediction`, then `aws ecs register-task-definition --cli-input-json file://deploy/aws/ecs-taskdef.json`.
4. **Service.** Create an ECS Fargate service from that task with an Application Load Balancer pointing at container `frontend`, port 80. Both containers share one task, so nginx reaches the backend on `127.0.0.1:8000`.
5. **Network.** Allow the task's security group to reach the RDS security group on port 5432, and the ALB to reach the task on port 80.

## Before real use

- There is no login. Anyone who can open the site can save or clear sales. Add authentication (for example Cognito on the ALB) before exposing it publicly.
- Use HTTPS on the load balancer (ACM certificate).
- Run a single backend task: the model is held in memory and retrained per task.

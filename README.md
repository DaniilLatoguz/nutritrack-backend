# NutriTrack Backend

REST API for a nutrition and calorie tracking service.

## About this repository

The project is built task by task, in the order a real backend would grow.
Architectural patterns are introduced only after the problem they solve has
been encountered — so early commits deliberately contain code without layers
or abstractions that are not yet justified.

Every significant decision is recorded in [`docs/architecture.md`](docs/architecture.md)
as an ADR, including the ones that were deliberately postponed.

## Stack

- Python 3.14
- FastAPI
- Uvicorn

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

## Run

```bash
uvicorn app.main:app --reload
```

- API docs: http://127.0.0.1:8000/docs
- Health check: http://127.0.0.1:8000/health

## Documentation

- [`docs/architecture.md`](docs/architecture.md) — architecture decision log
- [`docs/product-model.md`](docs/product-model.md) — domain model and endpoint contract
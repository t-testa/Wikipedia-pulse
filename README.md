# Wikipedia Pulse

A Python event-processing prototype that collects English Wikipedia changes, queues them in RabbitMQ, stores them through SQLAlchemy and exposes a small Flask report.

```text
Wikimedia EventStreams → collector → RabbitMQ → consumer → SQLite → report / analyzer
```

## What it does

- Filters the live Wikimedia stream to English Wikipedia and publishes up to 20 events per collector run by default.
- Uses a durable RabbitMQ queue and persistent messages.
- Acknowledges consumed messages after a database commit; requeues processing failures.
- Reports stored event counts, distinct pages, bot share, and top pages/editors.
- Provides `/health`, an HTML `/report`, and JSON request counters at `/metrics`.

## Local setup

The CI workflow uses Python 3.10. Install and start RabbitMQ locally with its default local connection settings, then run these commands from the repository root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m flask --app src/app.py db upgrade
python -m flask --app src/app.py run
```

Open http://127.0.0.1:5000/report. In separate terminals using the same environment and repository directory:

```powershell
python -m src.consumer
python -m src.collector
```

The consumer waits for events until interrupted. The collector requires internet access. For a terminal report:

```powershell
python -m src.analyzer
```

The default SQLite database is `src/instance/wikipedia_pulse.db`. An optional `DATABASE_URL` overrides it; every process must use the same database configuration.

## Tests

The repository's CI applies migrations and runs:

```powershell
python -m pytest -v
```

## Repository map

| Path | Purpose |
|---|---|
| `src/collector.py` | Wikimedia stream reader and RabbitMQ publisher |
| `src/consumer.py` | Queue consumer and database writes |
| `src/app.py` | Flask app, database model, report and request metrics |
| `src/analyzer.py` | Terminal analysis of stored events |
| `migrations/` | Database schema migrations |
| `tests/` | Collector and Flask tests |

## Prototype limitations

The collector samples English Wikipedia change events, not all Wikipedia activity, and does not filter solely to edit-type events. There is no event-ID deduplication; redelivery can create duplicate records. Failed messages are requeued without a retry limit or dead-letter queue. RabbitMQ settings are hard-coded to localhost, and the web UI includes an early echo form.

The report interpolates event text into HTML without explicit escaping. Keep this prototype local until output escaping and deployment hardening are addressed. No production availability or end-to-end runtime result is claimed by this README.

# Real-time Network Intrusion Detection Pipeline

**Work in progress** — Data Engineering Bootcamp final project (Veri Bilimi Okulu, 2026–2027)

A streaming security analytics pipeline that replays real network traffic from the **CIC-IDS2017** dataset through **Kafka**, detects attacks such as **brute force, port scans and DDoS** with rule-based **Spark Structured Streaming** jobs, and visualizes alerts in **Elasticsearch + Kibana**. Detection quality is evaluated against the dataset's ground-truth labels.


---

## Architecture

```
│  CSV replay  │───▶│  Kafka  │───▶│ Spark Structured │───▶│ Elasticsearch │───▶│  Kibana  │                   
```

1. **Producer** reads cleaned CIC-IDS2017 flow records, sorts them by event time and publishes them to Kafka as JSON.
2. **Spark Structured Streaming** consumes the topic, applies the same cleaning logic as the batch analysis and runs time-windowed detection rules.
3. **Elasticsearch** stores raw flows and generated alerts in separate indices.
4. **Kibana** provides dashboards for analysts: alert timeline, top offending IPs, attack-type distribution.

## Dataset

[CIC-IDS2017](https://www.unb.ca/cic/datasets/ids-2017.html) — Canadian Institute for Cybersecurity, University of New Brunswick. Five days of realistic network traffic with labelled attacks.

This project uses the **GeneratedLabelledFlows** CSVs, which include source/destination IPs and timestamps (the ML-ready `MachineLearningCSV` version drops these columns, which time-windowed detection rules require).


## Data quality findings

Full analysis: [`notebooks/01_data_quality.ipynb`](notebooks/01_data_quality.ipynb)

Before writing any detection rule, the dataset was profiled with Spark. Key findings:

- **Timestamps lack AM/PM.** Afternoon hours are stored in 12-hour format (`1:00` = 13:00) in all three files. This was verified, not assumed: attack labels line up with the official attack schedule only when hours 1–5 are read as 13–17. Fix: if hour < 8, add 12 hours.
- **TCP flag columns are unreliable.** `SYN Flag Count` is identical to `Fwd PSH Flags` in every row, which is impossible for real traffic (a known CICFlowMeter bug). Detection rules therefore rely on **behaviour** rather than flags.
- **All NaN/Infinity values share one root cause:** division by zero in zero-duration flows (`Flow Bytes/s`, `Flow Packets/s`).
- **Rare impossible values (<0.1% of rows):** 55 negative flow durations; overflow errors (huge negative values) in header length and `min_seg_size_forward` columns.
- **Redundant columns:** 10 constant (all-zero) columns, a duplicated `Fwd Header Length` column, and several exact duplicates.
- **Other checks:** column names contain leading spaces; `Flow ID` is not unique; 10,654 distinct IPs, all valid IPv4; labels are clean; 7 exact duplicate rows.
- **Class balance varies strongly by file** (3% vs ~56% attacks), so evaluation metrics will be reported per file / attack type.

**Cleaning decisions:** strip column names, parse and fix timestamps into `event_time`, drop rows with negative duration and exact duplicates, drop constant/duplicate/unreliable columns, and avoid the provided rate columns.

## Detection rules (planned)

| Rule | Logic (per source IP, time window) | Target label |
|---|---|---|
| Brute force | Many short connections to port 21/22 | FTP-Patator, SSH-Patator |
| Port scan | High number of distinct destination ports | PortScan |
| DoS / DDoS | Sudden spike in flows toward one destination | DDoS |

Alerts will be evaluated against ground-truth labels (precision, recall) per file and attack type.

## Tech stack

Kafka 4.1 · Spark 4.1 (PySpark, Structured Streaming) · Elasticsearch 9.1 · Kibana 9.1 · Docker Compose · JupyterLab

## Project structure

```
security-streaming-pipeline/
├── data/                         # CIC-IDS2017 CSVs (not committed)
├── notebooks/
│   └── 01_data_quality.ipynb     # Data profiling, findings and cleaning decisions
├── producer/                     # Kafka replay producer
├── spark_jobs/                   # Streaming cleaning + detection jobs
├── kibana/                       # Exported dashboards
├── docker-compose.yml
└── README.md
```


## Getting started

### 1. Get the data

1. Download `GeneratedLabelledFlows.zip` from the [official dataset page](https://www.unb.ca/cic/datasets/ids-2017.html).
2. Extract these three CSV files into `data/`:
   - `Tuesday-WorkingHours.pcap_ISCX.csv`
   - `Friday-WorkingHours-Afternoon-PortScan.pcap_ISCX.csv`
   - `Friday-WorkingHours-Afternoon-DDos.pcap_ISCX.csv`.

> **Windows note:** the folder inside the zip is named `TrafficLabelling ` with a trailing space, which Windows Explorer and PowerShell cannot extract. Extract the needed files directly with Python instead:
>
> ```bash
> python -c "import zipfile; z=zipfile.ZipFile('GeneratedLabelledFlows.zip'); [open('data/'+n.split('/')[-1],'wb').write(z.read(n)) for n in z.namelist() if n.split('/')[-1].startswith(('Tuesday','Friday-WorkingHours-Afternoon'))]"
> ```

### 2. Start the services

```bash
docker compose up -d
```

| Service | URL |
|---|---|
| JupyterLab (Spark) | http://localhost:8888 |
| Spark UI | http://localhost:4040 |
| Elasticsearch | http://localhost:9200 |
| Kibana | http://localhost:5601 |

Kafka is available at `localhost:9092` from the host and `kafka:29092` from other containers. The project folders are mounted into the Spark container under `/opt/project`.

> Allocate at least 8 GB of memory to Docker. Security is disabled for Elasticsearch and JupyterLab runs without a token — **local development only**.

## Roadmap

- [x] Project setup and Docker infrastructure (Kafka, Spark, Elasticsearch, Kibana)
- [x] Data quality analysis and cleaning decisions
- [ ] Cleaning notebook and reusable cleaning function
- [ ] Detection rule prototyping and threshold selection (batch)
- [ ] Kafka replay producer
- [ ] Spark streaming job: Kafka → Elasticsearch
- [ ] Streaming detection rules: brute force, port scan, DDoS
- [ ] Kibana dashboards
- [ ] Evaluation against ground-truth labels

## Acknowledgements

- Spark image (`veribilimiokulu/pyspark`) and parts of the Kafka configuration are based on the Veri Bilimi Okulu Data Engineering Bootcamp environment.
- Dataset: Iman Sharafaldin, Arash Habibi Lashkari, and Ali A. Ghorbani, "Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization", *ICISSP*, Portugal, January 2018.

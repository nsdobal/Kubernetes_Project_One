# Axion Telemetry Query Service

RESTful API providing access to historic metrics and active alarms for the UI and AI agents.

## Tech Stack
- Python FastAPI
- TimescaleDB (PostgreSQL)

## Features
- Exposes endpoints to retrieve time-series aggregations (e.g., average vibration over the last 15 minutes).
- Checks active bounds and returns triggered alarms.

## Running on a Virtual Machine (VM)

Follow these steps to deploy and run the service on a VM so it is accessible via a public IP address on port `8000`.

### Prerequisites
1. A Virtual Machine (e.g., AWS EC2, Azure VM, GCP Compute Engine).
2. Python 3.9+ installed on the VM.
3. Network Configuration: Ensure your VM's firewall or security group allows inbound traffic on **TCP port 8000**.

### Deployment Steps

1. **Clone or copy the project** to your VM:
   ```bash
   git clone <your-repo-url>
   cd axion-telemetry-query-service
   ```

2. **Set up a virtual environment** (recommended):
   ```bash
   python3 -m venv venv
   source venv/bin/activate  # On Windows use: venv\Scripts\activate
   ```

3. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

4. **Set up environment variables** (if needed by `config.py` / `settings`), such as database connection strings.

5. **Run the application**:
   To make the application accessible from the outside (via public IP), you must bind it to `0.0.0.0`:
   ```bash
   uvicorn main:app --host 0.0.0.0 --port 8000
   ```
   *Note: For a production deployment, consider using a process manager like `systemd` or a containerized approach (e.g., Docker).*

### Accessing the API

Once the service is running, you can access it from any browser or HTTP client using your VM's public IP address:

- **API Root**: `http://<YOUR_VM_PUBLIC_IP>:8000/`
- **Swagger UI Documentation**: `http://<YOUR_VM_PUBLIC_IP>:8000/docs`
- **Redoc Documentation**: `http://<YOUR_VM_PUBLIC_IP>:8000/redoc`

Replace `<YOUR_VM_PUBLIC_IP>` with the actual public IP address of your virtual machine.

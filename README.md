doc-ray
=======

This project implements an asynchronous document parsing service using Ray Serve. Users can submit documents, poll for parsing status, and retrieve results. It's designed for both standalone local development and cluster deployment.

## Features

- **Asynchronous API**: Submit, status, and result endpoints.
- **Scalable**: Built on Ray Serve, allowing for scaling from a single machine to a cluster.
- **Containerized**: Dockerfile provided for easy building and deployment.
- **Modern Tooling**: Uses `uv` for package management.

## Prerequisites

- Python 3.10+
- [uv](https://github.com/astral-sh/uv) (Python package manager)
- Docker (for containerized deployment)
- Ray (implicitly managed by `uv` and Docker setup for the most part)

## Development Setup

1.  **Clone the repository**:
    ```bash
    git clone https://github.com/apecloud/doc-ray
    cd doc-ray
    ```

2.  **Create and activate a virtual environment using `uv`**:
    ```bash
    uv venv
    source .venv/bin/activate
    ```

3.  **Install dependencies using `uv`**:
    ```bash
    uv sync --all-extras
    ```

4.  **Prepare MinerU prerequisites**:
    Run the script to download models required by MinerU and generate the `mineru.json` file.
    ```bash
    mineru-models-download -m pipeline
    cp ~/mineru.json .
    ```

5.  **Run the service locally**:
    The `run.py` script initializes a local Ray instance and deploys the Ray Serve application.
    ```bash
    python run.py
    ```
    The service will typically be available at `http://localhost:8639`.
    - API documentation (Swagger UI) is often available at `http://localhost:8639/docs`.
    - Ray Dashboard: `http://localhost:8265`.

## API Endpoints

-   **POST `/submit`**: Submits a document for parsing.
    -   Request Body: `{"document_data": "content of the document"}`
    -   Response: `{"job_id": "unique_job_id", "message": "Document submitted..."}` (Status 202)
-   **GET `/status/{job_id}`**: Checks the parsing status.
    -   Response: `{"job_id": "unique_job_id", "status": "processing|completed|failed", "error": "error message if failed"}`
-   **GET `/result/{job_id}`**: Retrieves the parsing result.
    -   Response (if completed): `{"job_id": "unique_job_id", "status": "completed", "result": {"markdown": "parsed markdown content"}}`
    -   Response (if pending/failed): `{"job_id": "unique_job_id", "status": "processing|failed", "message": "...", "error": "..."}`
-   **DELETE `/result/{job_id}`**: Deletes a job and its result to free up resources.
    -   Response (if successful): `{"job_id": "unique_job_id", "message": "Job and result deleted successfully."}` (Status 200)

## Testing with `client.py`

Once the `doc-ray` service is running, you can use the provided `client.py` script to submit a document for parsing and test the service. The script accepts both local file paths and URLs as input.

1.  **Ensure `client.py` is executable or run it with `python`**:
    The script is located in the `scripts` directory.

2.  **Basic Usage**:
    Navigate to the root directory of the project and run:

    * **For a local file:**
    ```bash
    python scripts/client.py path/to/your/document.pdf
    ```
    Replace `path/to/your/document.pdf` with the actual path to the local document you want to test.

    * **For a URL:**
    ```bash
    python scripts/client.py https://raw.githubusercontent.com/microsoft/markitdown/da7bcea527ed04cf6027cc8ece1e1aad9e08a9a1/packages/markitdown/tests/test_files/test.pdf
    ```
    Replace the URL with the actual URL of the document you want to test. The script will download the content from the URL before submitting it.

3.  **Specifying `DOCRAY_HOST` (if not default)**:
    If your `doc-ray` service is not running at the default `http://localhost:8639`, you need to set the `DOCRAY_HOST` environment variable:
    ```bash
    DOCRAY_HOST="http://your-doc-ray-service-address:port" python scripts/client.py path/to/your/document.pdf
    ```
    For example, if it's running on a different host or port.

## Deployment

### Standalone Mode with Docker

1.  **(Optional) Build the Docker image locally**:

    ```bash
    make build
    ```

2.  **Run the Docker container**:
    ```bash
    make run-standalone

    # Or run:
    # docker run -d -p 8639:8639 -p 8265:8265 --gpus=all --name doc-ray apecloud/doc-ray:latest
    ```
    - `-d`: Run in detached mode.
    - `-p 8639:8639`: Maps the container's port 8639 (Ray Serve HTTP) to the host's port 8639.
    - `-p 8265:8265`: Maps the container's port 8265 (Ray Dashboard) to the host's port 8265.
    - `--gpus=all`: Enables GPU support. If your Docker environment does not provide GPU access (e.g., Docker Desktop for macOS), omit this flag. **Using GPUs is strongly recommended for optimal performance.**
    - `--name doc-ray`: Assigns a name to the container for easier management.

    The service will be accessible at `http://localhost:8639` on your host machine.

### Cluster Mode

There are two supported paths:

1) Non-containerized remote Serve (ships code + installs deps)

- Ensure your client can connect to the cluster (e.g., `export RAY_ADDRESS=ray://<head_ip>:10001`).
- Use the provided config that includes a runtime_env to ship code and install Python deps on the cluster:

```bash
serve run serve_config_cluster.yaml
```

Notes:
- This downloads/install deps (incl. Torch) on the cluster nodes; ensure egress is allowed.
- Models are pulled at runtime via Hugging Face by default. To use cached/local models, set `MINERU_MODEL_SOURCE=local` and ensure `mineru.json` points to local paths.

2) Recommended: Containerized with KubeRay (image includes deps + models)

- Install KubeRay CRDs/controller per Ray docs.
- Apply the RayService manifest that uses the published image (includes all dependencies and models):

```bash
kubectl apply -f kubernetes/rayservice.yaml
```

- Access the Serve HTTP endpoint (port 8639):
  - Port-forward the head pod:
    ```bash
    kubectl get pods -l ray.io/node-type=head
    kubectl port-forward <head-pod-name> 8639:8639
    ```
  - Or create a Service/Ingress that exposes port 8639.

Tuning/Env:
- `PARSER_FORCE_GPU_PER_REPLICA` to request GPUs per replica (e.g., `1`).
- `PARSER_NUM_CPUS_PER_REPLICA` to bound CPU use; by default Ray decides.
- `MINERU_CONFIG_JSON` (`/mineru/mineru.json` in the image) and `MINERU_MODEL_SOURCE` (`local` in image).

Additional notes:
- `serve_config.yaml` is intended for use inside the container image where the code and deps are already present. For remote clusters without the image, use `serve_config_cluster.yaml`.
- To build and push images for your registry/cluster, use `make build-and-push-multiarch REGISTRY=<your-registry>/ IMAGE_NAME=<repo/name> IMAGE_TAG=<tag>` and update `kubernetes/rayservice.yaml` accordingly.

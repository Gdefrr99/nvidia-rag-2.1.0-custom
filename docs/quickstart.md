# Nvidia RAG Blueprint Installation Guide (Version 2.1.0)

This document serves as a guide for downloading and installing a functional RAG (Retrieval-Augmented Generation) system with Nvidia infrastructure using the Docker container deployment tool.

The version for which this guide was prepared is [2.1.0](https://github.com/NVIDIA-AI-Blueprints/rag/tree/v2.1.0). This version is designed to work with A100 80GB GPUs or higher. Therefore, if you aim to deploy the RAG with more limited resources (specifically **3xA100 40GB**), it is recommended to follow the steps described in this guide instead of those outlined in the [quickstart.md](https://github.com/NVIDIA-AI-Blueprints/rag/blob/v2.1.0/docs/quickstart.md) readme of the original repository.

Regardless, this guide does not replace the official documentation. If you wish to learn more about using a RAG with Nvidia infrastructure and its various functionalities, it is recommended to visit the [official Nvidia repository](https://github.com/NVIDIA-AI-Blueprints/rag/tree/v2.1.0).

## Minimum Requirements

* **Operating System:** Ubuntu 22.04 OS
* **Drivers:**
    * GPU: 530.30.02 (or later).
    * CUDA Version: 12.6 (or later).
* **Hardware:** 3xA100 40GB
* **Deployment Tool:** Docker.

## NGC API Key

An API key is required to access NIM (Nvidia Inference Microservices) services, access models hosted in the NVIDIA API catalog, and download local models.

To generate an NGC API Key, follow these steps:

1.  Register at [Nvidia for Developers](https://developer.nvidia.com/login).
2.  Enter the [Nvidia NGC portal](https://login.nvgs.nvidia.com/v1/nfactor/prompt-challenge?key=eyJhbGciOiJIUzI1NiJ9.eyJzZSI6IjJmeWgiLCJ0b2tlbklkIjoiMTQ2Nzk5NTAyMjg5Mzk0ODkyOCIsImV4cCI6MTc3MDA2ODU5OSwib3QiOiIxNDY3OTk1MDcyNzA5MjkyMDMyIiwianRpIjoiNzVkMDRkNGEtNWUxNC00ZWU3LWEzZDQtNGZkMGE1MWI3N2U3In0.iEgkcy-kWYIEQUe475otjbRoKL2YKxNLRwNrcEQ5dDk&client_id=323893095789756813&context=reset&code=651338d851274919a8f8fd60a3a98426&multipleOrigin=false&isAutoInit=false) using the same username and password as in the previous registration.
3.  Click on **Generate Personal Key**.
4.  Enter a name for the key and an expiration time.
5.  Select the **NGC Catalog** and **Public API Endpoints** services.
6.  Generate and copy the key to a safe place.

> **Note:** Even if an expiration time is selected, the key may start to fail if many requests are made; therefore, it is recommended to regenerate it periodically.

## RAG Deployment with Docker Compose

1.  Install [Docker Engine for Ubuntu](https://docs.docker.com/engine/install/ubuntu/) following one of the proposed methods (recommended: *Install using the apt repository*).
2.  Install [Docker Compose Plugin](https://docs.docker.com/compose/install/linux/) (recommended: *Install using the repository*). Verify that the installed version is **2.29.1** or later using `docker compose version`.
3.  To pull the model images required by the RAG from NGC, you must authenticate in Docker with `nvcr.io` using the previously created NGC API Key.

    ```bash
    export NGC_API_KEY="nvapi-..."
    echo "${NGC_API_KEY}" | docker login nvcr.io -u '$oauthtoken' --password-stdin
    ```

4.  Some containers are enabled for GPU acceleration, such as Milvus and locally implemented NVIDIA NIMs. To configure Docker for GPU-accelerated containers, [install](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) the **NVIDIA Container Toolkit** if it is not already installed.
5.  Outside the main folder, create a directory to cache the models and export the cache path as an environment variable:

    ```bash
    mkdir -p ~/.cache/model-cache
    export MODEL_DIRECTORY=~/.cache/model-cache
    ```

6.  Next, inside the main folder, go to the `deploy/compose/` subdirectory and create a `.env` file with the following content:

    ```bash
    # ==== Set User for local NIM deployment ====
    export USERID=$(id -u)

    # ==== Endpoints for using on-prem NIMS ====
    export APP_LLM_SERVERURL=nim-llm:8000
    export APP_EMBEDDINGS_SERVERURL=nemoretriever-embedding-ms:8000
    export EMBEDDING_NIM_ENDPOINT=http://nemoretriever-embedding-ms:8000/v1
    export APP_RANKING_SERVERURL=nemoretriever-ranking-ms:8000
    export PADDLE_GRPC_ENDPOINT=paddle:8001
    export PADDLE_INFER_PROTOCOL=grpc
    export YOLOX_GRPC_ENDPOINT=page-elements:8001
    export YOLOX_INFER_PROTOCOL=grpc
    export YOLOX_GRAPHIC_ELEMENTS_GRPC_ENDPOINT=graphic-elements:8001
    export YOLOX_GRAPHIC_ELEMENTS_INFER_PROTOCOL=grpc
    export YOLOX_TABLE_STRUCTURE_GRPC_ENDPOINT=table-structure:8001
    export YOLOX_TABLE_STRUCTURE_INFER_PROTOCOL=grpc

    # ==== Endpoints for using cloud NIMS ===
    # export APP_EMBEDDINGS_SERVERURL=""
    # export APP_LLM_SERVERURL=""
    # export APP_RANKING_SERVERURL=""
    # export EMBEDDING_NIM_ENDPOINT=https://integrate.api.nvidia.com/v1]
    # export PADDLE_HTTP_ENDPOINT=https://ai.api.nvidia.com/v1/cv/baidu/paddleocr
    # export PADDLE_INFER_PROTOCOL=http
    # export YOLOX_HTTP_ENDPOINT=https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-page-elements-v2
    # export YOLOX_INFER_PROTOCOL=http
    # export YOLOX_GRAPHIC_ELEMENTS_HTTP_ENDPOINT=https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-graphic-elements-v1
    # export YOLOX_GRAPHIC_ELEMENTS_INFER_PROTOCOL=http
    # export YOLOX_TABLE_STRUCTURE_HTTP_ENDPOINT=https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-table-structure-v1
    # export YOLOX_TABLE_STRUCTURE_INFER_PROTOCOL=http

    # Set GPU IDs for local deployment
    #==== LLM ====
    export LLM_MS_GPU_ID=1,2
    # ==== Embeddings ====
    export EMBEDDING_MS_GPU_ID=0
    #==== Reranker ====
    export RANKING_MS_GPU_ID=0
    #==== Vector DB GPU ID ====
    export VECTORSTORE_GPU_DEVICE_ID=0
    # ==== Ingestion NIMS GPU ids ====
    export YOLOX_MS_GPU_ID=0
    export YOLOX_GRAPHICS_MS_GPU_ID=0
    export YOLOX_TABLE_MS_GPU_ID=0
    export PADDLE_MS_GPU_ID=0
    ```

7.  Returning to the main folder, export all the environment variables required to use the models locally.

    ```bash
    source deploy/compose/.env
    source deploy/compose/accuracy_profile.env
    source deploy/compose/perf_profile.env
    ```

    > **Note:** The variables in the `deploy/compose/accuracy_profile.env` file can be modified following the [best practices](./accuracy_perf.md) readme according to user needs. On the other hand, the `deploy/compose/perf_profile.env` file contains recommended configurations for the optimization profile.
    
    **IMPORTANT**: If this repository has already been cloned, the user can skip all subsequent steps up to and including step 12. It is only necessary to execute `export OPENTELEMETRY_CONFIG_FILE=$(pwd)/deploy/config/otel-collector-config.yaml` from the main folder to enable tracing for observability services. Steps 8 through 12 are for users who wish to replicate the steps taken to achieve this modified version of NVIDIA's official RAG. If this is the case, users must clone the original project from the [official Nvidia GitHub repository for version 2.1.0](https://github.com/NVIDIA-AI-Blueprints/rag/tree/v2.1.0). Once cloned, **none of the project files should be moved**, and all subsequent steps must be carried out within this original cloned project.

8.  The inference, embedding generation, and reranking models offered by Nvidia are [llama-3.3-nemotron-super-49b-v1](https://github.com/NVIDIA-AI-Blueprints/rag/tree/v2.1.0), [llama-3.2-NV-EmbedQA-1B-v2](https://build.nvidia.com/nvidia/llama-3_2-nv-embedqa-1b-v2), and [llama-3.2-nv-rerankqa-1b-v2](https://build.nvidia.com/nvidia/llama-3_2-nv-rerankqa-1b-v2) respectively. However, as previously mentioned, working with these models requires several A100 80GB GPUs or higher.

    To work with **3xA100 40GB**, it is necessary to change the inference model to [llama3.1-nemotron-nano-4b-v1.1](https://build.nvidia.com/nvidia/llama-3_1-nemotron-nano-4b-v1_1) and the embedding generation model to [nv-embedqa-e5-v5](https://build.nvidia.com/nvidia/nv-embedqa-e5-v5). To do this, several files must be modified following the instructions in the [change-model.md](./change-model.md) readme. Specifically, the following changes must be made:

    * **Within [nims.yaml](../deploy/compose/nims.yaml):**
        * In the `nim-llm` service, change:
            `image: nvcr.io/nim/nvidia/llama-3.3-nemotron-super-49b-v1:1.2.3`
            to
            `image: nvcr.io/nim/nvidia/llama-3.1-nemotron-nano-4b-instruct:latest`
            and change `device_ids: ['${LLM_MS_GPU_ID:-1}']` to `device_ids: ['1', '2']`.
        * In the `nemoretriever-embedding-ms` service, change `image: nvcr.io/nim/nvidia/llama-3.2-nv-embedqa-1b-v2:1.0.0` to `image: nvcr.io/nim/nvidia/nv-embedqa-e5-v5:1.0.0`.
        * In the `nemoretriever-ranking-ms` service, change `device_ids: ['${RANKING_MS_GPU_ID:-0}']` to `device_ids: ['0', '1', '2']`.

    * **Within [docker-compose-ingestor-server.yaml](../deploy/compose/docker-compose-ingestor-server.yaml):**
        * Change:
            `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}`
            to
            `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}`
        * Change:
            `EMBEDDING_NIM_MODEL_NAME=${EMBEDDING_NIM_MODEL_NAME:-${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}}`
            to
            `EMBEDDING_NIM_MODEL_NAME=${EMBEDDING_NIM_MODEL_NAME:-${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}}`
        * And change `APP_EMBEDDINGS_DIMENSIONS: ${APP_EMBEDDINGS_DIMENSIONS:-2048}` to `APP_EMBEDDINGS_DIMENSIONS: ${APP_EMBEDDINGS_DIMENSIONS:-1024}`. This last step is necessary because we will be working with a model that generates embeddings of size 1024.

    * **Within [docker-compose-rag-server.yaml](../deploy/compose/docker-compose-rag-server.yaml), change:**
        * `APP_LLM_MODELNAME: ${APP_LLM_MODELNAME:-"nvidia/llama-3.3-nemotron-super-49b-v1"}`
            to `APP_LLM_MODELNAME: ${APP_LLM_MODELNAME:-"nvidia/Llama-3.1-Nemotron-Nano-4B-Instruct"}`
        * `NEXT_PUBLIC_MODEL_NAME: ${APP_LLM_MODELNAME:-nvidia/llama-3.3-nemotron-super-49b-v1}`
            to `NEXT_PUBLIC_MODEL_NAME: ${APP_LLM_MODELNAME:-nvidia/Llama-3.1-Nemotron-Nano-4B-Instruct}`
        * `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}`
            to `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}`
        * `NEXT_PUBLIC_EMBEDDING_MODEL: ${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}`
            to `NEXT_PUBLIC_EMBEDDING_MODEL: ${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}`

9.  To handle the size 1024 embeddings generated by the new model, two additional files must be modified:
    * **In [server.py](../src/ingestor_server/server.py), change:**
        ```python
        async def create_collections(
            vdb_endpoint: str = Query(default=os.getenv("APP_VECTORSTORE_URL"), include_in_schema=False),
            collection_names: List[str] = [os.getenv("COLLECTION_NAME")],
            collection_type: str = "text",
            embedding_dimension: int = 2048
        ```
        to
        ```python
        async def create_collections(
            vdb_endpoint: str = Query(default=os.getenv("APP_VECTORSTORE_URL"), include_in_schema=False),
            collection_names: List[str] = [os.getenv("COLLECTION_NAME")],
            collection_type: str = "text",
            embedding_dimension: int = 1024
        ```
    * **In [configuration.py](../src/configuration.py), change:**
        ```python
        dimensions: int = configfield(
            "dimensions",
            default=2048,
            help_txt="The required dimensions of the embedding model. Currently utilized for vector DB indexing.",
        )
        ```
        to
        ```python
        dimensions: int = configfield(
            "dimensions",
            default=1024,
            help_txt="The required dimensions of the embedding model. Currently utilized for vector DB indexing.",
        )
        ```

10. When working with A100 GPUs, it is recommended to deploy the collection database service on CPU instead of GPU to avoid out-of-memory issues. Modify the `milvus` service in [vectordb.yaml](../deploy/compose/vectordb.yaml) as follows:

    ```yaml
    milvus:
      container_name: milvus-standalone
      image: milvusdb/milvus:v2.5.3 # milvusdb/milvus:v2.5.3 for CPU
      command: ["milvus", "run", "standalone"]
      environment:
        ETCD_ENDPOINTS: etcd:2379
        MINIO_ADDRESS: minio:9000
        KNOWHERE_GPU_MEM_POOL_SIZE: 2048;4096
        APP_VECTORSTORE_ENABLEGPUSEARCH: "False"
        APP_VECTORSTORE_ENABLEGPUINDEX: "False"
      volumes:
        - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
        interval: 30s
        start_period: 90s
        timeout: 20s
        retries: 3
      ports:
        - "19530:19530"
        - "9091:9091"
      depends_on:
        - "etcd"
        - "minio"
      # Comment out this section if CPU based image is used and set below env variables to False
      # export APP_VECTORSTORE_ENABLEGPUSEARCH=False
      # export APP_VECTORSTORE_ENABLEGPUINDEX=False
      # deploy:
      #   resources:
      #     reservations:
      #       devices:
      #         - driver: nvidia
      #           capabilities: ["gpu"]
      #           device_ids: ['0']
    ```

    Additionally, change the variables `APP_VECTORSTORE_ENABLEGPUINDEX: ${APP_VECTORSTORE_ENABLEGPUINDEX:-True}` and `APP_VECTORSTORE_ENABLEGPUSEARCH: ${APP_VECTORSTORE_ENABLEGPUSEARCH:-True}` to `False` in the [docker-compose-ingestor-server.yaml](../deploy/compose/docker-compose-ingestor-server.yaml) file.

11. Regarding Docker configuration, you must enable the Nvidia runtime. Modify the `/etc/docker/daemon.json` file, which should contain something similar to:

    ```json
    {
       "runtimes": {
           "nvidia": {
               "path": "nvidia-container-runtime",
               "runtimeArgs": []
           }
       }
    }
    ```
    and add `"default-runtime": "nvidia"`:
    ```json
    {
       "runtimes": {
           "nvidia": {
               "path": "nvidia-container-runtime",
               "runtimeArgs": []
           }
       },
       "default-runtime": "nvidia"
    }
    ```

12. In the [docs](../docs/) folder, you can find the information required to add new services to enhance the RAG. However, due to deployment issues in the cloud and limited GPU memory, the only additional functionalities that can be added with our configuration and resources are:

    * **Hybrid Search:** Combines keyword matching and dense representations. Following the [hybrid_search.md](../docs/hybrid_search.md) readme, simply change `APP_VECTORSTORE_SEARCHTYPE: ${APP_VECTORSTORE_SEARCHTYPE:-"dense"}` to `APP_VECTORSTORE_SEARCHTYPE: ${APP_VECTORSTORE_SEARCHTYPE:-"hybrid"}` in the [docker-compose-ingestor-server.yaml](../deploy/compose/docker-compose-ingestor-server.yaml) and [docker-compose-rag-server.yaml](../deploy/compose/docker-compose-rag-server.yaml) files.
    * **Observability:** Enables tracing and observability for the RAG server using OpenTelemetry (OTel) Collector and Zipkin. To start the observability services, set the required environment variable for the OTel Collector Config from the main folder:
        ```bash
        export OPENTELEMETRY_CONFIG_FILE=$(pwd)/deploy/config/otel-collector-config.yaml
        ```
        Then, in the [docker-compose-rag-server.yaml](../deploy/compose/docker-compose-rag-server.yaml) file, change the variable `APP_TRACING_ENABLED: "False"` to `APP_TRACING_ENABLED: "True"`. Refer to the [observability.md](../docs/observability.md) readme for more details on this service.
    * **Query Rewriting Support:** Enhances precision for conversations by making an additional LLM call to decontextualize the incoming question before sending it to the retrieval stage. Following the [query_rewriter.md](../docs/query_rewriter.md) readme, to deploy this service in the cloud (as it does not fit on GPU with the other services), change `APP_QUERYREWRITER_SERVERURL: ${APP_QUERYREWRITER_SERVERURL:-"nim-llm:8000"}` to `APP_QUERYREWRITER_SERVERURL: ${APP_QUERYREWRITER_SERVERURL:-""}`, and `ENABLE_QUERYREWRITER: ${ENABLE_QUERYREWRITER:-False}` to `ENABLE_QUERYREWRITER: ${ENABLE_QUERYREWRITER:-true}`. To change the model used, modify the `APP_QUERYREWRITER_MODELNAME` variable.

13. Once the necessary files are modified to deploy the RAG with new models and added services, start the services one by one:

    ```bash
    sudo env NGC_API_KEY=nvapi-... USERID=$(id -u) docker compose -f deploy/compose/nims.yaml up -d
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/vectordb.yaml up -d
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-ingestor-server.yaml up -d
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-rag-server.yaml up -d
    sudo docker compose -f deploy/compose/observability.yaml up -d
    ```

14. The first execution will take significant time as the models must be downloaded and stored in the `MODEL_DIRECTORY` path, which acts as a cache. After execution, verify that everything is correct by running `sudo docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"`; the containers should be listed as Up and healthy.

    Once all services are correctly deployed, you can interact with the RAG through your browser at `http://localhost:8090`.
    You can also access the observability interface at `http://localhost:9411` (Zipkin) and the metrics collected by the OTel Collector at `http://localhost:8889/metrics`.

15. To stop all running services:

    ```bash
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-ingestor-server.yaml down
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/nims.yaml down
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-rag-server.yaml down
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/vectordb.yaml down
    sudo docker compose -f deploy/compose/observability.yaml down
    ```
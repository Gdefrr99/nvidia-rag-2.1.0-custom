<!--
  SPDX-FileCopyrightText: Copyright (c) 2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
  SPDX-License-Identifier: Apache-2.0
-->

# Guía para la instalación de Nvidia RAG Blueprint (versión 2.1.0)

Este documento sirve como guía para la descarga e instalación de un RAG funcional con infraestructura Nvidia utilizando la herramienta de despliegue de contenedores Docker.

La versión para la cual se ha elaborado la guía corresponde a la [2.1.0](https://github.com/NVIDIA-AI-Blueprints/rag/tree/v2.1.0). Esta versión está pensada para trabajar con GPUs tipo A100 80GB o superiores, de modo que si se busca desplegar el RAG con recursos más limitados (concretamente **3xA100 40GB**) se recomienda seguir los pasos descritos en esta guía en lugar de los expuestos en el readme [quickstart.md](https://github.com/NVIDIA-AI-Blueprints/rag/blob/v2.1.0/docs/quickstart.md) del repositorio original.

Independientemente, esta guía no sustituye a la documentación oficial, por lo que si se busca aprender más acerca de la utilización de un RAG con infraestructura Nvidia y sus distintas funcionalidades, se recomienda que visite el [repositorio oficial de Nvidia](https://github.com/NVIDIA-AI-Blueprints/rag/tree/v2.1.0).

## Requisitos mínimos

* **Sistema Operativo:** Ubuntu 22.04 OS
* **Drivers:**
    * GPU: 530.30.02 (o posterior).
    * Versión de CUDA: 12.6 (o posterior).
* **Hardware:** 3xA100 40GB
* **Herramienta de despliegue:** Docker.

## NGC API Key

Es necesario generar una clave API para acceder a los servicios NIM (Nvidia Inference Microservices), acceder a los modelos alojados en el catálogo de API de NVIDIA y descargar modelos locales.

Para generar una NGC API Key, debe seguir los siguientes pasos:

1.  Regístrese en [Nvidia for Developers](https://developer.nvidia.com/login).
2.  Entre en el portal [NGC de Nvidia](https://login.nvgs.nvidia.com/v1/nfactor/prompt-challenge?key=eyJhbGciOiJIUzI1NiJ9.eyJzZSI6IjJmeWgiLCJ0b2tlbklkIjoiMTQ2Nzk5NTAyMjg5Mzk0ODkyOCIsImV4cCI6MTc3MDA2ODU5OSwib3QiOiIxNDY3OTk1MDcyNzA5MjkyMDMyIiwianRpIjoiNzVkMDRkNGEtNWUxNC00ZWU3LWEzZDQtNGZkMGE1MWI3N2U3In0.iEgkcy-kWYIEQUe475otjbRoKL2YKxNLRwNrcEQ5dDk&client_id=323893095789756813&context=reset&code=651338d851274919a8f8fd60a3a98426&multipleOrigin=false&isAutoInit=false) utilizando el mismo usuario y contraseña que en el registro anterior.
3.  Haga click en **Generate Personal Key**.
4.  Introduzca un nombre para la clave y un tiempo de expiración.
5.  Seleccione los servicios **NGC Catalog** y **Public API Endpoints**.
6.  Genere y copie la clave en un lugar seguro.

> **Nota:** aun habiendo seleccionado un tiempo de expiración, si se realizan muchas peticiones la clave puede comenzar a fallar, por lo que se recomienda regenerarla cada cierto tiempo.

## Despliegue del RAG en Docker Compose

1.  Instale [Docker Engine para Ubuntu](https://docs.docker.com/engine/install/ubuntu/) siguiendo uno de los métodos propuestos (recomendado *Install using the apt repository*).
2.  Instale [Docker Compose Plugin](https://docs.docker.com/compose/install/linux/) (recomendado *Install using the repository*). Mediante `docker compose version` compruebe que la versión instalada es la **2.29.1** o posterior.
3.  Para obtener las imágenes de los modelos requeridas por el RAG desde NGC, es necesario autenticarse en Docker con `nvcr.io` utilizando la NGC API Key creada anteriormente.

    ```bash
    export NGC_API_KEY="nvapi-..."
    echo "${NGC_API_KEY}" | docker login nvcr.io -u '$oauthtoken' --password-stdin
    ```

4.  Algunos contenedores están habilitados para la aceleración de GPU, como Milvus y NVIDIA NIMs implementados localmente. Para configurar Docker para contenedores acelerados por GPU, [instale](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) **NVIDIA Container Toolkit** en caso de que no lo esté ya.
5.  Fuera de la carpeta principal cree un directorio para almacenar en caché los modelos y exporte la ruta a la caché como una variable de entorno:

    ```bash
    mkdir -p ~/.cache/model-cache
    export MODEL_DIRECTORY=~/.cache/model-cache
    ```

6.  A continuación, dentro de la carpeta principal, acceda al subdirectorio `deploy/compose/` y cree un archivo `.env` con el siguiente contenido:

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
    # export EMBEDDING_NIM_ENDPOINT=[https://integrate.api.nvidia.com/v1](https://integrate.api.nvidia.com/v1)
    # export PADDLE_HTTP_ENDPOINT=[https://ai.api.nvidia.com/v1/cv/baidu/paddleocr](https://ai.api.nvidia.com/v1/cv/baidu/paddleocr)
    # export PADDLE_INFER_PROTOCOL=http
    # export YOLOX_HTTP_ENDPOINT=[https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-page-elements-v2](https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-page-elements-v2)
    # export YOLOX_INFER_PROTOCOL=http
    # export YOLOX_GRAPHIC_ELEMENTS_HTTP_ENDPOINT=[https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-graphic-elements-v1](https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-graphic-elements-v1)
    # export YOLOX_GRAPHIC_ELEMENTS_INFER_PROTOCOL=http
    # export YOLOX_TABLE_STRUCTURE_HTTP_ENDPOINT=[https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-table-structure-v1](https://ai.api.nvidia.com/v1/cv/nvidia/nemoretriever-table-structure-v1)
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

7.  Volviendo a la carpeta principal, exporte todas las variables de entorno requeridas para usar los modelos localmente.

    ```bash
    source deploy/compose/.env
    source deploy/compose/accuracy_profile.env
    source deploy/compose/perf_profile.env
    ```

    > **Nota:** las variables presentes en el archivo `deploy/compose/accuracy_profile.env` se pueden modificar siguiendo el readme de buenas prácticas según las necesidades del usuario. Por otro lado, el archivo `deploy/compose/perf_profile.env` contiene configuraciones recomendadas para el perfil de optimización.
   
**IMPORTANTE**: si ya se ha clonado este repositorio, el usuario puede saltarse todos los pasos siguientes hasta el 12 incluido, siendo únicamente necesario ejecutar `export OPENTELEMETRY_CONFIG_FILE=$(pwd)/deploy/config/otel-collector-config.yaml` desde la carpeta principal para habilitar el rastreo los servicios de observabilidad. Los pasos del 8 al 12 sirven para aquellos usuarios que quieran replicar los pasos seguidos para conseguir esta versión modificada del RAG oficial de NVIDIA. Si este es el caso, los usuarios deben clonar el proyecto original desde el [repositorio oficial de Nvidia en GitHub para la versión 2.1.0](https://github.com/NVIDIA-AI-Blueprints/rag/tree/v2.1.0). Una vez clonado, **NO se debe cambiar de lugar ninguno de los archivos dentro del proyecto**, y todos los siguientes pasos se deben llevar acabo sobre este proyecto clonado.

8.  Los modelos de inferencia, generación de embeddings y reranking ofrecidos por Nvidia son `llama-3.3-nemotron-super-49b-v1`, `llama-3.2-NV-EmbedQA-1B-v2` y `llama-3.2-nv-rerankqa-1b-v2` respectivamente. Sin embargo, como se comentó anteriormente, para trabajar con estos modelos se requieren varias GPUs A100 80 GB o superiores.

    Para trabajar con **3xA100 40 GB** es necesario cambiar el modelo de inferencia por `llama3.1-nemotron-nano-4b-v1.1` y el modelo de generación de embeddings por `nv-embedqa-e5-v5`. Para ello, será necesario modificar una serie de archivos siguiendo las indicaciones del readme `change-model.md`. Más concretamente, los cambios que debemos realizar son los siguientes:

    * **a. Dentro de `nims.yaml`:**
        * i. En el servicio `nim-llm`, cambiamos:
            `image: nvcr.io/nim/nvidia/llama-3.3-nemotron-super-49b-v1:1.2.3`
            por
            `image: nvcr.io/nim/nvidia/llama-3.1-nemotron-nano-4b-instruct:latest`
            así como `device_ids: ['${LLM_MS_GPU_ID:-1}']` por `device_ids: ['1', '2']`.
        * ii. En el servicio `nemoretriever-embedding-ms` cambiamos `image: nvcr.io/nim/nvidia/llama-3.2-nv-embedqa-1b-v2:1.0.0` por `image: nvcr.io/nim/nvidia/nv-embedqa-e5-v5:1.0.0`.
        * iii. En el servicio `nemoretriever-ranking-ms` cambiamos `device_ids: ['${RANKING_MS_GPU_ID:-0}']` por `device_ids: ['0', '1', '2']`.

    * **b. Dentro de `docker-compose-ingestor-server.yaml`:**
        * Cambiamos:
            `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}`
            por
            `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}`
        * Cambiamos:
            `EMBEDDING_NIM_MODEL_NAME=${EMBEDDING_NIM_MODEL_NAME:-${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}}`
            por
            `EMBEDDING_NIM_MODEL_NAME=${EMBEDDING_NIM_MODEL_NAME:-${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}}`
        * Y `APP_EMBEDDINGS_DIMENSIONS: ${APP_EMBEDDINGS_DIMENSIONS:-2048}` por `APP_EMBEDDINGS_DIMENSIONS: ${APP_EMBEDDINGS_DIMENSIONS:-1024}`. Esto último es necesario puesto que vamos a trabajar con un modelo que genera embeddings de tamaño 1024.

    * **c. Dentro de `docker-compose-rag-server.yaml` cambiamos:**
        * `APP_LLM_MODELNAME: ${APP_LLM_MODELNAME:-"nvidia/llama-3.3-nemotron-super-49b-v1"}`
            por `APP_LLM_MODELNAME: ${APP_LLM_MODELNAME:-"nvidia/Llama-3.1-Nemotron-Nano-4B-Instruct"}`
        * `NEXT_PUBLIC_MODEL_NAME: ${APP_LLM_MODELNAME:-nvidia/llama-3.3-nemotron-super-49b-v1}`
            por `NEXT_PUBLIC_MODEL_NAME: ${APP_LLM_MODELNAME:-nvidia/Llama-3.1-Nemotron-Nano-4B-Instruct}`
        * `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}`
            por `APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}`
        * `NEXT_PUBLIC_EMBEDDING_MODEL: ${APP_EMBEDDINGS_MODELNAME:-nvidia/llama-3.2-nv-embedqa-1b-v2}`
            por `NEXT_PUBLIC_EMBEDDING_MODEL: ${APP_EMBEDDINGS_MODELNAME:-nvidia/nv-embedqa-e5-v5}`

9.  Para poder trabajar los embeddings de tamaño 1024 generados por nuestro nuevo modelo, además debemos de modificar dos archivos más:
    * **a. En `server.py`, debemos cambiar:**
        ```python
        async def create_collections(
            vdb_endpoint: str = Query(default=os.getenv("APP_VECTORSTORE_URL"), include_in_schema=False),
            collection_names: List[str] = [os.getenv("COLLECTION_NAME")],
            collection_type: str = "text",
            embedding_dimension: int = 2048
        ```
        por
        ```python
        async def create_collections(
            vdb_endpoint: str = Query(default=os.getenv("APP_VECTORSTORE_URL"), include_in_schema=False),
            collection_names: List[str] = [os.getenv("COLLECTION_NAME")],
            collection_type: str = "text",
            embedding_dimension: int = 1024
        ```
    * **b. En `configuration.py`, debemos cambiar:**
        ```python
        dimensions: int = configfield(
            "dimensions",
            default=2048,
            help_txt="The required dimensions of the embedding model. Currently utilized for vector DB indexing.",
        )
        ```
        por
        ```python
        dimensions: int = configfield(
            "dimensions",
            default=1024,
            help_txt="The required dimensions of the embedding model. Currently utilized for vector DB indexing.",
        )
        ```

10. Al trabajar con GPUs A100, es recomendable desplegar el servicio para la base de datos de colecciones en CPU en lugar de GPU para evitar problemas por falta de memoria. Para ello, modificaremos el servicio `milvus` en `vectordb.yaml` por:

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

    Además también será necesario cambiar las variables `APP_VECTORSTORE_ENABLEGPUINDEX: ${APP_VECTORSTORE_ENABLEGPUINDEX:-True}` y `APP_VECTORSTORE_ENABLEGPUSEARCH: ${APP_VECTORSTORE_ENABLEGPUSEARCH:-True}` por `False`, respectivamente, en el archivo `docker-compose-ingestor-server.yaml`.

11. En cuanto a la configuración de Docker, es necesario habilitar el runtime de Nvidia. Para ello, debemos modificar el archivo `/etc/docker/daemon.json`, que contendrá algo similar a:

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
    y añadir `"default-runtime": "nvidia"`:
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

12. En la carpeta `docs`, podemos encontrar la información necesaria para añadir nuevos servicios que mejorarán nuestro RAG. Sin embargo, debido a problemas en el despliegue de estos servicios en la nube y por falta de memoria en GPU, las únicas funcionalidades extra que se podrán añadir con nuestra configuración y recursos son:

    * **a. Búsqueda híbrida:** combina keyword matching y representaciones densas. Siguiendo el readme `hybrid_search.md`, solo necesitamos cambiar `APP_VECTORSTORE_SEARCHTYPE: ${APP_VECTORSTORE_SEARCHTYPE:-"dense"}` por `APP_VECTORSTORE_SEARCHTYPE: ${APP_VECTORSTORE_SEARCHTYPE:-"hybrid"}` en los archivos `docker-compose-ingestor-server.yaml` y `docker-compose-rag-server.yaml`.
    * **b. Observabilidad:** habilita el rastreo y la observabilidad para el servidor RAG usando OpenTelemetry (OTel) Collector y Zipkin. Para poder iniciar los servicios de observabilidad, desde la carpeta principal establezca la variable de entorno requerida para el OTel Collector Config:
        ```bash
        export OPENTELEMETRY_CONFIG_FILE=$(pwd)/deploy/config/otel-collector-config.yaml
        ```
        Posteriormente, en el archivo `docker-compose-rag-server.yaml` es necesario cambiar la variable `APP_TRACING_ENABLED: "False"` por `APP_TRACING_ENABLED: "True"`. Véase el readme `observability.md` para aprender más acerca del funcionamiento de este servicio.
    * **c. Query rewriting support:** permite una mayor precisión para conversaciones haciendo una llamada adicional al LLM para descontextualizar la pregunta entrante, antes de enviarla a la parte de retrieving. Siguiendo el readme `query_rewriter.md`, para desplegar este servicio en la nube (puesto que no cabe en GPU junto con el resto de servicios) es necesario cambiar la variable `APP_QUERYREWRITER_SERVERURL: ${APP_QUERYREWRITER_SERVERURL:-"nim-llm:8000"}` por `APP_QUERYREWRITER_SERVERURL: ${APP_QUERYREWRITER_SERVERURL:-""}`, así como `ENABLE_QUERYREWRITER: ${ENABLE_QUERYREWRITER:-False}` por `ENABLE_QUERYREWRITER: ${ENABLE_QUERYREWRITER:-true}`. Si se desea modificar también el modelo utilizado, se deberá modificar la variable `APP_QUERYREWRITER_MODELNAME`.

13. Una vez modificados los archivos necesarios para poder desplegar el RAG con los nuevos modelos y los servicios añadidos, levantaremos los servicios uno por uno:

    ```bash
    sudo env NGC_API_KEY=nvapi-... USERID=$(id -u) docker compose -f deploy/compose/nims.yaml up -d
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/vectordb.yaml up -d
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-ingestor-server.yaml up -d
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-rag-server.yaml up -d
    sudo docker compose -f deploy/compose/observability.yaml up -d
    ```

14. La primera ejecución de estos comandos tardará un tiempo considerable puesto que los modelos deben descargarse y almacenarse en el path especificado en `MODEL_DIRECTORY`, que servirá de caché. Una vez ejecutados estos comandos, para asegurarnos de que todo ha ido bien escribiremos por terminal `sudo docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"` y deberíamos ver los contenedores levantados (Up) y saludables (healthy).

    Una vez desplegados todos los servicios correctamente, podremos empezar a interactuar con el RAG accediendo a su interfaz a través de nuestro navegador con `http://localhost:8090`.
    También podremos acceder a la interfaz de observabilidad en `http://localhost:9411` (Zipkin), así como a las métricas recogidas por Otel Collector en `http://localhost:8889/metrics`.

15. Para parar todos los servicios en ejecución:

    ```bash
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-ingestor-server.yaml down
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/nims.yaml down
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/docker-compose-rag-server.yaml down
    sudo env NGC_API_KEY=nvapi-... docker compose -f deploy/compose/vectordb.yaml down
    sudo docker compose -f deploy/compose/observability.yaml down
    ```
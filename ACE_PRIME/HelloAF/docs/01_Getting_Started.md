# Getting Started with ACE_PRIME/HelloAF

This document provides instructions on how to run the ACE_PRIME/HelloAF project and configure it.

## Overview

The HelloAF project is a part of the Autonomous Cognitive Entity (ACE) Framework. It can be run in two main ways:
1.  **Directly via Python:** Running a single resource/layer for development or testing.
2.  **Via Docker Compose:** Running the full suite of services as defined in `docker-compose.yaml`, managed by `resource_manager.py` or standard `docker compose` commands. This is the typical way to run the interconnected system.

## Method 1: Running a Single Resource Directly (Python)

This method is suitable for developing or testing individual ACE resources (layers or components).

### Entry Point

The primary entry point for individual resources is:
`ACE_PRIME/HelloAF/src/main.py`

### How to Run

1.  **Navigate to the project directory:**
    ```bash
    cd path/to/ACE_PRIME/HelloAF
    ```
2.  **Set Required Environment Variables:**
    At a minimum, you need to set `ACE_RESOURCE_NAME`. You might also need to set others depending on the resource being run (e.g., `OPENAI_API_KEY` for resources that interact with OpenAI, or RabbitMQ variables if it needs to connect to the bus).

    Example for a hypothetical resource named `my_custom_layer`:
    ```bash
    export ACE_RESOURCE_NAME="my_custom_layer"
    export OPENAI_API_KEY="your_openai_api_key_here"
    # Potentially other variables like ACE_RESOURCE_SUBDIRECTORY if not using the default "hello_layers"
    # export ACE_RESOURCE_SUBDIRECTORY="your_custom_subdirectory"
    ```
3.  **Execute the script:**
    ```bash
    python src/main.py
    ```

### Key Environment Variables (for direct Python execution)

*   `ACE_RESOURCE_NAME` (Required): Specifies the name of the resource or layer you want to run. The `src/main.py` script uses this name to find and load the corresponding Python module. Example: `layer_1`, `logging`, `my_custom_resource`.
*   `ACE_RESOURCE_SUBDIRECTORY` (Optional, defaults to `hello_layers`): Specifies the subdirectory under `ace.resources.custom` or `ace.resources.core` where the resource code is located.
*   `OPENAI_API_KEY` (Conditionally Required): If the resource you are running interacts with OpenAI, this API key must be provided.
*   `ACE_LOG_LEVEL` (Optional): Sets the logging level for ACE components (e.g., `DEBUG`, `INFO`, `WARNING`).
*   `ACE_THIRD_PARTY_LOG_LEVEL` (Optional): Sets the logging level for third-party libraries.
*   RabbitMQ Variables (Conditionally Required): If the resource needs to connect to RabbitMQ for inter-layer communication:
    *   `ACE_RABBITMQ_HOSTNAME` (defaults to `rabbitmq` in Docker context, may need explicit setting for local Python runs if RabbitMQ is elsewhere)
    *   `ACE_RABBITMQ_USERNAME` (defaults to `rabbit`)
    *   `ACE_RABBITMQ_PASSWORD` (defaults to `carrot`)

## Method 2: Running with Docker Compose (Recommended for Full System)

This method uses Docker to build and run all the services (layers, communication buses, etc.) as defined in `ACE_PRIME/HelloAF/docker-compose.yaml`. This is the recommended way to run the complete system.

### Entry Point / Orchestration

*   `ACE_PRIME/HelloAF/docker-compose.yaml`: Defines all the services, their configurations, environment variables, and dependencies.
*   `ACE_PRIME/HelloAF/resource_manager.py`: A Python script that provides a command-line interface to manage the Docker Compose application (start, stop, monitor). Alternatively, you can use standard `docker compose` commands.

### How to Run (using `resource_manager.py`)

1.  **Navigate to the project directory:**
    ```bash
    cd path/to/ACE_PRIME/HelloAF
    ```
2.  **Ensure Docker is running.**
3.  **Set Required Environment Variables (in your shell):**
    The `docker-compose.yaml` file is configured to pick up certain variables from your shell environment. The most critical one is:
    *   `OPENAI_API_KEY`: Your OpenAI API key.
    ```bash
    export OPENAI_API_KEY="your_openai_api_key_here"
    ```
    You can also override other variables like `ACE_LOG_LEVEL`, `ACE_RESOURCE_SUBDIRECTORY`, or RabbitMQ settings if needed, though `docker-compose.yaml` provides sensible defaults for a Docker environment.
4.  **Run the resource manager:**
    *   To build (if necessary) and start all services in detached mode:
        ```bash
        python resource_manager.py --build --detach
        ```
    *   To see other options:
        ```bash
        python resource_manager.py --help
        ```

### How to Run (using direct `docker compose` commands)

1.  **Navigate to the project directory:**
    ```bash
    cd path/to/ACE_PRIME/HelloAF
    ```
2.  **Ensure Docker is running.**
3.  **Set Required Environment Variables (in your shell):**
    As above, primarily `OPENAI_API_KEY`.
    ```bash
    export OPENAI_API_KEY="your_openai_api_key_here"
    ```
4.  **Run Docker Compose:**
    *   To build (if necessary) and start all services in detached mode:
        ```bash
        docker compose up --build -d
        ```
    *   To stop services:
        ```bash
        docker compose down
        ```

### Key Environment Variables (for Docker Compose execution)

Many environment variables are set *within* the `docker-compose.yaml` file for each specific service. However, some are expected to be passed from your host environment:

*   `OPENAI_API_KEY` (Required): Your OpenAI API key. This is passed into services that need it (e.g., `aspirational_layer`, `global_strategy_layer`).
*   `ACE_LOG_LEVEL` (Optional, can be overridden in shell): Defaults are often set in `docker-compose.yaml` but can be overridden.
*   `ACE_THIRD_PARTY_LOG_LEVEL` (Optional, can be overridden in shell).
*   `ACE_RESOURCE_SUBDIRECTORY` (Optional, can be overridden in shell).
*   RabbitMQ variables (`ACE_RABBITMQ_HOSTNAME`, `ACE_RABBITMQ_USERNAME`, `ACE_RABBITMQ_PASSWORD`): These generally use defaults defined in `docker-compose.yaml` that are suitable for the Docker network but can be overridden from the shell if you're using an external RabbitMQ instance.

The `docker-compose.yaml` file itself defines which `ACE_RESOURCE_NAME` is used for each service (e.g., `aspirational_layer` service gets `ACE_RESOURCE_NAME: layer_1`).

## Configuration Summary

*   **Core Application Logic (`src/main.py`):**
    *   Driven by `ACE_RESOURCE_NAME`.
    *   Uses `ACE_RESOURCE_SUBDIRECTORY`.
*   **Dockerized System (`docker-compose.yaml`):**
    *   Defines multiple services, each running `src/main.py` with a specific `ACE_RESOURCE_NAME`.
    *   Requires `OPENAI_API_KEY` from the host environment.
    *   Manages RabbitMQ configuration for inter-service communication.
    *   Manages logging levels.
*   **Secrets:** The primary secret is `OPENAI_API_KEY`, which should be handled securely and provided as an environment variable.

# Overall Program Behavior in ACE_PRIME/HelloAF

This document describes the overall behavior of the ACE_PRIME/HelloAF system, focusing on its layered architecture, how Large Language Models (LLMs) like OpenAI's GPT are used, and how communication between different components (layers/agents) is handled.

## System Architecture

The ACE_PRIME/HelloAF system implements a layered cognitive architecture. Each "layer" (e.g., Aspirational, Global Strategy, Agent Model) is an instance of an ACE **Resource** (specifically, a class derived from `ace.framework.layer.Layer`).

*   **Dockerized Services:** In a typical deployment (using `docker-compose.yaml` and managed by `resource_manager.py`), each layer runs as a separate Docker service.
*   **Common Entry Point:** Each service uses `ACE_PRIME/HelloAF/src/main.py` as its entry point, configured by the `ACE_RESOURCE_NAME` environment variable (e.g., `layer_1`, `layer_2`) to load the correct layer-specific Python class.

## Communication Backbone: AMQP (RabbitMQ)

Inter-layer communication is the cornerstone of the ACE framework and is handled via an AMQP message broker (RabbitMQ).

*   **Managed by `Resource` Class:** The `ace.framework.resource.Resource` base class provides all the necessary machinery for:
    *   Connecting to RabbitMQ (using `ace.amqp.AMQPConnectionManager`).
    *   Declaring exchanges and queues based on a configuration (likely `ACE_PRIME/HelloAF/src/ace/amqp/messaging_config.yaml`).
    *   Publishing messages to specific exchanges.
    *   Subscribing to queues and routing incoming messages to appropriate handler methods within the layer.
*   **Asynchronous Operations:** AMQP operations are handled asynchronously in a dedicated thread within each resource, using the `aio_pika` library.
*   **Message Structure:** Messages exchanged between layers are typically YAML or JSON strings encoded as bytes. They generally include:
    *   `type`: Category of the message (e.g., `data`, `control`, `request`, `response`, `telemetry`, `command`).
    *   `resource`: A dictionary specifying `source` and `destination` layer names.
    *   `timestamp`: Time of message creation.
    *   `message`: The actual payload/content.

## Layer Operation Cycle: The "Execute" Cascade

The system operates on an event-driven model, primarily driven by an "execute" event that cascades down through the layers.

1.  **Initiation:**
    *   Layer 1 (Aspirational Layer) has a special `begin_work()` method that can be triggered to kickstart the entire process. This method typically involves an initial LLM interaction to set overall goals.
    *   After its initial processing, Layer 1 sends an `"execute"` event to its southbound pathway.
2.  **Event Handling (`handle_event`):**
    *   When a layer receives an `"execute"` event (typically from its northern layer or an initial trigger):
        *   It calls its `agent_run_layer()` method.
3.  **Message Aggregation (`agent_run_layer`):**
    *   This method is responsible for gathering all messages that the layer has received from its subscribed AMQP queues (e.g., data from north, data from south, control messages) since its last execution. These messages are stored internally by the `Resource` base class.
4.  **Core Logic (`process_layer_messages`):**
    *   `agent_run_layer()` then calls `process_layer_messages(control_messages, data_messages, ...)`, passing the collected messages.
    *   This is where the layer's primary intelligence, heavily reliant on LLM interaction, resides.

## OpenAI LLM Interaction (within `process_layer_messages`)

Each layer uses an LLM (specifically OpenAI's GPT models via `ace.framework.llm.gpt.GPT`) to process its input messages and generate output messages.

1.  **Prompt Construction:**
    *   A detailed prompt is dynamically constructed using `jinja2` templates. This prompt typically includes:
        *   **Identity:** A system message defining the role, purpose, and personality of the current layer (loaded from `.md` files in `ACE_PRIME/HelloAF/src/ace/resources/core/hello_layers/prompts/identities/`).
        *   **Context:** General contextual information about the ACE framework (e.g., from `ACE_PRIME/HelloAF/src/ace/resources/core/hello_layers/prompts/templates/ace_context.md`).
        *   **Layer-Specific Instructions:** Detailed instructions for how the current layer should process inputs and generate outputs (from `.../prompts/templates/layer_instructions.md` or more specific versions like `l1_layer_instructions.md`).
        *   **Input Messages:** The content of the messages received from other layers (northbound data, southbound responses, control signals, telemetry data).
        *   **Output Formatting Guidance:** Implicitly or explicitly, the prompt guides the LLM to produce output in a structured format (often JSON) that the system can parse. Templates from `.../prompts/outputs/` might assist here.
2.  **LLM Call:**
    *   The composed prompt (as a list of `GptMessage` dictionaries) is sent to the OpenAI API using `self.llm.create_conversation_completion(model, llm_messages)`.
    *   The `model` (e.g., "gpt-3.5-turbo", "gpt-4") is usually defined in the layer's `settings`.
    *   The `OPENAI_API_KEY` environment variable must be correctly set for this call to succeed.
3.  **Response Processing:**
    *   The raw textual response from the LLM is first logged for debugging and auditing via `self.resource_log()`.
    *   The system expects the LLM's response to be parsable, often as a JSON string. `ace.framework.util.parse_json()` is used to convert this text into a Python dictionary/list.
    *   The `self.parse_req_resp_messages(llm_messages_from_json)` method then interprets this parsed structure. This method is crucial as it determines which parts of the LLM's output are intended for layers above (northbound) and which for layers below (southbound), and what their message types are.

## Message Forwarding and Cascade Continuation

1.  **Publishing Outputs:**
    *   The northbound and southbound messages, as determined from the LLM's processed response, are then built into the standard ACE message format using `self.build_message()`.
    *   These messages are published back to the AMQP bus using `self.push_pathway_message_to_publisher_local_queue()`, which routes them to the appropriate exchanges for consumption by other layers.
2.  **Continuing the Cascade:**
    *   After a layer completes its processing cycle (including LLM interaction and publishing its own messages), its `handle_event` method (for the "execute" event) typically concludes by sending a new `"execute"` event to its southbound pathway: `self.send_event_to_pathway("southbound", "execute")`.
    *   This triggers the next layer down in the hierarchy to begin its `agent_run_layer` and `process_layer_messages` cycle.

## Overall Data Flow Example

1.  Layer 1 (Aspirational) might generate a high-level goal.
2.  This goal, as data, flows south to Layer 2 (Global Strategy).
3.  Layer 2 receives this, processes it with its own identity and instructions (via an LLM call), and might break it down into strategic components. These are sent further south and potentially some feedback or status north.
4.  This process continues down through Layer 6 (Task Prosecution), with each layer refining and acting upon the information using its specialized prompts and LLM interactions.
5.  Responses and results can also flow northbound, providing feedback to higher layers.

This event-driven, cascading execution model, powered by LLM intelligence at each layer and interconnected by an AMQP messaging system, allows the ACE_PRIME/HelloAF framework to implement complex, multi-stage cognitive processes.

+++
title = "Autonomous Robot System and AI Agent System"
date = 2026-06-11T00:00:00-07:00
draft = false
description = "An in-progress local MVP for a robot operations assistant built with LangGraph, RAG, FastAPI, Streamlit, SQLite, ChromaDB, and mock robot tools."
image = "/images/portfolio/autonomous-robot-llm-system.png"
tags = ["Robotics", "AI Agent", "LangGraph", "RAG", "FastAPI", "Streamlit", "SQLite", "ChromaDB"]
categories = ["AI Robotics", "Software Engineering"]
toc = true
+++

## Overview

This project is an in-progress Agentic Robot Operations Assistant: a local AI system designed to help users interact with a simulated robot operations workflow through natural language. The assistant can answer robot operation questions, troubleshoot navigation issues, check robot status, list available destinations, and create or manage navigation requests.

The goal is not only to build a chatbot. The goal is to build an AI engineering system that connects language understanding, retrieval-augmented generation, backend APIs, tool execution, database state, and safety validation into one coherent robotics-oriented application.

The current version is a local MVP. It does not control a real robot yet. Instead, it uses mock robot tools and SQLite database state to simulate robot operations safely. This makes the system easier to test, explain, and extend before connecting it to real robot middleware such as ROS2 or Nav2.

## Motivation

Robot operation systems often require users to understand navigation goals, maps, localization, task states, failure modes, and operation manuals. For a non-expert operator, this can be difficult. Even for developers, switching between documentation, backend APIs, logs, and status tools can slow down debugging.

I wanted to build an assistant that can act as a robot operations copilot. Instead of forcing the user to manually search through documentation or remember exact API commands, the user can ask questions in natural language:

- What is the robot status?
- What destinations are available?
- Create a request to go to Lab A.
- Start REQ-ABC123.
- The robot cannot reach the goal. What should I check?
- How do I add a new destination?

The assistant then decides whether the message should be handled as a documentation question, a troubleshooting question, or a structured robot operation command.

The important design principle is that LLMs should understand language and explain results, while deterministic backend code should validate and control safety-sensitive robot actions. This keeps the system more reliable than a simple chatbot that directly decides what to do.

## System Architecture

The system is built around a layered architecture:

```text
User message
-> Streamlit UI or FastAPI client
-> FastAPI /chat endpoint
-> Input understanding layer
-> Validation and safety layer
-> LangGraph workflow
-> RAG Agent, Tool Agent, Clarification Agent, or Unknown Agent
-> Tool, database, retriever, or LLM provider
-> Structured response
-> SQLite logging and UI display
```

The main components are:

- Streamlit frontend
- FastAPI backend
- LangGraph workflow
- Input understanding layer
- RAG pipeline with ChromaDB
- LLM provider layer
- Mock robot tools
- SQLite database
- Evaluation script

This structure makes the system more than a single prompt-based assistant. Each layer has a clear responsibility.

## Frontend Layer

The frontend is built with Streamlit. It provides a simple demo interface where users can chat with the assistant and inspect system state.

The UI displays chat messages, robot status, available destinations, recent navigation requests, recent chat logs, tool results, RAG sources, intent and entity metadata, understanding mode, and confidence.

The frontend communicates with the backend through HTTP requests. It does not directly import backend modules. This keeps the frontend decoupled from the backend and makes the system easier to replace with another frontend later.

## FastAPI Backend

The backend exposes API endpoints for chat, robot status, destinations, task management, and chat logs. Important endpoints include:

- `POST /chat`
- `GET /robot/status`
- `GET /destinations`
- `GET /tasks`
- `GET /chat/logs`
- `POST /tasks/{request_id}/start`
- `POST /tasks/{request_id}/complete`
- `POST /tasks/{request_id}/cancel`
- `POST /tasks/{request_id}/fail`

The `/chat` endpoint is the main entry point for natural language interaction. It passes the user message into the LangGraph workflow, receives a structured response, saves the interaction to SQLite, and returns the result to the frontend.

The backend is designed to stay thin. Its job is to receive requests, call the graph or tool functions, persist logs, and return structured responses. Business logic is kept in agents, tools, and database helper modules.

## LangGraph Workflow

LangGraph is used as the main orchestration layer. It gives the project an explicit agent workflow instead of a hidden prompt chain.

The graph starts by running an input understanding node. This node converts the raw user message into structured fields such as:

- intent
- entities
- confidence
- need_clarification
- understanding_mode
- validation_errors

After that, the graph routes the message based on the detected intent.

Tool-related intents go to the Tool Agent. Documentation and troubleshooting intents go to the RAG Agent. Missing information goes to the Clarification Agent. Unknown or unsupported requests go to a safe fallback response.

The expected routing structure is:

- `robot_status` -> Tool Agent
- `list_destinations` -> Tool Agent
- `create_navigation_request` -> Tool Agent
- `start_navigation_request` -> Tool Agent
- `complete_navigation_request` -> Tool Agent
- `cancel_navigation_request` -> Tool Agent
- `fail_navigation_request` -> Tool Agent
- `manual_qa` -> RAG Agent
- `troubleshooting` -> RAG Agent
- `clarification` -> Clarification Agent
- `unknown` -> Unknown Agent

This design makes the system easier to debug because the workflow is explicit. The router does not directly inspect raw text. It only looks at the structured intent created by the input understanding layer.

## Input Understanding Layer

The input understanding layer converts natural language into structured meaning. It supports three modes:

1. Rule-based mode
2. LLM mode
3. Hybrid mode

Rule-based mode uses string matching and regular expressions. It is fast, cheap, predictable, and easy to test. For clear commands such as "What is the robot status?" or "Start REQ-ABC123", rule-based mode works well.

LLM mode asks the configured language model to return structured JSON. This is better for flexible natural language. For example, the user might say, "Could you make the robot head over to the demo room?" The LLM can classify this as a request to create a navigation request and extract "Demo Room" as the destination.

Hybrid mode uses rules first. If rules find a clear intent, the system uses the rule result. If rules return unknown, the system asks the LLM. This is the recommended default because it combines stability with flexibility.

The key rule is that LLM output is never trusted directly. It must pass validation before it can trigger tool execution.

## Validation and Safety Layer

The validation layer is one of the most important parts of the project. The system validates:

- Whether the intent is allowed
- Whether the destination exists
- Whether the request ID has a valid format
- Whether required entities are present
- Whether a navigation request can move from one lifecycle state to another

For example, if the LLM suggests creating a navigation request to an invalid destination such as "Moon Base", the backend should reject it or ask for clarification. The system should not execute the action just because the LLM produced a confident answer.

The design rule is:

```text
LLM = language understanding and explanation
Rules = safety and validation
Tools = deterministic execution boundary
Database = record of state
LangGraph = workflow routing
```

This separation is what makes the project more robust than a normal chatbot.

## RAG Agent

The RAG Agent handles manual questions and troubleshooting questions.

The current RAG pipeline uses local Markdown documents stored under `data/docs`. These documents are ingested into ChromaDB. When the user asks a question, the retriever finds relevant chunks and passes them to the LLM to generate a grounded answer.

Example manual questions include:

- How do I add a new destination?
- What is the START page?
- How do I create a navigation request?

Example troubleshooting questions include:

- The robot cannot reach the goal. What should I check?
- The robot is stuck and not moving.

For troubleshooting, the RAG Agent can also call the mock robot status tool and include current robot status in the answer. This allows the assistant to combine documentation retrieval with live or simulated system state.

The RAG Agent should answer from retrieved documents and provided robot status. It should not invent robot facts that are not in the context.

## Tool Agent

The Tool Agent handles structured robot operation intents. Current tool functions include:

- `check_robot_status()`
- `list_destinations()`
- `create_navigation_request(destination)`
- `start_navigation_request(request_id)`
- `complete_navigation_request(request_id)`
- `cancel_navigation_request(request_id)`
- `fail_navigation_request(request_id)`

These tools are mock tools in the current MVP. They update or read local state, but they do not control a real robot.

For example, creating a navigation request inserts a row into SQLite and returns a request ID. Starting a request changes the request status from `created` to `running`. Completing, cancelling, and failing a request also update database state according to allowed lifecycle transitions.

The navigation request lifecycle is:

- `created` -> `running`
- `running` -> `completed`
- `created` -> `cancelled`
- `running` -> `cancelled`
- `running` -> `failed`

Invalid transitions should be rejected. For example, a completed request should not be started again.

## SQLite Persistence

SQLite is used to store chat logs and navigation request records. The database currently stores chat history, navigation requests, request statuses, and timestamps.

This gives the system persistent state across interactions. The frontend can show recent chat logs and recent navigation requests by calling backend endpoints.

SQLite is enough for a local MVP. For a larger system, the database layer could be refactored into a cleaner service or repository layer.

## LLM Provider Layer

The project includes a unified LLM client abstraction. The goal is to allow the rest of the codebase to call a single `generate_text(messages)` function without needing to know which provider is being used.

The current design supports:

- DeepSeek through an OpenAI-compatible API
- Ollama through a local HTTP API
- A fallback or no-LLM mode for local testing

This keeps the system flexible. Hosted models can be used for stronger output quality, while local models can be used for demos or offline development.

## Example Flow: Creating a Navigation Request

A typical flow starts with a user message such as:

> Create a request to go to Lab A.

The system processes the message as follows:

1. Streamlit sends the message to the FastAPI `/chat` endpoint.
2. FastAPI calls the LangGraph workflow.
3. The input understanding layer detects the intent as `create_navigation_request`.
4. The destination entity is extracted as `Lab A`.
5. Validation checks that `Lab A` is an allowed destination.
6. LangGraph routes the state to the Tool Agent.
7. The Tool Agent calls `create_navigation_request("Lab A")`.
8. The tool creates a request ID and inserts a new row into SQLite.
9. The response returns the request ID, destination, and status.
10. FastAPI saves the chat log.
11. Streamlit displays the assistant answer and metadata.

The final response might be:

> Created navigation request REQ-ABC123 to Lab A.

This flow shows how natural language becomes a validated structured action.

## Example Flow: Troubleshooting

A troubleshooting flow starts with a user message such as:

> The robot cannot reach the goal. What should I check?

The system processes the message as follows:

1. The input understanding layer classifies the intent as `troubleshooting`.
2. LangGraph routes the message to the RAG Agent.
3. The retriever searches relevant troubleshooting documents in ChromaDB.
4. The RAG Agent calls the mock robot status tool.
5. The LLM generates a troubleshooting checklist using retrieved context and robot status.
6. The assistant returns the answer, sources, and status information.

This flow demonstrates the value of combining RAG with tool access. The assistant can answer using both documentation and current system state.

## Current Status

The project is currently a strong local MVP. Completed or mostly completed components include:

- FastAPI backend
- Streamlit frontend
- LangGraph workflow
- Rule-based, LLM-based, and hybrid input understanding design
- RAG pipeline with ChromaDB
- Local Markdown documentation
- LLM provider abstraction
- Mock robot tools
- SQLite chat logs
- SQLite navigation request records
- Navigation request lifecycle states
- Evaluation script
- Tool result and metadata display in the frontend

The current system should be understood as a simulated robot operations assistant, not a production robot controller.

## Current Limitations

The biggest limitation is that the current tools do not control a real robot. They only update database state. This is intentional for the MVP because it makes the system safer and easier to test.

Other limitations include:

- No ROS2 or Nav2 integration yet
- No human confirmation layer yet
- No authentication or role-based permission system
- No real telemetry from hardware
- Robot status is still mostly static
- Destination list is hardcoded
- Multi-turn clarification memory is not finished
- LLM structured output parsing should be made stricter
- Logging does not yet capture all useful metadata
- Evaluation needs more safety and LLM test cases

These limitations are important because robot systems are safety-sensitive. Before connecting to a real robot, the system needs stronger confirmation, validation, logging, and adapter boundaries.

## Next Steps

The next major feature should be a human confirmation layer.

For example:

> User: Start REQ-ABC123.  
> Assistant: You are about to start request REQ-ABC123 to Lab A. Please confirm.  
> User: Yes.  
> Assistant: Request REQ-ABC123 is now running.

This is important because starting, cancelling, failing, or completing a robot task may become high-impact once real robot integration is added.

Other planned improvements include:

- Add stronger safety rejection categories
- Reject low-level control commands such as direct velocity commands
- Add strict Pydantic schemas for LLM structured output
- Add more evaluation cases
- Improve database logging
- Make mock robot status dynamic
- Add a robot adapter interface
- Implement a future ROS2/Nav2 adapter
- Add screenshots and polished README documentation
- Consider a React frontend, MCP tools, or voice mode after the safe MVP is stable

## Future ROS2 Integration

The current project should not connect ROS2 directly inside the existing mock tools. A better design is to create a robot adapter interface.

The adapter could define methods such as:

- `get_status()`
- `list_destinations()`
- `create_request()`
- `start_request()`
- `cancel_request()`
- `get_task_status()`

Then the project can have two implementations:

- `MockRobotAdapter` for local demos and testing
- `ROS2RobotAdapter` for future real robot integration

This would keep the current system testable while making real robot integration possible later.

## What I Learned

This project helped me practice building an AI system as an engineering application instead of a simple demo.

The main lessons were:

1. Agent workflows need clear boundaries.
2. LLMs are useful for language understanding, but they should not directly control safety-sensitive actions.
3. RAG becomes more useful when combined with system state and tool access.
4. A good AI application needs backend design, database state, evaluation, and error handling.
5. For robotics, safety and validation must be designed early, even in a mock system.
6. A portfolio project is stronger when it clearly explains what is real, what is simulated, and what the next engineering step is.

## Conclusion

The Agentic Robot Operations Assistant is a robotics-oriented AI engineering project that combines LangGraph, RAG, FastAPI, Streamlit, SQLite, ChromaDB, LLM providers, and mock robot tools into one system.

The current version is not a real robot controller yet. It is a safe local MVP that simulates robot operations and demonstrates the architecture needed for a future real robot operations assistant.

The most important design idea is that the LLM helps users communicate naturally, while deterministic validation and backend tools control what actions are allowed. This separation makes the system safer, easier to debug, and more realistic as a foundation for future ROS2/Nav2 integration.

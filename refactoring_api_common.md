# Refactoring Guide: API & Common Logic Separation

## Overview
This document outlines the architectural changes made to decouple the `keep-api` from shared logic, enabling the `keep-event-handler` (and other future microservices) to operate independently without pulling in the entire API application stack.

## Key Commits & Changes
The following key commits implemented the separation, excluding minor runtime fixes:

### 1. `c73af428`: Decouple API/EventHandler
The primary refactoring effort that established the separation.
*   **Moved Core Modules**: Transferred `metrics`, `workflows` to `keep.common.core` to resolve circular dependencies.
*   **Created `keep.common.core.init`**: Centralized initialization logic (DB, tenants) so the worker can start without the API.
*   **Refactored Config**: Decoupled `keep.api.config` from the main API app to allow independent configuration loading.

### 2. `68f8f900`: Separate Files
*   **Moved Dependencies**: Transferred `keep.api.core.dependencies` to `keep.common.core.dependencies` to share injection logic.

## Detailed Relocation Logic

### 1. Start-up & Initialization
*   **Problem**: `keep/event_handler/core/bootstrap.py` imported `keep.api.config`, which triggered a chain reaction importing the entire FastAPI application.
*   **Solution**: Extracted `init_services()`, `provision_resources()`, and `try_create_single_tenant()` into a new module `keep/common/core/init.py`.

### 2. Shared Core Modules
Modules required by both the API and the Worker were moved to `keep.common`:
*   **Metrics** (`keep.common.core.metrics`): Both services need to record metrics.
*   **Workflows** (`keep.common.core.workflows`): validation and parsing logic is needed by the worker for processing.
*   **Dependencies** (`keep.common.core.dependencies`): Shared dependency injection for database and other services.

### 3. Configuration & Auth
*   **Problem**: `keep/api/config.py` had a circular dependency with `keep.api.api` regarding `AUTH_TYPE`.
*   **Solution**: `AUTH_TYPE` reading was moved/refactored so `keep.common` can access it without importing the API.
*   **Renaming**: The configuration object in `keep.common` was renamed to `starlette_config` to avoid namespace collisions.

## Final State
*   **`keep.common`**: The foundation layer. Contains database models, shared core logic (metrics, workflows, usage), and initialization routines.
*   **`keep.api`**: The HTTP layer. Depends on `keep.common`. Contains FastAPI routes and API-specific logic.
*   **`keep.event_handler`**: The Worker layer. Depends on `keep.common`. Does **not** import `keep.api`.

## Verification
*   **Docker**: Both `Dockerfile.api` and `Dockerfile.event_handler` now explicitly copy `keep/searchengine`.
*   **Workflows**: `.github/workflows/verify-separate-worker-logic.yml` verifies this specific architecture.

.PHONY: all up down test env preflight init-db build push build-and-push set-latest set-dev setup-builder help

# --- Paths ---
ROOT := $(shell git rev-parse --show-toplevel)
ENV_FILE := $(ROOT)/.env
ENV_EXAMPLE := $(ROOT)/deploy/env-example

# DockerHub configuration
DOCKERHUB_USER ?= vexaai
IMAGE_NAME ?= vexa-lite
TAG ?= dev

# Multi-arch platforms
PLATFORMS ?= linux/amd64,linux/arm64

# Buildx builder name
BUILDER_NAME ?= vexa-multiarch

# Paths (Makefile is in deploy/lite/, Dockerfile is in same dir, context is repo root)
DOCKERFILE := $(shell pwd)/Dockerfile.lite
CONTEXT := $(shell pwd)/../..
LOCAL_TAG := $(DOCKERHUB_USER)/$(IMAGE_NAME):$(TAG)

# Generate unique timestamp tag: YYMMDD-HHMM
UNIQUE_TAG := $(shell date +%y%m%d-%H%M)
UNIQUE_DOCKERHUB_TAG := $(DOCKERHUB_USER)/$(IMAGE_NAME):$(UNIQUE_TAG)
LAST_TAG_FILE := .last-tag

define set_tag
	@echo "Setting '$(1)' tag to point to $(2)..."
	docker buildx imagetools create -t $(DOCKERHUB_USER)/$(IMAGE_NAME):$(1) $(DOCKERHUB_USER)/$(IMAGE_NAME):$(2)
	@echo "'$(1)' tag updated to $(2)"
endef

help:
	@echo "Vexa Lite Docker Build & Push (Multi-Arch)"
	@echo ""
	@echo "Usage:"
	@echo "  make build              - Build locally for current platform (tagged as dev)"
	@echo "  make push               - Build multi-arch (amd64+arm64), push with timestamp tag, update 'dev' tag"
	@echo "  make build-and-push     - Alias for push"
	@echo "  make set-latest [TAG=]  - Set 'latest' tag to specified tag (or last pushed)"
	@echo "  make set-dev [TAG=]     - Set 'dev' tag to specified tag (or last pushed)"
	@echo "  make setup-builder      - Create/bootstrap the multi-arch buildx builder"
	@echo ""
	@echo "Configuration:"
	@echo "  PLATFORMS=$(PLATFORMS)"

setup-builder:
	@if ! docker buildx inspect $(BUILDER_NAME) >/dev/null 2>&1; then \
		echo "Creating buildx builder '$(BUILDER_NAME)'..."; \
		docker buildx create --name $(BUILDER_NAME) --driver docker-container --bootstrap; \
	else \
		echo "Builder '$(BUILDER_NAME)' already exists"; \
	fi

build:
	@echo "Building $(LOCAL_TAG) for current platform..."
	docker build -f $(DOCKERFILE) -t $(LOCAL_TAG) $(CONTEXT)
	@echo "Build complete: $(LOCAL_TAG)"

push: setup-builder
	@echo "Building multi-arch $(UNIQUE_DOCKERHUB_TAG) for $(PLATFORMS)..."
	docker buildx build \
		--builder $(BUILDER_NAME) \
		--platform $(PLATFORMS) \
		-f $(DOCKERFILE) \
		-t $(UNIQUE_DOCKERHUB_TAG) \
		--push \
		$(CONTEXT)
	@echo "$(UNIQUE_TAG)" > $(LAST_TAG_FILE)
	@echo "Push complete: $(UNIQUE_DOCKERHUB_TAG)"
	$(call set_tag,dev,$(UNIQUE_TAG))

build-and-push: push

set-latest:
	@if [ -z "$(TAG)" ]; then \
		if [ -f $(LAST_TAG_FILE) ]; then \
			TAG_TO_USE=$$(cat $(LAST_TAG_FILE)); \
		else \
			echo "Error: No TAG specified and $(LAST_TAG_FILE) not found"; exit 1; \
		fi; \
	else \
		TAG_TO_USE=$(TAG); \
	fi; \
	echo "Setting 'latest' tag to point to $$TAG_TO_USE..."; \
	docker buildx imagetools create -t $(DOCKERHUB_USER)/$(IMAGE_NAME):latest $(DOCKERHUB_USER)/$(IMAGE_NAME):$$TAG_TO_USE; \
	echo "'latest' tag updated to $$TAG_TO_USE"

set-dev:
	@if [ -z "$(TAG)" ]; then \
		if [ -f $(LAST_TAG_FILE) ]; then \
			TAG_TO_USE=$$(cat $(LAST_TAG_FILE)); \
		else \
			echo "Error: No TAG specified and $(LAST_TAG_FILE) not found"; exit 1; \
		fi; \
	else \
		TAG_TO_USE=$(TAG); \
	fi; \
	echo "Setting 'dev' tag to point to $$TAG_TO_USE..."; \
	docker buildx imagetools create -t $(DOCKERHUB_USER)/$(IMAGE_NAME):dev $(DOCKERHUB_USER)/$(IMAGE_NAME):$$TAG_TO_USE; \
	echo "'dev' tag updated to $$TAG_TO_USE"

# ============================================================================
# Deployment (make lite from repo root)
# ============================================================================

-include $(ENV_FILE)

all: env preflight up init-db test
	@HOST_IP=$$(hostname -I 2>/dev/null | awk '{print $$1}'); \
	HOST=$${HOST_IP:-localhost}; \
	DASH_PORT=$$(grep -E '^DASHBOARD_HOST_PORT=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DASH_PORT=$${DASH_PORT:-3000}; \
	echo ""; \
	echo "Vexa Lite is running."; \
	echo "  Dashboard: http://$$HOST:$$DASH_PORT"; \
	echo "  API docs:  http://$$HOST:8056/docs"

# --- Environment (reuse compose env target) ---
env:
	@$(MAKE) --no-print-directory -C $(ROOT)/deploy/compose env

# --- Preflight (reuse compose preflight target) ---
preflight:
	@$(MAKE) --no-print-directory -C $(ROOT)/deploy/compose preflight

# --- Up ---
up:
	@if ! docker info > /dev/null 2>&1; then echo "ERROR: Docker is not running."; exit 1; fi
	@IMAGE_TAG=$$(grep -E '^IMAGE_TAG=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	IMAGE_TAG=$${IMAGE_TAG:-latest}; \
	REMOTE_DB=$$(grep -E '^REMOTE_DB=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DB_HOST=$$(grep -E '^DB_HOST=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DB_PORT=$$(grep -E '^DB_PORT=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DB_NAME=$$(grep -E '^DB_NAME=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DB_USER=$$(grep -E '^DB_USER=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DB_PASSWORD=$$(grep -E '^DB_PASSWORD=' $(ENV_FILE) 2>/dev/null | cut -d= -f2-); \
	DATABASE_URL=$$(grep -E '^DATABASE_URL=' $(ENV_FILE) 2>/dev/null | cut -d= -f2-); \
	PG_PORT=$$(grep -E '^LITE_POSTGRES_PORT=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	PG_PORT=$${PG_PORT:-$$(grep -E '^POSTGRES_HOST_PORT=' $(ENV_FILE) 2>/dev/null | cut -d= -f2)}; \
	PG_PORT=$${PG_PORT:-5432}; \
	DB_NAME=$${DB_NAME:-vexa}; \
	DB_USER=$${DB_USER:-postgres}; \
	DB_PASSWORD=$${DB_PASSWORD:-postgres}; \
	if [ -n "$$DATABASE_URL" ]; then \
		USE_MANAGED_POSTGRES=false; \
	elif [ "$$REMOTE_DB" = "true" ]; then \
		USE_MANAGED_POSTGRES=false; \
		DB_HOST=$${DB_HOST:-localhost}; \
		DB_PORT=$${DB_PORT:-5432}; \
		DATABASE_URL=$$(DB_USER="$$DB_USER" DB_PASSWORD="$$DB_PASSWORD" DB_HOST="$$DB_HOST" DB_PORT="$$DB_PORT" DB_NAME="$$DB_NAME" python3 -c 'import os; from urllib.parse import quote; print("postgresql://{}:{}@{}:{}/{}".format(quote(os.environ["DB_USER"], safe=""), quote(os.environ["DB_PASSWORD"], safe=""), os.environ["DB_HOST"], os.environ["DB_PORT"], quote(os.environ["DB_NAME"], safe="")))'); \
	else \
		USE_MANAGED_POSTGRES=true; \
		DB_HOST=localhost; \
		DB_PORT=$$PG_PORT; \
		DB_NAME=vexa; \
		DB_USER=postgres; \
		DB_PASSWORD=postgres; \
		DATABASE_URL=postgresql://postgres:postgres@localhost:$$PG_PORT/vexa; \
	fi; \
	REDIS_URL=$$(grep -E '^REDIS_URL=' $(ENV_FILE) 2>/dev/null | cut -d= -f2-); \
	REDIS_URL=$${REDIS_URL:-redis://localhost:6379/0}; \
	REDIS_URL_NO_SCHEME=$${REDIS_URL#*://}; \
	REDIS_HOSTPORTDB=$${REDIS_URL_NO_SCHEME#*@}; \
	REDIS_HOSTPORT=$${REDIS_HOSTPORTDB%%/*}; \
	REDIS_HOST=$$(grep -E '^REDIS_HOST=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	REDIS_PORT=$$(grep -E '^REDIS_PORT=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	REDIS_HOST=$${REDIS_HOST:-$${REDIS_HOSTPORT%%:*}}; \
	if [ "$$REDIS_HOSTPORT" != "$${REDIS_HOSTPORT#*:}" ]; then \
		REDIS_PORT=$${REDIS_PORT:-$${REDIS_HOSTPORT#*:}}; \
	fi; \
	REDIS_PORT=$${REDIS_PORT:-6379}; \
	DISABLE_TTS=$$(grep -E '^DISABLE_TTS_SERVICE=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DISABLE_TTS=$${DISABLE_TTS:-$$(grep -E '^LITE_DISABLE_TTS=' $(ENV_FILE) 2>/dev/null | cut -d= -f2)}; \
	DISABLE_TTS=$${DISABLE_TTS:-false}; \
	LITE_EXTERNAL_REDIS=$$(grep -E '^LITE_EXTERNAL_REDIS=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DISABLE_INTERNAL_REDIS=$$(grep -E '^DISABLE_INTERNAL_REDIS=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DISABLE_INTERNAL_REDIS=$${DISABLE_INTERNAL_REDIS:-$$LITE_EXTERNAL_REDIS}; \
	DISABLE_INTERNAL_REDIS=$${DISABLE_INTERNAL_REDIS:-false}; \
	if [ "$$DISABLE_TTS" = "true" ]; then \
		TTS_ENVS="-e DISABLE_TTS_SERVICE=true -e PIPER_DEFAULT_VOICES= -e PIPER_LOAD_VOICES= -e PIPER_PRELOAD_STRICT=false"; \
	else \
		TTS_ENVS=""; \
	fi; \
	TX_URL=$$(grep -E '^LITE_TRANSCRIPTION_SERVICE_URL=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	TX_URL=$${TX_URL:-$$(grep -E '^TRANSCRIPTION_SERVICE_URL=' $(ENV_FILE) 2>/dev/null | cut -d= -f2)}; \
	TX_TOKEN=$$(grep -E '^LITE_TRANSCRIPTION_SERVICE_TOKEN=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	TX_TOKEN=$${TX_TOKEN:-$$(grep -E '^TRANSCRIPTION_SERVICE_TOKEN=' $(ENV_FILE) 2>/dev/null | cut -d= -f2)}; \
	ADMIN_TK=$$(grep -E '^ADMIN_TOKEN=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	ADMIN_TK=$${ADMIN_TK:-changeme}; \
	DASH_PORT=$$(grep -E '^DASHBOARD_HOST_PORT=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DASH_PORT=$${DASH_PORT:-3000}; \
	if [ "$$USE_MANAGED_POSTGRES" = "true" ]; then \
		echo "Starting PostgreSQL on 127.0.0.1:$$PG_PORT..."; \
		if docker ps -q -f name=vexa-postgres 2>/dev/null | grep -q .; then \
			echo "  PostgreSQL already running"; \
		else \
			docker rm -f vexa-postgres 2>/dev/null || true; \
			docker run -d --name vexa-postgres --network host \
				-e POSTGRES_DB=vexa \
				-e POSTGRES_USER=postgres \
				-e POSTGRES_PASSWORD=postgres \
				postgres:17-alpine \
				-c listen_addresses=127.0.0.1 \
				-c port=$$PG_PORT \
				-c idle_in_transaction_session_timeout=60000 > /dev/null; \
		fi; \
		echo "  Waiting for PostgreSQL..."; \
		PG_READY=0; \
		for i in $$(seq 1 12); do \
			if ! docker ps -q -f name=vexa-postgres 2>/dev/null | grep -q .; then \
				echo "ERROR: PostgreSQL container exited during startup."; \
				docker logs --tail 80 vexa-postgres 2>/dev/null || true; \
				exit 1; \
			fi; \
			if docker exec vexa-postgres pg_isready -h 127.0.0.1 -p $$PG_PORT -U postgres -d vexa -q 2>/dev/null; then \
				PG_READY=1; \
				break; \
			fi; \
			sleep 5; \
		done; \
		if [ "$$PG_READY" != "1" ]; then \
			echo "ERROR: PostgreSQL did not become ready on 127.0.0.1:$$PG_PORT."; \
			docker logs --tail 80 vexa-postgres 2>/dev/null || true; \
			exit 1; \
		fi; \
		echo "  ✓ PostgreSQL ready"; \
	else \
		echo "Using external PostgreSQL: $$DATABASE_URL"; \
	fi; \
	echo "Pulling $(DOCKERHUB_USER)/$(IMAGE_NAME):$$IMAGE_TAG..."; \
	docker pull $(DOCKERHUB_USER)/$(IMAGE_NAME):$$IMAGE_TAG; \
	echo "Starting Vexa Lite..."; \
	docker rm -f vexa-lite 2>/dev/null || true; \
	docker run -d --name vexa-lite --shm-size=2g --network host \
		-v vexa-lite-recordings:/var/lib/vexa/recordings \
		-e DATABASE_URL="$$DATABASE_URL" \
		-e DB_HOST="$$DB_HOST" \
		-e DB_PORT="$$DB_PORT" \
		-e DB_NAME="$$DB_NAME" \
		-e DB_USER="$$DB_USER" \
		-e DB_PASSWORD="$$DB_PASSWORD" \
		-e DB_SSL_MODE=disable \
		-e REDIS_URL="$$REDIS_URL" \
		-e REDIS_HOST="$$REDIS_HOST" \
		-e REDIS_PORT="$$REDIS_PORT" \
		-e DISABLE_INTERNAL_REDIS="$$DISABLE_INTERNAL_REDIS" \
		-e ADMIN_API_TOKEN="$$ADMIN_TK" \
		-e DASHBOARD_HOST_PORT="$$DASH_PORT" \
		-e VEXA_AUTH_COOKIE_NAME=vexa-token-lite \
		-e VEXA_USER_INFO_COOKIE_NAME=vexa-user-info-lite \
		-e TRANSCRIPTION_SERVICE_URL="$$TX_URL" \
		-e TRANSCRIPTION_SERVICE_TOKEN="$$TX_TOKEN" \
		$$TTS_ENVS \
		$(DOCKERHUB_USER)/$(IMAGE_NAME):$$IMAGE_TAG > /dev/null; \
	echo "  Waiting for services (up to 2 min)..."; \
	LITE_READY=0; \
	for i in $$(seq 1 24); do \
		if ! docker ps -q -f name=vexa-lite 2>/dev/null | grep -q .; then \
			echo "ERROR: Vexa Lite container exited during startup."; \
			docker logs --tail 120 vexa-lite 2>/dev/null || true; \
			exit 1; \
		fi; \
		if curl -sf -o /dev/null http://localhost:8056/docs 2>/dev/null; then \
			LITE_READY=1; \
			break; \
		fi; \
		sleep 5; \
	done; \
	if [ "$$LITE_READY" != "1" ]; then \
		echo "ERROR: Vexa Lite API did not become ready at http://localhost:8056/docs."; \
		docker logs --tail 120 vexa-lite 2>/dev/null || true; \
		exit 1; \
	fi; \
	echo "  ✓ Vexa Lite running"

# --- Init DB ---
init-db:
	@echo "Running schema sync..."
	@docker exec vexa-lite python3 -c \
		"import asyncio; from admin_models.database import init_db; asyncio.run(init_db())" 2>/dev/null || true
	@docker exec vexa-lite python3 -c \
		"import asyncio; from meeting_api.database import init_db; asyncio.run(init_db())" 2>/dev/null || true
	@echo "  ✓ Schema sync complete"

# --- Test ---
test:
	@echo "Checking services..."
	@DASH_PORT=$$(grep -E '^DASHBOARD_HOST_PORT=' $(ENV_FILE) 2>/dev/null | cut -d= -f2); \
	DASH_PORT=$${DASH_PORT:-3000}; \
	for ep in "8056:API Gateway" "8057:Admin API" "$$DASH_PORT:Dashboard"; do \
		PORT=$${ep%%:*}; \
		NAME=$${ep##*:}; \
		if curl -sf -o /dev/null http://localhost:$$PORT/ 2>/dev/null; then \
			echo "  ✓ $$NAME (port $$PORT)"; \
		else \
			echo "  ✗ $$NAME not responding (port $$PORT)"; \
		fi; \
	done

# --- Down ---
down:
	@echo "Stopping Vexa Lite..."
	@docker stop vexa-lite 2>/dev/null || true
	@docker rm vexa-lite 2>/dev/null || true
	@docker stop vexa-postgres 2>/dev/null || true
	@docker rm vexa-postgres 2>/dev/null || true
	@echo "  ✓ Stopped"

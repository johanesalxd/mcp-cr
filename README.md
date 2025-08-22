# Zoo MCP Tutorial on Cloud Run 🦁🐧🐻

A comprehensive tutorial for deploying a Model Context Protocol (MCP) server to Google Cloud Run, featuring a zoo animal database with interactive tools.

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Project Setup](#project-setup)
- [Local Development](#local-development)
- [Cloud Run Deployment](#cloud-run-deployment)
- [Gemini CLI Setup & Testing](#gemini-cli-setup--testing)

## Introduction

This tutorial demonstrates how to build and deploy an MCP (Model Context Protocol) server that provides tools for querying a zoo animal database. The server includes:

- **Tools**: `get_animals_by_species`, `get_animal_details`
- **Prompts**: `find` (locate animals in exhibits)
- **Data**: 35+ animals across different species and enclosures

### Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Gemini CLI    │◄──►│   MCP Server     │◄──►│  Zoo Database   │
│   (Client)      │    │ (FastMCP/Python) │    │   (In-memory)   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
        │                       │
        │              ┌────────▼────────┐
        │              │  Local (dev)    │
        │              │  Cloud Run      │
        └──────────────┤  (production)   │
                       └─────────────────┘
```

## Prerequisites

- **Operating System**: Linux/macOS (Ubuntu recommended)
- **Google Cloud Account** with billing enabled
- **Python 3.13+**
- **Git**

## Environment Setup

### 1. Install pipx and uv

```bash
# Update system packages
sudo apt update

# Install pipx (Python package installer)
sudo apt install pipx
pipx ensurepath

# Install uv (fast Python package manager)
pipx install uv
```

### 2. Verify Installation

```bash
# Check uv installation
uv --version

# Check available Python versions
uv python list
```

## Project Setup

### 1. Initialize Project

```bash
# Create project directory
mkdir zoo-mcp-server
cd zoo-mcp-server

# Initialize with uv
uv init --description "Example of deploying an MCP server on Cloud Run" --bare --python 3.13

# Add FastMCP dependency
uv add fastmcp==2.11.1 --no-sync
```

### 2. Create Virtual Environment

```bash
# Create virtual environment
uv venv --python 3.13

# Install dependencies
uv sync
```

### 3. Project Structure

Your project should have this structure:

```
zoo-mcp-server/
├── .venv/              # Virtual environment
├── server.py           # MCP server implementation
├── Dockerfile          # Container configuration
├── pyproject.toml      # Project dependencies
├── uv.lock            # Dependency lock file
└── README.md          # This file
```

## Local Development

### 1. Run Server Locally

```bash
# Activate virtual environment and run server
uv run server.py
```

The server will start on `http://localhost:8080` by default.

### 2. Configure Local MCP Connection

Create a local configuration for testing:

```bash
# Create Gemini CLI config directory
mkdir -p ~/.gemini

# Configure local MCP server
cat <<EOF > ~/.gemini/settings.json
{
  "mcpServers": {
    "zoo-local": {
      "httpUrl": "http://localhost:8080/mcp/"
    }
  }
}
EOF
```

## Cloud Run Deployment

### 1. Setup Google Cloud Project

```bash
# Set your project ID (replace with your actual project ID)
gcloud config set project [PROJECT_ID]

# Enable required services
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

### 2. Deploy to Cloud Run

```bash
# Deploy the application
gcloud run deploy zoo-mcp-server \
    --no-allow-unauthenticated \
    --region=europe-west1 \
    --source=. \
    --labels=dev-tutorial=codelab-mcp
```

### 3. Setup Authentication

```bash
# Add IAM policy for invoker role
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member=user:$(gcloud config get-value account) \
    --role='roles/run.invoker'

# Get project number and ID token for authentication
export PROJECT_NUMBER=$(gcloud projects describe $GOOGLE_CLOUD_PROJECT --format="value(projectNumber)")
export ID_TOKEN=$(gcloud auth print-identity-token)
```

### 4. Configure Remote MCP Connection

```bash
# Configure Gemini CLI for remote server
cat <<EOF > ~/.gemini/settings.json
{
  "mcpServers": {
    "zoo-remote": {
      "httpUrl": "https://zoo-mcp-server-$PROJECT_NUMBER.europe-west1.run.app/mcp/",
      "headers": {
        "Authorization": "Bearer $ID_TOKEN"
      }
    }
  }
}
EOF
```

## Gemini CLI Setup & Testing

### 1. Install Gemini CLI

For easy installation, check out the 1-click deployment:
**🚀 [Gemini CLI 1-click Deployment](https://github.com/johanesalxd/gemini-cli-1c)**

You can use the interactive mode by using the command below:

```bash
gemini

# Inside the Gemini CLI
/mcp

# Ask the questions
Where can I find penguins?

# Confirm the MCP execution: Yes, always allow all tools from server "zoo-remote"
```

### 2. Test MCP Tools

Once Gemini CLI is installed and configured, you can test the MCP server:

#### Get Animals by Species

```bash
# Ask Gemini to find all lions
gemini -p "Show me all the lions in the zoo"

# Find all penguins
gemini -p "List all penguins and their details"

# Get elephants
gemini -p "What elephants are in the zoo?"
```

#### Get Specific Animal Details

```bash
# Find a specific animal
gemini -p "Tell me about Leo the lion"

# Get details for Waddles
gemini -p "What can you tell me about Waddles?"
```

#### Use the Find Prompt

```bash
# Find where an animal is located
gemini -p "Where can I find the penguins?"

# Locate specific exhibits
gemini -p "Which trail should I take to see the lions?"
```

### 3. Example Interactions

**Query**: "How many penguins are in the zoo?"

**Expected Response**: The MCP server will use `get_animals_by_species("penguin")` and return information about all 6 penguins: Waddles, Pip, Skipper, Chilly, Pingu, and Noot.

**Query**: "Where is Leo located?"

**Expected Response**: Using `get_animal_details("Leo")` and the `find` prompt, it will respond that Leo can be found in The Big Cat Plains on the Savannah Heights trail.
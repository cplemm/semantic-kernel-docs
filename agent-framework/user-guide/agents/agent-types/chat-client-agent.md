---
title: Agent based on any IChatClient
description: Learn how to use the Microsoft Agent Framework with any IChatClient implementation.
zone_pivot_groups: programming-languages
author: westey-m
ms.topic: tutorial
ms.author: westey
ms.date: 09/25/2025
ms.service: semantic-kernel
---

# Agent based on any IChatClient

::: zone pivot="programming-language-csharp"

The Microsoft Agent Framework supports creating agents for any inference service that provides a [`Microsoft.Extensions.AI.IChatClient`](/dotnet/ai/microsoft-extensions-ai#the-ichatclient-interface) implementation. This means that there is a very broad range of services that can be used to create agents, including open source models that can be run locally.

In this document, we will use Ollama as an example.

## Getting Started

Add the required NuGet packages to your project.

```powershell
dotnet add package Microsoft.Agents.AI --prerelease
```

You will also need to add the package for the specific `IChatClient` implementation you want to use. In this example, we will use [OllamaSharp](https://www.nuget.org/packages/OllamaSharp/).

```powershell
dotnet add package OllamaSharp
```

## Creating a ChatClientAgent

To create an agent based on the `IChatClient` interface, you can use the `ChatClientAgent` class.
The `ChatClientAgent` class takes `IChatClient` as a constructor parameter.

First, create an `OllamaApiClient` to access the Ollama service.

```csharp
using System;
using Microsoft.Agents.AI;
using OllamaSharp;

using OllamaApiClient chatClient = new(new Uri("http://localhost:11434"), "phi3");
```

The `OllamaApiClient` implements the `IChatClient` interface, so you can use it to create a `ChatClientAgent`.

```csharp
AIAgent agent = new ChatClientAgent(
    chatClient,
    instructions: "You are good at telling jokes.",
    name: "Joker");
```

> [!IMPORTANT]
> To ensure that you get the most out of your agent, make sure to choose a service and model that is well-suited for conversational tasks and supports function calling.

## Using the Agent

The agent is a standard `AIAgent` and supports all standard agent operations.

See the [Agent getting started tutorials](../../../tutorials/overview.md) for more information on how to run and interact with agents.

::: zone-end
::: zone pivot="programming-language-python"

The Microsoft Agent Framework supports creating agents for any inference service that provides a chat client implementation compatible with the `ChatClientProtocol`. This means that there is a very broad range of services that can be used to create agents, including open source models that can be run locally.

## Getting Started

Add the required Python packages to your project.

```bash
pip install agent-framework
```

You may also need to add packages for specific chat client implementations you want to use:

```bash
# For Azure AI
pip install agent-framework-azure-ai

# For custom implementations
# Install any required dependencies for your custom client
```

## Built-in Chat Clients

The framework provides several built-in chat client implementations:

### OpenAI Chat Client

```python
from agent_framework import ChatAgent
from agent_framework.openai import OpenAIChatClient

# Create agent using OpenAI
agent = ChatAgent(
    chat_client=OpenAIChatClient(model_id="gpt-4o"),
    instructions="You are a helpful assistant.",
    name="OpenAI Assistant"
)
```

### Azure OpenAI Chat Client

```python
from agent_framework import ChatAgent
from agent_framework.azure import AzureOpenAIChatClient

# Create agent using Azure OpenAI
agent = ChatAgent(
    chat_client=AzureOpenAIChatClient(
        model_id="gpt-4o",
        endpoint="https://your-resource.openai.azure.com/",
        api_key="your-api-key"
    ),
    instructions="You are a helpful assistant.",
    name="Azure OpenAI Assistant"
)
```

### Azure AI Agent Client

```python
from agent_framework import ChatAgent
from agent_framework.azure import AzureAIAgentClient
from azure.identity.aio import AzureCliCredential

# Create agent using Azure AI
async with AzureCliCredential() as credential:
    agent = ChatAgent(
        chat_client=AzureAIAgentClient(async_credential=credential),
        instructions="You are a helpful assistant.",
        name="Azure AI Assistant"
    )
```

> [!IMPORTANT]
> To ensure that you get the most out of your agent, make sure to choose a service and model that is well-suited for conversational tasks and supports function calling if you plan to use tools.

## Using the Agent

The agent supports all standard agent operations.

See the [Agent getting started tutorials](../../../tutorials/overview.md) for more information on how to run and interact with agents.

::: zone-end

> [!div class="nextstepaction"]
> [Agent2Agent](./a2a-agent.md)

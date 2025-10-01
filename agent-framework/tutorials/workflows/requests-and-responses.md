---
title: Handle Requests and Responses in Workflows
description: Learn how to handle requests and responses in workflows using the Agent Framework.
zone_pivot_groups: programming-languages
author: TaoChenOSU
ms.topic: tutorial
ms.author: taochen
ms.date: 09/29/2025
ms.service: agent-framework
---

# Handle Requests and Responses in Workflows

This tutorial demonstrates how to handle requests and responses in workflows using the Agent Framework Workflows. You'll learn how to create interactive workflows that can pause execution to request input from external sources (like humans or other systems) and then resume once a response is provided.

::: zone pivot="programming-language-csharp"

In .NET, human-in-the-loop workflows use `InputPort` and external request handling to pause execution and gather user input. This pattern enables interactive workflows where the system can request information from external sources during execution.

## Key Components

### InputPort and External Requests

An `InputPort` acts as a bridge between the workflow and external input sources. When the workflow needs input, it generates a `RequestInfoEvent` that your application handles:

```csharp
// Create an InputPort for handling human input requests
InputPort numberInputPort = InputPort.Create<NumberSignal, int>("GuessNumber");
```

### Signal Types

Define signal types to communicate different request types:

```csharp
/// <summary>
/// Signals used for communication between guesses and the JudgeExecutor.
/// </summary>
internal enum NumberSignal
{
    Init,     // Initial guess request
    Above,    // Previous guess was too high
    Below,    // Previous guess was too low
}
```

### Workflow Executor

Create executors that process user input and provide feedback:

```csharp
/// <summary>
/// Executor that judges the guess and provides feedback.
/// </summary>
internal sealed class JudgeExecutor() : ReflectingExecutor<JudgeExecutor>("Judge"), IMessageHandler<int>
{
    private readonly int _targetNumber;
    private int _tries;

    public JudgeExecutor(int targetNumber) : this()
    {
        _targetNumber = targetNumber;
    }

    public async ValueTask HandleAsync(int message, IWorkflowContext context)
    {
        _tries++;
        if (message == _targetNumber)
        {
            await context.YieldOutputAsync($"{_targetNumber} found in {_tries} tries!")
                         .ConfigureAwait(false);
        }
        else if (message < _targetNumber)
        {
            await context.SendMessageAsync(NumberSignal.Below).ConfigureAwait(false);
        }
        else
        {
            await context.SendMessageAsync(NumberSignal.Above).ConfigureAwait(false);
        }
    }
}
```

## Building the Workflow

Connect the InputPort and executor in a feedback loop:

```csharp
internal static ValueTask<Workflow<NumberSignal>> GetWorkflowAsync()
{
    // Create the executors
    InputPort numberInputPort = InputPort.Create<NumberSignal, int>("GuessNumber");
    JudgeExecutor judgeExecutor = new(42);

    // Build the workflow by connecting executors in a loop
    return new WorkflowBuilder(numberInputPort)
        .AddEdge(numberInputPort, judgeExecutor)
        .AddEdge(judgeExecutor, numberInputPort)
        .WithOutputFrom(judgeExecutor)
        .BuildAsync<NumberSignal>();
}
```

## Executing the Interactive Workflow

Handle external requests during workflow execution:

```csharp
private static async Task Main()
{
    // Create the workflow
    var workflow = await WorkflowHelper.GetWorkflowAsync().ConfigureAwait(false);

    // Execute the workflow
    StreamingRun handle = await InProcessExecution.StreamAsync(workflow, NumberSignal.Init).ConfigureAwait(false);
    await foreach (WorkflowEvent evt in handle.WatchStreamAsync().ConfigureAwait(false))
    {
        switch (evt)
        {
            case RequestInfoEvent requestInputEvt:
                // Handle human input request from the workflow
                ExternalResponse response = HandleExternalRequest(requestInputEvt.Request);
                await handle.SendResponseAsync(response).ConfigureAwait(false);
                break;

            case WorkflowOutputEvent outputEvt:
                // The workflow has yielded output
                Console.WriteLine($"Workflow completed with result: {outputEvt.Data}");
                return;
        }
    }
}
```

## Request Handling

Process different types of input requests:

```csharp
private static ExternalResponse HandleExternalRequest(ExternalRequest request)
{
    if (request.DataIs<NumberSignal>())
    {
        switch (request.DataAs<NumberSignal>())
        {
            case NumberSignal.Init:
                int initialGuess = ReadIntegerFromConsole("Please provide your initial guess: ");
                return request.CreateResponse(initialGuess);
            case NumberSignal.Above:
                int lowerGuess = ReadIntegerFromConsole("You previously guessed too large. Please provide a new guess: ");
                return request.CreateResponse(lowerGuess);
            case NumberSignal.Below:
                int higherGuess = ReadIntegerFromConsole("You previously guessed too small. Please provide a new guess: ");
                return request.CreateResponse(higherGuess);
        }
    }

    throw new NotSupportedException($"Request {request.PortInfo.RequestType} is not supported");
}

private static int ReadIntegerFromConsole(string prompt)
{
    while (true)
    {
        Console.Write(prompt);
        string? input = Console.ReadLine();
        if (int.TryParse(input, out int value))
        {
            return value;
        }
        Console.WriteLine("Invalid input. Please enter a valid integer.");
    }
}
```

## Implementation Concepts

### RequestInfoEvent Flow

1. **Workflow Execution**: The workflow processes until it needs external input
2. **Request Generation**: InputPort generates a `RequestInfoEvent` with the request details
3. **External Handling**: Your application catches the event and gathers user input
4. **Response Submission**: Send an `ExternalResponse` back to continue the workflow
5. **Workflow Resumption**: The workflow continues processing with the provided input

### Workflow Lifecycle

- **Streaming Execution**: Use `StreamAsync` to monitor events in real-time
- **Event Handling**: Process `RequestInfoEvent` for input requests and `WorkflowOutputEvent` for completion
- **Response Coordination**: Match responses to requests using the workflow's response handling mechanism

### Implementation Flow

1. **Workflow Initialization**: The workflow starts by sending a `NumberSignal.Init` to the InputPort.

2. **Request Generation**: The InputPort generates a `RequestInfoEvent` requesting an initial guess from the user.

3. **Workflow Pause**: The workflow pauses and waits for external input while the application handles the request.

4. **Human Response**: The external application collects user input and sends an `ExternalResponse` back to the workflow.

5. **Processing and Feedback**: The `JudgeExecutor` processes the guess and either completes the workflow or sends a new signal (Above/Below) to request another guess.

6. **Loop Continuation**: The process repeats until the correct number is guessed.

### Framework Benefits

- **Type Safety**: Strong typing ensures request-response contracts are maintained
- **Event-Driven**: Rich event system provides visibility into workflow execution
- **Pausable Execution**: Workflows can pause indefinitely while waiting for external input
- **State Management**: Workflow state is preserved across pause-resume cycles
- **Flexible Integration**: InputPorts can integrate with any external input source (UI, API, console, etc.)

### Complete Sample

For the complete working implementation, see the [Human-in-the-Loop Basic sample](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/GettingStarted/Workflows/HumanInTheLoop/HumanInTheLoopBasic).

This pattern enables building sophisticated interactive applications where users can provide input at key decision points within automated workflows.

::: zone-end

::: zone pivot="programming-language-python"

### What You'll Build

You'll create an interactive number guessing game workflow that demonstrates request-response patterns:

- An AI agent that makes intelligent guesses
- A `RequestInfoExecutor` that pauses the workflow to request human input
- A turn manager that coordinates between the agent and human interactions
- Interactive console input/output for real-time feedback

### Prerequisites

- Python 3.10 or later
- Azure OpenAI deployment configured
- Azure CLI authentication configured (`az login`)
- Basic understanding of Python async programming

### Key Concepts

#### RequestInfoExecutor

`RequestInfoExecutor` is a specialized workflow component that:
- Pauses workflow execution to request external information
- Emits a `RequestInfoEvent` with typed payload
- Resumes execution after receiving a correlated response
- Preserves request-response correlation via unique request IDs

#### Request-Response Flow

1. Workflow sends a `RequestInfoMessage` to `RequestInfoExecutor`
2. `RequestInfoExecutor` emits a `RequestInfoEvent` 
3. External system (human, API, etc.) processes the request
4. Response is sent back via `send_responses_streaming()`
5. Workflow resumes with the response data

### Setting Up the Environment

First, install the required packages:

```bash
pip install agent-framework-core
pip install azure-identity
pip install pydantic
```

### Define Request and Response Models

Start by defining the data structures for request-response communication:

```python
import asyncio
from dataclasses import dataclass
from pydantic import BaseModel

from agent_framework import (
    AgentExecutor,
    AgentExecutorRequest,
    AgentExecutorResponse,
    ChatMessage,
    Executor,
    RequestInfoEvent,
    RequestInfoExecutor,
    RequestInfoMessage,
    RequestResponse,
    Role,
    WorkflowBuilder,
    WorkflowContext,
    WorkflowOutputEvent,
    WorkflowRunState,
    WorkflowStatusEvent,
    handler,
)
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

@dataclass
class HumanFeedbackRequest(RequestInfoMessage):
    """Request message for human feedback in the guessing game."""
    prompt: str = ""
    guess: int | None = None

class GuessOutput(BaseModel):
    """Structured output from the AI agent with response_format enforcement."""
    guess: int
```

The `HumanFeedbackRequest` inherits from `RequestInfoMessage`, which provides:
- Strong typing for request payloads
- Forward-compatible validation
- Clear correlation semantics with responses
- Contextual fields (like the previous guess) for rich UI prompts

### Create the Turn Manager

The turn manager coordinates the flow between the AI agent and human:

```python
class TurnManager(Executor):
    """Coordinates turns between the AI agent and human player.
    
    Responsibilities:
    - Start the game by requesting the agent's first guess
    - Process agent responses and request human feedback
    - Handle human feedback and continue the game or finish
    """

    def __init__(self, id: str | None = None):
        super().__init__(id=id or "turn_manager")

    @handler
    async def start(self, _: str, ctx: WorkflowContext[AgentExecutorRequest]) -> None:
        """Start the game by asking the agent for an initial guess."""
        user = ChatMessage(Role.USER, text="Start by making your first guess.")
        await ctx.send_message(AgentExecutorRequest(messages=[user], should_respond=True))

    @handler
    async def on_agent_response(
        self,
        result: AgentExecutorResponse,
        ctx: WorkflowContext[HumanFeedbackRequest],
    ) -> None:
        """Handle the agent's guess and request human guidance."""
        # Parse structured model output (defensive default if agent didn't reply)
        text = result.agent_run_response.text or ""
        last_guess = GuessOutput.model_validate_json(text).guess if text else None

        # Craft a clear human prompt that defines higher/lower relative to agent's guess
        prompt = (
            f"The agent guessed: {last_guess if last_guess is not None else text}. "
            "Type one of: higher (your number is higher than this guess), "
            "lower (your number is lower than this guess), correct, or exit."
        )
        await ctx.send_message(HumanFeedbackRequest(prompt=prompt, guess=last_guess))

    @handler
    async def on_human_feedback(
        self,
        feedback: RequestResponse[HumanFeedbackRequest, str],
        ctx: WorkflowContext[AgentExecutorRequest, str],
    ) -> None:
        """Continue the game or finish based on human feedback."""
        reply = (feedback.data or "").strip().lower()
        # Use the correlated request's guess to avoid extra state reads
        last_guess = getattr(feedback.original_request, "guess", None)

        if reply == "correct":
            await ctx.yield_output(f"Guessed correctly: {last_guess}")
            return

        # Provide feedback to the agent for the next guess
        user_msg = ChatMessage(
            Role.USER,
            text=f'Feedback: {reply}. Return ONLY a JSON object matching the schema {{"guess": <int 1..10>}}.',
        )
        await ctx.send_message(AgentExecutorRequest(messages=[user_msg], should_respond=True))
```

### Build the Workflow

Create the main workflow that connects all components:

```python
async def main() -> None:
    # Create the chat agent with structured output enforcement
    chat_client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = chat_client.create_agent(
        instructions=(
            "You guess a number between 1 and 10. "
            "If the user says 'higher' or 'lower', adjust your next guess. "
            'You MUST return ONLY a JSON object exactly matching this schema: {"guess": <integer 1..10>}. '
            "No explanations or additional text."
        ),
        response_format=GuessOutput,
    )

    # Create workflow components
    turn_manager = TurnManager(id="turn_manager")
    agent_exec = AgentExecutor(agent=agent, id="agent")
    request_info_executor = RequestInfoExecutor(id="request_info")

    # Build the workflow graph
    workflow = (
        WorkflowBuilder()
        .set_start_executor(turn_manager)
        .add_edge(turn_manager, agent_exec)  # Ask agent to make/adjust a guess
        .add_edge(agent_exec, turn_manager)  # Agent's response goes back to coordinator
        .add_edge(turn_manager, request_info_executor)  # Ask human for guidance
        .add_edge(request_info_executor, turn_manager)  # Feed human guidance back to coordinator
        .build()
    )

    # Execute the interactive workflow
    await run_interactive_workflow(workflow)

async def run_interactive_workflow(workflow):
    """Run the workflow with human-in-the-loop interaction."""
    pending_responses: dict[str, str] | None = None
    completed = False
    workflow_output: str | None = None

    print("🎯 Number Guessing Game")
    print("Think of a number between 1 and 10, and I'll try to guess it!")
    print("-" * 50)

    while not completed:
        # First iteration uses run_stream("start")
        # Subsequent iterations use send_responses_streaming with pending responses
        stream = (
            workflow.send_responses_streaming(pending_responses) 
            if pending_responses 
            else workflow.run_stream("start")
        )
        
        # Collect events for this turn
        events = [event async for event in stream]
        pending_responses = None

        # Process events to collect requests and detect completion
        requests: list[tuple[str, str]] = []  # (request_id, prompt)
        for event in events:
            if isinstance(event, RequestInfoEvent) and isinstance(event.data, HumanFeedbackRequest):
                # RequestInfoEvent for our HumanFeedbackRequest
                requests.append((event.request_id, event.data.prompt))
            elif isinstance(event, WorkflowOutputEvent):
                # Capture workflow output when yielded
                workflow_output = str(event.data)
                completed = True

        # Check workflow status
        pending_status = any(
            isinstance(e, WorkflowStatusEvent) and e.state == WorkflowRunState.IN_PROGRESS_PENDING_REQUESTS
            for e in events
        )
        idle_with_requests = any(
            isinstance(e, WorkflowStatusEvent) and e.state == WorkflowRunState.IDLE_WITH_PENDING_REQUESTS
            for e in events
        )

        if pending_status:
            print("🔄 State: IN_PROGRESS_PENDING_REQUESTS (requests outstanding)")
        if idle_with_requests:
            print("⏸️  State: IDLE_WITH_PENDING_REQUESTS (awaiting human input)")

        # Handle human requests if any
        if requests and not completed:
            responses: dict[str, str] = {}
            for req_id, prompt in requests:
                print(f"\n🤖 {prompt}")
                answer = input("👤 Enter higher/lower/correct/exit: ").lower()
                
                if answer == "exit":
                    print("👋 Exiting...")
                    return
                responses[req_id] = answer
            pending_responses = responses

    # Show final result
    print(f"\n🎉 {workflow_output}")
```

### Running the Example

For the complete working implementation, see the [Human-in-the-Loop Guessing Game sample](https://github.com/microsoft/agent-framework/blob/main/python/samples/getting_started/workflow/human-in-the-loop/guessing_game_with_human_input.py).

### How It Works

1. **Workflow Initialization**: The workflow starts with the `TurnManager` requesting an initial guess from the AI agent.

2. **Agent Response**: The AI agent makes a guess and returns structured JSON, which flows back to the `TurnManager`.

3. **Human Request**: The `TurnManager` processes the agent's guess and sends a `HumanFeedbackRequest` to the `RequestInfoExecutor`.

4. **Workflow Pause**: The `RequestInfoExecutor` emits a `RequestInfoEvent` and the workflow pauses, waiting for human input.

5. **Human Response**: The external application collects human input and sends responses back using `send_responses_streaming()`.

6. **Resume and Continue**: The workflow resumes, the `TurnManager` processes the human feedback, and either ends the game or sends another request to the agent.

### Key Benefits

- **Structured Communication**: Type-safe request and response models prevent runtime errors
- **Correlation**: Request IDs ensure responses are matched to the correct requests
- **Pausable Execution**: Workflows can pause indefinitely while waiting for external input
- **State Preservation**: Workflow state is maintained across pause-resume cycles
- **Event-Driven**: Rich event system provides visibility into workflow status and transitions

This pattern enables building sophisticated interactive applications where AI agents and humans collaborate seamlessly within structured workflows.

::: zone-end

## Next Steps

> [!div class="nextstepaction"]
> [Learn about checkpointing and resuming workflows](checkpointing-and-resuming.md)

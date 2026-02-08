# HackSynth Integration with BoxPwnr

## Overview

HackSynth has been integrated into BoxPwnr as a new autonomous strategy. This integration allows BoxPwnr users to leverage HackSynth's dual-module architecture (Planner + Summarizer) for iterative CTF solving.

## Architecture

### Integration Approach

The integration follows a similar pattern to Claude Code:
- **Direct Python Import**: Uses HackSynth's `PentestAgent` class directly (no subprocess calls)
- **Container Adapter**: `DockerContainerAdapter` translates BoxPwnr's `DockerExecutor` interface to HackSynth's expected container API
- **Autonomous Execution**: Runs the complete plan/execute/summarize loop internally
- **No Neptune.ai**: Logging is disabled by default (HackSynth normally logs to Neptune.ai for experiment tracking)

### Key Components

1. **HackSynthStrategy** (`hacksynth.py`): Main strategy implementation
   - Inherits from `LLMStrategy` base class
   - Manages the HackSynth agent lifecycle
   - Handles flag detection and extraction

2. **DockerContainerAdapter**: Bridge between BoxPwnr and HackSynth
   - Translates `execute_command()` → `exec_run()`
   - Handles timeout parsing and command execution
   - Returns Docker SDK-compatible result objects

3. **Configuration**: Auto-generated from BoxPwnr's system prompt
   - Planner prompts: Generate bash commands iteratively
   - Summarizer prompts: Compile comprehensive action history
   - Model settings: Temperature, top_p, max_tokens, etc.

## Usage

### Basic Command

```bash
boxpwnr --platform htb --target Meow --strategy hacksynth --model gpt-5
```

### With Custom Parameters

```bash
boxpwnr \
  --platform htb \
  --target Meow \
  --strategy hacksynth \
  --model gpt-5 \
  --max-turns 30 \
  --max-cost 5.0 \
  --debug
```

### Supported Models

HackSynth supports:
- **OpenAI Models**: `gpt-5`, `gpt-5-mini`
- **Local Models**: Any HuggingFace model (requires `llm_model_local=true`)

## How It Works

### Execution Flow

1. **Initialization**: BoxPwnr initializes HackSynth with:
   - System prompt from platform (HTB, PortSwigger, etc.)
   - Container adapter for command execution
   - LLM configuration

2. **Autonomous Loop** (runs for `max_turns` iterations):
   ```
   For each turn:
     a. Planner generates a bash command
     b. Command is executed in Docker container
     c. Summarizer processes output into history
     d. Check for flag in output or summary
     e. If flag found → success, else → next turn
   ```

3. **Termination**: Loop ends when:
   - Flag is found (success)
   - Max turns reached (failure)
   - Error occurs

### Flag Detection

Flags are automatically detected using regex patterns:
- `HTB{...}`
- `picoCTF{...}`
- `flag{...}`, `FLAG{...}`, `CTF{...}`
- Generic: `[A-Za-z0-9_]{...}`

## Implementation Details

### Container Adapter

The adapter translates between BoxPwnr and HackSynth interfaces:

**HackSynth expects:**
```python
result = container.exec_run("timeout 10s /bin/bash -c 'command'")
output = result.output.decode('utf-8')
```

**BoxPwnr provides:**
```python
result = executor.execute_command("command", timeout=10)
stdout = result['stdout']
```

**Adapter bridges:**
```python
# Parse timeout from HackSynth's command format
# Execute via BoxPwnr's executor
# Return Docker SDK-compatible object
```

### Token Tracking

HackSynth tracks tokens through the LLM pipeline:
- **Local models**: Counts tokens via tokenizer
- **OpenAI models**: Uses response.usage metrics

BoxPwnr aggregates:
- `total_input_tokens`: Sum of all planner + summarizer input tokens
- `total_output_tokens`: Sum of all planner + summarizer output tokens

### Message History

All interactions are stored as LangChain messages:
- **AIMessage**: Planner outputs, Summarizer outputs
- **HumanMessage**: Command execution results

Each message includes metadata:
```python
{
    "timestamp": "2024-...",
    "step": 5,
    "phase": "planner" | "execution" | "summarizer",
    "command": "ls -la"  # for planner phase
}
```

## Differences from Original HackSynth

1. **No Neptune.ai logging**: Disabled to simplify integration
2. **No custom container creation**: Uses BoxPwnr's existing container
3. **No file downloads**: BoxPwnr handles challenge file setup
4. **Simplified configuration**: Auto-generated from system prompt
5. **Integrated reporting**: Uses BoxPwnr's reporting system

## Advantages

1. **Proven Architecture**: HackSynth is a published research framework
2. **Modular Design**: Separate Planner and Summarizer modules
3. **Iterative Approach**: Can self-correct based on previous outputs
4. **No External Dependencies**: Pure Python integration
5. **Compatible with BoxPwnr**: Works with all platforms (HTB, PortSwigger, etc.)

## Limitations

1. **Docker Only**: Requires Docker executor (like Claude Code)
2. **OpenAI Focused**: Best tested with OpenAI models
3. **No Interactive Mode**: Purely autonomous (unlike Claude Code's --interactive flag)
4. **No Cost Tracking**: Cost estimation is rough (not using tokencost library)

## Future Improvements

- [ ] Add support for local models with better resource management
- [ ] Implement proper cost tracking using tokencost library
- [ ] Add interactive mode for debugging
- [ ] Support SSH executor
- [ ] Add configurable planner/summarizer prompts via CLI
- [ ] Integrate with BoxPwnr's attempt analyzer

## References

- **HackSynth Paper**: https://arxiv.org/abs/2412.01778
- **Original Repository**: https://github.com/aielte-research/HackSynth
- **Citation**:
  ```bibtex
  @misc{muzsai2024hacksynthllmagentevaluation,
        title={HackSynth: LLM Agent and Evaluation Framework for Autonomous Penetration Testing}, 
        author={Lajos Muzsai and David Imolai and András Lukács},
        year={2024},
        eprint={2412.01778},
        archivePrefix={arXiv},
        primaryClass={cs.CR},
        url={https://arxiv.org/abs/2412.01778}, 
  }
  ```



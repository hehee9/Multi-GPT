---
name: multi-gpt
description: Run PC-local parallel-reasoning Multi GPT through the async `multi_gpt_start` and `multi_gpt_status` MCP tools. Use when the user asks for Multi GPT, Super GPT, multi-agent reasoning, multiple independent approaches with merge/judge/refine, complex design comparison, deep code or architecture review, or long-form problem solving that benefits from Planner to parallel Solver to Refiner to Loop to Organizer orchestration. Do not use for simple one-shot advice; use Codex sub-agents or normal reasoning instead.
---

# Multi GPT

## Overview

Use `multi_gpt_start` plus `multi_gpt_status` for PC-local parallel reasoning. Multi GPT is not a single Codex consultation call; it starts a background job that orchestrates multiple read-only `codex exec` stage calls as Planner -> parallel Solvers -> parallel Refiners -> Loop(Merger -> Refiner -> Judge) -> Organizer.

## Workflow

Call `multi_gpt_start` when the task needs several independent approaches, comparison, synthesis, or a judge/refine loop. The tool returns quickly with a `job_id`, avoiding the roughly 120-second MCP tool-call limit.

Multi GPT is slow by design. A typical run takes about 5 to 20 minutes, and most normal runs finish around 10 to 15 minutes. Do not poll `multi_gpt_status` every minute. After starting a job, wait several minutes before the first status check, then check only occasionally unless the user explicitly asks for tighter monitoring.

Poll `multi_gpt_status` with that `job_id` until it returns `status: completed` or `status: failed`.

Do not use Multi GPT for ordinary short answers, quick code questions, or one-off model consultation; the user expects those to use normal Codex reasoning or sub-agents.

Pass inputs to `multi_gpt_start` as follows:

- `prompt`: preserve the user's original request as the main prompt.
- `files`: pass absolute local file paths that should be read into context. The MCP server reads files read-only and rejects missing or overly large files.
- `model`: omit unless the user specifies a model. When omitted, the MCP server lets Codex CLI use its configured default model.
- `reasoning_effort`: omit for the default `medium`. Use `none` only when the user explicitly asks for non-reasoning mode. Valid values are `none`, `low`, `medium`, `high`, and `xhigh`.
- `max_iterations`: omit unless the user asks to change the loop budget. The default is `5`.

## Behavior to Preserve

Treat Multi GPT as a PC-local parallel reasoning pipeline:

1. Planner analyzes the problem and creates several approaches.
2. Solvers independently solve each approach in parallel.
3. Initial Refiners improve each Solver result, falling back to the original solution on failure.
4. The loop runs Merger -> Refiner -> Judge up to the iteration limit.
5. Judge can stop early by selecting the best sufficient solution.
6. Organizer converts the selected solution into the final user-facing answer.

When `multi_gpt_status` returns `status: completed`, show `result.final_answer` to the user. Stage summaries and fallback metadata may be used to report reliability or diagnose failures, but do not expose hidden chain-of-thought or raw internal transcripts.

## PC Runtime Constraints

This skill is PC-only. The MCP server invokes local `codex exec` with read-only sandboxing and no approval prompts for each reasoning stage.

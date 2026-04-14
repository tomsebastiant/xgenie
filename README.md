# xgenie

A production-style AI analysis system built using distributed backend and agent orchestration patterns.

---

## Overview

xgenie is a multi-layer system that combines:
- A Java-based data service for structured football event data
- A Python-based AI agent built with LangGraph
- Parallel analysis nodes with shared caching
- LLM-powered reasoning and report generation
- End-to-end observability and tracing

The system is designed to replicate production-grade AI workflows seen in modern distributed systems, focusing on orchestration, concurrency, and tool-based agent execution.
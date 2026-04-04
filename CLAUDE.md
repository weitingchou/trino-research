# Claude Guidelines

## Purpose
This repository is used for tracing and studying data engine source code (Trino, Doris, DataFusion, etc.). All work here is read-only research and note-taking. Each engine's research lives under its own folder at the project root.

## Rules

1. **Read-only submodules**: NEVER commit any changes to source code submodules (e.g., `trino/trino/`). Do not stage, commit, or push anything inside submodule directories. They exist solely for reading and reference.

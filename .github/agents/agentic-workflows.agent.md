---
description: GitHub Agentic Workflows (gh-aw) - Create, debug, and upgrade AI-powered workflows with intelligent prompt routing
disable-model-invocation: true
---

# GitHub Agentic Workflows Agent

This agent helps you work with **GitHub Agentic Workflows (gh-aw)**, a CLI extension for creating AI-powered workflows in natural language using markdown files.

## What This Agent Does

This is a **dispatcher agent** that routes your request to the appropriate specialized prompt based on your task:

- **Creating new workflows**: Routes to `create` prompt
- **Updating existing workflows**: Routes to `update` prompt
- **Debugging workflows**: Routes to `debug` prompt  
- **Upgrading workflows**: Routes to `upgrade-agentic-workflows` prompt
- **Creating report-generating workflows**: Routes to `report` prompt
- **Creating shared components**: Routes to `create-shared-agentic-workflow` prompt

## Files This Applies To

- Workflow files: `.github/workflows/*.md` and `.github/workflows/**/*.md`
- Workflow lock files: `.github/workflows/*.lock.yml`
- Shared components: `.github/workflows/shared/*.md`
- Configuration: https://github.com/github/gh-aw/blob/v0.72.1/.github/aw/github-agentic-workflows.md

## Quick Reference

```bash
gh aw init
gh aw compile [workflow-name]
gh aw run <workflow-name> --ref main
gh aw logs [workflow-name]
gh aw audit <run-id>
```

## Important Notes

- Workflows must be compiled to `.lock.yml` files before running in GitHub Actions
- Bash tools are enabled by default — workflows are sandboxed by the AWF
- Follow security best practices: minimal permissions, explicit network access, no template injection

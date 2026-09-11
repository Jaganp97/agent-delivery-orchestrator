# Multi-Agent Delivery Orchestrator

A file-based, editor-native multi-agent system for shipping data-platform changes (dbt models, orchestrator DAGs, ETL jobs) from ticket to merged PR — with mandatory human approval gates at every irreversible step.

Built as [Cursor](https://cursor.com) `.mdc` rule files. No runtime, no daemon, no database. All agent state lives in git-diffable YAML.

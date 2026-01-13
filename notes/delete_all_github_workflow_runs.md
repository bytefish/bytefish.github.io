title: Deleting all Workflow Runs for a GitHub Action
date: 2026-01-10 18:29
author: Philipp Wagner
tags: notes
category: notes
summary: Deleting all Workflow Runs for a GitHub Action

It took me too long to find this. 😓

If you need to delete all workflow runs of a GitHub Action, you can do it with PowerShell using this GitHub cli command:

```powershell
gh run list --limit 1000 --json databaseId -q '.[].databaseId' | ForEach-Object { gh run delete $_ }
```

Or this Bash / Git Bash GitHub cli command:

```bash
gh run list --limit 1000 --json databaseId -q '.[].databaseId' | xargs -I{} gh run delete {}
```
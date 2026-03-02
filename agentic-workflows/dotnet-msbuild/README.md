# MSBuild Agentic Workflow Templates

These are [GitHub Agentic Workflow](https://github.com/github/gh-aw) templates for MSBuild and .NET build automation.

## Available Workflows

| Workflow | Description | Trigger |
|----------|-------------|---------|
| [msbuild-pr-review](msbuild-pr-review.md) | Reviews MSBuild project file changes for anti-patterns, correctness issues, and modernization opportunities | `/review-msbuild` slash command |

## Setup

1. Install the `gh aw` CLI extension
2. Copy the desired workflow files to your repository's `.github/workflows/` directory
3. Copy the `shared/` directory as well (workflows import from it)
4. Compile: `gh aw compile`
5. Commit both the `.md` and generated `.lock.yml` files
6. Slash-command workflows are invoked by posting the command as a comment on an issue or PR; scheduled workflows run automatically

## Customization

- Adjust `safe-outputs` limits and triggers as needed

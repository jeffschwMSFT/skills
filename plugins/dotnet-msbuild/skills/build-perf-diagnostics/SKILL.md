---
name: build-perf-diagnostics
description: "Reference knowledge for diagnosing MSBuild build performance issues. Only activate in MSBuild/.NET build context. Use when builds are slow, to identify bottlenecks using binary log analysis. Covers timeline analysis, node utilization, expensive targets/tasks, Roslyn analyzer impact, RAR performance, and critical path identification."
---

## Performance Analysis Methodology

1. **Generate a binlog**: `dotnet build /bl`
2. **Analyze the binlog** using MSBuild Structured Log Viewer or equivalent tooling
3. **Get overall picture**: Identify expensive projects, targets, and tasks
4. **Analyze node utilization**: Check the timeline for parallelism
5. **Drill into bottlenecks**: Examine per-project target times and search for patterns
6. **Check analyzers**: Look for Roslyn analyzer overhead in compilation

## Key Metrics and Thresholds

- **Build duration**: what's "normal" — small project <10s, medium <60s, large <5min
- **Node utilization**: ideal is >80% active time across nodes. Low utilization = serialization bottleneck
- **Single target domination**: if one target is >50% of build time, investigate
- **Analyzer time vs compile time**: analyzers should be <30% of Csc task time. If higher, consider removing expensive analyzers
- **RAR time**: ResolveAssemblyReference >5s is concerning. >15s is pathological

## Common Bottlenecks

### 1. ResolveAssemblyReference (RAR) Slowness

- **Symptoms**: RAR taking >5s per project
- **Root causes**: too many assembly references, network-based reference paths, large assembly search paths
- **Fixes**: reduce reference count, use `<DesignTimeBuild>false</DesignTimeBuild>` for RAR-heavy analysis, set `<ResolveAssemblyReferencesSilent>true</ResolveAssemblyReferencesSilent>` for diagnostic
- **Advanced**: `<DesignTimeBuild>` and `<ResolveAssemblyWarnOrErrorOnTargetArchitectureMismatch>`

### 2. Roslyn Analyzers and Source Generators

- **Symptoms**: Csc task takes much longer than expected for file count (>2× clean compile time)
- **Diagnosis**: Check analyzer overhead in the binlog → identify top offenders, compare Csc duration with and without analyzers
- **Fixes**:
  - Conditionally disable in dev: `<RunAnalyzers Condition="'$(ContinuousIntegrationBuild)' != 'true'">false</RunAnalyzers>`
  - Per-configuration: `<RunAnalyzers Condition="'$(Configuration)' == 'Debug'">false</RunAnalyzers>`
  - Code-style only: `<EnforceCodeStyleInBuild Condition="'$(ContinuousIntegrationBuild)' == 'true'">true</EnforceCodeStyleInBuild>`
  - Remove genuinely redundant analyzers from inner loop
  - Severity config in .editorconfig for less critical rules
- **Key principle**: Preserve analyzer enforcement in CI. Never just "remove" analyzers — configure them conditionally.
- **GlobalPackageReference**: Analyzers added via `GlobalPackageReference` in `Directory.Packages.props` apply to ALL projects. Consider if test projects need the same analyzer set as production code.
- **EnforceCodeStyleInBuild**: When set to `true` in `Directory.Build.props`, forces code-style analysis on every build. Should be conditional on CI environment (`ContinuousIntegrationBuild`) to avoid slowing dev inner loop.

### 3. Serialization Bottlenecks (Single-threaded targets)

- **Symptoms**: Node timeline shows most nodes idle while one works
- **Common culprits**: targets without proper dependency declaration, single project on critical path
- **Fixes**: split large projects, optimize the critical path project, ensure proper `BuildInParallel`

### 4. Excessive File I/O (Copy tasks)

- **Symptoms**: Copy task shows high aggregate time
- **Root causes**: copying thousands of files, copying across network drives
- **Fixes**: use hardlinks (`<CreateHardLinksForCopyFilesToOutputDirectoryIfPossible>true</CreateHardLinksForCopyFilesToOutputDirectoryIfPossible>`), reduce CopyToOutputDirectory items, use `<UseCommonOutputDirectory>true</UseCommonOutputDirectory>` when appropriate

### 5. Evaluation Overhead

- **Symptoms**: build starts slow before any compilation
- See: `eval-performance` skill for detailed guidance

### 6. NuGet Restore in Build

- **Symptoms**: restore runs every build even when unnecessary
- **Fix**: separate restore from build: `dotnet restore` then `dotnet build --no-restore`

### 7. Large Project Count

- **Symptoms**: many small projects, each takes minimal time but overhead adds up
- **Consider**: project consolidation, or use `/graph` mode for better scheduling

## Performance Analysis Workflow

Step-by-step workflow for analyzing build performance from a binlog:

1. Open the binlog in MSBuild Structured Log Viewer → note total duration and node count
2. Find expensive projects (sort by exclusive time) → find where time is spent
3. Identify dominant targets → which targets consume the most time
4. Identify dominant tasks → which tasks consume the most time
5. Check the timeline → assess parallelism utilization
6. Check analyzer overhead → compare analyzer time vs compilation time
7. For a specific slow project: examine target breakdown
8. Deep dive: search for specific targets and their timing

## Quick Wins Checklist

- [ ] Use `/maxcpucount` (or `-m`) for parallel builds
- [ ] Separate restore from build (`--no-restore`)
- [ ] Enable hardlinks for Copy
- [ ] Disable analyzers in dev inner loop
- [ ] Check for broken incremental builds (see `incremental-build` skill)
- [ ] Check for bin/obj clashes (see `check-bin-obj-clash` skill)
- [ ] Use graph build (`/graph`) for multi-project solutions

---
name: test-trx
description: Runs .NET tests using dotnet test and collects results to a TRX log file. Use when you need machine-readable test results, CI/CD integration, or detailed test failure analysis.
metadata:
  author: dotnet
  version: "1.0"
---

# Test with TRX Output

Runs .NET tests and collects results to a TRX (Visual Studio Test Results) file for structured analysis, CI/CD integration, or detailed failure investigation.

## When to Use

- Running tests that need machine-readable output
- CI/CD pipelines requiring test result artifacts
- Analyzing test failures with full stack traces and timing data
- Comparing test runs over time
- Integrating with Azure DevOps or other systems that consume TRX files

## When Not to Use

- Quick local test runs where console output is sufficient
- Projects without .NET test infrastructure
- When you need real-time streaming test output only

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Project or solution path | No | Path to test project or solution. Defaults to current directory. |
| Results directory | No | Directory for TRX output. Defaults to `TestResults/`. |
| Filter | No | Test filter expression (e.g., `FullyQualifiedName~UnitTests`) |
| Configuration | No | Build configuration. Defaults to `Debug`. |

## Workflow

### Step 1: Locate the test project

Find the test project or solution file. Look for:
- `*.sln` files in the workspace root
- Projects ending in `.Tests.csproj` or containing test references
- Projects with `<IsTestProject>true</IsTestProject>` in the csproj

### Step 2: Run tests with TRX logger

Execute the test command with TRX output:

```powershell
dotnet test <project-or-solution> --logger "trx;LogFileName=TestResults.trx" --results-directory ./TestResults
```

Optional parameters:
- Add `--filter "<expression>"` for targeted test runs
- Add `--configuration Release` for release builds
- Add `--no-build` if already built
- Add `--blame` to capture crash dumps on test host crash

### Step 3: Locate the TRX file

The TRX file will be in the results directory:
- Default: `./TestResults/TestResults.trx`
- With timestamp: `./TestResults/<guid>/TestResults.trx`

List the results directory to find the exact path:

```powershell
Get-ChildItem -Path ./TestResults -Filter *.trx -Recurse
```

### Step 4: Parse and report results

Read the TRX file to extract results. Key elements in the XML:

- `/TestRun/ResultSummary/@outcome` - Overall pass/fail
- `/TestRun/ResultSummary/Counters` - Test counts (total, passed, failed, etc.)
- `/TestRun/Results/UnitTestResult` - Individual test results
- `/TestRun/Results/UnitTestResult/Output/ErrorInfo` - Failure details

Quick summary extraction:

```powershell
[xml]$trx = Get-Content ./TestResults/TestResults.trx
$counters = $trx.TestRun.ResultSummary.Counters
Write-Host "Total: $($counters.total), Passed: $($counters.passed), Failed: $($counters.failed)"
```

### Step 5: Report failures (if any)

For each failed test, extract:
- Test name from `@testName`
- Error message from `Output/ErrorInfo/Message`
- Stack trace from `Output/ErrorInfo/StackTrace`
- Duration from `@duration`

## Validation

- [ ] TRX file exists in the results directory
- [ ] TRX file is valid XML with `TestRun` root element
- [ ] `ResultSummary/@outcome` reflects actual test results
- [ ] All expected tests appear in the results

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| TRX file not found | Check `--results-directory` path; look for GUID subfolders |
| Multiple TRX files | Use `LogFileName` parameter or sort by timestamp |
| Tests not discovered | Verify test SDK version and project references |
| Build errors before tests | Run `dotnet build` separately first to isolate issues |
| Filter matches no tests | Validate filter syntax; use `--list-tests` to preview |

## TRX Structure Reference

Key XPath expressions for parsing:

| Data | XPath |
|------|-------|
| Outcome | `/TestRun/ResultSummary/@outcome` |
| Total tests | `/TestRun/ResultSummary/Counters/@total` |
| Passed | `/TestRun/ResultSummary/Counters/@passed` |
| Failed | `/TestRun/ResultSummary/Counters/@failed` |
| Test results | `/TestRun/Results/UnitTestResult` |
| Test name | `UnitTestResult/@testName` |
| Test outcome | `UnitTestResult/@outcome` |
| Error message | `UnitTestResult/Output/ErrorInfo/Message` |
| Stack trace | `UnitTestResult/Output/ErrorInfo/StackTrace` |

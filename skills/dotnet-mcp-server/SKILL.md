---
name: dotnet-mcp-server
description: Build MCP (Model Context Protocol) servers in .NET with guidance on transport selection (stdio vs HTTP), target framework choice, project scaffolding, tool authoring, and testing. Use when creating agent-extensibility services for Copilot, Claude, or other AI platforms.
---

# .NET MCP Server Authoring

Build production-ready MCP (Model Context Protocol) servers in .NET with correct transport, framework, and hosting decisions.

## When to Use

- Creating tools or resources for AI agents (GitHub Copilot CLI, VS Code, Claude Desktop, web-based agents)
- Building local CLI extensions with stdio transport
- Hosting shared MCP services over HTTP/SSE for multiple clients
- Integrating .NET APIs or databases with agent workflows

## When Not to Use

- Building MCP clients (use `ModelContextProtocol.Client` instead)
- Creating custom transport implementations (this skill covers stdio and HTTP only)
- Implementing MCP gateway/proxy services (out of scope)

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Use case | Yes | What will consume this server? (Copilot CLI, VS Code, Claude Desktop, web app, etc.) |
| Client model | Yes | Single local user or multiple concurrent clients? |
| Authentication needs | No | Does the HTTP endpoint need auth? (Only for HTTP transport) |
| Deployment target | No | Local dev machine, container, cloud service, or AOT binary |

## Workflow

### Step 1: Determine transport type

Ask clarifying questions if needed:

- **stdio transport** for:
  - Local tools consumed by CLI agents (e.g., GitHub Copilot CLI, Claude Desktop)
  - Single-user, single-session scenarios
  - Tools that need to read/write user's file system or local resources
  
- **HTTP/SSE transport** for:
  - Shared services used by multiple clients simultaneously
  - Web-based agents or remote access scenarios
  - Services deployed to cloud or containers
  - Scenarios requiring authentication or CORS

- **Both (dual hosting)** when you need stdio for local dev/testing and HTTP for production deployment

### Step 2: Select target framework

Guide the user based on their requirements:

- **.NET 8 LTS** (recommended for most cases):
  - Long-term support through November 2026
  - Stable and widely deployed
  - Full compatibility with `ModelContextProtocol` SDK
  - Best choice for production services with predictable support lifecycle

- **.NET 9 STS**:
  - Standard support through May 2026
  - Latest stable features (file-based apps, improved JSON source generation)
  - Consider if you need specific .NET 9 features
  - Shorter support window than LTS

- **.NET 10 (if available)**:
  - Next LTS release (expected November 2025)
  - Consider if project timeline aligns with .NET 10 release
  - Use preview versions only for experimentation

**Decision tree**: Use .NET 8 LTS unless the user has a specific reason to use .NET 9 features or is targeting .NET 10 for long-term projects starting after its release.

### Step 3: Scaffold the project

#### For stdio transport:

```bash
dotnet new console -n MyMcpServer
cd MyMcpServer
dotnet add package ModelContextProtocol
dotnet add package Microsoft.Extensions.Hosting
dotnet add package Microsoft.Extensions.Logging.Console
```

Create a minimal hosting setup in `Program.cs`:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using ModelContextProtocol;

var builder = Host.CreateApplicationBuilder(args);

builder.Logging.AddConsole();

builder.Services.AddMcpServer(options =>
{
    options.ServerInfo = new ServerInfo("my-mcp-server", "1.0.0");
    options.Capabilities = new ServerCapabilities
    {
        Tools = new ToolCapabilities()
    };
});

// Register your tools here
builder.Services.AddMcpTool<MyTool>();

var app = builder.Build();
await app.RunAsync();
```

#### For HTTP transport:

```bash
dotnet new web -n MyMcpServer
cd MyMcpServer
dotnet add package ModelContextProtocol
dotnet add package ModelContextProtocol.AspNetCore
```

Create `Program.cs` with ASP.NET Core hosting:

```csharp
using ModelContextProtocol;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMcpServer(options =>
{
    options.ServerInfo = new ServerInfo("my-mcp-server", "1.0.0");
    options.Capabilities = new ServerCapabilities
    {
        Tools = new ToolCapabilities()
    };
});

// Register your tools
builder.Services.AddMcpTool<MyTool>();

var app = builder.Build();

// Map MCP endpoint
app.MapMcp("/mcp");

// Optional: Add CORS if needed
// app.UseCors(policy => policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());

// Optional: Add authentication
// app.UseAuthentication();
// app.UseAuthorization();

await app.RunAsync();
```

### Step 4: Define tools and resources

#### Creating a tool:

Tools are defined using classes with attributes:

```csharp
using ModelContextProtocol;
using System.ComponentModel;

public class CalculatorTool : IMcpTool
{
    [McpToolMethod("add")]
    [Description("Adds two numbers together")]
    public Task<int> AddAsync(
        [Description("First number")] int a,
        [Description("Second number")] int b)
    {
        return Task.FromResult(a + b);
    }

    [McpToolMethod("multiply")]
    [Description("Multiplies two numbers")]
    public Task<int> MultiplyAsync(
        [Description("First number")] int a,
        [Description("Second number")] int b)
    {
        return Task.FromResult(a * b);
    }
}
```

Key patterns:
- Implement `IMcpTool` interface
- Use `[McpToolMethod("tool-name")]` attribute to define tool names
- Use `[Description]` attributes for documentation (helps agents understand when to use the tool)
- Return `Task` or `Task<T>` for async operations
- Parameters are automatically serialized/deserialized from JSON

#### Creating a resource:

Resources provide data that agents can read:

```csharp
using ModelContextProtocol;
using System.ComponentModel;

public class ConfigResource : IMcpResource
{
    [McpResourceMethod("config://app/settings")]
    [Description("Application configuration settings")]
    public Task<ResourceContents> GetSettingsAsync()
    {
        return Task.FromResult(new ResourceContents
        {
            Uri = "config://app/settings",
            MimeType = "application/json",
            Text = "{\"timeout\": 30, \"retries\": 3}"
        });
    }

    [McpResourceMethod("config://app/version")]
    [Description("Application version information")]
    public Task<ResourceContents> GetVersionAsync()
    {
        return Task.FromResult(new ResourceContents
        {
            Uri = "config://app/version",
            MimeType = "text/plain",
            Text = "1.0.0"
        });
    }
}
```

Key patterns:
- Implement `IMcpResource` interface
- Use `[McpResourceMethod("uri")]` attribute with a URI scheme
- Return `ResourceContents` with URI, MIME type, and content
- Use descriptive URI schemes (e.g., `config://`, `data://`, `file://`)

### Step 5: Add HTTP-specific concerns (if using HTTP transport)

#### Authentication:

```csharp
// In Program.cs, before app.MapMcp():
app.UseAuthentication();
app.UseAuthorization();

// Require authentication on the MCP endpoint
app.MapMcp("/mcp").RequireAuthorization();
```

Configure your preferred auth scheme (JWT, API keys, etc.) in the service registration.

#### CORS:

```csharp
// In Program.cs, add CORS services
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("https://your-client-domain.com")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// In the app configuration
app.UseCors();
```

#### Deployment considerations:

- Configure proper logging for production (`appsettings.json`)
- Set up health checks if deploying to orchestrated environments
- Use `appsettings.Production.json` for production-specific config
- Consider using reverse proxy (nginx, Azure App Service) for SSL/TLS termination

### Step 6: Test your server

#### Manual testing (stdio):

1. Build and run:
   ```bash
   dotnet run
   ```

2. Send an `initialize` request via stdin:
   ```json
   {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test-client","version":"1.0.0"}}}
   ```

3. Verify the server responds with its capabilities

4. Test a tool call:
   ```json
   {"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"add","arguments":{"a":5,"b":3}}}
   ```

#### Manual testing (HTTP):

1. Run the server:
   ```bash
   dotnet run
   ```

2. Use curl or a REST client to test the endpoint:
   ```bash
   curl -X POST http://localhost:5000/mcp \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test-client","version":"1.0.0"}}}'
   ```

#### Unit testing tools:

Create a test project and test tools in isolation:

```csharp
using Xunit;

public class CalculatorToolTests
{
    [Fact]
    public async Task AddAsync_ShouldReturnSum()
    {
        var tool = new CalculatorTool();
        var result = await tool.AddAsync(5, 3);
        Assert.Equal(8, result);
    }

    [Fact]
    public async Task MultiplyAsync_ShouldReturnProduct()
    {
        var tool = new CalculatorTool();
        var result = await tool.MultiplyAsync(4, 7);
        Assert.Equal(28, result);
    }
}
```

#### Integration testing:

For integration tests, instantiate the MCP server host and test message handling:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Xunit;

public class McpServerIntegrationTests
{
    [Fact]
    public async Task Server_ShouldHandleInitialize()
    {
        var builder = Host.CreateApplicationBuilder();
        builder.Services.AddMcpServer(options =>
        {
            options.ServerInfo = new ServerInfo("test-server", "1.0.0");
        });
        
        var host = builder.Build();
        
        // Test that the host starts successfully
        await host.StartAsync();
        await host.StopAsync();
    }
}
```

### Step 7: Configure for client consumption

#### For stdio (Copilot CLI, Claude Desktop):

Create a configuration file that points to your server binary:

For Claude Desktop (`claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "my-mcp-server": {
      "command": "dotnet",
      "args": ["run", "--project", "/path/to/MyMcpServer"],
      "env": {}
    }
  }
}
```

Or for a published executable:
```json
{
  "mcpServers": {
    "my-mcp-server": {
      "command": "/path/to/MyMcpServer",
      "args": [],
      "env": {}
    }
  }
}
```

#### For HTTP:

Provide the HTTP endpoint URL to clients:
- Development: `http://localhost:5000/mcp`
- Production: `https://your-domain.com/mcp`

Document any authentication requirements (API keys, OAuth tokens, etc.)

## Validation

- [ ] `dotnet --version` shows .NET 8, 9, or 10
- [ ] Project builds without errors: `dotnet build`
- [ ] Server starts successfully: `dotnet run`
- [ ] Server responds to `initialize` request with server info and capabilities
- [ ] Tools are discoverable via `tools/list` request
- [ ] Tool calls execute successfully and return expected results
- [ ] Resources (if any) are accessible via `resources/list` and `resources/read`
- [ ] For HTTP: endpoint is reachable and CORS/auth work as configured
- [ ] Unit tests pass: `dotnet test`

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Mixing stdio and HTTP hosting patterns | Choose one transport per project; use separate projects for dual hosting or detect at runtime |
| Missing `[Description]` attributes | Add descriptions to tools and parameters so agents know when and how to use them |
| Forgetting async operations | All tool methods should return `Task` or `Task<T>` |
| Not handling tool errors | Wrap logic in try-catch and return appropriate error messages via MCP error responses |
| Using reflection-heavy JSON under AOT | If targeting AOT, use source-generated JSON serialization contexts |
| Hardcoded file paths in stdio tools | Use relative paths or accept paths as parameters to be cross-platform compatible |
| No CORS configuration for HTTP | Add CORS policy to allow client origins |
| Not testing with actual MCP clients | Always test with a real client (Copilot CLI, Claude Desktop) before considering done |
| Tool names with spaces or special chars | Use lowercase, hyphenated names like `calculate-sum` not `Calculate Sum` |
| Large responses blocking stdio | For large data, consider pagination or streaming patterns |

## Additional Resources

- [Model Context Protocol Specification](https://modelcontextprotocol.io/specification)
- [.NET MCP SDK Documentation](https://github.com/microsoft/model-context-protocol-dotnet)
- [MCP Server Registry](https://github.com/modelcontextprotocol/servers)
- [.NET Generic Host Documentation](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host)

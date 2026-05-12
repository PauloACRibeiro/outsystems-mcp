---
name: externallibrary
description: Describes how to create, build, test, and deploy C# libraries for the OutSystems Developer Cloud (ODC) External Logic feature.
---

# OutSystems ODC External Library – Claude Code Skills

This file describes how to create, build, test, and deploy C# libraries for the OutSystems Developer Cloud (ODC) External Logic feature.

---

## Project Overview

OutSystems ODC External Logic allows extending ODC apps with custom C# code. The C# class library project is built locally, published as a ZIP file to the ODC platform. OutSystems generates a wrapper around the code that exposes the decorated interfaces as server actions consumable by ODC applications.

**SDK Reference:** https://success.outsystems.com/documentation/outsystems_developer_cloud/building_apps/extend_your_apps_with_custom_code/external_libraries_sdk_readme/

**Template Examples:** https://github.com/OutSystems/OutSystems.ExternalLibraries.SDK-templates/tree/main/templates

**Deployment API:** https://success.outsystems.com/documentation/outsystems_developer_cloud/odc_rest_apis/api_references/external_library_generation_service_api/

---

## Runtime Constraints

These are hard limits enforced by the ODC platform. Never design functions that approach or exceed them:

| Constraint | Limit |
|---|---|
| Max execution duration | **95 seconds** per function call |
| Max memory | **1,024 MB** |
| Max ephemeral storage | **512 MB** |
| Max payload (inputs + outputs combined) | **5.5 MB** |

For large data: stream in chunks, use references/URLs instead of raw bytes.

---

## Project Setup

### Target Framework
- Use **.NET 8** (current ODC recommendation). Do **not** use .NET 6 for new projects.
- Project type: **Class Library** (`<OutputType>Library</OutputType>`)

### Required NuGet Package
```xml
<PackageReference Include="OutSystems.ExternalLibraries.SDK" Version="1.5.0" />
```
Always use the latest stable version. Check: https://www.nuget.org/packages/OutSystems.ExternalLibraries.SDK

### Minimal `.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="OutSystems.ExternalLibraries.SDK" Version="1.5.0" />
  </ItemGroup>
</Project>
```

---

## SDK Attributes – Core Concepts

### `[OSInterface]` – Exposes the interface to ODC
Applied to a **public interface**. Every method on this interface becomes a server action in ODC.

```csharp
using OutSystems.ExternalLibraries.SDK;

namespace MyCompany.MyLibrary
{
    [OSInterface(
        Name = "MyLibrary",
        Description = "Short description of what this library does.",
        IconResourceName = "MyCompany.MyLibrary.Resources.icon.png"
    )]
    public interface IMyLibrary
    {
        string SayHello(string name);
    }
}
```

- `Name`: The library name visible in ODC Portal. Required for ODC Forge submission.
- `Description`: Shown in ODC Portal. Required for Forge submission.
- `IconResourceName`: Embedded resource path. Required for Forge submission.

### `[OSAction]` – Customise action metadata (optional)
Applied to individual interface methods to override defaults.

```csharp
[OSAction(
    Description = "Validates an IBAN and returns structured result.",
    ReturnName = "ValidationResult",
    ReturnDescription = "The validation outcome."
)]
ValidationResult ValidateIban(string iban);
```

### `[OSParameter]` – Customise parameter metadata (optional)
```csharp
void ProcessData(
    [OSParameter(Description = "Raw input bytes", DataType = OSDataType.BinaryData)]
    byte[] data,
    [OSParameter(Description = "Output result")]
    out string result
);
```

### `[OSStructure]` – Exposes a struct as an ODC Structure
Applied to a **public struct** (not class). Used for complex input/output types.

```csharp
[OSStructure(Description = "Represents a parsed IBAN value.")]
public struct IbanResult
{
    [OSStructureField(Description = "Two-letter country code", IsMandatory = true)]
    public string Country { get; set; }

    [OSStructureField(Description = "Basic Bank Account Number")]
    public string Bban { get; set; }

    [OSStructureField(Description = "Whether the IBAN is valid")]
    public bool IsValid { get; set; }
}
```

### `[OSIgnore]` – Hides a member from ODC
Apply to methods or properties that should not be exposed as actions or structure fields.

---

## Logging
To log your custom code and generate detailed logs including informational messages and errors, you must use the Microsoft.Extensions.Logging ILogger interface in your C# code.

```csharp
using Microsoft.Extensions.Logging;
using System.Diagnostics;

namespace MyCompany
{
    public class MyLibrary : IMyLibrary
    {
        private readonly ILogger _logger;

        public MyLibrary(ILogger logger)
        {
            _logger = logger;
        }

        public string SayHello(string name, string title = "Mr./Ms.")
        {
            using var activity = Activity.Current?.Source.StartActivity("MyLibrary.SayHello");
            _logger.LogInformation($"Saying hello to {name} with title {title}");
            return $"Hello, {title} {name}";
        }

        public string SayGoodbye(string name)
        {
            using var activity = Activity.Current?.Source.StartActivity("MyLibrary.SayGoodbye");
            _logger.LogInformation($"Saying goodbye to {name}");
            return $"Goodbye, {name}";
        }
    }
}
```

---

## Supported Data Types

Only these types are valid as method parameters and return types:

| C# Type | ODC Type |
|---|---|
| `string` | Text |
| `int` | Integer |
| `long` | Long Integer |
| `bool` | Boolean |
| `decimal` | Decimal |
| `float` | Decimal |
| `double` | Decimal |
| `DateTime` | DateTime |
| `byte[]` | Binary Data |
| struct with `[OSStructure]` | Structure |
| `IEnumerable<T>` of any above | List |

**Not supported:** classes, dictionaries, nullable value types, enums (wrap in int/string), generics other than IEnumerable.

---

## Recommended Project Structure

```
MyLibrary/
├── MyLibrary.csproj
├── Interfaces/
│   └── IMyLibrary.cs          # [OSInterface]-decorated interface
├── Implementations/
│   └── MyLibrary.cs           # Class implementing the interface
├── Structures/
│   └── MyResult.cs            # [OSStructure]-decorated structs
├── Resources/
│   └── icon.png               # Embedded icon (set as EmbeddedResource)
└── Tests/
    ├── MyLibrary.UnitTests/
    │   └── MyLibraryTests.cs
    └── MyLibrary.IntegrationTests/
        └── ContainerTests.cs
```

---

## Implementation Pattern

The interface and the implementing class must be in the same assembly. ODC finds the implementation via the interface.

```csharp
// Interfaces/IMyLibrary.cs
using OutSystems.ExternalLibraries.SDK;

namespace MyCompany.MyLibrary
{
    [OSInterface(Name = "MyLibrary", Description = "Example library")]
    public interface IMyLibrary
    {
        [OSAction(Description = "Greets a person by name")]
        string Greet(
            [OSParameter(Description = "Name to greet")] string name
        );
    }
}

// Implementations/MyLibrary.cs
using Microsoft.Extensions.Logging;

namespace MyCompany.MyLibrary
{
    public class MyLibrary : IMyLibrary
    {
        private readonly ILogger _logger;

        public MyLibrary(ILogger logger)
        {
            _logger = logger;
        }

        public string Greet(string name) => $"Hello, {name}!";
    }
}
```

### Output Parameters
Use `out` parameters for multiple return values:

```csharp
// In interface:
void Parse(string input, out bool success, out string errorMessage, out MyResult result);

// In implementation:
public void Parse(string input, out bool success, out string errorMessage, out MyResult result)
{
    // ...
}
```

---

## Testing

### Unit Tests
Use standard .NET test frameworks. The library code has no ODC runtime dependency, so it can be tested directly.

```xml
<!-- MyLibrary.UnitTests.csproj -->
<PackageReference Include="xunit" Version="2.*" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
<PackageReference Include="Moq" Version="4.*" />
```

```csharp
public class MyLibraryTests
{
    private readonly IMyLibrary _sut = new MyLibrary();

    [Fact]
    public void Greet_ReturnsExpectedString()
    {
        var result = _sut.Greet("World");
        Assert.Equal("Hello, World!", result);
    }
}
```

Run: `dotnet test`

### Integration Tests (Container)
Integration tests should run the code in a container matching the ODC runtime environment.

**Base image:** `edencore.azurecr.io/dotnet_aspnet`

Example `Dockerfile` for integration tests:
```dockerfile
FROM edencore.azurecr.io/dotnet_aspnet AS base

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish MyLibrary.IntegrationTests/MyLibrary.IntegrationTests.csproj -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "vstest", "MyLibrary.IntegrationTests.dll"]
```

Run integration tests:
```bash
docker build -t mylibrary-integration-tests .
docker run --rm mylibrary-integration-tests
```

**What to validate in integration tests:**
- All public actions execute without exceptions in the target runtime
- Memory usage stays within 1,024 MB for expected payloads
- Execution completes within 95 seconds
- Payload sizes (serialised inputs + outputs) stay under 5.5 MB

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| Interface not found | Implementing class not in same assembly | Ensure interface and class are in the same `.csproj` |
| Unsupported type | Using `class` instead of `struct` for complex types | Use `struct` with `[OSStructure]` |
| ZIP structure wrong | Folder nested inside ZIP | Zip the *contents* of `/publish`, not the folder itself |
| Payload too large | Input/output exceeds 5.5 MB | Chunk data, use URLs, or split into multiple actions |
| Timeout | Single action takes more than 95 seconds | Break up long-running work; consider async chunked patterns |
| Missing icon | Forge submission fails | Embed PNG as `EmbeddedResource` and set `IconResourceName` |

---

## SDK Version Notes

- **SDK 1.5.0** is the current latest version as of early 2025.
- **ODC now recommends .NET 8** – if the ODC Portal warns about .NET 6, upgrade the `<TargetFramework>`.
- Check https://www.nuget.org/packages/OutSystems.ExternalLibraries.SDK for the latest version before starting a new project.

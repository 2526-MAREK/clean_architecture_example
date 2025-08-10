Quick scan to ground the guide in your codebase’s existing EmailService pattern. Fetching the interface, implementation, DI registration, models, and any usages/config bindings.
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool

### Goal
You want a practical, end‑to‑end guide for adding a new cross‑cutting “service” to your Clean Architecture app (like your existing `EmailService` in Infrastructure) — from idea to implementation, wired through configuration and DI, and used in Application handlers.

Below is a concise, repeatable blueprint tailored to your repo’s structure and current `EmailService` pattern.

### When to put it in Infrastructure vs Application vs Domain
- **Domain service**: pure business logic, no frameworks or I/O.
- **Application service (interface only)**: orchestrates use cases; defines contracts needed by use cases.
- **Infrastructure service (implementation)**: uses external systems/frameworks (SMTP, SendGrid, HTTP APIs, file system, storage). This is what you want.

---

### 1) Define the contract in Application
- Location: `GloboTicket.TicketManagement.Application/Contracts/Infrastructure`
- Purpose: decouple your use cases from any vendor/client libraries.

Example (new SMS service):
```csharp
// Application/Contracts/Infrastructure/ISmsService.cs
namespace GloboTicket.TicketManagement.Application.Contracts.Infrastructure
{
    public interface ISmsService
    {
        Task<bool> SendAsync(string phoneNumber, string message, CancellationToken cancellationToken = default);
    }
}
```

If your service needs settings or models, keep simple POCOs in Application, like you already do for email:
- `Application/Models/Mail/EmailSettings.cs`
- `Application/Models/Mail/Email.cs`

For a new service, mirror that:
```csharp
// Application/Models/Sms/SmsSettings.cs
namespace GloboTicket.TicketManagement.Application.Models.Sms
{
    public class SmsSettings
    {
        public string ApiKey { get; set; }
        public string SenderId { get; set; }
        public string BaseUrl { get; set; }
    }
}
```

### 2) Implement the service in Infrastructure
- Location: `GloboTicket.TicketManagement.Infrastructure/<ServiceArea>/`
- Follow the pattern of `Infrastructure/Mail/EmailService.cs`.

Example:
```csharp
// Infrastructure/Sms/SmsService.cs
using System.Net.Http;
using System.Net.Http.Json;
using Microsoft.Extensions.Options;
using GloboTicket.TicketManagement.Application.Contracts.Infrastructure;
using GloboTicket.TicketManagement.Application.Models.Sms;

namespace GloboTicket.TicketManagement.Infrastructure.Sms
{
    public class SmsService : ISmsService
    {
        private readonly HttpClient httpClient;
        private readonly SmsSettings settings;

        public SmsService(HttpClient httpClient, IOptions<SmsSettings> settings)
        {
            this.httpClient = httpClient;
            this.settings = settings.Value;
        }

        public async Task<bool> SendAsync(string phoneNumber, string message, CancellationToken cancellationToken = default)
        {
            var payload = new { to = phoneNumber, sender = settings.SenderId, text = message };
            var response = await httpClient.PostAsJsonAsync("/messages", payload, cancellationToken);
            return response.IsSuccessStatusCode;
        }
    }
}
```

Notes:
- Prefer `IHttpClientFactory`/typed clients for HTTP. Avoid `new HttpClient()` directly.
- Keep vendor SDKs here, never in Application.

### 3) Register DI and bind configuration
- Location: `GloboTicket.TicketManagement.Infrastructure/InfrastructureServiceRegistration.cs`
- Pattern already present for email:
  - `services.Configure<EmailSettings>(configuration.GetSection("EmailSettings"));`
  - `services.AddTransient<IEmailService, EmailService>();`

Add your new service:
```csharp
// Infrastructure/InfrastructureServiceRegistration.cs
using GloboTicket.TicketManagement.Application.Contracts.Infrastructure;
using GloboTicket.TicketManagement.Application.Models.Sms;
using GloboTicket.TicketManagement.Infrastructure.Sms;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using System;

public static class InfrastructureServiceRegistration
{
    public static IServiceCollection AddInfrastructureServices(this IServiceCollection services, IConfiguration configuration)
    {
        // Existing
        services.Configure<EmailSettings>(configuration.GetSection("EmailSettings"));
        services.AddTransient<IEmailService, EmailService>();

        // New: bind settings
        services.Configure<SmsSettings>(configuration.GetSection("SmsSettings"));

        // New: typed HttpClient for SmsService
        services.AddHttpClient<ISmsService, SmsService>(client =>
        {
            var baseUrl = configuration["SmsSettings:BaseUrl"];
            client.BaseAddress = new Uri(baseUrl);
        });

        return services;
    }
}
```

Ensure your API startup wires Infrastructure:
- In `Program.cs` (API), you should already have something like:
  - `builder.Services.AddInfrastructureServices(builder.Configuration);`
  - `builder.Services.AddApplicationServices();`
  - `builder.Services.AddPersistenceServices(builder.Configuration);`

### 4) Add configuration values in API
- Location: `GloboTicket.TicketManagement.Api/appsettings.json`
- Follow your existing `EmailSettings` section:
```json
"SmsSettings": {
  "ApiKey": "your-key-here",
  "SenderId": "GloboTicket",
  "BaseUrl": "https://api.your-sms.com"
}
```
Use `appsettings.Development.json` for local overrides and a secret store/Key Vault for prod keys.

### 5) Use the service in Application handlers
- Inject only the interface from Application.Contracts.
- Example (send SMS after creating an order; similar to sending email after a review):

```csharp
public class CreateReviewCommandHandler : IRequestHandler<CreateReviewCommand, CreateReviewCommandResponse>
{
    private readonly IReviewRepository reviewRepository;
    private readonly IEmailService emailService; // already in your codebase
    private readonly ISmsService smsService;     // new

    public CreateReviewCommandHandler(
        IReviewRepository reviewRepository,
        IEmailService emailService,
        ISmsService smsService)
    {
        this.reviewRepository = reviewRepository;
        this.emailService = emailService;
        this.smsService = smsService;
    }

    public async Task<CreateReviewCommandResponse> Handle(CreateReviewCommand request, CancellationToken cancellationToken)
    {
        // ... save review ...

        // optional cross-cutting work
        await emailService.SendEmail(new Email { To = "...", Subject = "...", Body = "..." });
        await smsService.SendAsync("+123456789", "Thanks for your review!", cancellationToken);

        return new CreateReviewCommandResponse { /* ... */ };
    }
}
```

Keep this logic orchestration in Application; do not reference vendor packages here.

### 6) Add NuGet packages to Infrastructure only
- For HTTP/Polly: `Microsoft.Extensions.Http`
- For providers (e.g., SendGrid, Twilio SDKs), add them to `Infrastructure` project.
- Do not add provider packages to Application.

### 7) Testing strategy
- **Application unit tests**: mock `ISmsService`/`IEmailService` to verify interactions, not vendor behavior.
- **Infrastructure unit tests**: small tests around mapping/payload construction.
- **Integration tests**: test real HTTP against a sandbox or a local mock server.

### 8) Resilience, observability, and background work
- Add retry/circuit breaker via Polly on the typed client registration:
```csharp
services.AddHttpClient<ISmsService, SmsService>(client => { /* ... */ })
    .AddTransientHttpErrorPolicy(p => p.RetryAsync(3));
```
- Log inside Infrastructure; surface errors as domain-meaningful results to Application.
- For fire‑and‑forget side effects, consider MediatR notifications or a background queue.

### 9) Security and secrets
- Never store keys in source. Keep them in user secrets, environment variables, or Key Vault.
- The `EmailSettings` pattern you already use with `IOptions<T>` is correct.

### 10) Quick checklist (apply this for any new Infrastructure service)
- Define `IYourService` in `Application/Contracts/Infrastructure`.
- Add settings POCO in `Application/Models/<Area>/<Settings>.cs` if needed.
- Implement in `Infrastructure/<Area>/<YourService>.cs`.
- Bind config and register DI in `InfrastructureServiceRegistration`.
- Add config section in API `appsettings.json`.
- Ensure API `Program.cs` calls `AddInfrastructureServices`.
- Inject and use the interface in Application (handlers).
- Add tests: unit (Application with mocks), integration (Infrastructure).

---

### Use your existing EmailService as a reference
- Contract: `Application/Contracts/Infrastructure/IEmailService.cs`
- Settings: `Application/Models/Mail/EmailSettings.cs`
- Implementation: `Infrastructure/Mail/EmailService.cs`
- DI and config binding: `Infrastructure/InfrastructureServiceRegistration.cs`
- Config values: `Api/appsettings.json` under `EmailSettings`

This is the exact pattern you should replicate for any new cross‑cutting Infrastructure service.

Status: I inspected your current Email service contract, implementation, DI registration, and appsettings to ground this guide in your codebase’s patterns. No code edits were made.

- Provided a 10-step, repo-specific blueprint to add new Infrastructure services.
- Included minimal code snippets for interface, settings, implementation, DI registration, `appsettings`, and handler usage.
- Covered testing, resilience, secrets, and where each piece belongs.
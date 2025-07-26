

## Data Flow in Clean Architecture - GloboTicket Application

### **Architecture Overview**

This application follows Clean Architecture principles with four distinct layers:

1. **API Layer** (Presentation)
2. **Application Layer** (Business Logic)
3. **Domain Layer** (Core Business Entities)
4. **Infrastructure/Persistence Layer** (External Concerns)

### **Data Flow Diagram**

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Layer     │    │  Application     │    │  Domain Layer   │    │ Infrastructure  │
│   (Controllers) │───▶│  Layer (CQRS)    │───▶│  (Entities)     │───▶│  (Database)     │
└─────────────────┘    └──────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │                       │
         ▼                       ▼                       ▼                       ▼
   HTTP Request/Response    Commands/Queries         Domain Models         SQL Server
```

### **Detailed Data Flow**

#### **1. Request Flow (Create Event Example)**

```
HTTP POST /api/events
    ↓
EventsController.CreateEvent()
    ↓
MediatR.Send(CreateEventCommand)
    ↓
CreateEventCommandHandler.Handle()
    ↓
Validation (CreateEventCommandValidator)
    ↓
AutoMapper.Map<Event>(command)
    ↓
EventRepository.AddAsync(event)
    ↓
DbContext.SaveChanges()
    ↓
EmailService.SendEmail() (Infrastructure)
    ↓
Return EventId
```

#### **2. Query Flow (Get Events List Example)**

```
HTTP GET /api/events
    ↓
EventsController.GetAllEvents()
    ↓
MediatR.Send(GetEventsListQuery)
    ↓
GetEventsListQueryHandler.Handle()
    ↓
EventRepository.ListAllAsync()
    ↓
DbContext.Events.ToListAsync()
    ↓
AutoMapper.Map<List<EventListVm>>(events)
    ↓
Return List<EventListVm>
```

### **Layer Responsibilities**

#### **API Layer (`GloboTicket.TicketManagement.Api`)**
- **Controllers**: Handle HTTP requests/responses
- **Dependency**: Uses MediatR to send Commands/Queries
- **No Business Logic**: Only request/response handling

#### **Application Layer (`GloboTicket.TicketManagement.Application`)**
- **Commands/Queries**: CQRS pattern implementation
- **Handlers**: Business logic orchestration
- **Validators**: Input validation
- **DTOs/VMs**: Data transfer objects and view models
- **Contracts**: Interfaces for external dependencies

#### **Domain Layer (`GloboTicket.TicketManagement.Domain`)**
- **Entities**: Core business objects (Event, Category, Order)
- **Value Objects**: Immutable business concepts
- **Domain Logic**: Business rules and invariants

#### **Infrastructure Layer (`GloboTicket.TicketManagement.Infrastructure`)**
- **External Services**: Email, CSV Export, File operations
- **Implementation**: Concrete implementations of application contracts

#### **Persistence Layer (`GloboTicket.TicketManagement.Persistence`)**
- **DbContext**: Entity Framework configuration
- **Repositories**: Data access implementations
- **Migrations**: Database schema management

### **Key Clean Architecture Principles Demonstrated**

1. **Dependency Inversion**: Application layer defines contracts, Infrastructure implements them
2. **Separation of Concerns**: Each layer has a single responsibility
3. **CQRS Pattern**: Separate Commands (write) and Queries (read)
4. **MediatR Pattern**: Decoupled communication between layers
5. **Repository Pattern**: Abstract data access
6. **AutoMapper**: Clean separation between domain and DTOs

### **Dependency Flow**

```
API → Application → Domain
  ↓
Infrastructure → Application Contracts
  ↓
Persistence → Application Contracts
```

The arrows show that dependencies point inward toward the Domain layer, following the Dependency Inversion Principle.

### **Service Registration Flow**

```csharp
// API Layer registers all services
builder.Services.AddApplicationServices();      // MediatR, AutoMapper
builder.Services.AddInfrastructureServices();   // Email, CSV Export
builder.Services.AddPersistenceServices();     // EF Core, Repositories
```

This architecture ensures that:
- Business logic is isolated in the Application layer
- Domain entities remain pure and framework-agnostic
- External concerns (database, email) are abstracted through interfaces
- Testing is easier due to clear separation and dependency injection
- Changes in one layer don't affect others (loose coupling)

---

## 8. Backend Architecture (Clean Architecture + CQRS)

### 8.1 Cấu trúc chung cho Web API Services

```
ServiceName/
├── ServiceName.Domain/
│   ├── Entities/
│   ├── Enums/
│   └── Events/            # Domain events
│
├── ServiceName.Application/
│   ├── Commands/          # Write operations
│   ├── Queries/           # Read operations
│   ├── Handlers/          # MediatR handlers
│   ├── DTOs/
│   └── Interfaces/        # IRepository, ISqsPublisher, ...
│
├── ServiceName.Infrastructure/
│   ├── Persistence/       # DbContext, Repository implementations
│   ├── Messaging/         # SQS publisher implementation
│   └── ExternalServices/  # HTTP clients gọi service khác
│
└── ServiceName.API/
    ├── Controllers/
    └── Middleware/        # Auth, error handling, logging
```

### 8.2 Cấu trúc cho Worker Services

Worker không có HTTP layer nên chỉ cần 3 layer:

```
WorkerName/
├── WorkerName.Application/
│   ├── Handlers/          # Xử lý message từ SQS
│   ├── DTOs/
│   └── Interfaces/
│
├── WorkerName.Infrastructure/
│   ├── Messaging/         # SQS consumer implementation
│   └── ExternalServices/  # AWS SES, HTTP clients, ...
│
└── WorkerName.Worker/
    ├── Workers/           # BackgroundService implementations
    └── Program.cs         # DI setup, host configuration
```

### 8.3 CQRS Flow

```
HTTP Request
    ↓
Controller → Send(Command / Query) via MediatR
                ↓
          Handler (Application layer)
                ↓
          IRepository / IExternalService (interfaces)
                ↓
          Infrastructure implements
                ↓
          Database / AWS SQS / External API
```

### 8.4 Dependency Rule

```
API → Application → Domain
Infrastructure → Application → Domain

Domain không phụ thuộc vào bất kỳ layer nào
```


---

# 🧩 Use-Case Oriented Architecture (DDD-inspired) for Laravel

This repository demonstrates a **Use-Case Oriented Architecture** for Laravel applications.  
The approach is inspired by **DDD**, **Clean Architecture**, and **Hexagonal principles**, but adapted to be **practical, scalable, and Laravel-friendly**.

The main idea is simple:

> **Business logic lives in explicit use-cases, not in controllers or Eloquent models.**

---

## 🎯 Goals of the Architecture

- Keep controllers thin
    
- Make business logic explicit and testable
    
- Separate read/write responsibilities
    
- Avoid “fat models” and hidden side effects
    
- Scale complexity without turning code into spaghetti
    
- Stay pragmatic — no “pure DDD for the sake of DDD”
    

---

## 🏗 High-Level Structure

Each business module is fully isolated and structured as follows:

```
Chat
├─ Application
├─ Domain
├─ Infrastructure
├─ Database
├─ Tests
```

Each layer has **strict responsibility boundaries**.

---

## 📦 Application Layer

> **Purpose:** Orchestrates use-cases and application workflows.

This layer contains **what the system does**, not **how data is stored**.

```
Application
├─ Actions
│  ├─ Chat
│  │  ├─ CreateChatAction.php
│  │  ├─ UpdateChatAction.php
│  │  ├─ DeleteChatAction.php
│  │  └─ ShowChatAction.php
│  ├─ ChatMessage
│  │  ├─ SendMessageAction.php
│  │  └─ ReadMessageAction.php
│  └─ ChatParticipant
│     ├─ AddParticipantAction.php
│     └─ RemoveParticipantAction.php
│
├─ Flow
│  ├─ Chat
│  │  ├─ CreateChatFlow.php
│  │  ├─ UpdateChatFlow.php
│  │  └─ DeleteChatFlow.php
│  ├─ ChatMessage
│  │  └─ SendMessageFlow.php
│  └─ ChatParticipant
│     ├─ AddParticipantFlow.php
│     └─ ResolveParticipantsFlow.php
│
├─ Jobs
│  ├─ ChatCreatedJob.php
│  ├─ ChatDeletedJob.php
│  └─ MessageSentJob.php
│
└─ Policies
   ├─ ChatPolicy.php
   ├─ ChatMessagePolicy.php
   └─ ChatParticipantPolicy.php
```

### 🔹 Actions

**Actions** are entry points for use-cases.  
They are usually called from Controllers.

Responsibilities:

- Receive DTOs
    
- Call Policies
    
- Trigger Flow logic
    
- Coordinate Services, Repositories, Jobs
    
- Manage transactions
    

Example:

```php
class CreateChatAction {
    public function __invoke(ChatFormDTO $dto) {
        return $this->flow->create($dto);
    }
}
```

---

### 🔹 Flow

**Flow** represents **business workflows** that are too complex for a single method.

Typical responsibilities:

- Conditional business logic
    
- Derived data calculation
    
- Participant resolution
    
- Chat type detection
    
- Orchestration across multiple services
    

Example responsibilities inside `CreateChatFlow`:

- Determine chat type (support / group / private / market)
    
- Resolve participants list
    
- Build chat title
    
- Apply business constraints
    
- Execute write operations atomically
    

Flow **does not**:

- Talk directly to HTTP
    
- Render responses
    
- Contain persistence implementation details
    

---

### 🔹 Policies

Policies encapsulate **authorization and permission rules**.

Example:

```php
$this->chatPolicy->canSendMessage($dto, $chat);
```

Policies:

- Throw exceptions on access violations
    
- Contain no persistence logic
    
- Can be reused across Actions and Flows
    

---

## 🧠 Domain Layer

> **Purpose:** Describe the business language and rules.

This layer contains **what is allowed**, **what exists**, and **what is forbidden** in the business domain.

```
Domain
├─ Constants
│  ├─ ChatTypeSlugs.php
│  ├─ ChatLimits.php
│  └─ MessageTypes.php
│
├─ DTO
│  ├─ Form
│  │  ├─ ChatFormDTO.php
│  │  ├─ ChatParticipantFormDTO.php
│  │  └─ ChatMessageFormDTO.php
│  ├─ List
│  │  ├─ ChatListDTO.php
│  │  └─ ChatMessageListDTO.php
│  └─ Flow
│     └─ ChatFlowDTO.php
│
├─ Rules
│  ├─ ChatCreateRule.php
│  ├─ ChatTypeRule.php
│  └─ ParticipantRoleRule.php
│
├─ Exceptions
│  ├─ InvalidChatTypeException.php
│  ├─ ChatAccessDeniedException.php
│  └─ ParticipantLimitExceededException.php
│
├─ Filters
│  ├─ ChatVisibleForUserFilter.php
│  └─ MessageReadableFilter.php
│
├─ Enums
│  ├─ ChatTypeEnum.php
│  └─ ParticipantRoleEnum.php
│
└─ Traits
   ├─ AppliesToOrganization.php
   └─ HasParticipants.php
```

### 🔹 DTO (Data Transfer Objects)

DTOs are **the main contract between layers**.

They:

- Are immutable (or pseudo-immutable)
    
- Encapsulate validated data
    
- Prevent dependency on Request / Eloquent
    

Example:

```php
ChatFormDTO::fromRequest($request, $authUser);
```

---

### 🔹 Rules & Filters

- **Rules** — validate business invariants
    
- **Filters** — reusable query or dataset constraints
    

They contain **pure business logic**, without infrastructure concerns.

---

## 🧱 Infrastructure Layer

> **Purpose:** Implements technical details.

```
Infrastructure
├─ Models
│  ├─ Chat.php
│  ├─ ChatParticipant.php
│  ├─ ChatMessage.php
│  └─ ChatMessageRead.php
│
├─ Repositories
│  ├─ ChatRepository.php
│  ├─ ChatMessageRepository.php
│  └─ ChatParticipantRepository.php
│
├─ Services
│  ├─ ChatPersistenceService.php
│  ├─ MessagePersistenceService.php
│  └─ ParticipantPersistenceService.php
│
├─ Http
│  ├─ Controllers
│  ├─ Requests
│  └─ Middleware
│
├─ Resources
│  ├─ ChatResource.php
│  └─ ChatMessageResource.php
│
├─ Console
│  └─ Commands
│
└─ Helpers
   └─ ChatUrlHelper.php
```

### 🔹 Repositories

Repositories are **read-oriented**.

They:

- Fetch data
    
- Build complex queries
    
- Never mutate state
    

Example:

```php
$chat = $this->chatRepository->showOnceById($dto);
```

---

### 🔹 Services (Persistence)

Services handle **write operations only**.

They:

- Create, update, delete models
    
- Do not contain business rules
    
- Are usually called from Flow
    

Example:

```php
$this->chatService->create($dto);
```

---

## 🗄 Database Layer

```
Database
├─ Migrations
├─ Seeders
└─ Factories
```

This layer is fully isolated from business logic.

---

## 🧪 Tests

```
Tests
├─ Unit
└─ Feature
```

- **Unit** — Rules, Filters, Policies, Flows
    
- **Feature** — Use-cases through HTTP
    

---

## 🔄 Typical Request Flow

```
HTTP Request
   ↓
Controller
   ↓
Action
   ↓
Policy
   ↓
Flow
   ↓
Repository (read)
   ↓
Service (write)
   ↓
Job / Event
```

---

## 🧠 Is This Pure DDD?

No — and intentionally so.

This architecture:

- ❌ does not enforce Rich Domain Models
    
- ❌ does not isolate Domain from Infrastructure completely
    
- ✅ focuses on use-cases and business clarity
    
- ✅ works well with Laravel & Eloquent
    
- ✅ scales with growing complexity
    

---

## 📌 Naming

We call this approach:

> **Use-Case Oriented Architecture (DDD-inspired)**

---

## 🚀 When to Use This Architecture

✔ Large Laravel projects  
✔ Complex business logic  
✔ Multiple interaction channels (Web / API / Mobile)  
✔ Long-term maintenance

❌ Small CRUD-only apps  
❌ MVPs with simple rules

---

Example Module:
```
Chat
├─ Application
│ ├─ Actions
│ │ ├─ Chat
│ │ │ ├─ CreateChatAction.php
│ │ │ ├─ UpdateChatAction.php
│ │ │ ├─ DeleteChatAction.php
│ │ │ └─ ShowChatAction.php
│ │ ├─ ChatMessage
│ │ │ ├─ SendMessageAction.php
│ │ │ └─ ReadMessageAction.php
│ │ └─ ChatParticipant
│ │ ├─ AddParticipantAction.php
│ │ └─ RemoveParticipantAction.php
│ │
│ ├─ Flow
│ │ ├─ Chat
│ │ │ ├─ CreateChatFlow.php
│ │ │ | └─ Context
│ │ │ | ├─ ChatContextResolver.php
│ │ │ | ├─ ChatContextResolveFlow.php
│ │ │ | └─ Resolvers
│ │ │ | ├─ IssueChatResolver.php
│ │ │ | ├─ HouseChatResolver.php
│ │ │ | ├─ HouseComplexChatResolver.php
│ │ │ | └─ MarketChatResolver.php
│ │ │ ├─ UpdateChatFlow.php
│ │ │ └─ DeleteChatFlow.php
│ │ ├─ ChatMessage
│ │ │ └─ SendMessageFlow.php
│ │ └─ ChatParticipant
│ │ ├─ AddParticipantFlow.php
│ │ └─ ResolveParticipantsFlow.php
│ │
│ ├─ Crudler
│ │ ├─ Chat
│ │ │ ├─ Actions
│ │ │ │ ├─ ActionCreateChatCrudler.php
│ │ │ │ ├─ ActionUpdateChatCrudler.php
│ │ │ │ ├─ ActionDeleteChatCrudler.php
│ │ │ │ ├─ ActionRestoreChatCrudler.php
│ │ │ ├─ ActionChatCrudler.php
│ │ │ ├─ RepositoryChatCrudler.php
│ │ │ ├─ ServiceChatCrudler.php
│ │ │ ├─ ResourceChatCrudler.php
│ │ │ └─ RequestChatCrudler.php
│ │ ├─ ChatMessage
│ │ │ ├─ Actions
│ │ │ │ ├─ ActionCreateChatMessageCrudler.php
│ │ │ │ ├─ ActionUpdateChatMessageCrudler.php
│ │ │ │ ├─ ActionDeleteChatMessageCrudler.php
│ │ │ │ ├─ ActionRestoreChatMessageCrudler.php
│ │ │ ├─ ActionChatMessageCrudler.php
│ │ │ ├─ RepositoryChatMessageCrudler.php
│ │ │ ├─ ServiceChatMessageCrudler.php
│ │ │ ├─ ResourceChatMessageCrudler.php
│ │ │ └─ RequestChatMessageCrudler.php
│ │
│ ├─ Jobs
│ │ ├─ ChatCreatedJob.php
│ │ ├─ ChatDeletedJob.php
│ │ └─ MessageSentJob.php
│ │
│ └─ Policies
│ ├─ ChatPolicy.php
│ ├─ ChatMessagePolicy.php
│ └─ ChatParticipantPolicy.php
│
├─ Domain
│ ├─ Constants
│ │ ├─ ChatTypeSlugs.php
│ │ ├─ ChatLimits.php
│ │ └─ MessageTypes.php
│ │
│ ├─ DTO
│ │ ├─ List
│ │ │ ├─ ChatListDTO.php
│ │ │ └─ ChatMessageListDTO.php
│ │ ├─ Form
│ │ │ ├─ ChatFormDTO.php
│ │ │ ├─ ChatParticipantFormDTO.php
│ │ │ ├─ ChatMessageFormDTO.php
│ │ ├─ Flow
│ │ │ ├─ ChatFlowDTO.php
│ │
│ ├─ Exceptions
│ │ ├─ InvalidChatTypeException.php
│ │ ├─ ChatAccessDeniedException.php
│ │ └─ ParticipantLimitExceededException.php
│ │
│ ├─ Filters
│ │ ├─ ChatVisibleForUserFilter.php
│ │ └─ MessageReadableFilter.php
│ │
│ ├─ Rules
│ │ ├─ ChatTypeRule.php
│ │ ├─ ChatCreateRule.php
│ │ └─ ParticipantRoleRule.php
│ │
│ ├─ Traits
│ │ ├─ AppliesToOrganization.php
│ │ └─ HasParticipants.php
│ │
│ └─ Enums
│ ├─ ChatTypeEnum.php
│ └─ ParticipantRoleEnum.php
│
├─ Infrastructure
│ ├─ Models
│ │ ├─ Chat.php
│ │ ├─ ChatParticipant.php
│ │ ├─ ChatRole.php
│ │ ├─ ChatMessage.php
│ │ └─ ChatMessageRead.php
│ │
│ ├─ Repositories
│ │ ├─ ChatRepository.php
│ │ ├─ ChatMessageRepository.php
│ │ └─ ChatParticipantRepository.php
│ │
│ ├─ Services
│ │ ├─ ChatPersistenceService.php
│ │ ├─ MessagePersistenceService.php
│ │ └─ ParticipantPersistenceService.php
│ │
│ ├─ Resources
│ │ ├─ ChatResource.php
│ │ └─ ChatMessageResource.php
│ │
│ ├─ Http
│ │ ├─ Controllers
│ │ │ ├─ Private
│ │ │ │ ├─ ChatController.php
│ │ │ │ ├─ ChatMessageController.php
│ │ │ │ └─ ChatParticipantController.php
│ │ │ ├─ Public
│ │ │ │ ├─ ChatController.php
│ │ │ │ ├─ ChatMessageController.php
│ │ │ │ └─ ChatParticipantController.php
│ │ │ ├─ Mobile
│ │ │ │ ├─ ChatController.php
│ │ │ │ ├─ ChatMessageController.php
│ │ │ │ └─ ChatParticipantController.php
│ │ │ ├─ Shared
│ │ │ │ ├─ ChatController.php
│ │ │ │ ├─ ChatMessageController.php
│ │ │ │ └─ ChatParticipantController.php
│ │ ├─ Requests
│ │ │ ├─ CreateChatRequest.php
│ │ │ ├─ UpdateChatRequest.php
│ │ │ └─ SendMessageRequest.php
│ │ └─ Middleware
│ │ └─ EnsureChatAccess.php
│ │
│ ├─ Console
│ │ └─ Commands
│ │ └─ RebuildChatCountersCommand.php
│ │
│ └─ Helpers
│ └─ ChatUrlHelper.php
│
├─ Database
│ ├─ Migrations
│ │ ├─ create_chats_table.php
│ │ ├─ create_chat_participants_table.php
│ │ └─ create_chat_messages_table.php
│ │
│ ├─ Seeders
│ │ ├─ ChatSeeder.php
│ │ └─ Test
│ │ ├─ ChatTestSeeder.php
│ │ └─ ChatMessageTestSeeder.php
│ │
│ └─ Factories
│ ├─ ChatFactory.php
│ └─ ChatMessageFactory.php
├─ Tests
│ ├─ Unit
│ └─ Features
├─ Providers
│ ├─ ChatServiceProvider.php
```

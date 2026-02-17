# Job Application Tracking System (JobTrack)

## Software Design Document

### MVP — Version 1.0

---

**Document Status:** Draft  
**Language:** C# (.NET)  
**Prepared:** February 2026

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Goals and Scope](#2-goals-and-scope)
3. [Stakeholders and Users](#3-stakeholders-and-users)
4. [Functional Requirements](#4-functional-requirements)
5. [Non-Functional Requirements](#5-non-functional-requirements)
6. [Domain Model](#6-domain-model)
7. [Architecture Overview](#7-architecture-overview)
8. [Application Layers](#8-application-layers)
9. [Vertical Slice Design](#9-vertical-slice-design)
10. [Data Model and Persistence](#10-data-model-and-persistence)
11. [CLI Interface Design](#11-cli-interface-design)
12. [Business Rules and Invariants](#12-business-rules-and-invariants)
13. [Statistics Specification](#13-statistics-specification)
14. [Error Handling Strategy](#14-error-handling-strategy)
15. [Technology Stack](#15-technology-stack)
16. [Key Design Decisions and Rationale](#16-key-design-decisions-and-rationale)
17. [Glossary](#17-glossary)

---

## 1. Introduction

### 1.1 Purpose

This document describes the software design for **JobTrack**, a command-line application that allows individuals to record, manage, and analyse their job applications throughout one or more discrete job search periods. It defines the domain model, architecture, data model, and interface design for the Minimum Viable Product (MVP).

### 1.2 Background

Job seekers frequently lose track of where they have applied, the current status of each application, and the overall shape of their search. Spreadsheets are commonly used as a workaround but provide no enforced structure, no derived statistics, and no concept of distinct search periods. JobTrack addresses these gaps with a focused, opinionated tool that is simple to use from the terminal.

### 1.3 Scope of the MVP

The MVP delivers:

- Creation and management of a single active **Search Period** at a time.
- Full lifecycle tracking of **Job Applications** within a Search Period, including a complete status change audit trail.
- A rich set of **statistics** calculated per period.
- The ability to **move** an application from a closed period into an active one without altering the closed period's historical statistics.
- A **CLI interface** built with Spectre Console and Spectre Console CLI.
- Persistence via a local **Microsoft SQL Server** database managed through EF Core migrations.

The following are explicitly **out of scope** for the MVP:

- Multi-user or networked access.
- Web or desktop GUI.
- Managing more than one active Search Period simultaneously.
- Automated notifications or reminders.
- Integration with external job boards or email services.
- Authentication or authorisation.

---

## 2. Goals and Scope

### 2.1 Primary Goals

**G1 — Frictionless recording.** A user should be able to log a new application in seconds from the terminal.

**G2 — Accurate historical statistics.** Once a Search Period is closed, its statistics must be immutable and must faithfully reflect the applications that belonged to it at the time of closure.

**G3 — Clean period boundaries.** Moving an application forward into a new period must not alter any figures computed for the period from which it originated.

**G4 — Actionable insight.** Statistics must be presented in a form that immediately helps the user understand the health and progress of their job search.

**G5 — Maintainable codebase.** The design must enable feature additions without requiring structural rewrites, following Clean Architecture and Vertical Slice Architecture principles with Domain-Driven Design.

### 2.2 Success Criteria for MVP

- A user can complete the full lifecycle of an application (log → update status → view statistics) without consulting documentation.
- Closing a Search Period and opening a new one preserves the closed period's computed statistics exactly.
- All database schema changes are managed through EF Core migrations with no manual SQL required.

---

## 3. Stakeholders and Users

### 3.1 Primary User

**Job Seeker** — An individual actively applying for employment. They are comfortable using a terminal application. They want to log applications quickly and review their progress periodically without manual effort.

### 3.2 Developer / Maintainer

The developer responsible for building and extending JobTrack. This document is the primary reference for design decisions. The developer must be able to add new features (e.g., additional statistics, new fields) with minimal surface area of change, enabled by the Vertical Slice Architecture described in Section 7.

---

## 4. Functional Requirements

### 4.1 Search Period Management

**FR-SP-01** The user can create a new Search Period by providing a user-defined name (e.g., _"Summer 2025 Search"_). The system records the start date automatically using the current date.

**FR-SP-02** At any given time, only one Search Period may be in the `Active` state.

**FR-SP-03** The user can view a summary of all Search Periods, including their name, start date, end date (if closed), status, and the total number of applications recorded within each.

**FR-SP-04** The user can close the current Active Search Period. Before the statistics snapshot is captured and the period is marked as closed, the system presents the user with an interactive review step:

- All applications in a non-terminal status are listed.
- Applications in `Submitted` status for longer than one calendar month are visually highlighted.
- The user may select any of these applications and mark them as `AssumedRejection` before proceeding.
- The user may also choose to proceed without making any changes.
- Only after this step completes does the system capture the statistics snapshot and finalise the closure.

Applications in non-terminal statuses that are not actioned during this step remain frozen in the closed period until the user explicitly moves them forward.

**FR-SP-05** After closing a period, the user may begin a new Search Period. The system enforces that only one period is active at a time; attempting to create a new period while one is already active returns a clear error.

**FR-SP-06** A closed Search Period is permanently locked. Its application records and derived statistics cannot be altered, with the sole exception of the `Move Forward` action (see FR-APP-08), which does not modify the closed period's records, and note edits (see FR-APP-07), which have no effect on statistics.

### 4.2 Application Management

**FR-APP-01** The user can add a new Job Application to the currently Active Search Period. The following fields are captured:

|Field|Required|Notes|
|---|---|---|
|Company Name|Yes|Free text|
|Job Title|Yes|Free text|
|Date Submitted|Yes|Defaults to today; user may override|
|Application Source / Job Board|No|Free text (e.g., _LinkedIn_, _direct_)|
|Contact Person / Recruiter Name|No|Free text|
|Salary / Compensation Range|No|Free text (deliberately unstructured for flexibility)|
|Location / Remote Status|No|Free text (e.g., _London_, _Remote_, _Hybrid – Manchester_)|
|Notes|No|Multi-line free text|
|Link to Job Posting|No|URL string; no format validation in MVP|

**FR-APP-02** The user can list all applications within a given Search Period, with optional filtering by status.

**FR-APP-03** The user can view the full detail of a single application by its identifier, including its complete status change history.

**FR-APP-04** The user can edit any field of an application that belongs to the **Active** Search Period. Fields on applications belonging to **Closed** Search Periods are immutable.

**FR-APP-05** The user can update the status of an application in the Active period. Valid status transitions are defined in Section 12.2.

**FR-APP-06** When updating the status of an application to any terminal or soft-terminal status (`Hired`, `Rejected`, `Withdrawn`, `AssumedRejection`), the system automatically records the date of that status change if the user does not provide one.

**FR-APP-07** The user can add, edit, or delete notes on any application, regardless of whether the application belongs to an Active or Closed Search Period. Notes are not part of the statistical record and their mutation has no impact on any computed statistics or locked snapshot.

**FR-APP-08 — Move Forward.** For any application in a Closed Search Period, the user may choose to move it forward to the Active period. This action:

- Creates a **new** Application record in the Active period, copying all field values from the original at the point of the action.
- Records a reference (`OriginApplicationId`) on the new record pointing to the original record.
- Records a reference (`ForwardedApplicationId`) on the original record pointing to the new record, to prevent the same application being moved forward twice.
- **Does not alter** the original record's status, dates, or any other field.
- The new record's `DateSubmitted` remains the original submission date so that the user retains context; a `DateMovedForward` field is recorded separately.
- The new record starts in whatever status the original held at the time of the move, giving the user a natural starting point to update it.

**FR-APP-09** A moved-forward application is treated as a wholly independent application within the new period for all statistical purposes.

### 4.3 Statistics

**FR-STAT-01** The user can view statistics for any Search Period (Active or Closed). Statistics for a Closed period are read from the locked snapshot captured at closure. Statistics for the Active period are calculated from live data at query time.

The following statistics are produced per period:

|Statistic|Definition|
|---|---|
|Total Applications Sent|Count of all applications in the period|
|Response Rate|% of applications that received a qualifying employer response (see Section 13.2)|
|Rejection Rate|% of total applications that reached `Rejected` status|
|Assumed Rejection Rate|% of total applications that reached `AssumedRejection` status|
|Interview Conversion Rate|% of total applications that reached `Interviewing` status or beyond|
|Offer Rate|% of total applications that reached `OfferReceived` or `Hired` status|
|Average Time to First Response|Mean days from submission to first qualifying response (see Section 13.7)|
|Applications by Status|Count of applications in each status|
|Applications Over Time|Count of applications submitted per calendar week|

Full statistical definitions and edge case handling are specified in Section 13.

---

## 5. Non-Functional Requirements

**NFR-01 — Correctness of closed-period statistics.** Statistics for a Closed Search Period must produce identical results on every invocation, regardless of any changes made to Active period data.

**NFR-02 — Data integrity.** Foreign key constraints, non-nullable fields, and domain invariants must be enforced at both the domain layer and the database layer. EF Core migrations must reflect all constraints.

**NFR-03 — Usability.** All interactive prompts and tables must be rendered using Spectre Console. Error messages must be human-readable and suggest a corrective action where applicable.

**NFR-04 — Extensibility.** Adding a new feature (e.g., a new statistic, a new application field) must not require changes outside the relevant vertical slice and its supporting domain types.

**NFR-05 — Local-first.** All data is stored locally. There is no network dependency in the MVP beyond the SQL Server connection string pointing to a local instance.

**NFR-06 — Migration safety.** All schema changes must be expressed as EF Core migrations. Applying migrations must be a non-destructive operation on existing data wherever possible.

**NFR-07 — Startup resilience.** On first run, if the database does not exist or migrations are pending, the application applies them automatically and informs the user.

**NFR-08 — Structured logging.** All logging is handled by Serilog and written to a rolling file sink. Nothing is written to the console by the logger; the console is reserved exclusively for Spectre Console output.

---

## 6. Domain Model

This section defines the core domain concepts using Domain-Driven Design vocabulary.

### 6.1 Bounded Context

The MVP comprises a single bounded context: **Job Search Tracking**. There is no need for context mapping at this stage.

### 6.2 Enumerations as Smart Enums

All enumerations that carry behaviour or require display logic are implemented using **Ardalis.SmartEnum**.

The `ApplicationStatus` type carries status-transition logic and therefore uses an abstract base class pattern, where each status value is a private sealed nested class that overrides a `PermittedTransitions` property. `SearchPeriodStatus` carries no such behaviour and can be a straightforward sealed `SmartEnum<SearchPeriodStatus>`.

### 6.3 Aggregates

#### 6.3.1 SearchPeriod (Aggregate Root)

Represents a discrete period of job-seeking activity. It is the unit around which statistics are computed and locked.

**Identity:** `SearchPeriodId` — strongly-typed wrapper over `Guid` extending `AggregateRootId<Guid>` from `Erdmier.DomainCore`.

**Properties:**

|Property|Type|Description|
|---|---|---|
|`SearchPeriodId`|`SearchPeriodId`|Unique identifier|
|`Name`|`string`|User-defined label (required, max 200 chars)|
|`StartDate`|`DateOnly`|Set automatically at creation|
|`EndDate`|`DateOnly?`|Null while Active; set at closure|
|`Status`|`SearchPeriodStatus`|SmartEnum: `Active` or `Closed`|
|`LockedStatistics`|`LockedStatisticsSnapshot?`|Owned entity; null while Active|

**Behaviours:**

- `Close()` — Transitions status to `Closed`, sets `EndDate` to today. Returns `ErrorOr<Success>`. Raises `SearchPeriodClosedEvent`.
- Cannot be re-opened once closed; an attempt returns `Error.Conflict(...)`.

#### 6.3.2 JobApplication (Aggregate Root)

Represents a single job application submitted by the user.

**Identity:** `JobApplicationId` — strongly-typed wrapper over `Guid` extending `AggregateRootId<Guid>` from `Erdmier.DomainCore`.

**Properties:**

|Property|Type|Description|
|---|---|---|
|`JobApplicationId`|`JobApplicationId`|Unique identifier|
|`SearchPeriodId`|`SearchPeriodId`|FK to owning Search Period|
|`CompanyName`|`string`|Required|
|`JobTitle`|`string`|Required|
|`DateSubmitted`|`DateOnly`|Date application was sent|
|`Source`|`string?`|Job board or source channel|
|`ContactPerson`|`string?`|Recruiter or hiring manager name|
|`SalaryRange`|`string?`|Freeform compensation information|
|`Location`|`string?`|Location or remote status description|
|`JobPostingUrl`|`string?`|Link to the job posting|
|`Status`|`ApplicationStatus`|Current status (Smart Enum)|
|`StatusHistory`|`IReadOnlyList<StatusChangeRecord>`|Ordered audit trail of all status changes|
|`DateOfFirstQualifyingResponse`|`DateOnly?`|Set on first qualifying response (see Section 13.2)|
|`DateOfTerminalStatus`|`DateOnly?`|Set when a terminal/soft-terminal status is reached|
|`OriginApplicationId`|`JobApplicationId?`|Non-null if created via Move Forward|
|`ForwardedApplicationId`|`JobApplicationId?`|Non-null if this application has been moved forward|
|`DateMovedForward`|`DateOnly?`|Set when this record was created via a Move Forward action|
|`Notes`|`IReadOnlyList<ApplicationNote>`|Mutable notes collection|

**Behaviours:**

- `UpdateStatus(ApplicationStatus newStatus, DateOnly? effectiveDate)` — Validates the transition via the Smart Enum's `PermittedTransitions`. Appends a `StatusChangeRecord`. Updates `DateOfFirstQualifyingResponse` if this is the first qualifying transition away from `Submitted`. Updates `DateOfTerminalStatus` if the new status is terminal or soft-terminal. Returns `ErrorOr<Success>`. Raises `ApplicationStatusChangedEvent`.
- `AddNote(string content)` — Appends a new `ApplicationNote`.
- `EditNote(NoteId noteId, string newContent)` — Updates the content of an existing note. Returns `ErrorOr<Success>`.
- `DeleteNote(NoteId noteId)` — Removes a note. Returns `ErrorOr<Success>`.
- `MarkAsForwarded(JobApplicationId forwardedApplicationId)` — Records the forward reference, preventing a second forward action. Returns `ErrorOr<Success>`.

### 6.4 Value Objects and Entities

#### ApplicationStatus (Smart Enum)

Implemented as `abstract class ApplicationStatus : SmartEnum<ApplicationStatus>` with each concrete value as a private sealed nested class overriding a `PermittedTransitions` property returning `IReadOnlyList<ApplicationStatus>`.

|Value|Description|Statistical Treatment|
|---|---|---|
|`Submitted`|Application sent; no response yet|Active|
|`AcknowledgedUnderReview`|Employer has acknowledged receipt|Active|
|`Interviewing`|One or more interview stages underway|Active|
|`OfferReceived`|A job offer has been extended|Active|
|`Hired`|Offer accepted; position secured|Terminal|
|`Rejected`|Application declined by employer|Terminal|
|`Withdrawn`|Application withdrawn by the user|Terminal|
|`AssumedRejection`|No response after significant time|Soft-terminal (see Section 12.2)|

#### SearchPeriodStatus (Smart Enum)

|Value|Description|
|---|---|
|`Active`|The current, writeable period|
|`Closed`|A historical, locked period|

#### StatusChangeRecord (Entity within JobApplication)

An append-only audit record. Never modified or removed once created. Extends `Entity<StatusChangeRecordId>` from `Erdmier.DomainCore`.

|Property|Type|Description|
|---|---|---|
|`StatusChangeRecordId`|`StatusChangeRecordId`|Unique identifier|
|`JobApplicationId`|`JobApplicationId`|FK to parent application|
|`FromStatus`|`ApplicationStatus`|Status before the transition|
|`ToStatus`|`ApplicationStatus`|Status after the transition|
|`EffectiveDate`|`DateOnly`|Date the change took effect|
|`RecordedAt`|`DateTimeOffset`|UTC timestamp when this record was persisted|

#### ApplicationNote (Entity within JobApplication)

Extends `Entity<NoteId>` from `Erdmier.DomainCore`.

|Property|Type|Description|
|---|---|---|
|`NoteId`|`NoteId`|Unique identifier|
|`JobApplicationId`|`JobApplicationId`|FK to parent application|
|`Content`|`string`|Note text|
|`CreatedAt`|`DateTimeOffset`|UTC timestamp of initial creation|
|`LastModifiedAt`|`DateTimeOffset?`|UTC timestamp of most recent edit; null if never edited|

Notes are mutable. Edits and deletions are permitted regardless of whether the parent Search Period is Active or Closed, since notes carry no statistical weight.

#### LockedStatisticsSnapshot (Value Object — Owned Entity)

Stores pre-computed statistics for a closed Search Period. Extends `ValueObject` from `Erdmier.DomainCore`. Persisted as an EF Core owned entity to a dedicated table, giving each statistic a typed column rather than a JSON blob. Written atomically in the same transaction as `SearchPeriod.Close()`.

|Property|Type|Description|
|---|---|---|
|`TotalApplications`|`int`|Total count|
|`ResponseRate`|`decimal?`|Percentage; null if no applications|
|`RejectionRate`|`decimal?`|Percentage; null if no applications|
|`AssumedRejectionRate`|`decimal?`|Percentage; null if no applications|
|`InterviewConversionRate`|`decimal?`|Percentage; null if no applications|
|`OfferRate`|`decimal?`|Percentage; null if no applications|
|`AverageTimeToFirstResponseDays`|`decimal?`|Mean days; null if no qualifying responses|
|`ApplicationsByStatus`|`IReadOnlyList<StatusCount>`|Owned collection|
|`ApplicationsPerWeek`|`IReadOnlyList<WeeklyApplicationCount>`|Owned collection|

`StatusCount` and `WeeklyApplicationCount` are further owned entities persisted to `LockedStatusCounts` and `LockedWeeklyApplicationCounts` tables respectively.

### 6.5 Domain Services

#### StatisticsCalculationService

Calculates all statistics for a given collection of `JobApplication` entities and their `StatusHistory`. Returns `ErrorOr<LockedStatisticsSnapshot>`. Called in two contexts: on-demand for the Active period (result not persisted) and at period closure (result persisted as an owned entity).

#### ApplicationMoveService

Encapsulates the Move Forward operation. Returns `ErrorOr<JobApplication>`. Verifies the source belongs to a Closed period, has not already been forwarded, and that an Active period exists. Creates the new application, then calls `MarkAsForwarded` on the original.

### 6.6 Domain Events

Domain events are dispatched asynchronously using an EF Core `SaveChangesInterceptor` (`PublishDomainEventsInterceptor`). The interceptor collects all `IDomainEvent` instances from tracked `IHasDomainEvents` entities, clears them before the save, completes the save, then publishes each event to the Mediator `IPublisher`. This ensures events are only raised for data durably committed to the database.

`IDomainEvent` and `IHasDomainEvents` are provided by `Erdmier.DomainCore.Mediator`.

|Event|Raised By|Description|
|---|---|---|
|`SearchPeriodClosedEvent`|`SearchPeriod.Close()`|Triggers statistics calculation and snapshot persistence|
|`ApplicationStatusChangedEvent`|`JobApplication.UpdateStatus()`|Carries previous status, new status, and application ID|
|`ApplicationMovedForwardEvent`|`ApplicationMoveService`|Carries origin and new application IDs|

---

## 7. Architecture Overview

### 7.1 Architectural Approach

JobTrack is built on three complementary architectural patterns: **Clean Architecture** for layer boundaries, **Vertical Slice Architecture** for feature organisation, and **Domain-Driven Design** for the domain model.

**Clean Architecture** provides the layer structure. Domain and application logic are independent of infrastructure and presentation concerns. Dependencies point inwards — the domain has no dependency on EF Core, Spectre Console, or SQL Server.

**Vertical Slice Architecture** organises code by feature rather than by layer. Each user-facing feature (e.g., _Add Application_, _Close Period_, _View Statistics_) is self-contained in its own slice, with its own command, handler, request/response types, and any required validator. Shared infrastructure (DB context, domain types, base classes) lives in common areas but is kept minimal.

**Domain-Driven Design** defines the domain model as described in Section 6. The domain layer contains aggregates, value objects, domain services, and domain events. It has zero external dependencies beyond its declared NuGet packages.

### 7.2 Layer Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLI (Presentation)                         │
│   Spectre Console CLI commands / prompts / table rendering      │
│   Maps user input → Application layer requests                  │
│   Maps Application layer responses → console output             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ depends on
┌──────────────────────────▼──────────────────────────────────────┐
│                   Application Layer                             │
│   Vertical slices (one folder per feature)                      │
│   ICommand<T> / IQuery<T> handlers (Mediator)                   │
│   Validation (FluentValidation) pipeline behaviour              │
│   Orchestrates domain objects; owns use-case logic              │
│   Defines repository interfaces                                 │
└───────────┬──────────────────────────────────┬──────────────────┘
            │ depends on                        │ depends on (interfaces)
┌───────────▼──────────────┐   ┌───────────────▼──────────────────┐
│      Domain Layer        │   │       Persistence Layer           │
│  Aggregates              │   │   EF Core DbContext               │
│  Value Objects           │   │   Entity type configurations      │
│  Smart Enums             │   │   Repository implementations      │
│  Domain Services         │   │   EF Core migrations              │
│  Domain Events           │   │   PublishDomainEventsInterceptor  │
│  Repository interfaces   │   │   Database seeding                │
│  Deps: SmartEnum, ErrorOr│   │                                   │
│  Erdmier.DomainCore,     │   │                                   │
│  Erdmier.DomainCore      │   │                                   │
│  .Mediator               │   │                                   │
└──────────────────────────┘   └──────────────────────────────────┘
```

### 7.3 Dependency Flow

- **CLI** depends on **Application**.
- **Application** depends on **Domain**.
- **Persistence** depends on **Domain** (implements interfaces) and **Application** (for repository interfaces defined there).
- **Domain** depends on `Ardalis.SmartEnum`, `ErrorOr`, `Erdmier.DomainCore`, and `Erdmier.DomainCore.Mediator`; no EF Core, Spectre Console, or SQL Server dependencies.
- There is no separate Infrastructure layer. Domain event dispatching lives in the Persistence layer as an EF Core interceptor; logging configuration lives in the CLI composition root.

### 7.4 Mediator Pattern and CQRS

All application-layer use cases are expressed as CQRS commands or queries using **Martin Othamar's Mediator** library (source-generated, zero-reflection). Commands use `ICommand<TResponse>` handled by `ICommandHandler<TCommand, TResponse>`. Queries use `IQuery<TResponse>` handled by `IQueryHandler<TQuery, TResponse>`. The CLI dispatches through `ISender` and never calls handlers directly. All responses are typed as `ErrorOr<T>`.

---

## 8. Application Layers

### 8.1 Domain Layer (`JobTrack.Domain`)

Contains aggregates (`SearchPeriod`, `JobApplication`), entities (`StatusChangeRecord`, `ApplicationNote`), value objects (`LockedStatisticsSnapshot`, `StatusCount`, `WeeklyApplicationCount`), strongly-typed IDs (`SearchPeriodId`, `JobApplicationId`, `NoteId`, `StatusChangeRecordId`), Smart Enums (`ApplicationStatus`, `SearchPeriodStatus`), domain services (`StatisticsCalculationService`, `ApplicationMoveService`), domain events, repository interfaces, and domain-level error definitions (static partial `Errors` class).

Base types for aggregates, entities, entity IDs, value objects, and domain events (`AggregateRoot<,>`, `AggregateRootId<>`, `Entity<>`, `EntityId<>`, `ValueObject`, `IDomainEvent`, `IHasDomainEvents`) are consumed directly from the `Erdmier.DomainCore` and `Erdmier.DomainCore.Mediator` NuGet packages and are not re-implemented in the source code.

**NuGet dependencies:** `Ardalis.SmartEnum`, `ErrorOr`, `Mediator.Abstractions`, `Erdmier.DomainCore`, `Erdmier.DomainCore.Mediator`.

### 8.2 Application Layer (`JobTrack.Application`)

Contains one sub-folder per vertical slice, Mediator pipeline behaviours (`ValidationBehaviour`, `LoggingBehaviour`), and shared response types.

**NuGet dependencies:** `Mediator.Abstractions`, `Mediator.SourceGenerator`, `FluentValidation`, `FluentValidation.DependencyInjectionExtensions`.

### 8.3 Persistence Layer (`JobTrack.Persistence`)

Contains `JobTrackDbContext`, all `IEntityTypeConfiguration<T>` classes, EF Core migrations, `PublishDomainEventsInterceptor`, and repository implementations (`SearchPeriodRepository`, `JobApplicationRepository`).

**NuGet dependencies:** `Microsoft.EntityFrameworkCore`, `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Design`, `Ardalis.SmartEnum.EFCore`, `Mediator.Abstractions`.

### 8.4 CLI Layer (`JobTrack.Cli`)

Contains `Program.cs` (composition root: DI, EF Core, Mediator, Serilog, Spectre Console CLI, migration-on-startup), Spectre Console CLI `Command` implementations, and shared rendering helpers.

**NuGet dependencies:** `Spectre.Console`, `Spectre.Console.Cli`, `Microsoft.Extensions.DependencyInjection`, `Microsoft.Extensions.Hosting`, `Serilog`, `Serilog.Extensions.Hosting`, `Serilog.Sinks.File`.

---

## 9. Vertical Slice Design

Each slice lives under `JobTrack.Application/Features/<SliceName>/`. The corresponding CLI command lives under `JobTrack.Cli/Commands/<SliceName>/`.

### 9.1 Slice Catalogue

|Slice Name|Type|Description|
|---|---|---|
|`CreateSearchPeriod`|Command|Create a new Search Period|
|`CloseSearchPeriod`|Command|Interactive close flow; snapshot statistics; close period|
|`ListSearchPeriods`|Query|List all periods with summary data|
|`ViewSearchPeriodStatistics`|Query|Display statistics for a given period|
|`AddApplication`|Command|Log a new application in the Active period|
|`ListApplications`|Query|List applications, optionally filtered by period and/or status|
|`ViewApplication`|Query|Display full detail of a single application including status history|
|`EditApplication`|Command|Edit fields on an application in the Active period|
|`UpdateApplicationStatus`|Command|Change the status of an application|
|`AddApplicationNote`|Command|Add a note to an application|
|`EditApplicationNote`|Command|Edit an existing note|
|`DeleteApplicationNote`|Command|Delete a note|
|`MoveApplicationForward`|Command|Move a closed-period application to the Active period|
|`BulkMarkAssumedRejection`|Command|Mark multiple applications as `AssumedRejection` during the close flow|

### 9.2 Slice Internal Structure

```
Features/
  AddApplication/
    AddApplicationCommand.cs          ← ICommand<ErrorOr<JobApplicationResponse>>
    AddApplicationCommandHandler.cs   ← ICommandHandler implementation
    AddApplicationValidator.cs        ← AbstractValidator<AddApplicationCommand>
    AddApplicationResponse.cs         ← Response DTO (may be shared across slices)

  ViewSearchPeriodStatistics/
    ViewSearchPeriodStatisticsQuery.cs
    ViewSearchPeriodStatisticsQueryHandler.cs
    ViewSearchPeriodStatisticsResponse.cs
```

### 9.3 Pipeline Behaviours

Behaviours are registered as `IPipelineBehavior<TMessage, TResponse>` and execute in the following order for every command and query:

1. **LoggingBehaviour** — Logs the message type and elapsed time at `Debug` level via Serilog.
2. **ValidationBehaviour** — Runs all registered `AbstractValidator<TMessage>` instances. On failure, returns the validation errors as `Error.Validation(...)` without calling the handler.

Unexpected exceptions (database failures, programming errors) are not caught by any behaviour; they propagate to the global handler in `Program.cs`, where Serilog logs them and a concise terminal message is shown.

---

## 10. Data Model and Persistence

### 10.1 Database

A local Microsoft SQL Server instance. The connection string is read from user secrets in the CLI project.

### 10.2 Entity Relationship Overview

```
SearchPeriods ──────────────< JobApplications
     │                              │
     │                              ├──< StatusChangeRecords
     │                              │
     │                              └──< ApplicationNotes
     │
     └──[owns]── LockedStatisticsSnapshots
                        │
                        ├──< LockedStatusCounts
                        └──< LockedWeeklyApplicationCounts

JobApplications (self-referencing: OriginApplicationId / ForwardedApplicationId)
```

### 10.3 Table Definitions

#### `SearchPeriods`

|Column|Type|Constraints|
|---|---|---|
|`SearchPeriodId`|`uniqueidentifier`|PK, not null|
|`Name`|`nvarchar(200)`|Not null|
|`StartDate`|`date`|Not null|
|`EndDate`|`date`|Nullable|
|`Status`|`int`|Not null; SmartEnum integer value|
|`CreatedAt`|`datetimeoffset`|Not null|
|`UpdatedAt`|`datetimeoffset`|Not null|

#### `LockedStatisticsSnapshots`

Owned entity of `SearchPeriod`. One row per closed period.

|Column|Type|Constraints|
|---|---|---|
|`SearchPeriodId`|`uniqueidentifier`|PK & FK → `SearchPeriods`, not null|
|`TotalApplications`|`int`|Not null|
|`ResponseRate`|`decimal(5,2)`|Nullable|
|`RejectionRate`|`decimal(5,2)`|Nullable|
|`AssumedRejectionRate`|`decimal(5,2)`|Nullable|
|`InterviewConversionRate`|`decimal(5,2)`|Nullable|
|`OfferRate`|`decimal(5,2)`|Nullable|
|`AverageTimeToFirstResponseDays`|`decimal(8,2)`|Nullable|

#### `LockedStatusCounts`

|Column|Type|Constraints|
|---|---|---|
|`SearchPeriodId`|`uniqueidentifier`|PK (composite), FK → `SearchPeriods`|
|`Status`|`int`|PK (composite); SmartEnum integer value|
|`Count`|`int`|Not null|

#### `LockedWeeklyApplicationCounts`

|Column|Type|Constraints|
|---|---|---|
|`SearchPeriodId`|`uniqueidentifier`|PK (composite), FK → `SearchPeriods`|
|`WeekStartDate`|`date`|PK (composite); Monday of the ISO week|
|`Count`|`int`|Not null|

#### `JobApplications`

|Column|Type|Constraints|
|---|---|---|
|`JobApplicationId`|`uniqueidentifier`|PK, not null|
|`SearchPeriodId`|`uniqueidentifier`|FK → `SearchPeriods`, not null|
|`CompanyName`|`nvarchar(200)`|Not null|
|`JobTitle`|`nvarchar(200)`|Not null|
|`DateSubmitted`|`date`|Not null|
|`Source`|`nvarchar(200)`|Nullable|
|`ContactPerson`|`nvarchar(200)`|Nullable|
|`SalaryRange`|`nvarchar(200)`|Nullable|
|`Location`|`nvarchar(200)`|Nullable|
|`JobPostingUrl`|`nvarchar(2000)`|Nullable|
|`Status`|`int`|Not null; SmartEnum integer value|
|`DateOfFirstQualifyingResponse`|`date`|Nullable|
|`DateOfTerminalStatus`|`date`|Nullable|
|`OriginApplicationId`|`uniqueidentifier`|Nullable; FK → `JobApplications` (self), `DeleteBehavior.Restrict`|
|`ForwardedApplicationId`|`uniqueidentifier`|Nullable; FK → `JobApplications` (self), `DeleteBehavior.Restrict`|
|`DateMovedForward`|`date`|Nullable|
|`CreatedAt`|`datetimeoffset`|Not null|
|`UpdatedAt`|`datetimeoffset`|Not null|

#### `StatusChangeRecords`

|Column|Type|Constraints|
|---|---|---|
|`StatusChangeRecordId`|`uniqueidentifier`|PK, not null|
|`JobApplicationId`|`uniqueidentifier`|FK → `JobApplications`, cascade delete, not null|
|`FromStatus`|`int`|Not null; SmartEnum integer value|
|`ToStatus`|`int`|Not null; SmartEnum integer value|
|`EffectiveDate`|`date`|Not null|
|`RecordedAt`|`datetimeoffset`|Not null|

#### `ApplicationNotes`

|Column|Type|Constraints|
|---|---|---|
|`NoteId`|`uniqueidentifier`|PK, not null|
|`JobApplicationId`|`uniqueidentifier`|FK → `JobApplications`, cascade delete, not null|
|`Content`|`nvarchar(max)`|Not null|
|`CreatedAt`|`datetimeoffset`|Not null|
|`LastModifiedAt`|`datetimeoffset`|Nullable|

### 10.4 EF Core Configuration

Each entity has a dedicated `IEntityTypeConfiguration<T>` class. All primary keys use `ValueGeneratedNever()`. Smart Enum integer values are persisted via `configurationBuilder.ConfigureSmartEnum()` in `ConfigureConventions`. `LockedStatisticsSnapshot`, `LockedStatusCounts`, and `LockedWeeklyApplicationCounts` are configured as owned entities via `OwnsOne`/`OwnsMany`. A filtered unique index on `SearchPeriods` where `Status = <Active integer value>` enforces the database-level single-active-period constraint.

### 10.5 Domain Event Interceptor

`PublishDomainEventsInterceptor` inherits from `SaveChangesInterceptor` and overrides both `SavingChanges` and `SavingChangesAsync`. It collects all pending `IDomainEvent` instances from the EF Core change tracker, clears them, completes the save, then publishes each event via Mediator `IPublisher`. Events are only dispatched for durably committed data.

### 10.6 Migrations

Managed with `dotnet ef`. Startup project: `JobTrack.Cli`; migrations project: `JobTrack.Persistence`. `Database.MigrateAsync()` is called at startup, creating the database and applying pending migrations automatically.

### 10.7 Repository Interfaces (Domain Layer)

```csharp
public interface ISearchPeriodRepository
{
    Task<ErrorOr<SearchPeriod>> GetByIdAsync(SearchPeriodId id, CancellationToken ct = default);
    Task<SearchPeriod?> GetActiveAsync(CancellationToken ct = default);
    Task<IReadOnlyList<SearchPeriod>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(SearchPeriod period, CancellationToken ct = default);
    Task UpdateAsync(SearchPeriod period, CancellationToken ct = default);
}

public interface IJobApplicationRepository
{
    Task<ErrorOr<JobApplication>> GetByIdAsync(JobApplicationId id, CancellationToken ct = default);
    Task<IReadOnlyList<JobApplication>> GetByPeriodAsync(SearchPeriodId periodId, CancellationToken ct = default);
    Task AddAsync(JobApplication application, CancellationToken ct = default);
    Task UpdateAsync(JobApplication application, CancellationToken ct = default);
}
```

`GetByIdAsync` returns `ErrorOr<T>` so handlers can propagate typed not-found errors (e.g., `Errors.JobApplication.NotFound(id)`, `Errors.SearchPeriod.AlreadyClosed(id)`) without explicit null checks. `GetActiveAsync` returns a nullable because "no active period" is a legitimate system state for certain queries, not an error.

---

## 11. CLI Interface Design

### 11.1 Command Structure

```
jobtrack period create
jobtrack period list
jobtrack period close
jobtrack period stats [--id <period-id>]

jobtrack app add
jobtrack app list [--period <period-id>] [--status <status>]
jobtrack app view <id>
jobtrack app edit <id>
jobtrack app status <id> <new-status>
jobtrack app note add <id>
jobtrack app note edit <id> <note-id>
jobtrack app note delete <id> <note-id>
jobtrack app move <id>
```

### 11.2 Interactive Prompts

Where field values are not provided as arguments, `AnsiConsole.Prompt` collects input. Required fields must be answered; optional fields display a `[skip]` affordance. For status selection, a `SelectionPrompt<ApplicationStatus>` uses display names derived from the Smart Enum's `Name` property.

### 11.3 Close Period Interactive Flow

When `jobtrack period close` is invoked:

1. A `ConfirmationPrompt` warns the user they are about to close the current period.
2. All applications in a non-terminal status are queried.
3. If any exist, they are rendered in a Spectre Console `Table`. Applications in `Submitted` status with a `DateSubmitted` more than one calendar month ago are rendered with a yellow highlight.
4. A `MultiSelectionPrompt<JobApplication>` allows the user to select any applications to mark as `AssumedRejection`.
5. If any are selected, `BulkMarkAssumedRejection` is dispatched.
6. A final `ConfirmationPrompt` confirms the close. If confirmed, `CloseSearchPeriod` is dispatched, capturing the statistics snapshot and finalising the closure.

### 11.4 Output Rendering

**Tables** for list views; **Panels** for detail views. Status badges use coloured `Markup` tags:

|Status|Colour|
|---|---|
|`Submitted`|Blue|
|`AcknowledgedUnderReview`|Cyan|
|`Interviewing`|Yellow|
|`OfferReceived`|Green|
|`Hired`|Bold Green|
|`Rejected`|Red|
|`Withdrawn`|Grey|
|`AssumedRejection`|Dark Orange|

`AnsiConsole.Status` spinners are shown during database operations. `ConfirmationPrompt` is used before Close Period and Move Forward actions.

### 11.5 Error Display

Validation errors are displayed as a red-bordered panel listing each field and its message. `ErrorOr` business failures produce a single-line error panel. Unexpected exceptions are logged to file by Serilog; the terminal shows only a brief message with the log file path.

### 11.6 Logging

Serilog is configured as the sole logging provider, writing to a rolling file sink at `{ApplicationDirectory}/logs/jobtrack-.log` (daily rolling). Minimum level is `Debug` in development and `Warning` in production, configured via `appsettings.json`. No Serilog output appears on the console; all console output is produced exclusively by Spectre Console.

---

## 12. Business Rules and Invariants

### 12.1 Search Period Rules

**BR-SP-01** A Search Period's `Name` must be between 1 and 200 characters and must not be whitespace-only.

**BR-SP-02** Only one Search Period may hold `Active` status at a time. Enforced at the application layer and at the database layer via a filtered unique index.

**BR-SP-03** A Search Period cannot be closed if it is already closed. Returns `Error.Conflict(...)`.

**BR-SP-04** A new Search Period cannot be created while an Active period exists.

**BR-SP-05** The `LockedStatisticsSnapshot` for a closed period must never be modified after being written.

### 12.2 Application Status Transition Rules

Transitions are validated by the `PermittedTransitions` property of the current `ApplicationStatus` Smart Enum value. `AssumedRejection` is a **soft-terminal** status — it is counted as terminal in all statistical calculations, but permits the same outward transitions as `AcknowledgedUnderReview` to handle the edge case of a delayed employer response.

```
Submitted
  ├──► AcknowledgedUnderReview
  ├──► Interviewing
  ├──► Rejected
  ├──► AssumedRejection
  └──► Withdrawn

AcknowledgedUnderReview
  ├──► Interviewing
  ├──► OfferReceived
  ├──► Rejected
  ├──► AssumedRejection
  └──► Withdrawn

Interviewing
  ├──► OfferReceived
  ├──► Rejected
  └──► Withdrawn

OfferReceived
  ├──► Hired
  ├──► Rejected
  └──► Withdrawn

AssumedRejection  ← Soft-terminal. Same outward transitions as AcknowledgedUnderReview.
  ├──► Interviewing
  ├──► OfferReceived
  ├──► Rejected
  └──► Withdrawn

Hired      ← Terminal. No further transitions.
Rejected   ← Terminal. No further transitions.
Withdrawn  ← Terminal. No further transitions.
```

Any attempt to transition outside these paths returns `Error.Validation(...)` from `UpdateStatus()`. Re-entering the same status is not permitted.

### 12.3 Application Edit Rules

**BR-APP-01** Core fields may only be edited if the application belongs to the Active Search Period.

**BR-APP-02** Notes (add, edit, delete) may be performed on applications in any period — Active or Closed — since notes carry no statistical weight.

**BR-APP-03** `DateSubmitted` may not be set to a future date.

**BR-APP-04** `DateSubmitted` may not be set to a date before the `StartDate` of the owning Search Period.

### 12.4 Move Forward Rules

**BR-MV-01** The source application must belong to a Closed Search Period.

**BR-MV-02** The source application must not already have been forwarded (`ForwardedApplicationId` must be null).

**BR-MV-03** An Active Search Period must exist to receive the application.

**BR-MV-04** Once forwarded, the source application's core fields and status are immutable (it belongs to a Closed period, and `ForwardedApplicationId` being set makes this explicit in the domain).

---

## 13. Statistics Specification

Statistics are computed by `StatisticsCalculationService` using application records and their `StatusHistory`. For Closed periods, the pre-computed `LockedStatisticsSnapshot` is returned directly. For the Active period, statistics are computed from live data at query time.

### 13.1 Total Applications Sent

```
TotalApplications = COUNT(all applications in the period)
```

Moved-forward applications (non-null `OriginApplicationId`) count as full members of their current period.

### 13.2 Response Rate

A **qualifying response** is defined as: the first `StatusChangeRecord` with `FromStatus = Submitted` has a `ToStatus` that is **not** `Withdrawn`. A user withdrawing their own application as the very first action is not an employer response.

```
QualifyingResponseCount =
    COUNT(applications where the first StatusChangeRecord
          with FromStatus = Submitted has ToStatus ∉ {Withdrawn})

ResponseRate = (QualifyingResponseCount / TotalApplications) × 100
```

If `TotalApplications` is 0, displayed as N/A.

### 13.3 Rejection Rate

```
RejectionRate = (COUNT(applications where Status = Rejected) / TotalApplications) × 100
```

### 13.4 Assumed Rejection Rate

```
AssumedRejectionRate = (COUNT(applications where Status = AssumedRejection) / TotalApplications) × 100
```

### 13.5 Interview Conversion Rate

Status history is inspected so that applications which reached `Interviewing` but were subsequently `Rejected` are still counted.

```
InterviewConversionRate =
    (COUNT(applications where any StatusChangeRecord has
           ToStatus ∈ {Interviewing, OfferReceived, Hired})
     / TotalApplications) × 100
```

### 13.6 Offer Rate

```
OfferRate =
    (COUNT(applications where any StatusChangeRecord has
           ToStatus ∈ {OfferReceived, Hired})
     / TotalApplications) × 100
```

### 13.7 Average Time to First Qualifying Response

```
AverageTimeToFirstQualifyingResponse =
    MEAN(DateOfFirstQualifyingResponse − DateSubmitted)
    for all applications where DateOfFirstQualifyingResponse IS NOT NULL
```

`DateOfFirstQualifyingResponse` is set by `UpdateStatus()` when the first transition away from `Submitted` has a `ToStatus` that is not `Withdrawn` (the same qualifying condition as Section 13.2). If the first transition is to `Withdrawn`, `DateOfFirstQualifyingResponse` remains null and the application is excluded. Result expressed in whole days (rounded to nearest integer). If no qualifying responses exist, displayed as N/A.

### 13.8 Applications by Status

Count per status value. All eight status values are always shown, including those with a count of zero.

### 13.9 Applications Over Time

Applications grouped by ISO calendar week (Monday–Sunday) of `DateSubmitted`. Output is a list of `(WeekStartDate, Count)` pairs sorted chronologically. Weeks with no submissions are omitted.

### 13.10 Statistics Snapshot at Period Closure

When `SearchPeriodClosedEvent` is raised, its application-layer event handler calls `StatisticsCalculationService` with the full list of the period's applications and their status histories. The resulting `LockedStatisticsSnapshot` is attached to the `SearchPeriod` aggregate and persisted via `UpdateAsync` in the same unit of work.

---

## 14. Error Handling Strategy

### 14.1 Error Philosophy

`ErrorOr` is the primary error-signalling mechanism throughout the entire solution — domain methods, application handlers, and repository methods all return `ErrorOr<T>`. Exceptions are reserved for genuinely unrecoverable conditions: EF Core interceptors, Mediator pipeline behaviours, and unexpected runtime failures.

### 14.2 Error Categories

**Domain Errors** — Returned via `ErrorOr` from aggregate methods and domain services. Defined in static partial `Errors` classes in the Domain layer (e.g., `Errors.JobApplication.NotFound(id)`, `Errors.SearchPeriod.AlreadyClosed(id)`).

**Validation Errors** — Produced by `ValidationBehaviour` using FluentValidation before the handler executes. Aggregated failures are returned as `Error.Validation(...)` results.

**Persistence Errors** — EF Core throws on database failures. These propagate to the global handler in `Program.cs`.

**Unhandled Exceptions** — Caught only by the top-level handler in `Program.cs`. Logged in full by Serilog; the user sees a concise message with the log file path.

### 14.3 Mediator Pipeline Behaviours

1. **LoggingBehaviour** — Logs the message type and elapsed milliseconds at `Debug` level via Serilog.
2. **ValidationBehaviour** — Runs `AbstractValidator<TMessage>` instances. Returns validation errors as `ErrorOr` on failure; does not call the handler. Raises no exceptions.

---

## 15. Technology Stack

|Component|Technology|Notes|
|---|---|---|
|Language|C# / .NET 10||
|CLI Framework|Spectre.Console.Cli||
|Console Rendering|Spectre.Console||
|Mediator / CQRS|Mediator (Martin Othamar)|Source-generated; `ICommand<T>`, `IQuery<T>`|
|Validation|FluentValidation||
|Error Handling|ErrorOr (Amichai Mantinband)||
|Domain Core|Erdmier.DomainCore|Base types: `AggregateRoot<,>`, `AggregateRootId<>`, `Entity<>`, `EntityId<>`, `ValueObject`|
|Domain Events|Erdmier.DomainCore.Mediator|Base types: `IDomainEvent`, `IHasDomainEvents`|
|Smart Enums|Ardalis.SmartEnum||
|Smart Enums — EF Core|Ardalis.SmartEnum.EFCore||
|ORM|Entity Framework Core||
|Database Driver|EFCore.SqlServer||
|Database|Microsoft SQL Server|Local instance; Express edition sufficient|
|Logging|Serilog|File sink only; no console sink|
|Serilog Hosting|Serilog.Extensions.Hosting|`UseSerilog()` on the host builder|
|Serilog File Sink|Serilog.Sinks.File|Daily rolling log file|
|Dependency Injection|Microsoft.Extensions.DependencyInjection||
|Hosting|Microsoft.Extensions.Hosting|DI container and configuration|
|Configuration|Microsoft.Extensions.Configuration|`appsettings.json` and user secrets|

---

## 16. Key Design Decisions and Rationale

### 16.1 Why Vertical Slice Architecture over a traditional layered approach?

Vertical Slice Architecture organises code around features rather than technical layers. Adding a new feature requires changes only within its own slice folder, rather than touching Controllers, Services, Repositories, and DTOs scattered across separate layer directories. Given that JobTrack is expected to grow iteratively, VSA makes feature additions and deletions low-risk and self-contained.

### 16.2 Why keep `JobApplication` as a separate Aggregate Root rather than a child entity of `SearchPeriod`?

Making applications a child collection of `SearchPeriod` would require loading the entire period aggregate to perform any single application operation. More importantly, the Move Forward operation requires that an application can be referenced independently across period boundaries. A separate aggregate root with a foreign key to its period satisfies both concerns.

### 16.3 Why persist `LockedStatisticsSnapshot` as owned entity tables rather than JSON?

The statistics snapshot is a write-once, read-many artefact. Persisting it as a structured owned entity table gives each statistic a typed, queryable column — consistent with the `OwnsMany` patterns used for `SelectedTechniques` and `ScoredPlayerIds` in the reference codebase. It avoids EF Core value-converter complexity and makes the data directly inspectable. The trade-off — that adding a new statistic requires a migration — is acceptable given the MVP scope.

### 16.4 Why are notes mutable rather than append-only?

Notes are personal working memory for the user, not an audit trail. The `StatusChangeRecord` entity serves as the audit trail, and its append-only constraint is justified because status changes directly inform statistics. There is no correctness requirement for note immutability; enforcing it would create unnecessary user friction.

### 16.5 Why is `AssumedRejection` a soft-terminal status rather than a full terminal one?

The user's intent when marking an application as `AssumedRejection` is to acknowledge silence — not to permanently close the door. Employers do sometimes respond after long delays. Allowing the same outward transitions as `AcknowledgedUnderReview` means the user is not penalised for making a reasonable assumption, while still treating the status as terminal for all statistical rate calculations.

### 16.6 Why use Martin Othamar's Mediator rather than MediatR?

Mediator is source-generated and zero-reflection. Its interfaces explicitly distinguish `ICommand<T>` from `IQuery<T>`, enforcing CQRS intent at the type level — a command cannot accidentally be dispatched as a query and vice versa. This is consistent with the reference codebase. MediatR's `IRequest<T>` carries no such semantic distinction.

### 16.7 Why use strongly-typed ID wrappers rather than plain GUIDs?

Strongly-typed IDs prevent accidental cross-entity ID assignments at compile time. They are generated by the application (not the database), keeping the domain decoupled from database identity strategy.

### 16.8 Why is there no separate Infrastructure layer?

The MVP has no infrastructure concerns beyond persistence and logging. Domain event dispatching — which might traditionally live in Infrastructure — is implemented as an EF Core interceptor within the Persistence layer, following the reference codebase's `PublishDomainEventsInterceptor`. Serilog configuration lives in the CLI composition root. Adding a separate Infrastructure layer for these two concerns would introduce structural overhead without meaningful benefit at this stage.

### 16.9 Why is `DateOfFirstQualifyingResponse` distinct from a simple "first status change date"?

A user withdrawing their own application is not a response from the employer. Including that date in response-time averages would skew the metric with user-initiated, non-response events. The qualification rule (excluding transitions to `Withdrawn` as the first move away from `Submitted`) ensures the statistic reflects genuine employer engagement only.

---

## 17. Glossary

### Job Search Concepts

|Term|Definition|
|---|---|
|**Search Period**|A named, bounded window of time during which the user is actively job seeking. The primary organisational unit of the system.|
|**Active Search Period**|A Search Period currently in progress; applications may be added and modified.|
|**Closed Search Period**|A Search Period that has been ended by the user. Core application records and statistics are immutable; notes remain editable.|
|**Job Application**|A record representing a single application submitted by the user to a prospective employer within a given Search Period.|
|**Application Source**|The channel through which a Job Application was discovered or submitted (e.g., a job board, direct company website, or recruiter referral).|

### Application Lifecycle

|Term|Definition|
|---|---|
|**Application Status**|The current stage of a Job Application within its lifecycle, represented as a Smart Enum value with defined permitted transitions.|
|**Terminal Status**|An application status from which no further transitions are permitted (`Hired`, `Rejected`, `Withdrawn`).|
|**Soft-terminal Status**|A status treated as terminal for statistical calculations but permitting outward transitions. Specifically `AssumedRejection`.|
|**Assumed Rejection**|A soft-terminal application status indicating the user has assumed no employer response will be received. Permits outward transitions if an employer does eventually respond.|
|**Status Change Record**|An append-only audit entity recording a single status transition on a Job Application. Never modified or deleted once created.|
|**Qualifying Response**|A status transition away from `Submitted` to any status other than `Withdrawn` as the first transition. Used to calculate Response Rate and Average Time to First Response.|

### Statistics and History

|Term|Definition|
|---|---|
|**Locked Statistics**|Pre-computed, immutable statistics captured at Search Period closure and persisted as owned entity tables.|
|**Statistics Snapshot**|The specific instance of Locked Statistics attached to a single Closed Search Period, written atomically at the moment of closure.|
|**Response Rate**|The percentage of applications in a period that received at least one qualifying employer response.|
|**Rejection Rate**|The percentage of applications in a period that reached `Rejected` status.|
|**Assumed Rejection Rate**|The percentage of applications in a period that reached `AssumedRejection` status.|
|**Interview Conversion Rate**|The percentage of applications in a period that progressed to `Interviewing` status or beyond at any point in their history.|
|**Offer Rate**|The percentage of applications in a period that reached `OfferReceived` or `Hired` status at any point in their history.|
|**Average Time to First Response**|The mean number of days between `DateSubmitted` and `DateOfFirstQualifyingResponse` across all applications in a period that received a qualifying response.|

### Actions

|Term|Definition|
|---|---|
|**Move Forward**|The action of creating a linked copy of a closed-period application in the Active period, preserving the original record intact and unchanged.|
|**Bulk Mark as Assumed Rejection**|An action available during the Close Period flow, allowing the user to mark multiple non-terminal applications as `AssumedRejection` before the period is finalised.|

---

_End of Document_

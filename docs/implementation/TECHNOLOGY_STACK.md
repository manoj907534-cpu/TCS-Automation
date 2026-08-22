# TCS Automation — Technology Stack

**Version:** 0.1
**Status:** Implementation Baseline
**Baseline:** SRS v1.2 / Architecture v0.2 / Domain Model v0.3 / Test Run State Machine v0.4 / Adapter Interface Contract v0.1 / SQL Schema v0.1

## 1. Purpose

This document defines the approved technology baseline for TCS Automation V1.

Technology choices implement the approved architecture; they do not redefine the domain model, state machine, or adapter contract.

## 2. Technology Baseline

| Area | Technology | Version / Baseline | Status |
|---|---|---|---|
| Language | C# | 12 / .NET 8 | Approved |
| Runtime | .NET | 8 LTS | Approved |
| Desktop UI | WPF | .NET 8 | Approved |
| UI Pattern | MVVM | — | Approved |
| Database | SQLite | 3.x | Approved |
| Data Access | Dapper | 2.x | Approved |
| Windows Automation | FlaUI / Windows UI Automation | 4.x / UIA3 | Approved |
| Serialization | System.Text.Json | .NET 8 | Approved |
| Logging | Serilog | 4.x | Approved |
| Dependency Injection | Microsoft.Extensions.DependencyInjection | .NET 8 | Approved |
| Configuration | Microsoft.Extensions.Configuration | .NET 8 | Approved |
| TCS / Excel Import | ClosedXML | 0.1xx | Approved |
| Unit Testing | xUnit | 2.x | Approved |
| Mocking | NSubstitute | 5.x | Approved |
| Packaging | TBD | — | Deferred |
| Embedded Linux/Qt automation mechanism | Adapter-specific | TBD | Deferred |

> Exact package patch versions are controlled by the solution/project files. This document records the approved technology family and compatibility baseline rather than every patch release.

## 3. Architecture Mapping

| Architecture Area | Implementation Technology |
|---|---|
| Desktop Application / UI | WPF + MVVM |
| Core Domain | C# / .NET 8 class libraries |
| Application / Orchestration | C# / .NET 8 |
| Adapter Abstraction | C# interfaces defined by Adapter Interface Contract v0.1 |
| Windows Adapter | FlaUI / Windows UI Automation |
| Embedded Linux/Qt Adapter | Adapter-specific implementation; mechanism intentionally deferred |
| Data Layer | SQLite + Dapper |
| Evidence Store | Local filesystem |
| Logging | Serilog |
| Configuration | Microsoft.Extensions.Configuration + System.Text.Json where appropriate |
| TCS/Excel Import | ClosedXML |
| Automated Tests | xUnit + NSubstitute |

## 4. Key Technology Decisions

### 4.1 Desktop Application

**Decision:** WPF on .NET 8.

**Reason:** V1 is a Windows-hosted desktop application. WPF provides mature Windows integration and fits the planned Windows automation ecosystem.

The fact that the system under test may be Embedded Linux/Qt does not require the TCS Automation host UI to run on Linux. The host can communicate with the target through the appropriate adapter.

### 4.2 Database

**Decision:** SQLite for V1.

**Reason:** V1 is local and single-user, so a server database is unnecessary. SQLite provides a portable local database file and simple backup/restore.

The SQL Schema v0.1 currently documents PostgreSQL syntax with SQLite adjustment notes. During implementation, the executable DDL will be made explicitly SQLite-compatible. This does not change the domain model.

### 4.3 Data Access

**Decision:** Dapper.

**Reason:** The project already has an explicit relational SQL design and requires clear transaction boundaries for Test Run, Attempt and Result state changes. Dapper keeps SQL explicit and avoids unnecessary ORM behavior.

### 4.4 Windows Automation

**Decision:** FlaUI using the Windows UI Automation stack, initially targeting UIA3 where applicable.

**Reason:** It provides a .NET-native adapter implementation while keeping automation-specific details behind the Windows Adapter boundary.

The Core Engine must not depend directly on FlaUI APIs.

### 4.5 Embedded Linux / Qt Automation

**Decision:** Defer the concrete automation mechanism.

The adapter contract and canonical capabilities are frozen independently of the underlying Qt automation mechanism. The implementation will be selected after confirming how the target application exposes its UI/object information and communication interface.

### 4.6 Logging and Configuration

**Decision:** Serilog for application logging and Microsoft.Extensions.Configuration for configuration.

Logging must respect the evidence/security rules and must not expose credentials or other sensitive dataset values.

## 5. Technology Boundaries

The following boundaries are mandatory:

1. Core/domain code must not depend on WPF, FlaUI, SQLite or other infrastructure-specific APIs.
2. The Core Engine must use Adapter abstractions rather than directly calling an automation library.
3. Database access must remain behind the Data Layer.
4. Evidence storage must remain behind the Evidence Store abstraction.
5. Embedded Linux/Qt automation technology must remain inside its adapter implementation.
6. Technology-specific exceptions must be translated at the appropriate boundary into the error/fact model defined by the Adapter Contract.

## 6. Version and Change Management

This document is the implementation technology baseline.

### 6.1 Patch/minor dependency update

Example: updating a library within the approved major/compatibility baseline.

- Update the project dependency version.
- Run the relevant automated tests.
- No Technology Stack version bump is required unless the approved technology baseline changes.

### 6.2 Major framework/library change

A major version change requires a compatibility review. Update this document if the approved baseline changes.

### 6.3 Runtime change

A change such as .NET 8 → .NET 10 requires review of application compatibility, deployment, dependencies and test results before changing the baseline.

### 6.4 Technology replacement

A replacement such as SQLite → PostgreSQL, WPF → Avalonia, or FlaUI → another automation framework requires review of every affected architecture boundary and implementation contract.

Only affected design documents should be versioned; a technology change does not automatically require all documents to receive new versions.

## 7. Change Impact Guide

| Change | Technology Stack | Architecture | SQL Schema | Adapter Contract |
|---|---|---|---|---|
| Patch dependency update | Update project only | No | No | No |
| Major dependency update | Review | Maybe | Maybe | Maybe |
| .NET runtime change | Update/review | Review if behavior/boundaries change | No | No |
| WPF → Avalonia | Update | Review | No | Possibly |
| SQLite → PostgreSQL | Update | Review Data Layer | Update | No |
| FlaUI → another Windows framework | Update | Review adapter boundary | No | Review |
| New target adapter technology | Update | Review adapter architecture | No | Update if contract changes |

## 8. Deferred Decisions

The following are intentionally not frozen in Technology Stack v0.1:

- Exact Embedded Linux/Qt automation framework/mechanism.
- Final packaging/installer technology.
- Future centralized database technology.
- Future multi-user deployment technology.
- Exact patch-level dependency versions.

These may be selected during the corresponding implementation milestone without reopening unrelated design documents.

## 9. Implementation Rule

Once this technology baseline is accepted, implementation should proceed against it. Technology choices should not be changed merely for preference; a change should be justified by a demonstrated technical requirement, compatibility problem, or significant implementation constraint.

## 10. Changelog

### v0.1

- Initial implementation technology baseline.
- Selected C# / .NET 8, WPF, SQLite, Dapper, FlaUI, Serilog, System.Text.Json, ClosedXML, xUnit and NSubstitute.
- Deferred concrete Embedded Linux/Qt automation mechanism and packaging technology.

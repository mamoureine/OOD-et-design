# Library Management System — OOA and Architecture Design

## 1. Purpose and scope

This document models a small library system before implementation. It covers catalog search, member and catalog administration, circulation, and overdue tracking. It is an analysis and design artifact, not a complete application specification.

### Assumptions

- A catalog title can have several individually tracked physical copies. A copy, not a title, is issued to a member.
- Members can search the catalog; a librarian confirms checkouts and records returns.
- A member must be active and below the configured active-loan limit to borrow.
- Each copy can have at most one active loan. Loan duration and loan limit are library configuration.
- Overdue status is derived from an active loan's due date; no fines, reservations, payments, or inter-library transfers are in scope.
- Admin is a privileged librarian role that can manage staff, members, and catalog records.

## 2. Actors and requirements

### Actors

| Actor | Goal |
| --- | --- |
| Member | Search the catalog and see whether copies are available; review their active and past loans. |
| Librarian | Register or update members, maintain title and copy records, issue books, accept returns, and review overdue loans. |
| Administrator | Perform librarian tasks and manage staff access and library-wide settings. |
| Scheduler | Periodically identify overdue active loans and request notifications. |

### Main use cases

| Use case | Primary actor | Preconditions and outcome |
| --- | --- | --- |
| Search catalog | Member, Librarian | Search by title, author, ISBN, or subject; results include copy availability. |
| View member loans | Member, Librarian | A member sees their own loans; a librarian may look up a member's loans. |
| Issue a book | Librarian | Member is active, has capacity, and a copy is available; an active loan is created and the copy becomes issued. |
| Return a book | Librarian | An active loan is identified; the loan is closed and the copy becomes available after return processing. |
| Manage catalog | Librarian | Create or update a title and its physical copies; retire a copy only when it has no active loan. |
| Manage members | Librarian | Register, update, or deactivate a member; a member with an active loan cannot be deactivated. |
| Review overdue loans | Librarian, Scheduler | List active loans past their due date; scheduler requests a notice once per overdue loan. |
| Manage staff and settings | Administrator | Manage staff permissions and configure loan duration and loan limit. |

### Business rules

1. Checkout requires an active member, an available copy, and remaining member borrowing capacity.
2. Checkout must atomically create one active loan and mark its copy issued. A failure must leave both unchanged.
3. Return requires an active loan for the copy being returned. It records the return date and makes that copy available.
4. A copy with an active loan cannot be deleted or issued again.
5. A member can inspect only their own loan history unless the requester is a librarian or administrator.
6. Overdue means `loan.status = ACTIVE` and `today > loan.dueOn`. A notice is sent at most once for each loan.

## 3. High-level architecture

The application uses a layered design. The user interfaces depend on application services; services enforce use-case and domain rules; repositories isolate persistence. The scheduler uses the same service and notification boundary as interactive workflows.

```mermaid
flowchart TB
    subgraph Presentation["Presentation layer"]
        MemberUI["Member portal"]
        StaffUI["Librarian / admin console"]
    end

    subgraph Application["Application layer"]
        API["Controllers / API"]
        Auth["Authentication and role authorization"]
        CatalogSvc["Catalog service"]
        MemberSvc["Member service"]
        CirculationSvc["Circulation service"]
        ReportSvc["Loan and overdue queries"]
    end

    subgraph Domain["Domain layer"]
        Entities["BookTitle, BookCopy, Member, Loan"]
        Rules["Availability, eligibility, and loan rules"]
    end

    subgraph Infrastructure["Infrastructure layer"]
        Repos["Repository interfaces / implementations"]
        DB[("Relational database")]
        Scheduler["Overdue scheduler"]
        Notify["Notification adapter"]
    end

    MemberUI --> API
    StaffUI --> API
    API --> Auth
    Auth --> CatalogSvc
    Auth --> MemberSvc
    Auth --> CirculationSvc
    Auth --> ReportSvc
    CatalogSvc --> Entities
    MemberSvc --> Entities
    CirculationSvc --> Entities
    ReportSvc --> Entities
    Entities --> Rules
    CatalogSvc --> Repos
    MemberSvc --> Repos
    CirculationSvc --> Repos
    ReportSvc --> Repos
    Repos --> DB
    Scheduler --> ReportSvc
    Scheduler --> Notify
    ReportSvc --> Notify
```

### Component responsibilities

| Component | Responsibility |
| --- | --- |
| Member portal | Search and display catalog availability and the signed-in member's loans. |
| Librarian/admin console | Provide protected workflows for circulation, catalog, member, staff, and settings management. |
| Controllers and authorization | Validate requests, establish identity and role, invoke services, and map results/errors to the UI or API. |
| Application services | Orchestrate use cases and transaction boundaries; do not contain persistence details. |
| Domain entities | Protect state and enforce local invariants, such as legal copy and loan transitions. |
| Repositories | Load and persist entities; expose query operations needed by services. |
| Relational database | Persist catalog, copies, members, loans, staff, and audit history with referential integrity. |
| Overdue scheduler and notification adapter | Find overdue active loans and deliver a notice through a replaceable email/in-app adapter. |

## 4. Object-oriented model

### Class diagram

```mermaid
classDiagram
    class BookTitle {
        +String titleId
        +String title
        +String author
        +String isbn
        +String subject
        +updateMetadata(title, author, isbn, subject)
    }

    class BookCopy {
        +String copyId
        +String titleId
        +String barcode
        +String shelfLocation
        -CopyStatus status
        +isAvailable() bool
        +issue()
        +recordReturn()
        +makeAvailable()
        +retire()
    }

    class Member {
        +String memberId
        +String name
        +String email
        +MemberStatus status
        +int loanLimit
        +canBorrow(activeLoanCount) bool
        +updateContact(name, email)
        +deactivate()
    }

    class StaffAccount {
        +String staffId
        +String name
        +String role
        +bool active
        +hasPermission(action) bool
    }

    class Loan {
        +String loanId
        +Date checkedOutOn
        +Date dueOn
        +Date returnedOn
        +LoanStatus status
        +isOverdue(asOfDate) bool
        +close(returnedOn)
    }

    class CatalogService {
        +search(query, filters)
        +addTitle(title)
        +addCopy(titleId, copy)
        +updateTitle(titleId, changes)
        +retireCopy(copyId)
    }

    class MemberService {
        +registerMember(details)
        +updateMember(memberId, changes)
        +deactivateMember(memberId)
    }

    class CirculationService {
        +issueCopy(memberId, copyId, staffId)
        +returnCopy(copyId, staffId)
        +getMemberLoans(memberId)
        +findOverdue(asOfDate)
    }

    class BookTitleRepository {
        +search(query, filters)
        +findById(titleId)
        +save(title)
    }

    class BookCopyRepository {
        +findById(copyId)
        +findByBarcode(barcode)
        +save(copy)
    }

    class MemberRepository {
        +findById(memberId)
        +countActiveLoans(memberId)
        +save(member)
    }

    class LoanRepository {
        +findActiveByCopy(copyId)
        +findActiveByMember(memberId)
        +findOverdue(asOfDate)
        +save(loan)
    }

    BookTitle "1" o-- "0..*" BookCopy : has copies
    Member "1" -- "0..*" Loan : borrows
    BookCopy "1" -- "0..*" Loan : loan history
    CatalogService ..> BookTitleRepository
    CatalogService ..> BookCopyRepository
    MemberService ..> MemberRepository
    CirculationService ..> MemberRepository
    CirculationService ..> BookCopyRepository
    CirculationService ..> LoanRepository
    StaffAccount ..> CatalogService : authorized use
    StaffAccount ..> MemberService : authorized use
    StaffAccount ..> CirculationService : authorized use
```

`BookTitle` stores shared bibliographic information; `BookCopy` represents one physical, independently borrowable item. A `Loan` is the historical association between one member and one copy. Staff authorization is checked at the application boundary and is not delegated to UI visibility.

### Use-case diagram

```mermaid
flowchart LR
    Member((Member))
    Librarian((Librarian))
    Admin((Administrator))
    Scheduler((Scheduler))

    subgraph LMS["Library Management System"]
        Search([Search catalog])
        Availability([View copy availability])
        OwnLoans([View own loans])
        Issue([Issue a copy])
        Return([Return a copy])
        Catalog([Manage titles and copies])
        Members([Manage members])
        Overdue([Review overdue loans])
        Notify([Send overdue notice])
        Staff([Manage staff and settings])
    end

    Member --> Search
    Search --> Availability
    Member --> OwnLoans
    Librarian --> Search
    Librarian --> Issue
    Librarian --> Return
    Librarian --> Catalog
    Librarian --> Members
    Librarian --> Overdue
    Admin --> Staff
    Admin --> Search
    Admin --> Catalog
    Admin --> Members
    Admin --> Issue
    Admin --> Return
    Admin --> Overdue
    Scheduler --> Overdue
    Overdue --> Notify
```

### Sequence: issue a book

```mermaid
sequenceDiagram
    actor Member
    actor Librarian
    participant UI as Staff console
    participant Service as CirculationService
    participant Members as MemberRepository
    participant Copies as BookCopyRepository
    participant Loans as LoanRepository
    participant DB as Database transaction

    Member->>Librarian: Request a copy
    Librarian->>UI: Enter member and copy/barcode
    UI->>Service: issueCopy(memberId, copyId, staffId)
    Service->>Members: findById(memberId)
    Members-->>Service: Member(status, loanLimit)
    Service->>Loans: count active loans for member
    Loans-->>Service: activeLoanCount
    Service->>Copies: findById(copyId)
    Copies-->>Service: BookCopy(status)
    Service->>Service: Validate member, loan limit, and availability
    Service->>DB: Begin transaction
    Service->>Loans: Save active loan with due date
    Service->>Copies: Mark copy issued
    Service->>DB: Commit
    Service-->>UI: Loan confirmation and due date
    UI-->>Librarian: Show confirmation
    Librarian-->>Member: Provide copy and due date

    Note over Service,DB: Any persistence failure rolls back both the loan and copy update.
```

### Sequence: return a book

```mermaid
sequenceDiagram
    actor Member
    actor Librarian
    participant UI as Staff console
    participant Service as CirculationService
    participant Copies as BookCopyRepository
    participant Loans as LoanRepository
    participant DB as Database transaction

    Member->>Librarian: Return a copy
    Librarian->>UI: Scan barcode
    UI->>Service: returnCopy(copyId, staffId)
    Service->>Copies: findById(copyId)
    Copies-->>Service: BookCopy
    Service->>Loans: findActiveByCopy(copyId)
    Loans-->>Service: Active loan or none
    Service->>Service: Reject if no active loan exists
    Service->>DB: Begin transaction
    Service->>Loans: Close loan with returned date
    Service->>Copies: Record return, then make available
    Service->>DB: Commit
    Service-->>UI: Return confirmation
    UI-->>Librarian: Show confirmation
    Librarian-->>Member: Confirm return

    Note over Service,DB: Return date, loan status, and copy status are persisted atomically.
```

## 5. Behavioral model

### Book-copy state diagram

The `Returned` state represents the short return-processing step. After the item is checked in and cleared for circulation, it becomes `Available`. A damaged or missing copy should instead be taken out of circulation; that optional operational state can be added if required.

```mermaid
stateDiagram-v2
    [*] --> Available : copy registered
    Available --> Issued : checkout confirmed
    Issued --> Returned : active loan checked in
    Returned --> Available : return recorded / copy cleared
    Available --> Retired : librarian retires copy
    Returned --> Retired : copy cannot circulate
    Retired --> [*]
```

### Loan lifecycle

```mermaid
stateDiagram-v2
    [*] --> Active : checkout
    Active --> Active : due date passes / overdue is derived
    Active --> Returned : valid return recorded
    Returned --> [*]
```

An overdue loan remains active until returned; overdue is a date-based condition, not a terminal loan state. This avoids separate overdue state updates becoming inconsistent with the due date.

## 6. Data, functional, and behavioral models

### Data model

```mermaid
erDiagram
    BOOK_TITLE ||--o{ BOOK_COPY : has
    MEMBER ||--o{ LOAN : borrows
    BOOK_COPY ||--o{ LOAN : appears_in
    STAFF_ACCOUNT ||--o{ CIRCULATION_AUDIT : performs
    LOAN ||--o{ CIRCULATION_AUDIT : recorded_for

    BOOK_TITLE {
        string title_id PK
        string title
        string author
        string isbn
        string subject
    }
    BOOK_COPY {
        string copy_id PK
        string title_id FK
        string barcode UK
        string shelf_location
        string status
    }
    MEMBER {
        string member_id PK
        string name
        string email UK
        string status
        int loan_limit
        date joined_on
    }
    LOAN {
        string loan_id PK
        string member_id FK
        string copy_id FK
        string checked_out_on
        string due_on
        string returned_on
        string status
        bool overdue_notice_sent
    }
    STAFF_ACCOUNT {
        string staff_id PK
        string name
        string email UK
        string role
        bool active
    }
    CIRCULATION_AUDIT {
        string audit_id PK
        string staff_id FK
        string loan_id FK
        string action
        string occurred_at
    }
```

**Integrity constraints**

- `BOOK_COPY.title_id`, `LOAN.member_id`, and `LOAN.copy_id` reference existing rows.
- `BOOK_COPY.barcode` and account email addresses are unique.
- At most one `LOAN` per copy may have `status = ACTIVE`; enforce with a database constraint/index where supported and a transaction/locking strategy.
- `LOAN.due_on >= LOAN.checked_out_on`; if returned, `returned_on >= checked_out_on`.
- Store dates consistently (date-only for circulation dates, timestamp for audit events).
- Keep loan history when a copy or member is retired/deactivated; use status changes rather than deleting historical records.

### Functional model

```mermaid
flowchart TD
    SearchInput["Search terms and filters"] --> SearchCatalog["Search catalog"]
    SearchCatalog --> Titles["Matching titles"]
    Titles --> Availability["Join copies and availability"]
    Availability --> SearchResults["Titles with available-copy counts"]

    CheckoutInput["Member ID + copy barcode"] --> ValidateCheckout["Validate member, limit, copy"]
    ValidateCheckout -->|valid| CreateLoan["Create active loan"]
    CreateLoan --> SetIssued["Set copy to Issued"]
    SetIssued --> CheckoutResult["Confirmation + due date"]
    ValidateCheckout -->|invalid| CheckoutError["Explain validation failure"]

    ReturnInput["Copy barcode"] --> FindLoan["Find active loan"]
    FindLoan -->|found| CloseLoan["Record returned date"]
    CloseLoan --> SetAvailable["Set copy to Available"]
    SetAvailable --> ReturnResult["Return confirmation"]
    FindLoan -->|not found| ReturnError["Explain no active loan"]
```

### Key operation contracts

| Operation | Input | Success result | Rejection conditions |
| --- | --- | --- | --- |
| `searchCatalog(query, filters)` | Text and optional author/subject/availability filters | Matching titles, copy counts, and availability | Invalid filter format |
| `issueCopy(memberId, copyId, staffId)` | Member, physical copy, authenticated staff | New active loan and due date | Unknown/inactive member, loan limit reached, unknown/unavailable copy, unauthorized staff |
| `returnCopy(copyId, staffId)` | Physical copy and authenticated staff | Closed loan and available copy | Unknown copy, no active loan, unauthorized staff |
| `deactivateMember(memberId)` | Member identifier and authenticated staff | Member status set inactive | Unknown member or active loan exists |
| `findOverdue(asOfDate)` | Evaluation date | Active loans whose due date has passed | Invalid evaluation date |

## 7. Abstraction to implementation

The following JavaScript-style sketch demonstrates encapsulated copy-state transitions. Production checkout and return operations must persist all related changes within one database transaction.

```js
const CopyStatus = Object.freeze({
  AVAILABLE: "AVAILABLE",
  ISSUED: "ISSUED",
  RETURNED: "RETURNED",
  RETIRED: "RETIRED",
});

class BookCopy {
  #status = CopyStatus.AVAILABLE;

  constructor({ copyId, titleId, barcode, shelfLocation }) {
    if (!copyId || !titleId || !barcode) {
      throw new TypeError("copyId, titleId, and barcode are required");
    }

    this.copyId = copyId;
    this.titleId = titleId;
    this.barcode = barcode;
    this.shelfLocation = shelfLocation;
  }

  get status() {
    return this.#status;
  }

  isAvailable() {
    return this.#status === CopyStatus.AVAILABLE;
  }

  issue() {
    if (!this.isAvailable()) {
      throw new Error("Only an available copy can be issued");
    }
    this.#status = CopyStatus.ISSUED;
  }

  recordReturn() {
    if (this.#status !== CopyStatus.ISSUED) {
      throw new Error("Only an issued copy can be returned");
    }
    this.#status = CopyStatus.RETURNED;
  }

  makeAvailable() {
    if (this.#status !== CopyStatus.RETURNED) {
      throw new Error("Only a returned copy can become available");
    }
    this.#status = CopyStatus.AVAILABLE;
  }

  retire() {
    if (this.#status === CopyStatus.ISSUED) {
      throw new Error("An issued copy cannot be retired");
    }
    this.#status = CopyStatus.RETIRED;
  }
}
```

Application-service pseudocode for checkout:

```text
issueCopy(memberId, copyId, staffId):
    require staffId has permission "issue"
    begin database transaction
    member = members.findByIdForUpdate(memberId)
    copy = copies.findByIdForUpdate(copyId)
    activeCount = loans.countActiveForMember(memberId)
    require member exists and member.status is ACTIVE
    require member.canBorrow(activeCount)
    require copy exists and copy.isAvailable()
    loan = Loan(member, copy, today, today + configuredLoanPeriod)
    copy.issue()
    loans.save(loan)
    copies.save(copy)
    commit transaction
    return loan
```

The corresponding `returnCopy` operation should lock/load the copy and its active loan, record the return date, transition the copy through `Issued -> Returned -> Available`, save both changes, and commit. If any validation or persistence step fails, roll back and surface an actionable error to the librarian.

## 8. Quality and security considerations

- Enforce role checks in application services or controllers, never only in the user interface.
- Use parameterized database queries and validate all user-supplied identifiers, search filters, dates, and member details.
- Use database transactions and row-level locking or equivalent safeguards to prevent two concurrent checkouts of the same copy.
- Record staff circulation actions with timestamps for operational traceability.
- Return only the member's own personal and loan data unless a staff role authorizes broader access.
- Provide clear errors for unavailable copies, loan-limit violations, duplicate barcodes, and failed transactions.
- Keep loan duration and borrowing limits as explicit configuration values rather than scattering constants through the code.

# advdb-activity1-Group5 — Gym & Fitness Centre Management System

Advanced Database, Activity 1: Logical Database Design (Group 5). This is a logical data model for a local gym and fitness centre. The gym needs to manage its **members, trainers, classes and payments**.

**Team:** Hidalgo Altesse Imena, Noah Jobse, Melody Shih

![Gym & Fitness Centre EERD](images/EERD_Group5.png)

## Repository contents

| File | Description |
|---|---|
| [`images/EERD_Group5.png`](images/EERD_Group5.png) | Screenshot of the completed EERD |
| [`docs/Gym_DB_Design_Report_Group5.pdf`](docs/Gym_DB_Design_Report_Group5.pdf) | Report with the design process, business rules, normalization, EERD, **data dictionary** and **team contribution table** |
| [`docs/Gym_DB_Design_Report_Group5.html`](docs/Gym_DB_Design_Report_Group5.html) | Source of the PDF |
| `README.md` | Business rules, design and rationale (this file) |

## Modelling tool

The EERD was created with **Graphviz**. It uses crow's-foot notation plus EER extensions: an ISA triangle for specialization, double-bordered boxes for weak/associative entities, and a derived attribute. The documentation was written in **Markdown**.

## Design process

We started from the scenario's core operations: managing members, trainers, classes and payments. We picked out the nouns that needed to store data as candidate entities. Then we walked through the questions the gym needs to answer every day:

- Who can lead this class?
- What times does it run?
- Who is enrolled?
- Has this member paid?

Those questions showed us the relationships and constraints between entities. Each attribute went to the entity it functionally depends on. We chose single-column surrogate primary keys for stability and added foreign keys wherever a relationship needed to be enforced. We then normalized the model to 3NF and drew the final logical model as an EERD.

## Entities

**Main entities:** Member, Trainer (specialized into PersonalTrainer and GroupInstructor), Class, Payment

**Additional supporting entities:**

| Entity | Purpose |
|---|---|
| MembershipType | Plan pricing and duration, stored once per plan |
| Room | Facility used to host a class session |
| ClassSchedule | A recurring weekly day/time/room a class runs (**weak entity**, owned by Class) |
| Enrollment | **Associative entity** resolving the Member–ClassSchedule many-to-many relationship |
| PaymentMethod | Lookup entity separated from Payment to satisfy 3NF |

## Business rules

1. Every member must belong to **exactly one** membership type at a time. A membership type can be held by **many** members. Member participation is total: a member cannot exist without a plan.
2. A trainer can lead **many** classes, but each class is led by **exactly one** trainer. Trainers are specialized as either a Personal Trainer or a Group Instructor. The specialization is **disjoint and partial**: a trainer doesn't have to belong to a subtype (e.g. a newly hired trainer awaiting certification).
3. A class can run as **many** separate weekly scheduled sessions (different days, times and rooms). Each schedule entry belongs to **exactly one** class and cannot exist without it, which makes ClassSchedule a weak entity identified by (ClassID, ScheduleID).
4. A member can enroll in **many** class schedules, and a class schedule can hold **many** enrolled members. The Enrollment associative entity resolves this relationship and records the enrollment date and attendance status.
5. Each payment is made by **exactly one** member using **exactly one** payment method. A member can make **many** payments over time, and a payment method can be reused across **many** payments.
6. A room can host **many** class schedules over time, but each class schedule takes place in **exactly one** room at one time slot. The application layer prevents double-booking a room for overlapping times.

## Attributes and keys

PK = primary key (underlined in the EERD), FK = foreign key.

| Entity | Primary key | Foreign keys | Other attributes |
|---|---|---|---|
| MembershipType | MembershipTypeID | — | TypeName, DurationMonths, Price, Description |
| Member | MemberID | MembershipTypeID → MembershipType | FirstName, LastName, Email, Phone, DateOfBirth, JoinDate, *Age (derived)* |
| Trainer | TrainerID | — | FirstName, LastName, Email, Phone, HireDate, TrainerType (discriminator) |
| PersonalTrainer | TrainerID | TrainerID → Trainer | HourlyRate, MaxClientsPerWeek |
| GroupInstructor | TrainerID | TrainerID → Trainer | CertificationLevel, GroupSizeLimit |
| Class | ClassID | TrainerID → Trainer | ClassName, Description, Category, DurationMinutes |
| Room | RoomID | — | RoomName, Location, Capacity |
| ClassSchedule (weak) | ScheduleID; full identifier (ClassID, ScheduleID) | ClassID → Class (partial key, identifying); RoomID → Room | DayOfWeek, StartTime, EndTime |
| Enrollment | (MemberID, ScheduleID) | MemberID → Member; ScheduleID → ClassSchedule | EnrollmentDate, AttendanceStatus |
| PaymentMethod | PaymentMethodID | — | MethodName, ProcessorName, ProcessingFeePercent |
| Payment | PaymentID | MemberID → Member; PaymentMethodID → PaymentMethod | Amount, PaymentDate, Notes |

The data types, definitions and validation rules for every attribute are in the [data dictionary (report section 5)](docs/Gym_DB_Design_Report_Group5.pdf).

## Relationships, cardinality and participation

| Relationship | Type | Participation |
|---|---|---|
| Member *has* MembershipType | M:1 | Total on Member (every member has a plan); partial on MembershipType |
| Member *makes* Payment | 1:M | Total on Payment (every payment belongs to a member); partial on Member |
| PaymentMethod *used for* Payment | 1:M | Total on Payment; partial on PaymentMethod |
| Trainer *leads* Class | 1:M | Total on Class (exactly one trainer); partial on Trainer |
| Class *scheduled as* ClassSchedule | 1:M, **identifying** | Total on ClassSchedule (cannot exist without its class) |
| Room *hosts* ClassSchedule | 1:M | Total on ClassSchedule (exactly one room); partial on Room |
| Member *enrolls in* ClassSchedule | **M:N**, resolved by Enrollment (Member 1:M Enrollment, ClassSchedule 1:M Enrollment) | Partial on Member and ClassSchedule; total on Enrollment |
| Trainer **ISA** PersonalTrainer / GroupInstructor | Specialization | **Disjoint, partial** (TrainerType is the discriminator and may be null) |

### EER features

- **Specialization:** Trainer is the supertype of PersonalTrainer and GroupInstructor. The specialization is disjoint and partial. Subtypes share the supertype's key (TrainerID is both PK and FK).
- **Weak entity:** ClassSchedule is existence-dependent on Class through an identifying relationship. If a class is removed, its schedule entries are removed with it.
- **Associative entity:** Enrollment resolves the M:N relationship between Member and ClassSchedule and carries its own data.
- **Derived attribute:** Member.Age is calculated from DateOfBirth at query time, not stored.

## Normalization to 3NF: key decisions

- **Repeating class-session columns removed (1NF).** An early draft stored each class's sessions as repeating columns on Class (Day1/Time1, Day2/Time2 …). We moved sessions into a separate ClassSchedule entity with one row per weekly session.
- **MembershipType separated from Member (3NF).** Price and DurationMonths depend on the plan, not the member (MemberID → MembershipTypeID → Price). Keeping them on Member was a transitive dependency and caused an update anomaly, because changing a plan's price meant updating every member on that plan.
- **Payment method details separated from Payment (3NF).** ProcessorName and ProcessingFeePercent depend on the payment method, not the individual transaction. We moved them into a PaymentMethod lookup entity referenced by PaymentMethodID.
- **M:N resolved with an associative entity (2NF).** Enrollment has a composite key (MemberID, ScheduleID). Its attributes (EnrollmentDate, AttendanceStatus) depend on the whole key, so there are no partial dependencies.
- **Room kept independent of ClassSchedule.** RoomName, Location and Capacity depend only on the room, so they aren't repeated for every session held there.
- **ClassSchedule identifier.** ClassSchedule is weak because every row depends on its owning Class (ClassID). ScheduleID is generated system-wide, so it is also unique on its own, and Enrollment can reference a session with one column.
- **EndTime stored deliberately.** Class.DurationMinutes is the class's standard length. EndTime is the scheduled end of one specific session and can differ from it (e.g. an extended session). EndTime is therefore not determined by other attributes and does not create a transitive dependency.

After these changes, every non-key attribute depends on the key, the whole key and nothing but the key, so the model is in **3NF**.

## Team contributions

| Team member | Role | Specific tasks |
|---|---|---|
| Hidalgo Altesse Imena | Database Designer / EERD Lead | Identified entities and business rules; defined attributes, keys and relationships; carried out 3NF normalization; built the EERD; assembled the GitHub repository and report |
| Noah Jobse | Documentation & Business Rules | Drafted the business rules; wrote and reviewed the README design rationale; reviewed relationship cardinalities and participation |
| Melody Shih | Data Dictionary & QA | Compiled the data dictionary; validated attribute data types and validation rules; proofread the final report |

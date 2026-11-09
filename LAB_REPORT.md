# Lab Report, Designing the Data Management Layer with H2

Repository URL: https://github.com/Yumon-lab/itcs382-h2-lab

## Tools

| Item | Version or name |
| --- | --- |
| JDK | 25.0.4.1 |
| Maven | 3.10.0 |
| IDE | 1.143.1 |
| AI tool and exact model, if you used one | ChatGPT GPT-6 |

## Part 0. From a store in memory to H2

Commit of the starter, before any change: 3fb5044

0.3 Your prediction for `staff`, written before the second run:
P1001, S1001, SMALL, STORED, locker 1
P1002, S1002, LARGE, STORED, locker 3
Free lockers: 1

Output of the second run, for `staff` and `quit`:

```text
Free lockers: 3
```

State in one sentence where you think the parcels of the first run were kept.
```text
The parcels maybe stored somewhere in the program which is connected to the store commands, what is important is it is not saved and thus when we run the program against, the data is gone and all the lockers is free. 
```

0.5 The line of Main that you would change to keep the parcels somewhere else:

```java
ParcelService service = new ParcelService(repository);
```

0.6 Commit of the switch to H2:94bc8ec

0.7 Output of `SELECT * FROM PARCEL;` after you stored two parcels:

```text
sql> SELECT * FROM PARCEL;
CODE  | STUDENT_ID | PARCEL_SIZE | STATUS | LOCKER_NUMBER
P1001 | S1001      | SMALL       | STORED | 1
P1002 | S1002      | LARGE       | STORED | 3
(2 rows, 20 ms)
sql> 
```

0.8 Explain why ParcelPoint could not start while the H2 Shell was open, and identify the class that wrote the message `Cannot create the PARCEL table`.

ParcelPoint can't start because the database belong to the program that open it and that class is public H2ParcelRepository(String path)


0.9 Explain why `staff` shows no parcel while `find` shows each one, and why all three lockers look free.

From some inspection, it can be assume that findbycode Method from H2 is taken directly from database but if look at Parcel Service, it does not seem to take the database number into account so it ended up using empty database instead, resulting in all 3 lockers being free.

0.11 Your findAll method:

```java
 @Override
    public List<Parcel> findAll() {
        String sql = "SELECT * FROM PARCEL ORDER BY CODE";
        List<Parcel> parcels = new ArrayList<>();
        try (Connection connection = connect();
             PreparedStatement statement = connection.prepareStatement(sql)) {
            try (ResultSet row = statement.executeQuery()) {
               while (row.next()){
                 parcels.add(toParcel(row));
               }
            }
        } catch (SQLException e) {
            throw new IllegalStateException("Cannot read the parcels", e);
        }
        return parcels;
    }
```

Commit that adds findAll:dda4456

0.12 Output of `SELECT * FROM PARCEL;` after the pickup:

```text
sql> SELECT * FROM PARCEL;
CODE  | STUDENT_ID | PARCEL_SIZE | STATUS    | LOCKER_NUMBER
P1001 | S1001      | SMALL       | PICKED_UP | null
P1002 | S1002      | LARGE       | STORED    | 3
(2 rows, 31 ms)
sql> 
```

0.13 Optional. Describe what `find P9999'OR'1'='1` showed while the SQL was joined from strings, and what it showed after you put the placeholder back.

when SQL was joined from strings:
 find <code>
  staff
  quit
> find P9999
No parcel with code P9999
> find P9999'OR'1'='1
No parcel with code P9999'OR'1'='1
> find P9999'OR'1'='1'

When SQL return to normal
> find P9999'OR'1'='1'
No parcel with code P9999'OR'1'='1'

Apprarently, without placeholder, the program is vulnerable to SQL injection but in this case can't find the parcel or breach it. 

## Part A. Model the design

A1 Complete the flowchart with one node for each package and one arrow for each dependency, from the package that uses to the package that is used. Mark the package that accesses the data store.

```mermaid
flowchart TB
    parcelpoint
    cli
    service
    domain
    storage
    cli --> service
    cli --> domain
    service --> domain
    service --> storage
    storage --> domain
    parcelpoint --> domain
    parcelpoint --> service
    parcelpoint --> storage
```

State whether cli depends on storage, and explain why.
```
cli depend on storage indirectly as Cli need to access service to reach storage containing database. Should service be gone, Cli may need to reach the storage directly.

Package that accesses the data store
storage["storage, accesses the data store"]
domain["domain, create template for data value"]
```

A2 Complete the class diagram of the data management layer with ParcelService, the interface ParcelRepository with its three operations, and the two classes that implement it.

```mermaid
classDiagram
    class ParcelService{
        -lockers: List<Locker>
        -repository: ParcelRepository
        +ParcelService(repository: ParcelRepository)
        +store(parcel: Parcel): Optional<Locker>
        +pickUp(code: String, days: int): Optional<Double>
        +find(code: String): Optional<Parcel>
        +allParcels(): List<Parcel>
        +countFreeLockers(): long
        -lockerNumbered(number: int): Locker
    }
    class ParcelRepository {
        <<interface>>
        +save(parcel: Parcel): void
        +findByCode(code: String): Optional<Parcel>
        +findAll(): List<Parcel>
    }
    class InMemoryParcelRepository{
        -parcels: Map<String, Parcel>
        +save(parcel: Parcel): void
        +findByCode(code: String): Optional<Parcel>
        +findAll(): List<Parcel>
    }
    class H2ParcelRepository{
        -CREATE_TABLE: String
        -url: String
        +H2ParcelRepository(path: String)
        +save(parcel: Parcel): void
        +findByCode(code: String): Optional<Parcel>
        +findAll(): List<Parcel>
        -connect(): Connection
        -toParcel(row: ResultSet): Parcel
    }
    ParcelService --> ParcelRepository
    ParcelRepository <|.. InMemoryParcelRepository
    ParcelRepository <|.. H2ParcelRepository

```

A3 Draw a sequence diagram for the command `store P1003 S1003 MEDIUM`, from the staff member to the repository. Show each message as an operation of the class that receives it.

```mermaid
sequenceDiagram
    actor Staff
    participant cli as :ParcelCommandLine
    participant service as :ParcelService
    participant parcel as parcel:Parcel
    participant locker as locker:Locker
    participant repository as :ParcelRepository

    Staff->>cli: store P1003 S1003 MEDIUM
    cli->>service: store(parcel)

    loop For each locker until one fits
        service->>locker: fits(parcel)
        locker-->>service: fits: boolean
    end

    service->>locker: setParcel(parcel)
    service->>parcel: storeIn(2)
    service->>repository: save(parcel)
```

Explain why the last lifeline is the interface ParcelRepository rather than H2ParcelRepository.
```
This is because ParcelRepository is an abstract class and an interface mean to be used for communicating with the repository through the interface. also H2ParcelRepository and InMemoryParcelRepository depends on this interface so ParcelService can save parcels without knowing which repository implementation is being used.
```

## Part B. Design questions

1. Identify the files that you changed to move from the store in memory to H2, and explain why no other class had to change. Use the terms of this course.
```
Main.java and H2ParcelRepository.java. Other class take care of their respective function but these 2 files are the one responsible for interacting with the user directly and receive and manipulate data.

```

2. The PARCEL table stores the size of a parcel as SMALL, MEDIUM or LARGE rather than as 1, 2 or 3. Identify the type of connascence between ParcelSize and the table in each case, and explain which one is weaker.
```
Connascene of Name is storing the size as SMALL, MEDIUM or LARGE 
Connascene of Value is storing the value as number 1,2,3
From this, connascene of name is weaker because it tell the user directly of the size by just read the word SMALL, MEDIUM and Large.
However, if it is number, it usually mean the user has to know the mapping between number and parcel sizes which can be confusing.

```
3. Project Phase 2 asks for a data store that needs no separate server. Explain why an embedded database such as H2 meets this rule, and describe the limit of an embedded database that Step 0.8 showed.
```
Because this is a rather small and closed system, it does not need to start a server like application as that would make this too complicated so embedded database or H2 is a more simpler and easier approach for this project but it also has a limit as the database belong to program that run it, ordinary embedded access to a file-based H2 database can cause problems when multiple application instances attempt to open the same database simultaneously.
```

4. State the data store that you would choose for your Project Phase 2 team, files or an embedded database, the requirement that drives the choice, and the package that will access it.
```
I would choose embedded database as from this lab, I am more familiar with how to work on with H2 but this is tentative as it depends on the requirment of the project phase 2. The package that will access it will be parcelpoint.storage as seen from this lab, it has H2ParcelRepository which handles database connection and SQL.
```

## AI usage declaration

| AI tool and exact model | Date | Purpose | Files or answers it produced or changed | How you checked its output |
| --- | --- | --- | --- | --- |
|ChatGPT, GPT-6 is the AI used| 11/10/2026| To help me explain and work on instruction of slides and some part of code when I am lost | H2ParcelRepository.java, it help in production of findall() method in giving me hint on how to implement it correctly| I check its output by doing the lab instruction and test it whether it follow what the slide want|
| | | | | |

# Lab Report, Designing the Data Management Layer with H2

Repository URL: https://github.com/Yumon-lab/itcs382-h2-lab

## Tools

| Item | Version or name |
| --- | --- |
| JDK | 25.0.4.1 |
| Maven | 3.10.0 |
| IDE | 1.143.1 |
| AI tool and exact model, if you used one | |

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

0.6 Commit of the switch to H2:
import parcelpoint.storage.H2ParcelRepository;
ParcelRepository repository = new H2ParcelRepository("./data/parcelpoint");

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

Commit that adds findAll:

0.12 Output of `SELECT * FROM PARCEL;` after the pickup:

```text

```

0.13 Optional. Describe what `find P9999'OR'1'='1` showed while the SQL was joined from strings, and what it showed after you put the placeholder back.

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
```

State whether cli depends on storage, and explain why.

A2 Complete the class diagram of the data management layer with ParcelService, the interface ParcelRepository with its three operations, and the two classes that implement it.

```mermaid
classDiagram
    class ParcelService
    class ParcelRepository {
        <<interface>>
    }
```

A3 Draw a sequence diagram for the command `store P1003 S1003 MEDIUM`, from the staff member to the repository. Show each message as an operation of the class that receives it.

```mermaid
sequenceDiagram
    actor Staff
    participant cli as :ParcelCommandLine
    Staff->>cli: store P1003 S1003 MEDIUM
```

Explain why the last lifeline is the interface ParcelRepository rather than H2ParcelRepository.

## Part B. Design questions

1. Identify the files that you changed to move from the store in memory to H2, and explain why no other class had to change. Use the terms of this course.

2. The PARCEL table stores the size of a parcel as SMALL, MEDIUM or LARGE rather than as 1, 2 or 3. Identify the type of connascence between ParcelSize and the table in each case, and explain which one is weaker.

3. Project Phase 2 asks for a data store that needs no separate server. Explain why an embedded database such as H2 meets this rule, and describe the limit of an embedded database that Step 0.8 showed.

4. State the data store that you would choose for your Project Phase 2 team, files or an embedded database, the requirement that drives the choice, and the package that will access it.

## AI usage declaration

| AI tool and exact model | Date | Purpose | Files or answers it produced or changed | How you checked its output |
| --- | --- | --- | --- | --- |
| | | | | |

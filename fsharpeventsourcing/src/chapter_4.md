# In-Memory Projections and Detail Views

When modeling relationships between Aggregates in Event Sourced systems, developers often face the challenge of querying and associating data spread across multiple streams.

## The Complexity of Mutual References

It is tempting to model bidirectional (mutual) references between aggregates to easily navigate relationships (e.g., a `Student` holds a list of `Course` IDs, and a `Course` holds a list of `Student` IDs). 

However, **bidirectional references should be avoided.**

They introduce significant complexity because they potentially involve updating more objects at once. Crucially, they heavily increase the complexity when an object supports deletion, requiring intricate reference counting to ensure safe deletion without leaving orphaned or broken links.

## The Unidirectional Approach

Sharpino advocates for a **unidirectional design**. Instead of mutual links, you designate a single aggregate as the source of truth for the relationship. 

## Detail Views (Materialized Projections)

To compensate for the lack of bidirectional links when querying, Sharpino utilizes **Detail Views** (in-memory materialized projections). 

A Detail View listens to the events of multiple aggregates and continuously reconstructs the composite relationships in memory. This allows the application (and the UI) to easily query composite objects and navigate relations in any direction without tangling the underlying write-side domain models.

```mermaid
graph TD
    subgraph Bidirectional["Bidirectional (Avoid)"]
        A[Student Aggregate] <--> B[Course Aggregate]
        style A stroke:#f66,stroke-width:2px;
        style B stroke:#f66,stroke-width:2px;
    end

    subgraph Unidirectional["Unidirectional (Recommended)"]
        C[Enrollment Aggregate] --> D[Student Aggregate]
        C --> E[Course Aggregate]
        
        F[Student Courses Detail] -. "Materialized View" .-> C
    end
```

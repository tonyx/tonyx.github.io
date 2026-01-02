# Application service layer

An application service layer implements the logic and makes business logic calls, or queries, available.
Here is one of the simplest examples of an entry for a service involving a single context, by building and running an AddTag command.

Initializing the application service layer means providing the correct "viewers" for the streams (which can be based on the event store, mediated by the cache, or can be based on an event broker listener, as in the RabbitMq provided examples) plus some compound viewers if needed (i.e. to read the state of all instances of an aggregate type) and a "messageSenders" object that can be passed to the command handler to let it send messages to the event broker (if any) in a fire and forget way:

```FSharp

    type CourseManager
        (
            eventStore: IEventStore<string>,
            courseViewer: AggregateViewer<Course>,
            studentViewer: AggregateViewer<Student>,
            messageSenders: MessageSenders,
            allStudentsAggregateStatesViewer: unit -> Result<(Definitions.EventId * Student) list, string>
        )

```

A member of the service to provide the initialization of an aggregate:
```FSharp
        member this.AddStudent (student: Student) =
            result
                {
                    return!
                        runInit<Student, StudentEvents, string>
                        eventStore
                        messageSenders
                        student
                }
```

A member of the service to retrieve a student via its id

```FSharp

        member this.GetStudent (id: StudentId)  =
            result
                {
                    let! _, student = studentViewer id.Id
                    return student
                }
```
Note: by using the typed Id, we need to extract the proper Guid Id via the Id property if using the studentViewer.

An example using two commands related to two streams. The example assume that we prefer that the enrollments are stored in both the Student aggregate and the Course aggregate (i.e. bidirectional references). So we need to run two commands in a single transaction to preserve consistency.:

```fsharp
        member this.EnrollStudentToCourse (studentId: StudentId) (courseId: CourseId) =
            result
                {
                    let addCourseToStudentEnrollments = StudentCommands.Enroll courseId
                    let addStudentToCourseEnrollments = CourseCommands.EnrollStudent studentId
                    return!
                        runTwoAggregateCommands
                            studentId.Id
                            courseId.Id
                            eventStore
                            messageSenders
                            addCourseToStudentEnrollments
                            addStudentToCourseEnrollments
                }
```        

In the following example, the enrollment is a single object type that register enrollments in a single stream (a more conventional approach):
```fsharp
        member this.CreateEnrollment (studentId: StudentId) (courseId: CourseId) =
            result {
                let! enrollments = this.GetOrCreateEnrollments()
                do! 
                    enrollments.Enrollments
                    |> List.exists (fun e -> e.StudentId = studentId && e.CourseId = courseId)
                    |> not
                    |> Result.ofBool "Student is already enrolled in this course."

                let enrollmentItem = 
                    { CourseId = courseId
                      StudentId = studentId
                      EnrollmentDate = DateTime.UtcNow }
                    
                let studentDetailsKey =  DetailsCacheKey (typeof<RefreshableStudentDetails>, studentId.Id)
                let _ =
                    DetailsCache.Instance.UpdateMultipleAggregateIdAssociationRef [|courseId.Id|] studentDetailsKey ((TimeSpan.FromMinutes 10.0) |> Some)
                    
                let command = EnrollmentCommands.AddEnrollment enrollmentItem
                let! result = 
                    runAggregateCommand<Enrollments, EnrollmentEvents, string>
                        enrollmentId.Id
                        eventStore
                        messageSenders
                        command
                return result
            }

```
Note: the examples show the complexity of handling cache of the details. This aspect wil be covered up and hopefully simplified in future version of this documentation. Just note that any existing RefreshableStudentDetails object need to be notified that a new enrollment is created so that it can refresh its state when needed.

The quoted code is related to example 15.
















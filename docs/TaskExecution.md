# Task Execution
## Background
By nature fastlane.ci is a task executor service. It has a collection of `projects` that contain rules for when to run and what to run. Task executor services have unique synchronization challenges, here are just a few:

- One project can access the same resource two times:
  - e.g.: two builds kicked off at the same time, either manually, or through a trigger
- Multiple projects can push, pull, and merge from the same git repository
- Multiple users can modify configuration data that is stored locally
- Multiple users can push to remote configuration repos
- Multiple projects can write status and comments to a single GitHub project

### How fastlane.ci executes tasks
Most tasks are handled by individual threads that are spawned and then run on a schedule, or are triggered manually. This can lead to overlapping operations on the same resource. This can also lead to accidentally spawning too many threads and having a negative impact on the process.

## Proposed solution

### Requirements
At the highest level, the solution must:

- guarantee that no two tasks are going to attempt to modify the same resource at the same time.
- simplify how we handle tasks: people don't need to think about threads
- it must be scalable and easy to implement
- be general enough that it can apply to any task

### Solution
Utilize a [GCD-like](https://developer.apple.com/documentation/dispatch) mechanism to provide a TaskQueue with a configurable number of workers. Currently implemented in [TaskQueue project](https://github.com/fastlane/TaskQueue). The typical GCD workflow consists of submitting blocks of works to a queue with either 1 single worker -which guarantees first in first out execution- or a configurable amount of workers, which constrains the maximum number of tasks that can be executed at once. There are many more use cases for having a work queue, but those two are the most common. In our implementation, we expect:

- Each repo to be associated with a `serial` `TaskQueue`, so that any task needing to execute can be guaranteed to be executed in the correct order and only after the preceeding task as completed
- Any work that needs to take place on that repo can be submitted to the queue
- Every datasource that reads/writes to a file can be associated with a filewriter/reader `serial` `TaskQueue`
- Use a timer to submit a `task` to a concurrent queue with a set limit of workers. Before submitting another `task` the timer can check if the previous has been completed

#WIP

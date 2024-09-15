---
title: "Solving Concurrency Issues with PostgreSQL Advisory Locks"
description:
  "A practical approach to handling race conditions in database operations using
  PostgreSQL's built-in locking mechanism"
pubDate: "2024-09-15"
---

In my daily work, I encountered a common challenge in distributed systems: race
conditions during concurrent database operations. One specific instance of this
issue involved generating events, where simultaneous requests could lead to data
inconsistencies.

Here's a simplified version of my initial implementation that was susceptible to
race conditions:

```tsx
export module EventService {
  type ListParams = {
    schedule_id: string;
    start_date: string;
    end_date: string;
  };

  export async function list(params: ListParams) {
    // Fetch schedule from which events are generated
    const schedule = await ScheduleRepo.get({
      schedule_id: params.schedule_id,
    });

    // Generate events if not already generated
    if (schedule.generated_to < params.end_date) {
      // Call the function defined below
      await generate({
        schedule,
        generate_to: params.end_date,
      });
    }

    // List the events if they are already generated
    await EventRepo.list(params);
  }

  export async function generate(/* … */) {
    /* Generate events in the transaction */
  }
}
```

## Using PostgreSQL Advisory Locks

### Why PostgreSQL Advisory Locks?

When faced with this concurrency issue, I wanted to avoid introducing a new
service or complicating my architecture just to handle this problem.

I discovered that PostgreSQL, my database of choice, offers a built-in locking
mechanism I could leverage. This discovery was ideal because it allowed me to
solve the problem using a tool I already had in place, without adding any new
dependencies or services to the stack. PostgreSQL provides various
[locking functions](https://www.postgresql.org/docs/9.1/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS).

### Understanding pg_try_advisory_xact_lock

For my specific use case, the `pg_try_advisory_xact_lock` function proved to be
the most suitable:

> `pg_advisory_lock` locks an application-defined resource, which can be
> identified either by a single 64-bit key value or two 32-bit key values (note
> that these two key spaces do not overlap). If another session already holds a
> lock on the same resource identifier, this function will wait until the
> resource becomes available. The lock is exclusive. Multiple lock requests
> stack, so that if the same resource is locked three times it must then be
> unlocked three times to be released for other sessions' use.
>
> […]
>
> `pg_try_advisory_lock` is similar to `pg_advisory_lock`, except the function
> will not wait for the lock to become available. It will either obtain the lock
> immediately and return true, or return false if the lock cannot be acquired
> immediately.
>
> […]
>
> `pg_try_advisory_xact_lock` works the same as `pg_try_advisory_lock`, except
> the lock, if acquired, is automatically released at the end of the current
> transaction and cannot be released explicitly.

This function allows me to:

1. Acquire a lock using a specific key
2. Determine if a lock is already acquired, and exit the transaction early if it
   is
3. Automatically release the lock at the end of the transaction

Here's an example of using the function in a transaction:

```sql
START TRANSACTION;

-- Returns true if the lock was acquired
-- After the transaction ends, the lock is released
-- We can throw an error if the lock was not acquired, to quit the transaction
SELECT pg_try_advisory_xact_lock(1234);
```

## UUID to BigInt Conversion

One problem I encountered was that `pg_try_advisory_xact_lock` needs a bigint as
a key, but I use UUIDs which are too large to fit into a bigint. To solve this,
I devised a way to hash the UUID and use the first 16 bytes as the key. Here's
what my SQL looks like:

```sql
SELECT pg_try_advisory_xact_lock(
  ('x' || substr(md5(/** uuid here */), 1, 16))::bit(64)::bigint
);
```

While this solution effectively addresses my immediate problem, it's important
to consider potential drawbacks, such as the theoretical possibility of hash
collisions. Despite this minor concern, for my use case, this approach strikes a
good balance between solving the concurrency issue and maintaining simplicity in
my codebase.

## Implementation

I use [Kysely](https://kysely.dev/) to generate the SQL and execute it in a
transaction. Here's how I implemented the locking mechanism:

First, I created a utility function that takes a UUID and the Kysely transaction
and tries to acquire a lock for the given UUID.

```tsx
import { Transaction, sql } from "kysely";

export async function getPostgresLockForUuid(
  uuid: string,
  trx: Transaction<DB>,
): Promise<{ lock_acquired: boolean }> {
  // Creates a lock for the given UUID
  const result = await sql<{
    pg_try_advisory_xact_lock: boolean;
  }>`SELECT pg_try_advisory_xact_lock(
      ('x' || substr(md5(${uuid}), 1, 16))::bit(64)::bigint
  )`.execute(trx);

  // Default to locked to catch any issues
  const lock_acquired = result.rows[0]?.pg_try_advisory_xact_lock ?? false;

  return { lock_acquired };
}
```

I then updated the `EventService` to utilize this new function. Instead of
waiting for the lock to be released and retrying, I throw an error if the lock
cannot be acquired. This approach works well with my API client's automatic
retry mechanism.

```tsx
export module EventService {
  // […]

  export async function generate(params: {
    /* … */
  }) {
    await kysely.transaction.execute(async (trx) => {
      const { lock_acquired } = await getPostgresLockForUuid(
        params.schedule.id,
        trx,
      );

      if (!lock_acquired) {
        throw new Error("Could not acquire lock");
      }

      // … Code for generating events
    });
  }
}
```

## Conclusion

In this post, I shared how I solved a tricky problem with database operations
occurring simultaneously. By using PostgreSQL's advisory locks, I found a simple
way to prevent race conditions without complicating my system.

While this approach worked well for my specific case, it's important to remember
that every situation is different. Always consider your unique requirements and
potential trade-offs when implementing a solution like this.

If you're facing similar challenges with concurrent operations in your database,
consider giving PostgreSQL advisory locks a try. They might just be the
straightforward solution you're looking for!

Happy coding!

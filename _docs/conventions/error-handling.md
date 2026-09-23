# Error Handling

Unexpected errors must fail loudly and early. A health daemon monitors PHP's error log and alerts devs, while daemon and application-specific logs are generally inspected only during development or after exceptional occurrences. An unexpected error that is merely written to an application logger can therefore remain unnoticed for a long time.

## General rules

- Do not catch `Throwable`, `Error`, or a broad `Exception` only to write to an application log and continue.
- Syntax errors and other programming errors must terminate the affected execution and propagate to PHP's monitored error log.
- Catch only errors that are an expected part of the operation and that the caller can handle meaningfully.
- Prefer specific exception types for expected failures, such as the integration temporary- and permanent-failure exceptions.
- A catch used for mandatory cleanup, such as rolling back a transaction, must rethrow the original exception immediately.
- At HTTP and form boundaries, translate only known validation or domain exceptions into user-facing responses. Unexpected exceptions should remain visible to PHP error monitoring.
- In some cases, where a user is actively doing something, it may be best to catch and display errors directly, but be cautious as most end users of the system are non-technical. We also want to limit accidental exposure of inner workings to end users.

Expected operational failures should be recorded in the authoritative domain data when relevant. Application logs may provide supplementary development context, but they must not be the only durable indication of a failure that requires attention.

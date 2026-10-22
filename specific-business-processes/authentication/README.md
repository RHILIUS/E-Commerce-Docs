# Authentication specific business processes

## Existing diagrams

| SBP                                           | Purpose                                                                                                                                 |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| [Login](login.drawio.svg)                     | Implemented email/password login, validation and retry, browser refresh-cookie storage, in-memory session state, and return navigation. |
| [Signup](signup.drawio.svg)                   | Creating an account through the registration flow.                                                                                      |
| [Logout](logout.drawio.svg)                   | Implemented local sign-out, refresh-token cleanup, browser cookie deletion, failure handling, and route-aware navigation.               |
| [JWT interceptor](jwt-interceptor.drawio.svg) | Handling access-token checks and refresh during API requests.                                                                           |

Return to the [SBP index](../README.md) for other modules and maintenance conventions.

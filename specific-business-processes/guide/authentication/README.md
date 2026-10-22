# Authentication specific business processes

Start with a guide to understand the customer journey, then follow its diagram.
The guides distinguish current behavior from older diagram details.

## Existing coverage

| Process         | Guide                             | Diagram                                                               | Purpose                                                                          |
| --------------- | --------------------------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Login           | [Explanation](login.md)           | [Flowchart](../../diagrams/authentication/login.drawio.svg)           | Email/password checks, browser session setup and return navigation.              |
| Signup          | [Explanation](signup.md)          | [Flowchart](../../diagrams/authentication/signup.drawio.svg)          | Email verification and account creation; guide identifies older diagram details. |
| Logout          | [Explanation](logout.md)          | [Flowchart](../../diagrams/authentication/logout.drawio.svg)          | Immediate browser sign-out, refresh-token cleanup and page navigation.           |
| JWT interceptor | [Explanation](jwt-interceptor.md) | [Flowchart](../../diagrams/authentication/jwt-interceptor.drawio.svg) | Access-token refresh and request retry; guide identifies older diagram details.  |

## Approved designs awaiting implementation

| Process         | Guide                             | Diagram                                                               | Purpose                                                                                                                   |
| --------------- | --------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Forgot Password | [Explanation](forgot-password.md) | [Flowchart](../../diagrams/authentication/forgot-password.drawio.svg) | Email OTP recovery, temporary reset permission, password save and session invalidation; approved, implementation pending. |

Return to the [SBP index](../../README.md) for other modules and maintenance conventions.

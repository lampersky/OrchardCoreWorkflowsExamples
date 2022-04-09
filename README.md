# OrchardCoreWorkflowsExamples

## Simple CRUD app (chat)

Requirements:
 - only authorized users can access chat, not authorized users will be redirected to login page with **returnUrl** parameter,
 - users can add new text messages,
 - users can remove ONLY their own messages,
 - users can list messages.

https://youtu.be/fUhMQ3VYUaA

[![OrchardCore Workflows Chat App](https://user-images.githubusercontent.com/6403130/162592795-f34d1f35-4953-428c-baed-171a81140f3e.png)](https://www.youtube.com/watch?v=fUhMQ3VYUaA)

Events and tasks used:
- handle incoming HTTP request,
- validate user,
- set property,
- HTTP redirect,
- HTTP response,
- Retrieve content,
- Create content,
- Delete content,
- If/Else.

## Simple calculator

Requirements:
- users can calculate various formulas,
- users should be warn before executing long running,
- user should be warn when operation type is not selected.

https://youtu.be/drPYM7oexOE

[![OrchardCore Workflows Calculator](https://user-images.githubusercontent.com/6403130/162592808-67c91b8c-e510-4939-b7e4-bcef59bbf12c.png)](https://www.youtube.com/watch?v=drPYM7oexOE)

Events and tasks used:
- handle incoming HTTP request,
- signal,
- set property,
- HTTP redirect,
- HTTP response,
- script,
- while loop,
- for Loop,
- If/Else.

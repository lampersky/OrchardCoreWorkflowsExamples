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

## Analog Clock

Requirements:
- users can generate analog clock face with current server time.

https://youtu.be/2InmwPLkv-Q

[![analogClock](https://user-images.githubusercontent.com/6403130/162595393-79a73166-8128-4f87-a6f2-c3c1298a5080.png)](https://www.youtube.com/watch?v=2InmwPLkv-Q)

Events and tasks used:
- handle incoming HTTP request,
- set property,
- HTTP response,
- Fork,
- Join.

## simple blog

Requirements:
- build very simple blog and get some posts from the Internet,
- do not save already existing posts again,
- automatically check if there are new posts every 12h.

https://youtu.be/ECBQLhbzLJQ

[![dotnetBlog](https://user-images.githubusercontent.com/6403130/162836704-95f52216-e29b-4168-9338-dc5e9ba1b0f7.png)](https://www.youtube.com/watch?v=ECBQLhbzLJQ)

Events and tasks used:
- handle incoming HTTP request,
- timer event,
- script,
- set property,
- for each,
- log,
- HTTP response.

Other:
- creating new content item via scripting,
- executing some custom SQL queries via scripting,
- web scraping is done using regex.

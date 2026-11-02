Time Logging Bot lets the team log hours against Notion tasks from a Slack thread, with the user confirming each change before it's made.

1. **Request:** a team member posts the task, hours, and any client, role, status, or comment. A Make scenario forwards the message to the main n8n workflow.
2. **Lookup:** a GPT-4o orchestrator finds the team member's ID, then retrieves their open Maker or Manager tasks and subtasks from The Task List.
3. **Task matching:** the bot suggests similar tasks in the thread. The user can pick one, create a new task, or create a subtask under a parent task.
4. **Confirmation:** the bot posts a summary of the task and hours, then waits for a yes or no. Xata chat memory keeps the conversation connected across replies.
5. **Notion update:** a sub-agent carries out the confirmed change:
   - **New task or subtask:** finds the client ID, creates the page, records the hours and status, and adds any comment.
   - **Existing task:** adds the new hours to the previous total, updates the status, and adds a comment.
6. **Reply:** the bot posts the result in the Slack thread, including the task link.

Maker hours are stored in `Hours Act.`, and Manager hours are stored in `Manager Hours`.

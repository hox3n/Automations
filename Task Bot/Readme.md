TaskBot lets the team create structured Notion tasks from Slack, with a person approving each task before it's created.

1. **Request:** someone mentions the bot in a Slack channel, either with a message, a forwarded message, or both.
2. **Duplicate check:** OpenAI picks a keyword from the request, and the bot uses it to look for similar tasks in The Task List that aren't marked Done.
3. **Draft:** an AI agent turns the request into a task with a title, description, steps, success criteria, dependencies, and risks. It posts the draft in a Slack thread, along with a warning if it found a similar task.
4. **Review:** the user replies in the thread, and the bot reads the conversation history to decide what they want:
   - **Create:** approves the draft and starts the next step.
   - **Modify:** updates the draft, posts the new version, and asks for approval again.
   - **Cancel:** confirms in the thread that the task was cancelled.
5. **Task creation:** a second workflow, Convert Markdown Text to Notion Task, converts the draft to Notion headings, checklists, and bullet points. It then creates the page in The Task List and posts the link in the Slack thread.

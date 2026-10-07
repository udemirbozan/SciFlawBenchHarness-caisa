# Task files

Files that an agent must fetch from a URL given in the task prompt. One folder per task, named by its harness task ID:

```
task_files/
  2001/
    <data file>
  2002/
    <data file>
```

The prompt gives the agent the file's public URL; the agent reads it with the `visit_webpage` tool.

## Rules for the files
- **Use the raw URL.** For a file on GitHub, link `https://raw.githubusercontent.com/<owner>/<repo>/<branch>/task_files/<id>/<file>`, not the `github.com/.../blob/...` page, which adds ~7,000 characters of site navigation around the content.
- **Keep it under 40,000 characters.** `visit_webpage` cuts everything after that and appends a truncation note.
- **Use text formats** (`.txt`, `.csv`, `.md`, `.json`). The tool converts the response to Markdown, so a PDF arrives as unreadable text.
- **Underscores are escaped.** The Markdown conversion turns `task_id` into `task\_id`; numbers are unaffected.
- **Push the file before running the task.** A newly pushed file is not in any search index, so the agent can only reach it through the URL in the prompt.

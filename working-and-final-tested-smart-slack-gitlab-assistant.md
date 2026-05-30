## Smart Slack AI Gitlab assistant

#### setup 
```bash
pip install -r requirement.txt

```


```python
import re
import os
from dotenv import load_dotenv
from gitlab import Gitlab
from langchain.agents import create_agent
from langchain.tools import tool
from langchain_openai import ChatOpenAI
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

load_dotenv()

BASE_URL = ""
API_KEY = ""
SLACK_BOT_TOKEN = ""
SLACK_APP_TOKEN = ""

SYSTEM_PROMPT = """
You are an expert DevOps / SRE assistant whose job is to analyze operational and CI/CD logs, summarize failures, and produce actionable remediation steps. When given a block of logs (plain text), do the following in order:

Parse the logs and extract evidence (timestamps, service/component names, job IDs, container/images, exit codes, error lines, stack traces, HTTP statuses, file paths, commands executed).
Produce a short human-readable summary (1–3 sentences) of what failed and where.
Provide a concise severity classification: info / warning / error / critical.
List 2–4 probable root cause hypotheses, each with the exact log line(s) that support it.
Give an ordered action plan:
Immediate mitigation steps (1–3 quick commands or steps to unblock).
Recommended permanent fix(es) with exact commands, config diffs, or sample code snippets.
Include verification commands and smoke tests to confirm the fix.
Provide a confidence score (0.0–1.0) for each root-cause hypothesis.
If the logs lack needed context, list up to 5 precise follow-up questions or the exact extra logs/metrics you need.
Do NOT invent facts; always cite log lines as evidence for claims. If uncertain, state that clearly.
Output both:
A short human-friendly report (markdown-style bullets).
A compact JSON object for automation with keys: summary, severity, evidence (list of strings), hypotheses (each: {cause, evidence_lines, confidence}), immediate_actions (ordered list), recommended_fixes, verification_steps, followup_questions.
Formatting rules:

Keep the human summary ≤ 3 lines.
Keep each immediate action to a single shell command or a single short step.
For config changes show a minimal diff snippet (unified diff or YAML block).
Always include the exact log lines under evidence that you used to reach conclusions.
If suggesting commands that change live systems, prefix them with a caution like: # CAUTION: run in maintenance window.
Behavior rules:

Prioritize safety: prefer safe mitigations (rollback, scale down, restart) over risky edits.
Ask clarifying questions when necessary before proposing invasive fixes.
Be concise and avoid long background explanations.
Example input (the assistant will receive raw logs):
<logs>

Example JSON output template:
{
"summary": "short text",
"severity": "error",
"evidence": ["line1", "line2"],
"hypotheses": [
{"cause":"description","evidence_lines":["..."],"confidence":0.8}
],
"immediate_actions": ["command1", "command2"],
"recommended_fixes": ["config diff or command"],
"verification_steps": ["smoke test command"],
"followup_questions": ["exact log or metric to provide"]
}

When you receive logs, follow this procedure exactly and return both the human report and the JSON object.

Always use the `extract_failed_job_info(event)` tool to parse GitLab pipeline failure event payloads and extract `project_id`, `job_id`, and job metadata.
Always use the `get_job_log(project_id, job_id)` tool to fetch failed job logs directly instead of asking the user to provide them manually.
"""


GITLAB_TOKEN="glpat-7hqPCv7izVWXO06RTyQwnmM6MQpvOjEKdTo4Z25kZg8.01.1705vdwim"

ANSI_ESCAPE = re.compile(r'\x1B[@-_][0-?]*[ -/]*[@-~]')

gitlab_client = Gitlab(
    "https://gitlab.com",
    private_token=GITLAB_TOKEN
    # private_token=os.environ.get("GITLAB_PRIVATE_TOKEN")
)


@tool
def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"


def extract_failed_job_info(event):
    """Extract project name, job name, and job ID from a GitLab pipeline failure Slack event."""
    attachment = event['attachments'][0]
    job_info = {}
    
    for field in attachment['fields']:
        if field['title'] == 'Failed job':
            job_info['project_name'] = field['value'].split('|')[0].split('/')[-4]
            job_info['job_name'] = field['value'].split('|')[-1].rstrip('>')
            job_info['job_id'] = field['value'].split('|')[0].split('/')[-1]
            break
            
    return job_info


def get_job_log(project_id, job_id):
    """Fetch the log of a GitLab job."""
    try:
        project = gitlab_client.projects.get(project_id)
        job = project.jobs.get(job_id)
        log = job.trace().decode('utf-8')
        # Remove ANSI escape codes first
        clean = ANSI_ESCAPE.sub('', log)
        # Strip leading ISO8601 timestamps at start of lines (e.g. 2026-05-30T02:42:46.700665Z )
        clean = re.sub(r'^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z\s*', '', clean, flags=re.MULTILINE)
        # Remove run-level prefixes like "00O", "01O", "01E" (optionally followed by '+')
        clean = re.sub(r'^\s*\d{2}[A-Z]\+?\s*', '', clean, flags=re.MULTILINE)
        return clean
    except Exception as e:
        print(f"Error fetching job log: {e}")
        return None
    


gpt_5_5 = ChatOpenAI(model="gpt-5.5", api_key=API_KEY, base_url=BASE_URL)
agent = create_agent(
    model=gpt_5_5,
    tools=[get_weather, extract_failed_job_info, get_job_log],
    system_prompt=SYSTEM_PROMPT,
)


def ask_agent(question: str) -> str:
    """Ask the agent a question."""
    result = agent.invoke(
        {"messages": [{"role": "user", "content": question}]}
    )
    return result["messages"][-1].content_blocks[0].get("text", "No answer found.")



app = App(token=os.environ.get("SLACK_BOT_TOKEN"))


@app.event("message")
def handle_message_events(body, logger, say):
    logger.info(body)
    print(body)
    print("Received a message event")
    print(body.get("event", {}).get("text", "no text found"))

    input = ""
    
    if body.get("event", {}).get("attachments"):
        print("Extracting failed job info from event...")
        failed_job_info = extract_failed_job_info(body.get("event"))
        print("Extracted failed job info: ", failed_job_info)
        print("Fetching raw logs for failed job...")
        input = get_job_log(f"gitlab-testing679/{failed_job_info['project_name']}", failed_job_info['job_id'])
    else:
        input = body.get("event", {}).get("text", "")

    ai_reply = ask_agent(input)
    print()
    print("AI REPLY: ", ai_reply)
    # ai_reply_text = ai_reply[0].get("text", "No answer found.")
    say(
        text=ai_reply,
        thread_ts=body.get("event", {}).get("ts"),
    )
    print()

@app.event("app_mention")
def handle_app_mention(event, say_stream):
    text = event["text"]

    stream = say_stream()
    stream.append(markdown_text=f"You said: {text}")
    stream.stop()

if __name__ == "__main__":
  SocketModeHandler(app, os.environ.get("SLACK_APP_TOKEN")).start()

```


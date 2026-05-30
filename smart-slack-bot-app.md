### smart ai slack assistant

env set up
```
pip install slack_bolt

# python-dotenv is used to load environment variables from a .env file
pip install python-dotenv

# for openai
pip install -qU langchain "langchain[openai]"
```

Set these env vars
```
SLACK_BOT_TOKEN=""
SLACK_APP_TOKEN=""
SLACK_SIGNING_SECRET=""
BASE_URL=""
API_KEY=""
```


```python
import os
from dotenv import load_dotenv

from slack_bolt import App, SayStream, Say
from slack_bolt.adapter.socket_mode import SocketModeHandler
from slack_sdk import WebClient
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

BASE_URL = os.environ.get("BASE_URL")
API_KEY = os.environ.get("API_KEY")

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"


gpt_5_5 = ChatOpenAI(model="gpt-5.5", api_key=API_KEY, base_url=BASE_URL)
agent = create_agent(
    model=gpt_5_5,
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

def ask_agent(question: str) -> str:
    """Ask the agent a question."""
    result = agent.invoke(
        {"messages": [{"role": "user", "content": question}]}
    )
    return result["messages"][-1].content_blocks

load_dotenv()
app = App(token=os.environ.get("SLACK_BOT_TOKEN"))


@app.event("message")
def handle_message_events(body, logger, say):
    logger.info(body)
    print(body)
    print("Received a message event")
    print(body.get("event", {}).get("text", "no text found"))
    ai_reply = ask_agent(body.get("event", {}).get("text", ""))
    print("AI REPLY: ", ai_reply)
    ai_reply_text = ai_reply[0].get("text", "No answer found.")
    say(
        text=ai_reply_text,
        thread_ts=body.get("event", {}).get("ts"),
    )

@app.event("app_mention")
def handle_app_mention(event, say_stream):
    text = event["text"]

    stream = say_stream()
    stream.append(markdown_text=f"You said: {text}")
    stream.stop()

if __name__ == "__main__":
  SocketModeHandler(app, os.environ.get("SLACK_APP_TOKEN")).start()
```

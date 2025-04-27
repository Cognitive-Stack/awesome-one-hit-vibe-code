# Initializing an AI Chatbot with Telegram and OpenAI Agents SDK

This guide will walk you through setting up an AI-powered chatbot using Telegram and OpenAI Agents SDK.

## Prerequisites

1. Python 3.10 or higher
2. Poetry for dependency management
3. A Telegram bot token (from @BotFather)
4. An OpenAI API key
5. A NewsAPI key (for market news)
6. A publicly accessible server for webhooks

## Project Structure

```
├── app/
│   ├── __init__.py
│   ├── main.py           # FastAPI app initialization
│   ├── api/
│   │   ├── __init__.py
│   │   └── telegram_webhook.py  # Telegram webhook endpoint
│   └── startup.py        # Startup events: register Telegram webhook URL
├── bot/
│   ├── __init__.py
│   ├── bot.py            # Bot instance and context management
│   ├── agent.py          # AI agent implementation
│   ├── context.py        # Conversation context and history
│   └── telegram_handler.py  # Process incoming messages
```

## Step 1: Set Up the Project

1. Create a new project directory:
```bash
mkdir my-ai-chatbot
cd my-ai-chatbot
```

2. Initialize Poetry:
```bash
poetry init
```

3. Add required dependencies to `pyproject.toml`:
```toml
[tool.poetry.dependencies]
python = "^3.10"
fastapi = "^0.115.12"
python-telegram-bot = "^20.8"
uvicorn = "^0.27.1"
python-dotenv = "^1.0.0"
pydantic = "^2.6.1"
openai-agents-python = "^0.1.0"
```

## Step 2: Create the Context Management

Create `bot/context.py`:
```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Optional

@dataclass
class Message:
    timestamp: datetime
    sender: str
    content: str

@dataclass
class ChatContext:
    user_id: int
    current_date: datetime = field(default_factory=datetime.now)
    messages: List[Message] = field(default_factory=list)

    def add_message(self, sender: str, content: str) -> None:
        self.messages.append(Message(
            timestamp=datetime.now(),
            sender=sender,
            content=content
        ))

    def get_conversation_history(self) -> str:
        return "\n".join([
            f"{msg.sender}: {msg.content}"
            for msg in self.messages
        ])
```

## Step 3: Create the AI Agent

Create `bot/agent.py`:
```python
from agents import Agent, function_tool

@function_tool
async def search_web(query: str) -> str:
    """Search the web for information"""
    # Implement your web search logic here
    return f"Search results for: {query}"

# Create the AI agent
ai_agent = Agent(
    name="AI Assistant",
    instructions="""
    You are an AI assistant powered by OpenAI. Your role is to:
    1. Answer questions accurately and helpfully
    2. Provide relevant information
    3. Maintain context from previous messages
    
    Your responses should be:
    - Clear and concise
    - Informative and accurate
    - Contextually aware
    """,
    tools=[search_web],
    output_type=str
)
```

## Step 4: Set Up the Telegram Bot

Create `bot/bot.py`:
```python
import os
import logging
from telegram import Bot
from dotenv import load_dotenv
from bot.context import ChatContext

logger = logging.getLogger(__name__)

# Load environment variables
load_dotenv()

# Get bot token from environment
BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
if not BOT_TOKEN:
    raise ValueError("TELEGRAM_BOT_TOKEN not set in environment variables")

# Create bot instance
bot = Bot(token=BOT_TOKEN)

# Store user contexts
user_contexts = {}

async def get_user_context(user_id: int) -> ChatContext:
    """Get or create a context for a user"""
    if user_id not in user_contexts:
        user_contexts[user_id] = ChatContext(user_id=user_id)
    return user_contexts[user_id]
```

## Step 5: Create the Message Handler

Create `bot/telegram_handler.py`:
```python
import logging
from agents import Runner
from telegram import Update
from bot.bot import bot, get_user_context
from bot.agent import ai_agent

logger = logging.getLogger(__name__)

async def handle_telegram_update(update_data: dict):
    try:
        update = Update.de_json(update_data, bot)
        
        if update.message:
            if update.message.text.startswith('/'):
                await handle_command(update)
            else:
                await handle_message(update)
    except Exception as e:
        logger.error(f"Error handling Telegram update: {str(e)}")

async def handle_command(update: Update):
    command = update.message.text.split()[0].lower()
    
    if command == '/start':
        await update.message.reply_text("Hello! I'm your AI assistant. How can I help you today?")
    elif command == '/help':
        await update.message.reply_text("""
Available commands:
/start - Start the bot
/help - Show this help message

You can ask me anything, and I'll do my best to help!
""")

async def handle_message(update: Update):
    try:
        user_context = await get_user_context(update.message.from_user.id)
        user_context.add_message('user', update.message.text)
        
        result = await Runner.run(
            starting_agent=ai_agent,
            input=update.message.text,
            context=user_context
        )
        
        response = result.final_output
        user_context.add_message('bot', response)
        
        await update.message.reply_text(response)
    except Exception as e:
        logger.error(f"Error in handle_message: {str(e)}")
        await update.message.reply_text("I'm sorry, I encountered an error. Please try again later.")
```

## Step 6: Set Up the FastAPI Application

Create `app/main.py`:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api.telegram_webhook import router as telegram_router
from app.startup import startup_event

app = FastAPI(title="AI Chatbot")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(telegram_router, prefix="/api", tags=["telegram"])
app.add_event_handler("startup", startup_event)
```

## Step 7: Create the Webhook Endpoint

Create `app/api/telegram_webhook.py`:
```python
from fastapi import APIRouter, Request
from bot.telegram_handler import handle_telegram_update
import logging

router = APIRouter()
logger = logging.getLogger(__name__)

@router.post("/telegram/webhook")
async def telegram_webhook(request: Request):
    try:
        update = await request.json()
        await handle_telegram_update(update)
        return {"status": "ok"}
    except Exception as e:
        logger.error(f"Error processing Telegram webhook: {str(e)}")
        raise
```

## Step 8: Set Up Environment Variables

Create `.env.example`:
```dotenv
TELEGRAM_BOT_TOKEN=your_bot_token_here
TELEGRAM_WEBHOOK_URL=https://your-domain.com/api/telegram/webhook
HOST=localhost
PORT=8000
```

## Step 9: Run the Application

1. Install dependencies:
```bash
poetry install
```

2. Copy `.env.example` to `.env` and fill in your values:
```bash
cp .env.example .env
```

3. Start the server:
```bash
make run
```

## Step 10: Set Up the Webhook

1. Make sure your server is publicly accessible
2. The webhook will be automatically registered when the application starts
3. Test your bot by sending it a message

## Customization

You can customize your AI agent by:
1. Modifying the agent's instructions in `bot/agent.py`
2. Adding new tools for specific functionality
3. Adjusting the context management in `bot/context.py`
4. Enhancing the message handling in `bot/telegram_handler.py`

## Best Practices

1. Always handle errors gracefully
2. Implement rate limiting for API calls
3. Use proper logging for debugging
4. Keep sensitive information in environment variables
5. Implement proper security measures for your webhook endpoint

## Next Steps

1. Add more sophisticated tools to your agent
2. Implement persistent storage for conversation history
3. Add user authentication and authorization
4. Implement analytics and monitoring
5. Add support for media messages and other Telegram features 
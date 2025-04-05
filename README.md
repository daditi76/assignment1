# assignment1
from transformers import pipeline

# Load a pre-trained conversational model
chatbot = pipeline("conversational", model="microsoft/DialoGPT-medium")

def get_bot_response(user_input):
    from transformers import Conversation
    conversation = Conversation(user_input)
    result = chatbot(conversation)
    return str(result[0].generated_responses[-1])
import sqlite3
from datetime import datetime

# Initialize the database
def init_db():
    conn = sqlite3.connect("chatlogs.db")
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS chatlogs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT,
            user_input TEXT,
            bot_response TEXT
        )
    ''')
    conn.commit()
    conn.close()

def log_interaction(user_input, bot_response):
    conn = sqlite3.connect("chatlogs.db")
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO chatlogs (timestamp, user_input, bot_response)
        VALUES (?, ?, ?)
    ''', (datetime.now(), user_input, bot_response))
    conn.commit()
    conn.close()
from fastapi import FastAPI, Request
from pydantic import BaseModel
from chatbot_model import get_bot_response
from db import init_db, log_interaction

app = FastAPI()

# Initialize DB on startup
init_db()

class UserMessage(BaseModel):
    message: str

@app.post("/chat")
async def chat(user_message: UserMessage):
    user_input = user_message.message
    response = get_bot_response(user_input)
    log_interaction(user_input, response)
    return {"response": response}

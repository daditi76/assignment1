pip install flask spacy pdfplumber psycopg2-binary
python -m spacy download en_core_web_sm
import pdfplumber
import spacy
import re

nlp = spacy.load("en_core_web_sm")

def extract_text_from_pdf(file_path):
    with pdfplumber.open(file_path) as pdf:
        text = ''.join([page.extract_text() for page in pdf.pages if page.extract_text()])
    return text

def extract_details(text):
    doc = nlp(text)
    name = None
    email = None
    phone = None
    education = []
    skills = []

    # Regex for email and phone
    email_match = re.search(r'[\w\.-]+@[\w\.-]+', text)
    phone_match = re.search(r'\+?\d[\d -]{8,12}\d', text)

    email = email_match.group(0) if email_match else None
    phone = phone_match.group(0) if phone_match else None

    # Get name as the first PERSON entity
    for ent in doc.ents:
        if ent.label_ == "PERSON":
            name = ent.text
            break

    # Simple education keyword matching
    education_keywords = ['bachelor', 'master', 'b.tech', 'm.tech', 'phd', 'bsc', 'msc', 'mba']
    for sent in doc.sents:
        if any(edu in sent.text.lower() for edu in education_keywords):
            education.append(sent.text.strip())

    # Skill extraction (example skill list)
    skill_keywords = ['python', 'java', 'sql', 'machine learning', 'excel', 'c++', 'aws']
    for word in skill_keywords:
        if word.lower() in text.lower():
            skills.append(word)

    return {
        'name': name,
        'email': email,
        'phone': phone,
        'education': education,
        'skills': skills
    }
import psycopg2

def create_table():
    conn = psycopg2.connect(database="resumedb", user="postgres", password="your_password", host="localhost", port="5432")
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS candidates (
            id SERIAL PRIMARY KEY,
            name TEXT,
            email TEXT,
            phone TEXT,
            education TEXT,
            skills TEXT
        )
    ''')
    conn.commit()
    conn.close()

def insert_candidate(data):
    conn = psycopg2.connect(database="resumedb", user="postgres", password="your_password", host="localhost", port="5432")
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO candidates (name, email, phone, education, skills)
        VALUES (%s, %s, %s, %s, %s)
    ''', (
        data['name'],
        data['email'],
        data['phone'],
        ', '.join(data['education']),
        ', '.join(data['skills'])
    ))
    conn.commit()
    conn.close()
from flask import Flask, request, jsonify
from parser import extract_text_from_pdf, extract_details
from database import create_table, insert_candidate
import os

app = Flask(__name__)
create_table()

@app.route("/upload", methods=["POST"])
def upload_resume():
    if 'resume' not in request.files:
        return jsonify({"error": "No resume uploaded"}), 400

    file = request.files['resume']
    file_path = f"temp/{file.filename}"
    os.makedirs("temp", exist_ok=True)
    file.save(file_path)

    try:
        text = extract_text_from_pdf(file_path)
        data = extract_details(text)
        insert_candidate(data)
        os.remove(file_path)
        return jsonify(data)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == "__main__":
    app.run(debug=True)
resume_parser/
├── app.py
├── parser.py
├── database.py
├── temp/          # Temporary folder to store uploaded files

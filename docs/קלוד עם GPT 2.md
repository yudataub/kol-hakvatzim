תראה אני לא מבין בקודים אני יביא לך את הקוד הישן ותחליף לי רק את מי שעושה את התמלול

את הסיכום תשאיר לקלוד.

import os

import requests

import openai

import anthropic

from html2docx import html2docx

# Set your API key

openai.api\_key =

def transcribe\_audio\_openai(audio\_path: str, language: str = "he") -> str:

"""

Transcribes an audio file (e.g. MP3) using OpenAI's Whisper API via direct HTTP request,

since openai>=1.0.0 removed the old openai.Audio methods.

:param audio\_path: path to the local audio file

:param language: 2-letter code for the audio language (e.g., 'he' for Hebrew)

:return: The transcribed text

"""

url = "https://api.openai.com/v1/audio/transcriptions"

# Prepare the HTTP headers with authorization

headers = {

"Authorization": f"Bearer {openai.api\_key}"

}

# The multipart/form-data body:

# - 'file' must be a tuple: (filename, fileobj, mimetype)

files = {

"file": (os.path.basename(audio\_path), open(audio\_path, "rb"), "audio/mpeg")

}

# The data params for Whisper

data = {

"model": "whisper-1",

"language": language,

"response\_format": "text" # could also use 'json' or 'verbose\_json'

}

# Make the POST request

response = requests.post(url, headers=headers, files=files, data=data)

# Raise an error if something went wrong

response.raise\_for\_status()

# If response\_format="text", response.text is just the transcription string

transcribed\_text = response.text.strip()

return transcribed\_text

def summarize\_text\_claude(transcribed\_text: str, prompt: str) -> str:

"""

Summarizes Hebrew Torah lesson text using Claude (Anthropic API).

"""

client = anthropic.Anthropic(

api\_key="

if not transcribed\_text.strip():

return "Error: Empty text provided for summarization."

formatting\_instructions = """

Please output your response in \*HTML format\* using proper HTML tags. Example:

<h1>Main Heading</h1>

<p>This is a normal paragraph.</p>

<h2>Subheading Here</h2>

<p>Some more text. <b>This part is bold</b>.</p>

<blockquote>This is a quoted text.</blockquote>

Ensure proper spacing and structure for readability.

"""

# Structure the prompt with XML tags

main\_instructions = f"""

You are a summarization assistant, you are about to receive hebrew transcription, a prompt in hebrew explaining

how to summarize the transcription, and formating instructions in english for your output summarization.

Your output should always be in hebrew language, please make sure to keep RTL support.

<transcription>

{transcribed\_text}

</transcription>

<hebrew\_prompt>

{prompt}

</hebrew\_prompt>

<formating\_instructions>

{formatting\_instructions}

</formating\_instructions>

"""

try:

message = client.messages.create(

model="claude-3-5-sonnet-20241022",

max\_tokens=1024,

messages=[

{"role": "user", "content": main\_instructions}

],

temperature=0.4

)

return message.content[0].text

except Exception as e:

return f"Unexpected error: {str(e)}"

def summarize\_text\_gpt2(transcribed\_text: str) -> str:

"""

Summarizes Hebrew Torah lesson text using GPT-4o with OpenAI API.

The summarization follows structured guidelines to maintain authenticity, clarity, and readability.

"""

client = openai.OpenAI(api\_key="")

if not transcribed\_text.strip():

return "Error: Empty text provided for summarization."

prompt = (

"📌 \*כללים חשובים לסימון המבנה:\*\n"

"- הוסף סימון `### ` לפני כל כותרת ראשית.\n"

"- הוסף סימון `#### ` לפני כל כותרת משנה.\n"

"- הוסף סימון \*\* להדגשת מילים או משפטים חשובים.\n"

"- הוסף סימון `> ` לציטוטים ממקורות תורניים או ציטוטים ישירים מהרב.\n\n"

"כעת, שפר את הטקסט הבא בהתאם להנחיות:\n\n"

f"{transcribed\_text}\n\n"

"טקסט משופר ומסודר בעברית:"

)

try:

response = client.chat.completions.create(

model="gpt-4o",

messages=[{"role": "user", "content": prompt}],

temperature=0.4, # שמירה על יציבות הטקסט

max\_tokens=5000,

)

improved\_text = response.choices[0].message.content.strip()

return improved\_text

except openai.APIError as e:

return f"Error: {str(e)}"

except Exception as e:

return f"Unexpected error: {str(e)}"

def create\_docx\_from\_html(html\_content: str, output\_filename: str = "output.docx"):

"""

Converts HTML content into a DOCX file, handling BytesIO correctly.

"""

docx\_content = html2docx(html\_content, title="LLM Output") # Add a title

# Move to the beginning of the buffer

docx\_content.seek(0)

# Write to a .docx file

with open(output\_filename, "wb") as f:

f.write(docx\_content.read()) # Read bytes from BytesIO and write to file

print(f"DOCX file saved as: {output\_filename}")

def main():

from Prompts import prompt

# תיקיות המקור והיעד

input\_folder = r"C:\Users\USER\OneDrive\שולחן העבודה\AAA"

output\_folder = r"C:\Users\USER\OneDrive\שולחן העבודה\BBB"

prompt = prompt

if not os.path.exists(output\_folder):

os.makedirs(output\_folder)

# סריקה של כל קבצי ה-MP3 בתיקייה

mp3\_files = [f for f in os.listdir(input\_folder) if f.endswith(".mp3")]

for filename in mp3\_files:

audio\_path = os.path.join(input\_folder, filename)

output\_doc\_name = f"{os.path.splitext(filename)[0]}\_סיכום.docx"

output\_path = os.path.join(output\_folder, output\_doc\_name)

print(f"Processing: {filename}")

# שלב 1: תמלול

transcribed\_text = transcribe\_audio\_openai(audio\_path, language="he")

print("Transcription complete")

# שלב 2: סיכום

summary\_text = summarize\_text\_claude(transcribed\_text, prompt)

print("Summary complete")

# שלב 3: שמירה לוורד

create\_docx\_from\_html(summary\_text, output\_path)

print("All files processed.")

if \_\_name\_\_ == "\_\_main\_\_":

main()

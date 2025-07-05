# Museumticketsytemproj

import streamlit as st
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_community.llms import Ollama
from deep_translator import GoogleTranslator

st.title("Chennai Museum Chat Bot")

# Initialize chat history and language preference
if "messages" not in st.session_state:
    st.session_state.messages = []
if "language_code" not in st.session_state:
    st.session_state.language_code = "en"  # Default to English

# Razorpay payment link
razorpay_payment_link = "https://rzp.io/i/SF084RVU"

# Initialize the translator
translator = GoogleTranslator(source='auto', target='en')

# Function to read information from the local text file
def fetch_museum_info(file_path):
    try:
        with open(file_path, 'r', encoding='utf-8') as file:
            content = file.read()
        return content[:3000]  # Limit the amount of data read for concise answers
    except FileNotFoundError:
        return "Error: The specified file was not found."
    except Exception as e:
        return f"Error reading file: {e}"

# Function to translate text to a specified language
def translate_text(text, language_code):
    translator.target = language_code
    translation = translator.translate(text)
    return translation

# List of supported languages and their codes
supported_languages = {
    "hindi": "hi",
    "tamil": "ta",
    "telugu": "te",
    "bengali": "bn",
    "marathi": "mr",
    "gujarati": "gu",
    "malayalam": "ml",
    "punjabi": "pa",
    "kannada": "kn",
    "odia": "or"
}

# File path to the local text file
file_path = r"C:\moulidaran\projectm\utility\chennai museum.txt"

# Retrieve museum data from the local text file
museum_data = fetch_museum_info(file_path)

# Define the prompt with retrieved data
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a knowledgeable assistant with access to detailed museum information. Provide concise and accurate answers based on the user's query."),
    ("user", f"Museum information:\n{museum_data}\n\nUser query: {{query}}")
])

# LLM setup for gemma2:2b
llm = Ollama(model="gemma2:2b")
output_parser = StrOutputParser()
chain = prompt | llm | output_parser

# Display chat messages from history on app rerun
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# Display the "Book Ticket" button
if st.button("Book Ticket"):
    st.write("Redirecting to the payment page... Your ticket will be shared with you after successful payment.")
    st.markdown(f"[Click here to pay]({razorpay_payment_link})")

# Accept user input
if input_txt := st.chat_input("Please enter your queries here..."):
    # Detect if the user requests a specific language response
    selected_language = None
    for language, code in supported_languages.items():
        if language in input_txt.lower():
            selected_language = code
            st.session_state.language_code = selected_language
            break

    # Display user message in chat message container
    with st.chat_message("user"):
        st.markdown(input_txt)
    # Add user message to chat history
    st.session_state.messages.append({"role": "user", "content": input_txt})

    # Generate the chatbot response
    truncated_query = input_txt[:1000]
    result = chain.invoke({"query": truncated_query})

    # Filter out non-informative responses
    if "visit official website" not in result.lower():
        response = result.strip()

        # Translate the response if a language is set
        if st.session_state.language_code != "en":
            response = translate_text(response, st.session_state.language_code)

        # Display the response in the appropriate language
        with st.chat_message("assistant"):
            st.markdown(response)

        # Add the assistant's response to chat history
        st.session_state.messages.append({"role": "assistant", "content": response})

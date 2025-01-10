from flask import Flask, render_template, request, send_file
import pyttsx3
from gtts import gTTS
import os
import speech_recognition as sr
from googletrans import Translator

app = Flask(_name_)

def text_to_speech(text, language):
    """Converts text to speech using pyttsx3."""
    engine = pyttsx3.init()
    voices = engine.getProperty('voices')

    # Use pyttsx3 for Kannada (kn)
    if language == "kn":  # Kannada
        engine.setProperty('voice', voices[1].id)  # Check if Kannada voice is available
    else:
        engine.setProperty('voice', voices[0].id)  # Default voice

    engine.save_to_file(text, 'translated_speech.mp3')
    engine.runAndWait()

def gtts_speech(text, language):
    """Convert text to speech using gTTS."""
    supported_languages = ['en', 'hi', 'es', 'fr', 'de']  # Example of supported languages
    if language not in supported_languages:
        language = 'en'  # Default to English if unsupported language is entered
    
    tts = gTTS(text=text, lang=language, slow=False)
    tts.save("translated_speech.mp3")
    return 'translated_speech.mp3'

def translate_text(input_text, src_lang, dest_lang):
    """Translates text from source language to destination language."""
    translator = Translator()
    translation = translator.translate(input_text, src=src_lang, dest=dest_lang)
    return translation.text

def speech_to_text():
    """Converts speech to text with multiple attempts."""
    recognizer = sr.Recognizer()
    
    with sr.Microphone() as source:
        print("Listening... Please speak clearly.")
        recognizer.adjust_for_ambient_noise(source, duration=1)
        
        attempts = 0
        while attempts < 3:
            try:
                audio = recognizer.listen(source, timeout=5, phrase_time_limit=5)
                text = recognizer.recognize_google(audio)
                return text
            except sr.UnknownValueError:
                pass
            except sr.RequestError:
                pass
            
            attempts += 1
        return None

@app.route("/", methods=["GET", "POST"])
def index():
    translated_text = ""
    audio_file = ""
    
    if request.method == "POST":
        if "speakInput" in request.form:  # If the user wants to speak input
            input_text = speech_to_text()  # Capture speech as text
            if input_text:
                src_lang = request.form["srcLang"]
                dest_lang = request.form["destLang"]
                translated_text = translate_text(input_text, src_lang, dest_lang)
                # Convert the translated text to speech
                if dest_lang == "kn":  # Kannada
                    text_to_speech(translated_text, dest_lang)
                else:
                    gtts_speech(translated_text, dest_lang)
                audio_file = "translated_speech.mp3"
            else:
                translated_text = "Sorry, I could not understand your speech. Please try again."

        elif "textInput" in request.form:  # If the user inputs text directly
            input_text = request.form["inputText"]
            src_lang = request.form["srcLang"]
            dest_lang = request.form["destLang"]
            translated_text = translate_text(input_text, src_lang, dest_lang)
            # Convert the translated text to speech
            if dest_lang == "kn":  # Kannada
                text_to_speech(translated_text, dest_lang)
            else:
                gtts_speech(translated_text, dest_lang)
            audio_file = "translated_speech.mp3"

    return render_template("index.html", translated_text=translated_text, audio_file=audio_file)

@app.route("/download_audio")
def download_audio():
    return send_file("translated_speech.mp3", as_attachment=True)

if _name_ == "_main_":
    app.run(debug=True)

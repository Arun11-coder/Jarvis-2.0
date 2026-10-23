# Jarvis-2.0
This is the second version of Jarvis
import os
import time
import datetime
import re
import threading
import speech_recognition as sr
import pyttsx3
import pywhatkit

from google import genai
from google.genai import types


should_stop = False
is_jarvis_speaking = False

listener = sr.Recognizer()


client = genai.Client(
    api_key="YOUR_API_KEY_"
)

jarvis_behavior = types.GenerateContentConfig(
    system_instruction="""
    You are Jarvis, a futuristic AI assistant.

    Rules:
     Speak like a loyal assistant
     Call user sir
     Keep answers concise
     Avoid weird symbols
    """
)

jarvis_brain = client.chats.create(
   model="gemini-1.5-flash",
    config=jarvis_behavior
)


def clean_text_for_speech(text):

    if not text:
        return ""

    replacements = {
        "&": " and ",
        "%": " percent ",
        "@": " at ",
        "=": " equals ",
        "+": " plus "
    }

    for symbol, word in replacements.items():
        text = text.replace(symbol, word)

    text = re.sub(
        r'[\*#_`\[\]\(\)\{\}>]',
        ' ',
        text
    )

    return " ".join(text.split())



def listen_for_interrupt():

    global should_stop

    with sr.Microphone() as background_source:

        listener.adjust_for_ambient_noise(
            background_source,
            duration=0.3
        )

        while not should_stop:

            try:

                audio = listener.listen(
                    background_source,
                    timeout=1,
                    phrase_time_limit=3
                )

                phrase = listener.recognize_google(
                    audio
                ).lower()

                if (
                    "stop jarvis" in phrase
                    or "jarvis stop" in phrase
                    or "quiet jarvis" in phrase
                ):

                    should_stop = True
                    break

            except:
                continue


def talk(text):

    global should_stop
    global is_jarvis_speaking

    print("Jarvis:", text)

    text = clean_text_for_speech(text)

    if not text:
        return

    should_stop = False

    is_jarvis_speaking = True

    interrupt_thread = threading.Thread(
        target=listen_for_interrupt,
        daemon=True
    )

    interrupt_thread.start()

    try:

        engine = pyttsx3.init()

        voices = engine.getProperty('voices')

        if voices:

            engine.setProperty(
                'voice',
                voices[0].id
            )

        engine.setProperty(
            'rate',
            180
        )

        engine.say(text)

        engine.runAndWait()

        engine.stop()

    except Exception as e:

        print("Voice Error:", e)

    should_stop = True

    is_jarvis_speaking = False


def take_command():

    try:

        with sr.Microphone() as source:

            print("\nListening...")

            listener.adjust_for_ambient_noise(
                source,
                duration=1
            )

            voice = listener.listen(
                source,
                timeout=4,
                phrase_time_limit=6
            )

            print("Recognizing...")

            command = listener.recognize_google(
                voice
            ).lower()

            print("You:", command)

            return command

    except sr.WaitTimeoutError:

        return ""

    except Exception as e:

        print("Listening Error:", e)

        return ""



def run_jarvis():

    command = take_command()

    if command == "":
        return True

    # EXIT
    if any(word in command for word in [
        'goodbye',
        'terminate',
        'shutdown jarvis'
    ]):

        talk(
            "Shutting down systems. Goodbye sir."
        )

        return False

    # TIME
    elif 'time' in command:

        current_time = datetime.datetime.now().strftime(
            '%I:%M %p'
        )

        talk(
            f"The current time is {current_time}"
        )

    # OPEN CHROME
    elif 'open chrome' in command:

        talk(
            "Opening Chrome browser."
        )

        os.system(
            'start chrome'
        )

    # OPEN YOUTUBE ON TV
    elif 'open youtube on tv' in command:

        talk(
            "Opening YouTube on television, sir."
        )

        os.system(
            'cd "%USERPROFILE%\\OneDrive\\Desktop\\platform-tools\\platform-tools" && adb shell monkey -p com.google.android.youtube.tv -c android.intent.category.LAUNCHER 1'
        )

               # PLAY SONG ON TV
    elif 'play' in command and 'tv' in command:

        song = (
            command
            .replace('play', '')
            .replace('on tv', '')
            .replace('tv', '')
            .strip()
        )

        talk(
            f"Playing {song} on television, sir."
        )

        search = song.replace(" ", "+")

        os.system(
            f'cd "%USERPROFILE%\\OneDrive\\Desktop\\platform-tools\\platform-tools" && adb shell am start -a android.intent.action.VIEW -d "https://www.youtube.com/results?search_query={search}"'
        )

        time.sleep(6)

        os.system(
            'cd "%USERPROFILE%\\OneDrive\\Desktop\\platform-tools\\platform-tools" && adb shell input keyevent 23'
        )
        
        
    # TURN ON TV
    elif 'turn on tv' in command:

        talk(
            "Turning on television, sir."
        )

        os.system(
            'cd "%USERPROFILE%\\OneDrive\\Desktop\\platform-tools\\platform-tools" && adb shell input keyevent 26'
        )

    # TURN OFF TV
    elif 'turn off tv' in command:

        talk(
            "Turning off television, sir."
        )

        os.system(
            'cd "%USERPROFILE%\\OneDrive\\Desktop\\platform-tools\\platform-tools" && adb shell input keyevent 26'
        )

    # TV VOLUME UP
    elif 'volume up tv' in command:

        talk(
            "Increasing television volume, sir."
        )

        os.system(
            'cd "%USERPROFILE%\\OneDrive\\Desktop\\platform-tools\\platform-tools" && adb shell input keyevent 24'
        )

    # TV VOLUME DOWN
    elif 'volume down tv' in command:

        talk(
            "Reducing television volume, sir."
        )

        os.system(
            'cd "%USERPROFILE%\\OneDrive\\Desktop\\platform-tools\\platform-tools" && adb shell input keyevent 25'
        )

    # OPEN YOUTUBE ON PC
    elif 'open youtube' in command:

        talk(
            "Opening YouTube, sir."
        )

        os.system(
            'start https://youtube.com'
        )

    # PLAY SONG ON PC
    elif 'play' in command:

        song = command.replace(
            'play',
            ''
        ).strip()

        if song == "":

            talk(
                "Please tell me which song to play, sir."
            )

        else:

            talk(
                f"Playing {song} on YouTube, sir."
            )

            pywhatkit.playonyt(song)

    # GEMINI AI
    else:

        try:

            response = jarvis_brain.send_message(
                command
            )

            answer = response.text

            if not answer:

                answer = (
                    "Sorry sir, "
                    "I could not generate a response."
                )

            talk(answer)

            time.sleep(0.5)

        except Exception as e:

            print(
                "Gemini Error:",
                e
            )

            talk(
                "Sir, I lost connection to my AI servers."
            )

    return True



if __name__ == "__main__":

    talk(
        "Jarvis system initialized. Systems operational, sir."
    )

    running = True

    while running:

        running = run_jarvis()

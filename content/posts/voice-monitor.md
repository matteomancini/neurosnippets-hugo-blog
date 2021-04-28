+++
title =  "Hey Jarvis, what's up? A voice assistant to monitor a target application"
tags = ["SpeechRecognition", "dMRI", "MRtrix3", "tips-and-tricks"]
date = "2021-04-28"
image = "img/jarvis.gif"
caption = "And as a next step to this tutorial, you should make sound your assistant, you know, [_more real_](https://xkcd.com/1931/)."
+++


## Can't you just _tell_ me?

Let's be frank: we are always running a lot of stuff, we constantly juggle multiple flows of data processing, on top of hundreds of tabs in the browser, email clients, drafts, honestly my head starts spinning just if I keep following this train of thought. I always feel that this kind of _ordinary_ and _apparent_ multi-tasking approach always takes its toll on _focus_. When it comes to _deep focus_, I personally believe that there is no solution: one has to avoid any context switching. But for _shallow focus_, wouldn't it be nice to check out how the data processing is doing without looking for the right terminal, finding it, and forgetting what you were doing before? Why you can't just _ask_?

Is it a crazy idea though? It has been already five years from those times when making a personal assistant could sound like a [crazy challenge](https://www.theguardian.com/technology/2016/jan/03/mark-zuckerberg-2016-artificial-intelligence-butler 'What happened to the butler?'). These days, with so many names that TV and videos are forbidden to say aloud ("_Alexa?_", "_Cortana?_", "_Ok Google?_"), making a simple voice assistant is straightforward as ever (while we still may be quite far from [_intelligent conversations_](https://www.forbes.com/sites/robtoews/2020/07/19/gpt-3-is-amazingand-overhyped/ 'How many eyes does my foot have?')). And as a matter of fact, in this post I show how to put together a voice assistant to get updates from some script running! No more context switching between browser, terminal, Illustrator, VNC, ... (_the list is long_) All the related scripts are on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/tips-and-tricks/voice-monitor). 


## Hear no evil, speak no evil

As we are looking for a quick way to implement a voice assistant, we will rely on two packages that makes the experience painless (or almost painless, if sometimes your voice is not recognize correctly). To _vocalize_ the messages, we will use [`pyttsx3`](https://pyttsx3.readthedocs.io/en/latest/ 'pyttsx3 documentation'). To recognize the text from the sound, we will use [`SpeechRecognition`](https://github.com/Uberi/speech_recognition 'SpeechRecognition repository'). There are several examples online showing how to use this duo to make a virtual assistant that does all sorts of things (see the references at the end!).

For a change (not really), let's start by importing the necessary packages and defining the two main functions:

```
import speech_recognition as sr
import pyttsx3


def speak(text):
    engine = pyttsx3.init()
    engine.say(text)
    engine.runAndWait()
    
    
def get_audio():
    r = sr.Recognizer()
    with sr.Microphone() as source:
        audio = r.listen(source)
        said = ""

        try:
            said = r.recognize_google(audio)
            print(said)
        except Exception as e:
            print("Exception: " + str(e))

    return said.lower()
```

The `speak()` function cannot be more straightforward than this: you give it some text, it initializes the `pyttsx3` engine, `say()` queues up the text, and then `runAndWait()` makes you hear it! The other function, `get_audio()` has a bit more going on: using the `Microphone()` as a source, a `Recognizer()` object temporarily records some sounds and then tries to make sense of it through `recognize_google()`; if it manages to, we write that on screen (it is useful to understand _mispronunciation_ cases) and we return it as a lower-case string.

Next, we need to define some keywords and catchphrases:

```
wake_word = "hey jarvis"
queries = ["what is the status", "what is going on",
           "how is going with", "any update with"]
quit = "thank you"
```

I couldn't resist the [_Iron Man_ fashion](https://marvel-movies.fandom.com/wiki/Just_A_Rather_Very_Intelligent_System 'Just A Rather Very Intelligent System'). We can of course have multiple lists of queries, depending on the tasks we want to implement, but here we are keeping things simple.
We can now pull everything together:

```
while True:
    print("Listening...")
    text = get_audio()

    if text.count(wake_word) > 0:
        for phrase in queries:
            if phrase in text:
                filename = text.split(' ')[-1]+'.txt'
                try:
                    file_handle = open(filename,'r')
                    last_update = file_handle.readlines()[-1]
                    file_handle.close()
                    speak("The current status is: " + last_update)
                except IOError:
                    speak("I cannot find " + filename)
        else:
            if quit in text:
                speak("Goodbye")
                break
```

The use of an infinite loop will avoid the need to restart the script each time. For each iteration, we collect some sounds, and we check for the `wake_word`: if we called _Jarvis_, the next step will be seeing if it was followed by any of the expected queries. This is an important point for how this will work: implemented like this, _Jarvis_ expects that we ask what we want in one sentence (e.g. "_Hey Jarvis, what is going on with ...?_"). If a sentence is matched with an expected one, the last word of the sentence is retrieved and use to open a text file. Again, another important point for how this works: this simplification means that everytime we ask for updates, the name of the output file for our target application needs to go _last_. Once that file is read, the last line is retrieved and we finally get the update we want. If by any chance we referred to a file that does not exist, _Jarvis_ will let us know. After the `for`loop is done, if by any chance we said "_Thank you_", an _else_ statement will _release Jarvis from his duty_. We're done!


## Jarvis, can you hear me?

To test _Jarvis_, I wrote down a short script that downloads some diffusion data using [`dipy`](https://dipy.org) and then process everything using [`MRtrix3`](https://www.mrtrix.org). Importantly, I made sure that at each step the script prints on the screen what is going on:

```
#!/bin/bash

echo "Downloading data"
python -c "from dipy.data import fetch_sherbrooke_3shell;fetch_sherbrooke_3shell()"

echo "Converting files"
mrconvert -fslgrad ~/.dipy/sherbrooke_3shell/HARDI193.bvec \
    ~/.dipy/sherbrooke_3shell/HARDI193.bval ~/.dipy/sherbrooke_3shell/HARDI193.nii.gz \
    dwi.mif

echo "Creating a mask"
dwi2mask dwi.mif mask.mif

echo "Estimating response function"
dwi2response dhollander dwi.mif wm.txt gm.txt csf.txt

echo "Reconstructing fiber orientation distribution"
dwi2fod -mask mask.mif msmt_csd dwi.mif wm.txt wm.mif \
    gm.txt gm.mif csf.txt csf.mif

echo "Tracking"
tckgen -select 100000 -seed_image mask.mif wm.mif track.tck

echo "Done!"
```

To run it in a way that _Jarvis_ can monitor it, we need to [redirect the output](https://www.gnu.org/software/bash/manual/html_node/Redirections.html 'A bit more on redirection') to a text file:


```
./run.sh > pipeline.txt
```

It's running! We can now test _Jarvis_ running:


```
python jarvis.py
```

Now you can just say something like "_Hey Jarvis, what is going on with the pipeline?_" while browsing Twitter!


## Useful references

* [A tutorial on SpeechRecognition with Python](https://realpython.com/python-speech-recognition/)

* [An example of Voice Assistant](https://github.com/mmirthula02/AI-Personal-Voice-assistant-using-Python)

* [Another example of Voice Assistant](https://www.techwithtim.net/tutorials/voice-assistant/wake-keyword/)

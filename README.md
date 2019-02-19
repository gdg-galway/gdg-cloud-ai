# Google Cloud AI APIs

The goal of this tutorial is to write a simple web app that records a small audio sample, extract the transcript using the Speech-to-Text API, send the result to the Natural Language API to get the sentiment analysis on the content of our audio sample, and we will finally return it to the front-end and format the result. Once our web app ready, we'll deploy it on Google AppEngine using the GCloud SDK.

### Required dependencies

- GCloud SDK (https://cloud.google.com/sdk/)
- NodeJS 8+ (https://nodejs.org/en/)
- FFMPEG (https://www.ffmpeg.org/download.html)

### Languages/Syntax/Preferences

We will use HTML5, CSS3 and JavaScript (ESNext) in this tutorial. The code is not production-ready but can be passed through Webpack/Parcel/Babel for max compatibility.

I will use Yarn in this tutorial but you can use npm with the same result.

### Setup the project

I made it ease for you guys, you just need to clone this repository and execute `yarn` or `npm install` in the project folder.

You will also need to create a Google account if you don't have one, and register on Google Cloud Platform (https://console.cloud.google.com). You can get $300 credits for free to test the platform.

Once this is done, you can create a new project and enable the Natural Language API and the Speech-To-Text API.

Initialize your project using the following command `gcloud init`.

### Simple web server

Let's open the `index.js` file, and setup a simple web server using Express.

```js
const express = require('express')
const app = express()

// The static content will be served from the "app" folder
app.use(express.static(__dirname + '/app'))

const PORT = process.env.PORT || 8888
app.listen(PORT, () => console.log(`Server running on port ${PORT}`))
```

### Record a sample

Now, let's open `app/index.html`. I added a few things in there like a polyfill to implement the MediaRecorder API on Safari. We could have used RecordRTC to do record our audio sample but it's heavy and totally overkill for what we want to do.
I also added a bit of CSS to make things pretty.

We'll start by creating a function called `startRecording` and bind it to our Recording button. This function will take care of getting access to the user device that will be used to record our sample, initialize the MediaRecorder, and start recording.

```js
let mediaRecorder, recordTO

async function startRecording() {
    document.body.classList.add('recording')
    // Access user media and ask permission if needed
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    // Instanciate MediaRecorder using media device
    mediaRecorder = new MediaRecorder(stream)

    // Stop the recording after 30 seconds
    recordTO = setTimeout(stopRecording, 30000)
    
    // Start recording!
    mediaRecorder.start()

    const audioChunks = []
    
    // Event triggered when audio data is available
    mediaRecorder.addEventListener("dataavailable", event => {
        // Push chunk of data in an array
        audioChunks.push(event.data)
    })

    mediaRecorder.addEventListener("stop", async () => {
        document.body.classList.add('loading')
        const form = new FormData()
        // Transform our array into a Blob
        const blob = new Blob(audioChunks)
        form.append('audio', blob)
        // Send blob to the server
        const res = await fetch('/api/speech', {
            method: 'POST',
            body: form
        })
        if(res.ok) {
            // Code executes if response status from server is OK
            // Parse response to JSON
            const result = await res.json()
            console.log(result)
            // Create a link to play our audio Blob
            player.src = URL.createObjectURL(blob)
            // Insert data in our fantastic UI
            transcript.innerText = result.transcription
            sentiments.innerHTML = result.entities.reduce((html, entity) => {
                return html + `<div class="sentiment">
                    ${entity.name}<br>
                    <b>Type:</b> ${entity.type}<br>
                    <b>Score:</b> ${entity.sentiment.score.toFixed(2)}<br>
                    <b>Magnitude:</b> ${entity.sentiment.magnitude.toFixed(2)}<br>
                    <b>Salience:</b> ${entity.salience.toFixed(2)}
                </div>`
            }, '')
            document.body.classList.add('finished')
        }
        document.body.classList.remove('loading', 'recording')
    })
}

startBtn.addEventListener('click', startRecording)
```

We'll also create a function to stop recording when our pause button is pressed.

```js
function stopRecording() {
    if(mediaRecorder.state === 'recording') {
        clearTimeout(recordTO)
        mediaRecorder.stop()
    }
}

stopBtn.addEventListener('click', stopRecording)
```

### Walk on the server side

So now our audio recording is heading to our web server on `/api/speech`. First step is to get the audio file and convert it to align with the format requirements of Google Speech-To-Text API.

We use `multer` to upload our files directly in the `/uploads` folder by adding the following lines in our `index.js` file:

```js
const isProd = process.env.NODE_ENV === 'production'
const multer = require('multer')

const upload = multer({ dest: (isProd ? '/tmp' : __dirname) + '/uploads/' })

app.post('/api/speech', upload.single('audio'), async (req, res) => {
    /* Process file here */
})
```

We have the file saved, let's convert it to FLAC using FFMPEG. I made a simple function to do that.

```js
const fs = require('fs-extra')
const ffmpeg = require('fluent-ffmpeg')
const ffmpegStatic = require('ffmpeg-static')

const saveFLAC = async (media, destination) => {
    await new Promise((resolve, reject) => {
        ffmpeg(media)
            .setFfmpegPath(ffmpegStatic.path)
            .noVideo()
            .toFormat('flac')
            .on('error', err => {
                reject(err)
            })
            .on('end', () => {
                resolve()
            })
            .save(destination)
    })
    // Clean up
    return fs.unlink(media)
}

app.post('/api/speech', upload.single('audio'), async (req, res) => {
    try {
        // Path to the converted file
        const path = req.file.path + '-f.flac'
        // Convert audio sample to FLAC
        await saveFLAC(req.file.path, path)
        // Get converted file as Buffer
        const audio = await fs.readFile(path)
        /* Send the file to Speech-To-Text API */
    }
    catch(e) {
        console.log('ERROR', e)
        res.sendStatus(500)
    }
})
```

*Note: Not all libraries/binary files will run in Google AppEngine or Google Cloud Functions and certain libraries are preinstalled/configured. FFMPEG is amongst those libraries. Although we need to use the ffmpeg-static module to resolve the static path to the FFMPEG binary. Lucky for us, it's pretty simple.*


Next step is to send our file to Google Speech-To-Text API to get the transcript of the audio sample. We use the NodeJS library provided by `@google-cloud` to take care of this.

```js
const speech = require('@google-cloud/speech')
const speechClient = new speech.SpeechClient()
...
// Path to the converted file
const path = req.file.path + '-f.flac'
// Convert audio sample to FLAC
await saveFLAC(req.file.path, path)
// Get converted file as Buffer
const audio = await fs.readFile(path)
const [response] = await speechClient.recognize({
    config: {
        // Language used in audio file
        languageCode: 'en-US',
        // Audio format
        encoding: 'FLAC',
        sampleRateHertz: 48000,
        // The API will try to punctuate the output
        enableAutomaticPunctuation: true
    },
    audio: {
        // We convert the file to base64 to send it
        content: audio.toString('base64')
    }
})
// Clean up
await fs.unlink(path)
// Regroups the returned strings in one block of text
const transcription = response.results.map(result => result.alternatives[0].transcript.trim()).join('\n')
...
```

Now that we have a block of text that we can process, we can send it to the Natural Language API to extract the sentiments/topics. We use another library from `@google-cloud` to process our text. The Natural Language API requires a Key file to consume the API, you can create and download a key file on the page you accessed to activate the API in the `Credentials` section.

Once that's done, we can simply return the transcript and the sentiment analysis to the front-end in JSON format.

```js
const language = require('@google-cloud/language')
const languageClient = new language.LanguageServiceClient({
    // The path to your key file
    keyFilename: './service-key.json'
})
...
const transcription = response.results.map(result => result.alternatives[0].transcript).join('\n')
const [entities] = await languageClient.analyzeEntitySentiment({
    document: {
        content: transcription,
        type: 'PLAIN_TEXT'
    }
})
const result = {
    transcription,
    entities: entities.entities
}
res.json(result)
...
```

### Test our work

You can try test your code by launching your web server with the command `node index.js` and navigate to http://localhost:PORT_NUMBER (default `PORT_NUMBER` is `8888`). You can now use our simple interface to perform your tests. If an error occurs it will be displayed in the console (on the front-end) or in the terminal if the error occurs in the back-end.

### Deploy your application on Google Cloud Platform

Now that everything is tested and runs well, we'll deploy our application on GCP. If you have correctly installed the GCloud SDK, the deployment should be really straight forward, one command in fact `gcloud app deploy`.

*Note: The file `app.yaml` controls the way your application is deployed on GCP. You can deploy the app on the standard or flex environments and configure the instance(s) your application will be running on.*

### Conclusion

I hope you enjoyed this tutorial. If you have any questions, feel free to contact us on Twitter or Discord:
- Twitter: https://twitter.com/GDGgalway
- Discord: https://discord.gg/JWNVT4W
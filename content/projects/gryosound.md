---
author: "Sean Leishman"
title: "GyroSound: Handheld Music Playing"
date: "2023-07-16"
tags: ["Project", "Music", "App", "Android", "Kotlin"]
ShowBreadCrumbs: true
ShowToc: false
weight: 1
---

> Source code can be found at {{< newtabref href="https://github.com/Sean-Leishman/gyro-sound" title="Github">}}

{{< figure align=center width=auto height=500px  src="../../playSound.png" >}}

GyroSound was created as part a group project while on my exchange year in Singapore. We were tasked with coming up with an Android App which utilised the instrumentation available on the device. What came from this process was GyroSound.

## What is GyroSound?

GyroSound is an Android app that focussed on encouraging people of all ages to explore music with no barriers to entry. Rather than using any complicated instruments, users can use the application to create music on the fly in a fun and novel manner.

Users are able to record sounds or use prerecorded sounds in conjunction with pitch modulation to produce music. The user assigns sounds to buttons each of which can be pressed and adjusted by rotating and moving the phone.

As such, music can be produced by anyone, in an intuitive and friendly manner.

## Inspiration

It has to be said of course that the inspiration for this project came from a youtube video where someone had demonstrated this concept at a far greater scope: {{< youtube yjyhXKOq0Jg >}}

But we wanted to answer the question if this same product could be similarly applied through everyday use, in the form of a mobile application.

## System Deepdive

The system can be split into three major parts: the UI, sounds storage and sound playback & modulation. These parts in combination produce the final result

### UI

#### Main Page

The UI follows a functional approach as were particularly keen on creating a UI that focusses on delivering an intuitive and approachable experience. The main page consists of eight buttons which each displays the settings that are linked to the sound of that button.

#### Sound Selection & Editing Page

{{< figure align=center width=auto height=500px  src="../../editSound.png" >}}

There is then the option of going to the 'Edit Sounds' page that allows the user to select how to offset the original pitch of a recorded sound as a baseline by either semitones or octaves. The record button then allows the user to record sounds and save them using a custom filename.

### Sound Storage

{{< figure align=center width=auto height=500px  src="../../recordSound.png" >}}

Over the lifetime of the project there was a large amount of experimentation in our data modelling and how data would persist across activities.

At first, there was the option of passing data through intents, in particular from the main page to the editing page and vice versa. However we quickly realise that this would result in code that could not be easily extended due to how data would have to be passed between activities.

#### Android ViewModel & LiveData

{{< figure align=center width=auto height=500px  src="../../gyroArch.png" >}}

As such we chose to implement the Android `ViewModel` and Android `LiveData`. These classes allow data to be persistent for the lifecycle of the data. The `ViewModel` is tied to the activity it was initialised in and as such the data is persistent across the child fragments of an activity. This method has the benefit of providing a clear separation of concerns between the state and view of the app. Memory leaks are also prevented by using `LiveData` due to how it is lifecycle aware and as such destroys the data that belongs to it when the activity it belongs to is destroyed.

However, this brings up another issue, as we cannot share data across activities if a `ViewModel` is aware of the lifecycle of the activity it belongs to.

#### Data Repository

To resolve this issue, a singleton class was implemented to store data that should be persistent across activities. On initialisation of an activity, and as such it's `ViewModel`, data is gathered from the data repository and data is set consistently when it is required based on user input.

<!-- prettier-ignore-start -->

{{< highlight kotlin >}}
class MainActivityViewModel(application: Application) : AndroidViewModel(application) {
    private var soundPool = SoundPool(8, AudioManager.STREAM_MUSIC, 0)
    // for all functions, a button id corresponds to a specific slot in the following 2 arrays
    private var soundSelections = IntArray(8) // the soundSelections (any int) of button 1 to 8 (aka the track id that is selected, number ranges from 1 to as many tracks as there are)
    private var streamIds = IntArray(8) // the streamid (any int) of button 1 to 8 (stream ids are generated dynamically)
    private var sounds: HashMap<Int, File> = hashMapOf<Int, File>() //maps soundid to soundfile
    private var semitone = IntArray(8)
    private var octave = IntArray(8)

    private var liveSounds = MutableLiveData<HashMap<Int,File>>()
    private var liveSoundSelection: MutableLiveData<IntArray> = MutableLiveData<IntArray>()

    private var _freq = MutableLiveData<Float>() //global frequency modiifier according tto gyroscope
    private var baseVal = 1.25f
    private var two = 2.0f

    fun addSoundFile(file: File ) {
        val newSoundId = soundPool.load(file.absolutePath ?: "", 1) // load the corresponding track
        sounds.set(newSoundId, file)
        liveSounds.postValue(sounds)
        DataRepository.getInstance().setLiveSounds(sounds)
        Log.d("sounds", "added file at ${file.absolutePath}")
    }
}
{{< /highlight >}}
<!-- prettier-ignore-end -->

As such, we receive the benefit of both classes with no memory leakage and persistent and consistent data across both activities and fragments.

### Sound Playback & Modulation

{{< figure align=center width=auto height=500px  src="../../gyroSystem.png" >}}

The requirements of the project meant that it was necessary to identity a method that could be efficiently and with reasonable effectivess, change the pitch of the sound that is being played. We decided to use `SoundPool` due to it's inbuilt ability to modulate frequency.

#### Pitch Modulation Calculation

To compute a new pitch for the audio file there are two base calculations. First of all, the modulation of pitch of the original audio by a specific number of octaves & semitones and then translating the change in orientation into a change in pitch.

The former involves simply adjusting the frequency of the sound that is being played.

$$
frequency = {basefrequency} * 2^{\frac{N}{12}}
$$

This equation relates to adjusting the current frequency by N semitones.

In order to relate our pitch calculation to the devices gyroscopic measurements we made use of the roll value from the orientation of the gyroscope present on the phone

$$
frequency = {basefrequency} * 1.25^{roll}
$$

`1.25` is used as a base of the power in order to smoothen and normalise the noisy output that is present in the gyroscope data so that the pitch does not fluctuate too rapidly. The result was derived from mapping SoundPool's upper & lower frequency limits to the upper & lower limits of the gyroscope's sensor. As such, we could module the full range of the device's orientation.

As such a final frequency can be calculated that allows pitch to be specified by the user and then by the orientation of the device.

<!-- prettier-ignore-start -->
{{< highlight kotlin >}}
fun changeFreq(newFreq: Float) { //sets the base frequency according to gyroscope inout
  _freq.value = newFreq
  semitone = DataRepository.getInstance().getSemitone()
  octave = DataRepository.getInstance().getOctave()
  for (buttonNum in streamIds) { //iterate through all buttons and change their stream if it is not 0
    if (streamIds[buttonNum] != 0) {
      soundPool.setRate(streamIds[buttonNum], baseVal.pow(_freq.value!!.toFloat()) * two.pow((semitone[buttonNum] + 12 * octave[buttonNum])/12))
    }
  }
}
{{< /highlight >}}
<!-- prettier-ignore-end -->

## App Testing

{{< figure align=center width=auto height=500px  src="../../profiler.png" >}}

To experiment with testing and profiling the application the Android Profiler was used to monitor resource usage and performance throughout runtime.

### Main Page

{{< figure align=center width=auto height=500px  src="../../metrics.png" >}}

The main page showcased constant CPU usage during all stages of activity due to measurement at constant time steps. While memory usage is at 144.5 MB which is not too bad considering that this is essentially a music playback software.

### Navigation With Editing

There is a large spike in energy level as there is quite a bit of initialisation that must occur when starting an application.

### Editing Page

The levels of CPU and energy usage is consistent with the main page however there is an increase in memory usage. Although CPU usage spikes as the user records as a pop up fragment appears and this requires some initialisation and memory usage.

## Evaluation

Overall, there is a general level of satisfaction with delivering upon our original goal of providing a fun way to interact with sound usage, just by intuitive movements. The quality of the application is currently limited by the quality of audio files in use and the application does enhance these shortcomings. Where, the application modulates and transforms the clips distortions can appear and become unpleasant to listen to. As such further improvements in audio modulation and audio file quality could be used to improve the quality of the sound file.

There is also the issue of recording audio as the microphone takes time to pick up information and as such there is choppy playback of recorded sounds where the sound may not be perfectly aligned with the limits of the sound clip.

Certainly this leaves a lot of room to refine and improve the application. In particular, sound files are currently stored locally on the device however this means the user would lose all files if they decided to switch their device. And so, some sort of online account based service could be provided that allows the user to upload their sounds to be accessed using their account details.

This offers further room to expand with sharing functionalities of sounds and music made using the app with other users. Sounds could also be collected from other sources not directly related to the user such as youtube clips and music streaming services to allow a greater range of clips available.

It is also clear that the UI could do with a lot of improvement and refinement to make the application attractive and appealing.

## Lessons Learnt

The process was my first step in Android app development. One which taught me a lot about the process and the techniques involved in effective app design. In particular, the importance of user-orientated design to ensure a smooth & clean experience for the end-user. The project allowed me and the team to explore the fundamental concepts involved in Android development, such as Activities, Fragments, intents and permissions. Hopefully in the future I can use the experience to develop apps or even to apply myself with user-orientated design and development.

See you next time,

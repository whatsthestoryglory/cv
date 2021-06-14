---
title: "Baby Tunes"
date: 2021-06-13T16:00:00-05:00
description: Better Options for our Baby
hero: /images/posts/writing-posts/Idea.svg
menu:
  sidebar:
    name: Baby Tunes
    identifier: babytunes
    weight: 20
---

As any new parent will tell you, there is no shortage of toys for babies that play music and/or make white noise to help them sleep. Between stuffed animals with built-in electronics, baby monitor cameras, and wind-up music boxes that we received as baby shower gifts, we had at least 5 music/noise generators around our baby's room. Unfortunately, none of them worked very well.

## Off the Shelf

![OwlToy](/images/writing-posts/owl.png)

In the first few months, our baby - like most - stayed in our bedroom with us. I won't bore you with all the details of newborn sleep schedules, just trust me when I say there was lots of time spent trying to get a baby to sleep, or to make sure the sleeping baby stayed that way. Over this time, we tried all of them.

The camera was the most disappointing. We tried it first, as it had all the features we'd want - it could play white noise, lullaby's, we could trigger it from the monitor or from our phones - but in practice it was actually kind-of terrible. The microphone on the camera would pick up and horribly distort whatever noise it was playing. The white noise would stop and re-start every 2 minutes. It also just didn't sound very good in the room. We stopped using that one pretty quickly.

Next we tried the stuffed animals. One worked reasonably well, but the volume control on it - despite being a rotary knob - only appeared to have two settings, off and loud. It would also act strangely if the batteries weren't new, where some songs would stop working altogether and others would distort. Another had a decent lullaby that he responded to but no white noise.

Eventually, we settled on a two device setup. We would take the electronics out of one of the animals and use it as a standalone lullaby machine. Then, we repurposed an unused smartphone to run a white noise app on it and would leave that running each night as well. This was great, so long as you could stay on top of batteries, as each device had its own that needed charging.

## Enter Mac Miller.

![Mac Miller](/images/writing-posts/macmiller.png#center)

While pregnant, my wife listened to `Mac Miller - Circles` frequently and noted that the baby would respond in the womb while she was. Occasionally during their most difficult times she would bring it up on her phone while rocking the baby to sleep and it would help. There was one song in particular that really resonated.

{{< macmillerspotify >}}

It was like magic, the crying would be replaced with quiet and calm in an instant. We took note, and it's a good thing we did. We were in for some difficult nights ahead…

## Teeth

![](/images/writing-posts/babyteeth.png#center)

Teething is one of those things that no parent (or child) enjoys. Can you imagine the pain, discomfort, and confusion you must feel as a baby while your teeth form and push their way through your gums? You have no frame of reference for anything that's happening, just constant pain in your mouth for…oh no how long does this go on for? Add to that the fear that separation anxiety can bring about and all of a sudden all your "get the baby to sleep" tricks stop working so well.

One particularly difficult evening we have all our tricks going. The white noise is going. My wife is in there with him, changes our "white noise" phone into a "Surf" phone underneath the crib blasting on repeat. He's finally going out, so she replaces the music with the white noise and gets up to leave…and the screaming returns. "Fuck," she sighs exasperatedly to me. "He just needs a few more minutes. I wish I could just turn on Mac without going into the room and waking him up more."

## It's 2021, there has to be a better way

_Alternate title: I never thought I'd care this much about white noise_

If our baby was going to keep giving us nights like this (he was), I needed to figure something else out. I had a couple [raspberry pi](https://www.raspberrypi.org/) computers kicking around and figured with them I can get us down to a single device playing noise/music, eliminate our reliance on charged batteries, and give us remote control all at the same time.

![](/images/writing-posts/raspi.png)

### Proof of concept

I started by putting together a Pi with a portable speaker. I installed [moode audio](https://moodeaudio.org/) to start, which I chose because it was a full featured music player itself and it had Squeezelite support out of the box so it could integrate with the `Logitech Media Server` that I already use for music around the house.

Because this was going to be a wireless, headless device (no monitor or display attached) I wanted to handle all configuration the same way. After extracting the downloaded file onto my SD card, I copied the `moodecfg.ini.default` file to `moodecfg.ini` and modified it to have my wifi information at first boot. I also used this opportunity to customize the hostname, disable the HDMI output, and make it not wait for the Ethernet port during boot. These tweaks made it faster to boot, consume less power, and made it easy to interact with from other devices once running.

I then installed the micro-sd card, connected the Pi to power, and plugged in my headphones. A few seconds to boot up and I can use the web interface to play some internet radio. Everything works!

### Loud Noises

Internet radio is great and all, but I didn't want our white noise machine to be dependent upon the wifi and internet running, we've had some difficulties there lately. I found a couple music files on the internet with appropriate white noise that I downloaded to a USB drive and attached to the Pi. I didn't want to use the available storage on the SD card for this purpose to try and keep the amount of activity on the card to a minimum so it lasts longer. The white noise files were put into a playlist which was set on repeat, and then I configured the Moode software to resume playback any time the device was powered.

I now had a computer that, within about 45 seconds of receiving power from a USB plug, will play high quality white noise endlessly. Perfect

### Music

Adding to this challenge was the desire to integrate music. It wasn't enough to just have something we could play Surf on repeat with, we needed something that would make it easy to "just hit play" and then return to the white noise on its own. Squeezelite worked well to remotely control our other music devices, so it just made sense to me to build off of that success.

Everything worked 'out of the box,' it was great. Within the `Logitech Media Server` UI I took over the player from its white noise and was able to send any music I wanted to it with ease. I could even synchronize it to other currently playing music. Cool!

Then, the music stopped. There was silence. When `Logitech Media Server` takes over the player, it holds onto it by default. This was a problem, I was hoping that when the song ended, Moode would take back over and resume the white noise. Unfortunately, this isn't what happened. I could find a plugin that would release the `Logitech Media Server` takeover after 15 minutes of no activity, but that was far too long. I did notice, though, that the server had a "sleep" function I could set. When the sleep timer expired, the server would release its takeover.

### Automation

Now that I had a process flow that worked, I focused on making it easier. Enter [Home Assistant](https://www.home-assistant.io/). Home Assistant is a little bit like IFTTT, allowing you to control and automate pretty much anything in your house that's capable of being controlled, including Apple HomeKit and Samsung SmartThings devices. I was already using this for controlling a few smart lamps and outlets, and it was already integrated to `Logitech Media Server`, so it seemed like a natural step to go to next.

```
alias: BabySleep
sequence:
  - service: squeezebox.call_method
    entity_id: media_player.babytunes
    data:
      command: playlist
      parameters:
        - play
        - >-
          /media/Music/Mac
          Miller/Circles/11 - Surf.flac
  - service: squeezebox.call_method
    data:
      command: sleep
      parameters: 300
    entity_id: media_player.babytunes
mode: single
icon: 'mdi:mdiSleep'

```

Above is the script I created within Home Assistant. Upon being called, this script would tell `Logitech Media Server` to take over the Baby Player and play the Mac Miller song. Next it would immediately set the sleep timer to the length of the song. Finally, I went back into Moode and configured it to automatically resume whatever it was doing whenever released from being taken over.

After testing the script, I then created a button within one of my dashboards to trigger the script.

![PlayButton](/images/writing-posts/playmacbutton.png)

### Sleep

That night, I was prepared for anything. I plugged in the Pi as I was getting the baby ready for bed and the white noise started up exactly as intended. We had a great time reading and getting ready for bed was quiet and easy; I was starting to think I wouldn't get to test "in the field" that night. Boy was I wrong.

Fortunately, everything worked swimming-ly. (Okay, that was a bad Mac Miller joke, but like 1 person got it right?). When the baby got worked up, I'd hit the Zzz button, the white noise would stop, and Mac would come on and provide the necessary calming energy. After the song ended, the white noise would gently return providing the perfect atmosphere for sleep. It took a few rounds like that, but eventually the baby went to sleep. Success!

## Next Steps

Pulling out your phone, navigating to an app, then to the correct screen in the app, is a bit of a pain especially when a baby is screaming in your face. We have a TRÅDFRI remote that already controls the lights in the room through Home Assistant, so next will be assigning one of the buttons on the remote to trigger the music.

![Tradfri](/images/writing-posts/tradfributton.png)

Then, the proof of concept will be turned in to a more solid product that can be kept in the baby room without concern. The portable speaker will be replaced with standalone speakers and an enclosure will be built to house everything together and include a power switch. Pair this with a USB battery pack and I not only have a sleep machine for the baby, but a portable boom box for any time!

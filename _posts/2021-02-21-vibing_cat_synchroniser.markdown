---
layout: single
title:  "Vibing Cat Synchroniser - Generating Memes Using Waveform Analysis"
date:   2021-02-21 19:50:18 +0000
header:
  image: "/assets/images/blog/tea/yang_yan_gou_qing/header.jpg"
---

If you haven't come across it already, allow me to introduce you to the [vibing cat](https://www.youtube.com/watch?v=ocg8s6iRDuw).

This particular cat nodding expressively, as if to the rhythm of a song, has become an internet sensation.  I love it, even if the cynical side in me knows that the owner is probably just moving its head.  Amongst its newfound life as a chat react emoticon, the vibing cat has gone on to bless many music videos with its ~cat vibes~.

However, if you have a good sense of rhythm, perhaps you too are irked by how the animation is just _slightly_ off beat from the underlying song.  Quite frankly, we can't blame the cat, and manual beat matching is quite hard.  What can be done?

Introducing...

# The Vibing Cat Synchroniser

<div class="github-card" data-github="eddsalkield/vibing-cat-synchroniser" data-width="400" data-height="" data-theme="default"></div>
<script src="//cdn.jsdelivr.net/github-cards/latest/widget.js"></script>

> Sometimes I think I'm beyond the point of marvelling at how hilariously useless your projects are and then you hit me with something like this[^1]

Naturally, having concluded that the world needs access to better quality _vibing cat_ memes, I set about writing a program to automate the process.  Basically, it performs beat matching and then synchronises the cat to the beat.  So let's see some examples!

### Responding to pauses in the beat structure

<video width="480" height="200" controls="controls">
  <source src="/assets/video/blog/vibing_cat/never_count_on_cat.mp4" type="video/mp4">
</video>
[Haywyre - Never Count On Me](https://www.youtube.com/watch?v=0pNWOjQfGxU)

### Dealing with a changing tempo

<video width="480" height="320" controls="controls">
  <source src="/assets/video/blog/vibing_cat/come_on_eileen.mp4" type="video/mp4">
</video>
[Dexys Midnight Runners - Come On Eileen](https://www.youtube.com/watch?v=GbpnAGajyMc)

### Hilarity ensues when the rhythm is fast

<video width="480" height="320" controls="controls">
  <source src="/assets/video/blog/vibing_cat/through_the_fire_and_flames.mp4" type="video/mp4">
</video>
[DragonForce - Through the Fire and Flames](https://www.youtube.com/watch?v=0Q6668_tWkg)

## How it's done

The first thing that needs to be done is working out where all the beats are.  To achieve this, I used an implementation[^2] of the beat tracking algorithm as described in [this paper](http://www.cp.jku.at/research/papers/Boeck_Schedl_DAFx_2011.pdf): Enhanced Beat Tracking with Context-Aware Neural Networks.

It sounds complicated, but its result is simple: a list of probabilities, one for each 10ms time period in the song.  Each probability represents how likely it is that the given sample is part of a beat.  As an example, let's analyse a piece of music with a well-defined beat but variable tempo: [Come On Eileen](https://youtu.be/GbpnAGajyMc?t=150).

From about 150s in, the song starts to gradually speed up.  If we graph beat certainty against time, we get something looking like this:

<img src="/assets/images/blog/vibing_cat/come_on_eileen_beat_times.png" alt="Come On Eileen beat certainties">

As the rhythm speeds up, so the dots get closer together, although at this scale it's kind of imperceptible.  It's interesting to note, however, that the certainty of recognising a given beat decreases as the tempo increases.  This is because the beat detection is done by a contextually-aware neural network: it takes into account the rhythm that came before, and ranks deviations from it as less certain.

So we set a certainty threshold, and consider only points above this threshold to be "beats".

The increase in tempo is made more clear if we look at the duration of each beat:

<img src="/assets/images/blog/vibing_cat/come_on_eileen_beat_lengths.png" alt="Come On Eileen beat durations">

As the rhythm speeds up, so the duration ("time since last beat") decreases - we can see this as the part that's sloping downward.  The tempo increase is quite dramatic: at beat 360, the rhythm is approximately twice as fast as at beat 320.

Then at beat 366, something interesting happens: the beats are happening so quickly that every other beat is falling below the certainty threshold, resulting in a return to a tempo closer to whta we had at the start.  And then it just goes crazy during the heavy drumming section, before returning to normal.

So now we know with pretty high certainty where all the beats land in the song.  So we're done on the music front, right?  Not so fast...

## The beat break

Quite a few songs exhibit the pattern of beat breaks: the drums will be playing a recognisable beat, and then drop out for a bit before rejoining.  Unfortunately, this means our beat detection is going to stop working so reliably.  Although it does a great job overall, in practise it's going to drop at least a few beats.  Ideally we'd like to not have the cat just sitting there awkwardly during instrumentals, so we need a way of guessing what happens during these intervals.

Let's look at another example:

<iframe width="560" height="315" src="https://www.youtube.com/embed/Tzb0_qQWioQ?start=24" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Taking a look at the beat certainties again, we notice that the beat certainty drops significantly during the break:

<img src="/assets/images/blog/vibing_cat/apollo_beat_break_times.png" alt="Apollo beat certainties">

To avoid missed beats, we need to detect the breaks and forward-fill them with beats of an appropriate tempo.  The beat duration graph gives us a clue of how best to do this:

<img src="/assets/images/blog/vibing_cat/apollo_beat_break_lengths.png" alt="Apollo beat durations">

Look at these massive outliers - they're very distinguished from the group, and nice multiples of the base beat duration too!  So let's make the naïve assumption that any long outlier represents a beat break[^3], and see how far that gets us.

For each long outlier, we forward fill beats by taking an average of the beat lengths before and after the break, and scaling to fit.  And the results look promising:

<img src="/assets/images/blog/vibing_cat/apollo_interpolated_gaps.png" alt="Apollo beat interpolation">

The bottom row is the detected beats, and the top row is our interpolated beats.  Nice!

As an aside, it turns out that this same technique is useful when beats happen too quickly as well, such as bass drum rolls (such as at 1:12).  The beats get detected with a lower certainty, and so a long outlier beat duration is detected, and we can forward fill with the average.  Admittedly, I think it's a bit of a shame that the contextually-aware neural net doesn't spit out rapid hits as beats too, because the resulting cat would be hilarious.  But it's a bit too clever for that.

Anyway, we've so far achieved beat detection with forward filling, and it seems from the graphs that the results look promising.  Now to actually make the video...

## Beat matching the cat

Going back to the source video, you'll note that the timing isn't perfectly consistent.  In fairness, however, it's about as good as you could reasonably expect a cat to be.

I started from [this video](https://www.youtube.com/watch?v=0Ogenr6hVDY), which had already handily replaced the background with a nice, solid [green](green).  Creating a click track with audacity, edited the video to precisely 120bpm by stretching off-beat sections using [kdenlive](https://kdenlive.org/en/)'s time stretch feature.  This is the resulting video file:

<video width="480" height="250" controls="controls">
  <source src="/assets/video/blog/vibing_cat/cat.mp4" type="video/mp4">
</video>

Each beat should be aligned with the point at which that cat's head is in its maximally extended position, and so the idea is that we're going to stretch the video between these points to match the outputted beats.

Given that the cat bops exactly 20 beats in our overlay, and with a bpm of 120 and framerate of 30fps, we can calculate that in the video we have precisely 15 frames per cat movement.  So to align the cat with the beat, we take the duration of each beat as output by the program, and scale our 15 frame sections across that time, in order.

The final piece of the puzzle is compositing our cat video channel onto the music video itself; a pretty trivial operation for ffmpeg.  And with that, we're done!  Let's see how it all looks in practice:

<video width="480" height="200" controls="controls">
  <source src="/assets/video/blog/vibing_cat/apollo.mp4" type="video/mp4">
</video>

For such a long instrumental break, I'd say that's not at all bad!  Although it could probably be improved slightly by a smarter way of interpolating the break.

## Conclusion

I found a cat video funny, so I spent a weekend on this project.  I hope you enjoyed reading about it.

If you're interested in the code, take a look that the [repository](github.com/eddsalkield/vibing-cat-synchroniser/).  It's built to be generic, so can handle non-vibing-cat overlays too!

[^1]: A friend of mine, upon hearing about this project for the first time
[^2]: The library: https://github.com/CPJKU/madmom and documentation for the specific function: https://madmom.readthedocs.io/en/latest/modules/features/beats.html
[^3]: Ideas on how to tell the difference between genuine pauses and beat breaks welcome.

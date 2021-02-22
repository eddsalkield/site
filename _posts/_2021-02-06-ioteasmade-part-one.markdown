---
layout: single
title:  "ioTeasmade part 1: Hacking a 60s Alarm Clock That Makes Tea"
date:   2021-01-17 19:50:18 +0000
categories: technology, teasmade
header:
  image: "/assets/images/blog/tea/yang_yan_gou_qing/header.jpg"
---

An alarm clock.  That makes tea.  Using analogue electronics.

Amazing, right?  Let's hack one.

# The Backstory

Those who know me will recall that this isn't my first electronic tea brewing rodeo.  Back in 2017, I helped build a death trap of an internet-connected tea brewer that we dubbed the [ioTea](https://www.joshuasmailes.co.uk/projects/2017/11/26/iotea.html).

Consisting of a kettle with a hole in, which sat _atop_ the electronics, with pillaged washing maching parts that almost certainly had high voltage AC flowing through them, connected into the ground of an extension lead with an anti-static wristband.  We also stripped out the safety mechanisms making it fail-deadly, and didn't perform sanity checks to prevent requests for tea at 420° from being issued live on stage... Yeah, the hackathon organisers did well not to ask too many questions.

<iframe width="560" height="315" src="https://www.youtube.com/embed/qtW3_EyxSd8" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Needless to say, it made the tea for the demonstration and won us some prizes, but my friends vetoed its future use in communal living spaces.  It spent its retired life in my housemate's room before being returned to its rightful place at the recycling centre.

But it planted the seed.  I now wanted my own home-safe internet connected tea brewer.  I wanted to be able brew tea on command.  I wanted the street cred that comes with becoming a world expert on an obscure home appliance from the 60s.  And most of all I wanted to update the wikipedia page on HTCPCP with a photo of a device that implemented the -TEA extension that _actually brewed tea_ (unlike the sucky teapot-with-a-raspberry-pi that currently resides there).

# The plan

Introducing the teasmade.  A device that is already capable of brewing tea on command.  Its simplicity lends itself well to being understood (and therefore hacked).  And it's been designed to not set your house on fire, with neat safety mechanisms including a bimetallic strip to turn off the heating coil once it gets too hot.  It's also very reliable: the system for pouring the water has zero moving parts, heating the water inside a sealed vessel, with the pressure of the emitted steam forcing the hot water through a metal straw and into the teapot.

Naturally, I went on ebay and grabbed one.

The tea brewing side is easily sorted: we can just whack a single-board computer inside it with a mains relay to control the heating coil.  Now we can forget the onboard alarm system, and instead implement our own in software.  Great!

It also comes with a picture frame for displaying nice photographs of potpourri... doubling up as perfect slot for an LCD or e-ink display.  Neat!

We can keep the little buzzer that's used for the alarm system... or we could add a proper sound system and run an MPD server to play new tunes every morning.  Now that's what I'm talking about!

And finally, the clock itself.  Its interface is analogue with nice clock hands, but it slightly awkwardly keeps time under the assumption that the UK grid has an average frequency of 50Hz[^1].  Ah.  A cool solution for sure, but this isn't going to be as easy as just powering the clock with the correct voltage and expecting a crystal to oscillate for us.  Unless we power the entire original ciruit board.  Which, if I'm going to be soldering my own electronics to it in an ad-hoc, I don't really want to do.

But assuming we can reverse-engineer how the clock driver works, this can actually work to our advantage: certainly the clock itself won't be receiving mains!  There'll surely be some sort of counting circuit that pulses once per second to make the clock tick.  And that's where we can step in with some custom electronics.

But now we have a problem: keeping the clock in time.  Sure, we can just send out a pulse every second, but then I can't syncronise the clock against an NTP server, and where's the fun in that?  The only problem is that the clock, in the original design, wouldn't need to provide feedback, so we can't tell the position of the hands.

Wait a minute...

The alarm system!

Yes, if the alarm is set with the hand at 00:00, the clock will emit a signal twice a day, at midday and midnight!  We can measure the discrepancy every 12 hours, and slightly modify the delay between pulses to correct for the difference.  We didn't need the alarm hand anyway, because we can't advance it through the clock's mechanism.  And using this nifty system, the teasmade could even change to accommodate for daylight savings.  Brilliant!

So armed with a general plan, it was time to figure out how this thing works on the inside.

# Reverse Engineering the Teasmade

The teasmade, at its core, consists of a mechanism to boil water and dispense it into a teapot, with sufficient electronics to handle the rest.  As aforementioned, it uses a pressure-based system to perform the pouring.  Very cool.

The primary safety feature is a bimetallic strip that sits in series between the heating coil and the mains relay.  In true engineering style, it also has a secondary purpose as a standard switch.  The side of the teapot, when correctly positioned, pushes the metal contacts together - preventing the water from boiling if not seated properly.  If the user forgets to put water in, the kettle will overheat and the strip deform to cut the connection.  Then again, when it cools, the heater starts up again.  Not hugely unsafe, as long as it's not left in this state for a long time.

Other than that, there's not a lot to see on the inside other than the electronics connected to the main board.  These consist of the clock, buzzer, some indicator lights, switches to control the alarm and tea brewing processes independently[^2], a small bulb for lighting the front panel at night, a mains relay, and some electronics to control it all.

So I pulled out the main board, and wanted to answer two questions:
* Does the heating coil take pure mains?
* How does the clock circuitry logic work?

What followed was an incredibly deep, incredibly nerdy, and probably unnecessary (but also very fun) complete reverse-engineering of the circuit board.  Strap yourselves in

## The electronics

Two questions:


[^1] Which I suppose means the US will just have to make their own teasmades!
[^2] Although honestly, if you owned this device, would you ever want to use the alarm feature _without_ tea?  Of course not

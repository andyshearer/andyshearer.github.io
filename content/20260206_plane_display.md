+++
title = "Whats that behind you Andy?"
date = 2026-02-06
+++

{{ youtube(id="GtJx_pZjvzc") }}

This advert inspired me, as I often ask myself "Where is that plane going?" when generally pottering about and looking out the window.  Perhaps I'm a five year old at heart.  

So thought about building something that could answer that question for me.  I suspect the neighbours might not like me building something of that scale on the bungalow and opted for a simple ambient type screen / display in the office instead.

{{ youtube(id="ij1_njUdADM") }}

Here is what its showing.

For the nearest plane that is less than 5 miles away.
* The Callsign of the plane.  EZY16FA.
* The distance from me in miles.  3.9.
* The airport it took off from. EDI = Edinburgh Airport.
* The airport its landing at.  STN = London Stansted.
* The planes altitude. 33k, 33,000 feet.

Then, when there are no local planes show some temperature stuff.
* In 20.  Inside temperature.
* Out 7.  Outside temperature.
* Cons 9.  Conservatory temperature.

Its been running for over a year now, and its rather stable!

# Why build this/my requirements?
**Be fun to learn about stuff.** I wanted to play around with SDR, Raspberry Pis, Microcontrollers, Smart homes.  Perhaps 3d Printing? 

**Offline**. It would be simple to throw a load of cash on an Aviation API and just display it.   That however, would be costly,  and wouldn’t offer up to the second info I could get by doing this stuff locally.

**Unobtrusive**. I don’t want bright screens everywhere.  If I had displays I wanted them to fit into my general smart home, so dimmable via smart dimmer. Want them to be liveable and not just a random bunch of wires on a desk.

**Extendable**.  I wanted to add N number of devices which could show what is flying over head.  Think of a small screen in the conservatory, one in the living room.  Perhaps some local website. More in the future.

**Easy to build**.   I’m not a Radio or Aviation expert, nor know much about building electronics.  Nor do I want to really go deep into those topics.  I have a light interest in all three, but you aren’t going to get me talking on my self built HAM radio set anytime soon.

**A play area.**   My day job means I don’t get to write code anymore, but I do love it so.  So part of this project is giving me space to scratch that itch.  

---

# The Setup

## PiPlane - Raspberry Pi

<img src="/img/posts/20260206_plane_display/tar1090.png" alt="Image tar1090 displaying planes flying over a map in my local area" class="post-image" />

### What it does.
We have a little raspberry pi with a USB Software defined radio, that listens to plane transponders.  It entriches this transpoder data from other others sources. Hosts a web interface (above).  Also broadcasts the nearest plane to a MQTT server.

### Hardware

* Raspberry Pi 4.  SDCard. Power supply.
* [USB Software Defined Radio.](https://thepihut.com/products/flightaware-pro-stick-plus-usb-sdr-ads-b-receiver)
* [A fancy arial.](https://thepihut.com/products/60cm-1090mhz-antenna-for-ads-b?variant=41103129215171&country=GB&currency=GBP&utm_medium=product_sync&utm_source=google&utm_content=sag_organic&utm_campaign=sag_organic&gad_source=1&gad_campaignid=11673057096&gbraid=0AAAAADfQ4GF31UiksXK_gTGSr8CQJq9xJ&gclid=Cj0KCQiAiKzIBhCOARIsAKpKLAMsFHCQ55xkUfWaQKkg5Bmpt7QnjQVblm8aBxIfHdiBFL1bfoiRMkkaAiAqEALw_wcB)  (though you can get away with a smaller one)

I have this installed in the garage, with the arial in the loft.  It gives OK range, if you wanted you can go to town with mounting this outside, high, away from obstructions.   I’m not too bothered about chasing perfect reception/coverage here.   

---

## Home Assistant

### What it does
HomeAssistant is great for many things, in this case I happened to already have it running a load of things in the smart home, and a importantly a [Mosquito MQTT server instance](https://www.home-assistant.io/integrations/mqtt/).  So rather than having multiple message queues on the network, made sense to just open it up to other clients on the network to host the nearest plane messages.  


### Hardware
* Intel NUC pc running Proxmox and HAOS.

---

## The Displays
### Unicorn(s)
<img src="/img/posts/20260206_plane_display/unicorn.jpeg" alt="Photo showing to Galactic Unicorns on a wall displaying plane information" class="post-image" />

#### Whats it made of?

* 2 x [Pico 2 W Unicorn](https://shop.pimoroni.com/products/space-unicorns?variant=40842033561683)

 I got one to mess around with, then added another, so its an odd setup.  Now they have bigger ones, so if I was doing this again I would grab that.  The code is ropey, but its using Micropython and listening to the local plane and Smart lamp topics.  They also a few climate temperature topics (inside, outside, conservatory) populated from homeassistant integrations.  I also did hook it up to the doorbell topic to make a noise when fired, but is slightly redundent now.

### OLED Plane Display
{{ youtube(id="XmJ0cCHhQJM") }}

#### Whats it made of?

* Raspberry Pi ZeroW
* Raspbian
* OLED screen.

This sits in the living room for now.  I’m working on my 3D cad skills to print of a nicer container for it.  But this crappy box does for now.

---

# The Software
So really the job is to 
1. Collect Transponder information
2. Add in Airframe and Destination information for the planes.
3. Find the nearest one and send it over to the HomeAssistant MQTT server.
4. Configure the Displays to show Plane and temperature info.
5. Configure HomeAssistant and the displays to work as dimmable lamps in my smarthome setup


## 1. Collect Transponder information
Thankfully, loads of folks already have worked out the hard work of turning a tiny computer, a cheap USB FM/TV tuner, into a device that can listen to plane transponder transmissions.  Indeed, thousands of folks have them setup around the world and they populate services like flightradar24.com and flightaware.com.   FlightRadar24 have a great [tutorial over here](https://www.flightradar24.com/build-your-own), which is basically what I followed.

So how do we script this up to be used in our little project?  Well, part of the FlightRadar24 installation is the setup of dump1090, an open source library that listens and decodes the plane transponder broadcast from a Software Defined Radio devices.   I actually use a version which powers the FlightAware feeder which is slightly diffferent, [dump1090-fa.](https://github.com/edgeofspace/dump1090-fa), [fight aware installation details](https://www.flightaware.com/adsb/piaware/build/).  

When installed you can get a list of aircraft your device is tracking over here.

http://127.0.0.1/dump1090-fa/data/aircraft.json

Now, there are probably better ways of getting this out of dump1090 than relying on a JSON feed over HTTP.   However, when looking into it the nitty gritty of possible outputs of dump1090, my eyes start to glaze over and remember I don’t particularly want to go into the weeds on this. 

So this JSON endpoint has the aircraft, with callsigns, lat/longs , headings, altitude and a whole load of other data updated every second.    Plus JSON over HTTP is easy for a hack like me to play with.

## 2. Add in Airframe and Destination information for the planes.
The transponders from the planes won’t tell you their entire flight plans, but you get the planes callsign.

A callsign is the unique identifier used in radio communications between an aircraft and air traffic control, or between aircraft.   What is interesting, is that in the case of scheduled flights, which are the vast majority of flights near me, the format of these callsigns denote, the company, and a flight number.  e.g.

BA1383  = British Airways,  flight number 1383.

That route number typically responds to a regular route, in this case, Manchester to Heathrow.  Now these routes don’t change much, only when things go wrong or if the company decide to use the flight number for a different route.

How to convert a callsign to route, all locally? Step in folks over at Virtual Radar Server who built the following database to use for their virtual ATC software.  

[vradarserver/standing-data: Dumps of aviation data from Virtual Radar Server's SDM site.](https://github.com/vradarserver/standing-data)

(Interestingly this is the same Dataset that [adsb.lol API](https://api.adsb.lol/) are using, which I just found out when writing this up!)

This is stored as CSVs, but I for my needs its better in a DB, so I have a simple script that downloads this info and puts into a simple SQLlite db. Its sheduled to update once a week or so.

So I now have my Callsign along with the airports the plane has departed and arriving at.

### Airframe Info

I also wanted a little bit of detail of the plane themselves.  Think who the operator is, also the aricraft type.   Which the folks over at adsbexchange have.

[http://downloads.adsbexchange.com/downloads/basic-ac-db.json.gz](https://www.adsbexchange.com/data-samples/)

As with the flight data, I import the airframe info into a local DB.  


## 3. Find the nearest one and send it over to the HomeAssistant MQTT server.

Now we need to pull the flight data and the airframe information together, and ping it over to the MQTT server.  So i build a simple Python script that does does the following.

1. Find the nearest plane from dump1090 feed.
2. Enrich the plane info with flight and airframe info from our local SQLiteDBs.
3. Send over MQTT to be consumed by whatever display we want.

The first is pretty simple.  Open the dump1090-fa aircraft feed,  look up of the plan Lat long, my Lat Long ,calculate distance and sort the aircraft array by nearest.  Add the distance to the object.  Select the nearest.

Then, for the nearest plane, simply take the callsign and query the two local copies of data source for airframe and flight info in our local DB. 

Then its simply the case to ping it over MQTT. via a library like [paho-mqtt.](https://pypi.org/project/paho-mqtt/)

A final MQTT message  we send to our HomeAssistant server looks like this if a plane is within 5 miles, and is updated every few seconds.  In this case, a Boeing 737-800 operating by ASL Airlines UK, going from Manchester Airport to Charles de Gaulle.  We even get details stuff such as roll, speed, and the rather interesting [squawk codes](https://en.wikipedia.org/wiki/List_of_transponder_codes).

```
{
  "hex": "407f6e",
  "flight": "ABV9AN",
  "alt_baro": 13975,
  "alt_geom": 14075,
  "gs": 332.6,
  "ias": 288,
  "tas": 354,
  "mach": 0.56,
  "track": 142.8,
  "track_rate": 0.06,
  "roll": -0.2,
  "mag_heading": 149.8,
  "baro_rate": 3520,
  "geom_rate": 3488,
  "squawk": "6301",
  "emergency": "none",
  "category": "A3",
  "nav_qnh": 1013.6,
  "nav_altitude_mcp": 19008,
  "nav_altitude_fms": 37008,
  "nav_heading": 149.8,
  "lat": 53.108329,
  "lon": -2.271908,
  "nic": 8,
  "rc": 186,
  "seen_pos": 0.4,
  "version": 2,
  "nic_baro": 1,
  "nac_p": 10,
  "nac_v": 2,
  "sil": 3,
  "sil_type": "perhour",
  "gva": 2,
  "sda": 2,
  "mlat": [],
  "tisb": [],
  "messages": 5412,
  "seen": 0.3,
  "rssi": -2.9,
  "distance": 4.98,
  "inflated": true,
  "RouteId": 560806,
  "Callsign": "ABV9AN",
  "OperatorId": 1256,
  "OperatorIcao": "ABV",
  "OperatorIata": "",
  "OperatorName": "ASL Airlines UK",
  "FlightNumber": "9AN",
  "FromAirportId": 286,
  "FromAirportIcao": "EGCC",
  "FromAirportIata": "MAN",
  "FromAirportName": "Manchester",
  "FromAirportLatitude": 53.3536987304688,
  "FromAirportLongitude": -2.27495002746582,
  "FromAirportAltitude": 257,
  "FromAirportLocation": "Manchester",
  "FromAirportCountryId": 191,
  "FromAirportCountry": "United Kingdom",
  "ToAirportId": 82,
  "ToAirportIcao": "LFPG",
  "ToAirportIata": "CDG",
  "ToAirportName": "Charles de Gaulle",
  "ToAirportLatitude": 49.0127983093,
  "ToAirportLongitude": 2.54999995232,
  "ToAirportAltitude": 392,
  "ToAirportLocation": "Paris",
  "ToAirportCountryId": 65,
  "ToAirportCountry": "France",
  "icaotype": "B738"
}
```

## 4. Configure the Displays to show Plane and temperature info.
I'm not going to go into super detail on this, becuase my code here is hacky as hell.  The key things is though, the MiroPython Galactic Unicorn libraries are really detailed, so is mostly just editing them and using a simple MQTT client.  For the smaller OLED display, similar, but as its running on a normal Raspbian instance I'm using python3 running as a service.


## 5. Configure HomeAssistant and the displays to work as dimmable lamps in my smarthome setup
I wanted to be able to treat the displays as Smart lamps so I can switch them off with Hue Dimmer switches, or via Siri.  To do this, I added the following configuration in to HomeAssistant config yaml.  Basically creating lights that which when interacted with would publish messages to specific mqtt topics the displays could listen to and act accordingly. 


```
mqtt:
  - light:
      unique_id: "unicorn_id"
      name: "Unicorn light"
      state_topic: "office/unicorn/status"
      command_topic: "office/unicorn/switch"
      payload_on: "ON"
      payload_off: "OFF"
      brightness_state_topic: 'office/unicorn/brightness'
      brightness_command_topic: 'office/unicorn/brightness/set'
      qos: 0
      optimistic: true
  - light:
      unique_id: "planedisplay_id"
      name: "Plane display"
      state_topic: "livingroom/planedisplay/status"
      command_topic: "livingroom/planedisplay/switch"
      payload_on: "ON"
      payload_off: "OFF"
      brightness_state_topic: 'livingroom/planedisplay/brightness'
      brightness_command_topic: 'livingroom/planedisplay/brightness/set'
      qos: 0
      optimistic: true
```

I added these to Rooms in the HomeAssistant interface.  This means now, Unicorn, is turned off with the same Hue dimmer switch I use to turn off the lights at night.

---

## Conclusions / next stuff to do
I have had loads of fun on this one.  The Unicorns sit behind me in meetings at work and seem to interest folks.  For me, its a nice little useful project and means I can add to it, tinker and play, whilst being geniunaly useful as a thing/product just for me.    

Future things I will probably do with it.

* Totally clean up the code and share it here.  
* I could alter the displays to highlight military planes which might be slightly further away.   They wont have route info, but I could show the type of plane.
* 3D print some nicer cases for the displays.
* Use slight larger OLED screen for the living room display.
* For Unicorn displays I’d like to make it move through modes of what its displaying a little easier / with manual controls. 
* FlipDot displays could be a possibility.  Though haven’t found a simple premade solution that I could use with a API I could hook up to.   No middle ground between “build your own from displays taken out of a bus sign”, to “spend thousands on a locked down app driven display”.  Wonderful resource over here with a similar Plane theme .. [Flip Dot Displays with Raspberry Pi](https://simonprickett.dev/flip-dot-displays-with-raspberry-pi/). 
* Could move over from the FlightRadar24 based installation to something like this really impressive project (which brings together lots).  [sdr-enthusiasts/docker-adsb-ultrafeeder: ADSB-Ultrafeeder is an all-in-one ADSB container with readsb, tar1090, graphs1090, autogain, multi-feeder, and mlat-hub built in](https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder?tab=readme-ov-file#tar1090-configjs-configuration---route-display) 
* Probably should migrate from the polling of the aircraft.json to something else!
* Dimming of the displays work fine, I would go a bit futher and see if I could turn Unicorn display to coloured bulbs in HomeAssistant, so I could change the themes around a bit
# Govee-H6199-Reverse-Engineering
My attempt at reverse engineering the Govee Immersion H6199 RGB lighting strips BLE commands.
------
A huge thanks to BeauJBurroughs who provided most of the original information.

# A Message to Govee

>In the U.S., Section 103(f) of the Digital Millennium Copyright Act (DMCA) [(17 USC § 1201 (f) - Reverse Engineering)](https://www.law.cornell.edu/uscode/text/17/1201) specifically states that it is legal to reverse engineer and circumvent the protection to achieve interoperability between computer programs (such as information transfer between applications). Interoperability is defined in paragraph 4 of Section 103(f).
>
>It is also often lawful to reverse-engineer an artifact or process as long as it is obtained legitimately. If the software is patented, it doesn't necessarily need to be reverse-engineered, as patents require a public disclosure of invention. It should be mentioned that, just because a piece of software is patented, that does not mean the entire thing is patented; there may be parts that remain undisclosed.


Govee I love your product, and I mean no harm in releasing this information. I only did this as a side project so I can control the lighting strips from my own app that runs in my car. I decided to publish my findings and protocol reverse engineering so that anyone else who is looking to do the same might have a place to start. Long story short, __please don't sue me, or DMCA this repo__. If you wish for me to take it down, __please email me or leave a issue on this repo stating that you would like it to be removed, and I will happily do so__.

With all that out of the way, on to the documentation!

# My Findings

I have only tested this on the Govee H6199 so I am unsure if these packets or UUID's work for anything else.
Log is found by enabling developer options bluetooth_hci snoop log `adb bugreport anewbugreport && unzip anewbugreport.zip && wireshark FS/data/misc/bluetooth/logs/btsnoop_hci.log`
To filter the ATT packets:
(btatt or btgatt) and (btatt.handle in {0x15 0x11}) and btatt.opcode == 0x52 and not (btatt.value[0] == 0xaa)

### Checklist of packets
- [x] Keep alive
- [x] Change Color
- [x] Change Color of each of 15 segments
- [x] Gradient
- [x] Set global brightness
- [x] Change to music mode
- [x] Change music mode to cycle colors
- [X] Change Scenes(update)
- [x] Change to video mode

### How packets work
From my understanding, all packets are 20 bytes long. 
The first byte is a identifier, followed by 18 bytes of data, followed by an XOR of ALL the bytes.
0x33 seems to be a command indicator (the only alternatives value for the first byte is 0xaa, 0xa1)
    
    0x33: Indicator
    0xaa: keep alive

The second byte seems identify the packet type

    0x01: Power
    0x04: Brightness
    0x05: Color

The third byte differs based on type.

    For power packets, it's a boolean indicating the power state. (0x00, or 0x01)
    For brightness packets, it corresponds to a uint8 brightness value, affecting lights at about 0x14 to 1% - 0xfe to 100%
    For color packets, this indicates an operation mode.
    
    0x33: Indicator
        0x01: Power
            0x00: Off
            0x01: On
        0x04: Brightness
            0x14: 1%
            0xfe: 100%
        0x05: Color
            0x00: Video
            0x04: Scene
            0x0b: Segment/Color/CT
            0x0c: Music
            



Color commands are sent using the segment command set to all LEDs. Color packets also carry an RGB value, Followed by a CT value if a temperature command is sent

```
Seems to be different from the other strips in that there is no 0x02 Manual mode. 
The values for warm/cold-white LEDs cannot be set arbitrarily. The slider within the app UI uses a list of hardcoded color codes. (thanks Henje!)
````

Zeropadding follows. unless colors can be changed within mode.
Finally, a checksum over the payload is calculated by XORing all bytes.
     
     0x33: Indicator
        0x01: power
            0x00: Off
            0x01: On
        0x04: brightness
            0x14: 1%
            0xfe: 100%
        0x05: color
            0x00: Video
            0x04: Scene
                0x00: Sunrise
                0x01: Sunset
                0x04: Movie
                0x05: Dating
                0x07: Romantic
                0x08: Twinkle (Formerly Blinking)
                0x09: Candlelight
                0x0f: Snowflake
                0x10: Energetic
                0x0a: Breathe
                0x14: Crossing
                0x15: Rainbow
            0x0b: Segment/Color/CT
                0x00:Left Half(1,2,3,4,5,6,7,8)
                     0x00:Right Half (9,10,11,12,13,15)
            0x0c: Music
                0x00: Energic
                0x01: Spectrum(colors)
                    0x000000: red, green, blue
                    0xffffff: red, green, blue
                0x02: Rolling(colors)
                    0x000000: red, green, blue
                    0xffffff: red, green, blue
                0x03: Rhythm
        0xa3: gradient
            0x01: On
            0x00: Off
```
IDENTIFIER, PACKETTYPE, MODE/DATA, MODEID, MODEDATA/DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, DATA, XOR

```

| Type           | Unformatted UUID                 | Formatted UUID                       |
|----------------|----------------------------------|--------------------------------------|
| Service        | 000102030405060708090a0b0c0d1910 | 00010203-0405-0607-0809-0a0b0c0d1910 |
| Characteristic | 000102030405060708090a0b0c0d2b11 | 00010203-0405-0607-0809-0a0b0c0d2b11 |




### Keep Alive
It is always this, it never seems to change. This is sent every 2 seconds from the mobile app to the device.
```
0xAA, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0xAB
aa010000000000000000000000000000000000ab
```
### On/Off
```
0x33, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x33
3301010000000000000000000000000000000033 = on

0x33, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x32
3301000000000000000000000000000000000032 = off
```

### Set Color
RED, GREEN, BLUE range is 0 - 255 or 0x00 - 0xFF    -   Bytes 9 & 10 are set to 0xFF,0x7F to select all LED segments
```
0x33, 0x05, 0x0b, RED, GREEN, BLUE, 0x00, 0x00, 0xFF, 0x7F, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, XOR

#When setting Color Temperature using the slider CT1 & CT2 combine to store the color temperature value in K as reported by the strip. RGB values are set to a predetermined list

0x33, 0x05, 0x0b, RED, GREEN, BLUE, CT1 , CT2 , 0xFF, 0x7F, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, XOR
```

### Set Color Gradient On/Off
```
33:a3:01:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:91 = Gradient On
33:a3:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:90 = Gradient Off
```

### Set Color Segments
The individual 15, segments are distrubuted between left(1-8)(00-ff)and right(9-15)(00-7f).
To address individual segments see ***Color_Segments_chart.md***.
```
0x33, 0x05, 0x0b, RED, GREEN, BLUE, 0x00, 0x00, LEFT, RIGHT, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, XOR
```

### Set Brightness
BRIGHTNESS range is 0 - 255 or 0x00 - 0xFF
```
0x33, 0x04, BRIGHTNESS, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, (0x33 ^ 0x04 ^ BRIGHTNESS)
```

### Set Video Mode
```
P_A
    0x00 = Partial
    0x01 = All
G_M
    0x00 = Movie
    0x01 = Game
Saturation range is 0x00 - 0x64

0x33, 0x05, 0x00, P_A, G_M, Sat, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, XOR
```


### Set Music Modes
```
Valid modes are 0x00,0x02,0x03,0x05
Sensitivity range is 0x00 - 0x63
The next 4 bytes are data used depending on the mode -  WIP

0x33, 0x05, 0x0c, Mode, Sensitivity, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, XOR
```

### Set Scene
```
3305040000000000000000000000000000000032 = Scene(Sunrise)
3305040100000000000000000000000000000033 = Scene(Sunset)
3305040400000000000000000000000000000036 = Scene(Movie)
3305040500000000000000000000000000000037 = Scene(Dating)
3305040700000000000000000000000000000035 = Scene(Romantic)
330504080000000000000000000000000000003a = Scene(Blinking)
330504090000000000000000000000000000003b = Scene(Candlelight)
3305040f0000000000000000000000000000003d = Scene(snowflake)
```

gatttool -i hci0 -b (mac) --char-write-req -a 0x0015 -n (command)
### Examples
```
Video Mode-Partial-Game-Saturation max
gatttool -i hci0 -b <bt address here> --char-write-req -a 0x0015 -n 330500000163000000000000000000000000003d

Color Red
gatttool -i hci0 -b <bt address here> --char-write-req -a 0x0015 -n 33050bFF00000000FF7F00000000000000000042

Color CT 4500K
gatttool -i hci0 -b <bt address here> --char-write-req -a 0x0015 -n 33050bffdfb81194ff7f000000000000000000a0


```




Thank you to BeauJBurroughs,egold555,Freemanium, and ddxtanx for the initial findings.

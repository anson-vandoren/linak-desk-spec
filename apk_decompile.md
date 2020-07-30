As part of the effort to reverse-engineer the Linak BLE protocol, I wanted to
decompile the Desk Control app for Android to see if I could find any
references to what the various UUIDs meant.

I started with a [randomly-Google'ed guide](https://infosecguide.wordpress.com/2013/12/17/step-by-step-guide-to-decompiling-android-apps/)

I installed [APK Extractor](https://play.google.com/store/apps/details?id=com.ext.ui&hl=en_US)
on my phone, ran it, and pointed it to the Desk Control app
(com.linak.deskcontrol). This generated an APK file in /sdcard/ExtractedApks/
that I copied to my PC.

Next I downloaded and installed [Android Studio](https://developer.android.com/studio)
from Google. Prior to proceeding with the next steps, I took some time to load
up the APK in Android Studio and poke around to see if I could find anything
interesting. The `classes.dex` file has some classes at `com.linak` that could
be interesting.

In `com.linak.sdk.linakservices` I found a few notable things:

  - LinakServices$Characteristic$DPG bytecode has a string with the `0011`
    Characteristic UUID.
  - LinakServices$Characteristic$Control bytecode shows:
    - 0002 UUID has a name "COMMAND"
    - 0003 UUID has a name "ERROR"
  - LinakServices$Characteristic$ReferenceOutput bytecode shows:
    - 0021 UUID has name "ONE"
    - 0022 UUID has name "TWO"
    - 0023 UUID has name "THREE"
    - 0024 UUID has name "FOUR"
    - 0025 UUID has name "FIVE"
    - 0026 UUID has name "SIX"
    - 0027 UUID has name "SEVEN"
    - 0028 UUID has name "EIGHT"
    - 0029 UUID has name "MASK"
    - 002A UUID has name "DETECT_MASK"
  - LinakServices$Services seems to name the services:
    - 00001800-0000-1000-8000-00805F9B34FB -> GENERIC_ACCESS (as expected)
    - 0000180A-0000-1000-8000-00805F9B34FB -> DEVICE_INFORMATION (again, as expected)
    - 0001 service is named "CONTROL"
    - 0010 service is named "DPG"
    - 0020 service is named "REFERENCE_OUTPUT"
    - 0030 service is named "REFERENCE_INPUT"
  - LinakServices$Characteristic$DeviceInformation:
    - 00002A24-0000-1000-8000-00805F9B34FB is for "MODEL_NUMBER"
  - LinakServices$Characteristic$GenericAccess:
    - 00002A00-0000-1000-8000-00805F9B34FB is for "DEVICE_NAME"
  - LinakServices$Characteristic$ReferenceInput:
    - 99FA0031-338A-1024-8A49-009C0215F78A -> "ONE"
    - 99FA0032-338A-1024-8A49-009C0215F78A -> "TWO"
    - 99FA0033-338A-1024-8A49-009C0215F78A -> "THREE"
    - 99FA0034-338A-1024-8A49-009C0215F78A -> "FOUR"

In `com.linak.util` I found some useful information:
  - UnitConverter has the usual cm->in and similar conversions, but also
    `cmToRaw` with a value of 0x4059L (100dec). Based on the function calls, it
    seems that cmToRaw is multiplication and rawToCm is division.
    - There is also speedConverter which takes an int value (presumably from
      the BLE characteristic value), then multiplies by 0.09765625f (0x3dc8) and divides
      by 10.0f (0x412)
    - Note: I checked the hex->float with [this calculator](https://gregstoll.com/~gregstoll/floattohex/)

I paused here to see if there was an easier way to get actual decompiled
Java/Kotlin code instead of just the .dex bytecode. While readable, it was kind
of slow and I didn't want to spend the rest of the day just on this step.

This turned out to be easier than I expected - I basically followed the article
linked above step-by-step and ended up with the Java source code in JD-GUI.
I went back to examining `com.linak.sdk`


`com.linak.sdk.connect.RxConnectionManager.class` has some interesting
information about the connection (which was what I was trying to reverse
engineer first, I think). The function `setUpDevice()` seems to initiate the
following commands:

  - GET_CAPABILITIES
  - USER_ID
  - DESK_OFFSET
  - REMINDER_SETTING
  - GET_SET_MEMORY_POSITION_1
  - GET_SET_MEMORY_POSITION_2
  - GET_SET_MEMORY_POSITION_3
  - GET_SET_MEMORY_POSITION_4

`com.linak.sdk.command.DpgCommand.class` has some information about the
commands that were sent. See accompanying [dpg_commands.md](./dpg_commands.md)
file. It also shows how to construct these commands:

  - Byte.MAX_VALUE gives the `0x7F` at the beginning of the commands I saw
  - Next is the command code (shown in file noted above)
  - Remaining bytes are command-dependent:
    - Read commands have a final `0x00` byte (3rd byte)
    - readDeskOffset has two bytes after the command code `0x00-01`
    - Write commands seem to follow:
      Byte.MAX_VALUE, commandCode, Byte.MIN_VALUE, 1, data0, data1
    - Resetting values set them to 32769dec or 0x80-01 (unsure about
      order/ended-ness, may be 0x01-80)
    - Some other commands:
      - writeMemoryPosition
      - writeBleName
      - readMemoryPosition
      - readReferenceCustomSpeeds
      - resetMemoryPosition
      - writeReferenceCustomSpeeds (there are 8 speeds, and 5 required
        parameters here)
      - writeReminderTime
      - writeSettings (5 boolean, 7 int parameters)
      - writeUserID (1 int, 1 UUID param)
      - getControlCommand (returns idx 1 if exists else 0)
      - isReadOperation (varies depending on length of command)

`com.linak.sdk.devices.Desk.class`:

  - getDeskHeight:
    - checks `cmEnabled` which is true if results are in centimeters
    - sums getRawDeskHeight() and getRawDeskOffset() then converts and formats
    - cm conversion is raw / 100.0, and inch conversion is done via helper
      function
  - getFavoritePosition
  - getRawDeskHeight:
    - gets `device.references[0].position`
    - unsure where this is set, so far
  - getRawDeskOffset:
    - gets `device.references[0].offset`
  - moveDown() creates LinCommand.REF_1_DOWN, which should come out to a byte
    array of 0x46-00. Checking on the monitor, it seems to send 3 commands:
      - 0002: 0x46-00
      - 0002: 0xFF-00
      - 0031: 0x01-80
      - testing with holding down the button longer, the 0x46-00 is indeed the
        move down command, and it keeps repeating if the app button is held
        down. The last two commands only happen once each when the button is
        released.
  - moveTo sends a `ReferenceInputCommand` called `deskMoveTo(short param)`
    which calls `ReferenceInputCommand(Input.One, shortToBytes(param))`. It
    seems this then sends a command to `ReferenceInput` service on
    characteristic `ONE` with the value
      - Holding down a 'saved position' button on the app results in:
        - 0002: 0xFE-00
        - 0002: 0xFF-00
        - 0031: 0x2A-16 (repeated until button is released) This is
          ReferenceInput.ONE
        - When button is released:
          - 0002: 0xFF-00
          - 0031: 0x01-80
  - setCustomSpeed
  - requiresReferenceInputWakeUp
  - setDeskOffset()
  - setFavoritePosition()
  - setName()
  - stop()
  - wakeUp()
  - writeSettings()
  - writeUser()



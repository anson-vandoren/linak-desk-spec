## Discovered from desk (LightBlue on Android, and Bluetooth LE Explorer on Windows):
- Generic Access
  - Device Name:
    - Device Address: CE:A4:6E:95:96:A8
    - Service UUID: 00001800-0000-1000-8000-00805f9b34fb
    - Characteristic UUID: 00002a00-0000-1000-8000-00805f9b34fb
    - Readable, writable
    - UTF-8 string "Desk 8586"
    - Handle: 2
  - Appearance:
    - Service UUID: 00001800-0000-1000-8000-0805f9b34fb
    - Characteristic UUID: 00002a01-0000-1000-8000-00805f9b34fb
    - Value: 0x00 00
    - Readable
    - Handle: 4
  - Peripheral Preferred Connection Parameters:
    - See [reference link](https://googlechrome.github.io/samples/web-bluetooth/gap-characteristics.html)
    - Service UUID: 00001800-0000-1000-8000-00805f9b34fb
    - Characteristic UUID: 00002a04-0000-1000-8000-00805f9b34fb
    - Value: 0x10 00 30 00 00 00 32 00
    - Readable
    - Handle: 6
    - 2 bytes each (little endian):
      - Minimum connection interval (20msec here, 1.25msec per integer step)
      - Maximum connection interval (60msec here, 1.25msec per integer step)
      - Slave Latency (0msec here, 1msec per integer step)
      - Connection Supervision Timeout Multiplier (50 here)
  - Central Address Resolution:
    - Service UUID: 00001800-0000-1000-8000-00805f9b34fb
    - Characteristic UUID: 00002aa6-0000-1000-8000-00805f9b34fb
    - Value: 0x01
    - Readable
    - Handle: 8

- Generic Attribute
  - Service UUID: 00001801-0000-1000-8000-00805f9b34fb
  - Service Changed
    - Characteristic UUID: 00002a05-0000-1000-8000-00805f9b34fb
    - Value: N/A (subscribe)
    - Subscribable (indicatable?)
    - Handle: 11
    - Descriptors
      - Client Characteristic Configuration:
        - 00002902-0000-1000-8000-00805f9b34fb

- 99fa0001-338a-1024-8a49-009c0215f78a
    - Characteristic UUID: 99fa0002-338a-1024-8a49-009c0215f78a
        - Writable
        - Handle: 15
    - Characteristic UUID: 99fa-0003-338a-1024-8a49-009c0215f78a
        - Readable, Notifiable, Indicatable (?)
        - Default: NULL
        - Handle: 17
- 99fa0010-338a-1024-8a49-009c0215f78a
    - Characteristic UUID: 99fa0011-338a-1024-8a49-009c0215f78a
        - Readable, Writable, Notifiable
        - Default: 0x01 00
        - Handle: 21
- 99fa0020-338a-1024-8a49-009c0215f78a
    - Characteristic UUID: 99fa0021-338a-1024-8a49-009c0215f78a
        - Readable, Notifiable
        - Default: 0x2A 16 00 00
        - Handle: 25
        - Meaning: position (when stationary) velocity (when moving)
    - Characteristic UUID: 99fa0029-338a-1024-8a49-009c0215f78a
        - Readable
        - Default: 0x01
        - Handle: 28
    - Characteristic UUID: 99fa002a-338a-1024-8a49-009c0215f78a
        - Readable
        - Default: 0x01
        - Handle: 30
- 99fa0030-338a-1024-8a49-009c0215f78a
    - Characteristic UUID: 99fa0031-338a-1024-8a49-009c0215f78a
        - Writable (no response)
        - Handle: 33


# Investigation process

After discovering the endpoints above, I implemented them on an ESP32 with the
service and characteristic UUIDs that I found, as well as the attributes
(writable, readable, etc).

After uploading my code and running it, I attempted to connect to the emulated
desk with the Linak Desk Control app, while logging to serial (from the ESP32)
which values were being read and written.

To log these values, I set up callbacks on each characteristic by creating
custom callback class like this:

```clang
class SnoopCallbacks: public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* pChar) {
    std::string value = pChar->getValue();
    if (value.length() > 0) {
      Serial.print("New value for ");
      Serial.print(pChar->toString().c_str());
      Serial.print("---> ");
      for (int i=0; i < value.length(); i ++) {
        Serial.print((int) value[i]);
        Serial.print("-");
      }
      Serial.println("****");
    }
  }
  void onRead(BLECharacteristic *pChar) {
    Serial.print("Getting characteristic: ");
    Serial.println("pChar->toString().c_str());
    Serial.println("----");
  }
};
```

Then I set these callbacks on each characteristic like:

```clang
chr2->setCallbacks(new SnoopCallbacks());

```

- Desk Control app makes a connection
- DCA reads 0029 characteristic (initial value was NULL)
- DCA writes to 0011 characteristic with 0x7F-80-00
- DCA disconnects and reports an error

Looking back at my BLE explorer for the real desk, I see that 0029 has a value
of 01, so I made a change to initialze with that value on my emulator.
I uploaded and tried to connect again

- DCA reads 0029 (same)
- DCA writes to 0011 with 0x75-80-00 (same)
- DCA reads 0021 (new) which had a value of NULL in the emulator, but
  a non-zero value on the real desk
- DCA reports an error at this point
- Investigating manual movement of the desk (using the switch) while monitoring
  its BLE output, it appears that 0021 shows the position when static, and
  a velocity when in motion

Based on above, changed 0021 to have a zero value instead of NULL to start
with. Realized quickly that I had only set it to 2-byte length (uint16_t), so
went back and made it a 32-bit number instead. Still didn't like it, so changed
starting number to be the same as current reading of my desk, 0x52-16-00-00, or
5714dec

Disconnected and unpaired my actual desk from my PC, then connected/paired it
again. Some differences:
- 0011 is set to 0x01-02-23-01 now

Realized that 0029 was sending a 16-bit (2 octet) value instead of just one
octet like the real desk was. Changed this back to 8-bit `1` instead and tested
again. Still no luck.

Maybe one of the ClientCharacteristicConfigurations needs be be listened to?

Tried listening to all descriptors, but got a kernel panic - still not sure
why, but eliminated the callbacks one by one, and the panic stopped when
I eliminated the 2902 descriptor on the 0003 characteristic, so this must be reading or
written to.

Figured out why the panic happened - simple programming error (from a gross
concept error about pointers and constructors). Fixed this, and then added
callbacks again for all of the descriptors. Saw that two were being called
(written to) during the handshake process, but couldn't figure out which one,
since the callbacks don't have access to the characteristic to which the
descriptor belongs, and they can only spit out the UUID and handle of the
descriptor.

Added some logging to the startup function to show which descriptor handles go
with which descriptors - the UUIDs are all the same (2902 UUID). Also found
that descriptors don't get valid handles until after the grandparent service
has been started

The handshake operation now looks like this:

1. Set 0003 2902 descriptor to 2 (NOTIFY)
2. Set 0011 2902 descriptor to 2 (NOTIFY)
3. Get 0029 characteristic -> returns 1
4. Set 0011 characteristic to 0x7F-80
5. Get 0021 characteristic -> returns 0x58-16 (current position)
6. Set 0003 2902 descriptor to NULL
7. Fails to connect

I connected back to the real desk, and set up notifications for 0003 and 0011
characteristics, then did:
- Get 0029
  - No change to notifications
- Set 0011 to 0x7F-80
  - Got a notification on 0011 that value had changed to 0x01-02-23-01

So that's something. I don't know what that sequence is supposed to mean, but
I can set it back and notify in my emulator and see what happens. I wrote a new
callbacks class specifically for 0011 characteristic that responds with 0x01-02
23-01 when 0x7F-80 is written to the 0011 characteristic. This made some
progress, but looks like another command I need to respond to. Picking up after
(4) above:

5. Emulator sets 0011 to 0x01-02-23-01 and notifies client.
6. Client gets 0021 characteristic -> returns 0x58-16 (current position)
7. Client sets 0011 to 0x7F-86-00
8. Client sets 0003 2902 descriptor to NULL
9. Connection fails

Back to the real desk and Windows BLE explore, and I tried setting 0011 to
0x7F-86-00. The desk responds by changing the value to
0x01-11-00-C6-0F-CA-6A-36-34-47-60-A0-E1-A3-32-56-92-6C-E4 and notifying.

Again, I'm not sure what this value represents, but I can just respond with the
same value in the emulator.

This works, and gets me to the next client-set value on 0011 characteristic,
which is: 0x7f-81-00. It seems that the first and last bytes are staying the
same, and the second byte is the actual command that's being looked for.

My actual desk responds to the 0x7F-81-00 command with a value/notify of
0x01-03-01-66-18, so I add a 3rd handler.

As expected, this gets me a little further. As expected, there's a new command
I need to respond to, which is 0x7F-88-00. The real desk responds with
0x01-0B-07-37-05-32-0A-1E-1E-92-47-E4-7B

Whoa... this seemed to get through the handshaking, sort of...

Next request was 0x7F-89-00, which I hadn't made a handler for, but the app
still proceeded to set the 0003 descriptor back to NULL, but then disconnected
and immediately tried to reconnect again. In the app, I got past the pairing
part, and into the meat of the app. But in the background, it keeps
disconnecting, reconnecting, and asking for all parameters up to and including
0x7F-89-00, so I'll add that one in and see what happens.

The real desk gives a repsonse for 0x7F-89-00 of 0x01-07-01-88-05-59-44-E4-7B,
so I added this into a handler.

Next was 0x7F-8A-00, which the desk responds to with
0x01-07-01-2A-16-FD-6D-E4-7B

I went out on a limb and checked a few additional commands:

0x7F-8B-00: 0x01-05-00-00-00-00
0x7F-8C-00: 0x0B-00
0x7F-8D-00: No response

I added the 3 additional handlers (8A, 8B, 8C).

Success!

As expected, the Desk Control app asked for all 3 additional parameters, which
were successfully given by the 3 new handlers. Following this:

- Client set the hightSpeed (0021) 2902 descriptor to 2 (NOTIFY)
- Client set the 0011 characteristic value to
  0x7f-86-80-1-e5-c0-ca-d8-be-c4-48-e1-a0-8-3e-56-f8-d4-cf-ca
Nothing else happened after this, but the app went into "normal" (i.e.,
handshake complete) mode, and stayed connected to the emulated desk now.

I'm not sure if anything else is supposed to happen on this handshake, so
I plugged the really long command in (0x7F-86-80...) to the actual desk and got
a response of 0x01-00, so I'll add this to the handler too and see if I get any
more. Note that at this point, the app is connected to the desk, so I need to
disconnect and try to reconnect each time to keep making progress.

I did some playing around and it seems that only the first 5 bytes might matter to
this response:
  - If I don't write the 5th byte (E5), I get a response of 0x0B-00 (maybe an
    error?)
  - If I do write the 5th byte (E5), it seems I can use either 01 or 02 for the
    4th byte to get the same response (but not sure what the meanings are)
  - It doesn't seem to matter (response-wise) what I enter after the 5th byte,
    or if I even write anything there.

This seems to be the end of the handshaking - next step will be to start
playing around with the app, and monitoring the commands that get sent to my
emulator, to try to learn more about the communication protocol. Also, I may
try to decompile the Android app APK to see if I can learn anything there.


# Decoding desk capabilities

Earlier, I had seen that during the handshake process, the Desk Control app
sent my emulated desk the 0x7F-80 command on the 0011 characteristic. After
having decompiled the source of the Android app, I found that this command maps
to `GET_CAPABILITIES`. The response that my real desk gives to this is the
4 byte response 0x01-02-23-01. Looking at the `Dpg.class` file, in the
`handleChangeDPG()` method, it looks like the first byte needs to be `01`,
second byte can be anything, and then the final two bytes are the actual
capabilities, so in the case of my actual desk, it's `0x23-01`. The second byte
here `0x01` seems to have something to do with 'references', although I don't
understand that yet. The first byte, though, is more interesting. Of this byte:

  - The first 3 bits tell how many memory locations the desk has (mine has 3)
    - b011
  - 4th bit is "AutoUp"
  - 5th bit is "AutoDown"
  - 6th bit is "BleAllowed"
  - 7th bit is "HasDisplay"
  - 8th bit is "HasLight"

My desk only has the 6th bit set (for BLE allowed), which maps to the actual
hardware I have. To test this, I'm going to try setting 4 memory positions, and
turning on the AutoUp/AutoDown/HasDisplay/HasLight, and see if I can see some
corresponding changes in the app.

A bit string of b11111100 should give me what I'm looking for, which is
equivalent to 0xFC, so the full response I want to send is 0x01-02-FC-01.

OK, so that seemed to work. I definitely have 4 memory positions to choose from
on the app now, as well as a new settings page that gives me the option to turn
on AutoUp and AutoDown (after accepting the ToS, which probably says automatic
motion can be dangerous).

I don't know how to check the HasDisplay/HasLight. The handshake seems to be
the same as it was before even with these turned on, and I don't see anything
different in the app that would signify these were successfully changed, but
I think it's safe to say they were based on the other two.

# Handling movement commands

I want my emulator to be able to "move" up and down when the app commands it
to. I already have a way to keep track of my current position on the ESP32, so
I just need a fake speed (I'll figure out how to get/set real speed later), and
then monitor the commands for motion.

I did some testing on my real desk with the BLE Explorer, and if I send an up/down
command to the 0002 characteristic, the desk moves some preset amount and then
stops. If I send the command repeatedly (i.e., send a new command before the
preset amount of motion stops), the desk continues in a smooth motion. From the
`LinCommand.class` file, I see that up should be 0x47-00 and down should be
0x46-00. When the movement is complete, the app will send two stop commands
(based on `void stop()` in Desk.class):

  - One Lin `STOP_MOVEMENT` command (0xFF-00)
  - One ReferenceInputCommand `deskStop` (0x01-80), to characteristic ONE

I'm not 100% sure yet, but I think the ReferenceInputCommands are to set
a moveTo position command, so setting stop on that characteristic here is just
a safety measure, but doesn't really do anything since we were doing a job
motion instead of point-to-point.

Either way, I'll try to handle all these cases in my code

Height is read from from ReferenceOutput.ONE, as little-endian bytes. For
exanple:

`0x99-05-00-00` is read as 0x00-00-05-99, or just 0x05-99, which is 1433dec.
This number is divided by 100 to get centimeters

I implemented a callback on the COMMAND characteristic (0002) and listened for
the UP/DOWN/STOP commands. I'm not sure what distance the actual desk moves on
a single down command, but I chose 1cm. The implementation is kind of hacky for
now and probably not realistic, but this just needs to be a proof of concept,
and it works on that.

In the loop() function, I'm continuously checking if `lastHeight` differs from
`currentHeight` (which is being updated in the callback), and if they are
different, I'm notifying the client about the new current height, and setting
`lastHeight` appropriately.

It works well in practice, and moving up/down from the Desk Control app now
also updates the height both on the MCU, and in the app.

I could do more with this emulator, but instead I want to get going on building
a client app to connect to my real desk, which is the ultimate goal of this
project. To do this, I'll need:

- A web server on the ESP32 to show things like:
  - Current position
  - Up/down controls
  - Show saved positions (and let move to)
  - Desk connected to
  - Controls to help connect to a new desk
  - Controls to connect to MQTT
  - Controls to connect to WiFi
- A connection to MQTT
  - Update MQTT topics when position change
  - Listen on MQTT topics to know when to move up and down


# Side notes:

- I'm using a NodeMCU-ESP32s board, and I kept needing to hold down the reset
  button while flashing. This is annoying, and caused many times that flashing
  failed because I walked away to get another cup of coffee at just the wrong
  time. I fixed this problem with a 10µm electrolytic capacitor as decribed by
  [this article](https://randomnerdtutorials.com/solved-failed-to-connect-to-esp32-timed-out-waiting-for-packet-header/)


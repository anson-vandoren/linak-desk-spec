# Trying to connect to the desk from a ESP32 client

- Seems it does need to be paired.
- Not sure how the app knows what BLE device might be a valid desk. I guess it
  just tries to enumerate all possible devices and see which respond correctly
  to its queries?
    - Should be able to check this in the code and see what values it's looking
      for in a response on certain characteristics
    - Does the code show anything about filtering by name, manufacturer data,
      serviceUUIDs, payload?
- The app seems to store a list of addresses that might be desks, and then lets
  you try to connect to whichever you choose
- Looking at RxScanManager.java, there are ScanSettings (scanMode1) and
  scanFilter (LinakServices.Services.CONTROL.uuid()). So it seems that the
  CONTROL service is the only thing that must be present for the device to show
  up as a possibility to connect to.

# Scanning for appropriate devices

1. Scan all BTLE devices
2. For all devices found, check if they expose the service
   `99FA0001-338A-1024-8A49-009C0215F78A` which is the CONTROL service on
   a Linak DPG desk
3. Present a list of all devices that 'probably are' Linak products
4. When user picks one, then run through the checks to determine capabilities
   and to actually confirm this is a Linak desk
   
## Problem in the Arduino ESP32 BLE library

- Not sure if this is a problem necessarily with the client library, or with
  the device I'm connecting to.
- In some cases, there is a response (`ESP_GAP_SEARCH_INQ_RES_EVT`), but the
  `param->scan_st.scan_rsp_len == 0`, when it should have some additional data.
- This presents for me as failing to get a name for the device that got
  scanned, even when that device does have a name.
- This might be because the server is too slow to send a response?
- Or maybe the client isn't waiting long enough before returning?
- Made some changes to `BLEScan.cpp.handleGAPEvent()` to check if the stored
  `BLEAdvertisedDevice` had a non-zero response, and if it didn't, but the
  current event does have a non-zero response, to replace the stored device
  with the new one. This mostly works, but occasionally causes a reboot - not
  ideal. This might be due to a memory mismanagement issue.
- Probably the right fix is to find out why my device(s) aren't responding to
  the advertising inquiry (or not responding in time).
- This doesn't seem to actually work consistently, and will crash the ESP32
  after random intervals. Need to look for a root cause why the response isn't
  always coming through. Something about slave interval?
- Alternatively, just leave it alone for now, and if any device exposes DPG
  CONTROL service, but we don't have a name for it, just query the name
  separately.

2020-08-03:
- Re-did the edits to the `BLEScan` and `BLEAdvertisedDevice` files to check if
  we had a non-zero response this time compared to last, and fixed whatever the
  memory leak/random crash was, but still wasn't getting consistent results for
  some reason.
- Spent some more time on this. I had partial success by setting `is_continue`
  to `true`, which means that the secondary scans (and failed scan responses)
  to a given device won't overwrite the entry in the map. This works all the
  time unless the very first scan response didn't include the name, in which
  case the no-name will be saved and never overwritten. It's still not really
  a fix at all, since it doesn't address the underlying issue of a scan inquiry
  response being sent by the peripheral device, but not actually being read in
  by the client.

- Finally decided that setting `is_continue==true` is probably good enough for
  my needs for now. There's still something wrong here, but possibly with the
  low-level underlying BLE library. I don't think I'll gain much or be able to
  fix much at this point by dicking around here.
  
2020-08-04:
- Added a map to LinakScanManager to keep track of DPG devices, and also added
  JSON serializer of known devices.
- Still need mechanism to remove a device when it's not found in a subsequent
  scan (I guess?)
- Something is broken - after several scan iterations the JSON disappears

2020-08-05:
- Did some cleanup work and trying to sort out what appears to be memory
  issues.
- Learned [from here](https://arduinojson.org/v6/how-to/reuse-a-json-document/)
  that `JsonDocument`s are designed to be short-lived. Need to refactor to keep
  data in a struct, and create the JSON only when needed.

2020-08-07:
- After much headbanging, it seems that advertisement data may be split among
  a few messages. For example, one advertising message may have the name and
  appearance, while the next advertising message may have manufacturer data and
  service UUIDs.
- This means that to get complete data, one may need to listen to several
  sequential advertisements and compile them together.
- Hacked together a callback listener to update if the name comes in
  afterwards. This still isn't 100% perfect, but probably good enough to move
  on.
  
2020-08-08:
- Working on the web interface. Had everything working but now even though
  websocket responses are being sent back, the client is not picking them up.
- Added ability to display address & name of any DPG device.
- Added a button (not hooked up yet) to connect to any found device
- Added button to scan for all devices
- Apparently sending `client->ping()` on the `WS_EVT_CONNECT` causes the client
  to hang up?... wtf...? Maybe the pong response while still inside the connect
  event confuses it?

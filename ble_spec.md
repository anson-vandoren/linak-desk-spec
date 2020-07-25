# Client Characteristic Configuration
  - 0x01 00: notification on
  - 0x02 00: indicate on

# How to read Advertisement Payload
  - [Ref article](https://www.silabs.com/community/wireless/bluetooth/knowledge-base.entry.html/2017/02/10/bluetooth_advertisin-hGsf)
  - Little endian
  - First byte is length
  - Second byte is [AD Type](https://www.bluetooth.com/specifications/assigned-numbers/generic-access-profile/)
  - Remainoing `length` bytes are the value


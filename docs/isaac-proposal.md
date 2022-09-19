# Remote Unlocking System for Older Vehicles
## Main features
This project is designed to add remote unlocking, locking and keyless entry to older vehicles that do not have these features.  This project will involve a key fob and a main control unit inside the vehicle.  Incorporating ideas from Computer Networks and Cybersecurity, these devices will communicate securely through wireless channels.  The main control unit should be able to detect the keys presence automatically unlock the car when you are close to the driver's door.  This device should also be able to automatically lock the vehicle when the key is no longer detected (in case the user forgot to lock the doors).  Lastly, this device should be able to function as a normal key fob (i.e. unlock/lock car with the press of a button). 

## Competing projects, Similarities and Differences

The most similar project to mine is [Open-Source-RKS][1] on GitHub.  This project uses bluetooth to detect the presence of a key and unlock the vehicle, and lock when key is no longer detected.  The similarities of this project are that the main control device can detect the key fob and automatically unlock/lock the car.  But in my project I plan to have manual buttons as well to add another layer of reliability.  Open RKS allows the user to connect a smartphone via bluetooth to serve as the key fob, I would not be adding this feature to my project because I think it adds potential security vulnerabilities.

## Projects to pull ideas from
One project I would be pulling ideas from is this [ESP32-Based Smart Bluetooth Lock][2]. This project is a "smart home" lock that can unlock your house with the presence of your phone.  The similarities in ideas from this project are that an ESP32 board gets a signal from somewhere, then uses some sort of verification to verify the user, and then unlocks the door via a relay and electronic actuator.  In my project, I would be using a different wireless protocol, but I will be using basic ideas of receive, verify, unlock.

## Hardware
On the hardware side there are a few options, but I will only be discussing the two cheapest options.  The two options are the Raspberry Pi Pico W (shortened to Pico from here on) and the ESP32.  The first aspect I will look at to decide on hardware is the power draw of each device.  The Pico draws around 0.23 watts (4.6V * 0.05A = 0.23W) and around 0.69 watts (4.6V * 0.15A = 0.69W) at full load.`[1]` The ESP32 on the other hand is a little bit more power hungry, coming in at 0.3 watts at idle and 0.83 watts under full load (downloading and uploading files to network).`[2]`  The Pico seems to be slightly more efficient than the ESP32, but the ESP32 runs at double the clock speed as the Pico so they are probably pretty similar in the performance per watt side.  Speaking of clock speed, the Pico is powered by the RP2040 which is a dual core Arm cpu clocking at 133 MHz and has 264 KiB RAM and 2 MB of program storage.`[3]`  The ESP32 is powered by a Xtensa dual core cpu clocking at 240 MHz and has 520 KiB of SRAM and 488 KiB ROM.`[4]`  The ESP32 has a lot more 'horsepower' on paper than the Pico but the only thing that might work better for my application would be the larger program storage on the Pico, but they would still both work.  The next thing to look at for this comparison is software compatibility. The ESP32 has libraries for WiFi, bluetooth and the custom protocol [ESP-NOW][3], whereas the Pico currently only has libraries for WiFi, so to figure out witch microcontrollers to use, we need to look at these protocols.
## Software
In terms of protocols we have three choices, ESP-NOW, WiFi, and Bluetooth.  Since all three of these protocols are not supported by both microcontrollers, our choice here will decide what microcontroller we will have to use.  So, to help figure out wich is best, I will make a simple test program using each.  
### WiFi (Pico W)
First I tested using the Pico W using its only current supported protocol, WiFi.  In terms of actual programming, it was realtivly easy to set up, but as soon as you get into functionallity it soon became a nightmare.  For this test I used two Picos, one serving as an WiFi access point and hosting a simple webserver, the other as a client which would connect to the access point and then send HTTP requests to the server.  To set up the access point we just need to set a ssid and password and setup a network in access point mode.
```python
ssid = "PicoWAccessPoint"
password = "1234567890" ## This should be a secure password in final version, this is for testing purposes ONLY
ap = network.WLAN(network.AP_IF)
ap.config(essid=ssid, password=password)
ap.active(True)
print('Connection successful')
print(ap.ifconfig())
```
Then to connect to this access point we can do something like this, if the connection does not get properly established in 10 seconds, it will raise an exception and back in main() it will wait 10 seconds then try the whole thing again.
```python
def connect():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(ssid, password)
    
    tries = 0
    while not wlan.isconnected() and wlan.status() >= 0:
        print("Waiting to connect:")
        sleep(1)
        tries += 1
        
        if (tries == 10):
            raise RuntimeError("Unable to connect")
        
    if (wlan.isconnected()):
        print(f"connected, ip adress: {wlan.ifconfig()}")
```

Then back on the server side, the web server is just a super simple html page.  Then while listening for connections we look for the /light/on or /off to turn off or on the built in LED.
```python
html = """<!DOCTYPE html>
<html>
    <head> <title>Pico W</title> </head>
    <body> <h1>Pico W</h1>
        <p>%s</p>
    </body>
</html>
"""
addr = socket.getaddrinfo('0.0.0.0', 80)[0][-1]

s = socket.socket()
s.bind(addr)
s.listen(1)

# Listen for connections
while True:
    try:
        stateis = "LED is OFF"
        cl, addr = s.accept()
        print('client connected from', addr)
        request = cl.recv(1024)
        print(request)

        request = str(request)
        led_on = request.find('/light/on')
        led_off = request.find('/light/off')
        print( 'led on = ' + str(led_on))
        print( 'led off = ' + str(led_off))

        if led_on == 6:
            print("led on")
            led.on()
            stateis = "LED is ON"

        if led_off == 6:
            print("led off")
            led.off()
            stateis = "LED is OFF"

        response = html % stateis

        cl.send('HTTP/1.0 200 OK\r\nContent-type: text/html\r\n\r\n')
        cl.send(response)
        cl.close()

    except OSError as e:
        cl.close()
        print('connection closed')
```
Then on the client side the sending of these HTTP requests is dead simple only taking up 3 lines of code.  The 192.168.4.1 is the default gateway for the Pico access point, I was able to connect to this address on my phone when connected to the AP.
```python
def sendServerMessage(mes):
    r = urequests.get(f"http://192.168.4.1/light/{mes}")
    print(r.content)
    r.close()
```

These components worked great seperatly, I was able to connect to the access point on my phone and send HTTP requests to turn on and off a LED, and the other was able to connect to a network and send out HTTP requests to various webservers just fine.  But as soon as I put these together everything stopped working, and not to mention that the client seemed to only connect to the access point if the access point was on and in range when the client is first turned on, even though the code keeps trying to connect every 10 seconds for 10 seconds.  So for now we keep looking at the other protocols.  
### ESP-NOW
Next I tested ESP-NOW, it was a bit harder to understand and program it at first but the finnished product is much, much more robust.  Both the receiver and sender can be turned off (simulating losing conection from range) at will and once turned back on, they immedietly (within a few milliseconds) begin trasmitting again.  To use ESP-NOW we first establish a peer to peer connection, directly connecting with the devices MAC address.
```c++
// MAC Address of the ESP32 to send message to
uint8_t broadcastAddress[] = {0x94, 0xB5, 0x55, 0x26, 0x44, 0xB8};
esp_now_peer_info_t peerInfo;

....

void setup(){
  // Register a peer
  memcpy(peerInfo.peer_addr, broadcastAddress, 6);
  peerInfo.channel = 0;
  peerInfo.encrypt = false;  // NOTE: ESP-NOW messages can be encrypted

  // Add the peer
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer");
    return;
  }
}
```
Then to make a message we just use a simple struct with two attributes, there could be more if needed (maximum of 250 bytes total).
```c++
typedef struct struct_message {
    String a;
    bool b;
} struct_message;

struct_message example;
```
And then to send the message, it must be converted to `uint8_t` data type and sent with the `esp_now_send` method
```c++
void loop() {
  esp_err_t result = esp_now_send(broadcastAddress, (uint8_t *) &example, sizeof(LED_Off_Sig));
  reportError(result);
  delay(2000);

}
```
My project would greatly benefite from the added robustness of ESP-NOW, when comparing to WiFi, it is a night and day difference.  With WiFi you have to create an access point with a secure password and host a webserver to communicate.  Whereas with ESP-NOW all one has to do is register a device to broadcast to by it's MAC address and then you can send any data up to 250 bytes in size.  In the case of connection droppoff, the system continues to function and immediatly resumes sending data when the other device becomes in range again.  So for my project I will be using ESP-NOW to communicate.



`[1]` Forum thread where they are talking about power draw of the Pico https://forums.raspberrypi.com/viewtopic.php?t=337145

`[2]` Article on ESP32 power draw https://therandomwalk.org/wp/esp32-power-consumption/

`[3]` Raspberry Pi Pico W data sheet https://datasheets.raspberrypi.com/picow/pico-w-datasheet.pdf

`[4]` ESP32 data sheet PDF https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf

[1]: https://github.com/fryefryefrye/Open-Source-RKS "Open Source Remote Keyless System"
[2]: https://www.electronicsforu.com/electronics-projects/hardware-diy/esp32cam-based-smart-bluetooth-lock "Smart Bluetooth Lock using ESP32"
[3]: https://github.com/espressif/esp-now "ESP-NOW protocol github page"

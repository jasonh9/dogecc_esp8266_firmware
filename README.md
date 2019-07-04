# **Dogecc ESP8266 Firmware** #
Firmware toolchain to create your custom esp8266 firmware, we'll be using the nodemcu development version of the esp8266.

> Note: if you're too lazy to go through this process just download a prebuilt binary from the release page or [download now](https://github.com/jasonh9/dogecc_esp8266_firmware/releases/download/v1.0/nodemcu_float_master_20190622-0208.bin)


## **Building the firmware with docker**

* Configure your tags (build version, firmware name, etc) in `app/include/user_version.h`
* Add or remove modules from your build `app/include/user_modules.h`

Build your firmware
```
docker run --rm -ti -v `pwd`:/opt/nodemcu-firmware marcelstoer/nodemcu-build build
```
The bin file is located in : `/bin/nodemcu_float_master_20190622-0208.bin`

## **Flashing your firmware**

### Using esptool.py flash your resulting bin file

1. Zero out the current device by using esptool.py
```
./esptool.py --port /dev/ttyUSB0 erase_flash
```
2. Flash the firmware
```
 esptool.py --port /dev/ttyUSB0 write_flash -fm dio 0x00000 bin/nodemcu_float_master_20190622-0208.bin
```

## **Uploading your code**
Upload your code using esplorer https://github.com/4refr0nt/ESPlorer
* baudrate : 115200
* file name should be named `init.lua` this is the initial script that is loaded when the device is turned on.

### **Bootstrap**

The following bootstrap code will connect your device to a specified wifi access point and load a main `application.lua` file.

`init.lua`
> The initial program that runs when the devices is booted  
```lua
-- load credentials, 'SSID' and 'PASSWORD' declared and initialize in there
dofile("credentials.lua")
  
function startup()
    if file.open("init.lua") == nil then
        print("init.lua deleted or renamed")
    else
        print("Running")
        file.close("init.lua")
        -- the actual application is stored in 'application.lua'
        print("Application running...")
        dofile("application.lua")
    end
end

-- Define WiFi station event callbacks
wifi_connect_event = function(T)
    print("Connection to AP("..T.SSID..") established!")
    print("Waiting for IP address...")
    if disconnect_ct ~= nil then disconnect_ct = nil end
  end
  
  wifi_got_ip_event = function(T)
    -- Note: Having an IP address does not mean there is internet access!
    -- Internet connectivity can be determined with net.dns.resolve().
    print("Wifi connection is ready! IP address is: "..T.IP)
    print("Startup will resume momentarily, you have 3 seconds to abort.")
    print("Waiting...")
    tmr.create():alarm(1000, tmr.ALARM_SINGLE, startup)
  end
  
  wifi_disconnect_event = function(T)
    if T.reason == wifi.eventmon.reason.ASSOC_LEAVE then
      --the station has disassociated from a previously connected AP
      return
    end
    -- total_tries: how many times the station will attempt to connect to the AP. Should consider AP reboot duration.
    local total_tries = 75
    print("\nWiFi connection to AP("..T.SSID..") has failed!")
  
    --There are many possible disconnect reasons, the following iterates through
    --the list and returns the string corresponding to the disconnect reason.
    for key,val in pairs(wifi.eventmon.reason) do
      if val == T.reason then
        print("Disconnect reason: "..val.."("..key..")")
        break
      end
    end
  
    if disconnect_ct == nil then
      disconnect_ct = 1
    else
      disconnect_ct = disconnect_ct + 1
    end
    if disconnect_ct < total_tries then
      print("Retrying connection...(attempt "..(disconnect_ct+1).." of "..total_tries..")")
    else
      wifi.sta.disconnect()
      print("Aborting connection to AP!")
      disconnect_ct = nil
    end
  end
  
  -- Register WiFi Station event callbacks
  wifi.eventmon.register(wifi.eventmon.STA_CONNECTED, wifi_connect_event)
  wifi.eventmon.register(wifi.eventmon.STA_GOT_IP, wifi_got_ip_event)
  wifi.eventmon.register(wifi.eventmon.STA_DISCONNECTED, wifi_disconnect_event)
  
  print("Connecting to WiFi access point...")
  wifi.setmode(wifi.STATION)
  wifi.sta.config({ssid=SSID, pwd=PASSWORD})
  -- wifi.sta.connect() not necessary because config() uses auto-connect=true by default
```

`credentials.lua`
> Wifi access point credentials  
```lua
SSID="<wifi_access_point>"
PASSWORD="<wifi_access_point_password"
```

`application.lua`
> The application that runs when there is a successful wifi
connection.
* application description : toggle a LED wired to GPIO 5 when a button is pressed wired to GPIO 6. `debounce()` is an attempt to debounce the button using software.
```lua
local pin = 6 -- GPIO 6
local led = 5 -- GPIO 5
local toggle = 0

function debounce (func)
    local last = 0
    local delay = 50000 -- 50ms * 1000 as tmr.now() has μs resolution
    return function (...)
        local now = tmr.now()
        local delta = now - last
        if delta < 0 then delta = delta + 2147483647 end; -- 
        if delta < delay then return end;
        last = now
        return func(...)
    end
end

function onChange ()
    print('The pin value has changed to '..gpio.read(led))
    if toggle == 0 then gpio.write(led, gpio.HIGH) toggle = 1
    else gpio.write(led, gpio.LOW) toggle = 0
    end
end

gpio.mode(led, gpio.OUTPUT)
gpio.mode(pin, gpio.INPUT, gpio.PULLUP) 
gpio.trig(pin, 'up', debounce(onChange))
```
# OnlyFeathers
A ESP32-Cam birdhouse camera project
![OnlyfeathersLogo](/pic/Logo.png)

To learn some more about esp32 I wanted to setup a little birdnest cam.
I added [an external antenna](/Antenna.md) for better WiFi coverage as well.

![ProjectCollagePicture](/pic/IMG_2360.JPG)

I used an esp32-cam and [designed a mount](/ESP32-CAM%20custom%20mount.md) for it that also acts as a infrared light diffuser to light the birdhouse for pictures or video's.

For the esp32-cam I used the program [here](https://github.com/easytarget/esp32-cam-webserver/tree/master) which is an improved version of the default example program for esp32-cam webserver.
It works great and is better compared to the example program.

I used a [bright 940nm IR LED](/IR%20LED.md), which I connected to the original SMD flash LED that comes with the board. It was not hard to pry off the led and use a soldering iron and desolder thread to make it nice and clean again.

# finding it in several networks
I set it up to update the ip to a webserver that will receive the private IP address.
It was added because I configured multiple SSID's as redundancy which have their own subnets, this way I can always find it, regardless of the network it has connected to.

If it does not connect to WiFi, the first SSID set in the myconfig.h file will be setup as an accesspoint on the cam.


Change this codesnippet to make it update the IP on a webserver when it has internet connectivity.
```c++
// If we have connected, inform user
        if (WiFi.status() == WL_CONNECTED) {
            Serial.println("Client connection succeeded");
            accesspoint = false;
            // Note IP details
            ip = WiFi.localIP();
            net = WiFi.subnetMask();
            gw = WiFi.gatewayIP();
            Serial.printf("IP address: %d.%d.%d.%d\r\n",ip[0],ip[1],ip[2],ip[3]);
            Serial.printf("Netmask   : %d.%d.%d.%d\r\n",net[0],net[1],net[2],net[3]);
            Serial.printf("Gateway   : %d.%d.%d.%d\r\n",gw[0],gw[1],gw[2],gw[3]);
            calcURLs();
            // Flash the LED to show we are connected
            //for (int i = 0; i < 5; i++) {
            //    flashLED(50);
            //    delay(150);

            // log IP 
            ip = WiFi.localIP();

            String url = "https://YOURWEBSERVER/updateIPhere.php"; // Replace with your actual URL
            String postData = "ip=" + ip.toString(); // Adjust the parameter name as needed
            
            HTTPClient http;
            http.begin(url);
            http.addHeader("Content-Type", "application/x-www-form-urlencoded");
            int httpResponseCode = http.POST(postData);
            if (httpResponseCode == HTTP_CODE_OK) {
                Serial.println("IP address sent successfully");
            } else {
                Serial.printf("Error sending IP address (HTTP code %d)\n", httpResponseCode);
            }

           }
        else {
            Serial.println("Client connection Failed");
            WiFi.disconnect();   // (resets the WiFi scan)
        }
    }
```

You will need some php similar to this to receive the data:

```php
<?php
// thisistheipofthecam.php

// Check if the request method is POST
if ($_SERVER["REQUEST_METHOD"] === "POST") {
    // Retrieve the raw POST data (including all parameters)
    $rawData = file_get_contents("php://input");

    // Parse the raw data into an associative array
    parse_str($rawData, $postData);

    // Extract the relevant values (ip and timestamp)
    $ip = $postData["ip"];
    $timestamp = $postData["timestamp"];

    // Log the extracted data (you can customize the log file path)
    $logFilePath = "esp32-cam_IP.log";
    file_put_contents($logFilePath, "$timestamp $ip\n", FILE_APPEND);

    // Respond with a success message
    echo "Received data successfully";
} else {
    // Respond with an error message if the request method is not POST
    http_response_code(400); // Bad Request
    echo "Invalid request method";
}
?>
```
After it was put back on the shed, it had [a new inhabitant the next day](/First%20Visitor.md)!
Which makes it all worth the effort of course. 

A Raspberry Pi is used to take pictures every 20 seconds, which can be converted to a timelapse video.
For nightvision, it will turn on the LED and let the camera adjust itself to it for 4 seconds to take a picture and turn off the LED again. The script is scheduled in cron to execute every minute

I updated this in the original code to make it do that:

```bash
# Loop 3 times to perform the actions below.
for ((i=0;i<3;i++)); do

        # Turn on the infrared light before taking the screengrab
        curl "http://onlyfeathers.local/control?var=lamp&val=95"
        # Sleep to let the camera adjust itself
        sleep 4
        # Download an image from the new host URL and save it with a timestamped filename.
        wget http://onlyfeathers.local/capture -O "/home/onlyfeathers/timelapsepics/only_feathers_$(date +%Y-%m-%d_%H-%M-%S).jpg"
        # Turn off the infrared light again
        sleep 1
        curl "http://onlyfeathers.local/control?var=lamp&val=0"

        # Find the most recently downloaded image.
        latest=$(ls /home/onlyfeathers/timelapsepics/only_feathers_*.jpg -rt1 | tail -1)
        # Extract the filename without the extension for watermarking.
        Watermarktext=$(basename "$latest" .jpg | sed 's/only_feathers_//')
```

[Here are some video's](/Video.md) of the building process and [here are some pictures](/Pics.md) as well.

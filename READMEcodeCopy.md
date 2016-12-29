#include <Adafruit_NeoPixel.h>
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>

#define DEBUG //enable debug mode (serial out enabled on specific functions)

#define PIN            0 //sets the pin on which the neopixels are connected
#define NUMPIXELS      21 //defines the number of pixels in the strip
#define interval       50 //defines the delay interval between running the functions

Adafruit_NeoPixel pixels = Adafruit_NeoPixel(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);

uint32_t red = pixels.Color(255, 0, 0);
uint32_t blue = pixels.Color(0, 0, 255);
uint32_t green = pixels.Color(0, 255, 0);
uint32_t pixelColour;

float activeColor[] = {255, 0, 0};//sets the default color to red

boolean LEDstate[] = {false, false, false, false, true, false, false, false, false, false, false, false, false, false}; //saves the state of each of the functions
const char* ssid     = "SSID";
const char* password = "yourwifipassword";
String rawColor;  //raw color to differentiate if the color has changed
String rawBright; // raw brightness to differentiate if the brightness level has changed
IPAddress ip(192, 168, 1, 243); //static IP address
IPAddress gateway(192, 168, 1, 1); //gateway
IPAddress subnet(255, 255, 255, 0);//subnet

String color = "#ed07c7"; //default color for color wheel
int status = WL_IDLE_STATUS;

const char* html = // this is the webpage served to the client

  "<!DOCTYPE html><html><head><meta http-equiv='X-UA-Compatible' content='IE=Edge'><meta charset='utf-8'><title>Neopixel Color Picker</title><script type='text/javascript' src='https://ajax.googleapis.com/ajax/libs/jquery/1.11.3/jquery.js'></script><script type='text/javascript' src='http://gilbertsincobh.com/wp-content/plugins/revslider/js/farbtastic/farbtastic.js'></script><link rel='stylesheet' href='http://gilbertsincobh.com/wp-content/plugins/revslider/js/farbtastic/farbtastic.css' type='text/css' /><script type='text/javascript'>$(document).ready(function() {$('#demo').hide();$('#picker').farbtastic('#color');});"
  "function ResetWebpage(){if (window.location.href != 'http://#IPADDRESS/'){window.open ('http://#IPADDRESS/','_self',true)}};" //change the website value here to your static website
  "function myFunction(){document.getElementById('brightnessLevel').submit();}</script>"
  "<style type='text/css'>.bt{display:block;width:250px;height:100px;padding:10px;margin:10px;text-align:center;border-radius:5px;color:white;font-weight:bold;font-size:40px;text-decoration:none;}"
  "body{background:#fff;}"
  ".red{background:red;color:white;}"
  ".green{background:#0C0;color:white;}"
  ".blue{background:blue;color:white;}"
  ".white{background:white;color:black;border:1px solid black;}"
  ".off{background:#666;color:white;}.colorPicker{background:white;color:black;}"
  ".colorWipe{font-size:40px;  background:linear-gradient(to right, red, #0C0, blue);}"
  ".theatreChase{font-size:40px;background:linear-gradient(to right, red, black, red, black, #0C0, black, #0C0, black, blue, black, blue);}"
  ".rainbow{font-size:40px;background:red;background:linear-gradient(to right, red, orange, yellow, green, blue, indigo, violet,red, orange, yellow, green, blue, indigo, violet);}"
  ".rainbowCycle{font-size:40px;background:red;background:linear-gradient(to right, red, orange, yellow, green, blue, indigo, violet);}"
  ".rainbowChase{font-size:40px;background:red;background:linear-gradient(to right, red, black, orange, black, yellow, black, green, black, blue, black, indigo, black, violet);}"
  ".breathe{background-image: url('http://33.media.tumblr.com/tumblr_m24p94fUvU1r6aoq4o1_500.gif');background-repeat: no-repeat;background-position: center;background-color: #000000;}"
  ".cylon{background-image: url('http://i493.photobucket.com/albums/rr295/EmperorDarthSidious/Scanner_kitt.gif');background-repeat: no-repeat;background-position: center;background-color: #000000;}"
  ".heartbeat{background-image: url('http://megaicons.net/static/img/icons_sizes/10/371/128/heart-beat-icon.png');background-repeat: no-repeat;background-position: center;background-color: #000000;}"
  ".y{background:#EE0;height:100px;width:100px;border-radius:50px;}.b{background:#fff;height:100px;width:100px;border-radius:50px;}.a{font-size:35px;}td{vertical-align:middle;}</style></head>"
  "<body onload='ResetWebpage()'><table><tr><td width='100'><div class='TGT0'></div></td><td><a class='bt red' href='/L0?v=1'>Red</a></td><td><a class='bt colorWipe' href='/L5?v=1'>Color Wipe</a></td><td><div class='TGT5'></div></td></tr>"
  "<tr><td><div class='TGT1'></div></td><td><a class='bt green' href='/L1?v=1'>Green</a></td><td><a class='bt theatreChase' href='/L6?v=1'>Theatre Chase</a></td><td><div class='TGT6'></div></td></tr>"
  "<tr><td><div class='TGT2'></div></td><td><a class='bt blue' href='/L2?v=1'>Blue</a></td><td><a class='bt rainbow' href='/L7?v=1'>Rainbow</a></td><td><div class='TGT7'></div></td></tr>"
  "<tr><td><div class='TGT3'></div></td><td><a class='bt white' href='/L3?v=1'>White</a></td><td><a class='bt rainbowChase' href='/L8?v=1'>Rainbow Chase</a></td><td><div class='TGT8'></div></td></tr>"
  "<tr><td><div class='TGT9'></div></td><td><a class='bt cylon' href='/L9?v=1'>Cylon Chaser</a></td><td><a class='bt rainbowCycle' href='/L10?v=1'>Rainbow Cycle</a></td><td><div class='TGT_10'></div></td></tr>"
  "<tr><td><div class='TGT_11'></div></td><td><a class='bt breathe' href='/L11?v=1'>Breathe</a></td><td><a class='bt heartbeat' href='/L12?v=1'>Heartbeat</a></td><td><div class='TGT_12'></div></td></tr>"
  "<tr><td><div class='TGT4'></div></td><td><a class='bt off' href='/L4?v=1'>Off</a></td><td><form id='brightnessLevel'><input type='range' name='bright' max='255' min='0' value='#bright' onchange='myFunction()'class='bt off'></form></td></tr></table>"
  "<form name='color' value='#123456'><label for='color'><font color='white' style='padding-left:100px'>Color:</font></label><input type='text' id='color' name='color' value='#colors' /><input type='submit' id='colorPost' name='colorPost' value='SUBMIT' /></form><table><tr><td width = '100'><div class='TGT_13'></div></td> <td width = '400'><div id='picker'></div></td></tr></table></body></html>";


ESP8266WebServer server(80);

int neopixMode = 0; //sets a mode to run each of the functions
long previousMillis = 0; // a long value to store the millis()
int i = 0; //sets the pixel number in newTheatreChase() and newColorWipe()
int CWColor = 0; //sets the newColorWipe() color value 0=Red, 1=Green, 2=Blue
int j; //sets the pixel to skip in newTheatreChase() and newTheatreChaseRainbow()
int cycle = 0;//sets the cycle number in newTheatreChase()
int TCColor = 0;//sets the color in newTheatreChase()
int l = 0; //sets the color value to send to Wheel in newTheatreChaseRainbow() and newRainbow()
int m = 0; //sets the color value in newRainbowCycle()
int n = 2; //sets the pixel number in cyclonChaser()
int breather = 0; //sets the brightness value in breather()
boolean dir = true; //sets the direction in breather()-breathing in or out, and cylonChaser()-left or right
boolean beat = true; //sets the beat cycle in heartbeat()
int beats = 0; //sets the beat number in heartbeat()
int brightness = 150; //sets the default brightness value
String IP; //IP address converted from type IPAddress to String to replace in HTML
void setup() {

  pixels.begin(); //starts the neopixels
  pixels.setBrightness(brightness); // sets the inital brightness of the neopixels
  writeLEDS(0, 0, 0); //sets all the pixels to off
  Serial.begin(115200); //starts up Serial
  Serial.println("Connecting to WIFI AP...");

  WiFi.begin(ssid, password);//begin connection to wifi access point/router
  WiFi.config(ip, gateway, subnet); //sets the connection to static
  while (WiFi.status() != WL_CONNECTED) // Wait for connection
  {
    delay(500);//sets a delay to wait for the access point to create the connection
    Serial.println(".");//adds a dot for each half second that the connection is not established
  }
  Serial.print("Connected to "); //sends serial information about the connection
  Serial.println(ssid);
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
  IPAddress localIP = WiFi.localIP();
  IP = String(localIP[0]);
  IP += ".";
  IP += String(localIP[1]);
  IP += ".";
  IP += String(localIP[2]);
  IP += ".";
  IP += String(localIP[3]);

#if defined DEBUG
  Serial.print("IP as String: ");
  Serial.println(IP);
#endif

  server.on("/", handle_root); //when the page returned is "/" run the function 'handle_root()'
  server.on("/generate_204", handle_root);  //Android captive portal
  server.on("/L0", handle_L0); //when the page returned is "L0" run the function 'handle_L0' etc.
  server.on("/L1", handle_L1);
  server.on("/L2", handle_L2);
  server.on("/L3", handle_L3);
  server.on("/L4", handle_L4);
  server.on("/L5", handle_L5);
  server.on("/L6", handle_L6);
  server.on("/L7", handle_L7);
  server.on("/L8", handle_L8);
  server.on("/L9", handle_L9);
  server.on("/L10", handle_L10);
  server.on("/L11", handle_L11);
  server.on("/L12", handle_L12);
  server.onNotFound(handleNotFound); //when page not found
  server.begin();
  Serial.println("HTTP server started");

}

void loop() {

  server.handleClient(); //handle the webpage
  if (LEDstate[5] == true) // if the option has been selected keep running the function for that option
  {
    newColorWipe();
  }
  if (LEDstate[6] == true)
  {
    newTheatreChase();
  }
  if (LEDstate[7] == true)
  {
    newRainbow();
  }
  if (LEDstate[8] == true)
  {
    newTheatreChaseRainbow();
  }
  if (LEDstate[9] == true)
  {
    cylonChaser();
  }
  if (LEDstate[10] == true)
  {
    newRainbowCycle();
  }
  if (LEDstate[11] == true)
  {
    breathing();
  }
  if (LEDstate[12] == true)
  {
    heartbeat();
  }
}

void handle_root() {

#if defined DEBUG //if debug has been uncommented at the start of the script display the readouts below
  Serial.print("Args: "); //the number arguments in the webpage address
  Serial.println(server.args());
  Serial.print("ArgName1: ");//the name of the argument in the address
  Serial.println(server.argName(0));
  Serial.print("Arg1: "); //argument1
  Serial.println(server.arg(0));
  Serial.print("Arg2: ");//argument2
  Serial.println(server.arg(1));
  Serial.print("uri: ");//the webpage address
  Serial.println(server.uri());
  Serial.println("method");//the method - Post/Get
  Serial.println(server.method());
#endif
  if ((server.argName(0) == "color") && (server.arg(0) != rawColor)) //if the first argument name is "color" and it has changed from the last color
  {
    rawColor = server.arg(0);//save the argument data as the variable rawColor
    handle_color();//run the handle_color() function
  }
  if ((server.argName(0) == "bright") && (server.arg(0) != rawBright))//if the first argument is bright and it has changed from the last brightness level
  {
    rawBright = server.arg(0);
    handle_bright();
  }
#if defined DEBUG
  Serial.println("Page served");
#endif
  String toSend = html;

  toSend.replace("#IPADDRESS", IP); //replace the "#IPADDRESS" in the HTML above with the string IP
  toSend.replace("TGT0", LEDstate[0] ? "y" : "b"); //replace the style code "TGT0" with either y or b to indicate which function has been selected
  toSend.replace("TGT1", LEDstate[1] ? "y" : "b"); //same as above etc.
  toSend.replace("TGT2", LEDstate[2] ? "y" : "b");
  toSend.replace("TGT3", LEDstate[3] ? "y" : "b");
  toSend.replace("TGT4", LEDstate[4] ? "y" : "b");
  toSend.replace("TGT5", LEDstate[5] ? "y" : "b");
  toSend.replace("TGT6", LEDstate[6] ? "y" : "b");
  toSend.replace("TGT7", LEDstate[7] ? "y" : "b");
  toSend.replace("TGT8", LEDstate[8] ? "y" : "b");
  toSend.replace("TGT9", LEDstate[9] ? "y" : "b");
  toSend.replace("TGT_10", LEDstate[10] ? "y" : "b");
  toSend.replace("TGT_11", LEDstate[11] ? "y" : "b");
  toSend.replace("TGT_12", LEDstate[12] ? "y" : "b");
  toSend.replace("TGT_13", LEDstate[13] ? "y" : "b");
  toSend.replace("#colors", color); //replace "#colors" in the html with the color value
  toSend.replace("#bright", rawBright);//same as above with the brightness value
  server.send(200, "text/html", toSend); //send the html code to the client
  delay(500);//wait half a second after sending the data
}

void handleNotFound()
{
  Serial.print("\t\t\t\t URI Not Found: ");
  Serial.println(server.uri());
  server.send ( 200, "text/plain", "URI Not Found" );//send not found message

}

void handle_L0() {
  writeLEDS(255, 0, 0);//set the pixels to red
  activeColor[0] = 255;//sets the active color to red
  activeColor[1] = 0;
  activeColor[2] = 0;
  change_states(0);//change LEDstate to 0
  handle_root();//handle root again to send the html with changes
}

void handle_L1() {//same as hanle_L0 only with green
  writeLEDS(0, 255, 0);
  activeColor[0] = 0;
  activeColor[1] = 255;
  activeColor[2] = 0;
  change_states(1);
  handle_root();
}

void handle_L2() {//same as hanle_L0 only with blue
  writeLEDS(0, 0, 255);
  activeColor[0] = 0;
  activeColor[1] = 0;
  activeColor[2] = 255;
  change_states(2);
  handle_root();
}
void handle_L3() {//same as hanle_L0 only with white
  writeLEDS(255, 255, 255);
  activeColor[0] = 255;
  activeColor[1] = 255;
  activeColor[2] = 255;
  change_states(3);
  handle_root();
}

void handle_L4() {//same as hanle_L0 only turns off all pixels
  writeLEDS(0, 0, 0);
  change_states(4);
  handle_root();
}

void handle_L5() {
  change_states(5);//changes LEDstate to 5
  handle_root(); //resend the html with changes
  newColorWipe();//starts the function
}

void handle_L6() {//same as above etc.
  change_states(6);
  handle_root();
  newTheatreChase();
}

void handle_L7() {
  change_states(7);
  handle_root();
  newRainbow();
}

void handle_L8() {
  change_states(8);
  handle_root();
  newTheatreChaseRainbow();
}

void handle_L9() {
  change_states(9);
  handle_root();
  cylonChaser();
}

void handle_L10() {
  change_states(10);
  handle_root();
  newRainbowCycle();
}
void handle_L11() {
  change_states(11);
  handle_root();
  breathing();
}
void handle_L12() {
  change_states(12);
  handle_root();
  heartbeat();
}

void handle_color()
{
  color = server.arg(0);//sets the color1 string to the value in argument 0
#if defined DEBUG
  Serial.print("Color Received:");
  Serial.println(color);
#endif
  if (color.indexOf("%23") >= 0)//if the string has %23 which is "#"
  {
    color = "#";//add it in as a usable character
    color += color.substring((color.indexOf("%23") + 3));//add in the color code
  }
#if defined DEBUG
  Serial.print("Color: ");
  Serial.println(color);
#endif

  String r = "0x" + color.substring(1, 3);//sets string r to a string of the byte value
  String g = "0x" + color.substring(3, 5);//same as above
  String b = "0x" + color.substring(5, 7);

#if defined DEBUG
  String colors = r + g + b;//sets the whole string up for printing in debug mode
  Serial.print("colors - String: ");
  Serial.print(colors);
#endif

  const char *r1 = r.c_str(); //converts the string to a const char to convert to RGB values below
  const char *g1 = g.c_str(); //same as above
  const char *b1 = b.c_str();
  int red = RGBValue(r1);//converts to RGB value
  int green = RGBValue(g1);
  int blue = RGBValue(b1);
#if defined DEBUG
  Serial.print(" ");
  Serial.print(red);
  Serial.print(" ");
  Serial.print(green);
  Serial.print(" ");
  Serial.println(blue);
#endif
  writeLEDS(red, green, blue);//sets up the pixels to the color chosen
  activeColor[0] = red; //sets the color chosen as the activeColor to be used in other functions
  activeColor[1] = green;
  activeColor[2] = blue;
#if defined DEBUG
  Serial.println(activeColor[0]);
  Serial.println(activeColor[1]);
  Serial.println(activeColor[2]);
#endif



  for (int x = 0; x < 14; x++)
  {
    LEDstate[x] = LOW;//write all LEDstate values to LOW
  }
  LEDstate[13] = HIGH;//sets LEDstate to 13
  handle_root();//send the html code
}

void handle_bright()
{
  brightness = server.arg(0).toInt();//changes the argument0 to an int
  pixels.setBrightness(brightness);//sets it as the global brightness
  pixels.show();//displays new brightness level
}

void change_states(int tgt) {
  if (server.hasArg("v")) {//if there is a V argument
    int state = server.arg("v").toInt() == 1;//sets the state as an int value of the argument V
#if defined DEBUG
    Serial.print("LED");
    Serial.print(tgt);
    Serial.print("=");
    Serial.println(state);
#endif
    for (int x = 0; x < 14; x++)
    {
      LEDstate[x] = LOW;//writes all LEDstate values to LOW
    }
    LEDstate[tgt] = state ? HIGH : LOW; //sets the tgt value to HIGH

  }
}


unsigned int RGBValue(const char * s)//converts the value to an RGB value
{
  unsigned int result = 0;
  int c ;
  if ('0' == *s && 'x' == *(s + 1)) {
    s += 2;
    while (*s) {
      result = result << 4;
      if (c = (*s - '0'), (c >= 0 && c <= 9)) result |= c;
      else if (c = (*s - 'A'), (c >= 0 && c <= 5)) result |= (c + 10);
      else if (c = (*s - 'a'), (c >= 0 && c <= 5)) result |= (c + 10);
      else break;
      ++s;
    }
  }
  return result;
}

void heartbeat()
{
#if defined DEBUG
  Serial.print("testinterval");
  Serial.println(millis() - previousMillis);
#endif
  if (millis() - previousMillis > interval * 2)//if the time between the function being last run is greater than intervel * 2 - run it
  {
    if ((beat == true) && (beats == 0) && (millis() - previousMillis > interval * 7)) //if the beat is on and it's the first beat (beats==0) and the time between them is enough
    {
      for (int h = 50; h <= 255; h = h + 15)//turn on the pixels at 50 and bring it up to 255 in 15 level increments
      {
        writeLEDS((activeColor[0] / 255) * h, (activeColor[1] / 255) * h, (activeColor[2] / 255) * h);
        delay(3);
      }
      beat = false;//sets the next beat to off
      previousMillis = millis();//starts the timer again


    }
    else if ((beat == false) && (beats == 0))//if the beat is off and the beat cycle is still in the first beat
    {
      for (int h = 255; h >= 0; h = h - 15)//turn off the pixels
      {
        writeLEDS((activeColor[0] / 255) * h, (activeColor[1] / 255) * h, (activeColor[2] / 255) * h);
        delay(3);
      }
      beat = true;//sets the beat to On
      beats = 1;//sets the next beat to the second beat
      previousMillis = millis();
    }
    else if ((beat == true) && (beats == 1) && (millis() - previousMillis > interval * 2))//if the beat is on and it's the second beat and the interval is enough
    {
      for (int h = 50; h <= 255; h = h + 15)
      {
        writeLEDS((activeColor[0] / 255) * h, (activeColor[1] / 255) * h, (activeColor[2] / 255) * h); //turn on the pixels
        delay(3);
      }
      beat = false;//sets the next beat to off
      previousMillis = millis();
    }
    else if ((beat == false) && (beats == 1))//if the beat is off and it's the second beat
    {
      for (int h = 255; h >= 0; h = h - 15)
      {
        writeLEDS((activeColor[0] / 255) * h, (activeColor[1] / 255) * h, (activeColor[2] / 255) * h); //turn off the pixels
        delay(3);
      }
      beat = true;//sets the next beat to on
      beats = 0;//starts the sequence again
      previousMillis = millis();
    }
#if defined DEBUG
    Serial.print("previousMillis:");
    Serial.println(previousMillis);
#endif
  }
}
void breathing()
{
  if (millis() - previousMillis > interval * 2) //if the timer has reached its delay value
  {
    writeLEDS((activeColor[0] / 255) * breather, (activeColor[1] / 255) * breather, (activeColor[2] / 255) * breather); //write the leds to the color and brightness level
    if (dir == true)//if the lights are coming on
    {
      if (breather < 255)//once the value is less than 255
      {
        breather = breather + 15;//adds 15 to the brightness level for the next time
      }
      else if (breather >= 255)//if the brightness is greater or equal to 255
      {
        dir = false;//sets the direction to false
      }
    }
    if (dir == false)//if the lights are going off
    {
      if (breather > 0)
      {
        breather = breather - 15;//takes 15 away from the brightness level
      }
      else if (breather <= 0)//if the brightness level is nothing
        dir = true;//changes the direction again to on
    }
    previousMillis = millis();
  }
}

void cylonChaser()
{
  if (millis() - previousMillis > interval * 2 / 3)
  {
    for (int h = 0; h < pixels.numPixels(); h++)
    {
      pixels.setPixelColor(h, 0);//sets all pixels to off
    }
    if (pixels.numPixels() <= 10)//if the number of pixels in the strip is 10 or less only activate 3 leds in the strip
    {
      pixels.setPixelColor(n, pixels.Color(activeColor[0], activeColor[1], activeColor[2]));//sets the main pixel to full brightness
      pixels.setPixelColor(n + 1, pixels.Color((activeColor[0] / 255) * 50, (activeColor[1] / 255) * 50, (activeColor[2] / 255) * 50)); //sets the surrounding pixels brightness to 50
      pixels.setPixelColor(n - 1, pixels.Color((activeColor[0] / 255) * 50, (activeColor[1] / 255) * 50, (activeColor[2] / 255) * 50));
      if (dir == true)//if the pixels are going up in value
      {
        if (n <  (pixels.numPixels() - 1))//if the pixels are moving forward and havent reach the end of the strip "-1" to allow for the surrounding pixels
        {
          n++;//increase N ie move one more forward the next time
        }
        else if (n >= (pixels.numPixels() - 1))//if the pixels have reached the end of the strip
        {
          dir = false;//change the direction
        }
      }
      if (dir == false)//if the pixels are going down in value
      {
        if (n > 1)//if the pixel number is greater than 1 (to allow for the surrounding pixels)
        {
          n--; //decrease the active pixel number
        }
        else if (n <= 1)//if the pixel number has reached 1
        {
          dir = true;//change the direction
        }
      }
    }
    if ((pixels.numPixels() > 10) && (pixels.numPixels() <= 20))//if there are between 11 and 20 pixels in the strip add 2 pixels on either side of the main pixel
    {
      pixels.setPixelColor(n, pixels.Color(activeColor[0], activeColor[1], activeColor[2]));//same as above only with 2 pixels either side
      pixels.setPixelColor(n + 1, pixels.Color((activeColor[0] / 255) * 150, (activeColor[1] / 255) * 150, (activeColor[2] / 255) * 150));
      pixels.setPixelColor(n + 2, pixels.Color((activeColor[0] / 255) * 50, (activeColor[1] / 255) * 50, (activeColor[2] / 255) * 50));
      pixels.setPixelColor(n - 1, pixels.Color((activeColor[0] / 255) * 150, (activeColor[1] / 255) * 150, (activeColor[2] / 255) * 150));
      pixels.setPixelColor(n - 2, pixels.Color((activeColor[0] / 255) * 50, (activeColor[1] / 255) * 50, (activeColor[2] / 255) * 50));
      if (dir == true)
      {
        if (n <  (pixels.numPixels() - 2))
        {
          n++;
        }
        else if (n >= (pixels.numPixels() - 2))
        {
          dir = false;
        }
      }
      if (dir == false)
      {
        if (n > 2)
        {
          n--;
        }
        else if (n <= 2)
        {
          dir = true;
        }
      }
    }
    if (pixels.numPixels() > 20)//if there are more than 20 pixels in the strip add 3 pixels either side of the main pixel
    {
      pixels.setPixelColor(n, pixels.Color((activeColor[0] / 255) * 150, (activeColor[1] / 255) * 150, (activeColor[2] / 255) * 150));
      pixels.setPixelColor(n + 1, pixels.Color((activeColor[0] / 255) * 150, (activeColor[1] / 255) * 150, (activeColor[2] / 255) * 150));
      pixels.setPixelColor(n + 2, pixels.Color((activeColor[0] / 255) * 100, (activeColor[1] / 255) * 100, (activeColor[2] / 255) * 100));
      pixels.setPixelColor(n + 3, pixels.Color((activeColor[0] / 255) * 50, (activeColor[1] / 255) * 50, (activeColor[2] / 255) * 50));
      pixels.setPixelColor(n - 1, pixels.Color((activeColor[0] / 255) * 150, (activeColor[1] / 255) * 150, (activeColor[2] / 255) * 150));
      pixels.setPixelColor(n - 2, pixels.Color((activeColor[0] / 255) * 100, (activeColor[1] / 255) * 100, (activeColor[2] / 255) * 100));
      pixels.setPixelColor(n - 3, pixels.Color((activeColor[0] / 255) * 50, (activeColor[1] / 255) * 50, (activeColor[2] / 255) * 50));
      if (dir == true)
      {
        if (n <  (pixels.numPixels() - 3))
        {
          n++;
        }
        else if (n >= (pixels.numPixels() - 3))
        {
          dir = false;
        }
      }
      if (dir == false)
      {
        if (n > 3)
        {
          n--;
        }
        else if (n <= 3)
        {
          dir = true;
        }
      }
    }
    pixels.show();//show the pixels
    previousMillis = millis();
  }
}

void newTheatreChaseRainbow()
{
  if (millis() - previousMillis > interval * 2)
  {
    for (int h = 0; h < pixels.numPixels(); h = h + 3) {
      pixels.setPixelColor(h + (j - 1), 0);    //turn every third pixel off from the last cycle
    }
    for (int h = 0; h < pixels.numPixels(); h = h + 3)
    {
      pixels.setPixelColor(h + j, Wheel( ( h + l) % 255));//turn every third pixel on and cycle the color
    }
    pixels.show();
    j++;
    if (j >= 3)
      j = 0;
    l++;
    if (l >= 256)
      l = 0;
    previousMillis = millis();
  }
}


void newRainbowCycle()
{
  if (millis() - previousMillis > interval * 2)
  {
    for (int h = 0; h < pixels.numPixels(); h++)
    {
      pixels.setPixelColor(h, Wheel(((h * 256 / pixels.numPixels()) + m) & 255));
    }
    m++;
    if (m >= 256 * 5)
      m = 0;
    pixels.show();
    previousMillis = millis();
  }
}

void newRainbow()
{
  if (millis() - previousMillis > interval * 2)
  {
    for (int h = 0; h < pixels.numPixels(); h++)
    {
      pixels.setPixelColor(h, Wheel((h + l) & 255));
    }
    l++;
    if (l >= 256)
      l = 0;
    pixels.show();
    previousMillis = millis();
  }
}

void newTheatreChase()
{
  if (millis() - previousMillis > interval * 2)
  {
    uint32_t color;
    int k = j - 3;
    j = i;
    while (k >= 0)
    {
      pixels.setPixelColor(k, 0);
      k = k - 3;
    }
    if (TCColor == 0)
    {
      color = pixels.Color(255, 0, 0);
    }
    else if (TCColor == 1)
    {
      color = pixels.Color(0, 255, 0);
    }
    else if (TCColor == 2)
    {
      color = pixels.Color(0, 0, 255);
    }
    else if (TCColor == 3)
    {
      color = pixels.Color(255, 255, 255);
    }
    while (j < NUMPIXELS)
    {
      pixels.setPixelColor(j, color);
      j = j + 3;
    }
    pixels.show();
    if (cycle == 10)
    {
      TCColor ++;
      cycle = 0;
      if (TCColor == 4)
        TCColor = 0;
    }
    i++;
    if (i >= 3)
    {
      i = 0;
      cycle ++;
    }
    previousMillis = millis();
  }
}



void newColorWipe()
{
  if (millis() - previousMillis > interval * 2)
  {
    uint32_t color;
    if (CWColor == 0)
    {
      color = pixels.Color(255, 0, 0);
    }
    else if (CWColor == 1)
    {
      color = pixels.Color(0, 255, 0);
    }
    else if (CWColor == 2)
    {
      color = pixels.Color(0, 0, 255);
    }
    pixels.setPixelColor(i, color);
    pixels.show();
    i++;
    if (i == NUMPIXELS)
    {
      i = 0;
      CWColor++;
      if (CWColor == 3)
        CWColor = 0;
    }
    previousMillis = millis();
  }
}
uint32_t Wheel(byte WheelPos) {
  WheelPos = 255 - WheelPos;
  if (WheelPos < 85) {
    return pixels.Color(255 - WheelPos * 3, 0, WheelPos * 3);
  }
  if (WheelPos < 170) {
    WheelPos -= 85;
    return pixels.Color(0, WheelPos * 3, 255 - WheelPos * 3);
  }
  WheelPos -= 170;
  return pixels.Color(WheelPos * 3, 255 - WheelPos * 3, 0);
}

void writeLEDS(byte R, byte G, byte B)
{
  for (int i = 0; i < pixels.numPixels(); i ++)
  {
    pixels.setPixelColor(i, pixels.Color(R, G, B));
  }
  pixels.show();
}

void writeLEDS(byte R, byte G, byte B, byte bright)
{
  float fR = (R / 255) * bright;
  float fG = (G / 255) * bright;
  float fB = (B / 255) * bright;
  for (int i = 0; i < pixels.numPixels(); i ++)
  {
    pixels.setPixelColor(i, pixels.Color(R, G, B));
  }
  pixels.show();
}

void writeLEDS(byte R, byte G, byte B, byte bright, byte LED)
{
  float fR = (R / 255) * bright;
  float fG = (G / 255) * bright;
  float fB = (B / 255) * bright;
  pixels.setPixelColor(LED, pixels.Color(R, G, B));
  pixels.show();
}

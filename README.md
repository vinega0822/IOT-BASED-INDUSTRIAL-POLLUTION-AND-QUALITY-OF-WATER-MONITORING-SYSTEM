# IOT-BASED-INDUSTRIAL-POLLUTION-AND-QUALITY-OF-WATER-MONITORING-SYSTEM
code for Industrial pollution and quality of water monitoring system
#include "MQ135.h"

#include <SoftwareSerial.h>

#define DEBUG true

SoftwareSerial esp8266(9,10); // for Bluetooth - create an object. This makes pin 9 of Arduino as RX pin and pin 10 of Arduino as the TX pin

const int sensorPin= 0;

int air_quality;

#include <LiquidCrystal.h> 

LiquidCrystal lcd(12,11, 5, 4, 3, 2);


void setup() {

pinMode(8, OUTPUT);

lcd.begin(16,2);

lcd.setCursor (0,0);

lcd.print ("circuitdigest ");

lcd.setCursor (0,1);

lcd.print ("Sensor Warming ");

delay(1000);

Serial.begin(115200);

esp8266.begin(115200); // your esp's baud rate might be different

  sendData("AT+RST\r\n",2000,DEBUG); // reset module

  sendData("AT+CWMODE=2\r\n",1000,DEBUG); // configure as access point

  sendData("AT+CIFSR\r\n",1000,DEBUG); // get ip address

  sendData("AT+CIPMUair_quality=1\r\n",1000,DEBUG); // configure for multiple connections

  sendData("AT+CIPSERVER=1,80\r\n",1000,DEBUG); // turn on server on port 80

pinMode(sensorPin, INPUT); //Gas sensor will be an input to the arduino

lcd.clear();

}

float reads;
int pin = A0;

float vOut = 0 ;//voltage drop across 2 points
float vIn = 5;
float R1 = 1000;
float R2 = 0;
float buffer = 0;
float TDS;

float R = 0;//resistance between the 2 wires
float r = 0;//resistivity 
float L = 0.06;//distance between the wires in m
double A = 0.000154;//area of cross section of wire in m^2

float C = 0;//conductivity in S/m
float Cm = 0;//conductivity in mS/cm

int rPin = 9;
int bPin = 5;
int gPin = 6;
int rVal = 255;
int bVal = 255;
int gVal = 255;

//we will use this formula to get the resistivity after using ohm's law -> R = r L/A => r = R A/L

//creating lcd object from Liquid Crystal library
LiquidCrystal lcd(7,8,10,11,12,13);

void setup() {
  //initialise BT serial and serial monitor
  Serial.begin(9600);
  BTserial.begin(9600);

  //initialise lcd
  lcd.begin(16, 2);

  //set rgb led pins (all to be pwm pins on Arduino) as output
  pinMode(rPin,OUTPUT);
  pinMode(bPin,OUTPUT);
  pinMode(gPin,OUTPUT);
  pinMode(pin,INPUT);
  //Print stagnant message to LCD
  lcd.print("Conductivity: ");
}

void loop() {
    reads = analogRead(A0);
  
  vOut = reads*5/1023;
  Serial.println(reads);
// Serial.println(vOut);
  buffer = (vIn/vOut)-1;
  R2 = R1*buffer;
  Serial.println(R2);
  delay(500);
    //convert voltage to resistance
    //Apply formula mentioned above
      r = R2*A/L;//R=rL/A
    //convert resistivity to condictivity
    C = 1/r;
    Cm = C*10;
    //convert conductivity in mS/cm to TDS
    TDS = Cm *700;
    //Set cursor of LCD to next row
    lcd.setCursor(0,1);
    lcd.println(C);

    //display corresponding colours on rgb led according to the analog read
     if( reads < 600 )
  {
      if (reads <= 300){
        setColor( 255, 0, 255 ) ;
      }
      if (reads > 200){
        setColor( 200, 0, 255 ) ;
      }
  }
  else{
    if( reads <= 900 )
    {
      setColor( 0, 0, 255 ) ;
    }
    if( reads > 700 )
  {
    setColor( 0, 255, 255 ) ;
  }
    }

//send data to Ardutooth app on mobile phone through bluetooth
BTserial.print(C);
BTserial.print(",");
BTserial.print(TDS);
BTserial.print(";");
delay(500);
}


void setColor(int red, int green, int blue)
{
  analogWrite( rPin, 255 - red ) ;
  analogWrite( gPin, 255 - green ) ;
  analogWrite( bPin, 255 - blue ) ;  
}


void loop() {


MQ135 gasSensor = MQ135(A0);

float air_quality = gasSensor.getPPM();


if(esp8266.available()) // check if the esp is sending a message 

  {

    if(esp8266.find("+IPD,"))

    {

     delay(1000);

     int connectionId = esp8266.read()-48; /* We are subtracting 48 from the output because the read() function returns the ASCII decimal value and the first decimal number which is 0 starts at 48*/ 

     String webpage = "<h1>IOT Air Pollution Monitoring System</h1>";

       webpage += "<p><h2>";   

       webpage+= " Air Quality is ";

       webpage+= air_quality;

       webpage+=" PPM";

       webpage += "<p>";

     if (air_quality<=1000)

{

  webpage+= "Fresh Air";

}

else if(air_quality<=2000 && air_quality>=1000)

{

  webpage+= "Poor Air";

}


else if (air_quality>=2000 )

{

webpage+= "Danger! Move to Fresh Air";

}


webpage += "</h2></p></body>"; 

     String cipSend = "AT+CIPSEND=";

     cipSend += connectionId;

     cipSend += ",";

     cipSend +=webpage.length();

     cipSend +="\r\n";

     

     sendData(cipSend,1000,DEBUG);

     sendData(webpage,1000,DEBUG);

     

     cipSend = "AT+CIPSEND=";

     cipSend += connectionId;

     cipSend += ",";

     cipSend +=webpage.length();

     cipSend +="\r\n";

     

     String closeCommand = "AT+CIPCLOSE="; 

     closeCommand+=connectionId; // append connection id

     closeCommand+="\r\n";

     

     sendData(closeCommand,3000,DEBUG);

    }

  }


lcd.setCursor (0, 0);

lcd.print ("Air Quality is ");

lcd.print (air_quality);

lcd.print (" PPM ");

lcd.setCursor (0,1);

if (air_quality<=1000)

{

lcd.print("Fresh Air");

digitalWrite(8, LOW);

}

else if( air_quality>=1000 && air_quality<=2000 )

{

lcd.print("Poor Air, Open Windows");

digitalWrite(8, HIGH );

}

else if (air_quality>=2000 )

{

lcd.print("Danger! Move to Fresh Air");

digitalWrite(8, HIGH); // turn the LED on

}

lcd.scrollDisplayLeft();

delay(1000);

}

String sendData(String command, const int timeout, boolean debug)

{

    String response = ""; 

    esp8266.print(command); // send the read character to the esp8266

    long int time = millis();

    while( (time+timeout) > millis())

    {

      while(esp8266.available())

      {

        // The esp has data so display its output to the serial window 

        char c = esp8266.read(); // read the next character.

        response+=c;

      }  

    }

    if(debug)

    {

      Serial.print(response);

    }

    return response;

}

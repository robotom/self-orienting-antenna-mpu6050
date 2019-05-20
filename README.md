# self-orienting-antenna-mpu6050
GIT Blurb: 
Prototype software to orient antenna vertically no matter the orientation of the body (180deg servo limit) to which it is connected.
MPU6050 is the main GYRO/Accelerometer hardware.

VIEW ----> TEST.MOV for operational prototype. 
# NEED TO UPLOAD THIS
_____________________________________________________________________________

Original framework for Wire/i2c developed by JohnChi on August 17, 2014 = public domain. 
I expanded this code to suit my prototype on 25-Apr-2019.

_____________________________________________________________________________

MPU6050 WIRING:
VIN to 5v. Some MPU6050 modules use 3.3 but this board seems to be tolerant. 
GND to GND.
SDA & SCL go to the dedicated pins on the Mega which is what I am testing on; but on other boards this is usually analog 4 and 5.
INT should go to interrupt --> on the Mega this is D2,3,18, 19, 20, 21. Can be left broken.
_____________________________________________________________________________

//  I've opted not to use the MPU6050 library, I was getting an error which timed it out for whatever reason so this is the more reliable method. I suspect there was something wrong with the library.

_____________________________________________________________________________

IT IS IMPORTANT TO NOTE THAT THIS CODE IS USING A MICRO SERVO THAT IS LIMITED TO 180 DEGREES OF MOTION.
SO ALL OF THE CODE IS BASED AROUND THIS LIMITATION. 
IF I HAD A STEPPER OR CONTINUOUS SERVO ON HAND I WOULD HAVE ADJUSTED THE CODE ACCORDINGLY; WHICH ISN'T THAT DIFFICULT TO IMPLEMENT. 

_____________________________________________________________________________

# CODE BEGINS BELOW:

//MPU6050
#include<Wire.h>
const int MPU_addr=0x68;  // I2C address of the MPU-6050
int16_t AcX,AcY,AcZ,Tmp,GyX,GyY,GyZ;

//SERVO
#include <Servo.h>
Servo microservo1;
int servo_pin = 12; //pick your microcontroller pin

//CORRECTED ARCTAN2 VALUES
float corrected_angle; //this is necessary because the value that are mapped are not linear... they operate on a sine/cosine wave, so it'll only be correct at 0,90,etc. Everything in between will not map to servo properly.

//MAPPED VALUES
int16_t AcXmapped, AcYmapped, AcZmapped;
float AcCorrectedMapped;

void setup(){

  //SERVO SETUP
  microservo1.attach ( servo_pin );
  
  //WIRE SETUP
  Wire.begin();
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x6B);  // PWR_MGMT_1 register
  Wire.write(0);     // set to zero (wakes up the MPU-6050)
  Wire.endTransmission(true);
  Serial.begin(9600);
  Serial.println("Transmission Line Initializing In:"); 
  delay(150);
  Serial.println("3");
  delay(1000);
  Serial.println("2");
  delay(1000);
  Serial.println("1");
  delay(1000);

}

void loop(){

  //WIRE LOOP
  Wire.beginTransmission(MPU_addr); 
  Wire.write(0x3B);  // starting with register 0x3B (ACCEL_XOUT_H)
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_addr,14,true);  // request a total of 14 registers
  AcX=Wire.read()<<8|Wire.read();  // 0x3B (ACCEL_XOUT_H) & 0x3C (ACCEL_XOUT_L)    
  AcY=Wire.read()<<8|Wire.read();  // 0x3D (ACCEL_YOUT_H) & 0x3E (ACCEL_YOUT_L)
  AcZ=Wire.read()<<8|Wire.read();  // 0x3F (ACCEL_ZOUT_H) & 0x40 (ACCEL_ZOUT_L)
  Tmp=Wire.read()<<8|Wire.read();  // 0x41 (TEMP_OUT_H) & 0x42 (TEMP_OUT_L)
  GyX=Wire.read()<<8|Wire.read();  // 0x43 (GYRO_XOUT_H) & 0x44 (GYRO_XOUT_L)
  GyY=Wire.read()<<8|Wire.read();  // 0x45 (GYRO_YOUT_H) & 0x46 (GYRO_YOUT_L)
  GyZ=Wire.read()<<8|Wire.read();  // 0x47 (GYRO_ZOUT_H) & 0x48 (GYRO_ZOUT_L)
 
  //Serial.print(" | AcX = "); Serial.print(AcX);
  //Serial.print(" | AcY = "); Serial.print(AcY);
  //Serial.print(" | AcZ = "); Serial.println(AcZ);

  //Serial.print(AcY); Serial.print(","); //used these to test on the Serial Plotter (not monitor... take note)
  //Serial.println(AcZ);

  
  //Serial.print(" | Tmp =  "); Serial.print(Tmp/340.00+36.53);  //equation for temperature in degrees C from datasheet
  //Serial.print(" | GyX = "); Serial.print(GyX);
  //Serial.print(" | GyY = "); Serial.print(GyY);
  //Serial.print(" | GyZ = "); Serial.println(GyZ);

  //this is the corrected angle (see top of code doc) ---- calculate the angle using trig ---- the relationship is not linear so you have to adjust for that (it's actually a true trig wave)
  corrected_angle = atan2f(AcY,AcZ);   // Node angle in radian [ -pi , pi ]

  Serial.println(corrected_angle);

  //MAPPING (v2) the values from e.g. AcX to AcXmapped so they're better values for servo control.
  //set the AcX inside the map to negative if you want to invert the rotation of the servo.
  AcCorrectedMapped = AcCorrectedMapped *0.8f + (-corrected_angle*180.0f / M_PI + 90.0f) * 0.2f;   // Servo output angle in [ deg ] // added a filter to fix the jerky nature of the servo

  
  //MAPPING (v1) the values from e.g. AcX to AcXmapped so they're better values for servo control.
  //set the AcX inside the map to negative if you want to invert the rotation of the servo.
  //AcXmapped = map (AcX, -17000, 17000, 0, 180); //ROLL
  //AcYmapped = map (-AcY, -17000, 17000, 0, 180); //PITCH
  //AcZmapped = map (AcZ, -17000, 17000, 0, 180); //YAW

   //SERVO LOOP
  microservo1.write(AcCorrectedMapped); 
  //50 is usually nice and safe -- depending on how small you set the time delay between 
  //readings and thus rotation you'll get more frequent jitters but finer control.
  delay(150); 
   
}
  

  
    

The Gesture Detector
uses the TensorFlowLite library along with the Arduino microcontroller to create a ML model that detects different types of hand gestures and prints it out what the model thinks the gesture is.

You should comment out all portions of your portfolio that you have not completed yet, as well as any instructions:
```HTML 
<!--- This is an HTML comment in Markdown -->
<!--- Anything between these symbols will not render on the published site -->
```

| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Ethan T | Homestead High | Electrical Engineering | Incoming Sophomore

**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.**

![Headstone Image](logo.svg)

# Second Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/y3VAmNlER5Y" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For my second milestone, it was modifications that added to my base project. This time, I added two more gestures, mew and wave. I also added more training data to make the detection more accurate, which it previously wasn't. Lastly, instead of outputting the gesture in plain text, I used emojis to make it more fun. During these modifications, I did encounter some issues, namely the Arduino microcontroller spitting out random gibberish text and also crashing my computer whenever I would plug it into my computer with the USB-cable. I fixed it by uploading a new sketch into the microcontroller, which erased the previous activity.

# First Milestone

<iframe width="560" height="315" src="[https://www.youtube.com/embed/CaCazFBhYKs](https://www.youtube.com/watch?v=-SF8K8FDyk4)" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For my first milestone, I finished the overall project, where it can straight detect the gesture and print it out in the console. It works by having someone hold the microcontroller and then performing a gesture, to which the microcontroller sends the data to the computer which stores the model, which then predicts which gesture was executed. I faced some problems, such as trying to copy and pate the training data into a .csv file, which was very tedious since you had to manually copy and paste by sections instead of all at once. Another one was the .csv files not uploading correctly in Google Colab, which was because I forgot to save the files. Overall, although the project works, the predictions are not the most accurate. So for my next milestones and final project in general, I want to train the model with more data, as well as adding more gesture options for functionality purposes. Also, I plan to turn the gestures into emojis instead just bare text printed in the console to make it more appealing.

# Schematics 
Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser. 

# Code
Here's where you'll put your code. The syntax below places it into a block of code. Follow the guide [here]([url](https://www.markdownguide.org/extended-syntax/)) to learn how to customize it to your project needs. 

```
#include "Arduino_BMI270_BMM150.h"


#include <TensorFlowLite.h>
#include <tensorflow/lite/micro/all_ops_resolver.h>
#include <tensorflow/lite/micro/tflite_bridge/micro_error_reporter.h>
#include <tensorflow/lite/micro/micro_interpreter.h>
#include <tensorflow/lite/schema/schema_generated.h>
// #include <tensorflow/lite/version.h>


#include "model.h"

#include <PluggableUSBHID.h>
#include <USBKeyboard.h>

// Select an OS:
#define MACOS // You'll need to enable and select the unicode keyboard: System Preferences -> Input Sources -> + -> Others -> Unicode Hex Input
//#define LINUX

#if !defined(MACOS) && !defined(LINUX)
#error "Please select an OS!"
#endif

// use table: https://apps.timwhitlock.info/emoji/tables/unicode
const int bicep = 0x1f4aa;
const int punch = 0x1f44a;
const int wave = 0x1F44B;
const int mew = 0x1F92B;

USBKeyboard keyboard; 

const float accelerationThreshold = 2.5; // threshold of significant in G's
const int numSamples = 119;


int samplesRead = numSamples;


// global variables used for TensorFlow Lite (Micro)
tflite::MicroErrorReporter tflErrorReporter;


// pull in all the TFLM ops, you can remove this line and
// only pull in the TFLM ops you need, if would like to reduce
// the compiled size of the sketch.
tflite::AllOpsResolver tflOpsResolver;


const tflite::Model* tflModel = nullptr;
tflite::MicroInterpreter* tflInterpreter = nullptr;
TfLiteTensor* tflInputTensor = nullptr;
TfLiteTensor* tflOutputTensor = nullptr;


// Create a static memory buffer for TFLM, the size may need to
// be adjusted based on the model you are using
constexpr int tensorArenaSize = 8 * 1024;
byte tensorArena[tensorArenaSize] __attribute__((aligned(16)));


// array to map gesture index to a name
const char* GESTURES[] = {
 "punch",
 "mew",
 "wave",
 "flex"
};


#define NUM_GESTURES (sizeof(GESTURES) / sizeof(GESTURES[0]))


void setup() {
 Serial.begin(9600);
 while (!Serial);


 // initialize the IMU
 if (!IMU.begin()) {
   Serial.println("Failed to initialize IMU!");
   while (1);
 }


 // print out the samples rates of the IMUs
 Serial.print("Accelerometer sample rate = ");
 Serial.print(IMU.accelerationSampleRate());
 Serial.println(" Hz");
 Serial.print("Gyroscope sample rate = ");
 Serial.print(IMU.gyroscopeSampleRate());
 Serial.println(" Hz");


 Serial.println();


 // get the TFL representation of the model byte array
 tflModel = tflite::GetModel(model);
 if (tflModel->version() != TFLITE_SCHEMA_VERSION) {
   Serial.println("Model schema mismatch!");
   while (1);
 }


 // Create an interpreter to run the model
 tflInterpreter = new tflite::MicroInterpreter(tflModel, tflOpsResolver, tensorArena, tensorArenaSize);


 // Allocate memory for the model's input and output tensors
 tflInterpreter->AllocateTensors();


 // Get pointers for the model's input and output tensors
 tflInputTensor = tflInterpreter->input(0);
 tflOutputTensor = tflInterpreter->output(0);
}


void loop() {
 float aX, aY, aZ, gX, gY, gZ;


 // wait for significant motion
 while (samplesRead == numSamples) {
   if (IMU.accelerationAvailable()) {
     // read the acceleration data
     IMU.readAcceleration(aX, aY, aZ);


     // sum up the absolutes
     float aSum = fabs(aX) + fabs(aY) + fabs(aZ);


     // check if it's above the threshold
     if (aSum >= accelerationThreshold) {
       // reset the sample read count
       samplesRead = 0;
       break;
     }
   }
 }


 // check if the all the required samples have been read since
 // the last time the significant motion was detected
 while (samplesRead < numSamples) {
   // check if new acceleration AND gyroscope data is available
   if (IMU.accelerationAvailable() && IMU.gyroscopeAvailable()) {
     // read the acceleration and gyroscope data
     IMU.readAcceleration(aX, aY, aZ);
     IMU.readGyroscope(gX, gY, gZ);


     // normalize the IMU data between 0 to 1 and store in the model's
     // input tensor
     tflInputTensor->data.f[samplesRead * 6 + 0] = (aX + 4.0) / 8.0;
     tflInputTensor->data.f[samplesRead * 6 + 1] = (aY + 4.0) / 8.0;
     tflInputTensor->data.f[samplesRead * 6 + 2] = (aZ + 4.0) / 8.0;
     tflInputTensor->data.f[samplesRead * 6 + 3] = (gX + 2000.0) / 4000.0;
     tflInputTensor->data.f[samplesRead * 6 + 4] = (gY + 2000.0) / 4000.0;
     tflInputTensor->data.f[samplesRead * 6 + 5] = (gZ + 2000.0) / 4000.0;


     samplesRead++;


     if (samplesRead == numSamples) {
       // Run inferencing
       TfLiteStatus invokeStatus = tflInterpreter->Invoke();
       if (invokeStatus != kTfLiteOk) {
         Serial.println("Invoke failed!");
         while (1);
         return;
       }


       // Loop through the output tensor values from the model
       for (int i = 0; i < NUM_GESTURES; i++) {
         Serial.print(GESTURES[i]);
         Serial.print(": ");
         Serial.println(tflOutputTensor->data.f[i], 6);
         if(GESTURES[i]=="flex"){
          if(tflOutputTensor->data.f[i] > 0.9) {
            sentUtf8(bicep);
            Serial.println("we are flex");
          }
}

         if(GESTURES[i]=="punch"){
          if(tflOutputTensor->data.f[i] > 0.9) {
            sentUtf8(punch);
            Serial.println("we are punch");
          }
}

         if(GESTURES[i]=="wave"){
          if(tflOutputTensor->data.f[i] > 0.9) {
            sentUtf8(wave);
            Serial.println("we are wave");
          }
}

         if(GESTURES[i]=="mew"){
          if(tflOutputTensor->data.f[i] > 0.9) {
            sentUtf8(mew);
            Serial.println("we are mew");
          }
}
       }
       Serial.println();
     }
   }
 }
}

void sentUtf8(unsigned long c) {
  String s;

#if defined(MACOS)
  // https://apple.stackexchange.com/questions/183045/how-can-i-type-unicode-characters-without-using-the-mouse

  s = String(utf8ToUtf16(c), HEX);

  for (int i = 0; i < s.length(); i++) {
    keyboard.key_code(s[i], KEY_ALT);
  }
#elif defined(LINUX)
  s = String(c, HEX);

  keyboard.key_code('u', KEY_CTRL | KEY_SHIFT);

  for (int i = 0; i < s.length(); i++) {
    keyboard.key_code(s[i]);
  }
#endif
  keyboard.key_code(' ');
}

// based on https://stackoverflow.com/a/6240819/2020087
unsigned long utf8ToUtf16(unsigned long in) {
  unsigned long result;

  in -= 0x10000;

  result |= (in & 0x3ff);
  result |= (in << 6) & 0x03ff0000;
  result |= 0xd800dc00;

  return result;
}
```

# Bill of Materials

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Arduino Nano 33 BLE Sense Board | Used to detect the gestures | $27 | <a href="https://www.amazon.com/Arduino-Nano-Rev2-headers-ABX00072/dp/B0CNTRTL5W/ref=asc_df_B0CNTRTL5W?mcid=3df758d783473a4397b4583f9b901fe5&hvocijid=10140926521408242730-B0CNTRTL5W-&hvexpln=73&tag=hyprod-20&linkCode=df0&hvadid=721245378154&hvpos=&hvnetw=g&hvrand=10140926521408242730&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9198079&hvtargid=pla-2281435179018&psc=1"> Link </a> |
| RAMPOW Charging Cable | Connect the Arduino microcontroller to my computer | $6.99 | <a href="https://www.amazon.com/RAMPOW-Android-Charging-Braided-Samsung/dp/B01GJC4YMC?th=1"> Link </a> |

# Other Resources/Examples
One of the best parts about Github is that you can view how other people set up their own work. Here are some past BSE portfolios that are awesome examples. You can view how they set up their portfolio, and you can view their index.md files to understand how they implemented different portfolio components.
- [Example 1](https://docs.arduino.cc/tutorials/nano-33-ble-sense-rev2/get-started-with-machine-learning/)
- [Example 2](https://github.com/arduino/ArduinoTensorFlowLiteTutorials/blob/master/GestureToEmoji/ArduinoSketches/Emoji_Button/Emoji_Button.ino)

To watch the BSE tutorial on how to create a portfolio, click here.

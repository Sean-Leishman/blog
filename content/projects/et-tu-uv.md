---
author: "Sean Leishman"
title: "Et Tu UV: Designing Wearables"
date: "2023-07-03"
tags: ["Project", "IoT", "CroudSouring", "Web Design", "Python", "Flask"]
ShowBreadCrumbs: true
ShowToc: false
weight: 1
---

> Source code can be found at {{< newtabref href="https://github.com/Sean-Leishman/et-tu-uv" title="Github">}}

{{< figure align=center width=80% height=auto  src="../../fullsize.png" >}}

This project was created as part of the {{< newtabref href="https://x.smu.edu.sg/" title="SMU-X">}} program that I participated in during my exchange year. The main characteristic of this program is that students work with some organisation to deliver a project that closely mimics the requirements and ideologies of a real-world project. This project was part of an Internet-of-Things course that focussed on developing physical, internet-enabled devices for everyday social problems and investigating the viability and effect of those solutions. The course covered the process from the design to the testing phase of various IoT projects and solutions.

## What is Et-Tu-UV?

As part of the requirements, the task was set to devise a solution to aid user's in a health focussed arena. And, with inspiration from Singapore, we focussed on deterring the adverse effects of overexposure to both UV and other heat-related elements such as temperature and humidity. In our plan, there was an inherient focus on collecting data and thus using that data to generate insights for the final end-user.

The final product that was chosen, was an Arduino-based wearable that could communicate with our server which collected and transformed data for display on a custom dashboard.

## Building the Wearble

{{< figure align=center width=80% height=auto  src="../../wearable.jpg" >}}

Generally, there was an approach of quickly slamming together components to form a final product. Here there is the {{< newtabref href="https://www.adafruit.com/product/659" title="Adafruit Flora" >}} module, coupled with a bluetooth component, a UV/temperature/humidity sensor along with a buzzer. These components combined would allow us to communicate wirelessly with a phone and as such with the final web-app.

## Architecture

{{< figure align=center width=80% height=auto  src="../../ettuuv-arch.png" >}}

The final solution involved overcoming numerouse engineering challenges to allow communication between various different components, effectively and within a fairly short time frame. The main roadblock occured within the phone and arduino module as the chosen method of communication was bluetooth and as such, data was fairly easily sent out of the bluetooth module but then the phone itself had to send the data to a server. The
{{< newtabref href="https://play.google.com/store/apps/details?id=com.adafruit.bluefruit.le.connect&hl=en_GB&gl=US" title="app">}}, was not made ourselves and as such we were restricted by the chosen method.

{{<highlight cpp>}}
if (currentMillis - previousSendMillis >= sendInterval) {

    previousSendMillis += sendInterval;

    UVindex = uv.readUV();
    UVindex /= 100.0;

    dtostrf(UVindex, 0, 5, inputs);
    // the index is multiplied by 100 so to get the
    // integer index, divide by 100!

    // Sensor readings may also be up to 2 seconds 'old' (its a very slow sensor)
    relativeHumidity = dht.readHumidity();
    // Read temperature as Celsius (the default)
    temperature = dht.readTemperature();

    strcat(inputs, "A");
    dtostrf(temperature, 0, 2, &inputs[strlen(inputs)]);
    strcat(inputs, "A");
    dtostrf(relativeHumidity, 0, 2, &inputs[strlen(inputs)]);
    strcat(inputs, "A");

    //ble.println(UVindex);
    //Serial.println(typeof UVindex)
    ble.print("AT+BLEUARTTX=");
    ble.println(inputs);

}

// Check for incoming characters from Bluefruit
ble.println("AT+BLEUARTRX");
ble.readline();
if (strcmp(ble.buffer, "OK") == 0) {
// no data
return;
}
// Some data was found, its in the buffer
Serial.print(F("[Recv] "));
Serial.println(ble.buffer);
ble.waitForOK();

int chimeResult = shouldChime(ble.buffer);
chimeResult = -1;
{{</highlight>}}

An output was created and sent via BLE containing (Temp., Humidity, UV) within a singular string into the phone which in turn sent results via MQTT to the final server which processed the results.

## Server

The server contained various functionalities in order to generate the appropriate insights for the web app to display.

### ML Methods

Some machine learning methods were employed in order to provide some practical, predictive quality to the insights provided. These included the prediction of various metrics such as the time of the day with typically the highest exposure to UV radiation and also analying the amount of UV received by the user during a given day. By tracking these metrics, relevant and informative insights could be produced to the end-user.

### Alerts

The alert system allows the user to activate custom alerts that warned the user to some combination of metrics that can be defined by the user and sent to the user via a Telegram bot. The alert system is effective as we allowed for the muting, deletion and insertion of alerts as readily as the user would want to.

## Front-End

{{< figure align=center width=80% height=auto  src="../../timeseries.png" >}}

In order, to achieve a good-looking front-end some extra portions were introduced to show the effectiveness of the software that was possible. A map was introduced that contained spots of high heat activity, alerts were displayed as well as various graphs that also showcased some of the machine learning methods delivered earlier.

## Evaluation

There were some questions that we fought of ourselves while developing the software. Some, pertaining to the usefulness of the idea itself. However, this can be justified as users being able to judge for themselves their own sun levels however even the more experienced holiday travelers know that one can lose track of sun levels themselves. As well as this, this project would be useful for outdoor construction companies looking to overlook the health of their employees to ensure that their employees are not under undue and unhealthy conditions while carrying out labourious activities.

In terms of the product itself, improvements would have to be made to the prototype to make it wearable by a user and ensuring the hardware is error free. Other metrics, and the machine learning methods could be developed further as to allow a more accurate prediction of insights.

All in all, I felt the project went well and allowed the group to learn more about the integration of hardware and software in an effective manner. Furthermore, we learned how these sorts of projects could be easily and readily be able to impact a population, Singaporean or not, in their daily activities.

In the future I would look to this project to develop my ideas and improve my own skills in effective data collection. And as such using the data to generate useful and helpful insights that could ultimately help people.

See you next time,

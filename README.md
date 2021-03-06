# Temperature-monitoring-system
## Temperature monitoring system developed in Contiki using two Z1-mote (Unicast Communication) and Plotly API  

### Project Implementation

After gathering all the theoretical knowledge explained in the previous section of this report, we started our project implementation by finalizing the project architecture. The main idea of the project was to develop the temperature monitoring system for the patients in hospital using wearable temperature sensor. Due to the hardware limitation we have used two Z1 node for sensor network system and its communication. Similarly, we used inbuilt Tmp102 sensor of Z1 to measure the temperature. Since, the idea of the project was to make a real time application which relays the temperature data to a cloud server. To fulfil this requirement we have used free plotly web service to connect the sensor network to the cloud server. In this implementation we have used Z1 nodes as data transmitter and receiver using RPL protocol for unicast communication between the nodes. Our normal computer was used as a gateway to read the data received by the node and sending it to the plotly web server. Detailed explanation of these implementation will be discussed in the further section of this document. In the following figure the network architecture of the project of the project can be seen; 

![img](https://github.com/dinesh2043/Temperature-monitoring-system/blob/master/img1.jpg)
 
Figure 5: Overall network architecture of the project

In the beginning of the project we have downloaded and installed the gcc-msp430 compiler in contiki to be able to program in the Z1 mote. When the Contiki IDE was ready, we defined the project configuration file to use CSMA driver to be ContikiMAC with the channel check rate to be 8 Hz. ContikiMAC was used because it has lower energy consumption and 8 Hz channel check rate was used because the data was taken every 3 second this speed was fast enough for this project implementation. After this step we started to learn about the inbuilt temperature sensor of the Z1 microcontroller to be able to read the data from the sensor. According to project requirement we were supposed use two nodes for the connection and data transmission, due to that reason RPL unicast communication was used. As, we have learned during this course that in energy constraint devices it is always wise to design the system using less processing power to reduce energy consumption. For that reason, UDP transport protocol was used for data transmission. UDP protocol was used by trading-off the data loss with processing efficiency. In this particular application even after losing some data it will not have any effect because the data is send frequently and each packet consists a complete information of temperature. Then project implementation was continued developing unicast transmitter, unicast receiver and boarder router (gateway) with unlimited resources to transmit data to the web server. In the upcoming section of this document there will the detailed discussion of each of these implementations.

#### Unicast Transmitter Node
The implementation of the unicast transmission consists of two individual sections, temperature data acquisition from the sensor and transmitting that data using unicast communication to the receiver. To accomplish this task the sensor driver and Z1 device driver was used. Similarly for the unicast communication UDP was used on top of RPL protocol. In the first part of this document there is a discussion about the sensor data collection and data formatting.

##### Sensor Data 
 
The built-in temperature sensor of Z1 microcontroller has been designed to measure the environmental temperature from -25°C to +85°C. Due to the limitation of the available devices same sensor was used to get the test data for body temperature. To be able to sense the temperature data and read it in the Z1 mote we need to import sensor driver (i.e. dev/tmp102.h) and microcontroller driver (i.e. dev/i2cmaster.h). This Tmp102 temperature sensor driver helps us to read the digital temperature data and I2C communication device driver for Z1 sensor node is used to transfer data inside the microcontroller. Since it was the test project due to that reason the temperature sensing interval has been set to be 3 sec. Which makes it easy to show that the system is working properly during demo. But for the real life situation the sensor should read data in 60 seconds interval. The raw data gathered from the sensor must be formatted to convert it to human readable °C values. In the following code example the process of getting the temperature data and its formatting is shown; [10]

```
   tmp102_init();  /* to set up ports and pins for I2C communication */
 sign = 1;
 while(1) {
        raw = tmp102_read_temp_raw();  // Reading from the sensor
        absraw = raw;
        if (raw < 0) {        // Perform 2C's if sensor returned negative data
          absraw = (raw ^ 0xFFFF) + 1;
          sign = -1;
        }
       tempint  = (absraw >> 8) * sign;
        tempfrac = ((absraw>>4) % 16) * 625;         // Info in 1/10000 of degree
        minus = ((tempint == 0) & (sign == -1)) ? '-'  : ' ' ;

```

Figure 6: Methods used to read the temperature data

#### Unicast Sender Implementation

In this section a program was written for Z1 microcontroller that sends the temperature data obtained as explained in the previous section to the receiver node. But to achieve this functionality we have used IPv6 routing protocol for low power and lossy networks (RPL). According to the project requirement the unicast communication was the perfect match to design the efficient application. While writing the program for sender node all the necessary library were imported. But in this discussion most important among them will be discussed. To set run-time parameters like IP address uip.h header file was imported to use uIP configuration function. To support the program for the data structures of IPV6 uip-ds6.h header file was imported. UDP protocol was suitable for data transmission in this project and simple-udp module of Contiki OS was used by importing simple-udp.h header file. Similarly Contiki OS has server hack application which consists service registration and dissemination. These registered services are transmitted to all the neighbour nodes which are running servreg-hack application. At last UDP port number 1986 and service id 190 was defined to start writing the program for sender node [10].
Since the program was using IPv6 routing protocol the method was defined to set global IP address for the node. In the beginning of the implementation an IPv6 address was defined by providing the hexadecimal value. Then the data structure handling function was called for using neighbour discovery and auto configuration of the state machines.  After that unicast address structure to the IP address was added. Finally local table was checked and the address was configured according to tentative or preferred address. In the following code snippet we can see the implementation of this method; [10] 

```
   static void
   set_global_address(void)
   {
     uip_ipaddr_t ipaddr;
     int i;
     uint8_t state;
     uip_ip6addr(&ipaddr, 0xaaaa, 0, 0, 0, 0, 0, 0, 0);
     uip_ds6_set_addr_iid(&ipaddr, &uip_lladdr);
     uip_ds6_addr_add(&ipaddr, 0, ADDR_AUTOCONF);
     printf("IPv6 addresses: ");
     for(i = 0; i < UIP_DS6_ADDR_NB; i++) {
       state = uip_ds6_if.addr_list[i].state;
       if(uip_ds6_if.addr_list[i].isused &&
          (state == ADDR_TENTATIVE || state == ADDR_PREFERRED)) {
         uip_debug_ipaddr_print(&uip_ds6_if.addr_list[i].ipaddr);
         printf("\n");
       }
     }
   }
```

 Figure 7: Setting IPv6 address for the sender node
 
After setting the IP address, port number and service id for the node the implementation proceed further to the process thread part of the program. Where two etimer events were decleared and called periodic timer and send timer to send temperature data when the send timer expires. Inside the thread the method servreg_hack_init() was called to transmits the registered services to all the nodes running the server-hack application. All the services running in the neighbour nodes are stored in the local table which can be accessed and checked according to the requirement. Then the method that we have discussed previously was called to assign the appropriate IP address to the node. After that our program is ready to establish the UDP connection to the receiver. To achieve that UDP connection was registered by providing the arguments for local port, remote address, remote port and call-back function. In the implementation remote IP address was set to null to accept packets from any IP address. When a packet is received from remote node we can all the contents of that packets can be accessed. Which will be discussed in the receiver node section. When the connection between two nodes is established the IP address of the receiver can be accessed by providing service ID to servreg_hack_lookup method. At this point of the implementation all the required information was gathered and now it is ready to send the data packet using the connection. When the data collected from the sensor is formatted then the data is stored in the buffer and checked if there is a remote node running server hack application. After that simple_udp_sendto() method was used providing all the values for the arguments to send the temperature data packet. In the following code snippet this implementation is shown; [10]

```
     servreg_hack_init();
  set_global_address();
  simple_udp_register(&unicast_connection, UDP_PORT, NULL, UDP_PORT, receiver);
  while(1) {
         PROCESS_WAIT_EVENT_UNTIL(etimer_expired(&send_timer));
         addr = servreg_hack_lookup(SERVICE_ID);
   if(addr != NULL) {
          static unsigned int message_number;
          char buf[30];
          printf("Sending unicast temperature value to ");
          uip_debug_ipaddr_print(addr);
               sprintf(buf, "%c%d.%d\n", minus, tempint, tempfrac);
    printf(" message count:%d message: %s\n", message_number,buf);
          message_number++;
          simple_udp_sendto(&unicast_connection, buf, strlen(buf) + 1, addr);
       } else {
          printf("Service %d not found\n", SERVICE_ID);
       }
  }
```
  Figure 8: Unicast sender process thread code
  
After following the steps explained in the previous steps Z1 microcontroller was programmed as a sensor node which sense temperature data and sends the data to the receiver node using IPv6 UDP protocol. Complete temperature data is send in single packet due to that reason data loss during the transmission will not effects the application and processing power is saved using UDP transport protocol. The new line character was appended in the buffer to make it easier to read the packet contents. In the following figure the result of sender nodes programme execution is shown; [10]

![img](https://github.com/dinesh2043/Temperature-monitoring-system/blob/master/img2.jpg)
 
Figure 9: UDP message send to the receiver 

#### Unicast Receiver Implementation

While programming the receiver all the library files which were imported for sender node program were imported. For receiver also same port and same service id was used, and process of setting the IP address was also same. After setting the IP address that address value was passed to create RPL Directed Acyclic Graph (DAG) function, where the IPv6 data related structure was checked. After that checking it if the received value is null or not. If the value is not null then that address is set as root address and set the address of server as the root of dag and set its prefix. In the following code the process of creating RPL DAG is shown; [10]

```
   static void
   create_rpl_dag(uip_ipaddr_t *ipaddr){
      struct uip_ds6_addr *root_if;
      root_if = uip_ds6_addr_lookup(ipaddr);
      if(root_if != NULL) {
         rpl_dag_t *dag;
         uip_ipaddr_t prefix;
         rpl_set_root(RPL_DEFAULT_INSTANCE, ipaddr);
         dag = rpl_get_any_dag();
         uip_ip6addr(&prefix, 0xaaaa, 0, 0, 0, 0, 0, 0, 0);
         rpl_set_prefix(dag, &prefix, 64);
         PRINTF("created a new RPL dag\n");
      } else {
        PRINTF("failed to create a new RPL DAG\n");
     }
  
```

 Figure 10: Creating RPL DAG for receiver
 
This program is supposed to receive the data packets send by the sender node to achieve it first of all the server hack initialized and register its application with service id and receiver IP address. When the process thread reaches the simple_udp_register method it call-back the receiver function to receive the packets send by the sender by passing all the arguments. Since only the temperature data was send in the packet so that only the temperature value was printed as output. In the following code receiver function implementation can be seen; [10]

```
     static void
     receiver(struct simple_udp_connection *c,  const uip_ipaddr_t *sender_addr,  uint16_t sender_port, const uip_ipaddr_t *receiver_addr, uint16_t receiver_port, const uint8_t *data,  uint16_t datalen) {
     printf("%s",data);
     }
 
```
Figure 11: Receiver function in the program.

In the process thread of the receiver server hack, set the global IP address to the node and create RPL DAG was initialized as we have mentioned in the previous section. After that server hack application was registered with service ID and its IP address to have a connection to the clients with same service ID. Then a simple UDP connection was registered using unicast connection and other argument over RPL. In the following code snippet shows the implementation process; [10]

```
    PROCESS_THREAD(unicast_receiver_process, ev, data) {
     uip_ipaddr_t *ipaddr;
     PROCESS_BEGIN();
     servreg_hack_init();
     ipaddr = set_global_address();
     create_rpl_dag(ipaddr);
     servreg_hack_register(SERVICE_ID, ipaddr);
     simple_udp_register(&unicast_connection, UDP_PORT, NULL, UDP_PORT, receiver);
     while(1) {
        PROCESS_WAIT_EVENT();
     }
      PROCESS_END();
  }   
```

Figure 12: Receiver process thread implementation

After completing all the above mentioned steps the program was uploaded in the Z1 receiver node. When the connection between the receiver and sender was successful using RPL unicast connection the receiver was able to receive the temperature data send by the sender. According to project requirement only the temperature value was needed due to that reason only the temperature data was printed in the following output figure it is shown as a result; [10]

![img](https://github.com/dinesh2043/Temperature-monitoring-system/blob/master/img3.jpg)

Figure 13: Temperature data received by the receiver

#### Data Acquisition from Serial

Until the previous section we were able to program two Z1 as a sender and receiver node. According to our project plan, we were trying to send the data collected by the receiver to web service to make the data remotely available in World Wide Web. Since, we all know that sensor network system are the resource constraint devices. Due to that reason we were trying to implement our project in such a way where the more resource intensive operation was done using our own computer as a boarder router to have a connection to the web service to feed temperature data. To accomplish this task python programming language was used to write a program to read the temperature data received by the receiver node. When the temperature data is received by the node current time stamp was captured which helps to use Plotly API to create a graph of the data obtained during this process in the real time. Receiver node was connected to the computer using the data cable and the python script was written to read the data in the serial. For this implementation serial library and time library of python was imported.  After that serial port was defined by providing port address, baudrate to read data, parity, stop bits, byte size and time out value. Byte size was set to be 8 bits and baudrate (i.e. 115200) was defined exactly the same value while flashing the Z1 node. If the value is different than the program will not be able to read the value send by sender node. While writing the script if different value is assigned then the data printed in the serial will not be readable. In this program the seral data was read character by character inside forever loop. When the serial read method finds a new line character then the program knows that the packet data was complete and that value is stored in temperature string. At the same time the current time stamp is also sent to send to the Plotly server. The following script shows the implementation of serial data;

```
   import serial
   import time
   ser = serial.Serial( port='/dev/ttyUSB0',\ baudrate=115200,\  parity=serial.PARITY_NONE,\
       stopbits=serial.STOPBITS_ONE,\ bytesize=serial.EIGHTBITS,\ timeout=0)
   print("connected to: " + ser.portstr)
   temp_str = ""
   while True:
   for c in ser.read():
             if c == ' ':
         temp_str = temp_str
             elif c == '\n':
                  print("Temperature: ",float(temp_str))
          print("Time: ",time.strftime("%H:%M:%S"))
          temp_str = ""
     else:
          temp_str+=str(c)
   ser.close()
  
```

Figure 14: Script to read the data from serial

The result of this particular part of the implementation can be seen in the following figure;

![img](https://github.com/dinesh2043/Temperature-monitoring-system/blob/master/img4.jpg)
 
 Figure 15: Temperature and time values from serial
 
#### Plotly API Implementation

First of all user account was created in plotly website to get the API key and streaming API tokens which are needed to stream the traces.  Then plotly python library, numpy, plotly tools and plotly graph objects library was imported. Then the code to get stream id was written, after getting the id an instance of stream id object was made. Using this stream id object trace was initialized for streaming in plotly, where the array for X and y coordinates was defined. Then the title was added for the layout object and figure object was defined to draw the figure. Then after defining the streaming object with stream id the stream connection was opened.  Data needed for the two coordinates of layout object are send to the web server using HTTP post. Current data gathered inside the forever loop is appended to the plot in the graph. These implementations can be seen in the following code snippet; [11]

```
      import plotly 
      import certifi
      import urllib3
      http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED',ca_certs=certifi.where())
      plotly.tools.set_credentials_file(username='dinsap', api_key='0l2j9xmxa3')
      import numpy as np 
      import plotly.plotly as py  
      import plotly.tools as tls   
      import plotly.graph_objs as go
      #set stream_ids
      f = []
      f.append('ovm08xi3v8')
      tls.set_credentials_file(stream_ids=f)
      stream_ids = tls.get_credentials_file()['stream_ids']
      # Get stream id from stream id list 
      stream_id = stream_ids[0]
      # Make instance of stream id object 
      stream_1 = go.Stream(
          token=stream_id,  # link stream id to 'token' key
          maxpoints=80      # keep a max of 80 pts on screen
      )
      # Initialize trace of streaming plot by embedding the unique stream_id
      trace1 = go.Scatter(
          x=[],
          y=[],
          mode='lines+markers',
          stream=stream_1         # (!) embed stream id, 1 per trace
      )
      data = go.Data([trace1])
      # Add title to layout object
      layout = go.Layout(title='Time series')
      # Make a figure object
      fig = go.Figure(data=data, layout=layout)
      # We will provide the stream link object the same token that's associated with the trace we wish to stream to
      s = py.Stream(stream_id)
      # We then open a connection
      s.open()
      while True:
       for c in ser.read():
              if c == ' ':
                  ……
              elif c == '\n':
                  x = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S.%f')
                  y = temp_str
                  print("Temperature: "+ temp_str)
                  print("Time: ",time.strftime("%H:%M:%S"))
                  temp_str = ""
                  # Send data to your plot
                      s.write(dict(x=x, y=y))      
                      #     Write numbers to stream to append current data on plot,
                      #     write lists to overwrite existing data on plot	    
                      time.sleep(2)  # plot a point every second    
              else:
                  #line.append(c)
                  temp_str+=str(c)
      # Close the stream when done plotting
      s.close() 
      # Embed never-ending time series streaming plot
      tls.embed('streaming-demos','12')
 
```

Figure 16: Python script to use Plotly API

After this implementation our sensor network system was connected to the web service and to draw a real time graph according to the temperature and time values. Networking section of the project was successfully implemented but the key application part where we can set the alarm if the temperature value is above the normal range can be done in the future development. Similarly it is a health care application where the information and network is sensitive so that it is important to implement the security protocols of contiki OS to secure the network and data. The following figure shows the results of our implementation; 

![img](https://github.com/dinesh2043/Temperature-monitoring-system/blob/master/img4.jpg)
 
Figure 17: Real time graph obtained in the web server

###	Results and Performance Evaluation
The previous section gave the implementation method of our test-bed project. In the performance evaluation, we used third-party web application to display our test outcome. The figure 17 shows test output information Plotly web page. In the project, we used 3 seconds interval to send the data from the node. The data displayed on the web page is valid. We tested several times to see the accuracy of the data. Every time we heat up the temperature sensor, the graph shows an immediate change in real time. Besides, we also checked the compatibility of Z1 mote for body temperature functionality. For test purpose, we used a regular body temperature measuring thermometer to take our readings. To compare, we also tested with the Z1 mote by placing in elbow. We basically took five readings and every time Z1 mote shows on average of 5°Celsius less reading than the analog thermometer.
However, this was obvious to get an inaccurate reading while using Z1’s embedded sensor for measuring body temperature. The test outcome would be accurate if we could use a wearable external sensor with Z1 by using Phidgets port. Nevertheless, the test outcome provides a clear evidence of accurate wireless sensor network communication between the nodes and the PC connected through the Internet. The prototype system has been successfully tested and displayed the result on the web page.

### Future Development 

This project was implemented to use only two Z1 nodes for the network communication and the receiver node was connected to the computer to act as gateway to connect to the cloud services. This project can be further developed using RPL multicast communication to sense multiple patient body temperature using wearable sensors. All the data received by the web server are not important, so that in web server we can only store the information which is important. If the reading is not normal and the patient needs immediate attention then alert message system can be implemented to send the alert message to the responsible physician.  Since this implementation consists the sensitive health information of the patients due to that reason patients information and network must be secured. So, that in the further development security of the information and network must be implemented. As Contiki OS has library called ContikiSec that can be used to enforce security in Z1 nodes. Among the three modes of operation ContikiSec-AE is better because it provides confidentiality, integrity and authentication.   

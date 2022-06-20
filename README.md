# nanocoap_server
## overview
The nanocoap_server project is based on the RIOTOS example nanocoap_server. It was extended with functionality to request read temperature values from a ds18 temperature sensor. 

The client application uses the gcoap RIOTOS example. It is used to issue request to the nanocoap_server and consequently to request the temperature value from the nanocoap_server.
## Modules
Following modules are needed and included inside the Makefile:
* modules needed for the functioning of nano_coap
    *  `netdev_default`
        * Is a generic low-level network driver interface.
    *  `auto_init_gnrc_netif`
    * `gnrc_ipv6_default`
    * `sock_udp`
    * `gnrc_icmpv6_echo`
    * `nanocoap_sock`
* modules for reading temperature from ds18
    * `ds18`
        * contains ds18 convenience functions
    * added flag for configuration of connection to ds18
        * `CFLAGS+= -DDS18_PARAM_PIN=GPIO_PIN\(0,14\)`
##  Project
### main.c
#### Headers
```C
#include <stdio.h>
#include "net/nanocoap_sock.h"
#include "xtimer.h"
```
* `stdio.h`
    *  needed for basic c functionalities.
*  `"xtimerh` 
    *    Basic timer functionality.
* `naocoap_sock.h`
    * nanocoap library

#### Summary
This part of the application provides the functionality for the board to initiate the creation of the nanocoap server. It generates a socket, to which clients can connect to and the associated link local ip address.

### coap_handler.c
#### Overview
We want to create a Resource on our CoAP Server which provides us with the ability to read the temperature from the temperature sensor.

Resources can be configured inside the coap_handler.c file.

Inside the coap_handler.c file, multiple handlers are present. Handlers are needed to provide resources. They are called when a resource is requested via a CoAP method.

This part was extended with a custom handler to read temperature values, and reply to a request with the read temperature value. 
---
#### Headers
```C
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#include "fmt.h"
#include "net/nanocoap.h"
#include "hashes/sha256.h"
#include "kernel_defines.h"

#include "ds18.h"
#include "ds18_params.h"
```
Following headers were added:
* `ds18.h`
* `ds18_params.h`

These two librarys represent convenience functions for reading from a ds18 temperature sensor
---
#### Resource defintion
Handlers are defined in the format:
`typedef ssize_t(* coap_handler_t) (coap_pkt_t *pkt, uint8_t *buf, size_t len, void *context)`

To reply to a request, once the corresponding actions were taken inside the handler,
the `coap_reply_simple(coap_pkt_t* pkt,unsigned code, uint8_t* buf, size_t len, unsigned ct, const void* payload, size_t payload_len)` structure is used.
* pkt
    * the packet to reply to
* code
    * coap reply status code
* buf
    * the buffer to which the reply is written to
* len
    * the size of the buffer
* ct
    * content type of the payload, for example: COAP_FORMAT_TEXT
* payload
    * pointer to the payload!
* payload_len
    * length of the payload!
---
#### Temperature handler
```C
static ssize_t _get_temp_handler(coap_pkt_t *pkt, uint8_t *buf, size_t len, void *context)
{
    (void)context;
     int init = ds18_init(&ds18, ds18_params);
   if (init  == DS18_OK) {
      printf("DS18 initialized successfully\n");
   }
   else if (init == DS18_ERROR) {
     return coap_reply_simple(pkt, COAP_CODE_INTERNAL_SERVER_ERROR, buf,
                                 len, COAP_FORMAT_TEXT, NULL, 0);	
    
   }

    int16_t temp = 0;
    int err = ds18_get_temperature(&ds18,&temp);
    if( err == -1 ){
	return coap_reply_simple(pkt, COAP_CODE_INTERNAL_SERVER_ERROR, buf,
                                 len, COAP_FORMAT_TEXT, NULL, 0);	
    }
    else{
	char str[50];
	sprintf(str,"Temp: %i.%u C\n", (temp / 100), (temp % 100));
	//sprintf(str,"%i",temp);
	return  coap_reply_simple(pkt, COAP_CODE_205, buf, len,
            COAP_FORMAT_TEXT, (uint8_t*)str, strlen(str));
    }
    
}
```
I defined my own handler, for reading temperatures, in the same format as the predefined handlers. 

The body of the handler largely conforms to the code for reading temperature values in the second lab. 

Some differences can be observed tough:

* Instead of reading every x seconds from the temperature sensor, we simply read from the sensor when the handler is called -> thus when a request for this resource comes in
* Errors are not printed, we instead return a reply with the Inter Server Error status code
* to create our response message, we initiate a character array of length 50, and use sprintf to create our formatted and converted temperature string
* If everything went well, we return a reply with Status code 205
This reply contains our formated temperature string str
---
#### Including the handler and adding the path:
``` C
const coap_resource_t coap_resources[] = {
    COAP_WELL_KNOWN_CORE_DEFAULT_HANDLER,
    { "/echo/", COAP_GET | COAP_MATCH_SUBTREE, _echo_handler, NULL },
    { "/sha256", COAP_POST, _sha256_handler, NULL },
     { "/temp", COAP_GET, _get_temp_handler, NULL },

};
```

To actually use our handlers, we must define the routes and methods for our resources/handlers.
This is done inside the coap_resource_t structure.
It’s fields contain a path, the methods, a handler and a pointer to user defined context data.

The method to be used on our resource `/temp` is the COAP_GET method. 

# gcoap - CoAP Client
## Overview
The client application uses the gcoap RIOTOS example. It is used to issue request to the nanocoap_server and consequently to request the temperature value from the nanocoap_server.
## Issuing requests
Request to the nanocoap_server can be issued after connecting to the flashed board via a serial connection. 

To request the temperature, simply issue following command, over the serial connection:
`coap get <nano_coap server ip> 5683 /temp`

The response will be displayed in the terminal.











sources:
* https://doc.riot-os.org/group__net__nanosock.html
* https://www.netify.ai/resources/protocols/coap
* https://doc.riot-os.org/group__net__nanocoap.html#ga687f23858a4a5036b60aedee24fb6b90
* https://doc.riot-os.org/group__net__nanocoap.html#ga82578bcdaeadcb10bdc3ecc52b0b6888
* https://doc.riot-os.org/group__drivers__netdev__api.html



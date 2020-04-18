---
title: "The TUN/TAP interface on Linux"
date: 2010-12-08T11:27:07+02:00
draft: false
---

Recently, I had to work with the tun interface on Linux. There is not a lot of documentation on this subject so here is a little presentation.

TUN/TAP is like a physical Ethernet port but for user space program. Instead of receiving data from a physical device, it receives it from a user space program and instead of sending data through a wire, sends it to a user space program.

This interface corresponds to a file situated in /dev/net/tun or /dev/tun. When a program opens this file, an interface is created. The name of this interface can be set by the program or automatically set by the tun driver as tunXX, tapXX or bnepXX where XX can be 0, 1, 2… This interface can be manipulated by ifconfig such as eth0.

More information can be found here

Now, i describe in the following some steps to open and use a tun interface.

Opening the file :

```c
int tap = open(tunname[i], O_RDWR);
if (tap < 0) {
    fprintf(stderr, "TAP: failed to open TUN interfacen");
    return -1;
}
```

Configuring the interface :

```c
struct ifreq ifr;
memset(&ifr, 0, sizeof(ifr));
/* Flags: IFF_TUN   - TUN device (no Ethernet headers)
 *           IFF_TAP   - TAP device
 *
 *           IFF_NO_PI - Do not provide packet information
 */
 ifr.ifr_flags = IFF_TAP|IFF_NO_PI;
 strcpy(ifr.ifr_name, "bnep");

 if( ioctl(tap, TUNSETIFF, (void *) &ifr) < 0 ) {
     fprintf(stderr, "TAP: failed to set BNEP namen");
     close(tap);
     tap = -1;
     return -1;
 }
```

Then we are setting the mac address :

```c
struct sockaddr sap;
sap.sa_family = ARPHRD_ETHER;
((char*)sap.sa_data)[0]=0x00;
((char*)sap.sa_data)[1]=0x11;
((char*)sap.sa_data)[2]=0x22;
((char*)sap.sa_data)[3]=0x33;
((char*)sap.sa_data)[4]=0x44;
((char*)sap.sa_data)[5]=0x55;

memcpy((char *) &ifr.ifr_hwaddr, (char *) &sap,
       sizeof(struct sockaddr));

if (ioctl(tap, SIOCSIFHWADDR, &ifr) < 0) {
      fprintf(stderr, "TAP: failed to set MAC addressn");
}
```

The interface is configured. We can now start to listen :

```c
uint8_t* buffer = (uint8_t *)malloc(1500);
//size of an ethernet frame
assert(buffer != NULL);

while (tap >=0) {
   struct timeval tv={ 1, 0};
   fd_set rfd;
    FD_ZERO(&rfd);
    FD_SET(tap, &rfd);

   // We are waiting for data coming to the file
   if (select(tap+1, &rfd, NULL, NULL, &tv)<0) {
       fprintf(stderr, "bnep_sender: select failedn");
       break;
   }

   // If data to be read
   if (FD_ISSET(tap, &rfd)) {
        n = read(tap, buffer, BNEP_BUFFER_SIZE);
        if (n  0 )  {
            //do some stuff with the data
        }
    }
}

if (buffer != NULL) free(buffer);
if (tap >= 0) close(tap);

return 0;
```

On the same way, we can write data to the file using ‘write’. Don’t forget the Ethernet header !

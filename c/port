/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Serial port drivers                         <]
Current version   [> 00.18                                       <]
Version date      [> 27-January-1993                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "port.h"
#include "config.h"
#include "driver.h"

extern int driver_port;
extern void window_poll(void);
int    port_outsize,key_in,key_out;
char   key_buffer[256];

/*** Set Xon/Xoff flow control ************************************ EXTERNAL */

void port_xonxoff(int enable)
  {
  (*driver)(DRIVER_FLOWCONTROL,driver_port,enable?3:1);
  }

/*** Set serial port data speeds ********************************** EXTERNAL */

void port_speed(int inspeed,int outspeed)
  {
  (*driver)(DRIVER_TXSPEED,driver_port,outspeed);
  (*driver)(DRIVER_RXSPEED,driver_port,inspeed);
  }

/*** Set port parity ********************************************** EXTERNAL */

void port_parity(int par)
  {
  par=par;

  /* Set 8n1 */
  (*driver)(DRIVER_WORDFORMAT,driver_port,0);
  }

/*** Clear transmit buffer **************************************** EXTERNAL */

void port_txclear(void)
  {
  (*driver)(DRIVER_FLUSHTX,driver_port);

#ifdef NOTYET
  if (setbaud==2 && arq==1) /* HST destructive break */
    {
    (*driver)(DRIVER_BREAK,driver_port,3);
    }
#endif
  }

/*** Clear receive buffer ***************************************** EXTERNAL */

void port_rxclear(void)
  {
  (*driver)(DRIVER_FLUSHRX,driver_port);
  }

/*** Wait for buffer to empty ************************************* EXTERNAL */
                                      
void port_waitout(void)
  {
  if (driver_flags&DFLAG_WONTEMPTY) return;
  while(port_txbuffer()<port_outsize) window_poll();
  }

/*** Return TRUE if o/p buffer empty ****************************** EXTERNAL */

int port_allsent(void)
  {
  if (driver_flags&DFLAG_WONTEMPTY) return(TRUE);
  return(port_txbuffer()==port_outsize);
  }

void key_insert(int key)
  {
  if (port_rxbuffer()==255) return;
  key_buffer[key_in]=key;
  key_in=(key_in+1)%256;
  }


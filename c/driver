/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCterm VII                                 <]
Author            [> Hugo Fiennes                                <]
Date started      [> 05-March-1990                               <]
                  [>                                             <]
Module name       [> Driver loading                              <]
Current version   [> 00.04                                       <]
Version date      [> 09-December-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>    This source is COPYRIGHT © 1992 by       <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include <stdio.h>
#include <string.h>
#include "driver.h"
#include "os.h"

int  (*driver)(int,...);
int  *driver_speedtable,driver_flags,driver_version,driver_noofspeeds;
char *driver_info,*driver_creator;

void *driver_load(int *driver_block,char *drivername)
  {
  FILE *drv; long len; char temp[60];

  sprintf(temp,"SerialDev:Modules.%s.Driver",drivername);
  if ((drv=fopen(temp,"rb"))==NULL) return(NULL);
  fseek(drv,0,SEEK_END);
  len=ftell(drv);
  fseek(drv,0,SEEK_SET);
  fread(driver_block,1,len,drv);
  fclose(drv);

  driver_flags=driver_block[49];
  driver_version=driver_block[48];
  driver_speedtable=&driver_block[64];
  driver_info=(char*)&driver_block[32];
  driver_creator=(char*)&driver_block[40];

  driver_noofspeeds=0;
  while(driver_speedtable[driver_noofspeeds++]);
  driver_noofspeeds--;

  return(driver_block);
  }

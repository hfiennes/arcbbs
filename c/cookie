/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Fortune cookie reader                       <]
Current version   [> 01.09                                       <]
Version date      [> 09-November-1991                            <]
State             [> Finished                                    <]
                  [>                                             <]
                  [> This source is COPYRIGHT (c) 1989/90/91 by  <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
                                    
#define LIMIT 7177
#define BASE1 5

extern char *message_buffer;
char common[]="thinanitistoonaterenreouarasesleorllofedhestwhbesechalme";
char *cookiebuffer;

int xread_time(void)
  {
  os_regset r;
  os_swi(0x42,&r); /* OS_ReadMonotonicTime */
  return(r.r[0]);
  }
  
char *cookie()
  {
  FILE *cookiefile;
  int cookie,temp=0,over=0,space,b,pointer=0;

  /* Set random seed */
  srand((unsigned)(time(NULL)+xread_time()));

  /* Open file */
  if ((cookiefile=fopen("<Fortune$Data>","rb"))==NULL)
    {
    return("");
    }
    
  /* Get a random cookie */
  cookie=(rand()%LIMIT)+1;                          
  fseek(cookiefile,(cookie*3)+BASE1,SEEK_SET);
  fread(&temp,3,1,cookiefile);
  fseek(cookiefile,temp,SEEK_SET);
  cookiebuffer=message_buffer;

  do
    {
    if ((b=fgetc(cookiefile))>127) space=1; else space=0;
    b&=0x7f;

    switch(b)
      {
      case 0x1c:
        {
        cookiebuffer[pointer++]=13;
        cookiebuffer[pointer++]=10;
        break;
        }
      case 0x1d:
        {
        int l;
        for(l=0;l<8;l++) cookiebuffer[pointer++]=32;
        break;
        }
      case 0x1e:
        {
        over=1;
        break;
        }
      default:
        {
        if (b>=32) cookiebuffer[pointer++]=b;
        if (b<0x1c)
          {
          cookiebuffer[pointer++]=common[b<<1];
          cookiebuffer[pointer++]=common[(b<<1)+1];
          }
        break;
        }
      }
    if (space) cookiebuffer[pointer++]=32;
    }
  while(!over);

  /* Close the file */
  fclose(cookiefile);

  /* Terminate the string */
  cookiebuffer[pointer++]=13;
  cookiebuffer[pointer++]=10;
  cookiebuffer[pointer]=0;

  /* Return it */
  return(cookiebuffer);
  }

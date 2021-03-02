/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> ARCmail arc'er                              <]
Current version   [> 01.14                                       <]
Version date      [> 02-May-1993                                 <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include <stdio.h>
#include <stddef.h>
#include <time.h>
#include <stdlib.h>
#include <string.h>
#include <utils.h>

#include "wimp.h"        /*  access to WIMP SWIs                      */
#include "wimpt.h"       /*  wimp task facilities                     */
#include "os.h"          /*  low-level RISCOS access                  */
#include "dbox.h"        /*  dialogue box handling                    */
#include "werr.h"

#define BBS_CHUNK             0x41000
#define BBS_ARCID             BBS_CHUNK+60
#define BBS_ARC               BBS_CHUNK+61

int got_type=0;

void window_poll()
  {
  wimp_eventstr event;

  wimp_poll(0,&event);
  switch(event.e)
    {
    case 17:
    case 18:
      {
      switch(event.data.msg.hdr.action)
        {
        /* Pickup on quit message */
        case 0:
          {
          exit(0);
          }
        default:
          {
          if (event.data.msg.hdr.action>BBS_CHUNK && event.data.msg.hdr.action<(BBS_CHUNK+64))
            {
            got_type=event.data.msg.hdr.action;
            }
          break;
          }
        }
      break;
      }
    }
  }

void do_arc(char *a)
  {
  wimp_msgstr message;
  clock_t startarc;

  do
    {
    sprintf(message.data.chars,"arc -y 10");
    message.hdr.size=4*((24+strlen(message.data.chars))/4+1);
    message.hdr.your_ref=0;
    message.hdr.action=BBS_ARC;
    wimp_sendmessage(wimp_ESEND,&message,0);

    got_type=0; while(got_type!=BBS_ARC) window_poll(); 
    startarc=clock(); got_type=0;

    while(got_type!=BBS_ARC && (clock()-startarc)<250) window_poll();
    }
  while(got_type!=BBS_ARC);

  sprintf(message.data.chars,"arc %s",a);

  message.hdr.size=4*((24+strlen(message.data.chars))/4+1);
  message.hdr.your_ref=0;
  message.hdr.action=BBS_ARC;
  wimp_sendmessage(wimp_ESEND,&message,0);

  /* Extra one to catch broadcast to ourselves */
  got_type=0; while(got_type!=BBS_ARC) window_poll(); 
  got_type=0; while(got_type!=BBS_ARC) window_poll();
  }

char *strippath(char *in)
  {
  char *ptr=in+strlen(in)-1;
  while(ptr>=in && *ptr!='.' && *ptr!=':') ptr--;
  if (*ptr=='.' || *ptr==':') ptr++;
  return(ptr);
  }

int main(int argc,char *argv[])
  {
  int ourzone,ournet,ournode,ourpoint,zone,net,node,point,count=0,type=0,arcexist=0;
  short netdiff,nodediff;
  char *pktname,file[128],command[256],arcname[128];
  static char taskname[40];
  FILE *at;

  if (argc<3)
    {
    printf("Usage: !sparkmail ouraddress outaddress [\"sparkflags\"]\n  eg: arcmail 2:252/102.0 2:252/106.0 \"-G -H -v -i\"\n");
    exit(2);
    }

  if (argc==3) argv[3]="-G -H -v -v -i";

  if (sscanf(argv[1],"%d:%d/%d.%d",&ourzone,&ournet,&ournode,&ourpoint)!=4)
    {
    printf("Bad 'ouraddress' format: %s\n",argv[1]);
    exit(2);
    }

  if (sscanf(argv[2],"%d:%d/%d.%d",&zone,&net,&node,&point)!=4)
    {
    printf("Bad 'outaddress' format: %s\n",argv[2]);
    exit(2);
    }

  /* Find differences for arcmail packet name */
  netdiff=ournet-net;
  nodediff=ournode-node;
  if (zone==ourzone)
    {
    if (point)
      {
      sprintf(arcname,"<ARCbbs$OutboundBase>.Outbound.%04hx%04hxPN.%08xM1",net,node,point);
      sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04xPN.%04x#T",net,node,point);
      }
    else
      {
      sprintf(arcname,"<ARCbbs$OutboundBase>.Outbound.%04hx%04hxM1",netdiff,nodediff);
      sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04x#T",net,node);
      }
    }
  else
    {
    sprintf(arcname,"<ARCbbs$OutboundBase>.Outbound%02x.%04hx%04hxM1",zone,netdiff,nodediff);
    sprintf(file,"<ARCbbs$OutboundBase>.Outbound%02x.%04x%04x#T",zone,net,node);
    }

  sprintf(taskname,"SparkMail %d:%d/%d.%d",zone,net,node,point);
  wimpt_init(taskname);

  window_poll(); window_poll(); window_poll();

  /* Check for any packets on hold, etc */
  if ((pktname=dirscan(file))==NULL)
    {
    window_poll(); window_poll(); window_poll();
    exit(1);
    }
  
  do
    { 
    time_t now; char *b=strippath(pktname); int pos=point?4:8,giveup=60;

    if (b[pos]=='H' || b[pos]=='O' || b[pos]=='C' || b[pos]=='D')
      {
      FILE *arcchk;

      /* ARC packet */
      time(&now);

      /* Does ARC already exist? */
      if ((arcchk=fopen(arcname,"rb"))!=NULL)
        {
        arcexist=1;
        fclose(arcchk);
        }

      /* Can we open archive? */
      if ((arcchk=fopen(arcname,"rb+"))!=NULL)
        {
        /* Yes: archive it */
        fclose(arcchk);

        sprintf(command,"%s -a %s/ %s",argv[3],arcname,pktname);
        do_arc(command); 
        sprintf(command,"%s -R %s/%s %08X.PKT",argv[3],arcname,b,(int)now);
        do_arc(command);

        /* Wait until we can open archive */
        while((arcchk=fopen(arcname,"rb+"))==NULL && giveup>0) { window_poll(); window_poll(); window_poll(); giveup--; }

        if (giveup!=0)
          {
          /* Check end of archive */
          fseek(arcchk,-1,SEEK_END);
          if (fgetc(arcchk)!=0)
            {  
            fseek(arcchk,-1,SEEK_END);
            fflush(arcchk);
            clearerr(arcchk);

            fputc(0,arcchk);
            }  
          fclose(arcchk);

          /* Increment packet count */
          count++;

          /* Delete packet */
          remove(pktname);
          type=b[pos]; /* H, O, etc */
          
          pktname=dirscan(file); /* Start at top again */
          }
        else pktname=dirscan(0);
        }
      else pktname=dirscan(0);
      }
    else pktname=dirscan(0);
    }
  while(pktname!=NULL);

  if (type)
    {      
    /* Get types right! */
    if (type=='O') type='F';

    /* Generate file attach */
    if (zone==ourzone)
      {
      if (point) sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04xPN.%04x%cO",net,node,point,type);
      else sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04x%cO",net,node,type);
      }
    else
      sprintf(file,"<ARCbbs$OutboundBase>.Outbound%02x.%04x%04x%cO",zone,net,node,type);

    if ((at=fopen(file,"r+"))==NULL)
      {            
      /* File doesn't exist, create a new one */
      if ((at=fopen(file,"w"))!=NULL)
        {
        /* Delete after sending */
        fprintf(at,"^%s\n",arcname);
        }
      }
    else
      {
      char line[128];

      /* Read in lines, check to see if one is what we have already */
      sprintf(command,"^%s\n",arcname);
      do
        {
        if (fgets(line,127,at)==NULL)
          {
          fflush(at);
          fprintf(at,"^%s\n",arcname);
          break;
          }
        if (strcmp(line,command)==0) break;
        }
      while(1);
      }
    fclose(at);
    }

  exit(0);
  }

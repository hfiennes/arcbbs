/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> ARCmail arc'er                              <]
Current version   [> 01.12                                       <]
Version date      [> 16-December-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
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
#include "werr.h"

int lastpoll=0;

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
          exit(0);
        }
      break;
      }
    }              
                 
  lastpoll=clock();
  }

int copy(char *from,char *to)
  {
  char buffer[8192];
  FILE *i,*o; long fl,chunk;
  
  /* Open input and find length */
  if ((i=fopen(from,"rb"))==NULL) return(1);
  fseek(i,0,SEEK_END); fl=ftell(i); fseek(i,0,SEEK_SET);
  
  /* Open output */
  if ((o=fopen(to,"wb"))==NULL) { fclose(i); return(2); }
  
  do
    {
    /* Get chunk length and read it */
    chunk=(fl>8192)?8192:fl;
    fread(buffer,chunk,1,i);
    
    /* Write it */
    fwrite(buffer,chunk,1,o);

    /* Decrement */
    fl-=chunk;

    /* Poll if needed */
    if ((clock()-lastpoll)>10) window_poll();
    }
  while(fl>0);
  
  fclose(i);
  fclose(o);
  return(0);
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
  int ourzone,ournet,ournode,ourpoint,zone,net,node,point,type=0,method=1;
  short netdiff,nodediff;
  char *pktname,file[128],command[256],arcname[256],realarcname[256],*p;
  FILE *at;

  if (argc<3)
    {
    printf("Usage: !sparkmail ouraddress toaddress [compress|arc|zip|tar|<number>]\n  eg: arcmail 2:252/102.0 2:252/106.0 zip\n");
    exit(2);
    }
  
  if (argc>3)
    {
    p=argv[3]; do { *p=tolower(*p); } while(*++p);
    if (strcmp(argv[3],"compress")) method=1;
    else if (strcmp(argv[3],"arc")) method=3;
    else if (strcmp(argv[3],"zip")) method=4;
    else if (strcmp(argv[3],"tar")) method=7;
    if (isdigit(*argv[3])) method=atoi(argv[3]);
    }

  if (sscanf(argv[1],"%d:%d/%d.%d",&ourzone,&ournet,&ournode,&ourpoint)!=4)
    {
    printf("Bad 'ouraddress' format: %s\n",argv[1]);
    exit(2);
    }

  if (sscanf(argv[2],"%d:%d/%d.%d",&zone,&net,&node,&point)!=4)
    {
    printf("Bad 'toaddress' format: %s\n",argv[2]);
    exit(2);
    }

  /* Find differences for arcmail packet name */
  netdiff=ournet-net;
  nodediff=ournode-node;                                    
  if (zone==ourzone)
    {
    if (point)
      {
      sprintf(arcname,"SparkFS#%s.Outbound.%04hx%04hxPN.%08xM1",getenv("ARCbbs$OutboundBase"),net,node,point);
      sprintf(realarcname,"<ARCbbs$OutboundBase>.Outbound.%04hx%04hxPN.%08xM1",net,node,point);
      sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04xPN.%04x#T",net,node,point);
      }
    else
      {
      sprintf(arcname,"SparkFS#%s.Outbound.%04hx%04hxM1",getenv("ARCbbs$OutboundBase"),netdiff,nodediff);
      sprintf(realarcname,"<ARCbbs$OutboundBase>.Outbound.%04hx%04hxM1",netdiff,nodediff);
      sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04x#T",net,node);
      }
    }
  else
    {
    sprintf(arcname,"SparkFS#%s.Outbound%02x.%04hx%04hxM1",getenv("ARCbbs$OutboundBase"),zone,netdiff,nodediff);
    sprintf(realarcname,"<ARCbbs$OutboundBase>.Outbound%02x.%04hx%04hxM1",zone,netdiff,nodediff);
    sprintf(file,"<ARCbbs$OutboundBase>.Outbound%02x.%04x%04x#T",zone,net,node);
    }                                                

  /* Convert :'s to #'es */
  p=arcname; do { if (*p==':') *p='#'; } while(*++p);


  /* Check for any packets on hold, etc */
  if ((pktname=dirscan(file))==NULL)
    {
    /*printf("No outbound mail to process for %s\n",argv[2]);*/
    exit(1);
    }
  
  wimpt_init("ARCmail");
  window_poll();

  do
    { 
    time_t now; char *b=strippath(pktname); int pos=point?4:8;

    if (b[pos]=='H' || b[pos]=='O' || b[pos]=='C' || b[pos]=='D')
      {
      FILE *arcchk;

      /* ARC packet */      
      time(&now);
      
      /* Destination name */
      if (filetype(realarcname)==F_NONE)
        {
        sprintf(command,"SparkFScreate %d %s",method,realarcname);
        system(command);
        }

      sprintf(command,"%s:$.%08X/PKT",arcname,(int)now);
      copy(pktname,command);

      /* Check end of archive */
      if ((arcchk=fopen(realarcname,"r"))!=NULL)
        {
        fclose(arcchk);

        /* Delete packet */
        remove(pktname);
        type=b[pos]; /* H, O, etc */
        
        pktname=dirscan(file); /* Start at top again */
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
    else sprintf(file,"<ARCbbs$OutboundBase>.Outbound%02x.%04x%04x%cO",zone,net,node,type);
                         
    if ((at=fopen(file,"r+"))==NULL)
      {            
      /* File doesn't exist, create a new one */
      if ((at=fopen(file,"w"))!=NULL)
        {
        /* Delete after sending */               
        fprintf(at,"^%s\n",realarcname);
        }
      }
    else
      {
      char line[128];

      /* Read in lines, check to see if one is what we have already */
      sprintf(command,"^%s\n",realarcname);
      do
        {
        if (fgets(line,127,at)==NULL)
          {
          fflush(at);
          fprintf(at,"^%s\n",realarcname);
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

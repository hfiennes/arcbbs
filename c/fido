/*                  __________________________________________
                  [>                                          <]
Project           [> ARCbbs                                   <]
Author            [> Hugo Fiennes                             <]
Date started      [> 04-April-1989                            <]
                  [>                                          <]
Module name       [> Bundler/Tosser/Router FSC-0039           <]
Current version   [> 00.82                                    <]
Version date      [> 08-July-1993                             <]
State             [> Unfinished                               <]
                  [>                                          <]
                  [>   This source is COPYRIGHT © 1989-1993   <]
                  [>    by Hugo Fiennes of The Serial Port    <]
                  [>__________________________________________<]
*/

#include "include.h"   /* General includes */
#include "fidoa.h"     /* Assembler support */
#include "config.h"    /* ARCbbs config */
#include "mail.h"      /* Mail header */
#include "userlog.h"   /* User header */
#include "modcomm.h"   /* Module interface */
#include "servcomm.h"  /* Message-level interface */
#include "portmisc.h"  /* Port-level misc stuff */
#include "fido.h"      /* Fido support */
#include "miscbbs.h"   /* Misc BBS routines */
#include "servmess.h"
#include "str.h"

/* Number of cached file handles on export */
#define HANDLE_CACHE 8
struct { FILE *f; _address_arch to; } cache[HANDLE_CACHE];

extern char *fido_rules2(char*);
extern void _fclose(FILE**);
extern int lastpoll;
 
#ifdef SERVER  
#include "dbox.h"
extern void event_process(void);
int fido_inuse=0;
#define window_poll() event_process()
#endif
extern char *message_buffer;

/* Fido use varaibles */
_address_arch ouraddress;
char ourname[60];
static char file[60];
static char fido_conference[128][30];
static int  fido_bbsarea[128],fido_noofareas,junkarea;

extern int  fido_makebundle(_address_arch*,int,char*),ishex(int),savednr,
            checkhold(int,int,int,int);

extern void open_packet(int,int,int,int,short*,short*,short*,short*,FILE**,int);

#ifdef SERVER
static int packetsgen=0;
#endif

/*** Fidonet SETUP ***********************************************************/
int fido_setup()
  {
  FILE *config; char line[80]; int end=0,a;

  /* Clear handle cache */
  for(a=0;a<HANDLE_CACHE;a++) cache[a].f=NULL;

  if ((config=fopen("<ARCbbs$MiscData>.Fido.Setup","r"))==NULL) return(ERROR);

  do
    {
    /* Read a line */
    line[0]=0;
    if (fgets(line,80,config)!=NULL)
      {
      if (line[0]!=';')
        {
        if (strncmp(line,"END",3)==0 || feof(config)) end=1;
        else
          {
          if (strncmp("OURNODE",line,7)==0)
            {       
            if (sscanf(line,"OURNODE %d:%d/%d.%d",&ouraddress.zone,
                       &ouraddress.net,&ouraddress.node,&ouraddress.point)!=4)
              {
              werr(0,"Bad OURNODE setup : %s",line);
              fclose(config);
              return(ERROR);
              }
            }
          if (strncmp("OURNAME",line,7)==0)
            {              
            strcpy(ourname,line+8); ourname[strlen(ourname)-1]=0;
            }
          if (strncmp("JUNKAREA",line,7)==0)
            {
            if (sscanf(line,"JUNKAREA %d",&junkarea)!=1)
              {
              werr(0,"Bad JUNKAREA setup : %s",line);
              fclose(config);
              return(ERROR);
              }
            }
          }
        }
      }
    }
  while(!feof(config) && end==0);

  /* Close the file */
  fclose(config);     

  if ((config=fopen("<ARCbbs$miscdata>.Fido.Areas","r"))==NULL)
    {
    werr(0,"Can't open Fido.Areas");
    return(ERROR);
    }

  end=fido_noofareas=0;
  do
    {
    /* Read a line */
    if (fgets(line,80,config)!=NULL)
      {
      if (line[0]!=';')
        {
        if (strncmp("END",line,3)==0 || feof(config)) end=1;
        else
          {
          if (sscanf(line,"%d %s ",&fido_bbsarea[fido_noofareas],
                                  fido_conference[fido_noofareas])==2) fido_noofareas++;
          }
        }
      }
    }
  while(!feof(config) && end==0);

  fclose(config);
 
  /* Return */
  return(OK);
  }

/*** Fidonet EXPORT **********************************************************/
#ifdef SERVER 
void cached_close()
  {
  int a;

  for(a=0;a<HANDLE_CACHE;a++)
    {
    if (cache[a].f)
      {
      fputc(0,cache[a].f); fputc(0,cache[a].f);
      _fclose(&cache[a].f);
      }
    }
  }

int fido_dooutbound()
  {                
  user_block board; FILE *areas; int un,end=0,a; char line[256];
  dbox export; int export_areas=0,export_messages=0,start;

  /* Lock */
  if (fido_inuse) return(OK);
  fido_inuse=1;

  /* Export dialog */
  export=dbox_new("export");
  dbox_showstatic(export);

  packetsgen=0;

  if ((un=hash_find("EchoMail"))==NODATA)
    {
    error("dCouldn't find EchoMail in userlog");
    dbox_dispose(&export); fido_inuse=0;
    return(NODATA);
    }
  board.usernumber=un;
  get_user(&board);

  /* If we're new to the area, don't export anything */
  if (board.m_message==0) board.m_message=mod_getlastlookup();

  /* Open areas file */
  if ((areas=fopen("<ARCbbs$miscdata>.Fido.Areas","r"))==NULL)
    {
    error("dCan't open Fido.Areas");
    dbox_dispose(&export); fido_inuse=0;
    return(ERROR);
    }

  syslog("FIDO out","Starting export");
  start=clock();

  do
    {
    /* Read a line */
    if (fgets(line,256,areas)!=NULL)
      {
      if (line[0]!=';')
        {
        if (strncmp("END",line,3)==0) end=1; 
        else
          {
          _address_arch writeto,last; int area,goodaddr; char areaname[30];

          if (sscanf(line,"%d %s",&area,areaname)==2)
            {
            char *destaddr,*destsave,*destend=(line+strlen(line));

            strtok(line," ");
            destsave=strtok(NULL," "); /* Throw away first 2 parameters */
            while(*destsave++);
            window_poll();
            dbox_setfield(export,4,areaname);
            dbox_setnumeric(export,0,++export_areas);

            while((destaddr=strtok(destsave," "))!=NULL && destsave<destend)
              {  
              destsave=destaddr;
              while(*destsave++);

              dbox_setfield(export,3,destaddr);

              goodaddr=0;

              if (sscanf(destaddr,"%d:%d/%d.%d",&writeto.zone,&writeto.net,&writeto.node,&writeto.point)!=4)
                {
                if (sscanf(destaddr,"%d:%d/%d",&writeto.zone,&writeto.net,&writeto.node)!=3)
                  {
                  if (sscanf(destaddr,"%d/%d.%d",&writeto.net,&writeto.node,&writeto.point)!=3)
                    {
                    if (sscanf(destaddr,"%d/%d",&writeto.net,&writeto.node)!=2)
                      {
                      if (sscanf(destaddr,"%d",&writeto.node)!=1)
                        {
                        if (sscanf(destaddr,".%d",&writeto.point)!=1)
                          {
                          goodaddr=0;
                          }
                        else { writeto.zone=last.zone; writeto.net=last.net; writeto.node=last.node; goodaddr=1; }
                        }
                      else { writeto.zone=last.zone; writeto.net=last.net; writeto.point=0; goodaddr=1; }
                      }
                    else { writeto.zone=last.zone; writeto.point=0; goodaddr=1; }
                    }
                  else { writeto.zone=last.zone; goodaddr=1; }
                  }
                else { writeto.point=0; goodaddr=1; }
                }
              else
                {
                goodaddr=1;
                }

              if (goodaddr)
                {
                int current_message,count=0; mail_area mailarea; mail_block header;
                read_area(area,&mailarea);

werr(0,"add: %d:%d/%d.%d",writeto.zone,writeto.net,writeto.node,writeto.point);
                last=writeto;

                packetsgen=0;
                if (mailarea.last_message>board.m_message)
                  {
                  current_message=board.m_message+1;

                  do
                    {
                    /* Get a message */
                    header.message_number=current_message; get_messageh(&header);

                    if (header.message_area==area && (header.flags&MSG_SYSOP)==0 && header.status==1)
                      {
                      int error=fido_makebundle(&writeto,current_message,areaname);

                      dbox_setnumeric(export,2,++export_messages);
                      dbox_setnumeric(export,1,packetsgen);

                      if (error!=OK)
                        {
                        fclose(areas);

                        dbox_dispose(&export); fido_inuse=0;

                        /* Close cached */
                        cached_close();

                        syslog("FIDO out","Abnormal export termination");
                        return(error);
                        }
                      count++; current_message=header.message_forward;
                      }
                    else
                      {
                      current_message+=1;
                      }
                    }
                  while(current_message<=mailarea.last_message && current_message!=-1);

                  mod_ensurelookup();
                  mod_ensuredata();
                  mod_savemaps();
                  }
                }
              }
            }
          }
        }
      }
    }
  while(!feof(areas) && end==0);

  /* Log the export */
  a=clock()-start;
  syslog("FIDO out","Exported %d messages in %d.%02ds",export_messages,a/100,a%100);

  /* Close the file */
  fclose(areas);

  /* Close cached */
  cached_close();

  board.m_message=mod_getlastlookup();
  put_user(&board);

  dbox_dispose(&export); fido_inuse=0;
  return(OK);
  }

int fido_makebundle(_address_arch *to,int messagenr,char *areaname)
  {
  char buffer[128],other[256],*packstr,*seenbys,*temp; FILE *bundle;
  struct tm *now; mail_block header;
  _packed_message *msg=(void*)buffer;
  int pointer,ex=0,ex2=0,a,noof_seenbys,otherlen,firstblock,
      toweight=(to->net<<16)+to->node,newline;
  struct { short net,node; } seenby[128];

  header.message_number=messagenr;
  get_message(&header,message_buffer);
  
  /* Do we have a ^A.SEENBY: line, if not, make one */
  if (strncmp(message_buffer,"\001.SEENBY: ",10)!=0)
    {
    memmove(message_buffer+37,message_buffer,header.message_length);
    sprintf(message_buffer,"\001.SEENBY: abcdefghijklmnopqrstuvwxyz");
    header.message_length+=37;
    }

  /* Find SEEN-BY lines */
  for (pointer=0;pointer<header.message_length && ex2==0;pointer++)
    {
    if (message_buffer[pointer]==0)
      {
      if (strncmp(message_buffer+pointer+1,"SEEN-BY:",8)==0)
        {
        pointer++; ex2=1; seenbys=message_buffer+pointer;
        }
      }
    }

  if (ex2==0)
    {
    error("dNo seenby line on msg#%d (area %d)",messagenr,header.message_area);
    return(ERROR);
    }

  /* Read in SEEN-BY lines */
  noof_seenbys=1;
  seenby[0].node=seenby[0].net=-1;

  while(strncmp(seenbys,"SEEN-BY:",8)==0)
    {
    char *id; int sblen=strlen(seenbys);

    /* Read in this line */
    id=strtok(seenbys+9," ");

    do
      {
      if (sscanf(id,"%hd/%hd",&seenby[noof_seenbys].net,
                              &seenby[noof_seenbys].node)!=2)
        {
        if (sscanf(id,"%hd",&seenby[noof_seenbys].node)==1)
          {
          if (noof_seenbys==1)
            {
            error("dError: no net in first seenby (msg#%d, area %d)",messagenr,header.message_area);
            return(ERROR);
            }
          seenby[noof_seenbys].net=seenby[noof_seenbys-1].net;
          }
        }
      id=strtok(NULL," ");
      noof_seenbys++;
      }
    while(id!=NULL);

    /* End of line, skip to next line */
    seenbys+=(sblen+1);
    }

  /* Make sure we don't export a message that has already been SEEN-BY the node */
  for(a=1;a<noof_seenbys;a++)
    {
    if(seenby[a].net==to->net && seenby[a].node==to->node && (to->point==0 && ouraddress.point==0)) return(OK);
    }

  /* If we're sending to a point, check we havn't sent already */
  if (ouraddress.zone==to->zone && ouraddress.net==to->net && ouraddress.node==to->node)
    {
    if (isupper(message_buffer[10+to->point]))
      {
      /* Already sent to that point */
      return(OK);
      }
    else
      {
      /* Mark as sent to point */
      message_buffer[10+to->point]='A'+to->point;
      }
    }

  /* Is bundle open? */
  a=-1;
  while(++a<HANDLE_CACHE) if (memcmp(to,&cache[a].to,sizeof(_address_arch))==0 && cache[a].f!=NULL) break;
  if (a==HANDLE_CACHE)
    {
    short trash1,trash2,trash3,trash4;

    /* Not open, find free cache slot */
    a=-1;
    while(++a<HANDLE_CACHE) if (cache[a].f==NULL) break;
    if (a==HANDLE_CACHE)
      {
      /* Close handle 0 */
      fputc(0,cache[0].f); fputc(0,cache[0].f); _fclose(&cache[a=0].f);
      }
    cache[a].to=(*to);
    open_packet(to->zone,to->net,to->node,to->point,&trash1,&trash2,&trash3,&trash4,&cache[a].f,0);
    if (cache[a].f==NULL)
      {
      error("dCan't open bundle to %d:%d/%d.%d",to->zone,to->net,to->node,to->point);
      return(ERROR);
      }
    }
  bundle=cache[a].f;

  if (ex!=1)
    {
    window_poll();

    /* Make a packed_message */
    for(a=0;a<34;a++) buffer[a]=0;
    msg->hdr_0=2; msg->hdr_1=0;

    /* Fill in addresses */
    put_rint(to->net,msg->destnet);
    put_rint(to->node,msg->destnode);
    put_rint(ouraddress.net,msg->orignet);
    put_rint(ouraddress.node,msg->orignode);

    /* Cost */
    put_rint(0,msg->cost);

    /* Creation time */
    now=localtime(&header.date_sent);
    strftime(msg->datetime,20,"%d %b %y  %H:%M:%S",now);
  
    /* Fido attributes & clear some bits */
    put_rint((header.flags>>16)&0x7413,msg->attribute);

    /* From/to/subject */
    packstr=msg->text;
    packstr+=sprintf(packstr,"%s",header.to)+1;
    packstr+=sprintf(packstr,"%s",header.from)+1;
    packstr+=sprintf(packstr,"%s",header.subject)+1;

    /* Write it to file */
    if (fwrite(buffer,packstr-buffer,1,bundle)!=1)
      {
      error("dCan't write Packed Message Header to file");
      return(ERROR);
      }

    /* Write AREA kludge to file */
    fprintf(bundle,"AREA:%s\015",areaname);
    
    /* Write FMPT if needed */
    if (ouraddress.point) fprintf(bundle,"\001FMPT %d\015",ouraddress.point);

    /* Write MSGID: if appropriate */
    if (header.from_id!=0)
      {
      fprintf(bundle,"\001MSGID: %d:%d/%d.%d %08x\015",ouraddress.zone,
                     ouraddress.net,ouraddress.node,ouraddress.point,
                     header.message_number);
      }

    /* If we're sending to a point... */
    if (to->point)
      {
      /* Zero out pointer */
      temp=NULL;

      /* Write text to file */
      for (a=37;a<pointer;a++)
        {
        if (message_buffer[a]==0) fputc(13,bundle);
        else fputc(message_buffer[a],bundle);
        }

      /* Copy anything past the SEEN-BYs into a safe place while the SEEN-BY list expands */
      otherlen=header.message_length-(seenbys-message_buffer);
      if (otherlen<0) otherlen=0; else memcpy(other,seenbys,otherlen);

      /* Regenerate SEEN-BYs */
      seenbys=message_buffer+pointer-1; a=1;
      while(a<noof_seenbys)
        {
        temp=seenbys;

        /* Start line off */
        seenbys+=sprintf(seenbys,"SEEN-BY:");
        newline=1;

        do
          {
          if (seenby[a-1].net==seenby[a].net && newline==0)
            seenbys+=sprintf(seenbys," %hd",seenby[a].node);
          else
            seenbys+=sprintf(seenbys," %hd/%hd",seenby[a].net,seenby[a].node);
          newline=0; a++;
          }
        while((seenbys-temp)<66 && a<noof_seenbys);
        seenbys++;
        }

      /* Copy back the 'after' bits */ 
      if (otherlen) memcpy(seenbys,other,otherlen);
      seenbys+=otherlen;

      /* Calculate new message length */
      header.message_length=seenbys-message_buffer;

      /* Zero out pointer */
      temp=NULL;

      /* Write new SEEN-BYs to file */
      for (a=pointer;a<(seenbys-message_buffer);a++)
        {
        if (message_buffer[a]==0)
          {
          fputc(13,bundle);
          if (strncmp(message_buffer+a+1,"\001PATH:",6)==0) temp=(message_buffer+a);
          }
        else fputc(message_buffer[a],bundle);
        }
      }
    else
      {
      /* Write text to file */
      for (a=37;a<pointer;a++)
        {
        if (message_buffer[a]==0) fputc(13,bundle);
        else fputc(message_buffer[a],bundle);
        }

      /* Copy anything past the SEEN-BYs into a safe place while the SEEN-BY list expands */
      otherlen=header.message_length-(seenbys-message_buffer);
      if (otherlen<0) otherlen=0; else memcpy(other,seenbys,otherlen);

      /* Regenerate SEEN-BYs with destination node inserted */
      seenbys=message_buffer+pointer-1; ex2=0; a=1;
      while(a<noof_seenbys)
        {
        temp=seenbys;

        /* Start line off */
        seenbys+=sprintf(seenbys,"SEEN-BY:");

        do
          {
          int oldweight=(seenby[a-1].net<<16)+seenby[a-1].node,
              newweight=(seenby[a].net<<16)+seenby[a].node;

          /* Kludge for points sending to bosses */
          if (oldweight==toweight || newweight==toweight) ex2=1;

          if (oldweight<toweight && toweight<newweight && ex2==0)
            {
            /* Add the destination into the seen-by list at the right place */
            if (seenby[a-1].net==to->net) seenbys+=sprintf(seenbys," %d",to->node);
            else seenbys+=sprintf(seenbys," %d/%d",to->net,to->node);
            ex2=1;

            /* And carry on with the list */
            if (to->net==seenby[a].net) seenbys+=sprintf(seenbys," %hd",seenby[a].node);
            else seenbys+=sprintf(seenbys," %hd/%hd",seenby[a].net,seenby[a].node);
            }
          else
            {
            if (seenby[a-1].net==seenby[a].net) seenbys+=sprintf(seenbys," %hd",seenby[a].node);
            else seenbys+=sprintf(seenbys," %hd/%hd",seenby[a].net,seenby[a].node);
            }
          a++;
          }
        while((seenbys-temp)<66 && a<noof_seenbys);
        seenbys++;
        }

      /* Incase it would be the last in the list */
      if (ex2==0)
        {
        seenbys--;
        if (seenby[a-1].net==to->net) seenbys+=sprintf(seenbys," %hd",to->node);
        else seenbys+=sprintf(seenbys," %hd/%hd",to->net,to->node);
        seenbys++;
        }

      /* Copy back the 'after' bits */ 
      if (otherlen) memcpy(seenbys,other,otherlen);
      seenbys+=otherlen;

      /* Calculate new message length */
      header.message_length=seenbys-message_buffer;

      /* Zero out pointer */
      temp=NULL;

      /* Write new SEEN-BYs to file */
      for (a=pointer;a<(seenbys-message_buffer);a++)
        {
        if (message_buffer[a]==0)
          {
          fputc(13,bundle);
          if (strncmp(message_buffer+a+1,"\001PATH:",6)==0) temp=(message_buffer+a);
          }
        else fputc(message_buffer[a],bundle);
        }
      }

    /* Delete old text chain */
    if (_mail_freeup(header.block_forward)!=OK)
      {
      error("dError whilst freeing up chain for msg#%d",
           header.message_number);
      return(ERROR);
      }

    /* Chain on new text */
    if ((firstblock=mod_lockdata(header.message_length))==NOROOM)
      {
      error("dCouldn't allocate new mail text block");
      return(ERROR);
      }

    /* Write new text */
    header.block_forward=firstblock;
    put_messagehi(&header);
    if (_mail_buffertodatai(message_buffer,header.message_length,firstblock)!=OK)
      {
      error("dCouldn't write new text");
      return(ERROR);
      }

    /* Check last line */
    if (temp!=NULL)
      {
      /* There is a path line, how long is it? */
      if (strlen(temp)>66)
        {
        /* Too long, write new line to file */
        fseek(bundle,-2,SEEK_CUR);
        fprintf(bundle,"\001PATH: %d/%d\015",ouraddress.net,ouraddress.node);
        }
      else
        {
        /* Tack ourselves onto end */
        fseek(bundle,-3,SEEK_CUR);
        if (isdigit(fgetc(bundle))==0) fseek(bundle,-1,SEEK_CUR);
        fflush(bundle);
        fprintf(bundle," %d/%d\015",ouraddress.net,ouraddress.node);
        }
      }
    else
      {
      /* No PATH: line, add our own */
      fprintf(bundle,"\001PATH: %d/%d\015",ouraddress.net,ouraddress.node);
      }

    /* Zero to terminate message text */
    fputc(0,bundle);

    packetsgen++;
    }

  return(OK);
  }
#endif

/*** Fidonet EXPORT MAIL *****************************************************/
int fido_domail(mail_block *header,int isus)
  {
  char buffer[256],*packstr;
  int pointer,a; short re_zone,re_net,re_node,re_point;
  FILE *bundle; struct tm *tmtime;
  _packed_message *msg=(void*)buffer;

  /* Open bundle - do not route 'express' messages */
  open_packet(header->fidoto.zone,header->fidoto.net,header->fidoto.node,header->fidoto.point,
              &re_zone,&re_net,&re_node,&re_point,&bundle,(header->flags&MSG_EXPRESS)==0?1:0);

  if (bundle==NULL) return(ERROR);

  /* Make a packed_message */
  for(a=0;a<34;a++) buffer[a]=0;

  msg->hdr_0=2; msg->hdr_1=0;

  /* Fill in addresses */
  put_rint(re_net,msg->destnet);
  put_rint(re_node,msg->destnode);
  if (isus)
    {
    put_rint(ouraddress.net,msg->orignet);
    put_rint(ouraddress.node,msg->orignode);
    }
  else
    {
    put_rint(header->fidofrom.net,msg->orignet);
    put_rint(header->fidofrom.node,msg->orignode);
    }

  /* Cost */
  put_rint(0,msg->cost);

  /* Creation time */
  tmtime=localtime(&header->date_sent);
  strftime(msg->datetime,20,"%d %b %y  %H:%M:%S",tmtime);

  /* Fido attributes & clear some bits */
  header->flags=MSG_PRIVATE;
  put_rint((header->flags>>16)&0x7413,msg->attribute);

  /* From/to/subject length */
  packstr=msg->text;
  packstr+=sprintf(packstr,"%s",header->to)+1;
  packstr+=sprintf(packstr,"%s",header->from)+1;
  packstr+=sprintf(packstr,"%s",header->subject)+1;

  /* Write it to file */
  if (fwrite(buffer,packstr-buffer,1,bundle)!=1)
    {
    error("dCan't write packed message header to file");
    _fclose(&bundle);
    return(ERROR);
    }

  for (pointer=0;pointer<header->message_length;pointer++)
    {
    if (message_buffer[pointer]==0) message_buffer[pointer]=13;
    }
  message_buffer[pointer]=0;

  if (isus)
    {
    /* Check for interzone messages, add ^AINTL if necessary */
    if (header->fidoto.zone!=ouraddress.zone)
      {
      fprintf(bundle,"\001INTL %hd:%hd/%hd %hd:%hd/%hd\015",
              header->fidoto.zone,header->fidoto.net,header->fidoto.node,
              ouraddress.zone,ouraddress.net,ouraddress.node);
      }

    /* Check for to points and if necessary add ^ATOPT or ^AFMPT line */
    if (ouraddress.point) fprintf(bundle,"\001FMPT %hd\015",ouraddress.point);
    if (header->fidoto.point) fprintf(bundle,"\001TOPT %hd\015",header->fidoto.point);
    }

  /* Write text to file */
  fwrite(message_buffer,header->message_length+1,1,bundle);
   
  /* Write bundle trailer */
  buffer[0]=0; buffer[1]=0;
  if (fwrite(buffer,2,1,bundle)!=1)
    {
    error("dCan't write bundle trailer");
    _fclose(&bundle);
    return(ERROR);
    }

  _fclose(&bundle);
  return(OK);
  }

/*** Fidonet IMPORT **********************************************************/
#ifdef SERVER
void fido_doinbound(char *wildcard) /* Process inbound PKTs */
  {
  os_regset inout; char gbpbbuffer[16],buffer[256],bigname[60],monthtxt[4],areaname[20],*packstr;
  _address_arch packetorig; dbox import;
  int import_packets=0,import_echomail=0,import_matrix=0,route_matrix=0,start,lastmsg=clock(),
      temp,truncate,a,b,keeppacket;
  static char months[][4]={ "Jan","Feb","Mar","Apr","May","Jun",
                            "Jul","Aug","Sep","Oct","Nov","Dec" };
  FILE *packet; mail_block header;
  _packet_header *pktheader=(_packet_header*)buffer;
  _packed_message *msg=(_packed_message*)buffer;
  struct tm fidotime;
 
  /* Open dialog */
  import=dbox_new("import");
  dbox_showstatic(import);

  /* Log entry */
  syslog("FIDO in","Started import");
  start=clock();

  inout.r[4]=0;
  do
    {
nextname:
    inout.r[0]=9; /* Names only */
    inout.r[1]=(int)"<ARCbbs$Inbound>";
    inout.r[2]=(int)gbpbbuffer;
    inout.r[3]=1; /* Read one item */
    inout.r[5]=16;
    inout.r[6]=(int)wildcard;
    os_swi(0x0c,&inout); /* OS_GBPB */

    if (inout.r[3]>0)
      {
      /* Make filename */
      sprintf(bigname,"<ARCbbs$Inbound>.%s",gbpbbuffer);
      keeppacket=0;

      dbox_setnumeric(import,1,++import_packets);

      /* Open file */
      if ((packet=fopen(bigname,"rb"))==NULL)
        {
        error("cCan't open %s",bigname);
        syslog("FIDO in","Abnormal termination, can't open packet %s",bigname);
        return;
        }

      /* How long is file? */
      fseek(packet,0,SEEK_END);
      if (ftell(packet)<10)
        {
        /* Must be a bad pkt */
        fclose(packet);
        goto nextname;
        }
      fseek(packet,0,SEEK_SET);
      
      /* Read header */
      if (fread(pktheader,58,1,packet)!=1)
        {
        fclose(packet);
        error("cCan't read packet-header of %s",bigname);
        syslog("FIDO in","Abnormal termination, can't read packet-header of %s",bigname);
        return;
        }  

      if (pktheader->h_0!=2 || pktheader->h_1!=0)
        {
        fclose(packet);
        error("cIllegal packet header in %s",bigname);
        syslog("FIDO in","Abnormal termination, illegal packet-header in %s",bigname);
        return;
        }

      /* Get originator of packet */
      if ((packetorig.zone=get_rint(pktheader->qorigzone))==0) get_rint(pktheader->origzone);
      if (packetorig.zone==0) packetorig.zone=ouraddress.zone;
      packetorig.net=get_rint(pktheader->orignet);
      packetorig.node=get_rint(pktheader->orignode);
      packetorig.point=get_rint(pktheader->origpoint);

      /* Process packet! */
      do
        {
        truncate=0;

        /* Poll just to keep system ticking */
        if ((clock()-lastpoll)>MAXIMUM_POLL) window_poll();

        /* Read in message header */
        if (fread(msg,1,14,packet)!=14)
          {
          /* Missing header */
          error("cMissing packet terminator in packet '%s'",bigname);
          
          keeppacket=1;
          break;
          }

        if (msg->hdr_0==0 && msg->hdr_1==0) break;
        if (msg->hdr_0!=2 || msg->hdr_1!=0)
          {
          /* Bad header somewhere! */
          error("cBad message header in packet '%s'",bigname);

          keeppacket=1;
          break;
          }
            
        memset(&header,0,256);

        /* Get from/to/date */
        header.fidoto.zone=ouraddress.zone;
        header.fidoto.net=get_rint(msg->destnet);
        header.fidoto.node=get_rint(msg->destnode);
        header.fidoto.point=0;
        header.fidofrom.zone=ouraddress.zone;
        header.fidofrom.net=get_rint(msg->orignet);
        header.fidofrom.node=get_rint(msg->orignode);
        header.fidofrom.point=0;
        header.flags=MSG_FIDO|(get_rint(msg->attribute)<<16);
        packstr=msg->datetime;
        do { *packstr++=temp=fgetc(packet); } while(temp);
        packstr=header.to; 
        do { *packstr++=temp=fgetc(packet); } while(temp);
        packstr=header.from;
        do { *packstr++=temp=fgetc(packet); } while(temp);
        packstr=header.subject;
        do { *packstr++=temp=fgetc(packet); } while(temp);
        header.reply_from=-1; header.file_length=0; header.file_location=-1;
        if (sscanf(msg->datetime,"%d %s %d %d:%d:%d",&fidotime.tm_mday,
                   monthtxt,&fidotime.tm_year,&fidotime.tm_hour,
                   &fidotime.tm_min,&fidotime.tm_sec)==6)
          {
          fidotime.tm_mon=0;
          while(strcmp(monthtxt,months[fidotime.tm_mon])) fidotime.tm_mon++;
          header.date_sent=mktime(&fidotime);
          }
        else
          {
          if (sscanf(msg->datetime,"%d %s %d %d:%d",&fidotime.tm_mday,
                     monthtxt,&fidotime.tm_year,&fidotime.tm_hour,
                     &fidotime.tm_min)==5)
            {
            fidotime.tm_mon=0; fidotime.tm_sec=0;
            while(strcmp(monthtxt,months[fidotime.tm_mon])) fidotime.tm_mon++;
            header.date_sent=mktime(&fidotime);
            }
          else
            {
            if (sscanf(msg->datetime,"%*s %d %s %d %d:%d",&fidotime.tm_mday,
                       monthtxt,&fidotime.tm_year,&fidotime.tm_hour,
                       &fidotime.tm_min)==5)
              {
              fidotime.tm_mon=0; fidotime.tm_sec=0;
              while(strcmp(monthtxt,months[fidotime.tm_mon])) fidotime.tm_mon++;
              header.date_sent=mktime(&fidotime);
              }
            else
              {
              /* Bodge it and say it was 1 Jan 1970 00:00 */
              header.date_sent=0;
              error("cBad datetime format: %s, bodging",msg->datetime);
              }
            }
          }

        /* Date message arrived here */
        header.date_entered=time(NULL);

        /* Read in message text */
        a=0;
        do
          {
          do { b=fgetc(packet); } while(b==10);
          if (truncate==0)
            {
            if (b==13 || b==141) message_buffer[a++]=0;
            else message_buffer[a++]=b;

            if (a>15000)
              {
              truncate=1;
              message_buffer[a++]=0;
              a+=(sprintf(message_buffer+a,"[Message too long: It has been truncated]")+1);
              syslog("FIDO in","Message truncated");
              }
            }
          }
        while(b && b!=EOF);
        header.message_length=a;
                                              
        /* Check for 'AREA' line at the top */
        if (sscanf(message_buffer,"AREA:%s",areaname)==1 ||
            sscanf(message_buffer,"\001AREA:%s",areaname)==1)
          {
          int count,end=0;

          /* It IS in an area, figure out what ARCbbs area this
             translates to */
          for(count=0;count<fido_noofareas && end==0;count++)
            {
            if (strcmp(areaname,fido_conference[count])==0)
              {
              /* We have correct mail area, import message */
              end=2; header.message_area=fido_bbsarea[count];
              header.message_length-=(b=(strlen(message_buffer)+1));

              /* Give it a .SEENBY line */
              memmove(message_buffer+b+37,message_buffer+b,header.message_length);

              /* If we're a point, we're importing from our boss and hence he will
                 already have seen it. If we're not, it doesn't matter as we have
                 already seen it ourselves (ie we are .0) */
              sprintf(message_buffer+b,"\001.SEENBY: Abcdefghijklmnopqrstuvwxyz");

              /* If it's from one of our points, include them automagically in the
                 .SEENBY line */
              if (ouraddress.zone==packetorig.zone && ouraddress.net==packetorig.net &&
                  ouraddress.node==packetorig.node && packetorig.point!=0 && ouraddress.point==0)
                {
                message_buffer[b+10+packetorig.point]=('A'+packetorig.point);
                }

              header.message_length+=37;

              put_messagei(&header,message_buffer+b);

              import_echomail++;
              if ((clock()-lastmsg)>100)
                {
                dbox_setnumeric(import,2,import_echomail);
                lastmsg=clock();
                }
              }
            }

          if (end!=2)
            {
            header.message_area=junkarea;
            header.message_length-=(b=(strlen(message_buffer)+1));
            put_messagei(&header,message_buffer+b);
            }
          }
        else
          {
          /* Is netmail: Process special control lines */
          char *kludgept=message_buffer; short temp1,temp2;

          do
            {
            if (kludgept[0]==1) /* Kludge line */
              {
              /* Something at start of message is it an INTL line? */
              if (sscanf(kludgept+1,"INTL %hd:%*d/%*d %hd:%*d/%*d",
                                                    &temp1,&temp2)!=2)
                {
                /* MSGID: line? */
                if (sscanf(kludgept+1,"MSGID: %hd:%*d/%*d.%*d %*x",
                                                  &temp1)!=1)
                  {
                  /* A FMPT line? */
                  if (sscanf(kludgept+1,"FMPT %hd",&temp1)!=1)
                    {
                    /* A TOPT line? */
                    if (sscanf(kludgept+1,"TOPT %hd",&temp1)==1) header.fidoto.point=temp1;
                    }
                  else header.fidofrom.point=temp1;
                  }
                else header.fidofrom.zone=temp1;
                }
              else
                {
                header.fidoto.zone=temp1;
                header.fidofrom.zone=temp2;
                }
              }
            kludgept+=(strlen(kludgept)+1);
            }
          while(kludgept<(message_buffer+header.message_length));

          /* Is the message addressed to us? */
          if (header.fidoto.zone ==ouraddress.zone &&
              header.fidoto.net  ==ouraddress.net &&
              header.fidoto.node ==ouraddress.node &&
              header.fidoto.point==ouraddress.point)
            {
            /* Private message - try and locate username */
            header.message_area=0; /* Private mail */
            if ((header.to_id=hash_find(header.to))==NODATA)
              {
              char *anothertry;

              /* Try looking name up in tranlate file */
              anothertry=fido_rules2(header.to);
              if ((header.to_id=hash_find(anothertry))==NODATA)
                {
                if (stricmp(header.to,"areafix")==0)
                  {
                  FILE *areafix;

                  /* Nope, couldn't find user so direct it to Sysop */
                  header.to_id=1;

                  /* It's to areafix, stick it in a file */
                  if ((areafix=fopen("<ARCbbs$areafix>","a"))!=NULL)
                    {
                    fprintf(areafix,"::: AREAFIX REQUEST\n");
                    fprintf(areafix,"From   : %s (%d:%d/%d.%d)\n",header.from,
                                                                 header.fidofrom.zone,
                                                                 header.fidofrom.net,
                                                                 header.fidofrom.node,
                                                                 header.fidofrom.point);
                    fprintf(areafix,"Subject: %s\n",header.subject);
                    fwrite(message_buffer,1,header.message_length,areafix);
                    fprintf(areafix,"::: END\n\n");
                    fclose(areafix);

                    header.to_id=-1;
                    }
                  }
                }
              }
            if (header.to_id>0) put_messagei(&header,message_buffer);

            import_matrix++;
            if ((clock()-lastmsg)>100)
              {
              dbox_setnumeric(import,3,import_matrix);
              lastmsg=clock();
              }
            }
          else
            {
            char *eom=(message_buffer+header.message_length),buffer[40];
            time_t now; struct tm *nowa;

            /* Re-route. Add a tag line to the bottom to indicate routing */

            time(&now); nowa=localtime(&now);
            strftime(buffer,30,"%a %b %d at %H:%M:%S",nowa);
            header.message_length+=sprintf(eom,"Via ARCbbs %d:%d/%d, %s",
                                           ouraddress.zone,ouraddress.net,ouraddress.node,
                                           buffer)+1;
            fido_domail(&header,0);
            route_matrix++;
            }
          }
        }
      while(msg->hdr_0==2 && msg->hdr_1==0 && !feof(packet));

      fclose(packet); /* Close packetfile */

      /* Be safe! */
      mod_ensurelookup();
      mod_ensuredata();
      mod_savemaps();

      if (keeppacket)
        {
        char tmp[256];

        strcpy(tmp,bigname);
        strcpy(tmp+strlen(tmp)-2,"!!");
        if (rename(bigname,tmp))
          {
          /* If there are problems, return here! */
          dbox_dispose(&import);
          return;
          }
        }
      else
        {
        if (remove(bigname)) /* And delete it */
          {
          /* If there are problems, return here! */
          dbox_dispose(&import);
          return;
          }
        inout.r[4]=0;
        }
      }
    }
  while(inout.r[4]!=-1);

  /* System log entry */
  a=clock()-start;
  syslog("FIDO in","Imported - %d echomail, %d netmail. Routed - %d netmail",import_echomail,import_matrix,route_matrix);
  syslog("FIDO in","%d packets, completed in %d.%02ds",import_packets,a/100,a%100);

  /* Get rid of dialog */
  dbox_dispose(&import);
  }

void fido_dearc() /* Dearc incoming ARCmail into .PKT's */
  {
  os_regset inout; char gbpbbuffer[16];
  dbox dearc;

  dearc=dbox_new("dearc");
  dbox_showstatic(dearc);

  inout.r[4]=0;

  do
    {
    inout.r[0]=9; /* Names only */
    inout.r[1]=(int)"<ARCbbs$Inbound>";
    inout.r[2]=(int)gbpbbuffer;
    inout.r[3]=1; /* Read one item */
    inout.r[5]=16;
    inout.r[6]=(int)"*";
    os_swi(0x0c,&inout); /* OS_GBPB */

    if (inout.r[3]>0)
      {
      int a,b=1;

      /* Check that file is ARCmail */
      for(a=0;a<8;a++) if(ishex(gbpbbuffer[a])==0) b=0;
      if (isalpha(gbpbbuffer[8]) && isdigit(gbpbbuffer[9]) && b==1)
        {
        char bigname[60],bigname2[60];
        clock_t st;

        /* Make filename */
        sprintf(bigname,"<ARCbbs$Inbound>.%s",gbpbbuffer);

        dbox_setfield(dearc,0,gbpbbuffer);

        /* It's ARCmail alright, dearc it! (into inbound directory) */
        do_arc(bigname,"","-x","<ARCbbs$Inbound>");
        
        /* And rename it so we won't process it again */
        strcpy(bigname2,bigname);

        /* Swap last 2 characters */
        b=strlen(bigname2);
        a=bigname2[b-1]; bigname2[b-1]=bigname2[b-2]; bigname2[b-2]=a;

        /* Poll becase sometimes where is time delay beteen ARC returning
           and file being closed */
        while(rename(bigname,bigname2)) window_poll();

/* Delete arcmail *********************************************************/
        st=clock(); while((clock()-st)<100) window_poll();

        if (getenv("ARCbbs$oldarcmail")!=NULL)
          {
          char newname[40];

          /* Save away old arcmail */
          sprintf(newname,"<ARCbbs$oldarcmail>.%08x",time(NULL));
          
          /* Rename to move */
          rename(bigname2,newname);
          }

        /* Delete it */
        remove(bigname2);
        
        /* Restart scan from top */
        inout.r[4]=0;
/**************************************************************************/
        }
      }
    }
  while(inout.r[4]!=-1);

  dbox_dispose(&dearc);
  }

int ishex(int a)
  {
  if (isdigit(a)) return(TRUE);
  a=tolower(a);
  if (a>='a' && a<='f') return(TRUE);
  return(FALSE);
  }
#endif

int checkhold(int zone,int net,int node,int point)
  {
  char holdread[80]; int in_zone,in_net,in_node,in_point;
  FILE *holdfile;

  /* Open translation file */
  if ((holdfile=fopen("<ARCbbs$miscdata>.Fido.Hold","r"))==NULL)
    {
    return('H');
    }

  do
    {
    /* Read lines */
    mygets(holdread,holdfile);
    
    if (holdread[0] && holdread[0]!=';')
      {
      if (sscanf(holdread+2,"%d:%d/%d.%d",&in_zone,&in_net,&in_node,&in_point)==4)
        {
        if (in_zone==zone && in_net==net && in_node==node && in_point==point)
          {
          fclose(holdfile);
          return(toupper(holdread[0]));
          }
        }

      if (sscanf(holdread+2,"%d:%d/%d",&in_zone,&in_net,&in_node)==3)
        {
        if (in_zone==zone && in_net==net && in_node==node)
          {
          fclose(holdfile);
          return(toupper(holdread[0])); 
          }
        }
      }
    }
  while(holdread[0]);

  fclose(holdfile);
  return('O');
  }

void open_packet(int zone,int net,int node,int point,
                 short *m_zone,short *m_net,short *m_node,short *m_point,
                 FILE **bundle,int mail)
  {
  int  r_zone=zone,r_net=net,r_node=node,r_point=point,
       in_zone,in_net,in_node,out_zone,out_net,out_node,
       node_mapped=0,hold;
  FILE *routefile; char fline[80],*dpoint,*lower;
  time_t nowt; struct tm *now;
  _packet_header bundleh;

  *m_zone=zone; *m_net=net; *m_node=node; *m_point=point;
          
  /* Only re-route mail, not echomail */
  if (mail)
    {
    /* Open route file */
    if ((routefile=fopen("<ARCbbs$miscdata>.Fido.Route","r"))!=NULL)
      {
      do
        {
        /* Read lines */
        mygets(fline,routefile);
        if (fline[0] && fline[0]!=';')
          {
          /* Get verb in lowercase */
          *strchr(fline,' ')=0; dpoint=fline+strlen(fline)+1;
          lower=fline; while(*lower) *lower++=tolower(*lower);

          /* Direct for bypass */
          if (strcmp(fline,"direct")==0 && node_mapped==0)
            {
            if (sscanf(dpoint,"%d:%d/%d",&in_zone,&in_net,&in_node)==3)
              {
              if (in_zone==r_zone && in_net==r_net && in_node==r_node) node_mapped=1;
              }
            else
            if (sscanf(dpoint,"%d:%d",&in_zone,&in_net)==2)
              {
              if (in_zone==r_zone && in_net==r_net) node_mapped=1;
              }
            }

          /* Node mapping */
          if (strcmp(fline,"nodemap")==0 && node_mapped==0)
            {
            if (sscanf(dpoint,"%d:%d/%d %d:%d/%d",&in_zone,&in_net,&in_node,
                                              &out_zone,&out_net,&out_node)==6)
              {
              if (in_zone==r_zone && in_net==r_net && in_node==r_node)
                {
                r_zone=out_zone; r_net=out_net; r_node=out_node;
                }
              }
            else
            if (sscanf(dpoint,"%d:%d/%d",&out_zone,&out_net,&out_node)==3)
              {
              r_zone=out_zone; r_net=out_net; r_node=out_node;
              }
            }

          /* Net mapping */
          if (strcmp(fline,"netmap")==0 && node_mapped==0)
            {
            if (sscanf(dpoint,"%d:%d %d:%d/%d",&in_zone,&in_net,
                                               &out_zone,&out_net,&out_node)==5)
              {
              if (in_zone==r_zone && in_net==r_net)
                {
                r_zone=out_zone; r_net=out_net; r_node=out_node;
                }
              }
            }

          /* Zone mapping */
          if (strcmp(fline,"zonemap")==0 && node_mapped==0)
            {
            if (sscanf(dpoint,"%d %d:%d/%d",&in_zone,
                                               &out_zone,&out_net,&out_node)==4)
              {
              if (in_zone==r_zone)
                {
                if (r_zone!=out_zone)
                  {
                  /* We're remapping this zone, address mail to the ZoneGate */
                  r_zone=out_zone; r_net=out_net; r_node=out_node;    
                  *m_zone=out_zone; *m_net=out_zone; *m_node=in_zone;
                  }
                else
                  {
                  r_net=out_net; r_node=out_node;
                  }
                }
              }
            }
          }
        }
      while(fline[0]);
      fclose(routefile);
      }
    }

  /* Check hold status */
  hold=checkhold(r_zone,r_net,r_node,r_point);

  /* Open bundle */
  if (ouraddress.zone==r_zone)
    {
    if (r_point!=0 && ouraddress.zone==r_zone && ouraddress.net==r_net && ouraddress.node==r_node && ouraddress.point==0)
      {
      sprintf(file,"Cdir <ARCbbs$OutboundBase>.Outbound.%04x%04xPN",r_net,r_node);
      os_cli(file);
      sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04xPN.%04x%cT",r_net,r_node,r_point,hold);
      }
    else sprintf(file,"<ARCbbs$OutboundBase>.Outbound.%04x%04x%cT",r_net,r_node,hold);
    }
  else
    {
    sprintf(file,"<ARCbbs$OutboundBase>.Outbound%02x.%04x%04x%cT",r_zone,r_net,r_node,hold);
    }

  if ((*bundle=fopen(file,"rb+"))==NULL)
    {
    /* bundle does not exist, create one */
    if ((*bundle=fopen(file,"wb+"))==NULL)
      {
      error("dCan't open outbound bundle (%s)",file);
      return;
      }

    /* Make packet header */
    memset(&bundleh,0,58);

    /* Fill in addresses - address packet to routed address */
    put_rint(r_zone,bundleh.qdestzone);
    put_rint(r_zone,bundleh.destzone);
    put_rint(r_net,bundleh.destnet);
    put_rint(r_node,bundleh.destnode);
    put_rint(r_point,bundleh.destpoint);
    put_rint(ouraddress.zone,bundleh.qorigzone);
    put_rint(ouraddress.zone,bundleh.origzone);
    put_rint(ouraddress.net,bundleh.orignet);
    put_rint(ouraddress.node,bundleh.orignode);
    put_rint(ouraddress.point,bundleh.origpoint);

    /* Fill in time */
    time(&nowt); now=localtime(&nowt);
    put_rint(now->tm_year+1900,bundleh.year);
    put_rint(now->tm_mon+1,bundleh.month);
    put_rint(now->tm_mday,bundleh.day);
    put_rint(now->tm_hour,bundleh.hour);
    put_rint(now->tm_min,bundleh.minute);
    put_rint(now->tm_sec,bundleh.second);

    /* Baud rate ?! */
    put_rint(0,bundleh.baud);

    /* Fill in version */
    bundleh.h_0=2; bundleh.h_1=0;
    bundleh.productcodeL=0xff;

    /* Write it to file */
    if (fwrite(&bundleh,58,1,*bundle)!=1)
      {
      error("dCan't write PacketHeader to file");
      fclose(*bundle); *bundle=0;
      return;
      }
    }
  else
    {
    /* Skip to 2 bytes before end of current packet to remove end */
    fseek(*bundle,-2,SEEK_END);
    }
  }

#ifdef SERVER
extern int got_type;
void do_arc(char *arcname,char *arcname1,char *command,char *files)
  {
  wimp_msgstr message;

  if (command==NULL) command="";

  sprintf(message.data.chars,"arc -y%d %s %s%s %s",MAXIMUM_POLL,
          command,arcname,arcname1,files);

  message.hdr.size=4*((24+strlen(message.data.chars))/4+1);
  message.hdr.your_ref=0;
  message.hdr.action=BBS_ARC;
  wimp_sendmessage(wimp_ESEND,&message,0);

  /* Extra one to catch broadcast to ourselves */
  got_type=0; while(got_type!=BBS_ARC) window_poll();
  got_type=0; while(got_type!=BBS_ARC) window_poll();
  }

void mygets(char *buffer,FILE *file)
  {
  int ch,ch2;

  if (feof(file))
    {
    *buffer=0; return;
    }

  do
    {
    ch=*(buffer++)=fgetc(file);
    }
  while(ch!=10 && ch!=13 && !feof(file)); 

  /* Wipe out any following CR or LF */
  ch2=fgetc(file);
  if (ch2!=13 && ch2!=10) ungetc(ch2,file);
  else
    {
    if ((ch2==10 && ch==10) || (ch2==13 && ch==13)) ungetc(ch2,file);
    }

  *(buffer-1)=0;
  }
#endif

char *fido_rules2(char *realname)
  {
  static char return_name[31],pseud[31];
  FILE *a;

  /* Open translation file */
  if ((a=fopen("<ARCbbs$miscdata>.Fido.Translate","r"))==NULL)
    {
    return(realname);
    }

  do
    {
    /* Read lines */
    mygets(pseud,a);
    mygets(return_name,a);

    /* Match? */
    if (strcmp(return_name,realname)==0)
      {
      fclose(a);
      return(pseud);
      }
    }
  while(pseud[0]);

  fclose(a);
  return(realname);
  }

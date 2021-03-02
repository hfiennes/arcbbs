/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Fido write mail                             <]
Current version   [> 00.08                                       <]
Version date      [> 02-April-1993                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"         /* Standard library files */
#include "config.h"          /* System configuration */
#include "servmess.h"        /* Server message info */
#include "port.h"            /* Port driver functions */
#include "userlog.h"         /* Userlog format */
#include "crc.h"             /* CRC generator */
#include "parser.h"          /* Command parser */
#include "mail.h"            /* Mail format */
#include "portmisc.h"        /* Misc port commands */
#include "servcomm.h"        /* Misc server comm commands */
#include "mprintf.h"         /* Modem printf */
#include "arclist.h"         /* ARC file lister */
#include "modcomm.h"         /* Module interface */
#include "fido.h"            /* Fido stuff */
#include "kernel.h"

#include "miscbbs.h"
#include "bbs.h"
                    
void message_fidowrite(_address_arch2 *to,char *whoto,char *subject,int reply)
  {
  mail_block header; char tempaddress[20];
  NodelistDAT node_find; int clr;
                                      
  setup_mailheader(&header,&user,0);

  mprintf(1,"\n{bfg c}Area   : {fg w}Fidonet Netmail\n{fg c}Date   : {fg w}%s\n{fg c}From   : {fg w}%s\n",cctime(&header.date_sent),user.username);
                                
  if (whoto==NULL)
    {
    port_txstring("\015{bfg c}To     : ",1);
    port_readline(header.to,30,0); port_crlf();
    if (header.to[0]==0) return;
    }
  else
    {
    mprintf(1,"\015{bfg c}To     : {fg w}%s\n",strcpy(header.to,whoto));
    }

  if (to==NULL)
    {
    tryaddragain:
    port_txstring("\015{fg c}At     : ",1);
    port_readline(tempaddress,19,0);
    if (*tempaddress==0) { port_txstring("\n{std bfg r}Netmail send aborted\n",1); return; }

    header.flags|=MSG_FIDO;
    if (sscanf(tempaddress,"%hd:%hd/%hd.%hd",&header.fidoto.zone,
                                         &header.fidoto.net,
                                         &header.fidoto.node,
                                         &header.fidoto.point)!=4)
      {
      if (sscanf(tempaddress,"%hd/%hd.%hd",&header.fidoto.net,
                                        &header.fidoto.node,
                                        &header.fidoto.point)!=3)
        {
        if (sscanf(tempaddress,"%hd:%hd/%hd",&header.fidoto.zone,
                                        &header.fidoto.net,
                                        &header.fidoto.node)!=3)
          {
          if (sscanf(tempaddress,"%hd/%hd",&header.fidoto.net,
                                         &header.fidoto.node)!=2)
            {
            goto tryaddragain;
            }
          else
            {
            header.fidoto.zone=ouraddress.zone;  
            header.fidoto.point=0;
            }
          }
        else
          {
          header.fidoto.point=0;
          }
        }
      else
        {
        header.fidoto.zone=ouraddress.zone;
        }
      }
    }
  else
    {
    /* Preaddressed (ie reply) */
    header.fidoto=*to;
    }  
  header.reply_from=reply;

  if ((clr=search_nodelist(&header.fidoto,&node_find))==-1)
    {
    strcpy(node_find.QL_Name+1,"Unknown");
    node_find.QL_Name[0]=7;
    }

  if (clr==0 && to!=NULL) return;
  if (clr==0) goto tryaddragain;
  node_find.QL_Name[node_find.QL_Name[0]+1]=0; /* Pascal string->C string */

  mprintf(1,"\015{bfg c}Address: {fgbg wb}%hd:%hd/%hd.%hd {fg c}(%s){std}\n",
                                          header.fidoto.zone,
                                          header.fidoto.net,
                                          header.fidoto.node,
                                          header.fidoto.point,
                                          node_find.QL_Name+1);
  strcpy(header.subject,subject);


  if (entertext(&header,0))
    {
    mprintf(1,"\n{bfg g}Adding message (%d bytes) to outgoing packet...",header.message_length);
    if (fido_domail(&header,1)==OK)
      {
      mprintf(1,"Done!{std}\n");
      }
    }
  }       
 
int search_nodelist(_address_arch2 *searchfor,NodelistDAT *result)
  {
  char *nodelist_idx,*thisrec;
  int  idxlen,ptr;
  _kernel_swi_regs r;

  /* See if we can find nodelist first! */
  r.r[0]=0;
  if (_kernel_swi(0xc91c0,&r,&r)!=NULL)
    {
    mprintf(1,"{bfg r}Can't find nodelist index!{std}\n");
    return(-1);
    }          

  thisrec=nodelist_idx=(char*)r.r[0];
  idxlen=r.r[1];
              
  /* Search the list */
  for(ptr=0;ptr<idxlen;ptr++,thisrec+=8)
    {                               
    if (get_rint(thisrec)==searchfor->zone)
      {
      if (get_rint(thisrec+2)==searchfor->net)
        {
        if (get_rint(thisrec+4)==searchfor->node)
          {
          /* Got the record! */ 
          if ((bbs_file[9]=fopen("<ArcBBS$Nodelist>.QNL_DATbbs","rb"))==NULL)
            {
            mprintf(1,"\n{bfg r}Can't open nodelist data file!{std}\n");
            return(-1);
            }          
          fseek(bbs_file[9],125*ptr,SEEK_SET);
          fread(result,125,1,bbs_file[9]);
          _fclose(&bbs_file[9]);
          return(1);
          }
        }
      }
    }

  return(0);
  }

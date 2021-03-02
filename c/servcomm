/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Server interface commands                   <]
Current version   [> 00.53                                       <]
Version date      [> 12-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "mail.h"
#include "userlog.h"
#include "servmess.h"
#include "servcomm.h"
#include "modcomm.h"
#include "portmisc.h"
#include "mprintf.h"
#include "crc.h"
#include "config.h"

int savednr;
extern char *_message_buffer,*message_buffer;

#ifdef SERVER
extern void event_process(void);
#define window_poll() event_process()
#else
extern user_list *online_users;
extern mail_block *lastmsg;
#endif

void read_area(int arean,mail_area *area)
  {
  mod_readarea(arean,area);
  }

void write_area(int arean,mail_area *area)
  {
  mod_writearea(arean,area);
  }

int get_filenumber()
  {
  return(mod_filenumber());
  }

int read_msgnumber()
  {
  return(mod_getlastlookup());
  }

int read_filenumber()
  {      
  return(mod_readfilenumber());
  }

#ifndef EXT
void tolog(char *log)
  {
  /* Send log entry */
  send_strmessage(servertask,BBS_PUTLOG,portnumber,0,0,log);
  }

void send_message(int task,int action,int word1,int word2,int word3,int word4)
  {
  wimp_msgstr message;

  message.hdr.size=36;
  message.hdr.your_ref=0;
  message.hdr.action=action;
  message.data.words[0]=word1;
  message.data.words[1]=word2;
  message.data.words[2]=word3;
  message.data.words[3]=word4;
  errorlog(wimp_sendmessage(wimp_ESEND,&message,task),"SendMsg1");
  }

void send_strmessage(int task,int action,int word1,int word2,int word3,char *string)
  {
  wimp_msgstr message;

  message.hdr.size=4*((36+strlen(string))/4+1);
  message.hdr.your_ref=0;
  message.hdr.action=action;
  message.data.words[0]=word1;
  message.data.words[1]=word2;
  message.data.words[2]=word3;
  strcpy(message.data.chars+12,string);
  wimp_sendmessage(wimp_ESEND,&message,task);
  }        
#endif

/*** Freeup a chain in mail freespace map ************************* INTERNAL */

int _mail_freeup(int block)
  {
  mail_data md; int forward,lastblock=-1;

  forward=block;

  /* Free forward chain */
  do
    {                         
    if (forward>=0 && forward<=(mod_datalength()<<3))
      {
      mod_readdata(forward,&md);

      /* Does the next block point back to our last block? */
      if (md.block_backward!=lastblock)
        {
        mod_savemaps(); mod_ensuredata(); return(BADERROR);
        }

      md.status=0;
      mod_writedata(forward,&md);
      mod_freedata(forward);

      lastblock=forward; forward=md.block_forward;
      }
    else
      {
      forward=-1;
      }
    }
  while(forward!=-1);

  /* Done all freeing */
  return(OK);
  }

/*** Copy buffer to mailblocks ************************************ INTERNAL */

int _mail_buffertodata(char *buffer,int buflen,int firstblock)
  {
  if (buffer==message_buffer)
    {
    _message_buffer[0]=0xff;
    _message_buffer[1]=0xff;
    _message_buffer[2]=0xff;
    _message_buffer[3]=0xff;

    _message_buffer[4]=0xff;
    _message_buffer[5]=0xff;
    _message_buffer[6]=0xff;
    _message_buffer[7]=0xff;

    _message_buffer[8]=3;

    mod_writedatablock(firstblock<<8,buflen+9,_message_buffer);
    }
  else
    {
    static int header[3];

    /* Write header */
    header[0]=header[1]=-1;
    header[2]=3;
    mod_writedatablock(firstblock<<8,9,header);
   
    /* Write text */
    mod_writedatablock((firstblock<<8)+9,buflen,buffer);
    }

  mod_ensuredata();
  mod_savemaps();
  return(OK);
  }

/*** Copy buffer to mailblocks IMPORT ONLY ************************************ INTERNAL */

int _mail_buffertodatai(char *buffer,int buflen,int firstblock)
  {
  if (buffer==message_buffer)
    {
    _message_buffer[0]=0xff;
    _message_buffer[1]=0xff;
    _message_buffer[2]=0xff;
    _message_buffer[3]=0xff;

    _message_buffer[4]=0xff;
    _message_buffer[5]=0xff;
    _message_buffer[6]=0xff;
    _message_buffer[7]=0xff;

    _message_buffer[8]=3;

    mod_writedatablock(firstblock<<8,buflen+9,_message_buffer);
    }
  else
    {
    static int header[3];

    /* Write header */
    header[0]=header[1]=-1;
    header[2]=3;
    mod_writedatablock(firstblock<<8,9,header);
   
    /* Write text */
    mod_writedatablock((firstblock<<8)+9,buflen,buffer);
    }

  return(OK);
  }

/*** Put message header ******************************************* EXTERNAL */

int put_messageh(mail_block *header)
  {
  int a=mod_writelookup(header->message_number,header);
  mod_ensurelookup(); return(a);
  }

/*** Put message header ******************************************* EXTERNAL */

int put_messagehi(mail_block *header)
  {
  return(mod_writelookup(header->message_number,header));
  }

/*** Write new mail message, chain to header specified ************ EXTERNAL */

int put_message(mail_block *msg_header,char *msg_buffer)
  {
  int firstblock,error,a; mail_area area; mail_block temp;
   
  if (msg_header->message_area==0)
    {
    /* Private mail, msg number can be anywhere */
    msg_header->message_number=savednr=mod_locklookup();
    }
  else
    {
    /* Public mail/file, msg number must be greater than any other */
    msg_header->message_number=savednr=mod_lastlookup();
    }

  if (msg_header->message_length!=0)
    {
    /* Lock a block for our 1st datablock and point to it */
    if ((firstblock=mod_lockdata(msg_header->message_length))==NOROOM)
      {
      /* Free up now redundant header block */
      mod_freelookup(savednr);     
      mod_savemaps();
      return(NOROOM);
      }
    }
  else
    {
    firstblock=-1;
    }

  msg_header->status=1;
  msg_header->block_forward=firstblock;

  /* Message area? */
  if (msg_header->message_area!=0)
    {
    /* Yes */
    mod_readarea(msg_header->message_area,&area);

    if (area.count==0 || area.first_message==-1 || area.last_message==-1)
      {
      area.count=0;
      area.first_message=area.last_message=-1;
      }

    msg_header->message_backward=area.last_message;
    msg_header->message_forward=-1;
    }
  else
    {
    user_block dest;

    /* No, mail area */
    dest.usernumber=msg_header->to_id;
    get_user(&dest);        
                                       
    if (dest.mailcount==0 || dest.mailpointer_start==-1 ||
                             dest.mailpointer_end==-1)
      {
      dest.mailcount=0;
      dest.mailpointer_start=dest.mailpointer_end=-1;
      }

    if (dest.mailpointer_start==-1)
      {
      dest.mailpointer_start=dest.mailpointer_end=savednr;
      msg_header->message_forward=msg_header->message_backward=-1;
      }                              
    else
      {
      if (msg_header->flags&MSG_EXPRESS)
        {
        msg_header->message_forward=dest.mailpointer_start; 
        msg_header->message_backward=-1;
        dest.mailpointer_start=savednr;
        }
      else
        {
        msg_header->message_forward=-1;
        msg_header->message_backward=dest.mailpointer_end;
        dest.mailpointer_end=savednr;
        }
      }

    dest.mailcount++;
    put_user(&dest);
    }

  if (msg_header->message_backward!=-1)
    {
    /* Read previous message & chain its forward to us */
    if ((error=mod_readlookup(msg_header->message_backward,&temp))!=0) return(error);
    temp.message_forward=savednr;
    if ((error=mod_writelookup(msg_header->message_backward,&temp))!=0) return(error);
    }

  if (msg_header->message_forward!=-1)
    {
    /* Read next message & chain its backward to us */
    if ((error=mod_readlookup(msg_header->message_forward,&temp))!=0) return(error);
    temp.message_backward=savednr;
    if ((error=mod_writelookup(msg_header->message_forward,&temp))!=0) return(error);
    }
                                     
  if (msg_header->reply_from!=-1 && msg_header->message_area!=0)
    {
    int a;

    /* Increment replies count & put in reply chain */
    if ((error=mod_readlookup(msg_header->reply_from,&temp))!=0) return(error);

    temp.noof_replies++;
    for (a=7;a>0;a--) temp.replies[a]=temp.replies[a-1];
    temp.replies[0]=msg_header->message_number;

    if ((error=mod_writelookup(msg_header->reply_from,&temp))!=0) return(error);
    }

  /* Write header to file */
  if (mod_writelookup(msg_header->message_number,msg_header)!=OK)
    {
    /* Free up now redundant data block */
    mod_freedata(firstblock);   
    mod_savemaps();
    return(ERROR);
    }

  if (msg_header->message_length)
    {
    /* Write text to file */
    if ((error=_mail_buffertodata(msg_buffer,msg_header->message_length,firstblock))!=0)
      return(error);
    }

  /* Chain last message in chain to us */
  if (msg_header->message_area!=0)
    {
    area.count++;
    area.user_id=msg_header->from_id;
    area.newest=msg_header->date_sent;

    if (area.first_message==-1)
      {
      /* 1st message in area */
      area.first_message=msg_header->message_number;
      }
    area.last_message=msg_header->message_number;

    mod_writearea(msg_header->message_area,&area);
    }
      
#ifndef EXT                 
  /* Page user if new private mail arrives */
  if (msg_header->message_area==0)
    {
    /* Check to see if user is online */
    for(a=0;a<16;a++)
      {
      if (online_users[a].username[0]!=0)
        {
        if (online_users[a].usernumber==msg_header->to_id)
          {
          send_strmessage(servertask,BBS_MESSAGE,portnumber,a,0,
                          "\001\007You have new mail!\n");
          window_poll();
          break;
          }
        }
      }                      
    /* To sysop? */
    if (msg_header->to_id==1)
      {
      send_strmessage(servertask,BBS_SYSOPMAIL,0,0,0,msg_header->from);
      window_poll();
      }
    }
#endif           
          
  mod_ensurelookup();
  return(OK);
  }

/*** Write new mail message, chain to header specified IMPORT ONLY ************ EXTERNAL */

int put_messagei(mail_block *msg_header,char *msg_buffer)
  {
  int firstblock,error,a; mail_area area; mail_block temp;
   
  if (msg_header->message_area==0)
    {
    /* Private mail, msg number can be anywhere */
    msg_header->message_number=savednr=mod_locklookup();
    }
  else
    {
    /* Public mail/file, msg number must be greater than any other */
    msg_header->message_number=savednr=mod_lastlookup();
    }

  if (msg_header->message_length!=0)
    {
    /* Lock a block for our 1st datablock and point to it */
    if ((firstblock=mod_lockdata(msg_header->message_length))==NOROOM)
      {
      /* Free up now redundant header block */
      mod_freelookup(savednr);     
      mod_savemaps();
      return(NOROOM);
      }
    }
  else
    {
    firstblock=-1;
    }

  msg_header->status=1;
  msg_header->block_forward=firstblock;

  /* Message area? */
  if (msg_header->message_area!=0)
    {
    /* Yes */
    mod_readarea(msg_header->message_area,&area);

    if (area.count==0 || area.first_message==-1 || area.last_message==-1)
      {
      area.count=0;
      area.first_message=area.last_message=-1;
      }

    msg_header->message_backward=area.last_message;
    msg_header->message_forward=-1;
    }
  else
    {
    user_block dest;

    /* No, mail area */
    dest.usernumber=msg_header->to_id;
    get_user(&dest);        
                                       
    if (dest.mailcount==0 || dest.mailpointer_start==-1 ||
                             dest.mailpointer_end==-1)
      {
      dest.mailcount=0;
      dest.mailpointer_start=dest.mailpointer_end=-1;
      }

    if (dest.mailpointer_start==-1)
      {
      dest.mailpointer_start=dest.mailpointer_end=savednr;
      msg_header->message_forward=msg_header->message_backward=-1;
      }                              
    else
      {
      if (msg_header->flags&MSG_EXPRESS)
        {
        msg_header->message_forward=dest.mailpointer_start; 
        msg_header->message_backward=-1;
        dest.mailpointer_start=savednr;
        }
      else
        {
        msg_header->message_forward=-1;
        msg_header->message_backward=dest.mailpointer_end;
        dest.mailpointer_end=savednr;
        }
      }

    dest.mailcount++;
    put_user(&dest);
    }

  if (msg_header->message_backward!=-1)
    {
    /* Read previous message & chain its forward to us */
    if ((error=mod_readlookup(msg_header->message_backward,&temp))!=0) return(error);
    temp.message_forward=savednr;
    if ((error=mod_writelookup(msg_header->message_backward,&temp))!=0) return(error);
    }

  if (msg_header->message_forward!=-1)
    {
    /* Read next message & chain its backward to us */
    if ((error=mod_readlookup(msg_header->message_forward,&temp))!=0) return(error);
    temp.message_backward=savednr;
    if ((error=mod_writelookup(msg_header->message_forward,&temp))!=0) return(error);
    }
                            
  /* No replies */
  msg_header->reply_from=-1;
  msg_header->noof_replies=0;

  /* Write header to file */
  if (mod_writelookup(msg_header->message_number,msg_header)!=OK)
    {
    /* Free up now redundant data block */
    mod_freedata(firstblock);   
    mod_savemaps();
    return(ERROR);
    }

  if (msg_header->message_length)
    {
    /* Write text to file */
    if ((error=_mail_buffertodatai(msg_buffer,msg_header->message_length,firstblock))!=0)
      return(error);
    }


  /* Chain last message in chain to us */
  if (msg_header->message_area!=0)
    {
    area.count++;
    area.user_id=msg_header->from_id;
    area.newest=msg_header->date_sent;

    if (area.first_message==-1)
      {
      /* 1st message in area */
      area.first_message=msg_header->message_number;
      }
    area.last_message=msg_header->message_number;
    mod_writearea(msg_header->message_area,&area);
    }

  return(OK);
  }

/*** Delete mail message ****************************************** EXTERNAL */

int delete_message(int msg_number)
  {
  mail_block temp_block,temp_block2; mail_area area; user_block dest;
  int backward,forward,save=1,ret;

  if ((msg_number&(1<<31))!=0)
    {
    msg_number&=~(1<<31);
    save=0;
    }

  /* Load up header & check it is OK */
  mod_readlookup(msg_number,&temp_block);
  if (temp_block.status!=1) return(BADERROR);

  backward=temp_block.message_backward; forward=temp_block.message_forward;

  /* Set lookup entry to 'deleted' */
  temp_block.status=0;
  mod_writelookup(msg_number,&temp_block);
  mod_freelookup(msg_number);
  if (temp_block.message_area!=0) mod_readarea(temp_block.message_area,&area);

  /* Get user if necessary */
  if (temp_block.message_area==0)
    {
    dest.usernumber=temp_block.to_id;
    get_user(&dest);        
    }

  /* Re-chain other messages */
  /* Point backward block to forward block */
  if (backward>=0)
    {
    mod_readlookup(backward,&temp_block2);
    if (temp_block2.status!=1) return(BADERROR);
    if (temp_block2.message_forward!=msg_number) return(BADERROR);
    temp_block2.message_forward=forward;
    mod_writelookup(backward,&temp_block2);
    }
  else
    {
    /* Make forward message 1st message in chain */
    if (temp_block.message_area!=0)
      {
      area.first_message=forward;
      }
    else
      {
      dest.mailpointer_start=forward;
      }
    }

  /* Point forward block to backward block */
  if (forward>=0)
    {
    mod_readlookup(forward,&temp_block2);
    if (temp_block2.status!=1) return(BADERROR);
    if (temp_block2.message_backward!=msg_number) return(BADERROR);
    temp_block2.message_backward=backward;
    mod_writelookup(forward,&temp_block2);
    }
  else
    {
    /* Make backward message last message in chain */
    if (temp_block.message_area!=0)
      {
      area.last_message=backward;
      }
    else
      {
      dest.mailpointer_end=backward;
      }
    }

  /* Adjust area stuff */         
         
  if (temp_block.message_area!=0)
    {
    area.count--;
    if (area.count==0)
      {
      area.newest=0;
      area.user_id=-1;
      area.user[0]=0;
      }
    mod_writearea(temp_block.message_area,&area);
    }
  else
    {
    dest.mailcount--;
    put_user(&dest);
    }
               
  /* Now free up the chain */
  if (temp_block.message_length)
    {
    ret=_mail_freeup(temp_block.block_forward);
    if (save) mod_ensuredata();
    }
  else
    {
    ret=OK;
    }

  if (save)
    {
    mod_ensurelookup();
    mod_savemaps();
    }

  return(ret);
  }

/*** Get message header ******************************************* EXTERNAL */

int get_messageh(mail_block *header)
  {
#ifndef SERVER
  lastmsg=header;
#endif
  return(mod_readlookup(header->message_number,header));
  }

/*** Get max area number ****************************************** EXTERNAL */

int areamax()
  {
  return(mod_getlastarea());
  }

/*** Read mail message ******************************************** EXTERNAL */

int get_message(mail_block *msg_header,char *msg_buff)
  {
  int firstblock,lastblock=-1,error,count,a,msg_number=msg_header->message_number;
  mail_data data;

  /* Read in header block */
  if ((error=mod_readlookup(msg_number,msg_header))!=OK) return(error);
  if (msg_header->status!=1) return(NODATA);
#ifndef SERVER
  lastmsg=msg_header;
#endif

  /* Update read count */
  msg_header->noof_reads++; mod_writelookup(msg_number,msg_header);

  firstblock=msg_header->block_forward;

  if (msg_header->message_length>0 && firstblock>=0 && firstblock<=mod_datalength())
    {
    /* Read data into buffer */
    count=0;
    do
      {
      /* Read data block */
      mod_readdata(firstblock,&data);

      if (data.status==2)
        {
        if (data.block_backward!=lastblock)
          {
          msg_header->message_length=count; firstblock=-1;
          }
        else
          {
          /* Reset pointer */
          lastblock=firstblock;
          firstblock=data.block_forward;

          /* Read block into message */
          for(a=0;a<247;a++) msg_buff[count++]=data.data[a];
          }
        }
      else
        {
        if (data.status==3)
          {
          if (msg_buff==message_buffer)
            {
            mod_readdatablock(firstblock<<8,msg_header->message_length+9,_message_buffer);
            }
          else
            {
            mod_readdatablock((firstblock<<8)+9,msg_header->message_length,msg_buff);
            }
          break;
          }
        msg_header->message_length=count; firstblock=-1;
        }
      }
    while(firstblock>=0 && firstblock<=(mod_datalength()<<3));
    }

  msg_buff[msg_header->message_length]=0;
 
  /* Read whole message, return */
  return(OK);
  }

void dumpfiles()
  {
  mod_saveall();
  }

/*** Find/lock free userlog record ******************************** EXTERNAL */

int find_free()
  {
  int usernumber=0; user_block st;
  clock_t lastpoll=clock();
       
  /* Find an unused userlog record */
  do
    {
    if ((clock()-lastpoll)>MAXIMUM_POLL)
      {
      window_poll(); lastpoll=clock();
      }
    usernumber++; st.status=0;
    mod_readuserdata(usernumber,&st);
    }
  while(st.status);
                        
  /* Set it as 'in use' to stop anyone else grabbing it */
  st.status=1;      
  mod_writeuserdata(usernumber,&st);
  mod_ensureuser();                  

  return(usernumber);
  }

/*** Write userlog record ***************************************** EXTERNAL */

int put_user(user_block *data)
  {
  /* Check for wild pointers */
  if (data->usernumber>(mod_extentuser()+5)) return(ERROR);

  /* Write data out */
  if (mod_writeuserdata(data->usernumber,data)!=OK) return(NOROOM);
  mod_ensureuser();                 

  return(OK);
  }

/*** Read userlog record ****************************************** EXTERNAL */

int get_user(user_block *data)
  {
  int un=data->usernumber;

  /* Check for wild pointers */
  if (un>(mod_extentuser()+5)) return(ERROR);

  /* Read data */
  if (mod_readuserdata(data->usernumber,data)!=OK) return(NODATA);

  /* Set usernumber properly (just in case) */
  data->usernumber=un;

  return(OK);
  }

/*** Find userlog record given name ******************************* EXTERNAL */

int hash_find(char *un)
  {
  int crc,maxusers=mod_maxuser(); user_hash hashblock;
  char username[31],*un2;
                    
  strcpy(un2=username,un);

  /* Convert this entry to uppercase */
  while(*un2) *un2++=toupper(*un2);
               
  crc=calcrc(username,strlen(username)); 

  /* Check entries for user */
  do
    {
    /* Loop to size of table */
    crc=crc%(maxusers*4);

    /* Lookup hash table record */
    mod_readuserlookup(crc,&hashblock);

    /* If it's deleted OR not the one we're looking for, continue looking */
    crc+=1;
    }
  while(hashblock.status==1 || 
        (hashblock.status==2 && strcmp(username,hashblock.username)!=0));

  /* Was there no entry? */
  if (hashblock.status==0) return(NODATA);

  /* We've found the usernumber */
  return(hashblock.recordpointer);
  }

/*** Kill userlog hash table entry ******************************** EXTERNAL */

void hash_kill(char *un)
  {
  int crc,maxusers=mod_maxuser(); user_hash hashblock;
  char username[31],*un2;
                        
  strcpy(un2=username,un);

  /* Convert this entry to uppercase */
  while(*un2) *un2++=toupper(*un2);

  crc=calcrc(username,strlen(username));

  /* Check entries for user */
  do
    {
    /* Loop to size of table */
    crc=crc%(maxusers*4);

    /* Lookup hash table record */
    mod_readuserlookup(crc,&hashblock);

    /* If it's deleted OR not the one we're looking for, continue looking */
    crc+=1;
    }
  while(hashblock.status==1 || 
        (hashblock.status==2 && strcmp(username,hashblock.username)));

  /* Was there no entry? */
  if (hashblock.status==0) return;

  /* Delete the entry */
  hashblock.status=1; *hashblock.username=0;
  mod_writeuserlookup(crc-1,&hashblock);  
  mod_ensureuser();
  }

/*** Write userlog hash record given name ************************* EXTERNAL */

int hash_write(char *un,int pointer)
  {
  int crc,maxusers=mod_maxuser(); user_hash hashblock;
  char username[31],*un2;

  strcpy(un2=username,un);

  /* Convert this entry to uppercase */
  while(*un2) *un2++=toupper(*un2);

  crc=calcrc(username,strlen(username));

  if (hash_find(username)!=NODATA)
    {
    /* Find existing record */
    do
      {
      /* Loop to size of table */
      crc=crc%(maxusers*4);
 
      /* Lookup hash table record */
      mod_readuserlookup(crc,&hashblock);

      /* If it's deleted OR not the one we're looking for, continue looking */
      crc+=1;
      }
    while(hashblock.status==2 && strcmp(username,hashblock.username)!=0);
    }
  else
    {
    /* Find blank space */
    do
      {
      /* Loop to size of table */
      crc=crc%(maxusers*4);

      /* Lookup hash table record */
      mod_readuserlookup(crc,&hashblock); 

      /* If it's not free, continue looking */
      crc+=1;
      }
    while(hashblock.status==2);
    }

  /* Setup hashblock & write the new hash entry */
  hashblock.status=2;
  strcpy(hashblock.username,username);
  hashblock.recordpointer=pointer;
  mod_writeuserlookup(crc-1,&hashblock); 
  mod_ensureuser();
  return(crc-1);
  }

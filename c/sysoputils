/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Sysop utilities module                      <]
Current version   [> 00.23                                       <]
Version date      [> 10-December-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
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

#include "miscbbs.h"
#include "bbs.h"

extern void trim_area(int,int,int);


void generate_filelist(char *filename)
  {
  int filearea,current,max=areamax(),len,total_here,total=0;
  time_t now=time(NULL);
  mail_block fileheader; mail_area area; char shortd[44];
  clock_t lastpoll=clock();

  if ((bbs_file[7]=fopen(filename,"w"))==NULL)
    {
    mprintf(1,"{bfg r}Can't write %s!{std}\n",filename);
    return;
    }

  fprintf(bbs_file[7],"Filelist generated at %s\n\n",cctime(&now));

  for(filearea=0;filearea<max;filearea++)
    {
    read_area(filearea,&area);
    if ((area.areaflags&FLAG_FILEAREA)!=0 &&
        (area.areaflags&FLAG_READABLE)!=0)
      {          
      total_here=0;
      fprintf(bbs_file[7],"\nArea: %s (%d)\n",area.name,filearea);
      fprintf(bbs_file[7],"File # Name                 Length Short description\n");
      fprintf(bbs_file[7],"------ -------------------- ------ ----------------------------------------\n");
      current=area.last_message;
      if (current==-1)
        {
        fprintf(bbs_file[7],"No files\n");
        }   
      else
        {
        do
          {
          fileheader.message_number=current;
          get_messageh(&fileheader);

          if ((clock()-lastpoll)>MAXIMUM_POLL)
            {
            window_poll(); lastpoll=clock();
            }

          if ((fileheader.flags&FILE_SYSOP)==0)
            {
            len=strlen(fileheader.subject);
            strncpy(shortd,fileheader.subject,40);
            shortd[40]=0;
        
            fprintf(bbs_file[7],"%06d %-20s %6d %s\n",
                                fileheader.file_location,
                                fileheader.to,fileheader.file_length,
                                shortd);
            total_here+=fileheader.file_length;
            total+=fileheader.file_length;

            if (len>40)
              {
              fprintf(bbs_file[7],"                                   %s\n",
                      fileheader.subject+40);
              }
            }
          current=fileheader.message_backward;  
          }
        while(current!=-1);
        }
      fprintf(bbs_file[7],"\nSize of area %dk\n\n\n",total_here/1024);
      }
    }
  fprintf(bbs_file[7],"\n\nTotal size of filebase %dk\n",total/1024);
  fclose(bbs_file[7]); bbs_file[7]=0;
  }

void mend_area(int arean)
  {
  char *lookupmap=mod_lookupmap();
  mail_area area; mail_block msg,msg2;
  int msgn,last=-1,backward=-1,first=-1,msgcount=0,
  maxmsg=read_msgnumber();
  clock_t lastpoll=clock();

  read_area(arean,&area);
  
  mprintf(1,"{bfg c}Testing area {fg y}%s (%d)...",area.name,arean);
  msgn=area.first_message;
  if ((msgn<0 && msgn!=-1) || msgn>maxmsg) goto fixit;
  while(msgn>=0)
    {                  
    msg.message_number=msgn;
    get_messageh(&msg);                           
    if (msg.message_backward!=last) goto fixit;
    last=msgn; msgn=msg.message_forward;
    if (msgn<0 && msgn!=-1) goto fixit;
    if (msgn>maxmsg) goto fixit;
    if ((clock()-lastpoll)>MAXIMUM_POLL)
      {
      window_poll();
      lastpoll=clock();
      }
    }
  if (last!=area.last_message) goto fixit;
  mprintf(1,"{bfg c}Area OK\n");
  return;

  fixit:
  mprintf(1,"{bfg c}Detected error, fixing...\n");

  /* Scan all message headers */
  for(msgn=0;msgn<=maxmsg;msgn++)
    {
    /* Get a message */
    msg.message_number=msgn;
    get_messageh(&msg);

    if ((clock()-lastpoll)>=MAXIMUM_POLL)
      {
      window_poll();
      lastpoll=clock();
      }

    if (msg.status==1 && msg.message_area==arean)
      {
      if ((lookupmap[msgn/8]&(1<<(msgn%8)))==0)
        {
        mprintf(1,"Fixed badly allocated message header (msg #%d)\n",msgn);
        lookupmap[msgn/8]|=(1<<(msgn%8));
        }

      /* In our area */
      if (first==-1) first=msgn;
      msg.message_backward=backward;
      put_messageh(&msg);

      if (backward!=-1)
        {
        /* Fill in last forward pointer */
        msg2.message_number=backward;
        get_messageh(&msg2);
        msg2.message_forward=msgn;
        put_messageh(&msg2);
        }

      backward=msgn;
      msgcount++;
      }
    }

  area.first_message=first;
  area.last_message=backward;
  area.count=msgcount;
  area.newest=0; area.user[0]=0; area.user_id=0;
  write_area(arean,&area);

  /* Fill in last forward pointer */
  if (backward!=-1)
    {
    msg2.message_number=backward;
    get_messageh(&msg2);
    msg2.message_forward=-1;
    put_messageh(&msg2);
    }
             
  mod_saveall();

  mprintf(1,"\nRelinked %d messages in area %s.\n",msgcount,
                                                         area.name);
  }

void do_utils()
  {
  int ch,confirm;

  do
    {
    mprintf(1,"{bfgbg cb}Sysop utilities{eol std}\n\n");
    mprintf(1,"{bfg c}[{fg y}R{fg c}]{fg w}ead a user's private mail\n");
    mprintf(1,"{fg c}[{fg y}M{fg c}]{fg w}end a message/filebase's links\n");
    mprintf(1,"{fg c}[{fg y}T{fg c}]{fg w}otal mend (all bases)\n");
    mprintf(1,"{fg c}[{fg y}G{fg c}]{fg w}enerate new userlog Lookup file\n");
    mprintf(1,"{fg c}[{fg y}A{fg c}]{fg w}lter global conference access\n");
    mprintf(1,"{fg c}[{fg y}S{fg c}]{fg w}how conference access\n");
    mprintf(1,"{fg c}[{fg y}B{fg c}]{fg w}ulk user delete\n");
    mprintf(1,"{fg c}[{fg y}C{fg c}]{fg w}hange a message/filearea's areanumber\n");
    mprintf(1,"{fg c}[{fg y}!{fg c}]{fg w} Fix text links/recreate freespace maps\n");
    mprintf(1,"{fg c}[{fg y}P{fg c}]{fg w} Private mail re-linker\n");
    mprintf(1,"{fg c}[{fg y}1{fg c}]{fg w} Private mail re-linker (for specific user)\n");
    mprintf(1,"{fg c}[{fg y}*{fg c}]{fg w} Generate text-filelist\n");
    mprintf(1,"{fg c}[{fg y}${fg c}]{fg w} Trim messagebase\n");
    mprintf(1,"{fg c}[{fg y}2{fg c}]{fg w} Setup echomail rescan\n");
    mprintf(1,"{fg c}[{fg y}3{fg c}]{fg w} Export message/filebase to a file\n");
    mprintf(1,"{fg c}[{fg y}4{fg c}]{fg w} Import message/filebase from a file\n");
    mprintf(1,"{fg c}[{fg y}5{fg c}]{fg w} Compress mail.data\n");
    mprintf(1,"{fg c}[{fg y}Q{fg c}]{fg w}uit utils\n\n{fg c}Select: {fg w}");
    do
      {
      ch=toupper(port_get());
      eatthis:
      if (ch>31) port_txw(ch);
      if ((confirm=toupper(port_get()))!=13) { port_txw(8); port_txw(32); port_txw(8); ch=confirm; goto eatthis; }
      }
    while(confirm!=13);
    port_txw(8);

    switch(ch)
      {
      case 'R': /* Read private mail */
        {
        char userbuffer[31],*ptr1,*ptr2;
        int readuser;
        user_block tempuser;

        port_txstring("Read private mail\n\n{fg c}Username/number: ",1);
        port_readline(ptr1=userbuffer,30,0);

        /* Strip leading spaces */
        while(*ptr1==' ') ptr1++;
        if (!*ptr1)
          {
          port_crlf();
          break;
          }

        /* Convert string to uppercase */
        ptr2=ptr1;
        while(*ptr2)
          {
          *ptr2=toupper(*ptr2);
          ptr2++;
          }

        /* If numeric first digit, get as a usernumber */
        if (*ptr1>='0' && *ptr1<='9') 
          {
          readuser=atoi(ptr1);
          }
        else
          {
          /* Text is username, do hashing, etc */
          readuser=hash_find(ptr1);

          /* Non-existant? */
          if (readuser==-1)
            {
            mprintf(1,"\nCan't find user #%d\n",readuser); break;
            }
          }

        tempuser.usernumber=readuser;
        get_user(&tempuser);
        if (tempuser.status!=1)
          {
          mprintf(1,"\nUser is deleted\n");
          break;
          }

        mprintf(1,"\n\nReading private mail of %s\n",tempuser.username);

        do_checkmail(readuser,1); /* Check mail with sysop status */

        break;
        }
      case 'M': /* Mend messagebase */
        {
        char numb[8]; int arean;
       
        port_txstring("Mend links\n\n{fg c}Area number: ",1);
        port_readline(numb,7,0); port_crlf();
        if (*numb==0) break;

        if ((arean=atoi(numb))==0)
          {
          port_txstring("\nCannot mend private mail area\n",1);
          break;
          }

        mend_area(arean);

        break;
        }
      case 'T': /* Total mend! */
        {
        mail_area area;
        int arean;
                      
        port_txstring("Total mend\n",1);
                                        
        if (port_yesno("Sure? ")==0) break;

        for(arean=1;arean<mod_getlastarea();arean++)
          {      
          read_area(arean,&area);
          
          if (area.name[0]!=0) mend_area(arean);
          }          
        
        mod_saveall();

        break;
        }
      case 'G': /* Generate user lookup */
        {                                                            
        user_block temp; char zeros[64]; int a,count=0;

        for(a=0;a<64;a++) zeros[a]=0;

        port_txstring("Generate user lookup\n",1);

        if (port_yesno("Sure? ")==0) break;
        port_txstring("\nBlanking current file...",1);
             
        for(a=0;a<(mod_maxuser()*4);a++) mod_writeuserlookup(a,zeros);

        port_txstring("Writing new lookup entries...",1);

        for(a=0;a<mod_extentuser();a++)
          {
          mod_readuserdata(a,&temp);
          if (temp.status==1)
            {
            hash_write(temp.username,a); count++;
            }  
          }

        port_txstring("done!\n",1);
        mprintf(1,"done!\n%d user links generated.\n",count);
        break;
        }
      case 'A': /* Alter conference access */
        {                  
        char numb[5]; int a,conf,add=0; user_block temp;
        clock_t lastpoll=clock();

        port_txstring("Alter conferences\n\n{fg c}Area number (prefix with + to add, - to remove): ",1);
        port_readline(numb,4,0); port_crlf();
        if (numb[0]==0 || (numb[0]!='+' && numb[0]!='-')) break;
        conf=atoi(numb+1); if (numb[0]=='+') add++;                              

        port_txstring("Please wait, altering all users (except SYSOPs)...",1);

        for(a=0;a<mod_extentuser();a++)
          {                  
          /* Ensure multitasking */
          if ((clock()-lastpoll)>MAXIMUM_POLL)
            {
            window_poll(); lastpoll=clock();
            }

          /* Alter user */
          mod_readuserdata(a,&temp);
          if (temp.status==1 && (temp.f_user&USER_SYSOP)==0)
            {
            if (add)
              {
              conf_join(temp.conferences,conf);
              }
            else
              {
              conf_resign(temp.conferences,conf);
              }
            mod_writeuserdata(a,&temp);
            }
          }

        port_txstring("done!\n",1);   

        break;
        }
      case 'S': /* Show conference access */
        {                  
        char numb[5]; int a,conf; user_block temp;

        port_txstring("Show conference access\n\n{fg c}Area number: ",1);
        port_readline(numb,4,NUMONLY); port_crlf();
        if (numb[0]==0) break;
        conf=atoi(numb);

        for(a=0;a<mod_extentuser();a++)
          {
          mod_readuserdata(a,&temp);
          if (temp.status==1)
            {                
            if (conf_member(temp.conferences,conf))
              {
              mprintf(1,"{fg w}%4d {fg c}%s\n",a,temp.username);
              }
            }
          }

        break;
        }   
      case 'B': /* Bulk user delete */
        {    
        char numb[5]; int a,days; user_block temp; time_t now=time(NULL);

        mprintf(1,"Bulk user delete\n");
        if (port_yesno("Sure? ")==0) break;
                
        port_txstring("{bfg r}Note: This does not delete users who have the 'Registered' or the 'Sysop'\nflags set!\n",1);
        port_txstring("{fg c}Number of days unused before delete: ",1);
        port_readline(numb,4,NUMONLY); port_crlf();
        if (numb[0]==0) break;
        days=(atoi(numb)*(24*60*60)); /* Seconds */

        for(a=1;a<mod_extentuser();a++)
          {
          mod_readuserdata(a,&temp);
          if (temp.status==1)
            {  
            /* Has user expired? */
            if (temp.t_lastlogon<(now-days) &&
               (temp.f_user&USER_SYSOP)==0 &&
               (temp.f_user&USER_REGISTERED)==0)
              {
              int ch;

              mprintf(1,"User: %s (#%d)\nDelete [y/n/a] : ",
                        temp.username,temp.usernumber);
 
              ch=toupper(port_get());
              if (ch=='A') { mprintf(1,"Aborted\n"); break; }
              if (ch=='Y')
                {
                temp.status=0;
                mod_writeuserdata(a,&temp);
                hash_kill(temp.username); 
                if (temp.mailcount) do_checkmail(a,1); /* Check mail */
                mprintf(1,"Deleted\n");
                }
              else
                {
                mprintf(1,"Not deleted\n");
                }         
              }
            }
          }
        mprintf(1,"All users processed\n");

        break;
        }
      case 'C': /* Area number change */
        {
        char numb[5]; int source,destination,a,msgcount=0;
        mail_area source_a,destination_a;
        mail_block header; clock_t lastpoll=clock();

        mprintf(1,"Change areanumber (source/destination areas MUST be idle!)\n\n{bfg c}Enter source area#      :");
        port_readline(numb,4,NUMONLY); port_crlf();
        if (numb[0]==0) break;
        source=atoi(numb);     

        mprintf(1,"{bfg c}Enter destination area# :");
        port_readline(numb,4,NUMONLY); port_crlf();
        if (numb[0]==0) break;
        destination=atoi(numb);
 
        read_area(source,&source_a);
        read_area(destination,&destination_a);

        if (source_a.count==0)
          {
          mprintf(1,"{bfg r}Source area is empty.{std}\n");
          break;
          }

        if (destination_a.count!=0)
          {
          mprintf(1,"{bfg r}Destination area is not empty.{std}\n");
          break;
          }

        mprintf(1,"{bfg g}Processing...{fg w}\n");
                             
        for(a=0;a<mod_getlastlookup();a++)
          {                     
          if ((clock()-lastpoll)>=MAXIMUM_POLL)
            {
            window_poll();
            lastpoll=clock();
            }

          mod_readlookup(a,&header);
          if (header.status==1 && header.message_area==source)
            {
            /* Message is ok for transfer */
            header.message_area=destination;
            mod_writelookup(a,&header);
            mprintf(1,"%04d messages transferred\015",++msgcount);
            }
          }
          
        mprintf(1,"\n{bfg g}Moving pointers...");

        destination_a.first_message=source_a.first_message;
        destination_a.last_message =source_a.last_message;
        destination_a.count        =source_a.count;
        write_area(destination,&destination_a);

        if (source_a.count!=msgcount)
          {
          mprintf(1,"(msg count mismatch, use 'M'end on new msgbase)...");
          }
        
        source_a.first_message=source_a.last_message=-1;
        source_a.count=0;
        write_area(source,&source_a);

        mprintf(1,"Done!{std}\n");

        break;
        }
      case '!':
        {
        mail_block header; mail_data data;
        int a,lookuplen=mod_lookuplength()<<3,datalen=mod_datalength(),
            datalen2=mod_dataext();
        char *lookupmap=mod_lookupmap(),*datamap=mod_datamap(),text[128];
        clock_t lastpoll=clock();

        strcpy(text,".");                                            

        mprintf(1,"Check & fix freespace maps\n");
        if (port_yesno("Sure? ")==0) break;
        mprintf(1,"\nClearing LookupMap and DataMap...");

        for(a=0;a<(lookuplen>>3);a++) lookupmap[a]=0;
        for(a=0;a<datalen;a++) datamap[a]=0;
        mprintf(1,"done\n\nScanning for active headers...\n");
          
        for(a=0;a<mod_getlastlookup();a++)
          {   
          if ((clock()-lastpoll)>=200)
            {
            window_poll();
            lastpoll=clock();
            }

          mod_readlookup(a,&header);         

          if (header.status!=1)
            {
            if (header.status!=0)
              {
              mprintf(1,"* %06d Inactive{eol}\n",a); 
              header.status=0; 
              mod_writelookup(a,&header);
              }
            }
          else
            {
            /* Set freespace */
            lookupmap[a/8]|=(1<<(a%8));
           
            /* Check for correct message number in record */
            if (header.message_number!=a) header.message_number=a;
        
            if (header.message_length==0)
              {
              if (header.block_forward!=-1)
                {
                mprintf(1,"%06d Active - Area %03d - Link exists with no data{eol}\n",
                        a,header.message_area);   
                header.block_forward=-1;
                mod_writelookup(a,&header);
                }
              }
            else
              {
              if (header.block_forward==-1)
                {
                mprintf(1,"%06d Active - Area %03d - No link exists for data{eol}\n",
                        a,header.message_area);   
                header.message_length=0;
                mod_writelookup(a,&header);
                }
              else
                {         
                int currentlink=header.block_forward,lastblock=-1,trunc=0;

                do
                  {
                  if (currentlink<0 || currentlink>datalen2)
                    {
                    mprintf(1,"%06d Active - Area %03d - Bad forward link, text truncated{eol}\n",
                              a,header.message_area);
                    trunc=1;
                    }
                  else
                    {
                    /* Check map */
                    if (datamap[currentlink]!=0)
                      {
                      mprintf(1,"%06d Active - Area %03d - Link block already allocated! Text truncated{eol}\n",
                                a,header.message_area);
                      trunc=1; 
                      }
                    else
                      {
                      /* Get datablock */
                      mod_readdata(currentlink,&data);

                      if (data.block_backward!=lastblock)
                        {
                        mprintf(1,"%06d Active - Area %03d - Bad backward link, text truncated{eol}\n",
                                a,header.message_area);
                        trunc=1;
                        }
                      lastblock=currentlink;
                      currentlink=data.block_forward;
                      }
                    }
                  }
                while(trunc==0 && currentlink!=-1);
                  
                if (trunc==1)
                  {
                  /* Truncate message */
                  header.block_forward=-1;
                  header.message_length=0;
                  mod_writelookup(a,&header);
                  }
                else
                  {
                  int currentlink=header.block_forward;
                                         
                  if (currentlink!=-1)
                    {
                    /* Message text is OK, run through datablocks setting flags */
                    do
                      {
                      /* Get datablock */
                      mod_readdata(currentlink,&data);
                      if (data.status==3)
                        {
                        int noofb=(header.message_length+255+9)>>8;

                        /* Fast msg format */
                        datamap[currentlink++]=3;
                        while(--noofb)
                          {
                          datamap[currentlink++]=4;
                          }
                        break;
                        }
                      else
                        {
                        data.status=2;
                        mod_writedata(currentlink,&data);
                        datamap[currentlink]=2;
                        currentlink=data.block_forward;
                        }
                      }
                    while(currentlink!=-1);
                    }

                  sprintf(text,"%06d Active - Area %03d - Links fine{eol}\015",
                                  a,header.message_area);
                  }
                }
              }
            }
          }

        mprintf(1,"\nClearing any unused data bits...\n");
        for(a=0;a<datalen2;a++)
          {           
          if ((clock()-lastpoll)>=200)
            {                 
            mprintf(1,"%s",text);
            window_poll();
            lastpoll=clock();
            }

          if (datamap[a]==0)
            {
            mod_readdata(a,&data);
            if (data.status!=0)
              {
              memset(&data,0,256);
              mod_writedata(a,&data);
              }
            }
          }

        mprintf(1,"\n! Done - Press any key to continue\n");
        port_rxclear();
        port_get();
        mod_saveall();

        break;
        }
      case 'P': /* Private mail relinker */
        {
        mail_block msg; user_block fixuser;
        clock_t lastpoll=clock();
        int a,b,max=read_msgnumber(),maxuser=mod_maxuser(),msgcount,zerocount;
       
        mprintf(1,"Relink private mail\n");
        if (port_yesno("Sure? ")==0) break;

        for(a=0;a<maxuser;a++)
          {
          fixuser.usernumber=a;
          get_user(&fixuser);                                  
          mprintf(1,"%04d\015",a);

          if ((clock()-lastpoll)>=MAXIMUM_POLL)
            {
            window_poll();
            lastpoll=clock();
            } 
             
          if (fixuser.status!=0)
            {
            /* Start with NO mail */
            fixuser.mailcount=0;
            fixuser.mailpointer_start=fixuser.mailpointer_end=-1;
            put_user(&fixuser);
            }    
          }
           
        for(b=0;b<max;b++)
          {
          if ((clock()-lastpoll)>=MAXIMUM_POLL)
            {
            window_poll();
            lastpoll=clock();
            } 

          msg.message_number=b;
          get_messageh(&msg);          
             
          mprintf(1,"-%06d\015",b);

          /* To the right person? */
          if (msg.message_area==0 && msg.status==1)
            {
            /* Get the whole message */
            get_message(&msg,message_buffer);
                      
            mprintf(1,"=%06d\015",b);

            /* Delete this message */
            msg.status=0;
            mod_writelookup(b,&msg);
            mod_freelookup(b);
            _mail_freeup(msg.block_forward);
            msg.status=1;
  
            if (msg.message_length>0)
              {
              fixuser.usernumber=msg.to_id;
              get_user(&fixuser);
               
              if (user.status!=0)
                {
                /* Re-send it */                 
                msg.message_number=0;
                msg.message_forward=msg.message_backward=-1;
                put_message(&msg,message_buffer);  
                msgcount++;
                }
              }
            else
              {
              zerocount++;
              }
            }
          }
        mprintf(1,"\n%d messages recovered (%d null msgs deleted)\n",msgcount,zerocount);   

        break;
        }
      case '*': /* Generate filelist */
        {  
        mprintf(1,"Filelist generate\n");
        if (port_yesno("Sure? ")==0) break;
        mprintf(1,"\nFilename:");
        port_readline(tempbuffer2,65,0);
        generate_filelist(tempbuffer2);
        port_crlf();
        break;
        }
      case '$': /* Trim messagebase */
        {                                  
        char numb[10]; int trim,maxmsg,a;
        mail_area trimarea;

        mprintf(1,"Trim messagearea\n\n{bfg c}Enter area#          :");
        port_readline(numb,4,NUMONLY); port_crlf();
        if (numb[0]==0) break;
        trim=atoi(numb);     
        read_area(trim,&trimarea);

        mprintf(1,"{bfg c}Enter max nr of msgs :");
        port_readline(numb,6,NUMONLY); port_crlf();
        if (numb[0]==0) break;
        maxmsg=atoi(numb);
 
        if ((trimarea.areaflags&FLAG_FILEAREA)!=0)
          {
          mprintf(1,"{bfg r}Area %d is NOT a messagebase{std}\n",trim);
          break;
          }

        if (trimarea.count<=maxmsg)
          {
          mprintf(1,"{bfg r}No messages to remove{std}\n");
          break;
          }

        a=trimarea.count-maxmsg;

        mprintf(1,"{bfg r}Will remove %d messages: confirm please! {fg g}",a);
        if (port_yesno("")==0) break;

        trim_area(trim,maxmsg,1);
        break;
        }
      case '1': /* Private mail relinker */
        {
        mail_block msg; user_block fixuser;
        clock_t lastpoll=clock(); char numb[10];
        int b,max=read_msgnumber(),msgcount,zerocount;
       
        mprintf(1,"Relink private mail\nUser number: ");
        port_readline(numb,6,NUMONLY); port_crlf();
        if (numb[0]==0) break;
        fixuser.usernumber=atoi(numb);

        if (port_yesno("Sure? ")==0) break;

        get_user(&fixuser);                                  
        if (fixuser.status!=0)
          {
          /* Start with NO mail */
          fixuser.mailcount=0;
          fixuser.mailpointer_start=fixuser.mailpointer_end=-1;
          put_user(&fixuser);
          }    
        else break;
           
        mprintf(1,"Relinking mail for user: %s (#%d)\n",fixuser.username,fixuser.usernumber);

        for(b=0;b<max;b++)
          {
          if ((clock()-lastpoll)>=MAXIMUM_POLL)
            {
            window_poll();
            lastpoll=clock();
            } 

          msg.message_number=b;
          get_messageh(&msg);          
             
          /* To the right person? */
          if (msg.message_area==0 && msg.status==1 && msg.to_id==fixuser.usernumber)
            {
            /* Get the whole message */
            get_message(&msg,message_buffer);
                      
            mprintf(1,"=%06d\015",b);

            /* Delete this message */
            msg.status=0;
            mod_writelookup(b,&msg);
            mod_freelookup(b);
            _mail_freeup(msg.block_forward);
            msg.status=1;
  
            if (msg.message_length>0)
              {
              /* Re-send it */                 
              msg.message_number=0;
              msg.message_forward=msg.message_backward=-1;
              put_message(&msg,message_buffer);  
              msgcount++;
              }
            else
              {
              zerocount++;
              }
            }
          }
        mprintf(1,"\n%d messages recovered (%d null msgs deleted)\n",msgcount,zerocount);   

        break;
        }
      case '2': /* Setup Echomail lastread */
        {
        user_block board;
        char numb[10]; int un;
       
        mprintf(1,"Setup echomail rescan\nNumber of messages to rescan: ");
        port_readline(numb,6,NUMONLY); port_crlf();
        if (numb[0]==0) break;

        if ((un=hash_find("EchoMail"))==NODATA)
          {
          mprintf(1,"Couldn't find EchoMail in userlog\n"); 
          break;
          }

        board.usernumber=un;
        get_user(&board);
        board.m_message=mod_getlastlookup()-atoi(numb);
        if (port_yesno("Sure? ")==0) break;
        put_user(&board);

        break;
        }
      case '3': /* Output whole message/filebase as one file */
        {
        mail_block msg;
        clock_t lastpoll=clock();
        int b,max=read_msgnumber(),mc=0;
        FILE *gen;
       
        mprintf(1,"Message/filebase export\n");
        if (port_yesno("Sure? ")==0) break;
        mprintf(1,"Filename:");
        port_readline(tempbuffer2,65,0);
        if(tempbuffer2[0]==0) break;

        if ((gen=fopen(tempbuffer2,"wb"))==NULL)
          {
          mprintf(1,"Can't open output file\n");
          break;
          }
           
        for(b=0;b<max;b++)
          {
          if ((clock()-lastpoll)>=MAXIMUM_POLL)
            {
            mprintf(1,">%06d\015",b);
            window_poll();
            lastpoll=clock();
            } 

          msg.message_number=b;
          get_messageh(&msg);          
          if (msg.status==1)
            {
            /* Active message/file, output */
            get_message(&msg,message_buffer);
            fputs("item",gen);
            fwrite(&msg,256,1,gen);
            if (msg.message_length>0)
              {
              fwrite(message_buffer,msg.message_length,1,gen);
              }
            mc++;
            }
          }                                    
        fclose(gen);
        mprintf(1,"\n%d items written to file\n",mc);   

        break;
        }
      case '4': /* Import messagebase */
        {
        mail_block msg; mail_area area; user_block fixuser;
        clock_t lastpoll=clock();
        int a,mc=0,maxuser=mod_maxuser();
        FILE *gen; char tag[4];

        mprintf(1,"Message/filebase IMPORT\n");
        if (port_yesno("This will blank all areas and all user mail pointers first...sure? ")==0) break;
        if (port_yesno("Totally sure you want to exit back to the menu instead?")) break;
        mprintf(1,"Filename:");
        port_readline(tempbuffer2,65,0);
        port_crlf();
        if(tempbuffer2[0]==0) break;

        if ((gen=fopen(tempbuffer2,"rb"))==NULL)
          {
          mprintf(1,"Can't open file\n");
          break;
          }

        mprintf(1,"Blanking all user mail...\n");
        for(a=0;a<maxuser;a++)
          {
          fixuser.usernumber=a;
          get_user(&fixuser);                                  
          mprintf(1,"User #%04d\015",a);

          if ((clock()-lastpoll)>=MAXIMUM_POLL)
            {
            window_poll();
            lastpoll=clock();
            } 
             
          if (fixuser.status!=0)
            {
            /* Start with NO mail */
            fixuser.mailcount=0;
            fixuser.mailpointer_start=fixuser.mailpointer_end=-1;
            put_user(&fixuser);
            }    
          }

        mprintf(1,"\nBlanking all areas...\n");
        for(a=1;a<mod_getlastarea();a++)
          {      
          read_area(a,&area);
          area.first_message=-1;
          area.last_message=-1;
          area.newest=area.count=area.user_id=0;
          area.user[0]=0;
          write_area(a,&area);

          if ((clock()-lastpoll)>=MAXIMUM_POLL)
            {
            window_poll();
            lastpoll=clock();
            } 
          }          
        mod_saveall();

        mprintf(1,"Importing from file...\n");
        while(feof(gen)==0)
          {
          if (fread(tag,4,1,gen)!=1) break;
          if (strncmp(tag,"item",4)!=0)
            {
            mprintf(1,"Corrupted file\n");
            break;
            }
          if (fread(&msg,256,1,gen)!=1) break;
          if (msg.message_length>0)
            {
            fread(message_buffer,msg.message_length,1,gen);
            }
            
          /* Send the message */
          put_messagei(&msg,message_buffer); mc++;

          if ((clock()-lastpoll)>=1000)
            {
            mod_ensurelookup(); mod_ensuredata(); mod_savemaps();
            mprintf(1,"Imported %5d items\015",mc);
            window_poll();
            lastpoll=clock();
            } 
          }         

        fclose(gen);
        mprintf(1,"\nImport finished\n");
        break;
        }
      case '5': /* Compress datamap */
        {
        int a,lookuplen=mod_getlastlookup(),maxdist=(mod_dataext()/8),
            trynew,wrt=0,rlm=0;
        clock_t lastpoll=clock();
        mail_block m; char msgs[40];

        mprintf(1,"Compress mail.data\n");
        if (port_yesno("Really sure? ")==0) break;

        mprintf(1,"Working...");

        for(a=0;a<lookuplen;a++)
          {
          m.message_number=a;
          get_messageh(&m);
          if (m.status && m.message_length)
            {
            trynew=mod_lockdata(m.message_length);

            if (trynew<0)
              {
              mprintf(1,"Major problems! Call Hugo & tell him 'no room when reallocating dataspace'\n");
              break;
              }                            

            /* Only move if it's worth the hassle */
            if (trynew<(m.block_forward-maxdist))
              {
              get_message(&m,message_buffer);

              /* Delete old text chain */
              if (_mail_freeup(m.block_forward)!=OK)
                {
                error("eError whilst freeing up chain for msg#%d",m.message_number);
                mprintf(1,"Major problems! Call Hugo & tell him 'error when freeing chain'\n");
                break;
                }
        
              /* Write text in new space */
              m.block_forward=trynew;
              put_messageh(&m);
              if (_mail_buffertodatai(message_buffer,m.message_length,trynew)!=OK)
                {
                error("eCouldn't write new text in datamap crunch");
                mprintf(1,"Major problems! Call Hugo & tell him 'error when writing in new space'\n");
                break;
                }                  
              sprintf(msgs,"Relocated messagetext #%5d",a); rlm++;
              wrt=1;
              }
            else mod_freedata(trynew);

            if ((clock()-lastpoll)>1000)
              {
              if (wrt) { mod_ensuredata(); mod_savemaps(); wrt=0; }
              mprintf(1,"\015%s",msgs);
              window_poll(); lastpoll=clock();
              }
            } 
          }
          
        mod_ensuredata(); mod_savemaps();
        mprintf(1,"\015%s\nDone... relocated message text of %d messages\n",msgs,rlm);
        break;
        }
      }
    }
  while(ch!='Q');
  port_txstring("Quit\n",1);
  }

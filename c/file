/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> File non-overlay bits                       <]
Current version   [> 00.16                                       <]
Version date      [> 04-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
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

extern int kermittx(char*),kermitrx(char*);

extern FILE *bbs_file[10];
extern void _fclose(FILE**),message_upload(char*,int);
                    
void file_download(int filenumber,int filesize,char *filename,int download,int countit)
  {
  /* Check download ratio(n?) */
  if (countit)
    {        
    if (download)
      {
      if (user.downloads>=(user.ratio*(user.uploads+1)))
        {
        bbs_sendfile("<ARCbbs$prompts>.Exceed",0,0);
        return;
        }
      }
    else
      {
      if ((user.downloads+queue_length)>=(user.ratio*(user.uploads+1)) && strcmp(getenv("ARCbbs$system"),"cryton")!=0)
        {
        bbs_sendfile("<ARCbbs$prompts>.Exceed",0,0);
        return;
        }
      }
    }

  if (download)
    {
    int option;
    char buffer[40];
    
    sprintf(buffer,"<ARCbbs$temp>.Batch_%d",portnumber);
    if ((bbs_file[4]=fopen(buffer,"wb"))==NULL)
      {
      mprintf(1,"{bfg r}Can't write batchfile (%s){std}\n",buffer);
      return;
      }

    fputs(file_pathname(filenumber),bbs_file[4]);
    fputs("\n",bbs_file[4]);
    fputs(filename,bbs_file[4]);
    fputs("\n",bbs_file[4]);
    fclose(bbs_file[4]); bbs_file[4]=0;

    dlmenu1:
    bbs_sendfile("<ARCbbs$prompts>.Download",0,0);
    
    dlmenu2:
    switch(option=toupper(port_get()))
      {       
      case 'Q':
        {
        /* Queue file for download */
        file_download(filenumber,filesize,filename,0,countit);

        break;
        }
      case 'Z':
        {
        port_txstring("Zmodem\n",1);
        zmodem_error(zmodem(1),1,countit); break;
        }
      case 'S':
        {
        port_txstring("SEAlink\n",1);
        zmodem_error(sealink(1),1,countit); break;
        }
      case 'K':
        {
        port_txstring("Kermit\n",1);
        zmodem_error(kermit(1),1,countit); break;
        }
      case 'Y':
        {
        port_txstring("Ymodem Batch\n",1);
        zmodem_error(ymodem(1),1,countit); break;
        }
      case 'X':
      case '1':
        {
        port_txstring(option=='X'?"Xmodem\n":"Xmodem-1k\n",1);
        xmodem_error(xmodem(file_pathname(filenumber),(option=='X')?0:1,1),1,countit);
        break;
        }
      case 'C':
        {
        port_txstring("Cancelled\n",1); break;
        }
      case '?':
        {
        goto dlmenu1;
        }
      default:
        {
        goto dlmenu2;
        }
      }
    }
  else
    {
    if (queue_length)
      {
      int a,b;
      for(a=0;a<queue_length;a++)
        {
        if (filenumber==download_queue[a])
          {
          for(b=a;b<(queue_length-1);b++)
            {
            download_queue[b]=download_queue[b+1];
            download_queuesize[b]=download_queuesize[b+1];
            strcpy(download_queuename[b],download_queuename[b+1]);
            }
          queue_length--;
          mprintf(1,"\n{bfg g}%06d {fg w}%s {std}removed from queue.\n",filenumber,filename);
          queue_time();
          return;
          }
        }
      }

    if (queue_length==64)
      {
      port_txstring("\n{bfg g}Your queue is {fg r}FULL{fg g}.",1);
      queue_time();
      }
    else
      {
      int totallength=0,dtime=0,tme=(thislogon*60)-((clock()-logon_time)/100);

      if (queue_length)
        {
        int a;

        for(a=0;a<queue_length;a++) totallength+=download_queuesize[a];
        }

      if (baudrate)
        {
        dtime=(totallength*100)/(baudrate*10);
        }                                     
                               
      if (tme<dtime && strcmp(getenv("ARCbbs$system"),"cryton")!=0)
        {
        mprintf(1,"\n{bfg r}You do not have enough time to download the queue with this file added.{std}\n");
        return;
        }

      download_queue[queue_length]=filenumber;
      download_queuesize[queue_length]=filesize;
      strcpy(download_queuename[queue_length++],filename);
      mprintf(1,"\n{bfg g}%06d {fg w}%s {fg y}added to queue.{std}\n",filenumber,filename);
      queue_time();
      }
    }
  }
       
void queue_download(int countit)
  {
  int a;
  char buffer[40];
  
  if (queue_length)
    {
    queue_time();

    sprintf(buffer,"<ARCbbs$temp>.Batch_%d",portnumber);
    if ((bbs_file[4]=fopen(buffer,"wb"))==NULL)
      {
      mprintf(1,"{bfg r}Can't write batchfile (%s){std}\n",buffer);
      return;
      }

    for(a=0;a<queue_length;a++)
      {
      fputs(file_pathname(download_queue[a]),bbs_file[4]);
      fputs("\n",bbs_file[4]);
      fputs(download_queuename[a],bbs_file[4]);
      fputs("\n",bbs_file[4]);
      }

    fclose(bbs_file[4]); bbs_file[4]=0;
    }
  else
    {
    port_txstring("{bfg r}Your download queue is empty!{std}\n",1);
    return;
    }

  dlmenu1:
  bbs_sendfile("<ARCbbs$prompts>.DownloadQ",0,0);
          
  dlmenu2:
  switch(toupper(port_get()))
    {       
    case 'Z':
      {
      port_txstring("Zmodem\n",1);
      zmodem_error(zmodem(1),1,countit);
      break;
      }
    case 'S':
      {
      port_txstring("SEAlink\n",1);
      zmodem_error(sealink(1),1,countit);
      break;
      }
    case 'K':
      {
      port_txstring("Kermit\n",1);
      zmodem_error(kermit(1),1,countit);
      break;
      }
    case 'Y':
      {
      port_txstring("Ymodem Batch\n",1);
      zmodem_error(ymodem(1),1,countit);
      break;
      }
    case 'C':
      {
      port_txstring("Cancelled\n",1); break;
      }
    case '?':
      {
      goto dlmenu1;
      }
    default:
      {
      goto dlmenu2;
      }
    }
  }

int zmodem(int tx)
  {
  char buffer[30];

  if (portnumber>8) return(CANCEL);

  strcpy(online_users[portnumber].doing,tx?"Zmodem TX":"Zmodem RX");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=1,
                  0,online_users[portnumber].doing);

  mprintf(1,"{bfg g}Start your {fg r}Zmodem{fg g} %s now{std}\n",tx?"download":"upload");
    
  /* Leave Xon/Xoff alone, Zmodem coexists fine with these (ZDLE's them) */

  sprintf(buffer,"<ARCbbs$temp>.Batch_%d",portnumber);

  if (tx) got_code=zmodemtx(buffer); else got_code=zmodemrx(buffer);

  strcpy(online_users[portnumber].doing,"Zmodem");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Zmodem");
          
  return(got_code);
  }

int sealink(int tx)
  {
  char buffer[30];

  if (portnumber>8) return(CANCEL);

  strcpy(online_users[portnumber].doing,tx?"SEAlink TX":"SEAlink RX");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=1,
                  0,online_users[portnumber].doing);

  mprintf(1,"{bfg g}Start your {fg r}SEAlink{fg g} %s now{std}\n",tx?"download":"upload");

  /* Kill off Xon/Xoff to pass binary thru */
  port_xonxoff(0);

  sprintf(buffer,"<ARCbbs$temp>.Batch_%d",portnumber);

  if (tx) got_code=seatx(buffer); else got_code=searx(buffer);
                                 
  /* Re-enable Xon/Xoff */
  port_xonxoff(1);

  strcpy(online_users[portnumber].doing,"SEAlink");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"SEAlink");

  return(got_code);
  }

int kermit(int tx)
  {
  char buffer[30];

  if (portnumber>8) return(CANCEL);

  strcpy(online_users[portnumber].doing,tx?"Kermit TX":"Kermit RX");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=1,
                  0,online_users[portnumber].doing);

  mprintf(1,"{bfg g}Start your {fg r}Kermit{fg g} %s now{std}\n",tx?"download":"upload");

  /* Kill off Xon/Xoff to pass binary thru */
  port_xonxoff(0);

  sprintf(buffer,"<ARCbbs$temp>.Batch_%d",portnumber);

  if (tx) got_code=kermittx(buffer); else got_code=kermitrx(buffer);
                                 
  /* Re-enable Xon/Xoff */
  port_xonxoff(1);

  strcpy(online_users[portnumber].doing,"Kermit");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Kermit");

  return(got_code);
  }

int ymodem(int tx)
  {
  char buffer[30];

  if (portnumber>8) return(CANCEL);

  strcpy(online_users[portnumber].doing,tx?"Ymodem TX":"Ymodem RX");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=1,
                  0,online_users[portnumber].doing);

  mprintf(1,"{bfg g}Start your {fg r}Ymodem batch{fg g} %s now{std}\n",tx?"download":"upload");

  /* Kill off Xon/Xoff to pass binary thru */
  port_xonxoff(0);

  sprintf(buffer,"<ARCbbs$temp>.Batch_%d",portnumber);

  if (tx) got_code=ymodemtx(buffer); else got_code=ymodemrx(buffer);
                                 
  /* Re-enable Xon/Xoff */
  port_xonxoff(1);

  strcpy(online_users[portnumber].doing,"Ymodem");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Ymodem");

  return(got_code);
  }

int xmodem(char *filename,int onek,int tx)
  {
  int len,blks;
   
  if (portnumber>8) return(CANCEL);
                                   
  strcpy(online_users[portnumber].doing,tx?"Xmodem TX":"Xmodem RX");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=1,
                  0,online_users[portnumber].doing);

  if (tx)
    {
    if ((bbs_file[9]=fopen(filename,"rb"))==NULL) return(0);
    fseek(bbs_file[9],0,SEEK_END); len=ftell(bbs_file[9]);
    fclose(bbs_file[9]); bbs_file[9]=0;
    blks=(len/(onek?1024:128))+1;                             
 
    mprintf(1,"{bfg g}File is {fg w}%d {fg g}byte%s ({fg w}%d{fg g} block%s) long.\n",
              len,(len!=1)?"s":"",blks,(blks!=1)?"s":"");
    }

  mprintf(1,"Start your {fg r}%s{fg g} %s now{std}\n",onek?"Xmodem-1k":"Xmodem",
                 tx?"download":"upload"); 
                                             
  /* Kill off Xon/Xoff to pass binary thru */
  port_xonxoff(0);
                                         
  if (tx)
    {
    got_code=xmodemtx(filename,onek);
    }
  else
    {
    got_code=xmodemrx(filename);
    }
                  
  /* Re-enable Xon/Xoff */
  port_xonxoff(1);

  strcpy(online_users[portnumber].doing,"Xmodem");
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Xmodem");

  return(got_code);
  }

int xmodem_error(int error,int download,int countit)
  {
  port_crlf();
  switch(error)
    {
    case DONE:
      {
      port_txstring("{bfg g}Transfer successful{std}\n",1);
      if (download)
        {    
        if (countit)
          {
          /* Update user */
          get_user(&user);
          user.downloads++;
          put_user(&user);
          }
        send_message(servertask,BBS_FILECOUNTS,1,0,0,0);
        }
      break;
      }
    case TIMEOUT:
      {
      port_txstring("{bfg r}Timeout{std}\n",1);
      break;
      }
    case FILE_ERROR:
      {
      port_txstring("{bfg r}Filing system error{std}\n",1);
      break;
      }
    case DISK_FULL:
      {
      port_txstring("{bfg r}Disk full{std}\n",1);
      break;
      }
    case ERRORS:
      {
      port_txstring("{bfg r}Too many errors{std}\n",1);
      break;
      }
    case CANCEL:
      {
      port_txstring("{bfg r}Transfer cancelled{std}\n",1);
      break;
      }
    case CARRIER:
      {
      break;
      }
    }
  return(error);
  }

int zmodem_error(int error,int download,int countit)
  {
  port_crlf();
  switch(error)
    {
    case DONE:
      {
      if (files_done!=0)
        {
        mprintf(1,"{bfg g}Transfer successful (%d %s transferred){std}\n",
                files_done,files_done==1?"file":"files");
        }
      else
        {
        mprintf(1,"{bfg g}Transfer unsuccessful, no files transferred{std}\n");
        }
      if (download)
        {         
        if (countit)
          {
          /* Update user */
          get_user(&user);
          user.downloads+=files_done;
          put_user(&user);
          }
        send_message(servertask,BBS_FILECOUNTS,files_done,0,0,0);
        }
      break;
      }
    case TIMEOUT:
      {
      port_txstring("{bfg r}Timeout{std}\n",1);
      break;
      }
    case FILE_ERROR:
      {
      port_txstring("{bfg r}Filing system error{std}\n",1);
      break;
      }
    case DISK_FULL:
      {
      port_txstring("{bfg r}Disk full{std}\n",1);
      break;
      }
    case ERRORS:
      {
      port_txstring("{bfg r}Too many errors{std}\n",1);
      break;
      }
    case CANCEL:
      {
      port_txstring("{bfg r}Transfer cancelled{std}\n",1);
      break;
      }
    case CARRIER:
      {
      break;
      }
    }
  return(error);
  }

void file_upload(int area,int filelength,char *realname,char *filename)
  {
  mail_block header; int flags=0,type=0xfff; clock_t when; FILE *tempfile;
  mail_area areab; char settype[256],*p;

  memset(&header,0,255);
  header.reply_from=-1; header.message_area=area;
  header.from_id=user.usernumber;
  strcpy(header.from,user.username);
  strcpy(header.to,filename);
  header.to_id=0; /* Number of downloads */
  header.file_length=filelength;
  header.file_location=get_filenumber();

  /* Set date sent + expire 1 year afterwards */
  header.date_entered=header.date_sent=time(NULL);
     
  /* No linedrop */
  notimeout=1;
                  
  if (area==0)
    {
    message_upload(realname,0);
    return;
    }      

  read_area(area,&areab);
  mprintf(1,"\n{fg c}Filename: {fg w}%s\n{fg c}Date    : {fg w}%s\n{fg c}Uploader: {fg w}%s (#%d)\n",
           filename,cctime(&header.date_sent),user.username,user.usernumber);
  header.subject[0]=0;

  strcpy(settype,filename); p=settype; while(*p) *p++=tolower(*p);
  if (strstr(settype,"fastcash")!=NULL)
    {
    mprintf(1,"\n{bfg r}NOTE: ANY FALSE CLAIMS IN YOUR FILE OR FILE DESCRIPTION MAY LEAD TO PROSECUTION\nYour upload count has been decremented. hahahahahaha!\n\n");
    get_user(&user);
    if (user.uploads>0) user.uploads--;
    put_user(&user);
    }

  if (!entertext(&header,1)) header.message_length=0;

  if ((tempfile=fopen(realname,"rb"))!=NULL)
    {                                                                            
    port_txstring("\n{bfg r}Checking file type...{std}",1);
    if (fgetc(tempfile)==0x1a) { flags|=FILE_ARC; type=0xddc; }
    else fseek(tempfile,0,SEEK_SET);

    if (flags==0)
      {
      do
        {
        /* To ensure regular polling */
        window_poll();
        when=clock();
        do
          {
          int ch=fgetc(tempfile);
          if (ch>127 || (ch<32 && ch!=10 && ch!=13 && ch!=9 && ch!=12)) { flags|=FILE_BINARY; type=0; }
          }
        while(!flags && (clock()-when)<10 && !feof(tempfile));
        }
      while(!flags && !feof(tempfile));
      }
    fclose(tempfile);      
    }
  else
    {
    mprintf(1,"{bfg r}\nFATAL ERROR: Upload not found. Report to sysop!{std}\n");
    return;
    }

  /* Set binary or whatever */
  header.flags=flags;
  if ((areab.areaflags&FLAG_AUTOINVISIBLE)!=0) header.flags|=FILE_SYSOP;

  mprintf(1,"\n{bfg g}Adding entry for file #%06d... (Description %d bytes)",
                   header.file_location,header.message_length);
  if (put_message(&header,message_buffer)==OK)
    {
    mprintf(1,"...added{std}\n",got_code2);
    }

  /* Set the type of the file */
  if (type)
    {
    sprintf(settype,"SetType %s %x",realname,type);
    os_cli(settype);
    }

  /* Rename upload file to filepath file */
  rename(realname,file_pathname(header.file_location));

  /* Update user's flags */
  if ((areab.areaflags&FLAG_FREEUPLOAD)==0)
    {
    get_user(&user); user.uploads++; put_user(&user);
    }
  send_message(servertask,BBS_FILECOUNTS,0,1,0,0);
  }

void queue_time()
  {
  int totallength=0,dtime=0,a;

  if (queue_length)
    {
    for(a=0;a<queue_length;a++) totallength+=download_queuesize[a];
    }

  if (baudrate)
    {
    dtime=(totallength*100)/(baudrate*10);
    }
                                                                            
  if (strcmp(getenv("ARCbbs$system"),"cryton")==0)
    {                              
    int disks;               

    disks=(totallength/805000)+1; /* Say 780k per disk */
    mprintf(1,"{bfg w}%d {fg c}entr%s in queue, approx download time {fg w}%d:%02d {fg c}(or {fg w}%d {fg c}disk%s){std}\n",
          queue_length,queue_length==1?"y":"ies",dtime/60,dtime%60,
          disks,disks==1?"":"s");
    }
  else
    {
    mprintf(1,"{bfg w}%d {fg g}entr%s in queue, approx download time {fg w}%d:%02d.{std}\n",
          queue_length,queue_length==1?"y":"ies",
          dtime/60,dtime%60);
    }
  }

void queue_show()
  {
  if (queue_length)
    {
    int a,totalsize=0,dtime;

    port_txstring("{bfgbg wb}Number {fg y}Name                 {fg w}Length {fg y}DL time{std}\n",1);
    if (user.termtype!=TER_ANSI) port_txstring("------ -------------------- ------ -------\n",1);

    for(a=0;a<queue_length;a++)
      {
      dtime=0;

      if (baudrate)
        {
        dtime=(download_queuesize[a]*100)/(baudrate*10);
        }

      totalsize+=download_queuesize[a];
      mprintf(0,"{bfg g}%06d {fg w}",download_queue[a]);  
      mprintf(16,"%-20s ",download_queuename[a]);
      if (mprintf(0,"{fg y}%6d {fg c}%4d:%02d\n",download_queuesize[a],
                  dtime/60,dtime%60))
        {
        port_crlf();
        break;
        }
      }
    port_txstring("                           ------- -------\n",1);

    dtime=0;
    if (baudrate)
      {
      dtime=(totalsize*100)/(baudrate*10);
      }

    if (strcmp(getenv("ARCbbs$system"),"cryton")==0)
      {                              
      int disks;               

      disks=(totalsize/805000)+1; /* Say 780k per disk */
      mprintf(1,"                   {bfg c}Total   {fg y}%7d {fg w}%4d:%02d {fg c}(or {fg w}%d {fg c}disk%s){std}\n",totalsize,
              dtime/60,dtime%60,disks,disks==1?"":"s");
      }
    else
      {
      mprintf(1,"                   {bfg w}Total   {fg y}%7d {fg c}%4d:%02d{std}\n",totalsize,
                dtime/60,dtime%60);
      }
    }
  else
    {
    port_txstring("{bfg r}You have no files in your download-queue{std}\n",1);
    }
  }

void order_disk()
  {
  int disks,a,totalsize=0,cost;               

  if (strcmp(getenv("ARCbbs$system"),"cryton")) return;
                             
  if (queue_length==0)
    {
    port_txstring("{bfg r}You have no files in your download-queue{std}\n",1);
    return;
    }

  for(a=0;a<queue_length;a++)
    {
    totalsize+=download_queuesize[a];
    }

  disks=(totalsize/805000)+1; /* Say 780k per disk */
                                                                    
  queue_show();
  if (port_yesno("{fg c}Are you sure you want to order these files on disk? "))
    {
    char cardnumber[21],cardtype[11],cardname[31],address[4][31],
         postcode[11],cardexp[6];

    for(a=0;a<4;a++) strcpy(address[a],user.address[a]);
    strcpy(postcode,user.postcode);
                                   

    mprintf(1,"{bfg c}We have your address stored as:\n{fg w}  %s\n  %s\n  %s\n  %s\n  %s\n{fg c}",address[0],address[1],address[2],address[3],postcode);
tryenter:
    if (port_yesno("Is this correct? ")==0)
      {
      mprintf(1,"Enter new address:\n");
      for(a=0;a<4;a++)
        {        
        mprintf(1,"{bfg c}Line %d/4: {fg w}",a+1);
        port_readline(address[a],30,EXISTING);
        port_crlf();
        }
      mprintf(1,"{bfg c}Postcode : {fg w}");
      port_readline(postcode,10,EXISTING);
      port_crlf();
      goto tryenter;
      }                            
    mprintf(1,"{bfg c}Credit card type   : {fg w}");
    port_readline(cardtype,10,0);
    port_crlf();
    mprintf(1,"{bfg c}Credit card number : {fg w}");
    port_readline(cardnumber,20,0);     
    port_crlf();
    mprintf(1,"{bfg c}Expiry date        : {fg w}");
    port_readline(cardexp,5,0);     
    port_crlf();
    mprintf(1,"{bfg c}Cardholders name   : {fg w}");
    port_readline(cardname,30,0);
    port_crlf();
                                   
    cost=200+(150*(disks-1));                          

    mprintf(1,"{bfg c}A total of {fg w}%c%d.%02d {fg c}will be charged to your card.\nDo you wish to continue? ",156,cost/100,cost%100);
    if (port_yesno(""))
      {                                
      int ordernr=time(NULL);                                          

      if ((bbs_file[5]=fopen("<ARCbbs$order>.Log","a"))==NULL)
        {
        mprintf(1,"{bfg r}Unable to log order - order aborted. Please report problem to Sysop.\n");
        return;
        }
      fprintf(bbs_file[5],"# %s %08X DiskOrder: User %s (#%d), %d disk%s (%d.%02d)\n",cctime((time_t*)&ordernr),ordernr,user.username,user.usernumber,disks,disks==1?"":"s",cost/100,cost%100);
      fprintf(bbs_file[5],"  Name=%s Type=%s Number=%s Exp=%s\n",cardname,cardtype,cardnumber,cardexp);
      fprintf(bbs_file[5],"  %s\n  %s\n  %s\n  %s\n  %s\n",address[0],address[1],address[2],address[3],postcode); 
      fprintf(bbs_file[5],"  Ordered files:\n");
      for(a=0;a<queue_length;a++)
        {
        fprintf(bbs_file[5],"  %06d %-20s %6d\n",download_queue[a],download_queuename[a],download_queuesize[a]);  
        }
      fprintf(bbs_file[5],"                             -------\n");
      fprintf(bbs_file[5],"                     Total   %7d\n",totalsize);
      _fclose(&bbs_file[5]);

      if ((bbs_file[5]=fopen("<ARCbbs$order>.ToDo","a"))==NULL)
        {
        mprintf(1,"{bfg r}Unable to log order - order aborted. Please report problem to Sysop.\n");
        return;
        }
      fprintf(bbs_file[5],"%d\n",disks);
      fprintf(bbs_file[5],"%s\n",cardname);
      fprintf(bbs_file[5],"%s\n",cardtype);
      fprintf(bbs_file[5],"%s\n",cardnumber);
      fprintf(bbs_file[5],"%s\n",cardexp);
      fprintf(bbs_file[5],"%s\n%s\n%s\n%s\n%s\n",address[0],address[1],address[2],address[3],postcode);
      for(a=0;a<queue_length;a++)
        {                                                 
        fprintf(bbs_file[5],"%s\n",download_queuename[a]);
        fprintf(bbs_file[5],"%d %d\n",download_queue[a],download_queuesize[a]);  
        }                       
      fprintf(bbs_file[5],"end\n0 0\n");
      _fclose(&bbs_file[5]);

      if ((bbs_file[5]=fopen("<ARCbbs$order>.Charge","a"))==NULL)
        {
        mprintf(1,"{bfg r}Unable to log order - order aborted. Please report problem to Sysop.\n");
        return;
        }
      fprintf(bbs_file[5],"# %s %08X DiskOrder: User %s (#%d), %d disk%s (%d.%02d)\n",cctime((time_t*)&ordernr),ordernr,user.username,user.usernumber,disks,disks==1?"":"s",cost/100,cost%100);
      fprintf(bbs_file[5],"  Name=%s Type=%s Number=%s Exp=%s\n",cardname,cardtype,cardnumber,cardexp);
      fprintf(bbs_file[5],"  %s\n  %s\n  %s\n  %s\n  %s\n",address[0],address[1],address[2],address[3],postcode); 
      fprintf(bbs_file[5],"  Ordered files:\n");
      for(a=0;a<queue_length;a++)
        {
        fprintf(bbs_file[5],"  %06d %-20s %6d\n",download_queue[a],download_queuename[a],download_queuesize[a]);  
        }
      fprintf(bbs_file[5],"                             -------\n");
      fprintf(bbs_file[5],"                     Total   %7d\n",totalsize);
      _fclose(&bbs_file[5]);

      mprintf(1,"{bfg w}Order queued. Your disk%s will be despatched within 2 working days.\nPlease note dowm this order reference number incase of queries: {fg w}%08X\n",disks==1?"":"s",ordernr);
      }
    }
  }

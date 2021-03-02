/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs / ARCterm                            <]
Author            [> Hugo Fiennes                                <]
Date started      [> 02-August-1988                              <]
                  [>                                             <]
Module name       [> Xmodem file transfer                        <]
Current version   [> 01.56                                       <]
Version date      [> 03-April-1993                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>   This source is COPYRIGHT © 1989-1993 by   <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

/* Main includes */
#include "include.h"

/* System includes */
#ifdef SEVEN
/* filename=0 message=4 pos=1 size=3 */
#include "batch.h"
#include "term.h"
#include "msgs.h"
extern terminal_configuration current;

#define DONE    0
#define CANCEL -1
#define FILE_ERROR -2
#define ERRORS -3
#define DISK_FULL -4
#define TIMEOUT -5
#else
#include "servmess.h"
#include "config.h"
#endif

#include "port.h"
#include "crc.h"

#define SOH       1
#define STX       2
#define EOT       4
#define ACK       6
#define ERRORMAX 10
#define NAK      21
#define CAN      24
#define CTIMEOUT -1
#define ABORT    -2

static int  flag,xmodem_abort;
extern void window_poll(void),_fclose(FILE**);

#ifdef SEVEN
#define HDRTIMEOUT  current.xmodem_headertimeout
#define DATATIMEOUT current.xmodem_datatimeout
#define MAXRETRY    current.xmodem_maxretries
#else
#define HDRTIMEOUT  10
#define DATATIMEOUT  1
#define MAXRETRY    10
#endif

/* WIMP stuff */
#ifdef SEVEN
extern terminal_configuration current;
extern void write_iconint(int,int,int),write_icon(int,int,char*);
jmp_buf killx;
extern FILE *bbs_file[10];
extern int files_done,c_pressed(void);
extern char *strippath(char*);
extern wimp_w zmodem_handle;
#else
extern FILE *bbs_file[10];
extern int  files_done;
#endif

extern char *message_buffer;
static char *buffer;

#ifdef SEVEN
extern char rxpath[128];

void xmodem_msgproc(wimp_eventstr *e,void *handle)
  {
  handle=handle;

  /* Reselect our port */
  port_select();

  switch(e->e)
    {
    case wimp_EOPEN:             /* Open_Window_Request */
      {
      wimp_open_wind(&e->data.o);
      break;
      }
    case wimp_ECLOSE:             /* Close_Window_Request */
      {
      xmodem_abort=1;
      break;
      }
    }
  }
#endif

int readline(int seconds)
  {
  clock_t start=clock(); int d;

  while((clock()-start)<(seconds*100))
    {
    if ((d=port_rx())>=0) return(d); else window_poll();
#ifdef SEVEN
    if (c_pressed()) return(CTIMEOUT);
    if (xmodem_abort) longjmp(killx,1);
#endif
    }
  return(CTIMEOUT);
  }

void gdata(char *c,int len)
  {
  int a;

  if ((a=fread(c,1,len,bbs_file[4]))<len)
    {
    for(;a<len;a++) c[a]=26;
    flag=TRUE;
    }
  }

int pdata(char *c,int size)
  {
  if (fwrite(c,size,1,bbs_file[4])!=1) return(EOF); else return(0);
  }

int chkok(int crc)
  {
  int a,b;
  if (crc)
    {
    a=readline(DATATIMEOUT); b=readline(DATATIMEOUT);
    return((a*256)+b);
    }
  else
    {
    return(readline(DATATIMEOUT));
    }
  }

int xmodemrx(char *file)
  {
  int j,firstchar,sectnum=0,sectcurr,sectcomp,errors=0,toterr=0,checksum,
      errorflaga,crc=1,size,gotblock=0;
  clock_t start;
#ifdef SEVEN
  int pos=0;
#endif

  files_done=flag=0;
  buffer=message_buffer;

#ifdef SEVEN
  xmodem_abort=0; bbs_file[4]=0;
  if (setjmp(killx)==1)
    {
    _fclose(&bbs_file[4]);
    port_txw(CAN); port_txw(CAN);
    return(CANCEL);
    }
  write_icon(zmodem_handle,0,strippath(file));
  write_icon(zmodem_handle,2,"Unknown");
#endif

  if ((bbs_file[4]=fopen(file,"wb"))==0)
    {
    return(FILE_ERROR);
    }

  window_poll();

  port_rxclear();

  trystartagain:
  do
    {
#ifdef SEVEN
    sprintf(buffer+1024,msgs_lookup("7_rtry"),crc?"CRC":"checksum",errors+1);
    write_icon(zmodem_handle,4,buffer+1024);
#endif

    if (crc) port_txw('C'); else port_txw(NAK);
    start=clock();
    while((clock()-start)<300 && gotblock==0)
      {
      window_poll();
#ifdef SEVEN
      if (c_pressed()) start=clock()-300;
      if (xmodem_abort) longjmp(killx,1);
#endif
      gotblock=port_rx();
      if (gotblock!=SOH && gotblock!=STX) gotblock=0;
      }
    errors++;
    }
  while(errors<(crc?4:10) && gotblock==0);

  if (gotblock==0 && crc)
    {
    crc=errors=0;
    goto trystartagain;
    }

  if (gotblock==0)
    {
    _fclose(&bbs_file[4]);
    return(TIMEOUT);
    }

  errors=0;
  do
    {
    errorflaga=FALSE;
    do
      {
      if (gotblock)
        {
        firstchar=gotblock; gotblock=0;
        }
      else
        {
        firstchar=readline(HDRTIMEOUT);
        }
      }
    while(firstchar!=SOH && firstchar!=STX && firstchar!=EOT &&
          firstchar!=CAN && firstchar!=CTIMEOUT);

    if (firstchar==CAN)
      {
      if (readline(4)==CAN)
        {
        _fclose(&bbs_file[4]);
        return(CANCEL);
        }
      }

    if (firstchar==CTIMEOUT)
      {
#ifdef SEVEN
      write_icon(zmodem_handle,4,msgs_lookup("7_rnakt"));
#endif
      errorflaga=TRUE;
      }

    if (firstchar==SOH || firstchar==STX)
      {
#ifdef SEVEN
      write_icon(zmodem_handle,4,msgs_lookup("7_rdata"));
#endif

      size=(firstchar==SOH)?128:1024;
      if ((sectcurr=readline(DATATIMEOUT))==CTIMEOUT)
        { errorflaga=TRUE; goto goterror; }
      if ((sectcomp=readline(DATATIMEOUT))==CTIMEOUT)
        { errorflaga=TRUE; goto goterror; }

      if ((sectcurr+sectcomp)==255)
        {
        if (sectcurr==(sectnum+1)%256)
          {
          checksum=0;
          for(j=0;j<size && errorflaga==FALSE;j++)
            {
            if ((buffer[j]=readline(DATATIMEOUT))==CTIMEOUT) errorflaga=TRUE;
            if (!crc) checksum+=buffer[j];
            }

          if (errorflaga) goto goterror;

          if (crc) checksum=calcrc(buffer,size); else checksum&=0xff;
          if (chkok(crc)==checksum)
            {
            errors=0; sectnum++;
            if (pdata(buffer,size)==EOF)
              {
              _fclose(&bbs_file[4]);
              return(DISK_FULL);
              }

            port_txw(ACK);
#ifdef SEVEN
            pos+=size;
            write_iconint(zmodem_handle,1,pos);
            write_iconint(zmodem_handle,3,size);
            write_icon(zmodem_handle,4,msgs_lookup("7_rack"));
#endif

            }
          else
            {
#ifdef SEVEN
            write_icon(zmodem_handle,4,msgs_lookup("7_rnake"));
#endif
            errorflaga=TRUE;
            }
          }
        else
          {
          if (sectcurr==sectnum)
            {
            while(readline(DATATIMEOUT)!=TIMEOUT);
            port_txw(ACK);
#ifdef SEVEN
            write_icon(zmodem_handle,4,msgs_lookup("7_rackd"));
#endif
            }
          else
            {
#ifdef SEVEN
            write_icon(zmodem_handle,4,msgs_lookup("7_rnakm"));
#endif
            errorflaga=TRUE;
            }
          }
        }
      else
        {
#ifdef SEVEN
        write_icon(zmodem_handle,4,msgs_lookup("7_rnakg"));
#endif
        errorflaga=TRUE;
        }
      }

goterror:
    if (errorflaga==TRUE)
      {
      errors++;
      if (sectnum) toterr++;
      while(readline(DATATIMEOUT)!=CTIMEOUT);
      port_txw(NAK);
      }
    }
  while(firstchar!=EOT && errors!=ERRORMAX);

  _fclose(&bbs_file[4]);

  if (firstchar==EOT)
    {
    port_txw(ACK);
    files_done=1;
#ifdef SEVEN
    write_icon(zmodem_handle,4,"ACKing EOT");
#endif
    return(DONE);
    }
  else
    {
    return(ERRORS);
    }
  }

int xmodemtx(char *file,int onek)
  {
  int j,sectnum=1,crc=1,attempts=0,toterr=0,checksum,length,size,pos=0;

  files_done=flag=0;
  buffer=message_buffer;

#ifdef SEVEN
  xmodem_abort=0; bbs_file[4]=0;
  if (setjmp(killx)==1)
    {
    _fclose(&bbs_file[4]);
    port_txw(CAN); port_txw(CAN);
    return(CANCEL);
    }
#endif

  if((bbs_file[4]=fopen(file,"rb"))==0)
    {
    return(FILE_ERROR);
    }

  fseek(bbs_file[4],0,SEEK_END);
  length=(int)ftell(bbs_file[4]);
  fseek(bbs_file[4],0,SEEK_SET);

  port_rxclear();
#ifdef SEVEN
  write_icon(zmodem_handle,0,strippath(file));
  write_iconint(zmodem_handle,2,length);
  write_icon(zmodem_handle,4,msgs_lookup("7_xmsync"));
#endif

  notacan:
  do
    {
    attempts++;
    j=readline(HDRTIMEOUT);
    }
  while(j!='C' && j!=NAK && j!=CAN && attempts!=8);

  if (attempts==8)
    {
    _fclose(&bbs_file[4]);
    return(TIMEOUT);
    }

  if (j==CAN)
    {
    if (readline(4)==CAN)
      {
      _fclose(&bbs_file[4]);
      return(CANCEL);
      }
    else goto notacan;
    }

  if (j==NAK) crc--;

  attempts=0;

  while(attempts!=MAXRETRY && !flag)
    {
    if (onek && length>=1024) size=1024; else size=128;
    gdata(buffer,size);
    attempts=0;
    do
      {
      int count=0;

      port_txw(size==128?SOH:STX);
      port_txw(sectnum & 0xff);
      port_txw(255-(sectnum & 0xff));
      checksum=0;
      for(j=0;j<size;j++)
        {
        port_txw(buffer[j]);
        if (!crc) checksum+=buffer[j];
        }
      if (crc)
        {
        checksum=calcrc(buffer,size);
        port_txw(checksum>>8);
        }
      port_txw(checksum & 0xff);
      port_rxclear();
      attempts++; toterr++;
      do
        {
        j=readline(HDRTIMEOUT); count++;
        }
      while(j!=ACK && j!=CAN && j!=NAK && count<30);

#ifdef SEVEN
      if (j==NAK) write_icon(zmodem_handle,4,msgs_lookup("7_naked"));
#endif

      if (j==CAN) j=readline(4);
      }
    while(j!=ACK && j!=CAN && attempts!=MAXRETRY);

    if (j==CAN)
      {
      _fclose(&bbs_file[4]);
      return(CANCEL);
      }

#ifdef SEVEN
    pos+=size;
    write_iconint(zmodem_handle,1,pos);
    write_iconint(zmodem_handle,3,size);
    write_icon(zmodem_handle,4,msgs_lookup("7_acked"));
#endif

    sectnum++; toterr--; length-=size;
    }

  if (attempts==MAXRETRY)
    {
    _fclose(&bbs_file[4]);
    return(ERRORS);
    }
  else
    {
    attempts=0;
    do
      {
      port_txw(EOT);
      port_rxclear();
      attempts++;
      j=readline(HDRTIMEOUT);
      }
    while(j!=ACK && j!=CAN && attempts!=MAXRETRY);
    }
  _fclose(&bbs_file[4]);
  files_done=1;
#ifdef SEVEN
  batch_remove(file);
#endif
  return(DONE);
  }


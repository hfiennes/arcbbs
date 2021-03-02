/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs/ARCterm                              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 02-August-1988                              <]
                  [>                                             <]
Module name       [> Ymodem (& -g) batch file transfer           <]
Current version   [> 01.29                                       <]
Version date      [> 05-April-1993                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
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

#define END_BATCH 255

#ifdef SEVEN
#define HDRTIMEOUT  current.ymodem_headertimeout
#define DATATIMEOUT current.ymodem_datatimeout
#define MAXRETRY    current.ymodem_maxretries
#else
#define HDRTIMEOUT  10
#define DATATIMEOUT  1
#define MAXRETRY    10
#endif

/* WIMP stuff */
#ifdef SEVEN
extern terminal_configuration current;
extern void write_iconint(int,int,int),write_icon(int,int,char*);
jmp_buf killy;
extern FILE *bbs_file[10];
extern int files_done,c_pressed(void);
extern char *strippath(char*);
extern wimp_w zmodem_handle;
static int ymodem_abort;
#else
extern FILE *bbs_file[10];
extern int  files_done,portnumber;
#endif

static int  flag,gmode;
extern char *message_buffer;
static char *buffer;
static char *alias;

extern void window_poll(void),mygets(char*,FILE*),_fclose(FILE**),
            wimp_waitfor(int);
extern int  ymodemrxf(char*),ymodemtxf(char*);

#ifdef SEVEN
extern char rxpath[128];

void ymodem_msgproc(wimp_eventstr *e,void *handle)
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
      ymodem_abort=1;
      break;
      }
    }
  }
#endif

int yreadline(int seconds)
  {
  clock_t start=clock(); int d;

  do
    {
    if ((d=port_rx())>=0) return(d); else window_poll();
#ifdef SEVEN
    if (c_pressed()) return(CTIMEOUT);
    if (ymodem_abort) longjmp(killy,1);
#endif
    }
  while((clock()-start)<(seconds*100));
  return(CTIMEOUT);
  }

void ygdata(char *c,int len)
  {
  int a;

  if ((a=fread(c,1,len,bbs_file[4]))<len)
    {
    for(;a<len;a++) c[a]=26;
    flag=TRUE;
    }
  }

int ypdata(char *c,int size)
  {
  if (fwrite(c,size,1,bbs_file[4])!=1) return(EOF); else return(0);
  }

int ychkok()
  {
  int a,b;
  a=yreadline(DATATIMEOUT); b=yreadline(DATATIMEOUT);
  return((a*256)+b);
  }

#ifdef SEVEN
int ymodemrx(char *batchfilename,int mode)
  {
#else
int ymodemrx(char *batchfilename)
  {
  int mode=0;
#endif
  char genname[256];
  int status;

  /* Make local copy of -g mode */
  gmode=mode;
  buffer=message_buffer; alias=message_buffer+1024;
  files_done=0;

#ifdef SEVEN
  strcpy(rxpath,batchfilename);
  ymodem_abort=0; bbs_file[4]=0;
  if (setjmp(killy)==1)
    {
    _fclose(&bbs_file[4]);
    port_txw(CAN); port_txw(CAN);
    return(CANCEL);
    }

  while((status=ymodemrxf(genname))==DONE);
#else
  if ((bbs_file[3]=fopen(batchfilename,"wb"))==NULL)
    {
    return(FILE_ERROR);
    }

  do
    {
    sprintf(genname,"<ARCbbs$upload>.%02d/%04d",portnumber,files_done);
    if ((status=ymodemrxf(genname))==DONE)
      {
      fputs(alias,bbs_file[3]);
      fputs("\012",bbs_file[3]);
      files_done++;
      }
    }
  while(status==DONE);

  _fclose(&bbs_file[3]);
#endif

  if (files_done==0)
    {
    if (status!=CANCEL) return(FILE_ERROR); else return(CANCEL);
    }

  return(DONE);
  }

int ymodemrxf(char *filename)
  {
  int a,j,firstchar,sectnum=0,sectcurr,sectcomp,errors=0,checksum,
      errorflaga,size,left,fsize,pos=0,useattr=0,load,exec;
  clock_t start;

#ifdef SEVEN
  write_icon(zmodem_handle,0,"");
  write_icon(zmodem_handle,1,"");
  write_icon(zmodem_handle,2,"Unknown");
  sprintf(buffer+1024,msgs_lookup("7_rtry"),"CRC",errors+1);
  write_icon(zmodem_handle,4,buffer+1024);
#endif

  flag=0;

  window_poll();
  port_rxclear();
  errors=0;

  getheader:
  if (errors) while(yreadline(DATATIMEOUT)!=CTIMEOUT);

  /*** Try and get header packet */
  do
    {
    /*** Send try */
    port_txw(gmode?'G':'C');
    errors++;

    /*** Wait for SOH */
    start=clock();
    notsoh:
    while((clock()-start)<300)
      {
#ifdef SEVEN
      if (c_pressed()) break;
      if (ymodem_abort) longjmp(killy,1);
#endif
      window_poll();
      if ((j=port_rx())>=0) break;
      }

    if (j>=0)
      {
      if (j==CAN)
        {
        if (yreadline(4)==CAN)
          {
          _fclose(&bbs_file[4]);
          return(CANCEL);
          }
        }
      if (j!=SOH) goto notsoh;
      }
    else
      {
      j=0;
#ifdef SEVEN
      sprintf(buffer+1024,msgs_lookup("7_rtry"),"CRC",errors);
      write_icon(zmodem_handle,4,buffer+1024);
#endif
      }

    if (j==CAN)
      {
      if (yreadline(4)==CAN) return(CANCEL);
      }
    }
  while(errors<10 && j!=SOH);
  if (j!=SOH)
    {
    _fclose(&bbs_file[4]);
    return(TIMEOUT);
    }

#ifdef SEVEN
  write_icon(zmodem_handle,4,msgs_lookup("7_rdata"));
#endif

  /*** Read in header block */
  if (yreadline(DATATIMEOUT)!=0  ) goto getheader;
  if (yreadline(DATATIMEOUT)!=255) goto getheader;
  for(j=0;j<128;j++)
    {
    if ((buffer[j]=yreadline(DATATIMEOUT))==CTIMEOUT) goto getheader;
    }
  checksum=calcrc(buffer,128);
  if (ychkok()!=checksum) goto getheader;
  port_txw(ACK);

  /*** Got good header */
  if (buffer[0]==0) return(END_BATCH);

  if ((fsize=left=atoi(buffer+strlen(buffer)+1))<=0)
    {
    fsize=left=0xfffffff;
    }
#ifdef SEVEN
  else
    {
    write_iconint(zmodem_handle,2,fsize);
    }

  buffer[10]=0;
  for(a=0;a<10;a++)
    {
    if (buffer[a]=='.' || buffer[a]=='%' ||
        buffer[a]=='$' || buffer[a]=='@' ||
        buffer[a]=='&' || buffer[a]==':') buffer[a]='_';
    }
  sprintf(filename,"%s%s",rxpath,buffer);
  write_icon(zmodem_handle,0,buffer);
#else
  buffer[20]=0; /* Trim filename length */
  strcpy(alias,buffer);
#endif
                      
  /*** Check for attributes */
  if (buffer[124]==3)
    {
    useattr++;
    load =(buffer[120]<< 0);
    load|=(buffer[121]<< 8);
    load|=(buffer[122]<<16);
    load|=(buffer[123]<<24);
    exec =(buffer[116]<< 0);
    exec|=(buffer[117]<< 8);
    exec|=(buffer[118]<<16);
    exec|=(buffer[119]<<24);
    }

  /*** Open file */
  if ((bbs_file[4]=fopen(filename,"wb"))==NULL) return(FILE_ERROR);
  wimp_waitfor(10);
  port_txw(gmode?'G':'C');

  do
    {
    errors=0;
    do
      {
      errorflaga=FALSE;
      do
        {
        if ((firstchar=yreadline(HDRTIMEOUT))==CAN) firstchar=yreadline(4);
        }
      while(firstchar!=SOH && firstchar!=STX && firstchar!=EOT && firstchar!=CAN
            && firstchar!=CTIMEOUT);

      if (firstchar==CAN)
        {
        _fclose(&bbs_file[4]);
        return(CANCEL);
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
        size=(firstchar==SOH)?128:1024;
        sectcurr=yreadline(DATATIMEOUT);
        sectcomp=yreadline(DATATIMEOUT);

        if ((sectcurr+sectcomp)==255 && sectcurr==((sectnum+1)%256))
          {
          int t=0;

          for(j=0;j<size && t==0;j++)
            {
            if ((buffer[j]=yreadline(gmode?(DATATIMEOUT*4):DATATIMEOUT))==CTIMEOUT) t=1;
            }

          if (t==0)
            {
            checksum=calcrc(buffer,size);
            if (yreadline(gmode?(DATATIMEOUT*4):DATATIMEOUT)!=(checksum>>8  )) t=2;
            if (yreadline(gmode?(DATATIMEOUT*4):DATATIMEOUT)!=(checksum&0xff)) t=2;
            }

          if (t==0)
            {
            errors=0; sectnum++;

            if (ypdata(buffer,(left<size)?left:size)==EOF)
              {
              _fclose(&bbs_file[4]);
              return(DISK_FULL);
              }
            left-=(left<size)?left:size;

            /* ACK it */
            if (gmode==0) port_txw(ACK);
            pos+=size;
#ifdef SEVEN
            write_iconint(zmodem_handle,1,(pos>fsize)?fsize:pos);
            write_iconint(zmodem_handle,3,size);
            write_icon(zmodem_handle,4,msgs_lookup("7_rack"));
#endif
            }
          else
            {
            if (gmode)
              {
#ifdef SEVEN
              write_icon(zmodem_handle,4,msgs_lookup("7_gabrt"));
#endif
              port_txw(CAN); port_txw(CAN);
              port_txw(CAN); port_txw(CAN);
              port_txw(CAN); port_txw(CAN);
              port_txw(CAN); port_txw(CAN);
              wimp_waitfor(200);
              port_rxclear();

              _fclose(&bbs_file[4]);
              return(ERRORS);
              }
            else
              {
#ifdef SEVEN
              write_icon(zmodem_handle,4,msgs_lookup("7_rnake"));
#endif
              errorflaga=TRUE;
              }
            }
          }
        else
          {
          if ((sectcurr+sectcomp)==255)
            {
            /*** Duplicate block? */
            if (sectcurr==(sectnum&0xff))
              {
              while(yreadline(DATATIMEOUT)!=CTIMEOUT); port_txw(ACK);
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
          else
            {
#ifdef SEVEN
            write_icon(zmodem_handle,4,msgs_lookup("7_rnakg"));
#endif
            errorflaga=TRUE;
            }
          }

        if (errorflaga==TRUE)
          {
          errors++;
          while(yreadline(DATATIMEOUT)!=CTIMEOUT);
          port_txw(NAK);
          }
        }
      }
    while(firstchar!=EOT && errors<ERRORMAX);
    }
  while(firstchar!=EOT && errors<ERRORMAX);

  _fclose(&bbs_file[4]);

  if (firstchar==EOT)
    {
    port_txw(ACK);

    /*** Set file attribs? */
    if (useattr)
      {
      os_filestr attr;

      attr.action=2;
      attr.name=filename;
      attr.loadaddr=load;
      os_file(&attr);

      attr.action=3;
      attr.name=filename;
      attr.execaddr=exec;
      os_file(&attr);
      }

    return(DONE);
    }
  else
    {
    return(ERRORS);
    }
  }

int ymodemtx(char *batchfilename)
  {
  char realname[256]; int status;
#ifdef SEVEN
  batch_entry *file2send;
#endif
  files_done=0; gmode=0;
  buffer=message_buffer; alias=message_buffer+1024;

#ifndef SEVEN
  if ((bbs_file[3]=fopen(batchfilename,"rb"))==NULL)
    {
    return(FILE_ERROR);
    }

  do
    {
    mygets(realname,bbs_file[3]);
    mygets(alias,bbs_file[3]);
    if (*realname)
      {
      if ((status=ymodemtxf(realname))==DONE) files_done++;
      }
    }
  while(status==DONE && feof(bbs_file[3])==0 && *realname!=0);

  if (feof(bbs_file[3])) { alias[0]=0; ymodemtxf(""); }

  _fclose(&bbs_file[3]);
#else
  batch_next(1);

  do
    {
    if ((file2send=batch_next(0))!=NULL)
      {
      strcpy(realname,file2send->filename);
      strcpy(alias,file2send->alias);
      if ((status=ymodemtxf(realname))==DONE) files_done++;
      }
    }
  while(status==DONE && file2send!=NULL);
  *alias=0; ymodemtxf("");
#endif

  if (files_done==0)
    {
    if (status!=CANCEL) return(FILE_ERROR); else return(CANCEL);
    }

  return(DONE);
  }

int ymodemtxf(char *file)
  {
  int a,j,sectnum=1,attempts=0,checksum,length=0,size,count=0,pos=0,
      load,exec,useattr=0;

#ifdef SEVEN
  ymodem_abort=0; bbs_file[4]=0;
  if (setjmp(killy)==1)
    {
    _fclose(&bbs_file[4]);
    port_txw(CAN); port_txw(CAN);
    return(CANCEL);
    }
#endif

  flag=0;

  if (*file)
    {
    os_filestr attr;

    /* Get file attributes */
    attr.action=17;
    attr.name=file;
    if (os_file(&attr)==NULL)
      {
      useattr++;
      load=attr.loadaddr;
      exec=attr.execaddr;
      length=attr.start;
      }

    if((bbs_file[4]=fopen(file,"rb"))==0)
      {
      return(FILE_ERROR);
      }

    if (useattr==0)
      {
      fseek(bbs_file[4],0,SEEK_END);
      length=(int)ftell(bbs_file[4]);
      fseek(bbs_file[4],0,SEEK_SET);
      }
    }

  port_rxclear();
#ifdef SEVEN
  if (*file)
    {
    write_icon(zmodem_handle,0,strippath(file));
    write_iconint(zmodem_handle,2,length);
    }
  else
    {
    write_icon(zmodem_handle,0,"<EndOfBatch>");
    write_iconint(zmodem_handle,2,0);
    }
  write_icon(zmodem_handle,4,msgs_lookup("7_xmsync"));
#endif

  /*** First of all, make a header block */
  memset(buffer,0,128); /* Clean block */
  if (*file)
    {
    char *pt=buffer;

    pt+=(sprintf(pt,"%s",alias)+1);
    sprintf(pt,"%d 0 0 0",length);

    /* Attributes? */
    if (useattr)
      {
      buffer[124]=3; /* Both load & exec */
      buffer[120]=((load>>0 )&0xff);
      buffer[121]=((load>>8 )&0xff);
      buffer[122]=((load>>16)&0xff);
      buffer[123]=((load>>24)&0xff);
      buffer[116]=((exec>>0 )&0xff);
      buffer[117]=((exec>>8 )&0xff);
      buffer[118]=((exec>>16)&0xff);
      buffer[119]=((exec>>24)&0xff);
      }
    }
  checksum=calcrc(buffer,128);

  /*** Now send it until we get an ACK */
  do
    {
    attempts++;
    if ((j=yreadline(HDRTIMEOUT))==CAN)
      {
      if (yreadline(4)==CAN)
        {
        _fclose(&bbs_file[4]);
        return(CANCEL);
        }
      }

    if (j=='C' || j=='G')
      {
      port_txw(SOH); port_txw(0); port_txw(255);
      for(a=0;a<128;a++) port_txw(buffer[a]);
      port_txw(checksum>>8);
      port_txw(checksum&0xff);
      port_rxclear();

      /* Don't wait for ACK in Ymodem-G mode */
      if ((gmode=j)=='G') break;
      }
    }
  while(attempts<10 && j!=ACK);

  if (attempts>=10)
    {
    if (*file)
      {
      _fclose(&bbs_file[4]);
      return(TIMEOUT);
      }
    }

  if (*file==0) return(END_BATCH);

  /*** Wait for prompt C or G */
  attempts=0;
  do
    {
    attempts++;
    if ((j=yreadline(HDRTIMEOUT))==CAN) j=yreadline(4);
    }
  while(j!=gmode && j!=CAN && attempts<10);
  if (attempts>=10)
    {
    _fclose(&bbs_file[4]);
    return(*file==0?END_BATCH:TIMEOUT);
    }

  if (j==CAN)
    {
    _fclose(&bbs_file[4]);
    return(*file==0?END_BATCH:CANCEL);
    }

  /*** Send file */
  do
    {
    attempts=0;

    /*** Get data from file */
    size=(length>896)?1024:128;
    ygdata(buffer,size);

    /*** Send data until it's got OK */
    do
      {
      port_txw(size==128?SOH:STX);
      port_txw(sectnum & 0xff);
      port_txw((sectnum & 0xff)^0xff);
      for(j=0;j<size;j++) port_txw(buffer[j]);
      checksum=calcrc(buffer,size);
      port_txw(checksum>>8);
      port_txw(checksum & 0xff);
      port_rxclear();
      attempts++;
      if (gmode=='C')
        {
        count=0;
        do
          {
          if ((j=yreadline(HDRTIMEOUT))==CAN) j=yreadline(4);
          count++;
          if (sectnum==1 && j=='C') j=NAK;
          }
        while(j!=ACK && j!=CAN && j!=NAK && count<30);
        }
      else j=ACK;

#ifdef SEVEN
      if (j==NAK) write_icon(zmodem_handle,4,msgs_lookup("7_naked"));
#endif
      }
    while(j!=ACK && j!=CAN && attempts<MAXRETRY);
    if (j==CAN)
      {
      _fclose(&bbs_file[4]);
      return(CANCEL);
      }
    if (attempts==MAXRETRY)
      {
      _fclose(&bbs_file[4]);
      return(ERRORS);
      }

    pos+=size;
    length-=size; sectnum++;

#ifdef SEVEN
    write_iconint(zmodem_handle,1,pos);
    write_iconint(zmodem_handle,3,size);
    write_icon(zmodem_handle,4,msgs_lookup("7_acked"));
#endif
    }
  while(flag==0 && length>0);

  attempts=0;
  do
    {
    port_txw(EOT);
    port_rxclear();
    attempts++;
    j=yreadline(HDRTIMEOUT);
    }
  while(j!=ACK && j!=CAN && attempts!=MAXRETRY);

  _fclose(&bbs_file[4]);
#ifdef SEVEN
  batch_remove(file);
#endif
  return(DONE);
  }


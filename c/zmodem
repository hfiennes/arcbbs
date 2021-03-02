/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 31-August-1988                              <]
                  [>                                             <]
Module name       [> Zmodem file transfer                        <]
Current version   [> 01.99                                       <]
Version date      [> 30-November-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

/* Main includes */
#include "include.h"

/* System includes */
#ifndef SEVEN
#include "servmess.h"
#include "config.h"
#else
#include "batch.h"
#endif

#include "port.h"
#include "crc.h"
#include "zcrc.h"
#include "zmodem.h"

extern int  getzrxinit(void),wcs(void),wctxpn(void),zrbhdr32(char*),
            zsendfile(char*,int),zsendfdata(void),getinsync(void),
            wcreceive(void),tryz(void),rzfiles(void),zrdata(char*,int),
            rzfile(void),procheader(char*),putsec(char*,int),readock(int),
            zgethdr(char*),zrdat32(char*,int),
            zrbhdr(char*),zrbhdr(char*),zgethex(void),zdlread(void),
            noxrd7(void),zrhhdr(char*),getfree(void),timeout(void);
extern void wcsend(void),saybibi(void),ackbibi(void),canit(void),
            zmputs(char*),flushmo(void),showerror(void),zmodem_tx(void),
            throughput(int,int),zsbhdr(int,char*),zmodem_rx(void),
            zshhdr(int,char*),zsdata(char*,int,int),zsendline(int),
            stohdr(int),zputhex(int),window_poll(void),wimp_waitfor(int),
            monputs(char*),syslogz(char*),zsbh32(char*,int),
            zsda32(char*,int,int),mygets(char*,FILE*),_fclose(FILE**),
            write_icon(wimp_w,wimp_i,char*),write_iconint(wimp_w,wimp_i,int);

#ifdef SEVEN
#include "term.h"
extern terminal_configuration current;
FILE *bbs_file[10];
int files_done;
char message_buffer[3500];
extern wimp_w zmodem_handle;
#else
extern FILE *bbs_file[10];
extern int files_done;
extern char *message_buffer;
#endif

extern int  portnumber,baudrate;

#ifdef SEVEN
#define DONE    0
#define CANCEL -1
#endif

#undef ERROR
#undef TIMEOUT

#define OK    0
#define FALSE 0
#define TRUE  1
#define ERROR   (-1)
#define TIMEOUT (-2)
#define RCDO    (-3)

/* Ward Christensen / CP/M parameters */
#define SOH 1
#define STX 2
#define EOT 4
#define ENQ 5
#define ACK 6
#define NAK 025
#define XON    ('Q'&037)
#define XOFF   ('S'&037)
#define CAN    ('X'&037)
#define CPMEOF ('Z'&037)

#define WANTFCS32 TRUE           /* Default to using 32 bit FCS */
#define RXBINARY FALSE           /* Force binary mode uploads? */
#define RXASCII FALSE            /* Force ASCII mode uploads? */
#define LZCONV  0                /* Default ZMODEM file conversion mode */
#define LZMANAG 0                /* Default ZMODEM file management mode */
#define LZTRANS 0                /* Default ZMODEM file transport mode */
#define PATHLEN 128              /* What's the max legal MS-DOS path size? */
#define BUFLEN 64               /* Size of our scratchpad text buffer */
#define KSIZE 1024               /* Max allowable packet size */
#define S_IWRITE 0000200

#define sendline port_txw       /* No difference until 7-bit ZMODEM */
#define xsendline port_txw

int Filcnt;             /* Number of files opened for transmission */
int Errcnt;             /* Number of files unreadable */
int Noroom;             /* Flags 'insufficient disk space' errors */
int Rxbuflen;           /* Receiver's max buffer length */
int Tframlen = 0;       /* Override for tx frame length */
char Zconv;             /* ZMODEM file conversion request */
char Zmanag;            /* ZMODEM file management request */
char Ztrans;            /* ZMODEM file transport request */
int Rxtimeout;          /* Tenths of seconds to wait for something */
int Eofseen;            /* indicates cpm eof (^Z) has been received */
int Thisbinary;         /* Current file is to be received in binary mode */
int Tryzhdrtype=ZRINIT; /* Header type to send corresponding to Last rx close */
int Strtpos;            /* Starting byte position of download */
char alias[30];         /* Name to xmit instead of real file name */

static int useattr,attr_load,attr_exec,attr_length;

char *Txbuf;            /* Transmit buffer */
char *Secbuf;           /* Receive buffer */
char Msgbuf[BUFLEN];    /* Scratchpad buffer for printing messages */
char Filename[PATHLEN]; /* Name of the file being up/downloaded */
int zmodem_abort;

#ifdef SEVEN
jmp_buf killz;
#endif

void zmodem_reset()
  {
  Tframlen=0;
  Tryzhdrtype=ZRINIT;
  Txbuf=Secbuf=message_buffer;
  }

#ifdef SEVEN
char rxpath[128];

void zmodem_msgproc(wimp_eventstr *e,void *handle)
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
      zmodem_abort=1;
      break;
      }
    }
  }
#endif

int zmodemtx(char *batchfilename)
  {
  files_done=Filcnt=Errcnt=Noroom=zmodem_abort=0;
  batchfilename=batchfilename;

  zmodem_reset(); /* For overlay */

#ifndef SEVEN
  if ((bbs_file[3]=fopen(batchfilename,"r"))==NULL)
    {
    return(FILE_ERROR);
    }
#else
  if (setjmp(killz)==1)
    {
    _fclose(&bbs_file[3]);
    _fclose(&bbs_file[4]);
    return(CANCEL);
    }
#endif

  Rxtimeout = 614400/baudrate;             /* Enough time to send 6K of data */
    if (Rxtimeout < 600) Rxtimeout = 600;  /* or 1 minute, whichever is more */
  zmputs("rz\r");
  stohdr(0);
  zshhdr(ZRQINIT, Txhdr);
  if (getzrxinit()==ERROR)
    {
    zmputs("Download cancelled or timed out\n");
#ifndef SEVEN
    _fclose(&bbs_file[3]);
    _fclose(&bbs_file[4]);
#endif
    return(CANCEL);
    }
  wcsend();

#ifndef SEVEN
  _fclose(&bbs_file[3]);
  _fclose(&bbs_file[4]);
#endif

  /* Report errors to user if neccessary */
  if (Errcnt || Noroom)
    {
    zmputs("One or more files skipped due to errors\r");
    wimp_waitfor(300);
    }
  return(DONE);
  }

int zmodemrx(char *batchfilename)
  {
  files_done=Filcnt=Errcnt=Noroom=zmodem_abort=0;

  zmodem_reset(); /* For overlay */

#ifndef SEVEN
  if ((bbs_file[3]=fopen(batchfilename,"w"))==NULL)
    {
    return(FILE_ERROR);
    }
#else
  strcpy(rxpath,batchfilename);
#endif

  Rxtimeout = 100;          /* 10 seconds... receiver should stay busy */
  if (wcreceive()==ERROR)
    {
    zmputs("Upload cancelled or timed out\n");
#ifndef SEVEN
    _fclose(&bbs_file[3]);
#endif
    _fclose(&bbs_file[4]);
    return(CANCEL);
    }

#ifndef SEVEN
  _fclose(&bbs_file[3]);
#endif
  _fclose(&bbs_file[4]);

  /* Report errors to user if neccessary */
  if (Errcnt || Noroom)
    {
    wimp_waitfor(300);
    zmputs("One or more files skipped due to errors\r");
    if (Noroom)
      {
      sprintf(Msgbuf, "Insufficient disk space; only %d bytes available\r",
        getfree());
      zmputs(Msgbuf);
      }
    }

  return(DONE);
  }

/* Get the receiver's init parameters */
int getzrxinit()
  {
  int n;
  int rxflags;

  for (n=10; --n>=0; )
    {
    switch (zgethdr(Rxhdr))
      {
      case ZCHALLENGE:        /* Echo receiver's challenge number */
        stohdr(Rxpos);
        zshhdr(ZACK, Txhdr);
        continue;
      case ZCOMMAND:          /* They didn't see our ZRQINIT */
        stohdr(0);
        zshhdr(ZRQINIT, Txhdr);
        continue;
      case ZRINIT:
        rxflags = Rxhdr[ZF0];
#ifdef SEVEN
        Txfcs32 = (current.zmodem_32bit && (rxflags & CANFC32)!=0);
#else
        Txfcs32 = (WANTFCS32!=0 && (rxflags & CANFC32)!=0);
#endif
        Rxbuflen = ((unsigned int)Rxhdr[ZP1]<<8) | Rxhdr[ZP0];
        /* Override to force shorter frame length */
        if (Rxbuflen && (Rxbuflen>Tframlen) && (Tframlen>=32))
          Rxbuflen = Tframlen;
        if ( !Rxbuflen && (Tframlen>=32) && (Tframlen<=1024))
          Rxbuflen = Tframlen;
        return OK;
      case ZCAN:
      case RCDO:
      case TIMEOUT:
        return ERROR;
      case ZRQINIT:
        if (Rxhdr[ZF0] == ZCOMMAND) continue;
      default:
        zshhdr(ZNAK, Txhdr);
        continue;
      }
    }
  return ERROR;
  }

/* Attempts to send each file named in the Opus-to-Oz control file */
void wcsend()
  {
  /* Read and process each name remaining in control file */
#ifndef SEVEN
  do
    {
    mygets(Filename,bbs_file[3]);
    if (Filename[0])
      {
      mygets(alias,bbs_file[3]);
      if (wcs()==ERROR) return;      /* else try to send specified file */
      }
    }
  while(Filename[0] && !feof(bbs_file[3]));
#else
  batch_entry *file2send;

  batch_next(1);

  while((file2send=batch_next(0))!=NULL)
    {
    strcpy(Filename,file2send->filename);
    strcpy(alias,file2send->alias);
    if (wcs()==ERROR) return;
    }
#endif

  /* If we never got going properly, CANcel the receiver; */
  /* otherwise, end the download session gracefully */
  if (Filcnt) saybibi();
  else canit();
  }

/* Send the file named in Filename */
int wcs()
  {
  if ((bbs_file[4]=fopen(Filename, "r"))==NULL)
    {
    ++Errcnt;
    sprintf(Msgbuf, "ERROR opening %s\n",Filename);
    monputs(Msgbuf);
    syslogz(Msgbuf);
    return OK;      /* pass over it, there may be others */
    }
  else
    {
    os_filestr attr;

    /* Find file attributes */
    attr.action=17;
    attr.name=Filename;
    if (os_file(&attr)==NULL)
      {
      useattr=1;
      attr_load=attr.loadaddr;
      attr_exec=attr.execaddr;
      attr_length=attr.start;
      }
    else useattr=0;
    }

  ++Filcnt;

  switch (wctxpn())
    {
    case ERROR:
      ++Errcnt;
      return ERROR;
    case OK:
      _fclose(&bbs_file[4]);
    }
  return OK;
  }

/* Generate and transmit pathname block consisting of
 * pathname, file length, and time last modified
 */
int wctxpn()
  {
  char *q,*p;
  int length;

  /* Display outbound filename */
  sprintf(Msgbuf, "Sending %s", Filename);
  monputs(Msgbuf);
  memset(Txbuf,0,KSIZE);

  if (useattr==0)
    {
    fseek(bbs_file[4],0,SEEK_END);
    length=ftell(bbs_file[4]);
    fseek(bbs_file[4],0,SEEK_SET);
    }
  else
    {
    length=attr_length;

    /* Put attributes into packet */
    Txbuf[124]=3; /* Load and exec supplied */
    Txbuf[120]=((attr_load>>0 )&0xff);
    Txbuf[121]=((attr_load>>8 )&0xff);
    Txbuf[122]=((attr_load>>16)&0xff);
    Txbuf[123]=((attr_load>>24)&0xff);
    Txbuf[116]=((attr_exec>>0 )&0xff);
    Txbuf[117]=((attr_exec>>8 )&0xff);
    Txbuf[118]=((attr_exec>>16)&0xff);
    Txbuf[119]=((attr_exec>>24)&0xff);
    }

  /* Store filesize, time last modified, and file mode in header packet */
  p=q=Txbuf;
  strcpy(p,alias); p+=(strlen(alias)+1);
  sprintf(p,"%d 0 0",length);

#ifdef SEVEN
  write_icon(zmodem_handle,0,alias);
  write_iconint(zmodem_handle,2,length);
  write_iconint(zmodem_handle,1,0);
  write_iconint(zmodem_handle,3,0);
#endif

  /* Transmit the filename block and begin the download */
  throughput(0,0);
  
  if (useattr==0)
    {
    /* This line trims the packet to the exact length of fname */
    return zsendfile(Txbuf, 2+strlen(Txbuf)+strlen(p));
    }
  else
    {
    /* This sends the whole 128 bytes to preserve ftypes etc */
    return zsendfile(Txbuf, 128);
    }
  }

/* Send ZFILE frame and begin sending ZDATA frame */
int zsendfile(char *buf,int blen)
  {
  int c;

  for (;;)
    {
    Txhdr[ZF0] = LZCONV;    /* Default file conversion mode */
    Txhdr[ZF1] = LZMANAG;   /* Default file management mode */
    Txhdr[ZF2] = LZTRANS;   /* Default file transport mode */
    Txhdr[ZF3] = 0;
    zsbhdr(ZFILE, Txhdr);
    zsdata(buf, blen, ZCRCW);
again:
    switch (c = zgethdr(Rxhdr))
      {
      case ZRINIT:
        goto again;
      case ZCAN:
      case ZCRC:
      case RCDO:
      case TIMEOUT:
      case ZABORT:
      case ZFIN:
        return ERROR;
      case ZSKIP:
        monputs("SKIP command received");
        _fclose(&bbs_file[4]);
        return c;
      case ZRPOS:
        fseek(bbs_file[4], Rxpos, SEEK_SET);  /* Verify that seek past end = to EOF */
        Strtpos = Txpos = Rxpos;
        port_rxclear();
        return zsendfdata();
      }
    }
  }

/* Send the data in the file */
int zsendfdata()
  {
  int c, e;
  int newcnt, blklen, maxblklen, goodblks, goodneeded = 1;
  char *p;

  maxblklen = KSIZE;
  if (Rxbuflen && maxblklen>Rxbuflen) maxblklen = Rxbuflen;
  blklen = (baudrate<1200) ? 256 : KSIZE;
  if (blklen>maxblklen) blklen = maxblklen;

somemore:
  if (port_rxbuffer())
    {
waitack:
    switch (c = getinsync())
      {
      default:
        monputs("Transfer cancelled");
        _fclose(&bbs_file[4]);
        return ERROR;
      case ZSKIP:
        monputs("SKIP command received");
        _fclose(&bbs_file[4]);
        return c;
      case ZACK:
        break;
      case ZRPOS:
        blklen = (blklen>>2 > 64) ? blklen>>2 : 64;
        goodblks = 0;
        goodneeded = (goodneeded<<1) | 1;
        break;
      case ZRINIT:
        throughput(1,Txpos-Strtpos);
        files_done++;
#ifdef SEVEN
        /* Remove from batch */
        batch_remove(Filename);
#endif
        /* fprintf(Logfile, "Sent %s %d\n", Filename, Txpos-Strtpos); */        return OK;
      }

    while (port_rxbuffer())
      {
      switch (readock(1))
        {
        case CAN:
        case RCDO:
        case ZPAD:
          goto waitack;
        }
      }
    }

  newcnt = Rxbuflen;
  stohdr(Txpos);
  zsbhdr(ZDATA, Txhdr);

  do
    {
    p=Txbuf;
    c=fread(Txbuf,1,blklen,bbs_file[4]);
    if (c < blklen) e = ZCRCE;
    else if (Rxbuflen && (newcnt -= c) <= 0) e = ZCRCW;
    else e = ZCRCG;
    zsdata(Txbuf, c, e);

#ifdef SEVEN
    write_iconint(zmodem_handle,1,Txpos);
    write_iconint(zmodem_handle,3,c);
#endif
    /*sprintf(Msgbuf, "Sent %4d bytes at position %d", c, Txpos);
    monputs(Msgbuf);*/
    Txpos += c;
    if (blklen<maxblklen && ++goodblks>goodneeded)
      {
      blklen = (blklen<<1 < maxblklen) ? blklen<<1 : maxblklen;
      goodblks = 0;
      }
    if (e == ZCRCW) goto waitack;

    while (port_rxbuffer())
      {
      switch (readock(1))
        {
        case CAN:
        case RCDO:
        case ZPAD:
          /* Interruption detected; stop sending and process complaint */
          port_txclear();
          zsdata(Txbuf, 0, ZCRCE);
          goto waitack;
        }
      }
    }
  while (e == ZCRCG);

  for (;;)
    {
    stohdr(Txpos);
    zsbhdr(ZEOF, Txhdr);
    switch (getinsync())
      {
      case ZACK:
        continue;
      case ZRPOS:
        goto somemore;
      case ZRINIT:
        throughput(1,Txpos-Strtpos);
        files_done++;
#ifdef SEVEN
        batch_remove(Filename);
#endif
        /* fprintf(Logfile, "Sent %s %d\n", Filename, Txpos-Strtpos); */
        return OK;
      case ZSKIP:
        monputs("SKIP command received\n");
        _fclose(&bbs_file[4]);
        return c;
      default:
        monputs("Transfer cancelled\n");
        _fclose(&bbs_file[4]);
        return ERROR;
      }
    }
  }

/* Respond to receiver's complaint, get back in sync with receiver */
int getinsync()
  {
  register int c;

  for (;;)
    {
    c = zgethdr(Rxhdr);
    port_rxclear();
    switch (c)
      {
      case ZCAN:
      case ZABORT:
      case ZFIN:
      case RCDO:
      case TIMEOUT:
        return ERROR;
      case ZRPOS:
        clearerr(bbs_file[4]);   /* In case file EOF seen */
        fseek(bbs_file[4], Rxpos, SEEK_SET);
        Txpos = Rxpos;
        sprintf(Msgbuf, "Error reported; resending from byte %d", Txpos);
        monputs(Msgbuf);
        return c;
      case ZSKIP:
        monputs("SKIP command received\n");
      case ZRINIT:
        _fclose(&bbs_file[4]);
      case ZACK:
        return c;
      default:
        zsbhdr(ZNAK, Txhdr);
        continue;
      }
    }
  }

/* Say "bibi" to the receiver, try to do it cleanly */
void saybibi()
  {
  for (;;)
    {
    stohdr(0);
    zsbhdr(ZFIN, Txhdr);
    switch (zgethdr(Rxhdr))
      {
      case ZFIN:
        sendline('O'); sendline('O'); flushmo();
      case ZCAN:
      case RCDO:
      case TIMEOUT:
        return;
      }
    }
  }

/* Receives one batch of files */
int wcreceive()
  {
  switch (tryz())
    {
    case ZCOMPL:
      return OK;
    case ZFILE:
      if (rzfiles()==OK) return OK;
    }
  canit();
  return ERROR;
  }

/* Initialize for Zmodem receive attempt, try to activate Zmodem sender
 * Handles ZSINIT, ZFREECNT, and ZCOMMAND frames
 * Return ZFILE if Zmodem filename received,
 *   ZCOMPL if transaction finished,  else ERROR
 */
int tryz()
  {
  register int n;
  int errors = 0;

  for (n=10; --n>=0; ) {
    /* Set buffer length (0=unlimited, don't wait) and capability flags */
    stohdr(0);
#ifdef SEVEN
    Txhdr[ZF0] = (current.zmodem_32bit?CANFC32:0)|CANFDX|CANOVIO;
#else
    Txhdr[ZF0] = CANFC32|CANFDX|CANOVIO;
#endif
    zshhdr(Tryzhdrtype, Txhdr);
again:
    switch (zgethdr(Rxhdr))
      {
      case ZFILE:
        Zconv = Rxhdr[ZF0];
        Zmanag = Rxhdr[ZF1];
        Ztrans = Rxhdr[ZF2];
        Tryzhdrtype = ZRINIT;
        if (zrdata(Secbuf, KSIZE) == GOTCRCW) return ZFILE;
        zshhdr(ZNAK, Txhdr);
        goto again;
      case ZSINIT:
        if (zrdata(Attn, ZATTNLEN) == GOTCRCW)
          zshhdr(ZACK, Txhdr);
        else
          zshhdr(ZNAK, Txhdr);
        goto again;
      case ZFREECNT:
        stohdr(getfree());
        zshhdr(ZACK, Txhdr);
        goto again;
      case ZCOMMAND:   /* Paranoia is good for you... */
        if (zrdata(Secbuf, KSIZE) == GOTCRCW)
          {
          sprintf(Msgbuf, "Ignoring command: %s\r\n", Secbuf);
          monputs(Msgbuf);   /* Ignore and log all uploaded commands */
          syslogz(Msgbuf);
          stohdr(0);        /* and return successful completion code */
          do
            {
            zshhdr(ZCOMPL, Txhdr);
            }
          while (++errors<10 && zgethdr(Rxhdr) != ZFIN);
          ackbibi();
          return ZCOMPL;
          }
        else
          zshhdr(ZNAK, Txhdr);
        goto again;
      case ZCOMPL:
        goto again;
      case ZFIN:
        ackbibi();
        return ZCOMPL;
      case ZCAN:
      case RCDO:
        return ERROR;
      }
    }
  return ERROR;
  }

/* Receive a batch of files using ZMODEM protocol */
int rzfiles()
  {
  register int c;

  for (;;)
    {
    switch (c = rzfile())
      {
      case ZEOF:
      case ZSKIP:
        switch (tryz())
          {
          case ZCOMPL:
            return OK;
          default:
            return ERROR;
          case ZFILE:
            break;
          }
        break;
      default:
        _fclose(&bbs_file[4]);
        /*remove(Filename);*/
        return c;
      }
    }
  }

/* Receive one file; assumes file name frame is preloaded in Secbuf */
int rzfile()
  {
  register int c, n;
  int rxbytes;

  Eofseen=FALSE;
  if ((c=procheader(Secbuf)) == ERROR)
    {
    return (Tryzhdrtype = ZSKIP);
    }

  n = 10; rxbytes = c; /* For resume */

  for (;;)
    {
    stohdr(rxbytes);
    zshhdr(ZRPOS, Txhdr);
nxthdr:
    switch (c = zgethdr(Rxhdr))
      {
      default:
        return ERROR;
      case ZNAK:
      case TIMEOUT:
        if ( --n < 0) return ERROR;
        sprintf(Msgbuf, "at byte %d; %d retries", rxbytes, n);
        monputs(Msgbuf);
#ifdef SEVEN
        write_iconint(zmodem_handle,1,rxbytes);
#endif
        continue;
      case ZFILE:     /* Sender didn't see our ZRPOS yet */
        zrdata(Secbuf, KSIZE);
        continue;
      case ZEOF:
        if (Rxpos != rxbytes)
          {
          sprintf(Msgbuf, "Bad EOF; here=%d, there=%d\n", rxbytes, Rxpos);
          monputs(Msgbuf);
          continue;
          }
        throughput(2,rxbytes);
        _fclose(&bbs_file[4]);

        /* Set correct file attributes */
        if (useattr)
          {
          os_filestr attr;

          attr.action=2;
          attr.name=Filename;
          attr.loadaddr=attr_load;
          os_file(&attr);

          attr.action=3;
          attr.name=Filename;
          attr.execaddr=attr_exec;
          os_file(&attr);
          }

#ifndef SEVEN
        fprintf(bbs_file[3], "%s\012",alias);
#endif
        files_done++;
        return c;
      case ERROR:     /* Too much garbage in header search error */
        if ( --n < 0) return ERROR;
        sprintf(Msgbuf, "at byte %d; %d retries\n", rxbytes, n);
        monputs(Msgbuf);
        zmputs(Attn);
        continue;
      case ZDATA:
        if (Rxpos != rxbytes)
          {
          if ( --n < 0) return ERROR;
          sprintf(Msgbuf, "Data at bad position; here=%d, there=%d\n",
                  rxbytes, Rxpos);
          monputs(Msgbuf);
          zmputs(Attn);
          continue;
          }
moredata:
        switch (c = zrdata(Secbuf, KSIZE))
          {
          case ZCAN:
          case RCDO:
            monputs("Transfer cancelled");
            return ERROR;
          case ERROR:     /* CRC error */
            if ( --n < 0) return ERROR;
          sprintf(Msgbuf, "at byte %d; %d retries\n", rxbytes, n);
          monputs(Msgbuf);
          zmputs(Attn);
          continue;
        case TIMEOUT:
          if ( --n < 0) return ERROR;
          sprintf(Msgbuf, "at byte %d; %d retries\n", rxbytes, n);
          monputs(Msgbuf);
          continue;
        case GOTCRCW:
          n = 10;
          if (putsec(Secbuf, Rxcount) == ERROR) return ERROR;
          rxbytes += Rxcount;
        /*sprintf(Msgbuf, "Received %4d bytes; %d so far\r", Rxcount,
                  rxbytes);
          monputs(Msgbuf);*/
#ifdef SEVEN
          write_iconint(zmodem_handle,1,rxbytes);
          write_iconint(zmodem_handle,3,Rxcount);
#endif
          stohdr(rxbytes);
          zshhdr(ZACK, Txhdr);
          goto nxthdr;
        case GOTCRCQ:
          n = 10;
          if (putsec(Secbuf, Rxcount) == ERROR) return ERROR;
          rxbytes += Rxcount;
        /*sprintf(Msgbuf, "Received %4d bytes; %d so far\r", Rxcount,
                  rxbytes);
          monputs(Msgbuf);*/
#ifdef SEVEN
          write_iconint(zmodem_handle,1,rxbytes);
          write_iconint(zmodem_handle,3,Rxcount);
#endif
          stohdr(rxbytes);
          zshhdr(ZACK, Txhdr);
          goto moredata;
        case GOTCRCG:
          n = 10;
          if (putsec(Secbuf, Rxcount) == ERROR) return ERROR;
          rxbytes += Rxcount;
        /*sprintf(Msgbuf, "Received %4d bytes; %d so far\r", Rxcount,
                  rxbytes);
          monputs(Msgbuf);*/
#ifdef SEVEN
          write_iconint(zmodem_handle,1,rxbytes);
          write_iconint(zmodem_handle,3,Rxcount);
#endif
          goto moredata;
        case GOTCRCE:
          n = 10;
          if (putsec(Secbuf, Rxcount) == ERROR) return ERROR;
          rxbytes += Rxcount;
        /*sprintf(Msgbuf, "Received %4d bytes; %d so far\r", Rxcount,
                  rxbytes);
          monputs(Msgbuf);*/
#ifdef SEVEN
          write_iconint(zmodem_handle,1,rxbytes);
          write_iconint(zmodem_handle,3,Rxcount);
#endif
          goto nxthdr;
        }
      }
    }
  }

/* Process incoming file information header */
int procheader(char *name)
  {
  register char *p;
  char *openmode;
  int filesize,newpos=0;
#ifdef SEVEN
  int a;
#endif

  /* set default parameters and overrides */
  openmode = "wb";
#ifdef IFBIN
  Thisbinary = (!RXASCII) || RXBINARY;

  /* Process ZMODEM remote file management requests */
  if (!RXBINARY && Zconv == ZCNL) /* Remote ASCII override */
    Thisbinary = FALSE;
  if (Zconv == ZCBIN)     /* Remote Binary override */
    Thisbinary = TRUE;
#else
  Thisbinary=TRUE;
#endif

  /* Check file attribs */
  if (name[124]==3)
    {
    useattr=1;
    attr_load =(name[120]<< 0);
    attr_load|=(name[121]<< 8);
    attr_load|=(name[122]<<16);
    attr_load|=(name[123]<<24);
    attr_exec =(name[116]<< 0);
    attr_exec|=(name[117]<< 8);
    attr_exec|=(name[118]<<16);
    attr_exec|=(name[119]<<24);
    }
  else useattr=0;

  /* Extract and verify filesize, if given; reject if not at least 10K free */
  filesize = 0;
  p = name + 1 + strlen(name);
  if (*p) sscanf(p, "%d", &filesize);
  if (filesize+10240 > getfree())
    {
    sprintf(Msgbuf, "Insufficient disk space; need %d bytes, have %d\n",
      filesize+10240, getfree());
    monputs(Msgbuf);
    syslogz(Msgbuf);
    Noroom = TRUE;
    return ERROR;
    }

  /* Get and/or fix filename for uploaded file */
  p=name;
  while(*p && *p!=' ') p++;
  *p=0;
  strcpy(alias,name);

#ifndef SEVEN
  sprintf(Filename,"<ARCbbs$upload>.%02d/%04d",portnumber,files_done);
#else
  name[10]=0;
  for(a=0;a<10;a++)
    {
    if (name[a]=='.' || name[a]=='%' ||
        name[a]=='$' || name[a]=='@' ||
        name[a]=='&' || name[a]==':') name[a]='_';
    }
  sprintf(Filename,"%s%s",rxpath,name);
  write_icon(zmodem_handle,0,name);
  write_iconint(zmodem_handle,2,filesize);

  if ((bbs_file[4]=fopen(Filename,"rb"))!=NULL)
    {
    if (current.zmodem_resume)
      {
      fseek(bbs_file[4],0,SEEK_END);
      newpos=ftell(bbs_file[4]);
      openmode="rb+";
      }
    _fclose(&bbs_file[4]);
    }
#endif

  if ((bbs_file[4]=fopen(Filename, openmode)) == NULL)
    {
    sprintf(Msgbuf, "ERROR opening %s\n", Filename);
    monputs(Msgbuf);
    syslogz(Msgbuf);
    return ERROR;
    }

  fseek(bbs_file[4],newpos,SEEK_SET);

  if (filesize)
    sprintf(Msgbuf, "Receiving in %s mode",Thisbinary?"binary":"ASCII");
  else
    sprintf(Msgbuf, "Receiving in %s mode, unknown size",
            Thisbinary?"binary":"ASCII");
  monputs(Msgbuf);

  throughput(0,0);
  return(newpos);
  }

/* Writes the received file data to the output file.
 * If in ASCII mode, stops writing at first ^Z, and converts all
 *   solo CR's or LF's to CR/LF pairs.
 */
int putsec(char *buf,int n)
  {
  char *p;
  static char lastsent;

  if (1 /*Thisbinary*/)
    {
    for (p=buf; --n>=0; )
      if (fputc(*p++, bbs_file[4]) == EOF) goto diskfull;
    }
  else
    {
    if (Eofseen) return OK;
    for (p=buf; --n>=0; ++p )
      {
      if ( *p == CPMEOF)
        {
        Eofseen = TRUE;
        return OK;
        }
      else
        {
        if (*p==13 && lastsent==10)
          {
          lastsent=0;
          if (putc(10, bbs_file[4]) == EOF) goto diskfull;
          }
        if (*p==10 && lastsent==13)
          {
          lastsent=0;
          if (putc(10, bbs_file[4]) == EOF) goto diskfull;
          }
        if (*p!=10 && *p!=13)
          {
          if (putc((lastsent=*p), bbs_file[4]) == EOF) goto diskfull;
          }
        }
      }
    }
  return OK;

diskfull:
  sprintf(Msgbuf, "Error writing %s\n", Filename);
  monputs(Msgbuf);
  syslogz(Msgbuf);
  Noroom = TRUE;
  return ERROR;
  }

/* Ack a ZFIN packet, let byegones be byegones */
void ackbibi()
  {
  register int n;

  stohdr(0);
  for (n=4; --n;)
    {
    zshhdr(ZFIN, Txhdr);
    switch (readock(100))
      {
      case 'O':
        readock(1);    /* Discard 2nd 'O' */
      case TIMEOUT:
      case RCDO:
        return;
      }
    }
  }

/* send cancel string to get the other end to shut up */
void canit()
  {
  static char canistr[] = {
   24,24,24,24,24,24,24,24,24,24,8,8,8,8,8,8,8,8,8,8,0
  };

  zmputs(canistr);
  flushmo();
  }

/* Get a byte from the modem;
 * return TIMEOUT if no read within timeout tenths,
 * return RCDO if carrier lost
 */
/* Assembler coded now! */

/* Send a string to the modem, processing for \336 (sleep 1 sec)
 *   and \335 (break signal, ignored)
 */
void zmputs(char *s)
  {
  int c;

  while (*s)
    {
    switch (c = *s++)
      {
      case '\336':
        wimp_waitfor(100);
      case '\335':
        break;
      default:
        sendline(c);
      }
    }
  }

/* Write one character to the modem */
/*void xsendline(int c)
register int c;
  {
  int tenths, a,xonlimit;
  clock_t start=clock();

  tenths = 614400/baudrate;          / Enough time to send 6K of data /
  if (tenths < 600) tenths = 600; / or one minute, whichever is more /
  tenths=tenths*10;
  xonlimit = tenths/2;
  a=port_txbuffer();
  port_txw(c);

  for (;;)
    {
    if (port_txbuffer()>=a) return;
    window_poll();
    if ((clock()-start)>tenths)
      {
      monputs("Timeout while writing to modem; transfer aborted");
     /exit(1);/
      return;
      }
    if ((clock()-start)>=xonlimit)
      {
      port_xonoff(0);
      port_xonoff(1);
      xonlimit = 0xffffff;
      }
    }
  }
*/

/* Wait for modem to finish sending output buffer */
void flushmo()
  {
  int tenths, xonlimit;
  clock_t start=clock();

  tenths = 614400/baudrate;          /* Enough time to send 6K of data */
  if (tenths < 600) tenths = 600; /* or one minute, whichever is more */
  tenths=tenths*10;
  xonlimit = tenths/2;

  for (;;)
    {
    if (port_allsent()) return;
    window_poll();
    if ((clock()-start)>=tenths)
      {
      monputs("Timeout while writing to modem; transfer aborted");
      /*exit(1);*/
      }

    if ((clock()-start)>=xonlimit)
      {
#ifndef SEVEN
      port_xonxoff(0);
      port_xonxoff(1);
#endif
      xonlimit = 0xffffff;
      }
    }
  }

/* Print throughput message at end of transfer */
void throughput(int opt,int bytes)
  {
  static int started;
  int seconds;
  char *p;
  int cps;

  switch (opt)
    {
    case 0:
      started = clock();
      return;
    case 1:
      p = "Sent";
      break;
    case 2:
      p = "Received";
      break;
    }
  seconds = (clock()-started)/100;
  if (!seconds) seconds++;
  cps=(bytes*100)/seconds;

  sprintf(Msgbuf, "%s in %d:%02d, throughput %d.%02d CPS (%d%%)",
    p, seconds/60, seconds%60, cps/100, cps%100,
        (cps*10/baudrate));
  monputs(Msgbuf);
  }

#ifdef SEVEN
void monputs(char *a)
  {
  write_icon(zmodem_handle,4,a);
  }
#else
void monputs(char*a)
  {
  a=a;
  }
#endif

/* Write an error to the Opus system log, if one was specified */
void syslogz(char *s)
  {
  s=s;
  }

/* Figure out how many bytes are free on the drive we're uploading to */
int getfree()
  {
  return(100000000);
  }

/*
 *   Z M . C
 *    ZMODEM protocol primitives
 *    01-19-87  Chuck Forsberg Omen Technology Inc
 *
 * Entry point Functions:
 *      zsbhdr(type, hdr) send binary header
 *      zshhdr(type, hdr) send hex header
 *      zgethdr(hdr) receive header - binary or hex
 *      zsdata(buf, len, frameend) send data
 *      zrdata(buf, len) receive data
 *      stohdr(pos) store position data in Txhdr
 *      int rclhdr(hdr) recover position offset from header
 *
 * Minor changes by Rick Huebner for progress/error reporting, carrier drop
 *   detection, debug message logging
 */


static char frametypes[][11] = {
#ifdef SEVEN
        "abort",                /* -3 */
#else
        "Lost DCD",             /* -3 */
#endif
        "TIMEOUT",              /* -2 */
        "ERROR",                /* -1 */
#define FTOFFSET 3
        "ZRQINIT",
        "ZRINIT",
        "ZSINIT",
        "ZACK",
        "ZFILE",
        "ZSKIP",
        "ZNAK",
        "ZABORT",
        "ZFIN",
        "ZRPOS",
        "ZDATA",
        "ZEOF",
        "ZFERR",
        "ZCRC",
        "ZCHALLENGE",
        "ZCOMPL",
        "ZCAN",
        "ZFREECNT",
        "ZCOMMAND",
        "ZSTDERR",
        "xxxxx"
#define FRTYPES 22      /* Total number of frame types in this array */
                        /*  not including psuedo negative entries */
};

/* Send ZMODEM binary header hdr of type type */
/* Assembler coded
void zsbhdr(int type,char *hdr)
  {
  int n;
  unsigned short crc;

  xsendline(ZPAD); xsendline(ZDLE);

  if (Txfcs32) zsbh32(hdr, type);
  else
    {
    xsendline(ZBIN);
    zsendline(type);
    crc = updcrc(type, 0);
    for (n=4; --n >= 0;)
      {
      zsendline(*hdr);
      crc = updcrc(((unsigned short)(*hdr++)), crc);
      }
    crc = updcrc(((unsigned short)0),crc);
    crc = updcrc(((unsigned short)0),crc);
    zsendline(crc>>8);
    zsendline(crc);
    }
  if (type != ZDATA) flushmo();
  }
*/

/* Send ZMODEM binary header hdr of type type */
/* Assembler coded
void zsbh32(char *hdr,int type)
  {
  unsigned long crc;
  int n;

  port_txw(ZBIN32);
  zsendline(type);

  crc = 0xFFFFFFFF;
  crc = Z_32UpdateCRC (type, crc);

  for (n = 4; --n >= 0;)
    {
    zsendline(*hdr);
    crc = Z_32UpdateCRC (((unsigned short) (*hdr++)), crc);
    }

  crc = ~crc;
  for (n = 4; --n >= 0;)
    {
    zsendline(crc);
    crc >>= 8;
    }
  }
*/

/* Send ZMODEM HEX header hdr of type type */
void zshhdr(int type,char *hdr)
  {
  int n;
  register unsigned short crc;

  sendline(ZPAD); sendline(ZPAD); sendline(ZDLE); sendline(ZHEX);
  zputhex(type);

  crc = updcrc(type, 0);
  for (n=4; --n >= 0;)
    {
    zputhex(*hdr);
    crc = updcrc(((unsigned short)(*hdr++)), crc);
    }
  crc = updcrc(((unsigned short)0),crc);
  crc = updcrc(((unsigned short)0),crc);
  zputhex(crc>>8);
  zputhex(crc);

  /* Make it printable on remote machine */
  sendline('\r'); sendline('\n');
  /* Uncork the remote in case a fake XOFF has stopped data flow */
  if (type != ZFIN) sendline(021);

  flushmo();
  }

/*
 * Send binary array buf of length length, with ending ZDLE sequence frameend
 */
/* Assembler coded now */
/*
void zsdata(char *buf,int length,int frameend)
  {
  unsigned short crc;
            
#ifdef SEVEN
  if (zmodem_abort)
    {
    longjmp(killz,1);
    }
#endif

  if (Txfcs32) zsda32(buf, length, frameend);
  else
    {
    crc = 0;
    for (;--length >= 0;)
      {
      zsendline(*buf);
      crc = updcrc(((unsigned short)(*buf++)), crc);
      }
    xsendline(ZDLE);
    xsendline(frameend);
    crc = updcrc(frameend, crc);

    crc = updcrc(((unsigned short)0),crc);
    crc = updcrc(((unsigned short)0),crc);
    zsendline(crc>>8);
    zsendline(crc);
    }

  if (frameend == ZCRCW)
    {
    xsendline(XON);
    flushmo();
    }
  }

void zsda32(char *buf,int length,int frameend)
  {
  unsigned long crc;

  crc = 0xFFFFFFFF;
  for (; --length >= 0; ++buf)
    {
    zsendline(*buf);
    crc = Z_32UpdateCRC (((unsigned short) (*buf)), crc);
    }

  port_txw(ZDLE);
  port_txw(frameend);
  crc = Z_32UpdateCRC (frameend, crc);

  crc = ~crc;

  for (length = 4; --length >= 0;)
    {
    zsendline(crc);
    crc >>= 8;
    }
  }
*/

/*
 * Receive array buf of max length with ending ZDLE sequence
 *  and CRC.  Returns the ending character or error code.
 */
/* Assembler coded now */
/*
int zrdata(char *buf,int length)
  {
  int c;
  unsigned short crc;
  int d;

  if (Rxframeind == ZBIN32)
    return zrdat32(buf, length);

  crc = Rxcount = 0;
  for (;;)
    {
    if ((c = zdlread()) & ~0xFF)
      {
crcfoo:
      switch (c)
        {
        case GOTCRCE:
        case GOTCRCG:
        case GOTCRCQ:
        case GOTCRCW:
          crc = updcrc(((d=c)&0xFF), crc);
          if ((c = zdlread()) & ~0xFF) goto crcfoo;
          crc = updcrc(c, crc);
          if ((c = zdlread()) & ~0xFF) goto crcfoo;
          crc = updcrc(c, crc);
          if (crc & 0xFFFF)
            {
            monputs("Bad data packet CRC");
            return ERROR;
            }
          return d;
        case GOTCAN:
          monputs("Sender cancelled");
          return ZCAN;
        case TIMEOUT:
          monputs("Timeout");
          return c;
        case RCDO:
          monputs("Carrier loss detected");
          return c;
        default:
          monputs("Mangled data packet");
          return c;
        }
      }

    if (--length < 0)
      {
      monputs("ZMODEM data subpacket too int");
      return ERROR;
      }
    ++Rxcount;
    *buf++ = c;
    crc = updcrc(c, crc);
    continue;
    }
  }
*/
       
/* Assembler coded now */
/*
int zrdat32(char *buf,int length)
  {
  int c;
  unsigned int crc;
  int d;

  crc = 0xFFFFFFFF;  Rxcount = 0;
  for (;;)
    {
    if ((c = zdlread()) & ~0xFF)
      {
crcfoo:
      switch (c)
        {
        case GOTCRCE:
        case GOTCRCG:
        case GOTCRCQ:
        case GOTCRCW:
          crc = Z_32UpdateCRC( ((d=c)&0xFF), crc);

          if ((c = zdlread()) & ~0xFF) goto crcfoo;
          crc = Z_32UpdateCRC( (c), crc);
          if ((c = zdlread()) & ~0xFF) goto crcfoo;
          crc = Z_32UpdateCRC( (c), crc);
          if ((c = zdlread()) & ~0xFF) goto crcfoo;
          crc = Z_32UpdateCRC( (c), crc);
          if ((c = zdlread()) & ~0xFF) goto crcfoo;
          crc = Z_32UpdateCRC( (c), crc);

          if (crc != 0xDEBB20E3)
            {
            monputs("Bad data packet CRC");
            return ERROR;
            }
          return d;
        case GOTCAN:
          monputs("Sender Cancelled");
          return ZCAN;
        case TIMEOUT:
          monputs("Timeout");
          return c;
        case RCDO:
          monputs("Carrier loss detected");
          return c;
        default:
          monputs("Mangled data packet");
          return c;
        }
      }

    if (--length < 0)
      {
      monputs("ZMODEM data subpacket too long");
      return ERROR;
      }
    ++Rxcount;
    *buf++ = c;
    crc = Z_32UpdateCRC(c, crc);
    continue;
    }
  }
*/

/*
 * Read a ZMODEM header to hdr, either binary or hex.
 *  On success, set Zmodem to 1 and return type of header.
 *   Otherwise return negative on error
 */
int zgethdr(char *hdr)
  {
  int c, n;
  int cancount;

  n = baudrate;   /* Max characters before start of frame */
  cancount = 5;
again:
  Rxframeind = Rxtype = 0;
  switch (c = noxrd7())
    {
    case RCDO:
    case TIMEOUT:
      goto fifi;
    case CAN:
      if (--cancount <= 0)
        {
        c = ZCAN;
        goto fifi;
        }
    default:
agn2:
      if (--n <= 0)
        {
        monputs("Header search garbage count exceeded");
        return ERROR;
        }
      if (c != CAN) cancount = 5;
      goto again;
    case ZPAD:              /* This is what we want. */
      break;
    }
  cancount = 5;
splat:
  switch (c = noxrd7())
    {
    case ZPAD:
      goto splat;
    case RCDO:
    case TIMEOUT:
      goto fifi;
    default:
      goto agn2;
    case ZDLE:              /* This is what we want. */
      break;
    }

  switch (c = noxrd7())
    {
    case RCDO:
    case TIMEOUT:
      goto fifi;
    case ZBIN:
      Rxframeind = ZBIN;
      c =  zrbhdr(hdr);
      break;
    case ZBIN32:
      Rxframeind = ZBIN32;
      c =  zrbhdr32(hdr);
      break;
    case ZHEX:
      Rxframeind = ZHEX;
      c =  zrhhdr(hdr);
      break;
    case CAN:
      if (--cancount <= 0)
        {
        c = ZCAN; goto fifi;
        }
      goto agn2;
    default:
      goto agn2;
    }
  Rxpos = rclhdr(hdr);
fifi:
  switch (c)
    {
    case GOTCAN:
      c = ZCAN;
    case ZNAK:
    case ZCAN:
    case ERROR:
    case TIMEOUT:
    case RCDO:
      sprintf(Msgbuf, "Received %s %s ", frametypes[c+FTOFFSET],
        (c >= 0) ? "header" : "error");
      monputs(Msgbuf);
    }
  return c;
  }

/* Receive a binary style header (type and position) */
/* Assembler coded
int zrbhdr(char *hdr)
  {
  int c;
  int n;
  unsigned short crc;

  if ((c = zdlread()) & ~0xFF) return c;
  Rxtype = c;
  crc = updcrc(c, 0);

  for (n=4; --n >= 0;)
    {
    if ((c = zdlread()) & ~0xFF) return c;
    crc = updcrc(c, crc);
    *hdr++ = c;
    }
  if ((c = zdlread()) & ~0xFF) return c;
  crc = updcrc(c, crc);
  if ((c = zdlread()) & ~0xFF) return c;
  crc = updcrc(c, crc);
  if (crc & 0xFFFF)
    {
    monputs("Bad Header CRC");
    return ERROR;
    }
  return Rxtype;
  }
*/

/* Receive a binary style header (type and position) with 32 bit FCS */
/*Assembler now!
int zrbhdr32(char *hdr)
  {
  int c, n;
  unsigned long crc;

  if ((c = zdlread()) & ~0xFF) return c;
  Rxtype = c;
  crc = 0xFFFFFFFF;
  crc = Z_32UpdateCRC(c, crc);

  for (n=4; --n >= 0;)
    {
    if ((c = zdlread()) & ~0xFF) return c;
    crc = Z_32UpdateCRC(c, crc);
    *hdr++ = c;
    }
  for (n=4; --n >= 0;)
    {
    if ((c = zdlread()) & ~0xFF) return c;
    crc = Z_32UpdateCRC( c, crc);
    }
  if (crc != 0xDEBB20E3)
    {
    monputs("Bad Header CRC");
    return ERROR;
    }
  return Rxtype;
  }
*/

/* Receive a hex style header (type and position) */
int zrhhdr(char *hdr)
  {
  int c;
  unsigned short crc;
  int n;

  if ((c = zgethex()) < 0) return c;
  Rxtype = c;
  crc = updcrc(c, 0);

  for (n=4; --n >= 0;)
    {
    if ((c = zgethex()) < 0) return c;
    crc = updcrc(c, crc);
    *hdr++ = c;
    }
  if ((c = zgethex()) < 0) return c;
  crc = updcrc(c, crc);
  if ((c = zgethex()) < 0) return c;
  crc = updcrc(c, crc);
  if (crc & 0xFFFF)
    {
    monputs("Bad Header CRC");
    return ERROR;
    }
  if (readock(1) == '\r') readock(1);  /* Throw away possible cr/lf */
  return Rxtype;
  }

/* Send a byte as two hex digits */
void zputhex(int c)
  {
  static char digits[] = "0123456789abcdef";

  sendline(digits[(c&0xF0)>>4]);
  sendline(digits[(c)&0xF]);
  }

/*
 * Send character c with ZMODEM escape sequence encoding.
 *  Escape XON, XOFF. Escape CR following @ (Telenet net escape)
 */
/* Assmbler coded now
void zsendline(int c)
  {
  static int lastsent;

  switch (c & 0xFF)
    {
    case ZDLE:
      xsendline(ZDLE);
      xsendline (lastsent = (c ^= 0x40));
      break;
    case 015:
    case 0215:
      if ((lastsent & 0x7F) != '@') goto sendit;
    case 020:
    case 021:
    case 023:
    case 0220:
    case 0221:
    case 0223:
      xsendline(ZDLE);
      c ^= 0x40;
sendit:
    default:
      xsendline(lastsent = c);
    }
  }
*/

/* Decode two lower case hex digits into an 8 bit byte value */
int zgethex()
  {
  int c, n;

  if ((n = noxrd7()) < 0) return n;
  n -= '0';
  if (n > 9) n -= ('a' - ':');
  if (n & ~0xF) return ERROR;

  if ((c = noxrd7()) < 0) return c;
  c -= '0';
  if (c > 9) c -= ('a' - ':');
  if (c & ~0xF) return ERROR;

  return (n<<4 | c);
  }

/*
 * Read a byte, checking for ZMODEM escape encoding
 *  including CAN*5 which represents a quick abort
 */
/* Assembler coded */
/*
int zdlread()
  {
  int c;

  if ((c = readock(Rxtimeout)) != ZDLE) return c;
  if ((c = readock(Rxtimeout)) < 0) return c;
  if (c == CAN)
    if((c = readock(Rxtimeout)) < 0) return c;
  if (c == CAN)
    if((c = readock(Rxtimeout)) < 0) return c;
  if (c == CAN)
    if((c = readock(Rxtimeout)) < 0) return c;
  switch (c)
    {
    case CAN:
      return GOTCAN;
    case ZCRCE:
    case ZCRCG:
    case ZCRCQ:
    case ZCRCW:
      return (c | GOTOR);
    case ZRUB0:
      return 0x7F;
    case ZRUB1:
      return 0xFF;
    default:
      if ((c & 0x60) ==  0x40) return (c ^ 0x40);
      break;
    }
  monputs("Got bad ZMODEM escape sequence");
  return ERROR;
  }
*/

/*
 * Read a character from the modem line with timeout.
 *  Eat parity, XON and XOFF characters.
 */
/* Assembler coded now
noxrd7()
{
  register int c;

  for (;;) {
    if ((c = readock(Rxtimeout)) < 0) return c;
    switch (c &= 0x7F) {
    case XON:
    case XOFF:
      continue;
    default:
      return c;
    }
  }
}
*/

/* Store integer pos in Txhdr */
/* Assembler coded now
void stohdr(int pos)
  {
  Txhdr[ZP0] = pos;
  Txhdr[ZP1] = pos>>8;
  Txhdr[ZP2] = pos>>16;
  Txhdr[ZP3] = pos>>24;
  }
*/

/* Recover an integer from a header */
/* Assembler coded now
int rclhdr(char *hdr)
  {
  int l;

  l = hdr[ZP3];
  l = (l << 8) | hdr[ZP2];
  l = (l << 8) | hdr[ZP1];
  l = (l << 8) | hdr[ZP0];
  return l;
  }
*/

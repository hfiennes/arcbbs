/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 08-February-1990                            <]
                  [>                                             <]
Module name       [> SEAlink file transfer                       <]
Current version   [> 01.08                                       <]
Version date      [> 21-January-1991                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT (c) 1989/90/91 by  <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

/* Main includes */
#include "include.h"

/* System includes */
#ifdef SEVEN
/* filename=0 message=4 pos=1 blocksize=3 filesize=2 */
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
extern void _fclose(FILE**);
#else
#include "servmess.h"
#include "config.h"
#endif

#include "port.h"
#include "crc.h"
                                  
extern FILE *bbs_file[10];
extern int  files_done,portnumber;

extern int  xmtfile(char*,char*),com_getc(int),getblock(char*),
            shipblk(char*,int),ackchk(void),timerset(int),timeup(int);
extern char *rcvfile(char*);
extern void sendblk(FILE*,int),sendack(int,int),window_poll(void),
            mygets(char*,FILE*);

int     cancelled;
char    inname[80];

#define ACK 0x06
#define NAK 0x15
#define SOH 0x01
#define EOT 0x04
#define CAN 0x18

#ifdef SEVEN
extern char rxpath[128];
extern int zmodem_handle;
char msgblk[40];
extern void write_icon(int,int,char*),write_iconint(int,int,int);
extern char *strippath(char*);
int sealink_abort=0;
jmp_buf kills;             

void sealink_msgproc(wimp_eventstr *e,void *handle)
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
      sealink_abort=1;
      break;
      }
    }
  }
#endif

int seatx(char *batchfilename)
  {                  
  char realname[256],alias[21];
  int status,filecount=0;        
#ifdef SEVEN
  batch_entry *file2send;
  sealink_abort=0;
  if (setjmp(kills)==1)
    {  
    _fclose(&bbs_file[4]);
    port_txw(CAN); port_txw(CAN);
    return(NULL);
    }
#endif     

  files_done=cancelled=0;
                    
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
      if ((status=xmtfile(realname,alias))==1) filecount++;
      }
    }
  while(status==1 && feof(bbs_file[3])==0 && *realname!=0); 

  if (feof(bbs_file[3])) xmtfile(NULL,NULL);
                      
  fclose(bbs_file[3]); bbs_file[3]=0;
#else
  batch_next(1);

  do
    {
    if ((file2send=batch_next(0))!=NULL)
      {
      strcpy(realname,file2send->filename);
      strcpy(alias,file2send->alias);
      if ((status=xmtfile(realname,alias))==1) filecount++;
      batch_remove(realname);
      }
    }
  while(status==1 && file2send!=NULL); 

  xmtfile(NULL,NULL);
#endif             

  if (status==0)
    {
    if (cancelled==0) return(FILE_ERROR); else return(CANCEL);   
    }

  if ((files_done=filecount)==0) return(FILE_ERROR); else return(DONE);
  }

int searx(char *batchfilename)
  {
  char *status,genname[80];

#ifdef SEVEN
  strcpy(rxpath,batchfilename);
  sealink_abort=0;
  if (setjmp(kills)==1)
    {  
    _fclose(&bbs_file[4]);
    port_txw(CAN); port_txw(CAN);
    return(NULL);
    }
#endif

  files_done=cancelled=0;

#ifndef SEVEN
  if ((bbs_file[3]=fopen(batchfilename,"wb"))==NULL)
    {
    return(FILE_ERROR);
    }

  do
    {
    sprintf(genname,"<ARCbbs$upload>.%02d/%04d",portnumber,files_done);
    if ((status=rcvfile(genname))!=NULL)
      {
      fputs(inname,bbs_file[3]);
      fputs("\012",bbs_file[3]);
      files_done++;            
      }
    }
  while(status!=NULL); 

  fclose(bbs_file[3]); bbs_file[3]=0;
#else
  do
    {
    if ((status=rcvfile(""))!=NULL) files_done++;            
    }
  while(status!=NULL); 
#endif             

  if (files_done==0)
    {
    if (cancelled==0) return(FILE_ERROR); else return(CANCEL);
    }

  return(DONE);
  }

struct zeros                         /* block zero data structure */
  {
  int  flen;                         /* file length */
  int  fstamp;                       /* file date/time stamp */
  char fnam[17];                     /* original file name */
  char prog[15];                     /* sending program name */
  char fill[88];                     /* reserved for future use */
  };

int outblk;                     /* number of next block to send */
int ackblk;                     /* number of last block ACKed */
int blksnt;                     /* number of last block sent */
int slide;                      /* true if sliding window */
int ackst;                      /* ACK/NAK state */
int numnak;                     /* number of sequential NAKs */
int chktec;                     /* check type, 1=CRC, 0=checksum */

int com_getc(int time)
  {
  int t=timerset(time*10);
    
  do
    {
    if (port_rxbuffer()!=0) return(port_rx()); else window_poll();
#ifdef SEVEN
    if (sealink_abort) longjmp(kills,1);
#endif
    }
  while(!timeup(t));

  return(EOF);
  }

/*  File transmitter logic */
int xmtfile(char *name,char *alias)  /* transmit a file */
  {
  int t1;                            /* timers */
  int endblk;                        /* block number of EOT */
  struct zeros zero;                 /* block zero data */

  if(name && *name)                  /* if sending a file */
    {
    if((bbs_file[4]=fopen(name,"rb"))==NULL)
      {
      return 0;
      }
    
    memset(&zero,0,128);        /* clear out data block */
                               
    fseek(bbs_file[4],0,SEEK_END);
    zero.flen = ftell(bbs_file[4]);
    fseek(bbs_file[4],0,SEEK_SET);/* get file statistics */
#ifdef SEVEN
    write_iconint(zmodem_handle,2,zero.flen);
    write_iconint(zmodem_handle,3,128);
    write_icon(zmodem_handle,0,alias);
#endif
    zero.fstamp = -1;              /* Not supported */
    strcpy(zero.fnam,alias);
#ifdef SEVEN
    strcpy(zero.prog,"ARCterm 7");
#else
    strcpy(zero.prog,"ARCbbs MU");
#endif

    endblk = ((zero.flen+127)/128) + 1;
#ifdef SEVEN
    sprintf(msgblk,msgs_lookup("7_seared"),endblk-1);
    write_icon(zmodem_handle,4,msgblk);
  /*printf("Ready to send %d blocks of %s\n",endblk-1,zero.fnam);*/
#endif
    }
  else endblk = 0;                   /* fake for no file */

  outblk = 1;                        /* set starting state */
  ackblk = -1;
  blksnt = slide = ackst = numnak = 0;
  chktec = 2;                        /* undetermined */

  t1 = timerset(300);                /* time limit for first block */

  while(ackblk<endblk)               /* while not all there yet */
    {
    if(timeup(t1))
      {                                
#ifdef SEVEN
      write_icon(zmodem_handle,4,msgs_lookup("7_seafat"));
      /*printf("\nFatal timeout\n");*/
#endif
      goto abort;
      }

    if(outblk <= ackblk + (slide? 4 : 1))
      {
      if(outblk<endblk)
        {
        if(outblk>0) sendblk(bbs_file[4],outblk);
        else shipblk((char*)&zero,0);
#ifdef SEVEN
        sprintf(msgblk,msgs_lookup("7_seasb"),outblk);
        write_icon(zmodem_handle,4,msgblk);
        /*printf("Sent block #%d  \r",outblk);*/
#endif
        }
      else
        {
        if(outblk==endblk)
          {
          port_txw(EOT);
#ifdef SEVEN
          write_icon(zmodem_handle,4,msgs_lookup("7_seaeot"));
          /*printf("Sent EOT          \r");*/
#endif
          } 
        }
      outblk++;
      t1 = timerset(100);      /* time limit between blocks */
      }

    if (ackchk()==1) goto abort; /* User abort */
    if(numnak>10)
      {                          
#ifdef SEVEN
      write_icon(zmodem_handle,4,msgs_lookup("7_seatoo"));
      /*printf("\nToo many errors\n");*/
#endif
      goto abort;
      }
    }       

#ifdef SEVEN
  write_icon(zmodem_handle,4,msgs_lookup("7_seaeof"));
  /*printf("End of file       \n");*/
#endif

  _fclose(&bbs_file[4]);
  return 1;                          /* exit with good status */

  abort:
  _fclose(&bbs_file[4]);
  return 0;                          /* exit with bad status */
  }

/*  The various ACK/NAK states are:
0:   Ground state, ACK or NAK expected.
1:   ACK received
2:   NAK received
3:   ACK, block# received
4:   NAK, block# received
5:   Returning to ground state
*/

int ackchk()                      /* check for ACK or NAK */
  {
  int c;                             /* one byte of data */
  static int rawblk;                 /* raw block number */

  while((c=com_getc(0))!=EOF)
    {
    if(ackst==3 || ackst==4)
      {
      slide = 0;               /* assume this will fail */
      if(rawblk == c^0xff)     /* see if we believe the number */
        {
        rawblk = outblk - ((outblk-rawblk)&0xff);
        if(rawblk<=outblk && rawblk>outblk-20)
          {
          slide = 1;     /* we have sliding window! */
          if(ackst==3)
            ackblk = ackblk>rawblk? ackblk : rawblk;
          else
            {
            outblk = rawblk<0? 0 : rawblk;
            port_txclear();  /* purge pending output */
            }
#ifdef SEVEN
          sprintf(msgblk,msgs_lookup("7_seata"),ackst==3? "ACK":"NAK",rawblk);
          write_icon(zmodem_handle,4,msgblk);
          write_iconint(zmodem_handle,1,rawblk*128);
          /*printf("%s %d == ",ackst==3? "ACK":"NAK",rawblk);*/
#endif
          }
        }
      ackst = 5;               /* return to ground state */
      }

    if(ackst==1 || ackst==2)
      {
      rawblk = c;
      ackst += 2;
      }

    if(!slide || ackst==0)
      {   
      switch(c)
        {
        case ACK:
          {
          if(!slide)
            {
            ackblk++;    
#ifdef SEVEN
            sprintf(msgblk,"ACK %d",ackblk);
            write_icon(zmodem_handle,4,msgblk);     
            write_iconint(zmodem_handle,1,ackblk*128);
            /*printf("\rACK %d",ackblk);*/
#endif
            }
          ackst = 1;
          numnak = 0; 
          break;
          }  
        case 'C':
        case NAK:
          {
          if(chktec>1)        /* if method not determined yet */
            chktec = (c=='C');/* then do what rcver wants */
  
          if(!slide)
            {
            outblk = ackblk+1;

#ifdef SEVEN
            sprintf(msgblk,"NAK %d",ackblk+1);
            write_icon(zmodem_handle,4,msgblk);     
            write_iconint(zmodem_handle,1,(ackblk+1)*128);
            /*printf("\rNAK %d --",ackblk+1);*/
#endif
            }
          ackst = 2;
          numnak++; 
          break;
          }
        case 24: /* CAN */
          {
          if (com_getc(5)==CAN)
            {
            cancelled=1;
            return(1);
            }
          break;
          }
        }
      }

    if(ackst==5) ackst = 0;
    }      
  return(0);
  }

void sendblk(FILE *f,int blknum)   /* send one block */
  {
  int blkloc;                        /* address of start of block */
  char buf[128];                     /* one block of data */

  if(blknum != blksnt+1)             /* if jumping */
    {
    blkloc = (blknum-1) * 128;
    fseek(f,blkloc,SEEK_SET);        /* move where to */
    }
  blksnt = blknum;
  
  memset(buf,26,128);                /* fill buffer with control Zs */
  fread(buf,1,128,f);                /* read in some data */
  shipblk(buf,blknum);               /* pump it out the comm port */
  }

int shipblk(char *blk,int blknum) /* physically ship a block */
  {
  char *b = blk;                     /* data pointer */
  int crc = 0;                       /* CRC check value */
  int n;                             /* index */

  port_txw(SOH);                     /* block header */
  port_txw(blknum);                  /* block number */
  port_txw(blknum^0xff);             /* block number check value */

  for(n=0; n<128; n++)               /* ship the data */
    {
    if(chktec==0) crc+=*b;
    port_txw(*b++);
    }

  if(chktec)                         /* send proper check value */
    {
    crc=calcrc(blk,128);
    port_txw(crc>>8);
    port_txw(crc&0xff);
    }
  else port_txw(crc);

  return 1;
  }

/*  File receiver logic */

char *rcvfile(char *name)            /* receive file */
  {
  int c,a;                           /* received character */
  int tries;                         /* retry counter */
  int t1;                            /* timer */
  int blknum;                        /* desired block number */
  int inblk;                         /* this block number */
  char buf[128],*stat;               /* data buffer */
  struct zeros zero;                 /* file header data storage */
  int endblk;                        /* block number of EOT, if known */
  int left;                          /* bytes left to output */
  int cancount;

#ifndef SEVEN
  if((bbs_file[4]=fopen(name,"wb"))==NULL)  /* open output file */
    {
    return NULL;
    }
#else
  bbs_file[4]=0;        
#endif

  blknum = 1;                        /* start with block #1 */
  tries = -10;                       /* kludge for first time around */
  chktec = 1;                        /* try for CRC error checking */
  endblk = 0;                        /* we don't know the size yet */
  stat="Startup";                      

  memset(&zero,0,128);

  nakblock:                              /* we got a bad block */
#ifdef SEVEN                                         
  sprintf(msgblk,msgs_lookup("7_seanak"),blknum,stat);
  write_icon(zmodem_handle,4,msgblk);
  write_iconint(zmodem_handle,1,blknum*128);
/*printf("NAK block %d %-4s\r",blknum,stat);*/
#endif
  if(++tries>10)
    {
#ifdef SEVEN                                         
  write_icon(zmodem_handle,4,msgs_lookup("7_seatoo"));
  /*printf("\nToo many errors\n");*/
#endif
    goto abort;
    }
  if(tries==0)                       /* if CRC isn't going */
    chktec=0;                        /* then give checksum a try */

  sendack(0,blknum);                 /* send the NAK */
  goto nextblock;

  ackblock:                          /* we got a good block */
#ifdef SEVEN                        
  tries=0;          
  sprintf(msgblk,msgs_lookup("7_seaack"),blknum);
  write_icon(zmodem_handle,4,msgblk);
  write_iconint(zmodem_handle,1,blknum*128);
/*printf("ACK block %d %-4s\r",blknum-1,stat);*/
#endif

  nextblock:                         /* start of "get a block" */

  t1 = timerset(50);                 /* timer to start of block */
  cancount=0;
  while(!timeup(t1))
    {         
    switch(c=com_getc(0))
      {
      case EOT:
        {
        if(endblk==0 || endblk==blknum) goto endrcv;
        break;
        }     
      case SOH:
        {
        inblk=com_getc(5);
        if(com_getc(5)==(inblk^0xff)) goto blockstart;/* we found a start */
        break;
        }
      case 24: /* CANcel - needs 4 sucessive CAN's */
        {    
        if (++cancount==4)
          {
          cancelled=1;
          goto abort;
          }
        break;
        }
      case EOF:
        {
        stat="Timeout";
        break;
        }
      }
    if (c!=CAN) cancount=0;
    }
  goto nakblock;

  blockstart:               /* start of block detected */
  c = blknum&0xff;
  if(inblk==c && bbs_file[4]!=0) /* if this is the one we want */
    {
    if(getblock(buf))       /* else if we get it okay */
      {  
      int ts=128;

      sendack(1,inblk);     /* ack the data */
      if(endblk)            /* limit file size if known */
        {
        if (left<128) ts=left;
        left-=128;
        if (left<0) left=0;
        }
      if (fwrite(buf,ts,1,bbs_file[4])!=1) goto abort;
      tries = 0;               /* reset try count */
      blknum++;                /* we want the next block */
      goto ackblock;
      }
    else
      {
      goto nakblock;           /* ask for a resend */
      }
    }
  else if(inblk==0 && blknum==1)     /* else if this is the header */
    {
    if(getblock((char*)&zero))
      {                  
#ifdef SEVEN
      char fullname[256];
#endif

      sendack(1,inblk);             /* ack the header */
      if((left=zero.flen)!=0)       /* length to transfer */
        endblk = (left+127)/128 + 1;
#ifdef SEVEN
      zero.fnam[10]=0;
      for(a=0;a<10;a++)
        {
        if (zero.fnam[a]=='.' || zero.fnam[a]=='%' ||
            zero.fnam[a]=='$') zero.fnam[a]='_';
        }
      sprintf(fullname,"%s%s",rxpath,zero.fnam);
      write_icon(zmodem_handle,0,strippath(fullname));
      write_iconint(zmodem_handle,3,128);
      if (left==0) write_icon(zmodem_handle,2,"Unknown");
      else write_iconint(zmodem_handle,2,left);              

      if((bbs_file[4]=fopen(fullname,"wb"))==NULL)  /* open output file */
        {
        return NULL;
        }   
#else
      strcpy(inname,zero.fnam);                                
#endif     
      goto ackblock;
      }
    else
      {
      goto nakblock;           /* bad header block */
      }
    }
  else if(inblk<c || inblk>c+100)    /* else if resending what we have */
    {
    getblock(buf);                /* ignore it */
    sendack(1,inblk);             /* but ack it */
    goto ackblock;
    }
  else                               /* else if running ahead */
    {
    goto nextblock;
    }

  endrcv:
  sendack(1,blknum); /* ACK eot */

  if(blknum>1)                    /* if we really got anything */
    {
    _fclose(&bbs_file[4]);
    return(inname);             /* signal what file we got */
    }

  abort:                            
  _fclose(&bbs_file[4]);
  return NULL;
  }

void sendack(int acknak,int blknum) /* send an ACK or a NAK */
  {
  if(acknak)                         /* send the right signal */
    port_txw(ACK);
  else
    if(chktec)
      port_txw('C');
    else port_txw(NAK);

  port_txw(blknum);                  /* block number */
  port_txw(blknum^0xff);             /* block number check */
  }

int getblock(char *buf)              /* read a block of data */
  {
  int ourcrc=0,hiscrc;               /* CRC check values */
  int c;                             /* one byte of data */
  int n;                             /* index */
  char *b=buf;        

  for(n=0; n<128; n++)
    {
    if((c=com_getc(5))==EOF)
      {
#ifdef SEVEN                                         
      write_icon(zmodem_handle,4,msgs_lookup("7_seash"));
      /*printf("Short block     \r");*/
#endif
      return 0;
      }
    if(chktec==0) ourcrc+=c;
    *buf++=c;
    }

  if(chktec)
    {
    ourcrc=calcrc(b,128);
    hiscrc=com_getc(5)<<8;
    hiscrc|=com_getc(5);
    }
  else
    {
    ourcrc &= 0xff;
    hiscrc = com_getc(5) & 0xff;
    }

  if(ourcrc == hiscrc)
  return 1;
  else
    {
#ifdef SEVEN                                         
    write_icon(zmodem_handle,4,msgs_lookup("7_seach"));
    /*printf("Short block     \r");*/
#endif
  /*if(chktec)
    printf("Bad CRC         \r");
    else printf("Bad Checksum    \r");*/
    return 0;
    }
  }

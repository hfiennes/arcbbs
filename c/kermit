/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCterm VII / ARCbbs                        <]
Author            [> Hugo Fiennes                                <]
Date started      [> 05-March-1990                               <]
                  [>                                             <]
Module name       [> Kermit                                      <]
Current version   [> 00.08                                       <]
Version date      [> 31-December-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>    This source is COPYRIGHT (c) 1991 by     <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "port.h"
#include "kermit.h"

#ifdef SEVEN
#include "term.h"
#include "batch.h"
#include "tprintf.h"
#include "scr.h"
#include "msgs.h"
extern terminal_configuration current;
#else
#include "servmess.h"
#include "config.h"
#endif

extern void kerini(void),spack(char,int,int,char*),spar(char*),rpar(char*,int),
            prerrpkt(char*),notify_hook(void),upstat(char*),settitle(int,char*),
            open_swindow(int,int,int);
extern int  sendsw(void),sendcmdsw(char,char*),bufill(char*),
            bufemp(char*,int,int),gnxtfl(void),sinit(void),sfile(void),sdata(void),
            seof(void),sbreak(void),recsw(char*),rinit(void),rfile(void),
            rdata(void),resp_init(char,char*),resp_data(void),
            rpack(int*,int*,char*),chk1(char*),chk2(char*),chk3(char*);
extern char *strippath(char*);

extern void window_poll(void),write_icon(wimp_w,int,char*),write_iconint(wimp_w,int,int);
extern int files_done;
extern char rxpath[128],xferpath[];


static int  quote_8bit,maxtry,binary,quote_8bit_char,padchar,pad,timint;
static int  size,want_spsiz,spsiz,n,numtry,oldtry,packets_sent,soh;
static int  packets_received,bad_packets,naked_packets,isafile;
static int  filnamcnv,rx,filenumber,blkchk,want_blkchk;

static int  bytes_xferred,bytes_thisfile,packet_size;
static char state,eol,quote;
static char *recpkt,*packet,kermit_file[160],kermit_alias[24],
            kermit_showname[12];

extern FILE *bbs_file[10];

#ifdef SEVEN

#define upstat(s) write_icon(zmodem_handle,4,s)
extern char message_buffer[];
jmp_buf killk;
int kermit_abort;
extern wimp_w zmodem_handle;
extern void kermit_frommenu(void),kermit_update(void);

void kermit_msgproc(wimp_eventstr *e,void *handle)
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
      kermit_abort=1;
      break;
      }
    }
  }

void notify_hook()
  {
  write_icon(zmodem_handle,0,kermit_showname);
  write_iconint(zmodem_handle,1,bytes_thisfile);
  write_iconint(zmodem_handle,2,0);
  write_iconint(zmodem_handle,3,packet_size);
  }

int read_modem()
  {
  clock_t start=clock();
  int data;

  if ((data=port_rx())!=-1) return(data);
  do
    {
    window_poll();
    }
  while((clock()-start)<(timint*100) && (data=port_rx())==-1 && kermit_abort==0);

  if (kermit_abort) longjmp(killk,1);
  if (data!=-1) return(data);

  upstat("Timeout"); 
  return(1000);
  }
#else

#define upstat(s)
#define notify_hook()

extern void _fclose(FILE**);
extern int portnumber;
extern char *message_buffer;
extern void mygets(char*,FILE*);

int read_modem()
  {
  clock_t start=clock();
  int data;

  if ((data=port_rx())!=-1) return(data);
  do
    {
    window_poll();
    }
  while((clock()-start)<(timint*100) && (data=port_rx())==-1);
  if (data!=-1) return(data);

  if (port_dcd()==0) return(2000);
  return(1000);
  }
#endif

/*
 *  k e r i n i
 *
 *  Init protocol routines
 *
 */

void kerini()
  {
  _fclose(&bbs_file[4]);                      /* File pointer */

#ifdef SEVEN
  /* txrx specific */
  if (current.kermit.rx.size>MAXEPACKSIZ) current.kermit.rx.size=MAXEPACKSIZ;
  if (current.kermit.tx.size>MAXEPACKSIZ) current.kermit.tx.size=MAXEPACKSIZ;

  want_spsiz=rx?current.kermit.rx.size:current.kermit.tx.size;
  pad=rx?current.kermit.rx.padsize:current.kermit.tx.padsize;
  timint=rx?current.kermit.rx.timeout:current.kermit.tx.timeout;
  soh=rx?current.kermit.rx.start:current.kermit.tx.start;
  eol=rx?current.kermit.rx.end:current.kermit.tx.end;
  padchar=rx?current.kermit.rx.padchar:current.kermit.tx.padchar;

  quote_8bit=current.kermit.ebq;
  quote_8bit_char=current.kermit.ebqchar;
  maxtry=current.kermit.maxtries;
  binary=current.kermit.binary;
  want_blkchk=current.kermit.blockcheck;
  quote=current.kermit.ctrlchar;
#else
  want_spsiz=2048;
  pad=0;
  timint=10;
  soh=1;
  eol=13;
  padchar=0;

  quote_8bit=0;
  quote_8bit_char='&';
  maxtry=10;
  binary=1;
  want_blkchk=3;
  quote='#';
#endif

  blkchk=1;
  filnamcnv=TRUE;
  isafile=0;

  recpkt=packet=message_buffer; packet_size=0; kermit_showname[0]=0;
  bytes_xferred=bytes_thisfile=0;

  upstat("Starting");
  }

/*
 *  s e n d s w
 *
 *  Sendsw is the state table switcher for sending files.  It loops until
 *  either it finishes, or an error is encountered.  The routines called
 *  by sendsw are responsible for changing the state.
 *
 */

int sendsw()
  {
  state='S';                          /* Send initiate is the start state */
  n = 0;                              /* Initialize message number */
  packets_sent=packets_received=0;    /* Initialize statistics */
  bad_packets=naked_packets=0;
  bytes_xferred=0L;
  bytes_thisfile=0L;
  filenumber=0;                       /* First file! */

  numtry=0;                           /* Say no tries yet */
  while(TRUE)                         /* Do this as long as necessary */
    {
    switch(state)
      {
      case 'S': state = sinit();  upstat("Init"); break;
      case 'F': state = sfile();  upstat("File header"); break;
      case 'D': state = sdata();  break;
      case 'Z': state = seof();   upstat("End of file"); break;
      case 'B': state = sbreak(); upstat("End of transmission"); break;
      case 'C':                   upstat("Transfer complete");
                                  return (TRUE);
      case 'A':                   upstat("Transfer aborted");
          _fclose(&bbs_file[4]);
          return (FALSE);                  /* "Abort" */
      default:                upstat("Strange packet-abort");
          _fclose(&bbs_file[4]);
          return (FALSE);                  /* Unknown, fail */
      }
    }
  }


/*
 *  s i n i t
 *
 *  Send Initiate: send this host's parameters and get other side's back.
 */

int sinit()
  {
  int num, len;                       /* Packet number, length */

  if (numtry++ > maxtry) return('A'); /* If too many tries, give up */

  spar(packet);                       /* Fill up init info packet */
  spack('S',n,13,packet);             /* Send my init info */
  switch(rpack(&len,&num,recpkt))     /* What was the reply? */
    {
    case 'N':
      naked_packets++;                /* count NAKs */
      return(state);                  /* NAK, try it again */

    case 'Y':
      upstat("ACK"); /* ACK */
      if (n != num)                   /* If wrong ACK, stay in S state */
        return(state);                /* and try again */
      rpar(recpkt,len);               /* Get other side's init info */

      if (eol == 0) eol = '\n'; /* Check and set defaults */
      if (quote == 0) quote = MYQUOTE;

      numtry = 0;                     /* Reset try counter */
      n = (n+1)%64;                   /* Bump packet count */

      blkchk=want_blkchk;
      return('F');                    /* OK, switch state to F */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:
      return(state);                  /* Receive failure, try again */
        
    default:
      return('A');                    /* Anything else, just "abort" */
    }
  }


/*
 *  s f i l e
 *
 *  Send File Header.
 */

int sfile()
  {
  int num, len;                       /* Packet number, length */

  bytes_thisfile=0;
  if (numtry++ > maxtry) return('A'); /* If too many tries, give up */
    
  upstat("Opening file");
  
  gnxtfl();
  if ((bbs_file[4] = fopen(kermit_file,"rb"))==NULL)
    {
    char msg[80];
    int len;

    upstat("Cannot open file");
    sprintf(msg,"Cannot open file %s\n",kermit_file);
    len=strlen(msg);
    spack('E',n,len,msg);
    return('A');
    }

  len = strlen(kermit_alias);         /* Compute length of new filename */
  strcpy(kermit_showname,kermit_alias);

  notify_hook();                      /* Update filename */

  spack('F',n,len,kermit_alias);      /* Send an F packet */
  switch(rpack(&len,&num,recpkt))     /* What was the reply? */
    {                   
    case 'N':                         /* NAK, just stay in this state, */
      num = (--num<0 ? 63:num);       /* unless it's NAK for next packet */
      if (n != num)                   /* which is just like an ACK for */ 
        {
        naked_packets++;
        return(state);                /* this packet so fall thru to... */
        }

    case 'Y':
      if (n != num) return(state);    /* If wrong ACK, stay in F state */
      numtry = 0;                     /* Reset try counter */
      n = (n+1)%64;                   /* Bump packet count */
      size = bufill(packet);          /* Get first data from file */
      return('D');                    /* Switch state to D */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:
      return(state);                  /* Receive failure, stay in F state */

    default:
      return('A');                    /* Something else, just "abort" */
    }
  }


/*
 *  s d a t a
 *
 *  Send File Data
 */

int sdata()
  {
  int num, len;                       /* Packet number, length */

  if (numtry++ > maxtry) return('A'); /* If too many tries, give up */

  spack('D',n,size,packet);           /* Send a D packet */
  switch(rpack(&len,&num,recpkt))     /* What was the reply? */
    {               
    case 'N':                         /* NAK, just stay in this state, */
      num = (--num<0 ? 63:num);       /* unless it's NAK for next packet */
      if (n != num)                   /* which is just like an ACK for */
        {
        naked_packets++;
        return(state);                /* this packet so fall thru to... */
        }
               
    case 'Y':                         /* ACK */
      if (n != num) return(state);    /* If wrong ACK, fail */
      numtry = 0;                     /* Reset try counter */
      n = (n+1)%64;                   /* Bump packet count */
      if ((size = bufill(packet)) == EOF) /* Get data from file */
        return('Z');                  /* If EOF set state to that */
      return('D');                    /* Got data, stay in state D */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:
      return(state);                  /* Receive failure, stay in D */

    default:
      return('A');                    /* Anything else, "abort" */
    }
  }


/*
 *  s e o f
 *
 *  Send End-Of-File.
 */

int seof()
  {
  int num, len;                       /* Packet number, length */
  if (numtry++ > maxtry) return('A'); /* If too many tries, "abort" */

  spack('Z',n,0,packet);              /* Send a 'Z' packet */
  switch(rpack(&len,&num,recpkt))     /* What was the reply? */
    {
    case 'N':                         /* NAK, just stay in this state, */
      num = (--num<0 ? 63:num);       /* unless it's NAK for next packet, */
      if (n != num)                   /* which is just like an ACK for */
        {
        naked_packets++;
        return(state);                /* this packet so fall thru to... */
        }

    case 'Y':                         /* ACK */
      if (n != num) return(state);    /* If wrong ACK, hold out */
      numtry = 0;                     /* Reset try counter */
      n = (n+1)%64;                   /* and bump packet count */

      files_done++;
      upstat("Closing file");
      _fclose(&bbs_file[4]);                   /* Close the input file */

#ifdef SEVEN
      batch_remove(kermit_file);
#endif

      if (gnxtfl() == FALSE)          /* No more files to go? */
        return('B');                  /* if not, break, EOT, all done */
      return('F');                    /* More files, switch state to F */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:
      return(state);                  /* Receive failure, stay in Z */

    default:
      return('A');                    /* Something else, "abort" */
    }
  }


/*
 *  s b r e a k
 *
 *  Send Break (EOT)
 */

int sbreak()
  {
  int num, len;                       /* Packet number, length */
  if (numtry++ > maxtry) return('A'); /* If too many tries "abort" */

  spack('B',n,0,packet);              /* Send a B packet */
  switch (rpack(&len,&num,recpkt))    /* What was the reply? */
    {
    case 'N':                         /* NAK, just stay in this state, */
      num = (--num<0 ? 63:num);       /* unless NAK for previous packet, */
      if (n != num)                   /* which is just like an ACK for */
        {
        naked_packets++;
        return(state);                /* this packet so fall thru to... */
        }                        

    case 'Y':                         /* ACK */
      if (n != num) return(state);    /* If wrong ACK, fail */
      numtry = 0;                     /* Reset try counter */
      n = (n+1)%64;                   /* and bump packet count */
      return('C');                    /* Switch state to Complete */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:
      return(state);                  /* Receive failure, stay in B */

    default:
      return ('A');                   /* Other, "abort" */
    }
  }

/*
 *  r e c s w
 *
 *  This is the state table switcher for receiving files.
 */

int recsw(char *w)
  {
  state = 'R';                        /* Receive-Init is the start state */
  n = 0;                              /* Initialize message number */
  numtry = 0;                         /* Say no tries yet */
  packets_sent=packets_received=0;    /* Initialize statistics */
  bad_packets=naked_packets=0;
  bytes_xferred=bytes_thisfile=0;

  while(TRUE)
    {
    switch(state)                     /* Do until done */
      {
      case 'R': state = rinit(); upstat("Init"); break;
      case 'F': state = rfile(); upstat("File header"); break;
      case 'D': state = rdata(); break;
      case 'C':                  upstat("End of transmission"); 
                                 return(TRUE);
      case 'A': default:         upstat("Abort");
                                 _fclose(&bbs_file[4]);
                                 return(FALSE); /* "Abort" state */
      }
    }
  }
    
/*
 *  r i n i t
 *
 *  Receive Initialization
 */
  
int rinit()
  {
  int len, num;                       /* Packet length, number */

  if (numtry++ > maxtry) return('A'); /* If too many tries, abort */

  /* spack('R',n,strlen(filnam),filnam); Send "receive init"?! not one! */

  switch(rpack(&len,&num,recpkt))     /* Get a packet */
    {
    case 'S':                         /* Send-Init */
      rpar(recpkt,len);               /* Get the other side's init data */
      spar(packet);                   /* Fill up packet with my init info */
      spack('Y',n,13,packet);         /* ACK with my parameters */
      oldtry = numtry;                /* Save old try count */
      numtry = 0;                     /* Start a new counter */
      n = (n+1)%64;                   /* Bump packet number, mod 64 */

      blkchk=want_blkchk;
      return('F');                    /* Enter File-Receive state */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case 'N':                         /* NAK */
      naked_packets++;
      return(state);                  /* stay in this state */

    case FALSE:                       /* Didn't get packet */
      spack('N',n,0,0);               /* Return a NAK */
      return(state);                  /* Keep trying */

    default:
      return('A');                    /* Some other packet type, "abort" */
    }
  }


/*
 *  r f i l e
 *
 *  Receive File Header
 */

int rfile()
  {
  int num, len, i;                    /* Packet number, length */
  char c;          

  if (numtry++ > maxtry) return('A'); /* "abort" if too many tries */

  switch(rpack(&len,&num,packet))     /* Get a packet */
    {
    case 'S':                         /* Send-Init, maybe our ACK lost */
      if (oldtry++ > maxtry) return('A'); /* If too many tries "abort" */
      if (num == ((n==0) ? 63:n-1))   /* Previous packet, mod 64? */
        {                             /* Yes, ACK it again with  */
        spar(packet);                 /* our Send-Init parameters */
        spack('Y',num,13,packet);
        numtry = 0;                   /* Reset try counter */
        return(state);                /* Stay in this state */
        }
      else return('A');               /* Not previous packet, "abort" */

    case 'Z':
      upstat("End of file");          /* End-Of-File */
      if (oldtry++ > maxtry) return('A');
      if (num == ((n==0) ? 63:n-1))   /* Previous packet, mod 64? */
        {                             /* Yes, ACK it again. */
        spack('Y',num,0,0);
        numtry = 0;
        return(state);                /* Stay in this state */
        }
      else return('A');               /* Not previous packet, "abort" */

    case 'F':                         /* File Header (just what we want) */
      if (num != n) return('A');      /* The packet number must be right */
      bytes_thisfile=0;               /* Reset byte counter */
      _fclose(&bbs_file[4]);

#ifdef SEVEN
      i=0;                            /* Modify extension */
      while ((c=packet[i])!=NULL) 
        {
        if (c=='.' || c==':') packet[i]='_';
        i++;
        }
      packet[10]=0;                   /* And trim length if necessary */
#endif

      strcpy(kermit_showname,packet);
      strcpy(kermit_alias,packet);
      upstat("Opening file");

#ifdef SEVEN
      strcpy(kermit_file,xferpath);
      strcat(kermit_file,packet);     /* Copy the file name */
#else
      sprintf(kermit_file,"<ARCbbs$Upload>.%02d/%04d",portnumber,files_done);
#endif

      if ((bbs_file[4]=fopen(kermit_file,"wb"))==NULL)
        {
        char msg[80];
        int len;

        sprintf(msg,"Cannot create %s\n",kermit_file);
        len=strlen(msg);
        spack('E',n,len,msg);
        upstat("Cannot create file");
        return('A');
        }

      spack('Y',n,0,0);               /* Acknowledge the file header */
      oldtry = numtry;                /* Reset try counters */
      numtry = 0;
      n = (n+1)%64;                   /* Bump packet number, mod 64 */
      return('D');                    /* Switch to Data state */

    case 'B':                         /* Break transmission (EOT) */
      if (num != n) return ('A');     /* Need right packet number here */
      spack('Y',n,0,0);               /* Say OK */
      return('C');                    /* Go to complete state */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:                       /* Didn't get packet */
      spack('N',n,0,0);               /* Return a NAK */
      return(state);                  /* Keep trying */

    default:
      return ('A');                   /* Some other packet, "abort" */
    }
  }


/*
 *  r d a t a
 *
 *  Receive Data
 */

int rdata()
  {
  int num, len;                       /* Packet number, length */
  if (numtry++ > maxtry) return('A'); /* "abort" if too many tries */

  switch(rpack(&len,&num,packet))     /* Get packet */
    {
    case 'D':                         /* Got Data packet */
      if (num != n)                   /* Right packet? */
        {                             /* No */
        if (oldtry++>maxtry)
          return('A');                /* If too many tries, abort */

        if (num == ((n==0) ? 63:n-1)) /* Else check packet number */
          {                           /* Previous packet again? */
          spack('Y',num,6,packet);    /* Yes, re-ACK it */
          numtry = 0;                 /* Reset try counter */
          return(state);              /* Don't write out data! */
          }
        bad_packets++;
        return('A');                  /* sorry, wrong number */
        }

      /* Got data with right packet number */
      if (!bufemp(packet,len,1))
        {
        spack('E',n,COUNTED_STRING("Write failed"));
        upstat("Write failed");
        return('A');
        }

      spack('Y',n,0,0);               /* Acknowledge the packet */
      oldtry = numtry;                /* Reset the try counters */
      numtry = 0;
      n = (n+1)%64;                   /* Bump packet number, mod 64 */
      return('D');                    /* Remain in data state */

    case 'F':                         /* Got a File Header */
      if (oldtry++ > maxtry)
        return('A');                  /* If too many tries, "abort" */
      if (num == ((n==0) ? 63:n-1))   /* Else check packet number */
        {                             /* It was the previous one */
        spack('Y',num,0,0);           /* ACK it again */
        numtry = 0;                   /* Reset try counter */
        return(state);                /* Stay in Data state */
        }
      else return('A');               /* Not previous packet, "abort" */

    case 'Z':                         /* End-Of-File */
      if (num != n) return('A');      /* Must have right packet number */
      spack('Y',n,0,0);               /* OK, ACK it. */
      _fclose(&bbs_file[4]);                   /* Close the file */
#ifndef SEVEN
      fprintf(bbs_file[3],"%s\012",kermit_alias);
#endif
      files_done++;
      n = (n+1)%64;                   /* Bump packet number */
      return('F');                    /* Go back to Receive File state */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:                       /* Didn't get packet */
      spack('N',n,0,0);               /* Return a NAK */
      return(state);                  /* Keep trying */
            
    default:
      return('A');                    /* Some other packet, "abort" */
    }
  }

/*
 *  s e n d c m d s w
 *
 *  This is the state table switcher for sending server commands files.
 */

int sendcmdsw(char cmdchr,char *cmdstr)
  {
  state = 'R';                        /* Response-Init is the start state */
  n = 0;                              /* Initialize message number */
  numtry = 0;                         /* Say no tries yet */
  packets_sent = packets_received = 0;/* Initialize statistics */
  bad_packets = naked_packets = 0;
  bytes_xferred = 0L; isafile=0;

  while(TRUE)
    {
    switch(state)                     /* Do until done */
      {
      case 'R':
        if (numtry++ > maxtry) { state='A'; break; }
        spack(cmdchr,n,strlen(cmdstr),cmdstr); /* Send server command */

      case 'T':
        state = resp_init(cmdchr, cmdstr); /* Response-Init */
        break;

      case 'D':
        state = resp_data();
        break;                        /* Response-Data */

      case 'C':
        return(TRUE);                 /* Complete state */

      case 'A':
        return(FALSE);                /* "Abort" state */

      default:
        return(FALSE);
      }
    }
  }
    
/*
 *  r e s p _ i n i t
 *
 *  Server command response init
 */
  
int resp_init(char cmdchr,char *cmdstr)
  {
  int len,num,a;                      /* Packet length, number */

  switch(rpack(&len,&num,recpkt))     /* Get a packet */
    {
    case 'Y':                         /* ACK (short reply) */
      if (n != num) return(state);    /* If wrong ACK, fail */
      numtry = 0;                     /* Reset try counter */
      n = (n+1)%64;                   /* Bump packet count */

#ifdef SEVEN
      for(a=0;a<len;a++) DISPLAY_DATA(recpkt[a]);
#endif
      return('C');                    /* Transaction complete */

    case 'N':                         /* NAK */
      naked_packets++;
      return('R');                    /* stay in this state */

    case 'Z':                         /* End-Of-File */
      if (oldtry++ > maxtry) return('A');
      if (num == ((n==0) ? 63:n-1))   /* Previous packet, mod 64? */
        {                             /* Yes, ACK it again. */
        spack('Y',num,0,0);
        numtry = 0;
        return(state);                /* Stay in this state */
        }
      else return('A');               /* Not previous packet, "abort" */

    case 'X':                         /* Text Header (Long reply) */
      if (num != n) return('A');      /* The packet number must be right */
#ifdef SEVEN
      for(a=0;a<len;a++) DISPLAY_DATA(recpkt[a]);
#endif
      spack('Y',n,0,0);               /* Acknowledge the file header */
      oldtry = numtry;                /* Reset try counters */
      numtry = 0;                     /* ... */
      n = (n+1)%64;                   /* Bump packet number, mod 64 */
      return('D');                    /* Switch to Data state */

    case 'F':                         /* File Header (Long reply) */
      if (num != n) return('A');      /* The packet number must be right */
#ifdef SEVEN
      for(a=0;a<len;a++) DISPLAY_DATA(recpkt[a]);
#endif
      spack('Y',n,0,0);               /* Acknowledge the file header */
      oldtry = numtry;                /* Reset try counters */
      numtry = 0;                     /* ... */
      n = (n+1)%64;                   /* Bump packet number, mod 64 */
      return('D');                    /* Switch to Data state */

    case 'B':                         /* Break transmission (EOT) */
      if (num != n) return ('A');     /* Need right packet number here */
      spack('Y',n,0,0);               /* Say OK */
      return('C');                    /* Go to complete state */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case 'S':                         /* Send-Init */
      kerini();
      rpar(recpkt,len);               /* Get the other side's init data */
      spar(packet);                   /* Fill up packet with my init info */
      spack('Y',n,13,packet);         /* ACK with my parameters */
      oldtry = numtry;                /* Save old try count */
      numtry = 0;                     /* Start a new counter */
      n = (n+1)%64;                   /* Bump packet number, mod 64 */

      blkchk=want_blkchk;

      if (cmdchr=='R')
        {
        /* GET command */
        isafile=1;
        return(rfile());
        }
      return('T');                    /* Enter File-Receive state */

    case FALSE:                       /* Didn't get packet */
      spack('N',n,0,0);               /* Return a NAK */
      return('R');                    /* Keep trying */

    default:
      return ('A');                   /* Some other packet, "abort" */
    }
  }


/*
 *  r e s p _ d a t a
 *
 *  Receive server command response data (long response)
 */

int resp_data()
  {
  int num, len;                       /* Packet number, length */
  if (numtry++ > maxtry) return('A'); /* "abort" if too many tries */

  switch(rpack(&len,&num,packet))     /* Get packet */
    {
    case 'D':                         /* Got Data packet */
      if (num != n)                   /* Right packet? */
        {
        if (oldtry++ > maxtry)
          return('A');                /* If too many tries, abort */
        if (num == ((n==0) ? 63:n-1)) /* Else check packet number */
          {                           /* Previous packet again? */
          spack('Y',num,6,packet);    /* Yes, re-ACK it */
          numtry = 0;                 /* Reset try counter */
          return(state);              /* Don't write out data! */
          }

        bad_packets++;
        return('A');                  /* sorry, wrong number */
        }

      bufemp(packet,len,isafile);
      spack('Y',n,0,0);               /* Acknowledge the packet */
      oldtry = numtry;                /* Reset the try counters */
      numtry = 0;                     /* ... */
      n = (n+1)%64;                   /* Bump packet number, mod 64 */
      return('D');                    /* Remain in data state */

    case 'F':                         /* Got a File Header */
      if (oldtry++ > maxtry)
        return('A');                  /* If too many tries, "abort" */
      if (num == ((n==0) ? 63:n-1))   /* Else check packet number */
        {                             /* It was the previous one */
        spack('Y',num,0,0);           /* ACK it again */
        numtry = 0;                   /* Reset try counter */
        return(state);                /* Stay in Data state */
        }
      else return('A');               /* Not previous packet, "abort" */

    case 'Z':                         /* End-Of-File */
      if (num != n) return('A');      /* Must have right packet number */
      spack('Y',n,0,0);               /* OK, ACK it. */
      n = (n+1)%64;                   /* Bump packet number */
      if (isafile) return(rfile());
      return('T');                    /* Go back to Receive File state */

    case 'B':                         /* Break transmission */
      if (num != n) return('A');      /* Must have right packet number */
      spack('Y',n,0,0);               /* OK, ACK it. */
      n = (n+1)%64;                   /* Bump packet number */
      return('C');                    /* Go back to Receive File state */

    case 'E':                         /* Error packet received */
      prerrpkt(recpkt);               /* Print it out and */
      return('A');                    /* abort */

    case FALSE:                       /* Didn't get packet */
      spack('N',n,0,0);               /* Return a NAK */
      return(state);                  /* Keep trying */

    default:
      return('A');                    /* Some other packet, "abort" */
    }
  }

/*
 *  s p a c k
 *
 *  Send a Packet
 */

void spack(char type,int num,int len,char *data)
  {
  int i;                              /* Character loop counter */
  char chksum, buffer[2200];          /* Checksum, packet buffer */
  register char *bufp,*bp;            /* Buffer pointer */

  port_rxclear();                     /* Flush pending input */
  
  bufp = buffer;                      /* Set up buffer pointer */
  for (i=1; i<=pad; i++) port_txw(padchar); /* Issue any padding */

  *bufp++ = soh;                      /* Packet marker */
  if (len>94)
    {
    *bufp++=' ';                      /* Type 0 header */
    *bufp++=tochar(num);              /* Packet number */
    *bufp++=type;                     /* Packet type */
    *bufp++=tochar((len+blkchk)/95);
    *bufp++=tochar((len+blkchk)%95);  /* Packet length */
    *bufp=0;
    *bufp++=tochar(chk1(buffer+1));   /* Header checksum */
    }
  else
    {
    *bufp++=tochar(len+2+blkchk);     /* Send the character count */
    *bufp++=tochar(num);              /* Packet number */
    *bufp++=type;                     /* Packet type */
    }

  for(i=0;i<len;i++) *bufp++=data[i];

  *bufp=0;
  switch(blkchk)
    {
    case 1: /* Single byte checksum */
      chksum=chk1(buffer+1);
      *bufp++=tochar(chksum);
      break;
    case 2: /* 2 byte checksum */
      chksum=chk2(buffer+1);
      *bufp++=tochar((chksum>>6)&0x3f);
      *bufp++=tochar(chksum&0x3f);
      break;
    case 3: /* 3 byte CRC */
      chksum=chk3(buffer+1);
      *bufp++=tochar((chksum>>12)&0x3f);
      *bufp++=tochar((chksum>>6)&0x3f);
      *bufp++=tochar(chksum&0x3f);
      break;
    }

  *bufp++=eol;
  *bufp=0;

  bp=buffer;
  while(*bp) port_txw(*bp++);         /* Send the packet */
  packets_sent++;                     /* Count all */

  if (!rx) packet_size=len;
  notify_hook();                      /* Give a chance to update display */
  }

/*
 *  r p a c k
 *
 *  Read a Packet
 */

int rpack(int *len,int *num,char *data)
  {
  int i,t=0,packetlen;                /* Data character number, loop exit */
  char                                /* Current input character */
       type,                          /* Packet type */
       cchksum,                       /* Our (computed) checksum */
       rchksum,                       /* Checksum received from other host */
       p[MAXEPACKSIZ+100],*p1;

  while (t!=soh)                      /* Wait for packet header */
    {
    t=read_modem(); 
    if (t==1000) return(FALSE);       /* Timeout */
    if (t==2000) return('U');         /* User abort */
    }

  p1=p;
  do
    {
    t=read_modem();
    if (t==1000) return(FALSE);       /* Timeout */
    if (t==2000) return('U');         /* User abort */
    if (binary==0 || quote_8bit) t&=0x7f;
    if (t!=10 && t!=13 && t!=eol) *p1++=t; else *p1=0;
    }
  while(t!=10 && t!=13 && t!=eol && (p1-p)<(MAXEPACKSIZ+50));
  p1=p;

  packetlen=unchar(*p1++);            /* Character count */

  if (packetlen==0)
    {
    char lenx1,lenx2,hcheck;

    /* Type 0 header */
    *num=unchar(*p1++);               /* SEQ */
    type=*p1++;                       /* TYPE */
    lenx1=unchar(*p1++);              /* LENX1 */
    lenx2=unchar(*p1++);              /* LENX2 */
    hcheck=unchar(*p1);
    *p1=0;
    if (chk1(p)!=hcheck) return(FALSE);
    *p1++=tochar(hcheck);
    *len=((lenx1*95)+lenx2)-blkchk;
    }
  else
    {
    *len=packetlen-2-blkchk;
    *num = unchar(*p1++);             /* Packet number */
    type = *p1++;                     /* Packet type */
    }

  for (i=0; i<*len; i++) data[i]=*p1++;/* The data itself, if any */
  data[*len] = 0;                     /* Mark the end of the data */

  switch(blkchk)
    {
    case 1: /* Single byte checksum */
      rchksum=unchar(p1[0]);
      *p1=0;
      cchksum=chk1(p);
      break;
    case 2: /* 2 byte checksum */
      rchksum=(unchar(p1[0])<<6)|unchar(p1[1]);
      *p1=0;
      cchksum=chk2(p);
      break;
    case 3: /* 3 byte CRC */
      rchksum=(unchar(p1[0])<<12)|(unchar(p1[1])<<6)|unchar(p1[2]);
      *p1=0;
      cchksum=chk3(p);
      break;
    }
  port_rxclear();                     /* Flush any padding, etc. */

  if (rx) packet_size=*len;
  packets_received++;                 /* count all */
  if (type=='N') upstat("NAK");
  notify_hook();                      /* maybe update screen */

  if (cchksum!=rchksum)
    {
    bad_packets++;
    return(FALSE);
    }

  return(type);                       /* All OK, return packet type */
  }


/*
 *  b u f i l l
 *
 *  Get a bufferful of data from the file that's being sent.
 */

int bufill(char *buffer)
  {
  int i,                              /* Loop index */
      t,                              /* Char read from file */
      count;                          /* byte count */
  char t7;                            /* 7-bit version of above */

  i = 0;                              /* Init data buffer pointer */
  count = 0;
  while((t = getc(bbs_file[4])) != EOF)        /* Get the next character */
    {
    count++;
    t7 = t & 0177;                    /* Get low order 7 bits */

    if (quote_8bit && t==quote_8bit_char) buffer[i++] = quote;
    if (quote_8bit && (t&0x80))
      {
      buffer[i++] = quote_8bit_char;
      t=t7;
      }

    if (t7<SP || t7==DEL || t7==quote)/* Does this char require */
      {                               /* special handling? */
      if (t==10 && binary==0)
        {                             /* Do LF->CRLF mapping if !image */
        buffer[i++] = quote;
        buffer[i++] = ctl(13);
        }
      buffer[i++] = quote;            /* Quote the character */
      if (t7 != quote)
        {
        t = ctl(t);                   /* and uncontrolify */
        t7 = ctl(t7);
        }
      }

    if (binary)
      buffer[i++]=t;                  /* Deposit the character itself */
    else
      buffer[i++]=t7;

    if (i >= spsiz-8)                 /* Check length */
      {
      bytes_xferred += (long) count;
      bytes_thisfile += (long) count;
      return(i);
      }
    }

  bytes_xferred += (long) count;
  bytes_thisfile += (long) count;
  if (i==0) return(EOF);              /* Wind up here only on EOF */
  return(i);                          /* Handle partial buffer */
  }

/*
 *      b u f e m p
 *
 *  Put data from an incoming packet into a file.
 */

int bufemp(char *buffer,int len,int file)
  {
  int i,count=0,q8bit;
  char t;

  for (i=0; i<len; i++)               /* Loop thru the data field */
    {
    q8bit=FALSE;
    t=buffer[i];                      /* Get character */
    if (quote_8bit && t==quote_8bit_char)
      {
      q8bit=TRUE;
      t=buffer[++i];
      }

    if (t==quote)                     /* Control quote? */
      {                               /* Yes */
      t=buffer[++i];                  /* Get the quoted character */
      if ((t&0x7f)!=quote && (!(quote_8bit && t==quote_8bit_char)))
        t = ctl(t);                   /* No, uncontrollify it */
      }

    if (t==CR && binary==0 && file)    /* Don't pass CR if in image mode */
      continue;

    if (q8bit) t|=0200;
    count++;
    if (file)
      {
      if (fputc(t,bbs_file[4])==EOF)
        {
        spack('E',n,COUNTED_STRING("File write failed"));
        upstat("Write failed"); 
        bytes_xferred += (long) count;
        bytes_thisfile += (long) count;
        return(FALSE);
        }
      }
    else
      {
#ifdef SEVEN
      DISPLAY_DATA(t);
#endif
      }
    }

  bytes_xferred += (long) count;
  bytes_thisfile += (long) count;
  return(TRUE);
  }


/*
 *  g n x t f l
 *
 *  Get next file in a file group
 */

#ifdef SEVEN
int gnxtfl()
  {
  batch_entry *file2send=batch_next(0);

  if (file2send)
    {
    strcpy(kermit_file,file2send->filename);
    strcpy(kermit_alias,file2send->alias);
    return(TRUE);
    }
  return(FALSE);
  }
#else
int gnxtfl()
  {
  mygets(kermit_file,bbs_file[3]);
  mygets(kermit_alias,bbs_file[3]);
  if (feof(bbs_file[3])) return(FALSE);
  return(TRUE);
  }
#endif

/*
 *  s p a r
 *
 *  Fill the data array with my send-init parameters
 *
 */

void spar(char *data)
  {
  data[0] = tochar(want_spsiz>94?94:want_spsiz);
                                      /* Biggest packet I can receive */
  data[1] = tochar(timint);           /* When I want to be timed out */
  data[2] = tochar(pad);              /* How much padding I need */
  data[3] = ctl(padchar);             /* Padding character I want */
  data[4] = tochar(MYEOL);            /* End-Of-Line character I want */
  data[5] = quote;                    /* Control-Quote character I send */
  if (quote_8bit)
    data[6]=quote_8bit_char;          /* 8th-bit quote character */
  else
    data[6]='N';                      /* Won't do 8th-bit quoting */
  data[7] = want_blkchk+'0';          /* Checksum type */
  data[8] = ' ';                      /* No compression */
  data[9] = tochar(2);                /* Long packet support */
  data[10]= tochar(0);                /* No windows */
  data[11]= tochar(want_spsiz/95);
  data[12]= tochar(want_spsiz%95);    /* Max packet size */
  }


/*  r p a r
 *
 *  Get the other host's send-init parameters
 *
 */

void rpar(char *data,int len)
  {
  int i;

  spsiz=unchar(data[0]);              /* Maximum send packet size */
  if (spsiz>MAXPACKSIZ) spsiz=MAXPACKSIZ;

  if ((i=unchar(data[1]))>timint) timint = i;
  pad=unchar(data[2]);                /* Number of pads to send */
  padchar=ctl(data[3]);               /* Padding character to send */
  eol=unchar(data[4]);                /* EOL character I must send */
  quote=data[5];                      /* Incoming data quote character */
  if (len>=7)                         /* If other side mentioned 8th-bit */
    {
    /* 8th bit quoting */
    switch (data[6])
      {
      case 'N':
        quote_8bit=FALSE;
        break;
      case 'Y':
        break;
      default:
        if ((data[6] < 33 || data[6] > 126)
            || (data[6] > 62 && data[6] < 96))
          {
          quote_8bit = FALSE;
          break;
          }
        quote_8bit = TRUE;
        quote_8bit_char = data[6];
        break;
      }
    }

  if (len>=8)
    {
    /* Check type */
    if (data[7]>='1' && data[7]<='3') want_blkchk=data[7]-'0'; else want_blkchk=1;
#ifdef SEVEN
    if (want_blkchk>current.kermit.blockcheck) want_blkchk=current.kermit.blockcheck;
#else
    if (want_blkchk>3) want_blkchk=3;
#endif
    }

  if (len>=10)
    {
    char *tp=data+10;
    int capabilities=unchar(data[9]);

    /* Check capabilites */
    if (capabilities&1) tp++;

    if (capabilities&2)
      {
      /* Long packet, get packet size */
      spsiz=(unchar(tp[1])*95)+unchar(tp[2]);

      /*if (spsiz>MAXEPACKSIZ) spsiz=MAXEPACKSIZ;*/
      if (spsiz>want_spsiz) spsiz=want_spsiz;
      }
    }

  /*if (want_spsiz>spsiz) want_spsiz=spsiz;*/
  }

/*
 *  p r e r r p k t
 *
 *  Print contents of error packet received from remote host.
 */

void prerrpkt(char *msg)
  {
#ifdef SEVEN
  tprintf(0,"Aborted by remote: %s\r\n",msg);
#endif
  return;
  }


#ifdef SEVEN
int kermit(int istate, char *s1,char *s2)
  {
  kermit_abort=files_done=0;

  strcpy(rxpath,xferpath);

  kermit_frommenu();
  kermit_update();

  if (istate!='v' && istate!='r' && istate!='s')
    {
    win_register_event_handler(zmodem_handle,kermit_msgproc,0);
    settitle(zmodem_handle,msgs_lookup("7kers"));
    write_icon(zmodem_handle,9,"kermit");
    open_swindow(zmodem_handle,0,0);
    }

  if (setjmp(killk)==0)
    {
    switch(istate)
      {
      case 's':
        {
        batch_next(1);
        rx=FALSE;
        kerini();
        sendsw();
        break;
        }
      case 'v':
        {
        rx=TRUE;
        kerini();
        recsw(rxpath);
        break;
        }
      case 'g':
      case 'r':
        {
        rx=TRUE;
        kerini();
        sendcmdsw(toupper(istate),s1);
        break;
        }
      }
    }

  _fclose(&bbs_file[4]);

  if (istate!='v' && istate!='r' && istate!='s')
    {
    win_register_event_handler(zmodem_handle,0,0);
    wimp_close_wind(zmodem_handle);
    }

  return(1);
  }
#else
int kermittx(char *batchfilename)
  {
  files_done=0;

  if ((bbs_file[3]=fopen(batchfilename,"r"))==NULL) return(FILE_ERROR);

  rx=FALSE;
  kerini();
  sendsw();

  _fclose(&bbs_file[4]);
  _fclose(&bbs_file[3]);

  return(files_done==0?CANCEL:DONE);
  }

int kermitrx(char *batchfilename)
  {
  files_done=0;

  if ((bbs_file[3]=fopen(batchfilename,"w"))==NULL) return(FILE_ERROR);

  rx=TRUE;
  kerini();
  recsw("");

  _fclose(&bbs_file[4]);
  _fclose(&bbs_file[3]);

  return(files_done==0?CANCEL:DONE);
  }
#endif

/* CRC generation tables */

static int crcta[16] = {0, 010201, 020402, 030603, 041004,
  051205, 061406, 071607, 0102010, 0112211, 0122412, 0132613, 0143014,
  0153215, 0163416, 0173617};

static int crctb[16] = {0, 010611, 021422, 031233, 043044,
  053655, 062466, 072277, 0106110, 0116701, 0127532, 0137323, 0145154,
  0155745, 0164576, 0174367};

/*  C H K 1  --  Compute a type-1 Kermit 6-bit checksum.  */

int chk1(char *pkt)
  {
  int chk=chk2(pkt);

  chk = (((chk&0300)>>6)+chk)&0x3f;
  return(chk);
  }

/*  C H K 2  --  Compute the numeric sum of all the bytes in the packet.  */

int chk2(char *pkt)
  {
  int chk=0; 

  while(*pkt) chk=(chk+(*pkt++))&0xfff;
  return(chk);
  }


/*  C H K 3  --  Compute a type-3 Kermit block check.  */
/*
 Calculate the 16-bit CRC-CCITT of a null-terminated string using a lookup
 table.  Assumes the argument string contains no embedded nulls.
*/

int chk3(char *pkt)
  {
  int c, crc;

  for(crc=0;*pkt;pkt++)
    {
    c=*pkt^crc;
    crc = (crc>>8)^(crcta[(c&0xf0)>>4]^crctb[c&0xf]);
    }
  return(crc&0xffff);
  }

/*                _________________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Mail data header file                       <]
Current version   [> 00.21                                       <]
Version date      [> 18-January-1990                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/1990 by  <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "fidomail.h"

/****** Mail record structure */

typedef struct new_mail_block
  {
  /*** Record info */
  int  status;              /* Record status: should always be 1=
                               mail message header */
  int  message_number;      /* Message number of this header */
  int  message_area;        /* Message area of this message
                                 Bytes 0,1=message area
                                 Byte    2=0x00 Internal message
                                           0x01 FidoNet         */
  int  message_length;      /* Length of message (bytes) */

  /*** From/to information */
  union from
    {
    _address node;          /* Node from */
    int user;               /* Usernumber from */
    };       
  union to
    {
    _address node;          /* Node to */
    int user;               /* Usernumber to */
    };

  /*** Sent/expiry date */
  time_t date_sent;         /* Date sent */
  time_t date_expiry;       /*      of message expiry */

  /*** Message data link */
  int  block_forward;       /* Forward block link */

  /*** Message pointer link (mail only) */
  int  message_forward;     /* Forward message number link (-1=end) */
  int  message_backward;    /* Backward message number link (-1=end) */

  /*** Message reply links */
  int  reply_from;          /* Initial msg # in reply chain, -1 if not a
                               reply */

  /*** Message statistics */
  int  flags;               /* Message flags */
  int  noof_reads;          /* Number of times read */
  int  noof_replies;        /* Number of replies */

  /*** File attach info */
  int  file_length;         /* Length of file attach (if any) */
  int  file_location;       /* Location of file attach */

  /*** Reply data */
  int  replies[8];          /* Stores message numbers of most recent 8
                                                              replies */
  /*** Message voting data */
  int  votes_for;           /* Votes for subject */
  int  votes_against;       /* Votes against subject */

  char from[36];            /* From username / id # */
  char to[36];              /* To username / id # */
  char subject[61];         /* Subject field/filename (on file attach) */
                            /* Messages imported from FidoNet have their
                               subjects truncated... sorry FTSC! */
  char fill[6];             /* Fill to 256 bytes */

  } new_mail_block;         /* length =  256 */

/****** Mail data structure */

typedef struct mail_data
  {  
  /*** Link information */
  int  block_backward;      /* Backward block link */
  int  block_forward;       /* Forward block link */

  /*** Record info */
  char status;              /* Record status: should always be 2=
                               mail datablock */

  /*** Message data */
  char data[247];           /* 247 message bytes, null terminates message
                               on last block */

  } mail_data;         /* length = 256 */

/****** Message area header */

typedef struct mail_area
  {
  /*** Pointer to 1st message in area */
  int  first_message;       /* -1 if none */
  int  last_message;        /* -1 if none */
  time_t newest;            /* Date newest message sent */
  int  count;               /* Number of messages in area */
  int  level;               /* Security level to access area */
  int  flags;               /* Message flags needed */
  int  user_id;             /* Last person to update area */
  int  editor_id;           /* ID of area editor */
  char name[61];            /* Name of area */
  char user[31];            /* Last person to update area */
  char fill[4];             /* Make upto 128 bytes */
  } mail_area;
                            
#define MSG_CONFIRM              0x00000001 /* Recorded delivery */
#define MSG_VOTE                 0x00000002 /* Voting message */
#define MSG_SYSOP                0x00000004 /* For Sysop's eyes only */
#define MSG_EXPRESS              0x00000008 /* Express to front of queue */
                         
/* Fido mail flags */
#define MSG_PRIVATE              0x00010000
#define MSG_CRASH                0x00020000
#define MSG_RECD                 0x00040000
#define MSG_SENT                 0x00080000
#define MSG_FILEATTACHED         0x00100000
#define MSG_INTRANSIT            0x00200000
#define MSG_ORPHAN               0x00400000
#define MSG_KILLSENT             0x00800000
#define MSG_LOCAL                0x01000000
#define MSG_HOLDFORPICKUP        0x02000000
/* Unused flag here */
#define MSG_FILEREQUEST          0x08000000
#define MSG_RETURNRECEIPTREQUEST 0x10000000
#define MSG_ISRETURNRECEIPT      0x20000000
#define MSG_AUDITREQUEST         0x40000000
#define MSG_FILEUPDATEREQ        0x80000000

#define FILE_BINARY              0x00000001 /* File is binary & not ARC */
#define FILE_PRIVATE             0x00000002 /* File is private (user<>user) */
#define FILE_ARC                 0x00000004 /* File is an ARCfile */
#define FILE_SYSOP               0x00000008 /* For Sysop's eyes only */

#define FLAG_FILEAREA            0x80000000

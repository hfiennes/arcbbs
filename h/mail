/*                _________________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Mail data header file                       <]
Current version   [> 00.27                                       <]
Version date      [> 27-December-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "fidomail.h"

/****** Mail record structure */

typedef struct
  {
  /*** Record info */
  int  status;              /* Record status: should always be 1=
                               mail message header */
  int  message_number;      /* Message number of this header */
  int  message_area;        /* Message area of this message */
  int  message_length;      /* Length of message (bytes) */

  /*** From/to information */
  int  from_id;             /* Who from */
  int  to_id;               /* Who to */
                            /* Both 0 in the case of FidoNet netmail */
  /*** Sent/expiry date */
  time_t date_sent;         /* Date sent */
  time_t date_entered;      /*      message arrived on this system */

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
  int  replies[8];          /* Stores message number of most recent 8
                                                              replies */
  /*** Message voting data */
  int  votes_for;           /* Votes for subject */
  int  votes_against;       /* Votes against subject */

  char from[31];            /* From username / id # */
  char to[31];              /* To username / id # */
  char subject[61];         /* Subject field/filename (on file attach) */
  char align[1];            /* To align */

  _address_arch2 fidofrom;  /* Who from */
  _address_arch2 fidoto;    /* Who to */

  char fill[8];             /* Fill to 256 bytes */

  } mail_block;             /* length =  256 */

/****** Mail data structure */

typedef struct
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

typedef struct
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
  int  areaflags;           /* Flags for area */
  } mail_area;

/****** New message format bits */

/****** Each area is organised like this:
 ******
 ****** a/rchimedes/subconf/subconf:
 ******   Info            - Area name, number of entries,
 ******                     last entry date, area type, etc
 ******   Index           - For every message header in lookup:
 ******                     id/realmsgnumber/flags/date/crc from/
 ******                     crc to/crc subject
 ******   Members         - 4 bytes/member - 0=not joined, otherwise
 ******                     last message read.
 ******   <_Files>        - Files associated with area. If area is
 ******                     a file area then this is where the files
 ******                     go, otherwise it can be used for linking.
 ******
 ****** <Areas> - Central areas information          
 ******   Where           - List of all conferences and pathnames
 ******   Conferences     - Infoblocks for all conferences
 ******/

/****** Info file structure */
typedef struct
  {
  char area_name[44];       /* Name of area */
  int  noof_entries;        /* Number of messages */
  int  size_used;           /* Bytes used in text file */
  time_t lastupdated;       /* When last write to area was */
  time_t created;           /* When created */
  char fill[452];           /* 452 bytes fill */
  } mail_areainfo;          /* length = 512 */
              
/****** 'Who' information structure */

typedef struct
  {
  int  address_type;        /* Type of address bits 0-7:
                               0x00 = Internal user to this system
                               0x01 = PortNet network user
                               0x02 = FidoNet network user
                               0x03 = UUCP network user

                               Address flag bits 8-31 (unused) */
  int  address_data[8];     /* 8x32 bit words as numeric address
                               data */

  char address_text[124];   /* Mail address */
  char address_name[32];    /* Real name of the bod! */
  } who;                    /* length = 192 */
  

/****** Lookup record structure */

typedef struct
  {
  /*** Record info */
  int  id;                  /* Unique message ID */
  int  message_number;      /* Message number in area */
  int  length_text;         /* Length of text part of message */
  int  length_binary;       /* Length of binary part of message */
  int  textpointer;         /* Pointer to message text */
  int  flags;               /* Message flags */

  /*** From/to information */
  who  from;                /* Who from */
  who  to;                  /* Who to */

  /*** Sent/expiry date */
  time_t date_sent;         /* Date sent */
  time_t date_entered;      /* Date message actually entered */
  time_t date_expiry;       /*      of message expiry */

  /*** Message reply links */
  int  reply_to;            /* Initial msg ID in reply chain, -1 if not a
                               reply */

  /*** Message statistics */
  int  noof_reads;          /* Number of times read */
  int  noof_replies;        /* Number of replies */

  /*** Message subject */
  char subject[80];         /* Subject field */
                                              
  /*** Original area name */
  char original_area[44];   /* Name of where originally posted */
  char area_name[44];       /* Name of current area */

  /*** Filling to 1024 bytes */
  char fill[424];
  } mail_header;            /* length = 1024 */
             
/****** Index structure (when cached with area_open) */

typedef struct
  {               
  int  id;                  /* Unique message ID (position in lookup file) */
  int  message_number;      /* Message number in this area */
  int  flags;               /* Message flags */
  int  replyto;             /* Message number this is a reply to */
  int  date_sent;           /* When sent */
  int  crc_from;            /* 32-bit CRCs */
  int  crc_to;
  int  crc_subject;
  } mail_datemap;           /* length = 32 */
      
#define WHO_ISLOCAL              0
#define WHO_ISPORTNET            1
#define WHO_ISFIDO               2
#define WHO_ISUUCP               3

#define MSG_CONFIRM              0x00000001 /* Recorded delivery */
#define MSG_VOTE                 0x00000002 /* Voting message */
#define MSG_SYSOP                0x00000004 /* For Sysop's eyes only */
#define MSG_EXPRESS              0x00000008 /* Express to front of queue */
#define MSG_FIDO                 0x00000010 /* Fidonet message */
#define MSG_READ                 0x00000020 /* Message has been read */
#define MSG_OUTBASKET            0x00000040 /* This is an outbasket copy */
#define MSG_DOWNLOADED           0x00000080 /* Message has been downloaded */
                                          
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
#define MSG_UNUSED               0x04000000 /* Unused flag */
#define MSG_FILEREQUEST          0x08000000
#define MSG_RETURNRECEIPTREQUEST 0x10000000
#define MSG_ISRETURNRECEIPT      0x20000000
#define MSG_AUDITREQUEST         0x40000000
#define MSG_FILEUPDATEREQ        0x80000000

#define FILE_BINARY              0x00000001 /* File is binary & not ARC */
#define FILE_PRIVATE             0x00000002 /* File is private (user<>user) */
#define FILE_ARC                 0x00000004 /* File is an ARCfile */
#define FILE_SYSOP               0x00000008 /* For Sysop's eyes only */

#define FLAG_FILEAREA            0x00000001
#define FLAG_ECHOMAIL            0x00000002
#define FLAG_WRITEABLE           0x00000004
#define FLAG_READABLE            0x00000008
#define FLAG_AUTOINVISIBLE       0x00000010
#define FLAG_FREEDOWNLOAD        0x00000020
#define FLAG_FREEUPLOAD          0x00000040

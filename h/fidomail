/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> FidoMail : Packed mail format               <]
                  [> Based on FSC-0001 - Revision 15 (Zone info) <]
Current version   [> 00.08                                       <]
Version date      [> 04-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989/90/91/92   <]
                  [>     by Hugo Fiennes of The Serial Port      <]
                  [>_____________________________________________<]
*/

/****** FidoNet address structure */

typedef struct
  {
  /*** Address: ALWAYS STORE IN LO-HI FORMAT, EXCEPT FOR TYPE 3 BUNDLES */
  char zone[2],net[2],node[2],point[2];
  } _address;

typedef struct
  {
  int zone,net,node,point;
  } _address_arch;

typedef struct
  {
  short zone,net,node,point;
  } _address_arch2;

/****** Packed message format (note shorts are lo-hi here!) */

typedef struct
  {
  char              hdr_0;            /* Always 0x02 */
  char              hdr_1;            /* Always 0x00 */
  char              orignode[2];      /* Originating node */
  char              destnode[2];      /* Destination node */
  char              orignet[2];       /* Originating net */
  char              destnet[2];       /* Destination net */
  char              attribute[2];     /* Attributes */
  char              cost[2];          /* Cost */
  char              datetime[20];     /* date/time stamp */
  char              text[1];          /* To/From/Subject/Text */
  } _packed_message;
                             
/* Datetime is of format '02 Feb 90  12:10:14' */

/****** Packet header */

typedef struct
  {
  char              orignode[2];      /* Originating node */
  char              destnode[2];      /* Destination node */
  char              year[2];
  char              month[2];
  char              day[2];
  char              hour[2];
  char              minute[2];
  char              second[2];        /* Time of packet */
  char              baud[2];          /* Baudrate?! */
  unsigned char     h_0;              /* Always 0x02 */
  unsigned char     h_1;              /* Always 0x00 */
  char              orignet[2];       /* Originating net */
  char              destnet[2];       /* Destination net */
  unsigned char     productcodeL;     /* 0 for Fido & SEA (low byte) */
  unsigned char     serialnoH;        /* binary serial no (or null) */
  unsigned char     pwd[8];           /* password (or null) */
  char              qorigzone[2];     /* Originating zone */
  char              qdestzone[2];     /* Destination zone */
  char              filler[4];
  char              productcodeH;     /* Product code (hi byte) */
  char              serialnoL;        /* Product rev (low byte) */
  char              capword[2];       /* Capabilities */
  char              origzone[2];      /* Originating zone */
  char              destzone[2];      /* Destination zone */
  char              origpoint[2];     /* Originating point */
  char              destpoint[2];     /* Destination point */
  char              proddata[4];      /* Product data */
  /* Followed by zero or more packed messages */
  /* terminated by 2 zero bytes */
  } _packet_header;

/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> FidoMail : Mail bundle/Packed mail format   <]
                  [> Based on FSC-0014, 'A bundle proposal'      <]
                  [> January 1988 by Wynn Wagner III             <]
                  [> Also on FSC-0001 for fallback packing       <]
Current version   [> 00.04                                       <]
Version date      [> 25-January-1990                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/1990 by  <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#define BUNDLEVER 3  /* Bundle version */
#define EIDSIZ    (sizeof(struct _address)+sizeof(unsigned long))
                            
/* Bundle subpacket types */
#define END_SIGNAL      0
#define AREA_HEADER     1
#define MESSAGE_HEADER  2
#define TEXT            3
#define ECHOMAIL_INFO   4
/* 128<=MISCINFO<=256 */
                
/****** Kludges for byte aligned PC words/longs */

typedef struct PC_INT  { unsigned char data[2]; } PC_INT;
typedef struct PC_LONG { unsigned char data[4]; } PC_LONG;

/****** FidoNet address structure */

struct _address
  {
  /*** Address: ALWAYS STORE IN HI-LO FORMAT: use SWAP_W() to translate */
  char zone[2],net[2],node[2],point[2];
  } _address;

struct _address_arch
  {
  int zone,net,node,point;
  } _address_arch;

/****** Bundle header */

struct _bundleheader
  {
  struct   _address B_destination;    /* Where bundle is headed */
  struct   _address B_origination;    /* Where bundle is from */
  char              B_local1[2];      /* used by bundler */
  char              B_version[2];     /* Always 3 */
  char              B_creationtime[4];/* Unix-type stamp (ie time()) */
  char              B_bundlermajor[2];/* Major version of bundler */
  char              B_bundlerminor[2];/* Minor version of bundler */
  unsigned char     B_password[9];    /* 0 padded password: uppercase alpha only */
  unsigned char     B_local2;         /* Local use only */
  unsigned char     B_product[4];     /* Product code of bundler */
  unsigned char     B_FTSC[4];        /* Reserved by FTSC */
  } _bundleheader;
        
/****** Bundle footer */

struct _bundleend
  {
  unsigned char     M_Version;        /* Always 3                    */
  unsigned char     M_PacketType;     /* Always 0                    */
  } _bundleend;

/****** Message area header */

struct _areaheader
  {
  unsigned char     E_Version;        /* Always 3                    */
  unsigned char     E_PacketType;     /* Always 1                    */
  unsigned char     E_NameLength;     /* Actual bytes in E_Name      */
  unsigned char     E_Name[1];        /* VARIABLE-length field max63 */
  /* E_NAME can contain UPPERCASE characters, digits and $.-_&#@! */
  } _areaheader;

/****** Message header */

struct _messageheader
  {
  unsigned char     M_Version;        /* Always 3                    */
  unsigned char     M_PacketType;     /* Always 2                    */
  struct _address   M_destination;    /* FINAL Destination           */
  struct _address   M_origination;    /* Where message was entered   */
  char              M_CreationTime[4];/* Unix-type stamp (1970-base) */
                                      /* Kludge for word-boundaries  */
  char              M_attributes[2];  /* Standard Fidonet bitweights */
  unsigned char     M_FromLength;     /* Number of bytes in FROM     */
  unsigned char     M_ToLength;       /* Number of bytes in TO       */
  unsigned char     M_SubjectLength;  /* Number of bytes in SUBJECT  */
  char              M_FromToSubject[1];
  /* Followed by FROM, TO & SUBJECT (remember no terminations) */
  } _messageheader;

/****** Message text */

struct _text
  {
  unsigned char    T_Version;         /* Always 3                    */
  unsigned char    T_PacketType;      /* Always 3                    */
  char             T_ByteCount[2];    /* Number of bytes ( <0x1000)  */
  unsigned char    T_Data[1];         /* VARIABLE-length field       */
  /* Note: max is 4096 characters per text block: Text may contain:

        0x02  :  Material from here to next <lf> is a quote from the
                 parent message.
        0x0a  :  Forced <crlf>
        0x10  :  Replicate (0x10 <char> <nooftimes>)
  */
  } _text;

/****** Echomail packet */

struct _echomailinfo
  {
  unsigned char     EI_Version;       /* Always 3                    */
  unsigned char     EI_PacketType;    /* Always 4                    */
  unsigned char     EI_Parent[EIDSIZ];/* "up" message thread         */
  unsigned char     EI_Child[EIDSIZ]; /* "down" message thread       */
  char              EI_SeenbyCount[2];/* Number of SEENBY items      */
  struct _address   EI_Seenby[1];     /* VARIABLE-length field       */
  /* EI_parent & EI_child contain: _address followed by (PC_LONG) time_t
     stamp to help message reply linking */
  } _echomailinfo;


/****** Miscinfo packet */

struct _miscinfo
  {
  unsigned char     MI_Version;       /* Always 3                    */
  unsigned char     MI_PacketType;    /* 0x80-0xff, assigned by FTSC */
  unsigned char     MI_ByteCount;     /* Size of miscinfo data       */
  unsigned char     MI_WhoKnows;      /* Miscellaneous stuff         */
  } _miscinfo;

/****** Packed message format (note shorts are lo-hi here!) */

struct _packed_message
  {
  unsigned char     hdr_0;            /* Always 0x02 */
  unsigned char     hdr_1;            /* Always 0x00 */
  PC_INT            orignode;         /* Originating node */
  PC_INT            destnode;         /* Destination node */
  PC_INT            orignet;          /* Originating net */
  PC_INT            destnet;          /* Destination net */
  PC_INT            arrtibute;        /* Attributes */
  PC_INT            cost;             /* Cost */
  unsigned char     datetime[20];     /* date/time stamp */
  unsigned char     to_username[36];  /* Who to */
  unsigned char     from_username[36];/* Who from */
  unsigned char     subject[72];      /* Msg subject */
  unsigned char     text[1];          /* Text: null term */
  } _packed_message;

/****** Packet header */

struct _packet_header
  {
  PC_INT            orignode;         /* Originating node */
  PC_INT            destnode;         /* Destination node */
  PC_INT            year;
  PC_INT            month;
  PC_INT            day;
  PC_INT            hour;
  PC_INT            minute;
  PC_INT            second;           /* Time of packet */
  PC_INT            baud;             /* Baudrate?! */
  unsigned char     h_0;              /* Always 0x02 */
  unsigned char     h_1;              /* Always 0x00 */
  PC_INT            orignet;          /* Originating net */
  PC_INT            destnet;          /* Destination net */
  unsigned char     productcode;      /* 0 for Fido & SEA */
  unsigned char     fill[33];
  /* Followed by zero or more packed messages */
  /* terminated by 2 zero bytes */
  } _packet_header;

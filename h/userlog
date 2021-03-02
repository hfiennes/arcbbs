/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Userlog data header file                    <]
Current version   [> 00.28                                       <]
Version date      [> 23-August-1990                              <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/1990 by  <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/


/****** Userlog record structure */

typedef struct
  {
  /*** Record info */
  int  status;                /* Record status: should always be 1=
                                 user information header */

  /*** Local user identification information */
  int  usernumber;            /* User number */
  int  passcrc;               /* Passphrase CRC */

  /*** Local mail information */
  int  mailpointer_start;     /* Pointer to start of mail chain, -1=no mail */
  int  mailpointer_end;       /* Pointer to end */
  int  mailcount;             /* Count of number of mail messages */

  /*** Time information */
  time_t t_firstlogon;        /* Time of user's first logon */
  time_t t_lastlogon;         /* Time of user's last logon */
  int  m_message;             /* Highest msg nr read */
  int  m_file;                /* Highest file nr read */

  /*** User terminal information */
  int  termtype;              /* Terminal type supported */
  
  /*** User priv information */
  int  f_message;             /* Message flags */
  int  f_file;                /* File flags */
  int  f_user;                /* User flags */
  int  padsize;               /* Maximum allowed scratchpad size */
  int  ratio;                 /* Up/download ratio */
  int  userlevel;             /* Userlevel */
  int  logons;                /* Nunber of logons */
  int  uploads;               /* Number of uploads */
  int  downloads;             /* Number of downloads */
  int  timeallowed;           /* Time allowed online */
  int  timetoday;             /* Amount of time used today */

  /*** Other user information stored in userlog as linked-list */
  int  morepointer;           /* -1=no more information */

  char username[31];          /* Username */
  char realname[31];          /* Real name */
  char address[4][31];        /* User's address */
  char postcode[11];          /* Postcode */
  char telephone[31];         /* Telephone number */
  char callrate;              /* Call rate */
              
  char pagelen;               /* Page length */
  char fill1[2];              /* Fill to word-align */

  unsigned char conferences[64]; /* Bitmap of joined conferences */
                       
  int  netcredit;             /* Fidonet credit */
  int  fidoflags;             /* Fidonet flags */

  int  conference;            /* Current conference */
  int  filebase;              /* Current filebase */

  int  outbox_start;          /* Outbox start pointer */
  int  outbox_end;            /* Outbox end pointer */
  int  outboxcount;           /* Outbox mail count */
  char fill[96];              /* Fill to 512 bytes */
  } user_block;               /* Done! */

typedef struct
  {
  /*** Forward/backward pointers, -1 for non-existant */
  int  forward;               /* Forward pointer */
  int  backward;              /* Backward pointer */

  char status;                /* Record status: should always be 2=
                                 user datablock */

  /*** User data block, freeform */
  char data[503];             /* The actual data */

  } user_data;                /* 512 bytes long */

/****** New userlog record structure */

typedef struct
  {
  /*** Record info */
  int  status;                /* Record status: should always be 1=
                                 user information header */

  /*** Local user identification information */
  int  usernumber;            /* User number */
  int  passcrc;               /* Passphrase CRC */

  /*** Local mail information */
  int  mailcount;             /* Count of number of mail messages */
  int  outboxcount;           /* Outbox mail count */

  /*** Time information */
  time_t t_firstlogon;        /* Time of user's first logon */
  time_t t_lastlogon;         /* Time of user's last logon */
  int  logons;                /* Nunber of logons */

  /*** User terminal information */
  int  termtype;              /* Terminal type supported */
  int  pagelen;               /* Page length */
  
  /*** User priv information */
  int  f_message;             /* Message flags */
  int  f_file;                /* File flags */
  int  f_user;                /* User flags */
  int  padsize;               /* Maximum allowed scratchpad size */
  int  userlevel;             /* Userlevel */

  int  ratio;                 /* Up/download ratio */
  int  ratiok;                /* Ratio in kb */
  int  uploads;               /* Number of uploads */
  int  uploadk;               /* kBytes of uploads */
  int  downloads;             /* Number of downloads */
  int  downloadk;             /* kBytes of downloads */

  int  timeallowed;           /* Time allowed online */
  int  timetoday;             /* Amount of time used today */

  int  netcredit;             /* Fidonet credit */
  int  fidoflags;             /* Fidonet flags */

  unsigned char conferences[64]; /* Bitmap of joined conferences */
  int  conference;            /* Current conference */
  int  filebase;              /* Current filebase */

  char redirect_type;

  char username[31];          /* Username */
  char realname[31];          /* Real name */
  char address[4][31];        /* User's address */
  char postcode[11];          /* Postcode */
  char telephone[31];         /* Telephone number */
  char callrate;              /* Call rate */

  char fill[111];
  } user_blocknew;              /* Done! */
/****** Userlist structure */

typedef struct
  {
  /*** User info */
  int  baudrate,usernumber;
  char username[31];
  char doing[18];
  char fill[7];
  } user_list; /* 64 bytes */

/****** User hash table structure */

typedef struct
  {
  /*** Record info */
  int  status;                /* Record status, 0=free, 1=deleted, 2=data */
  int  recordpointer;         /* Pointer to userlog record */

  /*** Record data */
  char username[31];          /* Username of hashed user */

  char data[25];              /* Padding to 64 bytes */
  } user_hash;                /* 64 bytes */

#define TER_TTY                     0  /* TTY codes */
#define TER_VT52                    1  /* VT52 codes */
#define TER_VT100                   2  /* VT100 codes */
#define TER_ANSI                    3  /* ANSI codes */

#define USER_FILEPROCESS   0x00000001  /* Can do fileprocess */
#define USER_MUCHAT        0x00000002  /* Can participate in MU chat */
#define USER_PAGE          0x00000004  /* Can page sysop */
#define USER_NOVICE        0x00000008  /* Novice menus */
#define USER_VOTED         0x00000010  /* Has voted */
#define USER_REGISTERED    0x00000020  /* Is a registered user */
#define USER_SYSOP         0x00000040  /* Is a sysop */
#define USER_RESETTIME     0x00000080  /* Fresh limit each logon */
#define USER_OUTBOX        0x00000100  /* Outbox enabled */

#define USER_NOPAGE        0x08000000  /* No paging for MU chat */
#define USER_NOCLEAR       0x10000000  /* Clear screens? */
#define USER_NOMORE        0x20000000  /* More? prompting */
#define USER_DYNAMIC       0x40000000  /* Dynamic screen addressing */

/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Header for Fido                             <]
Current version   [> 00.10                                       <]
Version date      [> 29-November-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/
            
extern void fido_doinbound(char *);
extern int  fido_dooutbound(void),fido_setup(void),
            fido_domail(mail_block*,int);

extern void put_rint(int,void*),put_int(int,void*),put_rlong(int,void*),
            put_long(int,void*),put_zeropad(char*,int,char*),
            fido_dearc(void);
extern char *put_shortstring(char*,char*),*put_longstring(char*,char*),
            *put_noterm(char*,char*);
extern int  get_rint(void*),get_int(void*),get_rlong(void*),get_long(void*);
                         
/*--------------------------------------------------------------------------*/
/* QuickBBS 2.00 QNL_IDX.BBS                                                */
/* (File is terminated by EOF)                                              */
/*--------------------------------------------------------------------------*/

typedef struct
  {
  char QI_Zone[2];
  char QI_Net[2];
  char QI_Node[2];
  char QI_NodeType;
  } NodelistIDX;

/*--------------------------------------------------------------------------*/
/* QuickBBS 2.00 QNL_DAT.BBS                                                */
/* (File is terminated by EOF)                                              */
/*--------------------------------------------------------------------------*/

typedef struct
  {
  char QL_NodeType;
  char QL_Zone[2];
  char QL_Net[2];
  char QL_Node[2];
  char QL_Name[21];                             /* Pascal! 1 byte count, up
                                                  * to 20 chars */
  char QL_City[41];                             /* 1 + 40 */
  char QL_Phone[41];                            /* 1 + 40 */
  char QL_Password[9];                          /* 1 + 8 */
  char QL_Flags[2];                             /* Same as flags in new
                                                 * nodelist structure */
  char QL_BaudRate[2];
  char QL_Cost[2];
  } NodelistDAT;

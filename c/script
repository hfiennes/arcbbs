/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 05-March-1990                               <]
                  [>                                             <]
Module name       [> Script language interpreter                 <]
Current version   [> 00.06                                       <]
Version date      [> 19-February-1993                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1990-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#define MAXNEST 32
#define DELIMITER 1
#define VARIABLE  2
#define NUMBER    3

#define OP_LEFTSHIFT          128
#define OP_RIGHTSHIFT         129
#define OP_AND                130
#define OP_OR                 131
#define OP_NOTEQUAL           132
#define OP_EQUAL              133
#define OP_LESSTHANOREQUAL    134
#define OP_GREATERTHANOREQUAL 135

#include "include.h"
#include "script.h"
#include "scriptc.h"
#include "port.h"
#include "msgs.h"
#include "str.h"
#include "config.h"

#include "userlog.h"
#include "mail.h"

#include "bbs.h"

extern void window_poll(void);
extern void eval_exp(char*,int*);
extern char *pathonly(char*);

char    *gblock;        /* Where we are executing */
char    *cline,*cprog;  /* Current line (used to find line number) */
int     rcode;          /* Temporary return code */
jmp_buf script_kill;    /* To abort a script run */
int     script_cankill; /* We can abort a script */
int     script_stop;   

extern int lastpoll;
static char operators[]= " +-/*%^=<>()!&|~";       /* recoginised operators */
static char *sexp;

/* Script time */
struct tm script_time;

/* Int variables - pointers to MAXNEST levels of variables */
script_variable *script_variables[MAXNEST];
int script_variables_count[MAXNEST];
char *script_stringarea[MAXNEST];
int script_stringcount[MAXNEST];
int *script_integerarea[MAXNEST];
int script_integercount[MAXNEST];

/* Script nesting level & key struff*/
int  script_level=0,script_labelcount,script_functioncount,
     script_boot,boot_length,deb=0;
char *boot_memory,script_chain[128],script_name[128];

/* Script labels */
script_labelstr *script_labels;

/* Script files */
FILE *script_file[4];

/* Function pointers */
script_functionstr *script_functions;

/* Library file */
char *library_script,*driver_script;

char *global_script;
int global_len;

/* Separate function */
void _separate(char **hm)
  {
  int inquote=0,open=0,open2=0;
  char *howmuch=*hm;
   
  if (*howmuch==0) return;

  /* Separate */
  while((inquote!=0 || open!=0 || open2!=0 || *howmuch!=',') && *howmuch)
    {
    if (*howmuch=='\"') inquote=1-inquote;
    if (inquote==0)
      {
      if (*howmuch=='(') open++;
      if (*howmuch==')') open--;
      if (*howmuch=='[') open2++;
      if (*howmuch==']') open2--;
      }
    howmuch++;
    }
  *hm=howmuch;
  }

void _separate2(char **hm)
  {
  int inquote=0,open=0,open2=0;
  char *howmuch=*hm;

  if (*howmuch==0) return;

  /* Separate */
  while((inquote!=0 || open!=0 || open2!=0 || *howmuch!=' ') && *howmuch)
    {
    if (*howmuch=='\"') inquote=1-inquote;
    if (inquote==0)
      {
      if (*howmuch=='(') open++;
      if (*howmuch==')') open--;
      if (*howmuch=='[') open2++;
      if (*howmuch==']') open2--;
      }
    howmuch++;
    }
  *hm=howmuch;
  }

void stopit()
  {
  script_stop=0;
  if (script_cankill)
    {
    script_cankill=0;
    longjmp(script_kill,1);
    }
  }

void afterfind(char **a)
  {
  int level=0;
  char *b=*a;

  while(*b==' ' || *b==0) b++;

  if (*b!='{')
    {
    /* Single command, skip to next line */
    while(*b++);
    *a=b;
    return;
    }

  do
    {
    if (*b=='{') level++;
    if (*b=='}') level--;
    while(*b++);
    }
  while(level!=0);

  *a=b;
  }

/* Initialise */
int script_initialise()
  {
  int a;

  script_cankill=0; *script_chain=0;
  library_script=0;

  /* Initial nesting level */
  script_level=0;

  /* Setup variable pointers: 0 is global, MAXNEST-1 is most volatile */
  for(a=0;a<MAXNEST;a++)
    {
    script_variables[a]=NULL;
    script_variables_count[a]=0;
    script_stringarea[a]=NULL;
    script_stringcount[a]=0;
    script_integerarea[a]=NULL;
    script_integercount[a]=0;
    }

  /* Zero file info */
  for(a=0;a<4;a++) script_file[a]=0;

  /* Setup first 2 variables, '__VERSION' & '__VERSIONSTRING' */
  script_add_i_variable("__VERSION",0,1,7);
  script_add_s_variable("__VERSIONSTRING",0,1,4,"1.63");

  /* Initialise label stuff */
  if ((script_labels=malloc(sizeof(script_labelstr)))==NULL)
    return(0);
  script_labelcount=0;

  /* Initialise function pointer stuff */
  if ((script_functions=malloc(sizeof(script_functionstr)))==NULL)
    return(0);
  script_functioncount=0;

  /* Script library */
  if (script_includefile("<ARCserver$Dir>.Script.Library","Library",
                         &library_script)==NULL)
    {
    error("fError whilst including library");
    return(0);
    }

  /* Add pointers into userlog record */
  script_add_p_variable("user.status",0,1,0,&user.status,2);
  script_add_p_variable("user.usernumber",0,1,0,&user.usernumber,2);

  script_add_p_variable("user.mailpointer_start",0,1,0,&user.mailpointer_start,2);
  script_add_p_variable("user.mailpointer_end",0,1,0,&user.mailpointer_end,2);
  script_add_p_variable("user.mailcount",0,1,0,&user.mailcount,2);

  script_add_p_variable("user.t_firstlogon",0,1,0,&user.t_firstlogon,2);
  script_add_p_variable("user.t_lastlogon",0,1,0,&user.t_lastlogon,2);

  script_add_p_variable("user.m_message",0,1,0,&user.m_message,2);
  script_add_p_variable("user.m_file",0,1,0,&user.m_file,2);

  script_add_p_variable("user.termtype",0,1,0,&user.termtype,2);

  script_add_p_variable("user.f_message",0,1,0,&user.f_message,2);
  script_add_p_variable("user.f_file",0,1,0,&user.f_file,2);
  script_add_p_variable("user.f_user",0,1,0,&user.f_user,2);
  script_add_p_variable("user.padsize",0,1,0,&user.padsize,2);
  script_add_p_variable("user.ratio",0,1,0,&user.ratio,2);
  script_add_p_variable("user.userlevel",0,1,0,&user.userlevel,2);
  script_add_p_variable("user.logons",0,1,0,&user.logons,2);
  script_add_p_variable("user.uploads",0,1,0,&user.uploads,2);
  script_add_p_variable("user.downloads",0,1,0,&user.downloads,2);
  script_add_p_variable("user.timeallowed",0,1,0,&user.timeallowed,2);
  script_add_p_variable("user.timetoday",0,1,0,&user.timetoday,2);

  script_add_p_variable("user.username",0,1,30,user.username,3);
  script_add_p_variable("user.realname",0,1,30,user.realname,3);
  script_add_p_variable("user.address",0,4,30,user.address[0],3);
  script_add_p_variable("user.postcode",0,1,10,user.postcode,3);
  script_add_p_variable("user.telephone",0,1,30,user.telephone,3);
  script_add_p_variable("user.callrate",0,1,0,&user.callrate,4);

  script_add_p_variable("user.pagelen",0,1,0,&user.pagelen,4);

  script_add_p_variable("user.conference",0,1,0,&user.conference,2);
  script_add_p_variable("user.filebase",0,1,0,&user.filebase,2);

  /* Add system pointers */
  script_add_p_variable("system.portnumber",0,1,0,&portnumber,2);
  script_add_p_variable("system.baudrate",0,1,0,&baudrate,2);
  script_add_p_variable("system.arq",0,1,0,&arq,2);
  script_add_p_variable("system.conference",0,1,0,&conference,2);
  script_add_p_variable("system.filebase",0,1,0,&filebase,2);
  script_add_p_variable("system.scratchpadsize",0,1,0,&scratchpad,2);
  script_add_p_variable("system.files_done",0,1,0,&files_done,2);

  /* Queue info */
  script_add_p_variable("queue.length",0,1,0,&queue_length,2);
  script_add_p_variable("queue.filenumber",0,64,0,&download_queue[0],2);
  script_add_p_variable("queue.filesize",0,64,0,&download_queuesize[0],2);
  script_add_p_variable("queue.filename",0,64,20,download_queuename[0],3);

  /* Time breakdown */
  script_add_p_variable("time.seconds",0,1,0,&script_time.tm_sec,2);
  script_add_p_variable("time.minutes",0,1,0,&script_time.tm_min,2);
  script_add_p_variable("time.hours",0,1,0,&script_time.tm_hour,2);
  script_add_p_variable("time.month",0,1,0,&script_time.tm_mon,2);
  script_add_p_variable("time.year",0,1,0,&script_time.tm_year,2);
  script_add_p_variable("time.dayofweek",0,1,0,&script_time.tm_wday,2);
  script_add_p_variable("time.dayofmonth",0,1,0,&script_time.tm_mday,2);
  script_add_p_variable("time.dayofyear",0,1,0,&script_time.tm_yday,2);

  return(1);
  }

int script_compressbuffer(char *buffer,int length)
  {
  char *in=buffer,*out=buffer;
  int a,b,inquote;

  do
    {
    /* Find start of line */
    while(*in==' ') in++;
    if (*in==0)
      {
      *out++=*in++;
      continue;
      }

    if (*in!=';' && *in!='/')
      {
      /* Check for command on line, tokenise if necessary */
      a=0;
      while(script_command[a].command[0])
        {
        b=strlen(script_command[a].command);
        if ((in[b]=='(' || in[b]==' ' || in[b]==0) &&
            strnicmp(script_command[a].command,in,b)==0)
          {
          if (a<128) { *out++=128; *out++=128+a; }
          else { *out++=129; *out++=a; }
          in+=strlen(script_command[a].command);
          break;
          }
        else a++;
        }

      /* Copy line & tokenise too */
      inquote=0;
      while(*in)
        {
        if (inquote==0)
          {
          if (*in=='\"') inquote=1;
          if (inquote==0 && xstrchr("=(+-*/&|^ ",*in)!=NULL &&
              (isalpha(in[1]) || in[1]=='$'))
            {
            *out++=*in++;
            a=0;
            while(script_command[a].command[0])
              {
              b=strlen(script_command[a].command);
              if ((in[b]=='(' || in[b]==' ') &&
                  strnicmp(script_command[a].command,in,b)==0)
                {
                if (a<128) { *out++=128; *out++=128+a; }
                else { *out++=129; *out++=a; }
                in+=strlen(script_command[a].command);
                break;
                }
              else a++;
              }
            }
          else *out++=*in++;
          }
        else
          {
          if ((*out++=*in++)=='\"') inquote=0;
          }
        }

      /* Strip trailing spaces */
      if (*(out-1)==' ')
        {
        out-=2;
        while(*out==' ') out--;
        *++out=0;
        out++;
        }
      }
    else
      {    
      while(*in++);
      *out++=';';
      *out++=0;
      }
    }
  while((in-buffer)<length);
#ifdef DEBUGX
  { FILE *dumpt;
  if ((dumpt=fopen("tokeni","a"))!=NULL)
    {
    fwrite(buffer,out-buffer,1,dumpt);
    fclose(dumpt);
    }
  }
#endif

  return(out-buffer);
  }

int script_includefile(char *filename,char *tag,char **scrptr)
  {
  FILE *scriptfile;
  char *b,*end;
  int len;
           
  if ((scriptfile=fopen(filename,"r"))==NULL)
    {
    error("fCan't open include file %s",scriptfile);
    return(0);
    }

  fseek(scriptfile,0,SEEK_END);
  len=ftell(scriptfile);
  fseek(scriptfile,0,SEEK_SET);

  /* Get memory */
  if ((*scrptr=malloc(len))==NULL)
    {
    error("fNo memory to load include file");
    fclose(scriptfile);
    return(0);
    }

  /* Load file */
  fread(*scrptr,1,len,scriptfile);
  fclose(scriptfile);

  /* Convert file */
  b=*scrptr;
  end=*scrptr+len;
  while(b<end)
    {
    if (*b==10) *b=0;
    b++;
    }

  /* Remove spaces, ie compress buffer */
  len=script_compressbuffer(*scrptr,len);
  *scrptr=realloc(*scrptr,len);

  return(script_include(tag,*scrptr,len));
  }

void script_closedown(int complete)
  {
  int a;

  /* Free variable pointers */
  for(a=(complete?0:1);a<MAXNEST;a++)
    {
    script_level=a;
    script_unnest();
    }

  /* Free label stuff */
  if (complete)
    {
    free(script_labels);
    script_labels=NULL;
    }

  if (complete)
    {
    /* Free function pointers */
    free(script_functions);
    script_functions=NULL;

    if (library_script)
      {
      free(library_script);
      library_script=NULL;
      }

    if (driver_script)
      {
      free(driver_script);
      driver_script=NULL;
      }
    }

  script_level=0;
  }

/* Find a variable */
script_variable *script_findvariable(char *name,int *level)
  {
  int a,b; /* Search from most volatile -> global */

  for(a=(MAXNEST-1);a>=0;a--)
    {
    if (script_variables_count[a]!=0)
      {
      for(b=0;b<script_variables_count[a];b++)
        {
        if (stricmp(script_variables[a][b].name,name)==0)
          {
          *level=a;
          return(&script_variables[a][b]);
          }
        }
      }
    }
  return(NULL);
  }

/* Get an integer variable */
int script_get_i_variable(char *lname,int *result)
  {
  script_variable *var;
  int level,arraypos=0;
  char *ap,name[20];

  strcpy(name,lname);

  /* Find array position */
  if ((ap=xstrchr(name,'['))!=NULL)
    {
    *ap++=0;
    *(ap+strlen(ap)-1)=0;
    if (script_integer_evaluate(ap,&arraypos)==0) arraypos=0;
    }

  if ((var=script_findvariable(name,&level))!=NULL)
    {
    if (var->type==0) /* integer */
      {
      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      *result=script_integerarea[level][(var->value)+arraypos];
      return(1);
      }
    if (var->type==2) /* pointer to integer */
      {
      int *pt=(int*)var->value;

      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      pt=(int*)var->value;
      *result=pt[arraypos];
      return(1);
      }
    if (var->type==4) /* pointer to byte */
      {
      char *pt=(char*)var->value;

      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      *result=pt[arraypos];
      return(1);
      }
    }
  return(0);
  }

/* Put an integer variable */
int script_put_i_variable(char *lname,int value)
  {
  script_variable *var;
  int level,arraypos=0;
  char *ap,name[20];

  strcpy(name,lname);

  /* Find array position */
  if ((ap=xstrchr(name,'['))!=NULL)
    {
    *ap++=0;
    *(ap+strlen(ap)-1)=0;
    if (script_integer_evaluate(ap,&arraypos)==0) arraypos=0;
    }

  if ((var=script_findvariable(name,&level))!=NULL)
    {
    if (var->type==0) /* integer */
      {
      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      script_integerarea[level][(var->value)+arraypos]=value;
      return(1);
      }
    if (var->type==2) /* pointer to integer */
      {
      int *pt=(int*)var->value;

      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      pt[arraypos]=value;
      return(1);
      }
    if (var->type==4) /* pointer to byte */
      {
      char *pt=(char*)var->value;

      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      pt[arraypos]=value;
      return(1);
      }
    }
  return(0);
  }

/* Get a string variable */
int script_get_s_variable(char *lname,char *result)
  {
  script_variable *var;
  int arraypos=0,level;
  char *ap,name[20];

  strcpy(name,lname);
  /* Find array position */
  if ((ap=xstrchr(name,'['))!=NULL)
    {
    *ap++=0;
    *(ap+strlen(ap)-1)=0;
    if (script_integer_evaluate(ap,&arraypos)==0) arraypos=0;
    }

  if ((var=script_findvariable(name,&level))!=NULL)
    {
    if (var->type==1) /* string */
      {
      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      strcpy(result,script_stringarea[level]+var->value+(arraypos*(var->maxlen+1)));
      return(1);
      }
    if (var->type==3) /* pointer to string */
      {
      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      strcpy(result,((char*)var->value)+(arraypos*(var->maxlen+1)));
      return(1);
      }
    }
  return(0);
  }

/* Put a string variable */
int script_put_s_variable(char *lname,char *value)
  {
  script_variable *var;
  int arraypos=0,level;
  char *ap,name[20];

  strcpy(name,lname);
  /* Find array position */
  if ((ap=xstrchr(name,'['))!=NULL)
    {
    *ap++=0;
    *(ap+strlen(ap)-1)=0;
    if (script_integer_evaluate(ap,&arraypos)==0) arraypos=0;
    }

  if ((var=script_findvariable(name,&level))!=NULL)
    {
    if (var->type==1) /* string */
      {
      if (strlen(value)>var->maxlen)
        {
        /* Too long, truncate */
        *(value+var->type)=0;
        }
      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      strcpy(script_stringarea[level]+var->value+(arraypos*(var->maxlen+1)),value);
      return(1);
      }
    if (var->type==3) /* pointer to string */
      {
      if (strlen(value)>var->maxlen)
        {
        /* Too long, truncate */
        *(value+var->maxlen)=0;
        }
      if (arraypos>=var->arraysize) arraypos=var->arraysize-1;
      strcpy(((char*)var->value)+(arraypos*(var->maxlen+1)),value);
      return(1);
      }
    }
  return(0);
  }

void script_add_i_variable(char *name,int level,int arraysize,int value)
  {
  int entries=script_variables_count[level],a;

  if (script_variables_count[level]==NULL)
    {
    /* Allocate a bit of memory */
    script_variables[level]=malloc(sizeof(script_variable));
    script_variables_count[level]=1;
    }
  else
    {
    /* Allocate some more memory */
    script_variables[level]=realloc(script_variables[level],(entries+1)*sizeof(script_variable));
    script_variables_count[level]++;
    }

  strcpy(script_variables[level][entries].name,name);
  script_variables[level][entries].type=0;
  script_variables[level][entries].arraysize=arraysize;
  script_variables[level][entries].value=script_integercount[level];
  script_integercount[level]+=arraysize;

  if (script_integerarea[level]==NULL)
    {
    if ((script_integerarea[level]=malloc(script_integercount[level]*4))==NULL)
      {
      error("fLine %d: Can't allocate memory for integer variable %s[%d]",script_findline(cline),name,arraysize);
      stopit();
      return;
      }
    }
  else
    {
    if ((script_integerarea[level]=realloc(script_integerarea[level],script_integercount[level]*4))==NULL)
      {
      error("fLine %d: (extend) Can't allocate memory for integer variable %s[%d]",script_findline(cline),name,arraysize);
      stopit();
      return;
      }
    }

  for(a=0;a<arraysize;a++)
    {
    script_integerarea[level][script_variables[level][entries].value+a]=value;
    }
  }

void script_add_p_variable(char *name,int level,int arraysize,int maxlen,void *value,int type)
  {
  int entries=script_variables_count[level];

  if (script_variables_count[level]==NULL)
    {
    /* Allocate a bit of memory */
    script_variables[level]=malloc(sizeof(script_variable));
    script_variables_count[level]=1;
    }
  else
    {
    /* Allocate some more memory */
    script_variables[level]=realloc(script_variables[level],(entries+1)*sizeof(script_variable));
    script_variables_count[level]++;
    }

  strcpy(script_variables[level][entries].name,name);
  script_variables[level][entries].type=type;
  script_variables[level][entries].maxlen=maxlen;
  script_variables[level][entries].arraysize=arraysize;
  script_variables[level][entries].value=(int)value;
  }

void script_add_s_variable(char *name,int level,int arraysize,int maxlen,char *string)
  {
  int entries=script_variables_count[level],a;

  if (script_variables_count[level]==NULL)
    {
    /* Allocate a bit of memory */
    script_variables[level]=malloc(sizeof(script_variable));
    script_variables_count[level]=1;
    }
  else
    {
    /* Allocate some more memory */
    script_variables[level]=realloc(script_variables[level],(entries+1)*sizeof(script_variable));
    script_variables_count[level]++;
    }

  /* Setup variable */
  strcpy(script_variables[level][entries].name,name);
  script_variables[level][entries].type=1;
  script_variables[level][entries].maxlen=maxlen;
  script_variables[level][entries].arraysize=arraysize;
  script_variables[level][entries].value=script_stringcount[level];

  script_stringcount[level]+=(maxlen+1)*arraysize;
  if (script_stringarea[level]==NULL)
    {
    if ((script_stringarea[level]=malloc(script_stringcount[level]))==NULL)
      {
      error("fLine %d: Can't allocate memory for string variable %s[%d][%d]",script_findline(cline),name,arraysize,maxlen);
      stopit();
      return;
      }
    }
  else
    {
    if ((script_stringarea[level]=realloc(script_stringarea[level],script_stringcount[level]))==NULL)
      {
      error("Line %d: (extend) Can't allocate memory for string variable %s[%d][%d]",script_findline(cline),name,arraysize,maxlen);
      stopit();
      return;
      }
    }

  for(a=0;a<arraysize;a++)
    {
    strcpy(script_stringarea[level]+script_variables[level][entries].value+(a*(maxlen+1)),string);
    }
  }

/* Execute a single script command */
int script_executecommand(char *command,int *result1)
  {
  int a=0;
  char *tail;

  if ((clock()-lastpoll)>MAXIMUM_POLL) window_poll();

  tail=command;
  while(*tail)
    {
    if (*tail>=128) tail+=2;
    if (*tail=='(')
      {
      char *end;
      *tail++=0;
      end=tail+strlen(tail)-1;
      while(*end!=')' && *end) end--;
      if (*end!=')')
        {
        error("fLine %d: No close bracket in line: %s",script_findline(cline),command);
        stopit();
        return(0);
        }
      *end=0;
      while(*tail==' ' && *tail) tail++;
      break;
      }

    if (*tail==' ')
      {
      *tail++=0;
      while(*tail==' ' && *tail) tail++;
      break;
      }

    if (*tail=='=')
      {
      tail="";
      break;
      }
      
    if (*tail==0) break;

    tail++;
    }
             
  if (command[0]>=128) 
    {
    if (command[0]==128) a=command[1]-128; else a=command[1];
    goto gotcommand;
    }

  if ((a=script_findcommand(command,script_command[0].command))<0)
    {
    char wholestring[256],*value,*ws;
    int result;

    /* Check to see if it's a function */
    if (script_dofunction(command,tail,result1)) return(1);

    /* Check for variable assignment */
    ws=wholestring;
    while((*ws++=*command++)!=NULL); ws--;
    while((*ws++=*tail++)!=NULL);

    if ((value=xstrchr(wholestring,'='))!=NULL)
      {
      script_variable *dest; int junk;
      char *ap;

      *value++=0;

      /* Find array position */
      if ((ap=xstrchr(wholestring,'['))!=NULL) *ap=0;

      /* Set variable */
      dest=script_findvariable(wholestring,&junk);
      if (dest->type==0 || dest->type==2)
        {
        if (script_integer_evaluate(value,&result))
          {
          if (ap) *ap='[';
          if (script_put_i_variable(wholestring,result)) return(1);
          }
        }
      else
        {
        char res[256];

        if (script_string_evaluate(value,res))
          {
          if (ap) *ap='[';
          if (script_put_s_variable(wholestring,res)) return(1);
          }
        }
      }

    /* Is command 'break' (drop back to next level) */
    if (stricmp(command,"break")==0) return(2);

    return(0);
    }
    
  gotcommand:

  /* Parse arguments */
  if (script_command[a].params[0]==0)
    {
    /* Let function do it! */
    *result1=(*script_command[a].function)((void*)tail);
    }
  else
    {
    script_pass pass[8];
    char *next,line[256],s[2][256];
    int na=strlen(script_command[a].params)/2,b,su=0;
  
    strcpy(next=line,tail);

    for(b=0;b<na;b++)
      {
      int m=(script_command[a].params[b*2]=='m'),
          t=script_command[a].params[(b*2)+1];
      char *this=next;

      if (m && *this==0)
        {
        error("fLine %d: Missing parameter(s) %s(%s)",script_findline(cline),command,line);
        stopit();
        return(1);
        }

      if (m==0 && *this==0)
        {
        pass[b].p=0;
        }
      else
        {
        _separate(&next); if (*next) *next++=0;

        if (t=='s')
          {
          if (script_string_evaluate(this,s[su])==0)
            {
            if (command[0]=='$') *result1=(int)""; else *result1=0;
            return(1);
            }
          pass[b].d.s=s[su++];
          pass[b].p=1;
          }
        else
          {
          if (script_integer_evaluate(this,&pass[b].d.i)==0)
            {
            if (command[0]=='$') *result1=(int)""; else *result1=0;
            return(1);
            }
          pass[b].p=1;
          }
        }
      }
    /* Let function at it */
    *result1=(*script_command[a].function)(pass);
    }

  /* Is it a 'for' or other ones that can alter program pointer ? */
  if (script_command[a].type==1)
    {
    if (*result1<16)
      {
      *result1=rcode;
      return(3);
      }
    return(*result1);
    }

  /* A 'return' or '$return'? */
  if (script_command[a].type==2) return(3);
  return(1);
  }

int script_string_evaluate(char *toeval,char *result)
  {
  int ch,ch2,pra=-1;
  char operand[256],*op;

  while(*toeval==' ' && *toeval) toeval++;

  while((ch=*toeval)!=NULL)
    {
    if (ch==' ')
      {
      toeval++; continue;
      }

    op=operand;

    /* If variable/constant, copy it! */
    if (isalpha(ch) || isdigit(ch) || ch=='_' || ch=='$' || ch>127)
      {
      while(*toeval!='+' && *toeval!='(' && *toeval) *op++=*toeval++;
      if (*toeval=='(')
        {
        int open=0,instring=0;
        char *res;

        do
          {
          *op++=*toeval;
          if (*toeval=='\"') instring=1-instring;
          if (instring==0)
            {
            if (*toeval=='(') open++;
            if (*toeval==')') open--;
            }
          }
        while(*++toeval && open!=0);
        *op--=0;
        while(*op==' ') *op--=0;

        pra=-1;
        if (operand[0]==128) pra=operand[1]-128;
        if (operand[0]==129) pra=operand[1];

        /* Execute function */
        if ((operand[0]!='$' && operand[0]<128) ||
            (script_command[pra].command[0]!='$' && operand[0]>=128))
          {
          error("fLine %d: Function %s does not return a string",script_findline(cline),operand);
          stopit();
          }
        else
          {
          if (script_executecommand(operand,(int*)&res)==0)
            {
            error("fLine %d: Function %s does not exist",script_findline(cline),operand);
            stopit();
            }
          }
        strcpy(result,res);
        result+=strlen(result);
        }
      else
        {
        *op--=0;
        while(*op==' ') *op--=0;

        /* Get variable */
        if (script_get_s_variable(operand,result)==0)
          {
          error("fLine %d: Can't find variable '%s'",script_findline(cline),operand);
          stopit();
          }
        else
          {
          result+=strlen(result);
          }
        }
      }
    else
      {
      /* Check for constant string */
      if (ch=='\"')
        {
        toeval++;

        /* Copy string */
        while(1)
          {
          ch2=*toeval++;

          if (ch2=='\"' && *toeval=='\"')
            {
            *result++='\"'; toeval++;
            }
          else
            {
            if (ch2=='\"' && *toeval!='\"') break;
            else *result++=ch2;
            }
          }
        }
      else
        {
        if (ch!='+' && ch<128)
          {
          error("fLine %d: '%c' is an unacceptable operand when evaluating a string",script_findline(cline),ch);
          stopit();
          return(0);
          }
        toeval++;
        }
      }
    }
  *result=0;
  return(1);
  }

int script_executeblock(char *block,int *result,int isfunction)
  {
  char line[256],*lptr,*eolptr;
  int trash,sig;

  cline=block;

  if ((++script_level)==MAXNEST)
    {
    error("fLine %d: Over nested!",script_findline(cline));
    stopit();
    return(0);
    }

  while(*block==0) block++;
  if (*block=='{') block+=2;

newpointer:

  do
    {
    /* Get a line */
    lptr=line;
    cline=block;

    while(*block) *lptr++=*block++;
    eolptr=lptr-1; *lptr=0; lptr=line; gblock=++block;

    /* Comment? */
    if (*lptr==';' || *eolptr==':') goto newpointer;
    if (*lptr=='}')
      {
      script_unnest();
      *result=0;
      return(1);
      }
    else
      {
      if (*lptr)
        {
        /* Do command */
        if ((sig=script_executecommand(lptr,&trash))==NULL)
          {
          error("fLine %d: Unknown command %s",script_findline(cline),lptr);
          stopit();
          }
        /* Was command a break? */
        if (sig==2)
          {
          /* Drop out of this level */
          script_unnest();
          return(1);
          }

        /* Was command a return? */
        if (sig==3)
          {
          /* Drop down to function level */
          script_unnest();
          *result=trash;
          if (isfunction) return(1); else return(3);
          }
        if (sig>16) { block=(char*)sig; goto newpointer; }
        }
      }
    }
  while(1);
  }

void script_unnest()
  {
  /* Free variables */
  if (script_variables_count[script_level])
    {
    if (script_variables[script_level]!=NULL)
      {
      free(script_variables[script_level]);
      script_variables[script_level]=0;
      }
    script_variables[script_level]=0;
    script_variables_count[script_level]=0;
    }

  /* Free integers */
  if (script_integercount[script_level])
    {
    if (script_integerarea[script_level]!=NULL)
      {
      free(script_integerarea[script_level]);
      script_integerarea[script_level]=0;
      }
    script_integercount[script_level]=0;
    }

  /* Free strings */
  if (script_stringcount[script_level])
    {
    if (script_stringarea[script_level]!=NULL)
      {
      free(script_stringarea[script_level]);
      script_stringarea[script_level]=0;
      }
    script_stringcount[script_level]=0;
    }
  script_level--;
  }

void script_bomb()
  {
  int b;

  /* Free any function definitions made from our code */
  for(b=0;b<script_functioncount;b++)
    {
    if (script_functions[b].pointer>=global_script &&
        script_functions[b].pointer<=(global_script+global_len))
      {             
      script_functioncount--;
      memmove(&script_functions[b],
              &script_functions[b+1],
              sizeof(script_functionstr)*(script_functioncount-b));
      b--;
      }
    }

  /* Squeeze up the block now */
  script_functions=realloc(script_functions,sizeof(script_functionstr)*script_functioncount);

  /* Free any label definitions made from our code */
  for(b=0;b<script_labelcount;b++)
    {
    if (script_labels[b].pointer>=global_script &&
        script_labels[b].pointer<=(global_script+global_len))
      {                                                         
      script_labelcount--;
      memmove(&script_labels[b],
              &script_labels[b+1],
              sizeof(script_labelstr)*(script_labelcount-b));
      b--;
      }
    }

  /* Squeeze up the block now */
  script_labels=realloc(script_labels,sizeof(script_labelstr)*script_labelcount);

  /* Free memory */
  free(global_script);
  script_closedown(FALSE); /* Free memory but NOT globals */

  script_cankill=0;
  }

/* Parse a script file, storing any functions (#include command) */
int script_include(char *module,char *script,int length)
  {
  int nestinglevel=0,linecount=-1,a;
  char *scrptr=script,*endptr=(script+length),line[256],*lptr,*pptr;

  cprog=script;

  /* Parse file for functions & labels / global variables */
  while(scrptr<endptr)
    {
    lptr=line; linecount++;
    cline=scrptr;

    /* Get a line */
    if (*scrptr==NULL) { scrptr++; continue; }
    if (*scrptr==';') { while(*scrptr++); continue; }
    pptr=scrptr;
    while(*scrptr) *lptr++=*scrptr++;
    *lptr=0; lptr=line; scrptr++;
               
    /* Label? */
    if (line[a=(strlen(line)-1)]==':')
      {
      if (nestinglevel==0)
        {
        error("fModule %s, line %d: Can't have global labels",module,linecount);
        return(0);
        }

      script_labelcount++;
      if ((script_labels=realloc(script_labels,sizeof(script_labelstr)*script_labelcount))==NULL)
        {
        error("fModule %s, line %d: Can't allocate memory for label definition",module,linecount);
        return(0);
        }
      line[a]=0;
      strcpy(script_labels[script_labelcount-1].label,line);
      script_labels[script_labelcount-1].level=nestinglevel;
      script_labels[script_labelcount-1].pointer=scrptr;
      }

    /* Brace? */
    if (*lptr=='{' || *lptr=='}')
      {
      nestinglevel+=(*lptr=='{')?1:-1;
      if (nestinglevel<0)
        {
        error("fModule %s, line %d: Too many '}'s",module,linecount);
        return(0);
        }
      }
    else
      {
      char *sep;

      if (nestinglevel==0)
        {
        /* Separate it into command/tail */
        if ((sep=xstrchr(lptr,' '))==NULL)
          {
          error("fModule %s, line %d: '%s' illegal in global sense",module,linecount,lptr);
          return(0);
          }

        *sep++=0;
        while(*sep==' ') sep++;

        /* Check for 'integer' or 'string', otherwise it's a function */
        if (lptr[0]==128 && (lptr[1]>=128 && lptr[1]<=131))
          {
          int trash;
          char exec[256],*e;

          e=exec;
          while((*e++=*lptr++)!=NULL); e--; *e++=' ';
          while((*e++=*sep++)!=NULL);

          /* Run it through executecommand */
          script_executecommand(exec,&trash);
          }
        else
          {
          /* Get memory for function table */
          if ((script_functions=realloc(script_functions,(script_functioncount+1)*sizeof(script_functionstr)))==0)
            {
            error("fModule %s, line %d: Can't allocate memory for function definition",module,linecount);
            return(0);
            }

          strcpy(script_functions[script_functioncount].name,lptr);
          script_functions[script_functioncount++].pointer=pptr;
          }
        }
      }
    }
  return(1);
  }

/* Run a script */
int script_main(char *script,int length)
  {
  int trash;

  cprog=script;

  if (script_include("Main",script,length))
    {
    script_dofunction("main","",&trash);
    return(1);
    }
  else
    {
    return(0);
    }
  }

int script_dofunction(char *name,char *tail,int *result)
  {
  int a=0,trash;
  char *exec,*endexec,*endtail,chopline[256];

  if ((clock()-lastpoll)>MAXIMUM_POLL) window_poll();

  /* Find function first */
  while(stricmp(script_functions[a].name,name)!=0 && a<script_functioncount) a++;
  if (a>=script_functioncount) return(0);


  exec=strcpy(chopline,script_functions[a].pointer);

  /* Check line for parameters, and assign them */
  while(*exec!='(' && *exec) exec++;
  if (*exec==0)
    {
    error("fCan't find '(' in function definition: %s",script_functions[a].pointer);
    stopit();
    return(0);
    }

  exec++; endexec=exec+strlen(exec)-1;
  while(*endexec!=')' && *endexec) endexec--;
  if (*endexec==0)
    {
    error("fCan't find ')' in function definition: %s",script_functions[a].pointer);
    stopit();
    return(0);
    }
  *endexec=0;

  if ((++script_level)==MAXNEST)
    {
    error("fOver nested");
    return(0);
    }

  /* Parse for each one */
  do
    {
    endexec=xstrchr(exec,',');
    if (endexec!=NULL) *endexec++=0;

    endtail=tail;
    _separate(&endtail);
    *endtail++=0;

    if (*exec && *tail)
      {
      char evalbuffer[256],*e;
                               
      /* Strip leading spaces */
      while(*exec==' ') exec++;

      e=evalbuffer;
      while((*e++=*exec++)!=NULL); e--; *e++='=';
      while((*e++=*tail++)!=NULL);
      script_executecommand(evalbuffer,&trash);
      }

    exec=endexec;
    tail=endtail;
    }
  while(exec!=NULL && tail!=NULL);

  exec=script_functions[a].pointer;
  while(*exec++);

  --script_level;

  script_executeblock(exec,result,1);
  return(1);
  }

void script_run(char *filename)
  {
  FILE *scriptfile;
  int len; char *script;
           
  deb=0;

  if ((scriptfile=fopen(filename,"r"))==NULL)
    {
    error("fCan't open script file %s",filename);
    return;
    }

  fseek(scriptfile,0,SEEK_END);
  len=ftell(scriptfile);
  fseek(scriptfile,0,SEEK_SET);

  /* Get memory */
  if ((script=malloc(len))==NULL)
    {
    error("fCan't allocate memory to load script!");
    fclose(scriptfile);
    return;
    }

  /* Load file */
  fread(script,1,len,scriptfile);
  fclose(scriptfile);

  strcpy(script_name,filename);
  script_run2(script,len);
  }
 
void script_run2(char *script,int len)
  {
  char *end=script+len,*a=script;

  global_script=script; global_len=len;

  /* Convert file */
  while(a<end)
    {
    if (*a==10) *a=0;
    a++;
    }

  /* Remove spaces, ie compress buffer */
  len=script_compressbuffer(script,len);
  script=realloc(script,len);

  script_level=0;
  if (setjmp(script_kill)==0)
    {
    script_cankill=1;
    *script_chain=0;
    script_enter();
    script_main(script,len);
    }
  script_cankill=0;

  script_exit();
  script_bomb();

  if (*script_chain)
    {                        
    /* Call the next script */
    script_run(script_chain);
    }
  }

int script_runfunction(char *function,char *tail,int *result)
  {
  int aborted=1;

  script_level=0;
  if (setjmp(script_kill)==0)
    {
    script_cankill=1;
    script_enter();
    if (script_dofunction(function,tail,result)==0)
      {
      script_cankill=0;
      return(0);
      }
    aborted=0;
    }
  script_exit();
  script_cankill=0;

  /* Free memory */
  script_closedown(FALSE); /* Free memory but NOT globals */

  return(aborted?2:1);
  }

void script_enter()
  {
  int a;

  for(a=0;a<4;a++) script_file[a]=0;
  }

void script_exit()
  {
  int a;

  for(a=0;a<4;a++) if (script_file[a]!=0) _fclose(&script_file[a]);
  }

int script_findline(char *erl)
  {
  char *a=cprog;
  int b=0;

  if (erl<=a) return(0);

  while(a<erl) if(*a++==0) b++;

  return(b+1);
  }

/*****************************************************************************/
/*                                                                           */
/* Recursive Data Parser (?)                                                 */
/*                                                                           */
/* Original by Nigel Brown, this hack by Hugo Fiennes!                       */
/*                                                                           */
/* v0.01 22-Oct-1991                                                         */
/*                                                                           */
/*****************************************************************************/


/*
  
   Nigel: this is a 'fuller' version than yours was I think, I have
          implemented most of the C bits in the order specified in
          my ANSI C guide, and also done << >> >= <= != == ! ~ & | ^
          || &&. Note that this accepts '=' as '==' (can be altered in
          gettoken() because that's how arcscript works at the moment).

          Also, there is no global concept of 'program' and 'pc' - hence
          even from inside the lowest level you can call it independently
          and it will not interact, for example:

          if (a>65 || function(d*6,e<<3)==0)

          here, when evaluating the parameters for function to call it,
          and even when executing function the original evaluation will
          not be screwed up.

*/

char tok_type;
char tokn[256]="\0";

extern void eval_exp_2(char**,int*),
            eval_exp_3(char**,int*),
            eval_exp_4(char**,int*),
            eval_exp_5(char**,int*),
            eval_exp_6(char**,int*),
            eval_exp_7(char**,int*),
            eval_exp_8(char**,int*),
            eval_exp_9(char**,int*),
            eval_exp_10(char**,int*),
            eval_exp_11(char**,int*),
            eval_exp_12(char**,int*),
            eval_exp_13(char**,int*),
            atom(char**,int*);
extern void eval_exp(char*,int*);

/*==========================================================================
* return the next token
*=========================================================================*/

void get_token(char **exp)
  {
  char *temp=tokn,*ex=*exp;
  
  tok_type=0; tokn[0]=0;
  while(*ex==' ') ex++;                  /* skip white space */
  if(*ex==0) return; /* end of expression */

  if (isdigit(*ex))
    {
    tok_type=NUMBER;
    while(xstrchr(operators,*ex)==0) *temp++=*ex++;
    goto gottoken;
    }

  if (isalpha(*ex) || *ex=='_' || *ex>127 || *ex==39)
    {
    /* Variable or function */
    tok_type=VARIABLE;

    if (*ex==39)
      {
      *temp++=*ex++;
      *temp++=*ex++;
      *temp++=*ex++;
      goto gottoken;
      }

    while(xstrchr(operators,*ex)==0) *temp++=*ex++;
    if (*ex=='(')
      {
      int open=0,inquote=0;
      do
        {
        if (*ex=='\"') inquote=!inquote;
        if (inquote==0)
          {
          if (*ex=='(') open++;
          if (*ex==')') open--;
          }
        *temp++=*ex++;
        }
      while(*ex && open!=0);
      }
    goto gottoken;
    }
                  
  if(xstrchr("<>!=&|",*ex))
    {   
    tok_type=DELIMITER;
 
    *temp=*ex++;
    switch(*temp)
      {
      case '<':
        if (*ex=='<') { *temp++=OP_LEFTSHIFT;            ex++; goto gottoken; }
        if (*ex=='>') { *temp++=OP_NOTEQUAL;             ex++; goto gottoken; }
        if (*ex=='=') { *temp++=OP_LESSTHANOREQUAL;      ex++; goto gottoken; }
        break;       
      case '>':
        if (*ex=='=') { *temp++=OP_GREATERTHANOREQUAL;   ex++; goto gottoken; }
        if (*ex=='>') { *temp++=OP_RIGHTSHIFT;           ex++; goto gottoken; }
        break;
      case '=':
        if (*ex=='=') { *temp++=OP_EQUAL;                ex++; goto gottoken; }
        if (*ex!='=') { *temp++=OP_EQUAL;                      goto gottoken; }
        break;
      default:
        if (*temp=='!' && *ex=='=')
          { *temp++=OP_NOTEQUAL;             ex++; goto gottoken; }
        if (*temp=='&' && *ex=='&')
          { *temp++=OP_AND;                  ex++; goto gottoken; }
        if (*temp=='|' && *ex=='|')
          { *temp++=OP_OR;                   ex++; goto gottoken; }
        break;
      }
    temp++; goto gottoken;
    }

  if(xstrchr(operators,*ex))
    {   
    tok_type=DELIMITER;
    *temp++=*ex++;
    }

  gottoken:
  *temp=0;
  *exp=ex;
  }

/*==========================================================================
* recursive descent parser
*=========================================================================*/

int script_integer_evaluate(char *exp,int *answer)
  {                 
  char *pcpoint=exp,old_tok_type,old_token[256];
           
  /* Save old token pointers incase we are recursing */
  old_tok_type=tok_type;
  strcpy(old_token,tokn);
  sexp=exp;
                              
  /* Get first token & start */
  get_token(&pcpoint);
  if(!*tokn) goto restore;

  if (*pcpoint==NULL) atom(&pcpoint,answer);
  else eval_exp_2(&pcpoint,answer);

  restore:
  /* Restore old token pointers before leaving */
  tok_type=old_tok_type;
  strcpy(tokn,old_token);

  return(1);
  }

/********************************* Level 2 - || operator */

#ifdef NOWINASM
void eval_exp_2(char **exp,int *answer)
  {     
  eval_exp_3(exp,answer);
  while(*tokn==OP_OR)
    {
    int temp;
    get_token(exp); eval_exp_3(exp,&temp);
    *answer=*answer||temp;
    }
  }
#endif

/********************************* Level 3 - && operator */

#ifdef NOWINASM
void eval_exp_3(char **exp,int *answer)
  {     
  int temp;

  eval_exp_4(exp,answer);
  while(*tokn==OP_AND)
    {
    get_token(exp); eval_exp_4(exp,&temp);
    *answer=*answer&&temp;
    }
  }
#endif

/********************************* Level 4 - bitwise | operator */

#ifdef NOWINASM
void eval_exp_4(char **exp,int *answer)
  {     
  int temp;

  eval_exp_5(exp,answer);
  while(*tokn=='|')
    {
    get_token(exp); eval_exp_5(exp,&temp);
    *answer=*answer|temp;
    }
  }
#endif

/********************************* Level 5 - bitwise ^ operator */

#ifdef NOWINASM
void eval_exp_5(char **exp,int *answer)
  {     
  int temp;

  eval_exp_6(exp,answer);
  while(*tokn=='^')
    {
    get_token(exp); eval_exp_6(exp,&temp);
    *answer=*answer^temp;
    }
  }
#endif

/********************************* Level 6 - bitwise & operator */

#ifdef NOWINASM
void eval_exp_6(char **exp,int *answer)
  {     
  int temp;

  eval_exp_7(exp,answer);
  while(*tokn=='&')
    {
    get_token(exp); eval_exp_7(exp,&temp);
    *answer=*answer&temp;
    }
  }
#endif

/********************************* Level 7 - equality: != = */

void eval_exp_7(char **exp,int *answer)
  {     
  char op; int temp;

  eval_exp_8(exp,answer);
  while((op=*tokn)==OP_EQUAL || op==OP_NOTEQUAL)
    {
    get_token(exp); eval_exp_8(exp,&temp);
    if (op==OP_EQUAL)
      *answer=(*answer==temp);
    else
      *answer=(*answer!=temp);
    }
  }

/********************************* Level 8 - inequality: < > <= >= */

void eval_exp_8(char **exp,int *answer)
  {     
  char op; int temp;

  eval_exp_9(exp,answer);
  while((op=*tokn)=='<' || op=='>' ||
        op==OP_GREATERTHANOREQUAL || op==OP_LESSTHANOREQUAL)
    {
    get_token(exp); eval_exp_9(exp,&temp);
    switch(op)
      {
      case '<':
        *answer=(*answer<temp);
        break;
      case '>':
        *answer=(*answer>temp);
        break;
      case OP_GREATERTHANOREQUAL:
        *answer=(*answer>=temp);
        break;
      case OP_LESSTHANOREQUAL:
        *answer=(*answer<=temp);
        break;
      }
    }
  }

/********************************* Level 9 - shift << and >> */

void eval_exp_9(char **exp,int *answer)
  {     
  char op; int temp;

  eval_exp_10(exp,answer);
  while((op=*tokn)==OP_LEFTSHIFT || op==OP_RIGHTSHIFT)
    {
    get_token(exp); eval_exp_10(exp,&temp);
    if (op==OP_LEFTSHIFT)
      *answer=(*answer)<<temp;
    else
      *answer=(*answer)>>temp;
    }
  }

/********************************* Level 10 - additive: + - */

void eval_exp_10(char **exp,int *answer)
  {     
  char op; int temp;

  eval_exp_11(exp,answer);
  while((op=*tokn)=='+' || op=='-')
    {
    get_token(exp); eval_exp_11(exp,&temp);
    if (op=='+')
      *answer=(*answer)+temp;
    else
      *answer=(*answer)-temp;
    }
  }

/********************************* Level 11 - multiplicative: * / % */

void eval_exp_11(char **exp,int *answer)
  {     
  char op; int temp;

  eval_exp_12(exp,answer);
  while((op=*tokn)=='*' || op=='/' || op=='%')
    {
    get_token(exp); eval_exp_12(exp,&temp);         
    switch(op)
      {
      case '*':
        *answer=(*answer)*temp;
        break;
      case '/':
        *answer=(*answer)/temp;
        break;
      case '%':
        *answer=(*answer)%temp;
        break;
      }
    }
  }

/********************************* Level 12 - unary + - ! ~ */

void eval_exp_12(char **exp,int *answer)
  {     
  if (tok_type==DELIMITER &&
      (*tokn=='+' || *tokn=='-' || *tokn=='!' || *tokn=='~'))
    {
    char op=*tokn;
    get_token(exp);
    eval_exp_13(exp,answer);
    switch(op)
      {
      case '-':
        *answer=-(*answer);
        break;
      case '+':
        break;
      case '!':
        *answer=!(*answer);
        break;
      case '~':
        *answer=~(*answer);
        break;
      }
    return;
    }

  if (*tokn!='(') atom(exp,answer); else eval_exp_13(exp,answer);
  }

/********************************* Level 13 - expression */

void eval_exp_13(char **exp,int *answer)
  {
  if (*tokn=='(')
    {
    get_token(exp);
    eval_exp_2(exp,answer);
    if(*tokn!=')')
      {
      error("fUnbalanced parentheses in line %d '%s'",script_findline(cline),sexp);
      stopit();
      }
    get_token(exp);
    }
  else
    {
    atom(exp,answer);
    }
  }

/*==========================================================================
* variable or numeric ?
*=========================================================================*/

void atom(char **exp,int *answer)
  {  
  switch(tok_type)
    {
    case VARIABLE:
      {
      int res;
      char *bb,*ba;
        
      if (tokn[0]==39)
        {
        *answer=tokn[1];
        get_token(exp);
        return;
        }

      bb=xstrchr(tokn+1,'['); ba=xstrchr(tokn+1,'(');

      /* It's a variable or a function */
      if (ba!=NULL && (bb==NULL || bb>ba))
        {
        if (tokn[0]=='$')
          {
          error("fLine %d: Function %s returns a string, not an integer",script_findline(cline),tokn);
          stopit();
          res=0;
          }
        else
          {
          if (script_executecommand(tokn,&res)==0)
            {
            error("fLine %d: Function %s does not exist",script_findline(cline),tokn);
            stopit();
            res=0;
            }
          }
        }
      else
        {
        /* Variable */
        if (script_get_i_variable(tokn,&res)==0)
          {
          error("fLine %d: Can't find variable '%s'",script_findline(cline),tokn);
          stopit();
          res=0;
          }
        }
      *answer=res;

      get_token(exp);
      return;
      }
    case NUMBER:
      {
      char *t=tokn;

      *answer=0;
      if (tokn[0]=='0' && tolower(tokn[1])=='x')
        {
        static char hex[]={"0123456789abcdef"};
        int a;

        /* Hex */
        do
          {
          a=strchr(hex,*t++)-hex;
          if (a>=0) *answer=((*answer)*16)+a;
          }
        while(*t);
        }
      else
        {
        do
          {
          *answer=((*answer)*10)+(*t++-'0');
          }
        while(*t);
        }

      get_token(exp);
      return;
      }
    default:
      {
      error("fUnidentified operand in line %d",script_findline(cline));
      stopit();
      break;
      }
    }
  }

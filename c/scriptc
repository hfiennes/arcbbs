/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 05-March-1990                               <]
                  [>                                             <]
Module name       [> Script internal functions                   <]
Current version   [> 00.04                                       <]
Version date      [> 28-March-1993                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1990/91/92/93 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "script.h"
#include "port.h"
#include "kernel.h"
#include "mprintf.h"
#include "str.h"

#include "userlog.h"
#include "mail.h"

#include "bbs.h"
#include "servmess.h"
#include "servcomm.h"
#include "portmisc.h"

#define I0 tail[0].d.i
#define I1 tail[1].d.i
#define I2 tail[2].d.i

#define S0 tail[0].d.s
#define S1 tail[1].d.s
#define S2 tail[2].d.s

#define P0 tail[0].p
#define P1 tail[1].p
#define P2 tail[2].p

extern void afterfind(char**),do_command(char*,int);

static char script_tempstring[256];
extern FILE *script_file[4];
extern char *gblock,script_chain[128];
extern void window_poll(void),wimp_waitfor(int);

extern int script_functioncount,rcode,script_level,
           lastpoll,open_spool(char*,void*),deb,
           script_stop,portnumber,script_labelcount,files_done;
extern script_functionstr *script_functions;
extern script_labelstr *script_labels;
extern void _separate(char **),_fclose(FILE**);
extern int  *script_integerarea[];
extern char *script_stringarea[];
extern struct tm script_time;

static void erep(char *msg)
  {
  error(msg);
  stopit();
  }

static void erepi(char *msg,int a)
  {
  error(msg,a);
  stopit();
  }

static void ereps(char *msg,char *a)
  {
  error(msg,a);
  stopit();
  }

int script_c_integer(script_pass *tail2)
  {
  char *onevar,*endvar,*equal,*array,*tail=(char*)tail2;
  int result,asize;

  onevar=tail;

  /* Parse tail */
  do
    {
    endvar=onevar;
    _separate(&endvar);
    if (*endvar)
      {
      *endvar++=0;
      while(*endvar && *endvar==' ') endvar++;
      }

    /* Check for array */
    if ((array=strchr(onevar,'['))!=NULL)
      {
      int open=1,inquote=0;
      char *a2;

      *array++=0;
      a2=array;

      while((inquote!=0 || open!=0) && *a2)
        {
        if (*a2=='\"') inquote=1-inquote;
        if (inquote==0)
          {
          if (*a2=='[') open++;
          if (*a2==']') open--;
          }
        a2++;
        }

      *--a2=0;
      if (script_integer_evaluate(array,&asize)==0) asize=1;
      script_add_i_variable(onevar,script_level,asize,0);
      }
    else
      {
      /* Check for assignment */
      if ((equal=strchr(onevar,'='))==NULL)
        {
        script_add_i_variable(onevar,script_level,1,0);
        }
      else
        {
        *equal++=0;
        /* Evaluate expression */
        if (script_integer_evaluate(equal,&result)==0) return(0);
        script_add_i_variable(onevar,script_level,1,result);
        }
      }
    onevar=endvar;
    }
  while(*onevar);
  return(0);
  }

int script_c_string(script_pass *tail2)
  {
  char *onevar,*endvar,*len,*a2,result[256],*tail=(char*)tail2;

  result[0]=0; onevar=tail;

  /* Parse tail */
  do
    {
    int open=1,open2=0,inquote=0,length,pos=0,asize=1;

    endvar=onevar;
    _separate(&endvar);
    if (*endvar)
      {
      *endvar++=0;
      while(*endvar && *endvar==' ') endvar++;
      }

    /* Check for length */
    if ((len=strchr(onevar,'['))==NULL)
      {
      ereps("fNo length indicated: string %s",onevar);
      return(0);
      }
    *len++=0; a2=len;

    while((inquote!=0 || open2!=0 || open!=0) && *a2)
      {
      if (*a2=='\"') inquote=1-inquote;
      if (inquote==0)
        {
        if (*a2=='[') open++;
        if (*a2==']') open--;
        if (*a2=='(') open2++;
        if (*a2==')') open2--;
        if (*a2==',' && open2==0)
          {
          *a2++=0;
          if (script_integer_evaluate(len,&asize)==0) asize=1;
          pos++; len=a2;
          continue;
          }
        }
      a2++;
      }

    *--a2=0;
    a2++;

    if (script_integer_evaluate(len,&length)==0) return(0);

    while(*a2==' ' && *a2) a2++;

    if (*a2=='=')
      {
      if (script_string_evaluate(a2+1,result)==0) result[0]=0;
      }

    script_add_s_variable(onevar,script_level,asize,length,result);

    onevar=endvar;
    }
  while(*onevar);
  return(0);
  }

int script_c_print(script_pass *tail)
  {          
  mprintf(0,"%d",I0);
  return(0);
  }

int script_c_vdu(script_pass *tail)
  {
  port_txw(I0);
  return(0);
  }

int script_c_prints(script_pass *tail)
  {
  port_txstring(S0,1);
  return(0);
  }

int script_c_return(script_pass *tail)
  {
  return(P0?I0:0);
  }

int script_c_stringreturn(script_pass *tail)
  {
  static char result[256];

  strcpy(result,S0);
  return((int)result);
  }

int script_c_len(script_pass *tail)
  {
  return(strlen(S0));
  }

int script_c_asc(script_pass *tail)
  {
  return(S0[0]);
  }

int script_c_val(script_pass *tail)
  {
  return(atoi(S0));
  }

int script_c_instr(script_pass *tail)
  {
  char *offset;
  int a;

  offset=strstr(S0,S1);
  if (offset==NULL) return(0);
  a=offset-S0;
  return(a+1);
  }

int script_c_compare(script_pass *tail)
  {
  return(strcmp(S0,S1)?0:-1);
  }

int script_c_comparei(script_pass *tail)
  {
  return(stricmp(S0,S1)?0:-1);
  }

int script_c_strcmp(script_pass *tail)
  {
  return(strcmp(S0,S1));
  }

int script_c_stricmp(script_pass *tail)
  {
  return(stricmp(S0,S1));
  }

int script_c_stringleft(script_pass *tail)
  {
  static char temp[256];

  strcpy(temp,S0);
  if (I1<256) temp[I1]=0;
  return((int)temp);
  }

int script_c_stringright(script_pass *tail)
  {
  int slen;
  static char temp[256];

  strcpy(temp,S0);
  slen=strlen(temp);
  if (I1>slen) I1=slen;

  return((int)temp+(slen-I1));
  }

int script_c_stringmid(script_pass *tail)
  {
  int slen,howmuchi=P2?I2:1;
  static char temp[256];
  char *howmuch;

  strcpy(temp,S0);
  if (I1<1) I1=1;
  --I1;
  if (I1>strlen(temp)) return((int)"");
  howmuch=temp+I1;
  if (howmuchi>(slen=strlen(howmuch))) howmuchi=slen;
  howmuch[howmuchi]=0;

  return((int)howmuch);
  }

int script_c_chr(script_pass *tail)
  {
  static char ret[2];

  ret[0]=I0;
  ret[1]=0;
  return((int)ret);
  }

int script_c_str(script_pass *tail)
  {
  static char ret[20];

  sprintf(ret,"%d",I0);
  return((int)ret);
  }

int script_c_pause(script_pass *tail)
  {
  wimp_waitfor(I0);
  return(0);
  }

int script_c_portrxbuffer(script_pass *tail)
  {
  tail=tail;
  return(port_rxbuffer());
  }

int script_c_porttxbuffer(script_pass *tail)
  {
  tail=tail;
  return(port_txbuffer());
  }

int script_c_portrxclear(script_pass *tail)
  {
  tail=tail;
  port_rxclear();
  return(1);
  }

int script_c_porttxclear(script_pass *tail)
  {
  tail=tail;
  port_txclear();
  return(1);
  }

int script_c_portrx(script_pass *tail)
  {
  tail=tail;
  return(port_rx());
  }

int script_c_porttx(script_pass *tail)
  {
  port_txw(I0);
  return(0);
  }

int script_c_time(script_pass *tail)
  {
  tail=tail;
  return(clock());
  }

int script_c_for(script_pass *tail2)
  {
  char *a,*b,*c,*afterblock,*repblock=gblock,forvarn[20],*tail=(char*)tail2;
  int initial,final,step,junk,code,tm;
  script_variable *forvar;

  /* Find end of block */
  afterblock=repblock;
  afterfind(&afterblock);

  if ((a=strchr(tail,'='))==NULL)
    {
    ereps("fCan't find '=': for %s",tail);
    return((int)afterblock);
    }
  *a++=0;
  while(*a && *a==' ') a++;

  /* Trim stuff off tail */
  b=tail+strlen(tail)-1;
  while(*b==' ' && b>tail) *b--=0;

  if ((b=strstr(a,"to"))==NULL)
    {
    error("fCan't find 'to': for %s=%s",tail,a);
    stopit();
    }
  *b++=0; b++;
  while(*b && *b==' ') b++;

  if (script_integer_evaluate(a,&initial)==0)
    {
    error("fCan't evaluate initial value: for %s=%s to %s",tail,a,b);
    stopit();
    }

  if ((c=strstr(b,"step"))==NULL)
    {
    step=1;
    }
  else
    {
    *c=0;
    if (script_integer_evaluate(c+4,&step)==0)
      {
      error("fCan't evaluate step value: for %s=%s to %s %s",tail,a,b,c);
      stopit();
      }
    }

  if (script_integer_evaluate(b,&final)==0)
    {
    error("fCan't evaluate final value: for %s=%s to %s",tail,a,b);
    stopit();
    }

  if ((forvar=script_findvariable(tail,&junk))==NULL)
    {
    error("fCan't find variable '%s': for %s=%s to %s %s",tail,tail,a,b,c);
    stopit();
    }

  if (forvar->type!=0)
    {
    error("fVariable '%s' is a string: for %s=%s to %s %s",tail,tail,a,b,c);
    stopit();
    }

  strcpy(forvarn,tail);
  script_put_i_variable(forvarn,initial);

  do
    {
    /* Execute block */
    code=script_executeblock(repblock,&junk,0);
    if (code==2) { return((int)afterblock); }
    if (code==3) { rcode=junk; return(3); }

    script_get_i_variable(forvarn,&tm);
    tm+=step;
    script_put_i_variable(forvarn,tm);
    }
  while(tm<=final);

  return((int)afterblock);
  }

int script_c_swi(script_pass *tail2)
  {
  char *a,*tail=(char*)tail2;
  script_variable *in;
  os_regset i; int level,swin,c;

  if ((a=strchr(tail,','))==NULL)
    {
    ereps("fCan't find ',': swi(%s)",tail);
    return(0);
    }
  *a++=0;
  while(*a && *a==' ') a++;

  if (script_integer_evaluate(tail,&swin)==0)
    {
    ereps("fCan't evaluate swi number: %s",tail);
    return(0);
    }

  if ((in=script_findvariable(a,&level))==NULL)
    {
    ereps("fCan't find variable '%s'",tail);
    return(0);
    }

  if (in->type!=0)
    {
    ereps("fVariable '%s' is a string",tail);
    return(0);
    }

  for(c=0;c<10;c++) i.r[c]=script_integerarea[level][(in->value)+c];
  os_swix(swin,&i);
  for(c=0;c<10;c++) script_integerarea[level][(in->value)+c]=i.r[c];

  return(0);
  }

int script_c_address(script_pass *tail2)
  {
  char *tail=(char*)tail2;
  script_variable *in;
  int level;

  if ((in=script_findvariable(tail,&level))==NULL)
    {
    ereps("fCan't find variable '%s'",tail);
    return(0);
    }

  if (in->type!=0) return((int)script_stringarea[level]+in->value);
  return((int)&script_integerarea[level][in->value]);
  }

int script_c_while(script_pass *tail2)
  {
  char *afterblock,*repblock=gblock,*tail=(char*)tail2;
  int junk,code;

  /* Find end of block */
  afterblock=repblock;
  afterfind(&afterblock);

  if (script_integer_evaluate(tail,&code)==0)
    {
    ereps("fCan't evaluate: while(%s)",tail);
    return((int)afterblock);
    }

  while(code)
    {
    /* Execute block */
    code=script_executeblock(repblock,&junk,0);
    if (code==2) { return((int)afterblock); }
    if (code==3) { rcode=junk; return(3); }

    if (script_integer_evaluate(tail,&code)==0)
      {
      ereps("fCan't evaluate: while(%s)",tail);
      return((int)afterblock);
      }
    }

  return((int)afterblock);
  }

/*** Switch & case *********************************************************/

char switch_exp[128],*switch_end,switch_default;

int script_c_switch(script_pass *tail2)
  {
  char *afterblock,*repblock=gblock,*tail=(char*)tail2;
  int code,junk,save_level;
  char sswitch_exp[128],*sswitch_end,*se;
  int sswitch_default,holding;

  /* Find end of block */
  afterblock=repblock;
  afterfind(&afterblock);

  if (*switch_exp!=0)
    {
    strcpy(sswitch_exp,switch_exp);
    sswitch_end=switch_end;
    sswitch_default=switch_default;
    holding=1;
    }

  strcpy(switch_exp,tail);
  switch_end=afterblock;
  switch_default=1;

  save_level=script_level;
  code=script_executeblock(repblock,&junk,0);
  while(script_level>save_level) script_unnest();

  se=switch_end;

  /* Restore old ones for nesting */
  if (holding)
    {
    strcpy(switch_exp,sswitch_exp);
    switch_end=sswitch_end;
    switch_default=sswitch_default;
    }

  if (code==2) { return((int)se); }
  if (code==3) { rcode=junk; return(3); }
  return((int)se);
  }

int _script_c_case(char *cm,char *tail)
  {
  char *afterblock,*repblock=gblock,temp[256],*nxt;
  int junk,code;

  /* Find end of block */
  afterblock=repblock;
  afterfind(&afterblock);

  do
    {
    nxt=tail;
    _separate(&nxt);
    if (*nxt) *nxt++=0;

    sprintf(temp,cm,switch_exp,tail);

    if (script_integer_evaluate(temp,&code)==0)
      {
      return((int)switch_end);
      }

    if (code!=0)
      {
      switch_default=0;
      code=script_executeblock(repblock,&junk,0);
      if (code==2) { return((int)switch_end); }
      if (code==3) { rcode=junk; return(3); }
      }

    tail=nxt;
    }
  while(*tail!=0);

  return((int)afterblock);
  }

int script_c_case(script_pass *tail)
  {
  return(_script_c_case("%s==%s",(char*)tail));
  }

int script_c_cases(script_pass *tail)
  {
  return(_script_c_case("compare(%s,%s)",(char*)tail));
  }

int script_c_casesi(script_pass *tail)
  {
  return(_script_c_case("comparei(%s,%s)",(char*)tail));
  }

int script_c_default(script_pass *tail)
  {
  char *afterblock,*repblock=gblock;
  int junk,code;

  tail=tail;

  /* Find end of block */
  afterblock=repblock;
  afterfind(&afterblock);

  if (switch_default)
    {
    switch_default=0;

    code=script_executeblock(repblock,&junk,0);
    if (code==2) { return((int)switch_end); }
    if (code==3) { rcode=junk; return(3); }
    }

  return((int)afterblock);
  }
            
/*** goto ******************************************************************/

int script_c_goto(script_pass *tail)
  {
  int a;

  /* Where is label */
  for(a=0;a<script_labelcount;a++)
    {
    if (stricmp((char*)tail,script_labels[a].label)==0)
      {
      if (script_level<script_labels[a].level)
        {
        erep("fCan't goto to a higher nesting level");
        return((int)gblock);
        }
      while(script_level!=script_labels[a].level) script_unnest();
      return((int)script_labels[a].pointer);
      }
    }                 

  ereps("fCan't find label %s",(char*)tail);
  return((int)gblock);
  }

/*** repeat...until ********************************************************/

int script_c_repeat(script_pass *tail2)
  {
  char *afterblock,*repblock=gblock,*endblock,temp[128],*t;
  int junk,code;

  tail2=tail2;

  /* Find end of block */
  afterblock=repblock;
  afterfind(&afterblock);
  endblock=afterblock;
  while(*endblock++);

  if (strnicmp(afterblock,"until(",6)!=0)
    {
    erep("funtil() missing after a repeat");
    return((int)endblock);
    }

  afterblock+=6;
  strcpy(temp,afterblock);
  t=(temp+strlen(temp)-1);
  while(*t!=')' && t>temp) t--;
  *t=0;

  do
    {
    code=script_executeblock(repblock,&junk,0);
    if (code==2) { return((int)endblock); }
    if (code==3) { rcode=junk; return(3); }
     
    if (script_integer_evaluate(temp,&code)==0)
      {
      return((int)endblock);
      }
    }
  while(code==0);

  return((int)endblock);
  }

/*** if...then...else ******************************************************/

int script_c_if(script_pass *tail2)
  {
  char *afterblock,*elseblock=0,*repblock=gblock,*tail=(char*)tail2;
  int junk,code;

  /* Find end of block */
  afterblock=repblock;
  afterfind(&afterblock);
  if (strnicmp(afterblock,"else",4)==0)
    {
    /* An else! */
    while(*afterblock++);
    elseblock=afterblock;
    afterfind(&afterblock);
    }

  if (script_integer_evaluate(tail,&code)==0)
    {
    ereps("fCan't evaluate: if (%s)",tail);
    return((int)afterblock);
    }

  if (code)
    {
    /* Execute block */
    code=script_executeblock(repblock,&junk,0);
    if (code==2) { return((int)afterblock); }
    if (code==3) { rcode=junk; return(3); }
    }
  else
    {
    if (elseblock)
      {
      /* Execute <else> block */
      code=script_executeblock(elseblock,&junk,0);
      if (code==2) { return((int)afterblock); }
      if (code==3) { rcode=junk; return(3); }
      }
    }

  return((int)afterblock);
  }

int script_c_debug(script_pass *tail)
  {
  deb=I0;
  return(0);
  }

int script_c__fileopen(script_pass *tail,char *who,char *how)
  {
  if (I0<0 || I0>3)
    {
    error("ffile_open%s: invalid filenumber (%d)",who,I0);
    stopit();
    }

  if (script_file[I0]!=0) _fclose(&script_file[I0]);

  if ((script_file[I0]=fopen(S1,how))==NULL) return(0);
  return(1);
  }

int script_c__filecheck(int file,char *co)
  {
  if (file<0 || file>3)
    {
    error("ffile_%s: invalid filenumber (%d)",co,file);
    stopit();
    }

  if (script_file[file]==NULL)
    {
    error("ffile_%s: file %d is not open!",co,file);
    stopit();
    }
  return(0);
  }

int script_c_fileopenin(script_pass *tail)
  {
  return(script_c__fileopen(tail,"in","rb"));
  }

int script_c_fileopenout(script_pass *tail)
  {
  return(script_c__fileopen(tail,"out","wb+"));
  }

int script_c_fileopenup(script_pass *tail)
  {
  return(script_c__fileopen(tail,"up","rb+"));
  }

int script_c_fileclose(script_pass *tail)
  {
  if (I0<0 || I0>3)
    {
    erepi("ffile_close: invalid filenumber (%d)",I0);
    return(0);
    }

  _fclose(&script_file[I0]);
  return(0);
  }

int script_c_filereadline(script_pass *tail)
  {
  int a=0,b;
  static char temp[256];

  if (script_c__filecheck(I0,"readline")) return(0);
  while(a<256)
    {
    if ((b=fgetc(script_file[I0]))==EOF) break;
    if (b==10) break;
    temp[a++]=b;
    }
  temp[a]=0;

  return((int)temp);
  }

int script_c_filewriteline(script_pass *tail)
  {
  char *t=S1;

  if (script_c__filecheck(I0,"writeline")) return(0);
  while(*t) fputc(*t++,script_file[I0]);
  fputc(10,script_file[I0]);

  return(1);
  }

int script_c_filereadbyte(script_pass *tail)
  {
  int b;

  if (script_c__filecheck(I0,"readbyte")) return(0);
  if ((b=fgetc(script_file[I0]))==EOF) return(-1);
  return(b);
  }

int script_c_filewritebyte(script_pass *tail)
  {
  if (script_c__filecheck(I0,"writebyte")) return(0);
  fputc(I1,script_file[I0]);
  return(1);
  }

int script_c_fileeof(script_pass *tail)
  {
  if (script_c__filecheck(I0,"eof")) return(0);
  return(feof(script_file[I0])?1:0);
  }

int script_c_fileptr(script_pass *tail)
  {
  if (script_c__filecheck(I0,"ptr")) return(0);
  return(ftell(script_file[I0]));
  }

int script_c_fileext(script_pass *tail)
  {
  int b,c;

  if (script_c__filecheck(I0,"ext")) return(0);
  b=ftell(script_file[I0]);
  fseek(script_file[I0],0,SEEK_END);
  c=ftell(script_file[I0]);
  fseek(script_file[I0],b,SEEK_SET);

  return(c);
  }

int script_c_filesetptr(script_pass *tail)
  {
  if (script_c__filecheck(I0,"setptr")) return(0);
  fseek(script_file[I0],I1,SEEK_SET);
  return(1);
  }

int script_c_oscli(script_pass *tail)
  {
  os_cli(S0);
  return(atoi(getenv("Sys$ReturnCode")));
  }

int script_c_system(script_pass *tail)
  {
  return(system(S0));
  }

int script_c_getenv(script_pass *tail)
  {
  return((int)getenv(S0));
  }

int script_c_dirscan(script_pass *tail)
  {
  _kernel_swi_regs regs;
  _kernel_oserror *err;

  static int ptr,len,last_notwild;
  static char *file,*dir,*name,path[255],res[255];

  if (S0 && *S0)
    {
    strcpy (path,S0);

    /* Try for a directory prefix */
    file = strrchr (path, '.');

    /* If not, try for a filing system prefix */
    if (file == NULL)
      {
      if (*path == '-')
        file = strchr (path + 1, '-');
      else
        file = strchr (path, ':');
      }

    if (file)
      {
      len = file - path + 1;
      strncpy (res, path, len);

      name = &res[len];
      len = 255 - len;

      dir = path;
      *file++ = '\0';
      }
    else
      {
      name = res;
      len = 255;

      file = path;
      dir = "";
      }

    /* Is the file name wildcarded? */
    if (strchr (file, '*') || strchr (file, '#'))
      last_notwild = 0;
    else
      {
      strcpy (name, file);
      last_notwild = 1;
      return((int)res);
      }

    ptr = 0;
    }

  /* If we're looking for more from a non-wild name, no luck */
  if (last_notwild) return((int)"");

  regs.r[0] = 9;
  regs.r[1] = (int) dir;
  regs.r[2] = (int) name;
  regs.r[4] = ptr;
  regs.r[5] = len;
  regs.r[6] = (int) file;

  do
    {
    regs.r[3] = 1;
    err = _kernel_swi (0x0C, &regs, &regs);

    if (err)
      {
      error("f*** Error (%d) - %s\n",err->errnum, err->errmess);
      return((int)"");
      }
    }
  while (regs.r[3] == 0 && regs.r[4] >= 0);

  ptr = regs.r[4];

  if (ptr < 0) return((int)"");

  return((int)res);
  }

int script_c_end(script_pass *tail)
  {
  tail=tail;

  script_stop=1;
  window_poll();

  return(0);
  }

int script_c_doubleclick(script_pass *tail)
  {
  int filetype=0; /*getfiletype(S0);*/
  wimp_msgstr message;

  if (filetype<0) return(0);

  message.hdr.size=4*((48+strlen(S0))/4+1);
  message.hdr.your_ref=0;
  message.hdr.action=5; /* DataOpen */
  message.data.words[0]=0;
  message.data.words[1]=0;
  message.data.words[2]=0; /* x/y of click */
  message.data.words[3]=0;
  message.data.words[4]=0;
  message.data.words[5]=filetype;

  strcpy(message.data.chars+24,S0);
  wimpt_complain(wimp_sendmessage(wimp_ESEND,&message,0));

  return(1);
  }

int script_c_chain(script_pass *tail)
  {
  script_stop=1;
  strcpy(script_chain,S0);

  window_poll();
  return(0);
  }

int script_c_swinr(script_pass *tail)
  {
  int r0,r1;

  if (os_swi2r(0x20039,0,(int)S0,&r0,&r1))
    {
    ereps("fSWI name not known '%s'",S0);
    return(0);
    }
  
  return(r0);
  }

int script_c_userget(script_pass *tail)
  {
  tail=tail;

  get_user(&user);
  return(0);
  }

int script_c_userput(script_pass *tail)
  {
  tail=tail;

  put_user(&user);
  return(0);
  }

int script_c_userfind(script_pass *tail)
  {
  return(hash_find(S0));
  }

int script_c_userfindwrite(script_pass *tail)
  {
  return(hash_write(S0,I1));
  }

int script_c_userfree(script_pass *tail)
  {
  tail=tail;

  return(find_free());
  }

int script_c_menucommand(script_pass *tail)
  {
  do_command(S0,1);
  return(0);
  }

int script_c_localtime(script_pass *tail)
  {
  tail=tail;

  return(time(NULL));
  }

int script_c_strftime(script_pass *tail)
  {
  static char temp[256];
  time_t now=I1;
  struct tm *t;

  t=localtime(&now);
  strftime(temp,254,S0,t);

  return((int)temp);
  }

int script_c_usetime(script_pass *tail)
  {
  time_t now=I0;

  memcpy(&script_time,localtime(&now),sizeof(struct tm));
  return(0);
  }

int script_c_sendfile(script_pass *tail)
  {
  if (P1) return(bbs_sendfile(S0,S1,0));
  return(bbs_sendfile(S0,"",0));
  }

int script_c_doing(script_pass *tail)
  {
  send_strmessage(servertask,BBS_USERTASK,portnumber,0,0,S0);
  strcpy(online_users[portnumber].doing,S0);
  return(0);
  }

int script_c_doset(script_pass *tail)
  {
  do_set(I0,0,0,0,0);
  return(0);
  }

int script_c_lastmsg(script_pass *tail)
  {
  tail=tail;
  return(read_msgnumber());
  }

script_commandstr script_command[] =
   { /* Standard commands *******************************************/
     { "integer"         ,0,script_c_integer               ,""      },/* 0 */
     { "int"             ,0,script_c_integer               ,""      },
     { "string"          ,0,script_c_string                ,""      },
     { "char"            ,0,script_c_string                ,""      },
     { "return"          ,2,script_c_return                ," i"    },
     { "$return"         ,2,script_c_stringreturn          ,"ms"    },
     { "for"             ,1,script_c_for                   ,""      },
     { "while"           ,1,script_c_while                 ,""      },
     { "if"              ,1,script_c_if                    ,""      },
     { "switch"          ,1,script_c_switch                ,""      },
     { "case"            ,1,script_c_case                  ,""      },
     { "case$i"          ,1,script_c_casesi                ,""      },
     { "case$"           ,1,script_c_cases                 ,""      },
     { "default"         ,1,script_c_default               ,""      },
     { "repeat"          ,1,script_c_repeat                ,""      },
     { "goto"            ,1,script_c_goto                  ,""      },
     /* String commands *********************************************/
     { "len"             ,0,script_c_len                   ,"ms"    },/* 16 */
     { "asc"             ,0,script_c_asc                   ,"ms"    },
     { "val"             ,0,script_c_val                   ,"ms"    },
     { "$left"           ,0,script_c_stringleft            ,"msmi"  },
     { "$right"          ,0,script_c_stringright           ,"msmi"  },
     { "$mid"            ,0,script_c_stringmid             ,"msmi i"},
     { "$chr"            ,0,script_c_chr                   ,"mi"    },
     { "$str"            ,0,script_c_str                   ,"mi"    },
     { "comparei"        ,0,script_c_comparei              ,"msms"  },
     { "compare"         ,0,script_c_compare               ,"msms"  },
     { "strcmp"          ,0,script_c_strcmp                ,"msms"  },
     { "stricmp"         ,0,script_c_stricmp               ,"msms"  },
     { "instr"           ,0,script_c_instr                 ,"msms"  },
     /* Output commands *********************************************/
     { "prints"          ,0,script_c_prints                ,"ms"    },
     { "print"           ,0,script_c_print                 ,"mi"    },
     { "vdu"             ,0,script_c_vdu                   ,"mi"    },
     /* Port control commands ***************************************/
     { "port_rxbuffer"   ,0,script_c_portrxbuffer          ,""      },
     { "port_txbuffer"   ,0,script_c_porttxbuffer          ,""      },
     { "port_rxclear"    ,0,script_c_portrxclear           ,""      },
     { "port_txclear"    ,0,script_c_porttxclear           ,""      },
     { "port_rx"         ,0,script_c_portrx                ,""      },
     { "port_tx"         ,0,script_c_porttx                ,"mi"    },
     /* Other commands **********************************************/
     { "debug"           ,0,script_c_debug                 ,"mi"    },
     { "oscli"           ,0,script_c_oscli                 ,"ms"    },
     { "system"          ,0,script_c_system                ,"ms"    },
     { "$getenv"         ,0,script_c_getenv                ,"ms"    },
     { "$dirscan"        ,0,script_c_dirscan               ,"ms"    },
     { "swinr"           ,0,script_c_swinr                 ,"ms"    },
     { "swi"             ,0,script_c_swi                   ,""      },
     { "address"         ,0,script_c_address               ,""      },/* 64 */
     /* File commands ***********************************************/
     { "file_openin"     ,0,script_c_fileopenin            ,"mims"  },
     { "file_openout"    ,0,script_c_fileopenout           ,"mims"  },
     { "file_openup"     ,0,script_c_fileopenup            ,"mims"  },
     { "file_close"      ,0,script_c_fileclose             ,"mi"    },
     { "$file_readline"  ,0,script_c_filereadline          ,"mi"    },
     { "file_writeline"  ,0,script_c_filewriteline         ,"mims"  },
     { "file_readbyte"   ,0,script_c_filereadbyte          ,"mi"    },
     { "file_writebyte"  ,0,script_c_filewritebyte         ,"mimi"  },
     { "file_eof"        ,0,script_c_fileeof               ,"mi"    },
     { "file_ptr"        ,0,script_c_fileptr               ,"mi"    },
     { "file_ext"        ,0,script_c_fileext               ,"mi"    },
     { "file_setptr"     ,0,script_c_filesetptr            ,"mimi"  },
     /* Spool, etc commands *****************************************/
     { "end"             ,0,script_c_end                   ,""      },
     { "doubleclick"     ,0,script_c_doubleclick           ,"ms"    },
     { "chain"           ,0,script_c_chain                 ,"ms"    },/* 128 */
     /* Time commands ***********************************************/
     { "pause"           ,0,script_c_pause                 ,"mi"    },
     { "time"            ,0,script_c_time                  ,""      },
     { "clock"           ,0,script_c_time                  ,""      },
     { "localtime"       ,0,script_c_localtime             ,""      },
     { "$strftime"       ,0,script_c_strftime              ,"msmi"  },
     { "usetime"         ,0,script_c_usetime               ,"mi"    },
     /* BBS commands ************************************************/
     { "user_get"        ,0,script_c_userget               ,""      },
     { "user_put"        ,0,script_c_userput               ,""      },
     { "user_find"       ,0,script_c_userfind              ,"ms"    },
     { "user_findwrite"  ,0,script_c_userfindwrite         ,"msmi"  },
     { "user_free"       ,0,script_c_userfree              ,""      },
     { "menucommand"     ,0,script_c_menucommand           ,"ms"    },
     { "sendfile"        ,0,script_c_sendfile              ,"ms s"  },
     { "doing"           ,0,script_c_doing                 ,"ms"    },
     { "do_set"          ,0,script_c_doset                 ,"mi"    },
     { "lastmsg"         ,0,script_c_lastmsg               ,""      },
     /* Terminator **************************************************/
     { ""                ,0,NULL                           ,""      } };

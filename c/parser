/*                _________________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Lloyd Zusman (pd)                           <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Command line parser for menus               <]
Current version   [> 00.01                                       <]
Version date      [> 04-October-1989                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>    This source is COPYRIGHT (c) 1989 by     <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include <ctype.h>

/*

        result=parser(flag,token,maxtok,string,white,break,quote,escape,
                      brkused,next,quoted)

        result:         0 if we haven't reached EOS (end of string), and
                        1 if we have (this is an "int").

        flag:           right now, only the low order 3 bits are used.
                        1 => convert non-quoted tokens to upper case
                        2 => convert non-quoted tokens to lower case
                        0 => do not convert non-quoted tokens
                        (this is a "char").

        token:          a character string containing the returned next token
                        (this is a "char[]").

        maxtok:         the maximum size of "token".  characters beyond
                        "maxtok" are truncated (this is an "int").

        string:         the string to be parsed (this is a "char[]").

        white:          a string of the valid white spaces.  example:

                        char whitesp[]={" \t"};

                        blank and tab will be valid white space (this is
                        a "char[]").

        break:          a string of the valid break characters.  example:

                        char breakch[]={";,"};

                        semicolon and comma will be valid break characters
                        (this is a "char[]").
                        
                        IMPORTANT:  do not use the name "break" as a C
                        variable, as this is a reserved word in C.

        quote:          a string of the valid quote characters.  an example
                        would be

                        char quote[]={"'\"");

                        (this causes single and double quotes to be valid)
                        note that a token starting with one of these characters
                        needs the same quote character to terminate it.

                        for example, 

                        "ABC '
                        
                        is unterminated, but

                        "DEF" and 'GHI'

                        are properly terminated.  note that different quote
                        characters can appear on the same line; only for
                        a given token do the quote characters have to be
                        the same (this is a "char[]").

        escape:         the escape character (NOT a string ... only one
                        allowed).  use zero if none is desired (this is
                        a "char").

        brkused:        the break character used to terminate the current
                        token.  if the token was quoted, this will be the
                        quote used.  if the token is the last one on the
                        line, this will be zero (this is a pointer to a
                        "char").

        next:           this variable points to the first character of the
                        next token.  it gets reset by "parser" as it steps
                        through the string.  set it to 0 upon initialization,
                        and leave it alone after that.  you can change it
                        if you want to jump around in the string or re-parse
                        from the beginning, but be careful (this is a
                        pointer to an "int").

        quoted:         set to 1 (true) if the token was quoted and 0 (false)
                        if not.  you may need this information (for example:
                        in C, a string with quotes around it is a character
                        string, while one without is an identifier).

                        (this is a pointer to a "char").

        Example 1:


*/

/* states */

#define IN_WHITE 0
#define IN_TOKEN 1
#define IN_QUOTE 2
#define IN_OZONE 3

int _p_state;      /* current state      */
unsigned _p_flag;  /* option flag        */
char _p_curquote;  /* current quote char */
int _p_tokpos;     /* current token pos  */

/* routine to find character in string ... used only by "parser" */

int sindex(ch,string)
char ch,*string;

{
  char *cp;
  for(cp=string;*cp;++cp)
    if(ch==*cp)
      return (int)(cp-string);  /* return postion of character */
  return -1;                    /* eol ... no match found */
}
    
/* routine to store a character in a string ... used only by "parser" */

void chstore(string,max,ch)
char *string,ch;
int max;

{
  char c;
  if(_p_tokpos>=0&&_p_tokpos<max-1)
  {
    if(_p_state==IN_QUOTE)
      c=ch;
    else
      switch(_p_flag&3)
      {
        case 1:             /* convert to upper */
          c=toupper(ch);
          break;
  
        case 2:             /* convert to lower */
          c=tolower(ch);
          break;
      
        default:            /* use as is */
          c=ch;
          break;
      }
    string[_p_tokpos++]=c;
  }
  return;
}
  
/* here it is! */

int parser(inflag,token,tokmax,line,white,brkchar,quote,eschar,brkused,next,quoted)
unsigned inflag;
char *token,*line,*white,*brkchar,*quote,*brkused,*quoted,eschar;
int tokmax,*next;

{
  int qp;
  char c,nc;
          
  *brkused=0;           /* initialize to null */          
  *quoted=0;            /* assume not quoted  */

  if(!line[*next])      /* if we're at end of line, indicate such */
    return 1;

  _p_state=IN_WHITE;       /* initialize state */
  _p_curquote=0;           /* initialize previous quote char */
  _p_flag=inflag;          /* set option flag */

  for(_p_tokpos=0;(c=line[*next])!=0;++(*next))      /* main loop */
  {
    if((qp=sindex(c,brkchar))>=0)  /* break */
    {
      switch(_p_state)
      {
        case IN_WHITE:          /* these are the same here ...  */
        case IN_TOKEN:          /* ... just get out             */
        case IN_OZONE:          /* ditto                        */
          ++(*next);
          *brkused=brkchar[qp];
          goto byebye;
        
        case IN_QUOTE:           /* just keep going */
          chstore(token,tokmax,c);
          break;
      }
    }
    else if((qp=sindex(c,quote))>=0)  /* quote */
    {
      switch(_p_state)
      {
        case IN_WHITE:   /* these are identical, */
          _p_state=IN_QUOTE;        /* change states   */
          _p_curquote=quote[qp];         /* save quote char */
          *quoted=1;    /* set to true as long as something is in quotes */
          break;
  
        case IN_QUOTE:
          if(quote[qp]==_p_curquote)    /* same as the beginning quote? */
          {
            _p_state=IN_OZONE;
            _p_curquote=0;
          }
          else
            chstore(token,tokmax,c);    /* treat as regular char */
          break;

        case IN_TOKEN:
        case IN_OZONE:
          *brkused=c;                   /* uses quote as break char */
          goto byebye;
      }
    }
    else if((qp=sindex(c,white))>=0)       /* white */
    {
      switch(_p_state)
      {
        case IN_WHITE:
        case IN_OZONE:
          break;                /* keep going */
          
        case IN_TOKEN:
          _p_state=IN_OZONE;
          break;
          
        case IN_QUOTE:
          chstore(token,tokmax,c);     /* it's valid here */
          break;
      }
    }
    else if(c==eschar)                  /* escape */
    {
      nc=line[(*next)+1];
      if(nc==0)                 /* end of line */
      {
        *brkused=0;
        chstore(token,tokmax,c);
        ++(*next);
        goto byebye;
      }
      switch(_p_state)
      {
        case IN_WHITE:
          --(*next);
          _p_state=IN_TOKEN;
          break;

        case IN_TOKEN:
        case IN_QUOTE:
          ++(*next);
          chstore(token,tokmax,nc);
          break;

        case IN_OZONE:
          goto byebye;
      }
    }
    else        /* anything else is just a real character */
    {
      switch(_p_state)
      {
        case IN_WHITE:
          _p_state=IN_TOKEN;        /* switch states */
          
        case IN_TOKEN:           /* these 2 are     */
        case IN_QUOTE:           /*  identical here */
          chstore(token,tokmax,c);
          break;

        case IN_OZONE:
          goto byebye;
      }
    }
  }             /* end of main loop */

byebye:
  token[_p_tokpos]=0;   /* make sure token ends with EOS */
  
  return 0;
  
}


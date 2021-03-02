/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 05-March-1990                               <]
                  [>                                             <]
Module name       [> Script command header                       <]
Current version   [> 00.01                                       <]
Version date      [> 16-February-1993                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1990-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

typedef struct  { union { char *s; int i; } d; int p; } script_pass;

typedef struct { char label[20];
                 int  level;
                 char *pointer;                   } script_labelstr;
      
typedef struct { char command[19]; char type;
                 int  (*function)(script_pass *);
                 char params[8]; } script_commandstr;
                          

typedef struct { char name[20];
                 char *pointer;                   } script_functionstr;

typedef struct { char command[20];
                 char *(*function)(char *,char*); } script_stringfunctionstr;
                          
typedef struct { char name[48];
                 int  type;
                 int  maxlen;
                 int  arraysize;
                 int  value;    } script_variable;

extern int  script_level,script_keys,script_keyin,script_keyout,script_cankill;
extern char script_keybuffer[256];

extern void script_add_i_variable(char*,int,int,int),
            script_add_s_variable(char*,int,int,int,char*),
            script_add_p_variable(char*,int,int,int,void*,int),
            script_closedown(int),
            script_run(char*),
            script_unnest(void),script_enter(void),
            script_exit(void),
            script_run2(char*,int),
            script_bomb(void);
extern int  script_initialise(void),script_get_i_variable(char*,int*),
            script_put_i_variable(char*,int),script_isoperator(int),
            script_executecommand(char*,int*),
            script_executeblock(char*,int*,int),
            script_integer_evaluate(char*,int*),script_main(char*,int),
            script_dofunction(char*,char*,int*),
            script_runfunction(char*,char*,int*),
            script_get_s_variable(char*,char*),
            script_put_s_variable(char*,char*),
            script_string_evaluate(char*,char*),
            script_include(char*,char*,int),
            script_includefile(char*,char*,char**),
            script_findline(char*),
            script_findcommand(char*,void*);
extern char *script_infix_to_polish(char*,char*),
            *script_polish_evaluate(char*,int*);
extern script_variable *script_findvariable(char*,int*);

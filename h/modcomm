/*                _________________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Module interface header                     <]
Current version   [> 00.11                                       <]
Version date      [> 06-November-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

extern int  mod_readarea(int,void*),mod_writearea(int,void*),
            mod_readdata(int,void*),mod_writedata(int,void*),
            mod_readdatablock(int,int,void*),
            mod_writedatablock(int,int,void*),
            mod_readlookup(int,void*),mod_writelookup(int,void*),
            mod_lockdata(int),mod_locklookup(void),mod_lastlookup(void),
            mod_getlastlookup(void),mod_getlastarea(void),
            mod_readuserdata(int,void*),mod_writeuserdata(int,void*),
            mod_readuserlookup(int,void*),mod_writeuserlookup(int,void*),
            mod_extentuser(void),mod_maxuser(void),mod_readcallcount(void),
            mod_filenumber(void),mod_readfilenumber(void),
            mod_lookuplength(void),mod_datalength(void),
            mod_dataext(void),mod_lookupext(void);
extern void mod_freedata(int),mod_freelookup(int),mod_saveall(void),
            mod_openall(void),mod_closeall(void),mod_clearflag(void),
            mod_inccallcount(void),mod_ensurelookup(void),
            mod_ensuredata(void),mod_savemaps(void),mod_ensureuser(void);
extern char *mod_getstatuspointer(void),*mod_lookupmap(void),
            *mod_datamap(void);                           

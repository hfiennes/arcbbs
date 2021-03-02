/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 06-November-1989                            <]
                  [>                                             <]
Module name       [> Includes header                             <]
Current version   [> 00.02                                       <]
Version date      [> 24-November-1989                            <]
State             [> Unfinished                                  <]
                  [>_____________________________________________<]
*/
            
#include <stdio.h>
#include <stddef.h>
#include <setjmp.h>
#include <time.h>
#include <stdlib.h>
#include <ctype.h>
#include <string.h>

#include "wimp.h"        /*  access to WIMP SWIs                      */
#include "wimpt.h"       /*  wimp task facilities                     */
#include "win.h"         /*  registering window handlers              */
#include "event.h"       /*  poll loops, etc                          */
#include "baricon.h"     /*  putting icon on icon bar                 */
#include "sprite.h"      /*  sprite operations                        */
#include "werr.h"        /*  error reporting                          */
#include "res.h"         /*  access to resources                      */
#include "resspr.h"      /*  sprite resources                         */
#include "template.h"    /*  reading in template file                 */
#include "bbc.h"         /*  olde-style graphics routines             */
#include "os.h"          /*  low-level RISCOS access                  */
#include "dbox.h"        /*  dialogue box handling                    */
#include "saveas.h"      /*  data export from dbox by icon dragging   */
#include "define.h"

extern void error(char*, ...),syslog(char*,char*, ...);

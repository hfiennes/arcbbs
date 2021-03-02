/*
 * (C) Copyright 1988, Software by Sagredo, "est Omnibus"
 *
 *      this product may be freely used, duplicated and distributed
 *      provided that no charge is made for this code, and that this
 *      copyright notice is preserved in all copies and derivative works.
 *
 * module:
 *      ed_defs.h               version 1.6
 *
 * purpose:
 *      basic structure and constant definitions used throughout editor
 */

/* convenient defaults - change them or override them as you wish       */
#define MAX_COL         80              /* default line width           */
#define MAX_ROW         24              /* default screen height        */
#define MAX_LINES       100             /* default longest file length  */
#define TAB_WIDTH       4               /* default tab width            */
#define WRAP_MGN        4               /* default word-wrap margin     */

/* basic parameters - change them as necessary                          */
#define MAX_ICODE       40              /* number of editor commands    */
#define MAX_OCODE       24              /* number of output seqs        */
#define MAX_CMDS        80              /* number user command seqs     */
#define MAX_ISEQ_LEN    8               /* longest allowable input seq  */
#define MAX_OSEQ_LEN    16              /* longest allowable output seq */

/*
 * format of the command lexing table - shared between config and input modules
 */
struct ed_cmd 
{       int     c_code;                 /* associated command routine   */
        int     c_length;               /* required sequence length     */
        char    c_string[MAX_ISEQ_LEN]; /* input command string         */
};                                      /* code == zero -> unused entry */
                                        /* if length is zero we need a  */
                                        /* complete match.  If non-zero */
                                        /* we need the first char and   */
                                        /* that many characters         */

#define commands ed_commands

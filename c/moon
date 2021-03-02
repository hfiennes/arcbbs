/*
 *              Phase of the Moon Conversion.
 *
 * Synopsis
 *
 *      char    *moontxt(buffer, year, month, day)
 *      char    *buffer;        -- Output goes here.
 *      int     year;           -- Year         1979 = 1979
 *      int     month;          -- Month        Jan. = 1
 *      int     day;            -- Day          1    = 1
 *
 * The phase of the moon (in readable ascii) is written into the buffer,
 * followed by a null.  The routine returns a pointer to the trailing null.
 *
 */


#include "include.h"

char *phasetxt[] = {
        "new",
        "waxing crescent",
        "in its first quarter",
        "waxing gibbous",
        "full",
        "waning gibbous",
        "in its last quarter",
        "waning crescent"
};

static int day_year[] = {                       /* Days in year for each month  */
        -1, -1, 30, 58, 89, 119, 150, 180, 211, 241, 272, 303, 333
};                                      /* Note: Jan. 1 will equal zero */

char *moontxt(buffer, year, month, day)
char    *buffer;                        /* Where to put the text        */
int     year;                           /* Year, 1978 = 1978            */
int     month;                          /* Month, Jan = 1               */
int     day;                            /* Day, 1 = 1                   */
/*
 * Output the phase of the moon for the given year, month, day.
 * The routine calculates the year's epact (the age of the moon on Jan 1.),
 * adds this to the number of days in the year, and calculates the phase
 * of the moon for this date.
 *
 * In the algorithm:
 *
 *      diy     Is the day of the year - 1 (i.e., Jan 1 is day 0).
 *
 *      golden  Is the number of the year in the Mentonic cycle, used to
 *              determine the position of the calender moon.
 *
 *      epact   Is the age of the calender moon (in days) at the beginning
 *              of the year.  To calculate epact, two century-based
 *              corrections are applied:
 *              Gregorian:      (3 * cent)/4 - 12
 *                      is the number of years such as 1700, 1800 when
 *                      leap year was not held.
 *              Clavian:        (((8 * cent) + 5) / 25) - 5
 *                      is a correction to the Mentonic cycle of about
 *                      8 days evry 2500 years.  Note that this will
 *                      overflow 16 bits in the year 409600.  Beware.
 *
 * The algorithm is accurate for the Gregorian calender only.
 *      
 * The magic numbers used in the phase calculation are as follows:
 *       29.5           The moon's period in days.
 *      177             29.5 scaled by 6
 *       22             (29.5 / 8) scaled by 6 (this gets the phase)
 *       11             ((29.5 / 8) / 2) scaled by 6
 *
 * Theoretically, this should yield a number in the range 0 .. 7.  However,
 * two days per year, things don't work out too well.
 *
 * Epact is calculated by the algorithm given in Knuth vol. 1 (calculation
 * of Easter).  See also the article on Calenders in the Encyclopaedia
 * Britannica and Knuth's algorithm in CACM April 1962, page 209.
 */
{
        int             phase;          /* Moon phase                   */
        int             cent;           /* Century number (1979 = 20)   */
        int             epact;          /* Age of the moon on Jan. 1    */
        int             diy;            /* Day in the year              */
        int             golden;         /* Moon's golden number         */

        if (month < 0 || month > 12) month = 0; /* Just in case         */
        diy = day + day_year[month];            /* Day in the year      */
        if ((month > 2) && ((year % 4 == 0) &&
                        ((year % 400 == 0) || (year % 100 != 0))))
                diy++;                          /* Leapyear fixup       */
        cent = (year / 100) + 1;                /* Century number       */
        golden = (year % 19) + 1;               /* Golden number        */
        epact = ((11 * golden) + 20             /* Golden number        */
                + (((8 * cent) + 5) / 25) - 5   /* 400 year cycle       */
                - (((3 * cent) / 4) - 12)) % 30;/* Leap year correction */
        if (epact <= 0)
                epact += 30;                    /* Age range is 1 .. 30 */
        if ((epact == 25 && golden > 11) || epact == 24)
                epact++;
        /*
         * Calculate the phase, using the magic numbers defined above.
         * Note that (phase and 7) is equivalent to (phase mod 8) and
         * is needed on two days per year (when the algorithm yields 8).
         */
        phase = (((((diy + epact) * 6) + 11) % 177) / 22) & 7;
        strcpy(buffer, phasetxt[phase]);
}


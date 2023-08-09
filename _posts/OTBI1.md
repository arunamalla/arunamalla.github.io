About This article talk about the manipulation of a presentation variable with a date datatype.

Dashboard Prompt input formats and presentation variable values for DATE & DATETIME columns are standardized to YYYY-MM-DD & YYYY-MM-DD HH24:MI:SS

Articles Related:

Oracle Forum threads Related:


The big mistake 

One big mistake that is made with the date, is that people may confuse between :
the format of a date and the data type of a date. Why ? Because a lot of database include an implicit datatype transformation from a string into a date. See this example below on Oracle :

select DAY_NAME from times where time_id = '01-JAN-95';

## DAY_NAME

Sunday PL/SQLDownload Oracle take the string '01-JAN-95' transform it as a date and perform the query.

But what happen if you change the format of the date with the NLS_DATE_FORMAT parameter because you are in a multi-language environment :

ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY/MM/DD';

Session altered.

select DAY_NAME from times where time_id = '01-JAN-95'; select DAY_NAME from times where time_id = '01-JAN-95' \* ERROR at line 1: ORA-01858: a non-numeric character was found where a numeric was expected PL/SQLDownload You fired an error because Oracle expected an other date format to be able to transform it as a date data type.

To be able to support the localization, you must send to the database not a string but a real value with a date data type. You can do that with the TO_DATE function in Oracle.

select DAY_NAME from times where time_id = TO_DATE('01-JAN-1995','DD-MONTH-YYYY');

## DAY_NAME

Sunday PL/SQLDownload It works !

Then especially when you work in a multi-language environment, you always must set in a filter not a formatted string but a real value with a date data type. The DATE function of the OBIEE logical Sql have this purpose.

To understand more the difference between the data type and the date format, check out this article : Toad - The date format with null and decode

The date function and its date format In OBIEE, an equivalent of the function TO_DATE is the DATE function which has this syntax

DATE 'YYYY-MM-DD' Plain textDownload The date format is unique where :

YYYY is the Year with 4 numbers MM is the Month of year 01, 02...12 DD is the Day of the month in numbers (i.e. 28) And you use it with a presentation variable (for instance in a filter) as

DATE '@MyDatePresentationVariable{1995-07-06}' PL/SQLDownload See the paragraph examples below to have more insights

In fact, with Oracle, you will receive :

select date '2009-12-15' from dual--TRUE select date '2009-15-12' from dual--ORA-01843: not a valid month select date '2009/10/15' from dual--ORA-01861: literal does not match format string Plain textDownload Understanding the datatype of a presentation variable Before going further, you have to be sure that you pass the date data type to your presentation variable. See this paragraph which show you how to verify it : understanding the datatype of a presentation variable

The localization and the filter When you set up a filter on a date, you see a string but in background, Oracle BI Presentation Service see it as a real date data type.

Obiee Filter On Date

To demonstrate it, below is a little report in a dashboard, the first one with the LOCALE value as English and the second one as French.

To change the LOCALE value, you can do it on the connexion windows or in your account settings (Settings / Account) Obiee Preference Myaccount Locale Weblanguage

In English In French Obiee Filter Date In En Locale Obiee Filter Date In Fr Locale Example In Edit-Box Dashboard prompt In all language configuration (french, english, ...) , if you use a edit-box dashboard prompt, you must use this format :

Obiee Dashboard Prompt Edit Box Date

In a formula CASE WHEN Calendar."Time Id" = DATE '@MyDatePresentationVariable{1995-07-06}' THEN 'It''s the same date than the presentation variable' ELSE 'It''s not the same date than the presentation variable' END PL/SQLDownload In a filter To transform the default value as a date data type, you have to use this statement :

@MyFilterDate{cast('1995-06-30' as date)} PL/SQLDownload or this one :

@MyFilterDate{date '1995-06-30'} PL/SQLDownload Obiee Filter Default Value Date Presentation Variable

of in the advanced Sql (Advanced / Convert this filter in Sql):

Calendar."Time Id" \<= date'@MyFilterDate{1995-07-06}' PL/SQLDownload Documentation / Reference For the date example, Forum Thread with Goran epmos/faces/ui/km/DocumentDisplay.jspx

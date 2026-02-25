/* USER SETUP  */
/* A) Set your project folder path  */
%let proj = YOUR_PROJECT_FOLDER_PATH;   /* e.g., C:\Users\project\lbw_project */

/*  B) Input file */
%let datafile = &proj./data_raw/baystate_lbw.csv;  /* or .xlsx */

/*C) VARIABLE NAMES  */
%let y_birthwt   = birth_weight;        /* numeric grams, e.g., 3200 */
%let y_lbw       = lbw;                 /* binary: 1=LBW (<2500g), 0=Not LBW */
%let x_smoke     = smoke;               /* binary: 1=smoker, 0=non-smoker */

/* Covariates used in adjusted model (EDIT names/coding as needed) */
%let x_race      = race;                /* categorical */
%let x_htn       = hypertension;        /* binary */
%let x_age       = maternal_age;        /* numeric */
%let x_prenatal  = prenatal_visits;     /* numeric or categorical */
%let x_parity    = parity;              /* numeric or categorical */

/* Create a covariate list for models */
%let covars = &x_race &x_htn &x_age &x_prenatal &x_parity;
/*IMPORT DATA */
/* If CSV */
proc import datafile="&datafile"
    out=work.raw
    dbms=csv
    replace;
    guessingrows=max;
run;

/* If you have Excel instead, comment CSV import and use this:
proc import datafile="&datafile"
    out=work.raw
    dbms=xlsx
    replace;
run;*/

proc contents data=work.raw order=varnum;
run;

/* 2) DATA CLEANING + LBW CREATION (IF NEEDED) */

data work.analysis;
set work.raw;
if not missing(&y_birthwt) then do;
    if &y_birthwt < 2500 then &y_lbw = 1;
    else &y_lbw = 0;
end;
*/

/* Ensure binary coding is 0/1 for outcome & smoking*/
* Example if smoke is "Yes"/"No":
* if &x_smoke="Yes" then &x_smoke=1; else if &x_smoke="No" then &x_smoke=0;

* Example if lbw is "LBW"/"Normal":
* if &y_lbw="LBW" then &y_lbw=1; else if &y_lbw="Normal" then &y_lbw=0;
run;

/* missingness check for key variables */
proc means data=work.analysis n nmiss;
var &y_lbw &x_smoke &x_htn &x_age;
run;
proc freq data=work.analysis;
tables &y_lbw &x_smoke / missing;
run;

/* DESCRIPTIVE STATISTICS */

/* Overall descriptives */
proc freq data=work.analysis;
tables &x_smoke &y_lbw &x_htn &x_race / missing;
run;

proc means data=work.analysis mean std median min max maxdec=2;
var &x_age &y_birthwt;
run;

/* Descriptives by LBW status */
proc freq data=work.analysis;
tables (&x_smoke &x_htn &x_race)*&y_lbw / chisq expected norow nocol nopercent;
run;

proc means data=work.analysis mean std median maxdec=2;
class &y_lbw;
var &x_age &y_birthwt;
run;

/* Fisher's exact (useful if small cells) */
proc freq data=work.analysis;
tables &x_smoke*&y_lbw / fisher chisq;
run;

/* 4) UNADJUSTED LOGISTIC REGRESSION*/

ods graphics on;

/* Unadjusted: Smoking -> LBW */
proc logistic data=work.analysis descending;
model &y_lbw(event='1') = &x_smoke / clodds=wald;
oddsratio &x_smoke;
title "Unadjusted Logistic Regression: Smoking and Low Birth Weight";
run;

/* Optional: unadjusted models for other predictors */
proc logistic data=work.analysis descending;
class &x_race (param=ref ref=first) /;
model &y_lbw(event='1') = &x_race / clodds=wald;
title "Unadjusted Logistic Regression: Race and Low Birth Weight";
run;

proc logistic data=work.analysis descending;
model &y_lbw(event='1') = &x_htn / clodds=wald;
oddsratio &x_htn;
title "Unadjusted Logistic Regression: Hypertension and Low Birth Weight";
run;

ods graphics off;
title;

/*  5) ADJUSTED LOGISTIC REGRESSION (MAIN MODEL) */

/* Adjusted model: Smoking + covariates */
proc logistic data=work.analysis descending;
class &x_race (param=ref ref=first) /;
model &y_lbw(event='1') = &x_smoke &covars / clodds=wald;
oddsratio &x_smoke;
oddsratio &x_htn;
title "Adjusted Logistic Regression: Smoking and Low Birth Weight (Baystate)";
run;

title;

/*  6) MODEL CHECKS (OPTIONAL)*/

/* Hosmer-Lemeshow goodness-of-fit */
proc logistic data=work.analysis descending;
class &x_race (param=ref ref=first) /;
model &y_lbw(event='1') = &x_smoke &covars / lackfit;
title "Model Fit: Hosmer-Lemeshow (LACKFIT)";
run;

title;

/* 7) EXPORT OUTPUT (OPTIONAL) */
ods excel file="&proj./output/lbw_results.xlsx";
proc logistic data=work.analysis descending;
class &x_race (param=ref ref=first) /;
model &y_lbw(event='1') = &x_smoke &covars / clodds=wald;
run;
ods excel close;
*/

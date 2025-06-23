# HEALRise
Code for HEALRise Project 2025

This code is for HEALRISE (HEALth, Racism, Inequities, and Social Epidemiology) Scholarship by the Department of Epidemiology at UCLA Fielding School of Public Health. The projec is titled "Assessing the Effectiveness of Gun Violence Restraining Orders in Reducing Intimate Partner Homicides in California."
/**************************************************************
Name: HEALRise Project Code
Created by: Seda Shirinian
Creation Date: 5/5/2025 
Purpose: Complete Code for HEALRise
**************************************************************/

/*importing code for GVRO data*/
%web_drop_table(WORK.IMPORT3);

FILENAME REFFILE '/home/u63767048/Heal RISE/merge-csv_5.4.25.xlsx';

PROC IMPORT DATAFILE=REFFILE
	DBMS=XLSX
	OUT=WORK.IMPORT3;
	GETNAMES=YES;
RUN;

PROC CONTENTS DATA=WORK.IMPORT3; RUN;
%web_open_table(WORK.IMPORT3);


/*importing Code for Homicide Actuals Data*/
/* Generated Code (IMPORT) */
/* Source File: HomicideActuals19872023.csv */
/* Source Path: /home/u63767048/Heal RISE */
/* Code generated on: 5/16/25, 11:05 AM */

%web_drop_table(WORK.IMPORT1);
FILENAME REFFILE '/home/u63767048/Heal RISE/HomicideActuals19872023.csv';
PROC IMPORT DATAFILE=REFFILE
	DBMS=CSV
	OUT=WORK.IMPORT1;
	GETNAMES=YES;
RUN;

PROC CONTENTS DATA=WORK.IMPORT1; RUN;
%web_open_table(WORK.IMPORT1);


proc print data=work.import1(obs=10);
    title "First 10 Rows - Import2 Dataset";
run;

proc print data=work.import1(firstobs=2920 obs=2930);
    title "Problematic Rows - Import2 Dataset";
run;




data work.import3;
    set work.import3;
    if _N_ > 1; /* Keep observations starting from the 1st row */
run;

proc print data=work.import3(obs=10); /* Preview first 10 rows */
run;

data work.import3_clean;
    set work.import3 (drop=G H I J K);
    label
        "merge-csv_5.4.25"n = "Year"
        B                   = "County"
        C                   = "Order Type"
        D					= "Order Description"
        E                   = "Requesting Agency"
        F                   = "Number of Orders";
        
run;

proc print data=work.import3_clean label;
run;

proc contents data=work.import3_clean; run;

proc sgplot data=work.import3_clean;
    vbar C / stat=freq;
    xaxis label="Order Type";
    yaxis label="Count (N)";
    title "Frequency of Each Type of Restraining Order from 2016-2024";
run;

/*EGV: Emergency—21 days*/
/*HGV: Order After Hearing on EPO-002 1-5 years*/ /*HGV= EGV+OGV*/
/*OGV=Order After Hearing 1-5 years*/
/*TGV= Temporary-21 days*/

proc sgplot data=work.import3_clean;
    vbar "merge-csv_5.4.25"n / group=C stat=freq groupdisplay=cluster;
    xaxis label="Year";
    yaxis label="Count (N)";
    title "Restraining Order Types by Year (2016–2024)";
run;

proc freq data=work.import3_clean;
    tables E*C / nocol nopercent norow;
    title "Cross-tab of Requesting Agency by Order Type";
run;


data import3_clean_numeric;
    set work.import3_clean;
    Orders_Num = input(F, best12.);
run;

proc sql;
    select sum(Orders_Num) as Total_Orders
    from import3_clean_numeric;
quit;

proc sql;
    create table orders_by_year as
    select "merge-csv_5.4.25"n as Year, 
           sum(Orders_Num) as Total_Orders
    from import3_clean_numeric
    group by "merge-csv_5.4.25"n
    order by Year;
quit;

proc sgplot data=orders_by_year;
    vbar Year / response=Total_Orders datalabel;
    xaxis label="Year";
    yaxis label="Number of GVROs";
    title "Distribution of GVROs Issued by Year";
run;


/*top ten counties*/
data work.import3_clean_numeric;
    set work.import3_clean;
    Orders_Num = input(F, best12.);
run;

proc sql;
    create table county_order_summary as
    select B as County,
           sum(Orders_Num) as Total_Orders
    from work.import3_clean_numeric
    group by B
    order by Total_Orders desc;
quit;

proc print data=county_order_summary (obs=10);
    title "Top 10 Counties by Total Number of Restraining Orders";
run;

proc sgplot data=county_order_summary (obs=10);
    vbar County / response=Total_Orders datalabel;
    xaxis label="County";
    yaxis label="Total Orders";
    title "Top 10 Counties by Total Number of Restraining Orders (2016–2024)";
run;

proc sql;
    create table orders_by_type_year as
    select "merge-csv_5.4.25"n as Year, 
           C as Order_Type, 
           sum(Orders_Num) as Orders
    from work.import3_clean_numeric
    group by Year, Order_Type;
quit;






/*HOMICIDE DATA*/

data import1_fixed;
    set work.import1;
    length v_race_char $8;

    /* Check variable type using VTYPE function */
    if vtype("V race"n) = 'N' then 
        v_race_char = strip(put("V race"n, best8.));  /* convert numeric to char */
    else 
        v_race_char = strip("V race"n);  /* already character, just clean */
run;
proc print data=import1_fixed(firstobs=64203 obs=64210);
    var "V race"n v_race_char;
run;

proc contents data=work.import1;
run;

proc freq data=work.import1;
    tables "V race"n / missing;
run;





proc contents data=work.import1;
run;

data work.homicide_subset;
    set work.import1;
    if "death YR"n >= 2010;
run;





proc format;
    value weapfmt
        0  = "Unknown"
        1  = "Firearm (Unspecified)"
        2  = "Handgun (Pistol/Revolver)"
        3  = "Rifle"
        4  = "Shotgun"
        5  = "Other Firearm"
        6  = "Knife or Stabbing Instrument"
        7  = "Blunt Object"
        8  = "Personal Weapon (Hands/Feet, etc.)"
        9  = "Poison"
        10 = "Drugs/Narcotics (Overdose)"
        11 = "Rope/Garrote"
        13 = "Arson/Fire"
        14 = "Explosion"
        15 = "Other"
        16 = "Neglect"
        17 = "Pellet Gun"
        20 = "Drowning"
        25 = "Asphyxiation";
        
      value $racefmt
        '0' = 'Unknown'
        'X' = 'Unknown'
        '1' = 'White'
        'W' = 'White'
        '2' = 'Hispanic'
        'H' = 'Hispanic'
        '3' = 'Black'
        'B' = 'Black'
        '4' = 'American Indian'
        'I' = 'American Indian'
        '5' = 'Chinese'
        'C' = 'Chinese'
        '6' = 'Japanese'
        'J' = 'Japanese'
        '7' = 'Filipino'
        'F' = 'Filipino'
        '8' = 'Other'
        'O' = 'Other'
        '9' = 'Pacific Islander'
        'P' = 'Pacific Islander'
        'A' = 'Other Asian'
        'D' = 'Cambodian'
        'G' = 'Guamanian'
        'K' = 'Korean'
        'L' = 'Laotian'
        'S' = 'Samoan'
        'U' = 'Hawaiian'
        'V' = 'Vietnamese'
        'Z' = 'Asian Indian';
run;
        



/*HOMICIDES BY WEAPON*/
data work.homicide_grouped;
    set work.homicide_subset;
    
    length Weapon_Group $12;
    
    if Weap in (1, 2, 3, 4, 5, 17) then Weapon_Group = "Gun";
    else if Weap ne . then Weapon_Group = "Other";
    else Weapon_Group = "Missing";
run;

proc freq data=work.homicide_grouped;
    tables Weapon_Group / nocum nopercent;
    title "Gun vs. Other Weapons in Homicide Cases (2016+)";
run;

proc sgplot data=work.homicide_grouped;
    vbar Weapon_Group / stat=freq datalabel;
    yaxis label="Number of Cases";
    title "Gun vs. Other Weapons in Homicide Cases";
run;



/*restricted to IPV*/
proc freq data=work.homicide_grouped;
    where VO_1 in (1, 2, 3, 4, 22, 23, 24, 25, 29);
    tables Weapon_Group / nocum nopercent;
    title "Gun vs. Other Weapons in IPV Cases";
run;


proc sgplot data=work.homicide_grouped;
    where VO_1 in (1, 2, 3, 4, 22, 23, 24, 25, 29);
    vbar Weapon_Group / stat=freq datalabel;
    yaxis label="Number of Cases";
    title "Gun vs. Other Weapons in IPV Cases 2010-2022";
run;



/*HOMICIDES BY RELATIONSHIP*/
proc format;
    value vofmt
        1  = "Husband"
        2  = "Wife"
        3  = "Common-law Husband"
        4  = "Common-law Wife"
        5  = "Mother"
        6  = "Father"
        7  = "Son"
        8  = "Daughter"
        9  = "Brother"
        10 = "Sister"
        11 = "In-law"
        12 = "Stepfather"
        13 = "Stepmother"
        14 = "Stepson"
        15 = "Stepdaughter"
        16 = "Other Family"
        20 = "Neighbor"
        21 = "Acquaintance"
        22 = "Boyfriend/Ex-Boyfriend"
        23 = "Girlfriend/Ex-Girlfriend"
        24 = "Ex-Husband"
        25 = "Ex-Wife"
        26 = "Employer"
        27 = "Employee"
        28 = "Friend"
        29 = "Homosexual Relationship"
        30 = "Other, Known to Victim"
        40 = "Stranger"
        45 = "Gang Member"
        50 = "Relationship Undetermined";
run;

data work.homicide_grouped;
    set work.homicide_grouped; 
    length Relationship_Group $20;
    if VO_1 in (1, 2, 3, 4, 22, 23, 24, 25, 29) then Relationship_Group = "Romantic Partner";
    else if VO_1 ne . then Relationship_Group = "Other";
    else Relationship_Group = "Missing";
run;

proc freq data=work.homicide_grouped;
    tables Relationship_Group / nocum nopercent;
    title "Romantic Partners vs. Others in Homicide Cases";
run;

proc sgplot data=work.homicide_grouped;
    vbar Relationship_Group / stat=freq datalabel;
    yaxis label="Number of Cases";
    title "Romantic Partners vs. Others in Homicide Cases";
run;

proc sgplot data=work.homicide_subset_filtered;
    vbar VO_1 / stat=freq;
    format VO_1 vofmt.;
    xaxis label="Victim-Offender Relationship";
    yaxis label="Number of Cases";
    title "Frequency by Victim-Offender (Victim's Relationship to Offender) (2010-2022)";
run;

proc sgplot data=work.homicide_subset_filtered;
    vbar VO_1 / stat=freq;
    format VO_1 vofmt.;
    xaxis label="Victim-Offender Relationship";
    yaxis label="Number of Cases" max=2000;
    title "Frequency by Victim-Offender (Victim's Relationship to Offender) (2010-2022)";
run;




/* Homicide data cleaning - work.import1 */
data work.homicide_clean;
    set work.import1;
    /* Retain only rows where 'Rpt Yr' is between 2010 and 2023 */
    where "Rpt Yr"n >= 2010 and "Rpt Yr"n <= 2023;
    rename "Rpt Yr"n = Rpt_Yr; /* Rename to a standard format */
run;

proc contents data=work.homicide_clean;
    title "Dataset After Renaming Rpt Yr to Rpt_Yr";
run;

proc print data=work.homicide_clean(obs=10);
    title "First 10 Rows of Filtered Data (2010-2023)";
run;

data work.homicide_clean_v2;
    set work.homicide_clean(drop=NCIC BCS "Week day"n LOC);
run;

proc contents data=work.homicide_clean_v2;
    title "Dataset After Dropping Unnecessary Columns";
run;

proc print data=work.homicide_clean_v2(obs=10);
    title "First 10 Rows - Cleaned Data Without Unnecessary Columns";
run;

data work.homicide_clean_v2;
    set work.homicide_clean_v2;

    /* Create Total_CA_Population variable based on Rpt_Yr */
    select (Rpt_Yr);
        when (2010) Total_CA_Population = 37319550;
        when (2011) Total_CA_Population = 37636311;
        when (2012) Total_CA_Population = 37944551;
        when (2013) Total_CA_Population = 38253768;
        when (2014) Total_CA_Population = 38586706;
        when (2015) Total_CA_Population = 38904296;
        when (2016) Total_CA_Population = 39149186;
        when (2017) Total_CA_Population = 39337785;
        when (2018) Total_CA_Population = 39437463;
        when (2019) Total_CA_Population = 39437610;
        when (2020) Total_CA_Population = 39501653;
        when (2021) Total_CA_Population = 39142991;
        when (2022) Total_CA_Population = 39029342;
        when (2023) Total_CA_Population = 39200000;
        otherwise Total_CA_Population = .;
    end;
run;

proc freq data=work.homicide_clean_v2;
    tables Rpt_Yr / nocum nopercent;
    title "Number of Observations by Year (2010-2023)";
run;


proc sql;
    create table yearly_obs as
    select Rpt_Yr, 
           count(*) as Num_Observations
    from work.homicide_clean_v2
    group by Rpt_Yr
    order by Rpt_Yr;
quit;

proc print data=yearly_obs;
    title "Number of Observations by Year (2010-2023)";
run;

data yearly_obs_prop;
    merge yearly_obs(in=a) work.homicide_clean_v2(in=b keep=Rpt_Yr Total_CA_Population);
    by Rpt_Yr;
    if a;  /* Keep only rows with observation counts */

    /* Calculate Proportion */
    if Total_CA_Population > 0 then 
        Proportion = Num_Observations / Total_CA_Population;
    else 
        Proportion = .;
run;

proc sgplot data=yearly_obs_prop;
    series x=Rpt_Yr y=Proportion / lineattrs=(thickness=2 color=blue);
    yaxis label="Proportion (Observations / Population)";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Proportion of Homicides Relative to CA Population (2010-2023)";
run;


proc reg data=yearly_obs_prop;
    model Proportion = Rpt_Yr / clb;
    title "Linear Trend Analysis of Proportion Over Time (2010-2023) - Beta Coefficients and Confidence Intervals";
run;

proc sgplot data=yearly_obs_prop;
    reg x=Rpt_Yr y=Proportion / lineattrs=(color=red thickness=2);
    series x=Rpt_Yr y=Proportion / lineattrs=(color=blue thickness=1);
    yaxis label="Proportion (Homicides / CA Population)";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Proportion of Observations Relative to CA Population (2010-2023) with Regression Line";
run;


data its_data;
    set yearly_obs_prop;
    
    /* Time variable */
    Time = Rpt_Yr - 2010 + 1;
    
    /* Intervention variable */
    Intervention = (Rpt_Yr >= 2016);
    
    /* Time After Intervention */
    Time_After = Time - (2016 - 2010 + 1);
    if Rpt_Yr < 2016 then Time_After = 0;
run;

proc print data=its_data(obs=10);
    var Rpt_Yr Time Intervention Time_After Proportion;
    title "Data for Interrupted Time Series Analysis";
run;
proc reg data=its_data;
    model Proportion = Time Intervention Time_After / clb;
    title "Interrupted Time Series Analysis: Effect of 2016 on Proportion";
run;
proc sgplot data=its_data;
    scatter x=Rpt_Yr y=Proportion / markerattrs=(color=blue symbol=circlefilled);
    series x=Rpt_Yr y=Proportion / lineattrs=(color=blue thickness=1);
    refline 2016 / axis=x lineattrs=(color=red thickness=2 pattern=shortdash);
    yaxis label="Proportion (Observations / Population)";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Time Series Analysis (2010-2023) with 2016 Intervention";
run;




/*interrupted time series*/

proc sql;
    create table yearly_counts as
    select Rpt_Yr,
           count(*) as Num_Homicides
    from work.homicide_clean_v2
    group by Rpt_Yr
    having Rpt_Yr between 2010 and 2023
    order by Rpt_Yr;
quit;
data population;
    input Rpt_Yr Total_CA_Population;
    datalines;
2010 37319550
2011 37636311
2012 37944551
2013 38253768
2014 38586706
2015 38904296
2016 39149186
2017 39337785
2018 39437463
2019 39437610
2020 39501653
2021 39142991
2022 39029342
2023 39200000
;
run;

proc sql;
    create table its_data as
    select 
        a.Rpt_Yr, 
        a.Num_Homicides,
        b.Total_CA_Population,
        (a.Num_Homicides / b.Total_CA_Population) as Proportion
    from yearly_counts as a
    left join population as b
    on a.Rpt_Yr = b.Rpt_Yr;
quit;


data its_data;
    set its_data;

    /* Time: starts at 1 for 2010 */
    Time = Rpt_Yr - 2010 + 1;

    /* Intervention: binary indicator for post-2016 period */
    Intervention = (Rpt_Yr >= 2016);

    /* Time_After: time since intervention (0 if pre-2016) */
    Time_After = Time - (2016 - 2010 + 1);
    if Rpt_Yr < 2016 then Time_After = 0;
run;


proc reg data=its_data;
    model Proportion = Time Intervention Time_After / clb;
    title "Interrupted Time Series Regression: Effect of 2016 Intervention";
run;

proc reg data=its_data outest=estimates noprint;
    model Proportion = Time Intervention Time_After;
    output out=its_with_preds p=Predicted_Proportion;
run;

proc sgplot data=its_with_preds;
    scatter x=Rpt_Yr y=Proportion / markerattrs=(color=blue symbol=circlefilled);
    series x=Rpt_Yr y=Predicted_Proportion / lineattrs=(color=red thickness=2);
    refline 2016 / axis=x lineattrs=(color=black pattern=shortdash);
    yaxis label="Proportion (Homicides / CA Population)";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Observed vs Predicted Proportion with 2016 Intervention";
run;

/*assuming lag time by a year*/
data its_data_lagged;
    set its_data;

    /* Lagged intervention starts in 2017 */
    Intervention_Lag1 = (Rpt_Yr >= 2017);

    /* Time after lagged intervention */
    Time_After_Lag1 = Time - (2017 - 2010 + 1);
    if Rpt_Yr < 2017 then Time_After_Lag1 = 0;
run;
proc reg data=its_data_lagged outest=estimates_lag noprint;
    model Proportion = Time Intervention_Lag1 Time_After_Lag1;
    output out=its_lagged_with_preds predicted=Predicted_Proportion_Lag;
run;

data combined_preds;
    merge its_with_preds (in=a keep=Rpt_Yr Proportion Predicted_Proportion)
          its_lagged_with_preds (in=b keep=Rpt_Yr Predicted_Proportion_Lag);
    by Rpt_Yr;
run;

proc sgplot data=its_lagged_with_preds;
    scatter x=Rpt_Yr y=Proportion / markerattrs=(color=blue symbol=circlefilled);
    series x=Rpt_Yr y=Predicted_Proportion_Lag / lineattrs=(color=red thickness=2);
    refline 2017 / axis=x lineattrs=(color=black pattern=shortdash);
    yaxis label="Proportion (Homicides / CA Population)";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Observed vs Predicted Proportion (Lagged GVRO Start in 2017)";
run;


data its_data_lagged_scaled;
    set its_lagged_with_preds;
    Proportion_per100k = Proportion * 100000;
    Predicted_per100k = Predicted_Proportion_Lag * 100000;
run;
proc sgplot data=its_data_lagged_scaled;
    scatter x=Rpt_Yr y=Proportion_per100k / markerattrs=(color=blue symbol=circlefilled);
    series x=Rpt_Yr y=Predicted_per100k / lineattrs=(color=red thickness=2);
    refline 2017 / axis=x lineattrs=(color=black pattern=shortdash);
    yaxis label="Rate of Homicides per 100,000 People";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Observed vs Predicted Homicide Rate (Lagged GVRO Start in 2017)";
run;



/*one predicition line*/
proc sgplot data=its_data_lagged_scaled;
    scatter x=Rpt_Yr y=Proportion_per100k / markerattrs=(color=blue symbol=circlefilled);
    series x=Rpt_Yr y=Predicted_per100k / lineattrs=(color=red thickness=2);
    refline 2017 / axis=x lineattrs=(color=black pattern=shortdash);
    yaxis label="Homicide Rate per 100,000 People";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Observed vs Predicted Homicide Rate (Lagged GVRO Start in 2017)";
run;

/*two lines*/
data split_preds;
    set its_data_lagged_scaled;
    if Rpt_Yr < 2017 then Predicted_Pre = Predicted_per100k;
    else Predicted_Post = Predicted_per100k;
run;

proc sgplot data=split_preds;
    scatter x=Rpt_Yr y=Proportion_per100k / markerattrs=(color=black symbol=circlefilled);
    series x=Rpt_Yr y=Predicted_Pre / lineattrs=(color=green thickness=2 pattern=solid) legendlabel="Predicted (Pre-2017)";
    series x=Rpt_Yr y=Predicted_Post / lineattrs=(color=red thickness=2 pattern=solid) legendlabel="Predicted (Post-2017)";
    refline 2017 / axis=x lineattrs=(color=black pattern=shortdash);
    yaxis label="Homicide Rate per 100,000 People";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "ITS: Predicted Trends Before and After GVRO Implementation (Lagged)";
    keylegend / position=bottom;
run;


/*annual number of homicides over the years - make sure these are intimate partner homicides*/
proc sgplot data=its_data;
    vbar Rpt_Yr / response=Num_Homicides datalabel;
    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Number of Homicides";
    title "Annual Number of Homicides in California (2010–2023)";
run;




data pre_policy;
    set yearly_counts;
    if Rpt_Yr < 2016;
    Time = Rpt_Yr - 2010 + 1;
run;

proc reg data=pre_policy outest=cf_est noprint;
    model Num_Homicides = Time;
run;
data _null_;
    set cf_est;
    call symputx("cf_intercept", Intercept);
    call symputx("cf_slope", Time);
run;
data counterfactual;
    do Rpt_Yr = 2010 to 2023;
        Time = Rpt_Yr - 2010 + 1;
        Pred_Counterfactual = &cf_intercept + &cf_slope * Time;
        output;
    end;
run;
data combined_plot;
    merge yearly_counts counterfactual;
    by Rpt_Yr;
run;

proc sgplot data=combined_plot;
    series x=Rpt_Yr y=Num_Homicides / lineattrs=(color=blue thickness=2) legendlabel="Observed";
    series x=Rpt_Yr y=Pred_Counterfactual / lineattrs=(color=red thickness=2 pattern=shortdash) legendlabel="Counterfactual (No Policy)";
    refline 2016 / axis=x lineattrs=(pattern=shortdash color=black);
    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Number of Homicides";
    title "Observed vs Counterfactual Homicide Trend (Policy Introduced in 2016)";
    keylegend / position=bottom;
run;



data split_preds;
    set its_data_lagged_scaled;
    if Rpt_Yr < 2017 then Predicted_Pre = Predicted_per100k;
    else Predicted_Post = Predicted_per100k;
run;

proc sgplot data=split_preds;
    /* Observed values as bold dots */
    scatter x=Rpt_Yr y=Proportion_per100k / 
        markerattrs=(color=black symbol=circlefilled size=10)
        legendlabel="Observed Rate";

    /* Pre-policy trend */
    series x=Rpt_Yr y=Predicted_Pre / 
        lineattrs=(color=green thickness=2) 
        legendlabel="Predicted (Pre)";

    /* Post-policy trend */
    series x=Rpt_Yr y=Predicted_Post / 
        lineattrs=(color=red thickness=2) 
        legendlabel="Predicted (Post)";

    /* Policy intervention marker */
    refline 2017 / axis=x lineattrs=(pattern=shortdash color=black);

    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Homicide Rate per 100,000 People";
    title "ITS: Predicted Trends Before and After GVRO Implementation (Lagged)";
    keylegend / position=bottom;
run;
proc reg data=its_data_lagged_scaled(where=(Rpt_Yr < 2017)) outest=pre_est noprint;
    model Proportion_per100k = Time;
run;
proc reg data=its_data_lagged_scaled(where=(Rpt_Yr < 2017)) outest=pre_est noprint;
    model Proportion_per100k = Time;
run;

data _null_;
    set pre_est;
    call symputx("cf_intercept", Intercept);
    call symputx("cf_slope", Time);
run;
data counterfactual_line;
    do Rpt_Yr = 2010 to 2023;
        Time = Rpt_Yr - 2010 + 1;
        Pred_Counterfactual = &cf_intercept + &cf_slope * Time;
        output;
    end;
run;
data final_plot;
    merge split_preds counterfactual_line;
    by Rpt_Yr;
run;

proc sgplot data=final_plot;
    scatter x=Rpt_Yr y=Proportion_per100k / markerattrs=(color=black symbol=circlefilled size=10) legendlabel="Observed Rate";

    series x=Rpt_Yr y=Predicted_Pre / lineattrs=(color=green thickness=2) legendlabel="Predicted (Pre)";
    series x=Rpt_Yr y=Predicted_Post / lineattrs=(color=red thickness=2) legendlabel="Predicted (Post)";
    series x=Rpt_Yr y=Pred_Counterfactual / lineattrs=(color=gray pattern=shortdash thickness=2) legendlabel="Counterfactual (No Policy)";

    refline 2017 / axis=x lineattrs=(color=black pattern=shortdash);
    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Homicide Rate per 100,000 People";
    title "ITS with Counterfactual: Projected Trend Without GVRO Implementation";
    keylegend / position=bottom;
run;


/*final plot*/
proc sgplot data=final_plot;
    scatter x=Rpt_Yr y=Proportion_per100k / 
        markerattrs=(color=black symbol=circlefilled size=10) 
        legendlabel="Observed Rate";

    series x=Rpt_Yr y=Predicted_Pre / 
        lineattrs=(color=green thickness=2) 
        legendlabel="Predicted (Pre)";

    series x=Rpt_Yr y=Predicted_Post / 
        lineattrs=(color=red thickness=2) 
        legendlabel="Predicted (Post)";

    series x=Rpt_Yr y=Pred_Counterfactual / 
        lineattrs=(color=gray pattern=shortdash thickness=2) 
        legendlabel="Counterfactual (No Policy)";

    refline 2017 / axis=x lineattrs=(color=black pattern=shortdash);

    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Homicide Rate per 100,000 People";
    title "ITS with Counterfactual: Projected Trend Without GVRO Implementation";
    keylegend / position=bottom;
run;




data homicide_iph;
    set work.homicide_clean_v2;
    
    /* Define IPH codes */
    if VO_1 in (1, 2, 3, 4, 22, 23, 24, 25, 29) or
       VO_2 in (1, 2, 3, 4, 22, 23, 24, 25, 29) or
       VO_3 in (1, 2, 3, 4, 22, 23, 24, 25, 29) or
       VO_4 in (1, 2, 3, 4, 22, 23, 24, 25, 29);
run;

proc freq data=homicide_iph;
    tables VO_1 VO_2 VO_3 VO_4;
    title "Frequency of Relationship Codes in Filtered IPH Dataset";
run;

/*restricting IPH*/
/*-------------------------------------------
  Step 1: Restrict to Intimate Partner Homicides
-------------------------------------------*/
data homicide_iph;
    set work.homicide_clean_v2;

    if VO_1 in (1,2,3,4,22,23,24,25,29) or
       VO_2 in (1,2,3,4,22,23,24,25,29) or
       VO_3 in (1,2,3,4,22,23,24,25,29) or
       VO_4 in (1,2,3,4,22,23,24,25,29);
run;

/*-------------------------------------------
  Step 2: Count IPH Cases per Year
-------------------------------------------*/
proc sql;
    create table yearly_iph_counts as
    select Rpt_Yr,
           count(*) as Num_Homicides
    from homicide_iph
    where Rpt_Yr between 2010 and 2023
    group by Rpt_Yr
    order by Rpt_Yr;
quit;

/*-------------------------------------------
  Step 3: Add CA Population per Year
-------------------------------------------*/
data population;
    input Rpt_Yr Total_CA_Population;
    datalines;
2010 37319550
2011 37636311
2012 37944551
2013 38253768
2014 38586706
2015 38904296
2016 39149186
2017 39337785
2018 39437463
2019 39437610
2020 39501653
2021 39142991
2022 39029342
2023 39200000
;
run;

/*-------------------------------------------
  Step 4: Merge Counts with Population
-------------------------------------------*/
proc sql;
    create table its_data as
    select a.*, 
           b.Total_CA_Population,
           (a.Num_Homicides / b.Total_CA_Population) as Proportion
    from yearly_iph_counts as a
    left join population as b
    on a.Rpt_Yr = b.Rpt_Yr;
quit;

/*-------------------------------------------
  Step 5: Create ITS Variables
-------------------------------------------*/
data its_data;
    set its_data;
    Time = Rpt_Yr - 2010 + 1;
    Intervention = (Rpt_Yr >= 2016);
    Time_After = Time - (2016 - 2010 + 1);
    if Rpt_Yr < 2016 then Time_After = 0;

    Proportion_per100k = Proportion * 100000;
run;

/*-------------------------------------------
  Step 6: ITS Linear Regression Model
-------------------------------------------*/


proc reg data=its_data;
    model Proportion_per100k = Time Intervention Time_After / clb;
    title "ITS Model: IPH Rate per 100,000 with 2016 GVRO Intervention";
run;

/*-------------------------------------------
  Step 7: Plot Observed and Fitted Trends
-------------------------------------------*/
proc reg data=its_data outest=estimates noprint;
    model Proportion_per100k = Time Intervention Time_After;
    output out=its_preds p=Predicted_per100k;
run;

proc sgplot data=its_preds;
    scatter x=Rpt_Yr y=Proportion_per100k / markerattrs=(color=black symbol=circlefilled size=10) legendlabel="Observed Rate";
    series x=Rpt_Yr y=Predicted_per100k / lineattrs=(color=red thickness=2) legendlabel="Fitted Trend";
    refline 2016 / axis=x lineattrs=(color=black pattern=shortdash);
    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Intimate Partner Homicide Rate per 100,000 People";
    title "Observed and Predicted IPH Rate with 2016 GVRO Intervention";
    keylegend / position=bottom;
run;


proc sgplot data=yearly_iph_counts;
    series x=Rpt_Yr y=Num_Homicides / lineattrs=(color=blue thickness=2);
    scatter x=Rpt_Yr y=Num_Homicides / markerattrs=(color=black symbol=circlefilled size=10);
    refline 2016 / axis=x lineattrs=(pattern=shortdash color=red thickness=2);
    yaxis label="Number of IPH Homicides";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Annual Number of Intimate Partner Homicides (2010–2023)";
run;





data its_data_counts;
    set its_data;
    log_Pop = log(Total_CA_Population);
run;
proc genmod data=its_data_counts;
    model Num_Homicides = Time Intervention Time_After / dist=poisson link=log offset=log_Pop;
    output out=poisson_out resdev=dev;
run;

proc genmod data=its_data_counts;
    model Num_Homicides = Time Intervention Time_After / dist=poisson link=log offset=log_Pop;
    title "Poisson ITS Model for IPH Counts";
run;


/*visualizing using poisson model*/
proc genmod data=its_data_counts;
    model Num_Homicides = Time Intervention Time_After / dist=poisson link=log offset=log_Pop;
    output out=poisson_preds pred=Predicted_Count;
run;

proc sgplot data=poisson_preds;
    scatter x=Rpt_Yr y=Num_Homicides / markerattrs=(color=black symbol=circlefilled size=10) legendlabel="Observed Count";
    series x=Rpt_Yr y=Predicted_Count / lineattrs=(color=red thickness=2) legendlabel="Predicted Count (Poisson)";
    refline 2016 / axis=x lineattrs=(color=black pattern=shortdash);
    yaxis label="Number of Intimate Partner Homicides";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Poisson ITS: Observed vs Predicted IPH Counts (2010–2023)";
    keylegend / position=bottom;
run;
proc genmod data=its_data_counts;
    model Num_Homicides = Time Intervention Time_After / dist=poisson link=log offset=log_Pop;
    output out=poisson_preds pred=Predicted_Count;
run;

proc sgplot data=poisson_preds;
    scatter x=Rpt_Yr y=Num_Homicides / markerattrs=(color=black symbol=circlefilled size=10) legendlabel="Observed Count";
    series x=Rpt_Yr y=Predicted_Count / lineattrs=(color=red thickness=2) legendlabel="Predicted Count (Poisson)";
    refline 2016 / axis=x lineattrs=(color=black pattern=shortdash);
    yaxis label="Number of Intimate Partner Homicides";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Poisson ITS: Observed vs Predicted IPH Counts (2010–2023)";
    keylegend / position=bottom;
run;
proc genmod data=its_data_counts(where=(Rpt_Yr < 2016)) outest=cf_est noprint;
    model Num_Homicides = Time / dist=poisson link=log offset=log_Pop;
run;
data counterfactual_line;
    set its_data_counts;
    log_expected = &cf_intercept + &cf_slope * Time;
    /* Already modeled with offset in model, so just exponentiate */
    Pred_Counterfactual = exp(log_expected); 
run;

data poisson_full_plot;
    merge poisson_preds counterfactual_line(keep=Rpt_Yr Pred_Counterfactual);
    by Rpt_Yr;
run;
proc sgplot data=poisson_full_plot;
    scatter x=Rpt_Yr y=Num_Homicides / 
        markerattrs=(color=black symbol=circlefilled size=10) 
        legendlabel="Observed Count";

    series x=Rpt_Yr y=Predicted_Count / 
        lineattrs=(color=red thickness=2) 
        legendlabel="Predicted (Poisson Model)";

    series x=Rpt_Yr y=Pred_Counterfactual / 
        lineattrs=(color=gray pattern=shortdash thickness=2) 
        legendlabel="Counterfactual (No Policy)";

    refline 2016 / axis=x lineattrs=(color=black pattern=shortdash thickness=2);
    yaxis label="Number of Intimate Partner Homicides";
    xaxis label="Year" values=(2010 to 2023 by 1);
    title "Poisson ITS with Counterfactual: IPH Counts and Predicted Trends";
    keylegend / position=bottom;
run;





/*-------------------------------
 Step 1: Restrict to IPH cases
-------------------------------*/
data homicide_iph;
    set work.homicide_clean_v2;
    if VO_1 in (1,2,3,4,22,23,24,25,29) or
       VO_2 in (1,2,3,4,22,23,24,25,29) or
       VO_3 in (1,2,3,4,22,23,24,25,29) or
       VO_4 in (1,2,3,4,22,23,24,25,29);
run;

/*-------------------------------
 Step 2: Count IPH per year
-------------------------------*/
proc sql;
    create table yearly_iph_counts as
    select Rpt_Yr,
           count(*) as Num_Homicides
    from homicide_iph
    where Rpt_Yr between 2010 and 2023
    group by Rpt_Yr
    order by Rpt_Yr;
quit;

/*-------------------------------
 Step 3: Add CA population
-------------------------------*/
data population;
    input Rpt_Yr Total_CA_Population;
    datalines;
2010 37319550
2011 37636311
2012 37944551
2013 38253768
2014 38586706
2015 38904296
2016 39149186
2017 39337785
2018 39437463
2019 39437610
2020 39501653
2021 39142991
2022 39029342
2023 39200000
;
run;

/*-------------------------------
 Step 4: Merge & create ITS vars
-------------------------------*/

proc sql;
    create table its_data as
    select a.*, 
           b.Total_CA_Population
    from yearly_iph_counts as a
    left join population as b
    on a.Rpt_Yr = b.Rpt_Yr;
quit;

data its_data_counts;
    set its_data;
    Time = Rpt_Yr - 2010 + 1;
    Intervention = (Rpt_Yr >= 2016);
    Time_After = Time - (2016 - 2010 + 1);
    if Rpt_Yr < 2016 then Time_After = 0;
    log_Pop = log(Total_CA_Population);
run;

/*-------------------------------
 Step 5: Poisson ITS model
-------------------------------*/
proc genmod data=its_data_counts;
    model Num_Homicides = Time Intervention Time_After / dist=poisson link=log offset=log_Pop;
    output out=poisson_preds pred=Predicted_Count;
    title "Poisson ITS Model for IPH Counts";
run;

/*-------------------------------
 Step 6: Counterfactual (Pre-Trend Only)
-------------------------------*/
proc genmod data=its_data_counts(where=(Rpt_Yr < 2016)) outest=cf_est noprint;
    model Num_Homicides = Time / dist=poisson link=log offset=log_Pop;
run;

data _null_;
    set cf_est;
    call symputx("cf_intercept", Intercept);
    call symputx("cf_slope", Time);
run;

data counterfactual_line;
    set its_data_counts;
    /* Predict log rate */
    log_rate = &cf_intercept + &cf_slope * Time;
    /* Scale to counts using actual population */
    Pred_Counterfactual = exp(log_rate) * Total_CA_Population;
run;

/*-------------------------------
 Step 7: Merge and Plot All
-------------------------------*/
proc sort data=poisson_preds; by Rpt_Yr; run;
proc sort data=counterfactual_line; by Rpt_Yr; run;
data poisson_full_plot;
    merge poisson_preds(in=a)
          counterfactual_line(in=b keep=Rpt_Yr Pred_Counterfactual);
    by Rpt_Yr;
    if a;  /* Keep only years with observed/predicted data */
run;

proc sgplot data=poisson_full_plot;
    scatter x=Rpt_Yr y=Num_Homicides / 
        markerattrs=(color=black symbol=circlefilled size=10) 
        legendlabel="Observed Count";

    series x=Rpt_Yr y=Predicted_Count / 
        lineattrs=(color=red thickness=2) 
        legendlabel="Predicted (Poisson Model)";

    series x=Rpt_Yr y=Pred_Counterfactual / 
        lineattrs=(color=gray pattern=shortdash thickness=2) 
        legendlabel="Counterfactual (No Policy)";

    refline 2016 / axis=x lineattrs=(color=black pattern=shortdash);
    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Number of Intimate Partner Homicides";
    title "Poisson ITS with Counterfactual: IPH Counts and Predicted Trends";
    keylegend / position=bottom;
run;







/* Step 0: Make sure you have this already */
data its_data_counts;
    set its_data;
    Time = Rpt_Yr - 2010 + 1;
    Intervention = (Rpt_Yr >= 2016);
    Time_After = Time - (2016 - 2010 + 1);
    if Rpt_Yr < 2016 then Time_After = 0;
    log_Pop = log(Total_CA_Population);
run;


/* Step 1: Fit Poisson model to pre-policy data and capture coefficients */
ods output ParameterEstimates=cf_estimates;

proc genmod data=its_data_counts(where=(Rpt_Yr < 2016));
    model Num_Homicides = Time / dist=poisson link=log offset=log_Pop;
run;

ods output close;

/* Step 2: Store the intercept and slope as macro variables */
proc sql noprint;
    select Estimate into :cf_intercept from cf_estimates where Parameter="Intercept";
    select Estimate into :cf_slope     from cf_estimates where Parameter="Time";
quit;

/* Step 3: Create counterfactual predictions assuming no policy */
data counterfactual_line;
    set its_data_counts;
    log_rate = &cf_intercept + &cf_slope * Time;
    Pred_Counterfactual = exp(log_rate) * Total_CA_Population;
run;

/* Step 4: Run full Poisson ITS model on all years and get predicted counts */
proc genmod data=its_data_counts;
    model Num_Homicides = Time Intervention Time_After / dist=poisson link=log offset=log_Pop;
    output out=poisson_preds pred=Predicted_Count;
run;

/* Step 5: Merge observed, predicted, and counterfactual data */
data poisson_full_plot;
    merge poisson_preds(in=a)
          counterfactual_line(keep=Rpt_Yr Pred_Counterfactual);
    by Rpt_Yr;
    if a;

    /* Convert to rates per 100,000 people */
    Observed_Rate = (Num_Homicides / Total_CA_Population) * 100000;
    Predicted_Rate = (Predicted_Count / Total_CA_Population) * 100000;
    Counterfactual_Rate = (Pred_Counterfactual / Total_CA_Population) * 100000;
run;


/* Step 6: Plot observed, predicted, and counterfactual trends */
proc sgplot data=poisson_full_plot;
    scatter x=Rpt_Yr y=Observed_Rate / 
        markerattrs=(color=black symbol=circlefilled size=10) 
        legendlabel="Observed Rate";

    series x=Rpt_Yr y=Predicted_Rate / 
        lineattrs=(color=red thickness=2) 
        legendlabel="Predicted (Poisson Model)";

    series x=Rpt_Yr y=Counterfactual_Rate / 
        lineattrs=(color=gray pattern=shortdash thickness=2) 
        legendlabel="Counterfactual (No Policy)";

    refline 2016 / axis=x lineattrs=(color=black pattern=shortdash);
    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="IPH Rate per 100,000 People";
    title "Poisson ITS with Counterfactual: IPH Rate per 100,000";
    keylegend / position=bottom;
run;


/*annual distribution of intimate partner homicides*/


proc sgplot data=yearly_iph_counts;
    vbar Rpt_Yr / response=Num_Homicides datalabel;
    xaxis label="Year" values=(2010 to 2023 by 1);
    yaxis label="Number of Intimate Partner Homicides";
    title "Annual Distribution of Intimate Partner Homicides in California (2010–2023)";
run;
proc sql;
    select sum(Num_Homicides) as Total_IPH_2010_2023
    from yearly_iph_counts;
quit;


proc sql;
    create table reg_ready as
    select a.*, b.Time, b.Intervention, b.Time_After
    from poisson_full_plot as a
    left join its_data_counts as b
    on a.Rpt_Yr = b.Rpt_Yr;
quit;
proc reg data=reg_ready;
    model Observed_Rate = Time Intervention Time_After / clb;
    title "Linear ITS Model: IPH Rate per 100,000 with 2016 GVRO Intervention";
run;



/***FIGURE OUT RACE AND ETHNICITY*/

/*race and ethnicity*/
proc contents data=work.homicide_clean_v2;
run;

data homicide_iph_victims;
    set work.homicide_clean_v2;

    /* Create IPH flag */
    if VO_1 in (1,2,3,4,22,23,24,25,29) or
       VO_2 in (1,2,3,4,22,23,24,25,29) or
       VO_3 in (1,2,3,4,22,23,24,25,29) or
       VO_4 in (1,2,3,4,22,23,24,25,29) then IPH_case = 1;
    else IPH_case = 0;
run;

data homicide_iph_race_letters;
    set homicide_iph_victims;

    /* Keep only IPH cases */
    if IPH_case = 1;

    /* Keep only letter-based V race codes */
    if "V race"n in (
        'W', 'H', 'B', 'I', 'C', 'J', 'F', 'O', 'P',
        'A', 'D', 'G', 'K', 'L', 'S', 'U', 'V', 'Z', 'X'
    );
run;




data homicide_iph_victims;
    set work.homicide_clean_v2;
    /* Restrict to IPH */
    if VO_1 in (1,2,3,4,22,23,24,25,29) or
       VO_2 in (1,2,3,4,22,23,24,25,29) or
       VO_3 in (1,2,3,4,22,23,24,25,29) or
       VO_4 in (1,2,3,4,22,23,24,25,29);
run;


proc format;
    value $racefmt
        '0' = 'Unknown'
        'X' = 'Unknown'
        '1' = 'White'
        'W' = 'White'
        '2' = 'Hispanic'
        'H' = 'Hispanic'
        '3' = 'Black'
        'B' = 'Black'
        '4' = 'American Indian'
        'I' = 'American Indian'
        '5' = 'Chinese'
        'C' = 'Chinese'
        '6' = 'Japanese'
        'J' = 'Japanese'
        '7' = 'Filipino'
        'F' = 'Filipino'
        '8' = 'Other'
        'O' = 'Other'
        '9' = 'Pacific Islander'
        'P' = 'Pacific Islander'
        'A' = 'Other Asian'
        'D' = 'Cambodian'
        'G' = 'Guamanian'
        'K' = 'Korean'
        'L' = 'Laotian'
        'S' = 'Samoan'
        'U' = 'Hawaiian'
        'V' = 'Vietnamese'
        'Z' = 'Asian Indian';
run;

proc contents data= homicide_iph_victims;
run;

proc freq data=homicide_iph_victims;
    tables / nocum nopercent;
    title "Victim Race and Ethnicity in IPH Cases (2010–2023)";
run;

proc sgplot data=homicide_iph_victims;
    vbar Victim_Race / stat=freq datalabel;
    yaxis label="Number of IPH Victims";
    xaxis label="Victim Race";
    title "Distribution of Victim Race in IPH Cases (2010–2023)";
run;



proc sql;
    select count(*) as Total_IPH_2010_2023
    from homicide_iph
    where Rpt_Yr between 2010 and 2023;
quit;




/*




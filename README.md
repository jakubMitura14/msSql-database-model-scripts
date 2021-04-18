# msSql-database-model-scripts
Project created for advanced database modelling - subjectgiven in WWSI Warsaw on Big Data specialization in computer science masters 
Database model created for  hypothethical 3d printer company that receives orders to print sets of 
Company has multiple branches each can have multiple printers, each branch is working 3 8h shifts 5 days a week 
Each printer may break  and we are required to keep track on when and where the printer broke, also printer can have multiple failures in the same period of time and they can be of diffrent duration and can overlap. Status 0 means it works other statuses are pointing out diffrent states of repair

Database need to support set of prespecified queries  that will be listed below
#Database Model

![image](https://user-images.githubusercontent.com/53857487/115156935-206d5f80-a087-11eb-8aa3-4a3d2055471f.png)
#Helper Functions
## getShiftOfDate
Here given the date it will return on which shift the printer had broken
```
create  PROCEDURE getShiftOfDate
-- Add the parameters for the stored procedure here
@dateFrom DateTime,
@shift_number int OUTPUT
AS
begin
-- we will specify here beginning and the end of shifts HOURS !!  one and two in case we are not in shift one or two we are in shift 3
declare @shiftOneStart int;
declare @shiftOneEnd int;

set @shiftOneStart = (select startHour from  Shift where shiftNumber =1 ); 
set @shiftOneEnd= @shiftOneStart+ (select shiftLength from  Shift where shiftNumber =1 )/60; 

declare @shiftTwoStart int;
declare @shiftTwoEnd int;

set @shiftTwoStart = (select startHour from  Shift where shiftNumber =2 ); 
set @shiftTwoEnd= @shiftTwoStart+ (select shiftLength from  Shift where shiftNumber =2 )/60; 
--we need to establish with which shift we start 

set @shift_number = (case when DATEPART(hh,@dateFrom)  between @shiftOneStart and    @shiftOneEnd then  1
			 when DATEPART(hh,@dateFrom)  between @shiftTwoStart and   @shiftTwoEnd then  2
			else 3
			END		)
end 

```

## addShiftsData
Using recursion creates table with all shifts that were in given time interval starting from the shit that started just before printer failure 
```

alter  PROCEDURE addShiftsData
-- Add the parameters for the stored procedure here
@dateFrom DateTime,
@dateTo DateTime

AS
BEGIN
-- first using other function we get the shift number that we start from
declare @shift_number int;
EXECUTE [dbo].[getShiftOfDate] @dateFrom , @shift_number = @shift_number OUTPUT;  
-- now we need to specify when the shift that during which @dateFrom had been had started (beginning of this particular shift)
-- so we delete all other data about time and set only what we care taken from https://stackoverflow.com/questions/2847851/set-time-part-of-datetime-variable-to-1800
declare @modifiedDateTime DateTime;

declare @first_shift_start DateTime;
SET @first_shift_start = @dateFrom;
SET @first_shift_start = DateAdd(mi,- (DatePart(mi,@first_shift_start)), @first_shift_start) ;
SET @first_shift_start = DateAdd(ss,- (DatePart(ss,@first_shift_start)), @first_shift_start) ;
SET @first_shift_start = DateAdd(ms,- (DatePart(ms,@first_shift_start)), @first_shift_start) ;

declare @hours int;
set @hours = (case when @shift_number= 1   then ( select top 1 startHour from Shift where shiftNumber = 1)
					when @shift_number= 2  then  ( select top 1 startHour from Shift where shiftNumber = 2)
					else ( select top 1 startHour from Shift where @shift_number = 3)
			END		)

SET @modifiedDateTime = DateAdd(hh,- (DatePart(hh,@first_shift_start))+@hours, @first_shift_start) ;

--first we will add 480 minute to the date until it will reach final date starting from the bagining of the shift we are currently in
with shifts as
(select @modifiedDateTime as fullDate,@shift_number as  shiftNumber
union all
	select DATEADD(hh,8,fullDate) as fullDate, ( case when shiftNumber = 3 then 1 else  shiftNumber+1 end  )  as shiftNumber
	from shifts
	where fullDate <@dateTo)
--now we filter all non working days - sundays and saturdays
,onlyWorkingDays as (select * from shifts  where DATEPART(dw,fullDate) not in (1,7) )
-- now we need to associate the date times with particular shifts 
select * from onlyWorkingDays option (maxrecursion 32767)


END





```













#query 1 On which shift the printer had failure

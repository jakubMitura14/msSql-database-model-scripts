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

## dataFailureOverlapB
 as stated before  the  failures periods may overlap we need to be able to fuse the all periods that overlapped hence this table creating utility table  that  looks to given interval of time and return all failures that had happened in this period and associated dates date min and date max is representing time or specified failure and  newestPre , newestEnd represent the beginning and eng of accumulated , fused overlapping intervals
```

alter  PROCEDURE dataFailureOverlapB
	-- Add the parameters for the stored procedure here
	@dateFrom DateTime,
	@dateTo DateTime

AS
BEGIN

	SET NOCOUNT ON;


--with prim  as  ( select idBreaking ,min(dateTimeOfStatusChange) as dateMin, max(dateTimeOfStatusChange) as dateMax from PrinterStatusLog 
--where  idBreaking in (select distinct p1.idBreaking from  PrinterStatusLog as p1 where dateTimeOfStatusChange between @dateFrom AND @dateTo  ) group by idBreaking)
--,dobl as(select *, DATEDIFF(D, dateMin,dateMax) as failureIntervalTime from prim )
----so now we have 
--select *  from dobl



with prim  as  ( select idBreaking, min(dateTimeOfStatusChange) as dateMin, max(dateTimeOfStatusChange) as dateMax from PrinterStatusLog 
where  idBreaking in (select distinct p1.idBreaking from  PrinterStatusLog as p1 where dateTimeOfStatusChange between @dateFrom AND @dateTo  ) group by idBreaking)
,diffr as(select *, DATEDIFF(D, dateMin,dateMax) as failureIntervalTime from prim)
--so now we have to establish are those periods overlapping if so set corrected begin an end dates
-- first we will dublicate  the date min and date max column so later it will make us finding the overl
,dobl as (select  dateMin as dateMinB ,dateMax as dateMaxB from diffr)
--below we are looking for new begining and end dates of each intervals weather they are encompassed in any other interval if they are we will return the beginning or end date of this encompassing interval
--there may be also the case that the new begining and end dates are still in some intervals hence we need to establish in which intervals we have overlapping so
--each case we have a begining date and look is it in some other interval - some other idBreaking  - this may return multiple id breaking  we do it for all of the id breaking with both begining and end  time
-- so we will repeat procedure
,tri as (select *, newPre=(select top 1 dateMinB from dobl where dateMin between dateMinB And dateMaxB Order by dateMinB asc), newEnd =  (select top 1 dateMaxB from dobl where dateMax between dateMinB And dateMaxB Order by dateMaxB desc)   from diffr)
,triB as (select newPre as newPreB, newEnd as newEndB from tri)
-- now we need to do it one more time in case we have deeply nested structure
,tetra as (select idBreaking,dateMin,dateMax,  newestPre=(select top 1 newPreB from triB where newPre between newPreB And newEndB Order by newPreB asc), newestEnd =  (select top 1 newEndB from triB where newEnd between newPreB And newEndB Order by newEndB desc)   from tri)
-- now we need to iteratively add data about  work shifts using data from 
select * from tetra

END



```



# query 1 On which shift the printer had failure given idbreaking and period of looking



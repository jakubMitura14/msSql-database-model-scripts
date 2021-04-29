# msSql-database-model-scripts
Project created for advanced database modelling - subjectgiven in WWSI Warsaw on Big Data specialization in computer science masters 
Database model created for  hypothethical 3d printer company that receives orders to print sets of 
Company has multiple branches each can have multiple printers, each branch is working 3 8h shifts 5 days a week 
Each printer may break  and we are required to keep track on when and where the printer broke, also printer can have multiple failures in the same period of time and they can be of diffrent duration and can overlap. Status 0 means it works other statuses are pointing out diffrent states of repair

Database need to support set of prespecified queries  that will be listed below
#Database Model

![image](https://user-images.githubusercontent.com/53857487/115156935-206d5f80-a087-11eb-8aa3-4a3d2055471f.png)
#Helper Functions
# Helper Functions

```

alter FUNCTION addShiftsDataFunct
(
@dateFrom DateTime, @dateTo DateTime) 
RETURNS @ResultTable TABLE
( 
shiftNumber INT, fullDate DateTime
) AS BEGIN

-- first using other function we get the shift number that we start from
declare @shift_number int;
SET @shift_number = (select [dbo].[getShiftOfDateFunct] (@dateFrom)) ;
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

       INSERT INTO @ResultTable
          SELECT shiftNumber,fullDate  from onlyWorkingDays option (maxrecursion 32767)
        
RETURN
END
```

```
create function getCountOfShifts (
@dateFrom DateTime, @dateTo DateTime) 
returns  int
AS
begin
--this points how many shofts we had between two specified time points
declare @shift_number int;
set @shift_number= (select COUNT(fullDate) from [dbo].[addShiftsDataFunct]( @dateFrom,@dateTo) )


return @shift_number
end 


```


```
alter FUNCTION dataFailureOverlapC
(
@dateFrom DateTime, @dateTo DateTime) 
RETURNS @ResultTable TABLE
( 
idBreaking INT, idEveryPrinter INT, dateMin DateTime,  dateMax DateTime, newestPre DateTime ,newestEnd DateTime
) AS BEGIN



with prim  as  ( select idBreaking,idEveryPrinter, min(dateTimeOfStatusChange) as dateMin, max(dateTimeOfStatusChange) as dateMax from PrinterStatusLog 
where  idBreaking in (select distinct p1.idBreaking from  PrinterStatusLog as p1 where dateTimeOfStatusChange between @dateFrom AND @dateTo  ) group by idBreaking, idEveryPrinter)
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
,tetra as (select idBreaking,idEveryPrinter, dateMin,dateMax,  newestPre=(select top 1 newPreB from triB where newPre between newPreB And newEndB Order by newPreB asc), newestEnd =  (select top 1 newEndB from triB where newEnd between newPreB And newEndB Order by newEndB desc)   from tri)



       INSERT INTO @ResultTable
          SELECT idBreaking,idEveryPrinter, dateMin,dateMax,newestPre,newestEnd   from tetra
        
RETURN
END



```


```
CREATE FUNCTION dataFailureOverlapGivenPrinter
(
@dateFrom DateTime, @dateTo DateTime, @idPrinter int) 
RETURNS @ResultTable TABLE
( 
idBreaking INT, dateMin DateTime,  dateMax DateTime, newestPre DateTime ,newestEnd DateTime
) AS BEGIN



with prim  as  ( select idBreaking, min(dateTimeOfStatusChange) as dateMin, max(dateTimeOfStatusChange) as dateMax from PrinterStatusLog 
where  idBreaking in (select distinct p1.idBreaking from  PrinterStatusLog as p1 where dateTimeOfStatusChange between @dateFrom AND @dateTo  ) and idEveryPrinter = @idPrinter group by idBreaking)
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



       INSERT INTO @ResultTable
          SELECT idBreaking,dateMin,dateMax,newestPre,newestEnd   from tetra
        
RETURN
END

```


```

alter function getShiftOfDateFunct (@dateFrom DateTime) returns  int
AS
begin
-- we will specify here beginning and the end of shifts HOURS !!  one and two in case we are not in shift one or two we are in shift 3
declare @shiftOneStart int;
declare @shiftOneEnd int;
declare @shift_number int;

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

return @shift_number
end 



```

# Queries examples
## query 1 On which shift the printer had failure given idbreaking and period of looking
--Wskaż na jakiej zmianie dane urządzenie uległo awarii 

```
select [dbo].[getShiftOfDateFunct](datemin) from  dataFailureOverlapC('2020-04-01 00:00:00','2020-12-02 23:59:59') where idBreaking = 1

```

![image](https://user-images.githubusercontent.com/53857487/115269941-1bb6b300-a13c-11eb-9fad-fb4c1f464d29.png)

## query 2 On which Shift  printer was repaired
--Wskaż na jakiej zmianie maszyna została naprawiona 

```
select [dbo].[getShiftOfDateFunct](datemax) from  dataFailureOverlapC('2020-04-01 00:00:00','2020-12-02 23:59:59') where idBreaking = 1

```
![image](https://user-images.githubusercontent.com/53857487/115270397-8d8efc80-a13c-11eb-8125-4a2dd5c8a9de.png)


## query 3 Sum the time the printer was broken in given period of time (includin weekends)
--Wskaż jaki był czas postoju danego urządzenia w ciągu zadanego okresu czasu. 

```
with prim as (select distinct newestPre, newestEnd  from  dataFailureOverlapC('2020-04-01 00:00:00','2020-12-02 23:59:59'))
select sum(DATEDIFF(dd,newestPre, newestEnd)) from prim 

```
![image](https://user-images.githubusercontent.com/53857487/115272025-3b4edb00-a13e-11eb-893d-2f29bed6a05a.png)

## query 4  time in which printer was non functioning  excluding weekends of some particular printer 
--Wskaż jaki był czas postoju danego urządzenia w ciągu zadanego okresu czasu nie wliczając w to czasu kiedy zakład produkcyjny nie pracował (weekendy) 
so i will just sum duration of all  shifts related to failure
![image](https://user-images.githubusercontent.com/53857487/115406826-d18f0980-a1ef-11eb-8a4e-0900a5490cec.png)



## query 5 time in which printers were non functioning  excluding weekends
--Jaki był sumaryczny czas postoju wszystkich urządzeń w ciągu zadanego okresu czasu nie wliczając w to czasu kiedy zakład produkcyjny nie pracował (weekendy) 

so i will just sum duration of all  shifts related to failure

![image](https://user-images.githubusercontent.com/53857487/115405013-23cf2b00-a1ee-11eb-8b34-39266a27da39.png)


## query 6  point out to company branch where printers broke most frequently 
--Wskaż oddział w którym w 2020 roku urządzenia psuły się najczęściej 

![image](https://user-images.githubusercontent.com/53857487/116288828-315a5700-a792-11eb-82d9-791fd570227a.png)

## query 7 in which company branch the time where printers were broken was longest
--Wskaż oddział w którym w 2020 roku był najdłuższy  czas postoju urządzeń. 
with prim as (select distinct newestPre, newestEnd, idEveryPrinter from  dataFailureOverlapC('2019-04-01 00:00:00','2023-12-02 23:59:59')  )
select companyBranch , sum([dbo].[getCountOfShifts](newestPre, newestEnd)*8/24)  as timeOfBroken from  prim  
		join [dbo].[EveryPrinter] on prim.idEveryPrinter = EveryPrinter.idEveryPrinter  group by companyBranch  order by timeOfBroken desc 

![image](https://user-images.githubusercontent.com/53857487/116289059-6e264e00-a792-11eb-8ac4-d75329d99969.png)


## query 8   How many devices are in any particular state 
--Ile % urządzeń jest w poszczególnej fazach awarii w stosunku do ilości wszystkich dostępnych  

-- first we are getting all of the current statuses 
with prim as ( select [printerStatus],[dateTimeOfStatusChange], [idEveryPrinter] as idP, [idBreaking] from [dbo].[PrinterStatusLog])
,freshStatus as (select  idP  , printerStatus,dateTimeOfStatusChange as freshDate 
from prim
where    dateTimeOfStatusChange = (select top 1 dateTimeOfStatusChange from [dbo].[PrinterStatusLog] 
										where  idP = idEveryPrinter
										order by 	dateTimeOfStatusChange desc 				  ) )
-- now we prepare the number of the printers available
,connected  AS (select * from freshStatus FULL join  EveryPrinter ON EveryPrinter.idEveryPrinter = freshStatus.idP)
,withPrinterNumberPerBranch as (select *, count(idP) over(partition by companyBranch) as numberOfPrinters from connected)
,withPrinterNumberPerStatus as (select *, count(idP) over(partition by companyBranch, printerStatus) as numberInEachStaus from withPrinterNumberPerBranch)
select *, (CAST(   numberInEachStaus as FLOAT) / numberOfPrinters) as percent_in_status from withPrinterNumberPerStatus


![image](https://user-images.githubusercontent.com/53857487/116289991-484d7900-a793-11eb-86b7-bcde56bccd9c.png)


## query 9 calculate on how many shifts the printer was not working
--Wskaż na ilu zmianach nie pracowała maszyna (wliczając to zmianę na której zgłoszono awarię i na której uruchomiono ja znów produkcyjnie ) 

with prim as (select distinct newestPre, newestEnd  from  dataFailureOverlapGivenPrinter('2020-04-01 00:00:00','2020-12-02 23:59:59',1))
select  sum([dbo].[getCountOfShifts](newestPre, newestEnd)) from prim

![image](https://user-images.githubusercontent.com/53857487/116290177-7b900800-a793-11eb-9c0f-53eeee69bba0.png)


## query 10  How many orders each comapny branch have
--Ile zamówień ma dany oddział do realizacji. 
--first filtering only those that we have not yet completed printing
with prim as (select * from [dbo].[OrderHistory] where [dateTimeOfCompletion] is null )
select distinct companyBranch, count(idOrderHistory) over (partition by companyBranch) as numberOfNotCompleted from prim

![image](https://user-images.githubusercontent.com/53857487/116290477-c7db4800-a793-11eb-9d0a-325cb38a1f7e.png)


## query 11 What will be joint time of printing all required elements in printer
--Jaki będzie łączny czas drukowania zleconych oddziałowi elementów. 


-- selecting only non completed orders for printer 1 joining data about element sets that are not yet completed
with prim as (select [dateTimeOfIssuing],[quantity],idElement as idEl  from [dbo].[OrderHistory] join SetDetail on [dbo].[OrderHistory].ChosenSet = SetDetail.idSet
where [dateTimeOfCompletion] is null  And idPrinter =1)
-- we add data about how many minutes element needs in order to be printed
, dobl as ( select * from prim join [dbo].[Element] on Element.IdElement = prim.idEl)
--choosing only needed columns to clean up
,tri as (select [dateTimeOfIssuing],[quantity] ,[timeOfPrintMInutes] from dobl)
-- multiplying time of print of an element with amount of this element
, tetra as (select [dateTimeOfIssuing], [quantity]*[timeOfPrintMInutes] as totalTime from tri )
-- now we choose earlier date related to not completed order and sum all of the time required to complete what we have to do 
select  SUM(totalTime) from tetra

![image](https://user-images.githubusercontent.com/53857487/116455440-5964bb80-a861-11eb-9df7-9adc1c00dddb.png)

result is in minutes

## query 12 
Is it possible to admit new order  to be completed in 36 hours
--Czy jest możliwe przyjęcie zgłoszenia zamówienia w danym oddziale aby było zrealizowane w ciągu 36 h roboczych. 


-- selecting only non completed orders for company brach 1 joining data about element sets that are not yet completed
with prim as (select [dateTimeOfIssuing],[quantity],idElement as idEl  from [dbo].[OrderHistory] join SetDetail on [dbo].[OrderHistory].ChosenSet = SetDetail.idSet
where [dateTimeOfCompletion] is null  And [companyBranch] =1)
-- we add data about how many minutes element needs in order to be printed
, dobl as ( select * from prim join [dbo].[Element] on Element.IdElement = prim.idEl)
--choosing only needed columns to clean up
,tri as (select [dateTimeOfIssuing],[quantity] ,[timeOfPrintMInutes] from dobl)
-- multiplying time of print of an element with amount of this element
, tetra as (select [dateTimeOfIssuing], [quantity]*[timeOfPrintMInutes] as totalTime from tri )
-- now we choose earlier date related to not completed order and sum all of the time required to complete what we have to do 
,fifth as (select  SUM(totalTime) as timeRes, MIN([dateTimeOfIssuing])  as minDate from tetra)
select DATEADD(mi,timeRes ,minDate) from fifth

![image](https://user-images.githubusercontent.com/53857487/116456836-00962280-a863-11eb-9ac3-597bcf3b85cc.png)

So we returned the time when the printer will become available


## query 13 Does the failure of given printer will lead to a risk 
--Czy awaria danego urządzenia zagraża czasom poprawnej realizacji zleceń już zgłoszonych w danym oddziale. (Trigger)







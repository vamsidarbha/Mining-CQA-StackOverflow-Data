                      /****************Gray Areas*************************/

--Creating UnansweredPosts Table which contains Id and Tags of the posts that are unanswered in third quarter 2014 dataset

select ID,Tags 
into Unansweredposts 
from
(select P.Id,P.Tags,A.ID as i 
 from
  (select Id,Tags 
   from Posts_ThirdQuarter 
   where PostTypeId=1
  )P
  left join
  (select Id 
   from Posts_ThirdQuarter_FAD
  )A
  on P.ID=A.ID
)X
where X.i is NULL

--Intermediate table for finding TagFrequencyUnanswered posts

select TagName,Tags
into dbo.TagFrequencyUnansweredTemp
from dbo.tags 
inner join 
(select Tags 
 from Unansweredposts
)X
on Tags like '%<'+TagName+'>%' collate SQL_Latin1_General_CP1_CS_AS

--Creating TagFrequencyUnanswered table which contains TagName and number of unanswered questions for that particular tag
 
select TagName,count(1) as frequency 
into TagFrequencyUnanswered 
from dbo.TagFrequencyUnansweredTemp 
group by TagName 

-- Intermediate table for creating Gray area table
 
select A.TagName,A.TagFrequency-U.frequency as AnsweredTagFrequency,U.frequency as UnansweredTagFrequency,
Convert(float,U.Frequency)/Convert(float,A.TagFrequency) as PerUnansweredQuestions
into QuestionAnsweredUnansweredStats
from TagFrequency A
inner join TagFrequencyUnanswered U
on A.TagName=U.TagName


--Creating GrayArea table which can be queried to find grey areas
--Column explanation 
--TagName=Name of the Tag or Area
--AnsweredTagFrequency which is number of answered questions for a particular Tag
--UnansweredTagFrequency which is number of unanswered questions for a particular Tag
--PerUnansweredQuestions is the ratio of unanswered questions to total questions for a particular tag

select T.TagName,AnsweredTagFrequency,UnansweredTagFrequency,PerUnansweredQuestions,subscribersPerTag 
into GrayArea
from QuestionAnsweredUnansweredStats T 
inner join
dbo.TagFeatures F
on T.TagName=F.TagName
 
-- Query to Find first perceptive gray areas,i.e. areas where subscribers are 0 
select TagName from GrayArea where subscribersPerTag=0 

--Query to find second perceptive of gray areas, Areas where number of subscibers are greater than a particular threshold(100 here)
-- and where number of unanswered posts are more than answered ones
select * from GrayArea where subscribersPerTag>100 and PerUnansweredQuestions>0.5


 


 
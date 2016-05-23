DBCC FREESYSTEMCACHE('ALL')
DBCC FREESESSIONCACHE
DBCC FREEPROCCACHE
--------------------------------------------------------------------------------------------
--Taking Subset of Posts Data for our analysis,ThirdQuarter 0f year 2014 in precise...

select * 
into dbo.Posts_ThirdQuarter 
from dbo.Posts 
where year(creationdate)=2014 and month(creationdate)>6 and month(creationdate)<10
                        

						/************ Tag Features*************/
---------------------------------------------------------------------------------------------
----TagFrequencyThirdQuarterTemp-Intermediate table for computing TagFrequency  
                                   
select TagName,Tags
into dbo.TagFrequencyThirdQuarterTemp
from dbo.tags inner join 
(select Tags from dbo.Posts_ThirdQuarter where posttypeId=1)X
on Tags like '%<'+TagName+'>%' collate SQL_Latin1_General_CP1_CS_AS

----------------------------------------------------------------------------------------------

                             /********* Tag Frequency***********/
/* Computing TagFrequency Table which contains records of each tag with the number of times 
it gets repeated in all posts of third quarter 2014*/
                            
select TagName,count(1) as TagFrequency 
into TagFrequency 
from TagFrequencyThirdQuarterTemp 
group by TagName

------------------------------------------------------------------------------------
--Posts_ResponsiveSubscribersTemp-Intermediate table for computing ResponsiveUsersPerTag

select T.OwnerUserId as OwnerUserId,P.Tags as Tags
into Posts_ResponsiveSubscribersTemp 
from 
(select ParentId,OwnerUserId,CreationDate 
 from Posts_ThirdQuarter 
 where PostTypeId=2
) T
inner join
(select Id,Tags,CreationDate 
 from Posts_ThirdQuarter 
 where PostTypeId=1
) P 
on T.ParentId=P.Id
where datediff(HH,P.CreationDate,T.CreationDate)=0
---------------------------------------------------------------------------------------
--dbo.ResponsiveUsersTemp-Intermediate table for computing ResponsiveUsersPerTag

select U.Id as USERID,Tags 
into dbo.ResponsiveUsersTemp
from dbo.Users U
inner join 
(select Owneruserid,Tags 
 from Posts_ResponsiveSubscribersTemp
)P
on U.ID= P.OwnerUserId
----------------------------------------------------------------------------------------- 
--dbo.ResponsiveUsersnew-Intermediate table for computing ResponsiveUsersPerTag

select USERID,TagName
into dbo.ResponsiveUsersnew
from dbo.tags 
inner join 
(select USERID,Tags 
 from dbo.ResponsiveUserstemp
)X
on Tags like '%<'+TagName+'>%' collate SQL_Latin1_General_CP1_CS_AS
------------------------------------------------------------------------------------------
                           /********* Responsive Subscribers Per Tag***********/
/* Computing ResponsiveUsersPerTag Table which contains records of each tag with the number of responsive 
subscribers of that tag in the span of third quarter 2014*/

select TagName,count(1) as ResponsiveSubscribers 
into ResponsiveUsersPerTag 
from dbo.responsiveusersnew 
group by TagName

----------------------------------------------------------------------------------------
--dbo.PostsActiveusersTemp-Intermediate table for finding ActiveSubscribers
-- Obtaining Tags Column for answer records(i.e. posttypeid=2) from questionrecords(i.e. Posttypeid=1) 

select T.OwnerUserId as OwnerUserId,P.Tags as Tags
into dbo.PostsActiveusersTemp 
from 
(select ParentId,OwnerUserId 
 from Posts_ThirdQuarter 
 where PostTypeId=2
) T
inner join
(select Id,Tags 
 from Posts 
 where PostTypeId=1
) P 
on T.ParentId=P.Id
--------------------------------------------------------------------------------------
--dbo.ActiveUsersTemp-Intermediate for finding ActiveSubscribers
select U.Id as USERID,Tags 
into dbo.ActiveUsersTemp
from dbo.Users U
inner join 
(select Owneruserid,Tags 
 from PostsActiveUsersTemp
)P
on U.ID= P.OwnerUserId
---------------------------------------------------------------------------------------------
--dbo.ActiveUsers-Intermediate table for finding ActiveSubscribers
select USERID,TagName,Tags
into dbo.ActiveUsers
from dbo.tags 
inner join 
(select USERID,Tags 
 from dbo.ActiveUsersTemp
)X
on Tags like '%<'+TagName+'>%' collate SQL_Latin1_General_CP1_CS_AS

------------------------------------------------------------------------------------------------
--QuestionsAnsweredbyUSERPERTAG-Intermediate table for finding ActiveSubscribers
select UserID,TagName,Count(1) as #questionsAnsweredbyUSERPERTAG  
into QuestionsAnsweredbyUSERPERTAG
from dbo.ActiveUsers 
group by TagName,USERID
 
-------------------------------------------------------------------------------------------------
--Computing number of active users from ActiveUsers table which contain records of each instance 
--of userid,TagName when a user answers a question containing a particular tag by classifying Users
-- as active when they answer thresholds of 10,20 and 30 questions for a particular tag in a span of 
--3 months
                /********* Active Subscribers Per Tag 10,20 and 30 threshold***********/
With activeUsers10 as
(
Select USERID,TagName from  QuestionsAnsweredbyUSERPERTAG where #questionsAnsweredbyUSERPERTAG>=10
), 
activeUsers20 as
(
Select USERID,TagName from  QuestionsAnsweredbyUSERPERTAG where #questionsAnsweredbyUSERPERTAG>=20
),
activeUsers30 as
(
Select USERID,TagName from  QuestionsAnsweredbyUSERPERTAG where #questionsAnsweredbyUSERPERTAG>=30
)
select R.TagName,R.ActiveUsers_10Threshold,R.ActiveUsers_20Threshold,Z.ActiveUsers_30Threshold 
into ActiveUsersThreshold 
from 
(Select P.TagName,P.ActiveUsers_10Threshold,Y.ActiveUsers_20Threshold 
 from
 (select Q.TagName,X.ActiveUsers_10Threshold 
 from Tags Q
 left join 
 (select TagName,count(1) as ActiveUsers_10Threshold 
 from activeUsers10 group by TagName
 )x 
 on Q.TagName=X.TagName
 )P
 left join 
 (select TagName,count(1) as ActiveUsers_20Threshold 
 from activeUsers20 
 group by TagName
 )Y 
 on P.TagName=Y.TagName
)R
left join 
(select TagName,count(1) as ActiveUsers_30Threshold 
from activeUsers30 
group by TagName
)Z 
on R.TagName=Z.TagName 
-----------------------------------------------------------------------------------------------
--Updating ActiveUsers_Threshold columns having nulls with default values 

update ActiveUsersThreshold set ActiveUsers_10Threshold=0 where ActiveUsers_10Threshold is NULL
update ActiveUsersThreshold set ActiveUsers_20Threshold=0 where ActiveUsers_20Threshold is NULL
update ActiveUsersThreshold set ActiveUsers_30Threshold=0 where ActiveUsers_30Threshold is NULL


-----------------------------------------------------------------------------------------------
--Computing Number of subcribers per Tag from dbo.ActiveUsersNew Table which contains records
--of instances of each user answering a question of a particular tagNAme
--dbo.ActiveUsersnew,subscribersPerTag-intermediate tables for computing Tag Popularity

select USERID,TagName
into dbo.ActiveUsersnew
from dbo.ActiveUsers

select TagName,count(1) as Subscribers 
into subscribersPerTag 
from dbo.Activeusersnew 
group by TagName
 

----------------------------------------------------------------------------------------
--Computing TagPopularity based on TagFrequency Threshold of 25,50 and 100 from TagFrequency Table
--and storing it in TagFrequencyThresholdTable 
                    /*********Tag Popularity 25,50 and 100 threshold***********/
select TagName,TagFrequency,
Case when TagFrequency>=25 then 50
else 25 end as TagPopularity_25Threshold,
Case when TagFrequency>=50 then 75
else 25 end as TagPopularity_50Threshold,
Case when TagFrequency>=100 then 100
else 25 end as TagPopularity_100Threshold
into TagFrequencyThreshold
from TagFrequency
------------------------------------------------------------------------------------------
--Combining all TagRelated Features by joining ActiveusersThreshold,SubscribersPerTag,
--ResponsiveUsersPerTag and TagFrequencyThreshold tables

Select T.TagName,L.TagFrequency,L.TagPopularity_25Threshold,L.TagPopularity_50Threshold,
L.TagPopularity_100Threshold,T.ActiveSubscribers_10Threshold,T.ActiveSubscribers_20Threshold,
T.ActiveSubscribers_30Threshold,T.Subscriberspertag,T.ResponsiveSubscribersPerTag
into TagFeaturesTemp
from (Select X.TagName,X.ActiveSubscribers_10Threshold,X.ActiveSubscribers_20Threshold,
      X.ActiveSubscribers_30Threshold,X.Subscriberspertag,R.ResponsiveSubscribers as ResponsiveSubscribersPerTag
      from (select a.TagName,a.ActiveUsers_10Threshold as ActiveSubscribers_10Threshold,
            a.ActiveUsers_20Threshold as ActiveSubscribers_20Threshold,
            a.ActiveUsers_30Threshold as ActiveSubscribers_30Threshold,s.Subscribers as Subscriberspertag
            from ActiveUsersThreshold a
            left join 
            subscribersPerTag S
            on a.TagName=S.TagName
		  )X
          left join
          ResponsiveUsersPerTag R 
          on X.TagName=R.TagName
	 )T
left join
TagFrequencyThreshold L
on T.TagName=L.TagName

---------------------------------------------------------------------------------------
--Updating all NULL values in TagFeaturesTemp table with their default values
 
update TagFeaturesTemp Set TagFrequency=0 where TagFrequency is NULL
Update TagFeaturesTemp Set TagPopularity_25Threshold=25 where TagPopularity_25Threshold is NULL
Update TagFeaturesTemp Set TagPopularity_50Threshold=25 where TagPopularity_50Threshold is NULL
Update TagFeaturesTemp Set TagPopularity_100Threshold=25 where TagPopularity_100Threshold is NULL 
update TagFeaturesTemp set Subscriberspertag=0 where subscriberspertag is NULL
update TagFeaturesTemp Set ResponsiveSubscribersPerTag=0 where ResponsiveSubscribersPerTag is NULL



--------------------------------------------------------------------------------------
      --TagFeature final Table after computing %ActiveSubscribers and %responsiveSubscribers  
select
TagName,
TagFrequency,
TagPopularity_25Threshold,
TagPopularity_50Threshold,
TagPopularity_100Threshold,
ActiveSubscribers_10Threshold,
ActiveSubscribers_20Threshold,
ActiveSubscribers_30Threshold,
case when subscriberspertag!=0 then 
(Convert(float,ActiveSubscribers_10Threshold)/convert(float,SubscribersperTag)) 
else 0 end as PercentActiveSubscribers_10Threshold,
case when subscriberspertag!=0 then 
(Convert(float,ActiveSubscribers_20Threshold)/convert(float,SubscribersperTag)) 
else 0 end as PercentActiveSubscribers_20Threshold,
case when subscriberspertag!=0 then 
(Convert(float,ActiveSubscribers_30Threshold)/convert(float,SubscribersperTag)) 
else 0 end as PercentActiveSubscribers_30Threshold,
Subscriberspertag,
ResponsiveSubscribersPerTag,
case when subscriberspertag!=0 then 
(Convert(float,ResponsiveSubscribersPerTag)/convert(float,SubscribersperTag)) 
else 0 end as PercentResponsiveSubscribersPerTag
into dbo.TagFeatures
from dbo.TagFeaturesTemp 
order by TagFrequency desc


--------------------------------------------------------------------------------------------------------------------
                             /***** Questioneer Based Features*********/
select OwnerUserId,count(1) as QuestionspostedUSER,
Avg(case when datediff(MI,CreationDate,ISNULL(FirstAnswerDate,GETDATE()))>1440 then 1440
else datediff(MI,CreationDate,ISNULL(FirstAnswerDate,GETDATE())) end) as AvgAnswerTime
into QuestionereDataTraining
from dbo.Posts_ThirdQuarter_FAD
group by OwnerUserId 
-------------------------------------------------------------------------------------------------------------------
select
ID,
Tags,
CreationDate,
FirstAnswerDate,
AcceptedAnswerID,
Body,
P.OwnerUserId,
Title,
ISNULL(Q.QuestionspostedUSER,0) as NPostedQuestionsByUser,
ISNULL(Q.AvgAnswerTime,17) as AvgAnswerTime
into dbo.Posts_ThirdQuarterTrain_FAD
from dbo.Posts_ThirdQuarter_FAD P
left join QuestionereDataTraining Q
on P.OwnerUserId=Q.OwnerUserId

-----------------------------------------------------------------------------------------------------------------------
select
ID,
Tags,
CreationDate,
FirstAnswerDate,
AcceptedAnswerID,
Body,
P.OwnerUserId,
Title,
ISNULL(Q.QuestionspostedUSER,0) as NPostedQuestionsByUser,
ISNULL(Q.AvgAnswerTime,17) as AvgAnswerTime
into dbo.Posts_ThirdQuarterTest_UD_FAD
from dbo.Posts_ThirdQuarterTest_FAD P
left join QuestionereDataTraining Q
on P.OwnerUserId=Q.OwnerUserId

---------------------------------------------------------------------------------------------------------------
--Posts_ThirdQuarter_FAD-Intermediate table for computing first answer time or response time of each post of Training data

select N.Id as ID,Tags,CreationDate,FirstAnswerDate,AcceptedAnswerID,Body,OwnerUserId,Title 
into Posts_ThirdQuarter_FAD 
from
(select Id,Tags,CreationDate,AcceptedAnswerID,Body,OwnerUserId,Title 
 from Posts_ThirdQuarter 
 where PostTypeId=1
)N
inner join
(select Id,min(Creationdate) as FirstAnswerDate 
 from 
  (select ParentId,CreationDate 
   from Posts_ThirdQuarter 
   where PostTypeId=2
  ) T
  inner join
  (Select Id 
   from Posts_ThirdQuarter 
   where PostTypeId=1
  ) P
  on T.ParentId=P.ID 
  group by Id
) X 
on N.ID=X.ID

------------------------------------------------------------------------------------------------------------------------
--Posts_ThirdQuarterTest_FAD-Intermediate table for computing first answer time or response time of each post of Test data

select N.Id as ID,Tags,CreationDate,FirstAnswerDate,AcceptedAnswerID,Body,OwnerUserId,Title 
into Posts_ThirdQuarterTest_FAD 
from
(select Id,Tags,CreationDate,AcceptedAnswerID,Body,OwnerUserId,Title 
 from Posts_Test 
 where PostTypeId=1
)N
inner join
(select Id,min(Creationdate) as FirstAnswerDate 
 from 
  (select ParentId,CreationDate 
   from Posts_Test 
   where PostTypeId=2
  ) T
  inner join
  (Select Id 
   from Posts_Test 
   where PostTypeId=1
  ) P
  on T.ParentId=P.ID 
  group by Id
) X 
on N.ID=X.ID

----------------------------------------------------------------------------------------------------------
-- computing non tag features and first response time for posts in training data
                             /***********Non Tag Features **************/ 
select
ID,
Tags,
CreationDate,
FirstAnswerDate,
AcceptedAnswerID,
Body,
OwnerUserId,
Title,
NPostedQuestionsByUser,
AvgAnswerTime,
case when datediff(MI,CreationDate,ISNULL(FirstAnswerDate,GETDATE()))>1440 then 1440
else datediff(MI,CreationDate,ISNULL(FirstAnswerDate,GETDATE())) end as FirstAnswerTime,
(LEN(Body) - LEN(REPLACE(Body,'<Img src=', '')))/LEN('<img src=') Num_of_images,
(LEN(Body) - LEN(REPLACE(Body,'<code>', '')))/LEN('<code>') Num_code_Snippets,
len(body) as Body_Length,
len(Title) as Title_Length,
case when Title like '%?' then 1 else 0 end as End_Question_mark,
case when Title like 'Wh%' then 1 
     when title like 'hw%' then 1 else 0 end as Begin_Question_word,
case when DATEPART(DW,CreationDate)=1 then 1
     When DATEPART(DW,CreationDate)=7 then 1 else 0 end as IS_Weekend
into dbo.Posts_ThirdQuarterFinal_FAD
from dbo.Posts_ThirdQuarterTrain_FAD

---------------------------------------------------------------------------------------------------------------------
-- computing non tag features and first response time for posts in test data 


select
ID,
Tags,
CreationDate,
FirstAnswerDate,
AcceptedAnswerID,
Body,
OwnerUserId,
Title,
NPostedQuestionsByUser,
AvgAnswerTime,
case when datediff(MI,CreationDate,ISNULL(FirstAnswerDate,GETDATE()))>1440 then 1440
else datediff(MI,CreationDate,ISNULL(FirstAnswerDate,GETDATE())) end as FirstAnswerTime,
(LEN(Body) - LEN(REPLACE(Body,'<Img src=', '')))/LEN('<img src=') Num_of_images,
(LEN(Body) - LEN(REPLACE(Body,'<code>', '')))/LEN('<code>') Num_code_Snippets,
len(body) as Body_Length,
len(Title) as Title_Length,
case when Title like '%?' then 1 else 0 end as End_Question_mark,
case when Title like 'Wh%' then 1 
     when title like 'hw%' then 1 else 0 end as Begin_Question_word,
case when DATEPART(DW,CreationDate)=1 then 1
     When DATEPART(DW,CreationDate)=7 then 1 else 0 end as IS_Weekend
into dbo.Posts_ThirdQuarterTestFinal_FAD
from dbo.Posts_ThirdQuarterTest_UD_FAD

----------------------------------------------------------------------------------------------------------------
--Posts_TQID_FAD-Intermediate Table for integrating Tag Features table with that of posts table of Train Data
select 
P.id,
T.[TagFrequency],
case when T.[TagPopularity_25Threshold]=25 then 0 else 1 end as [TagPopularity_25Threshold],
case when T.[TagPopularity_50Threshold]=25 then 0 else 1 end as [TagPopularity_50Threshold],
case when T.[TagPopularity_100Threshold]=25 then 0 else 1 end as [TagPopularity_100Threshold],
T.[ActiveSubscribers_10Threshold],
T.[ActiveSubscribers_20Threshold],
T.[ActiveSubscribers_30Threshold],
T.[PercentActiveSubscribers_10Threshold],
T.[PercentActiveSubscribers_20Threshold],
T.[PercentActiveSubscribers_30Threshold],
T.[Subscriberspertag],
T.[ResponsiveSubscribersPerTag],
T.[PercentResponsiveSubscribersPerTag],
1 as flag
into Posts_TQID_FAD
from posts_thirdquarterfinal_FAD P 
inner join TagFeatures T 
on P.Tags like '%<'+T.TagName+'>%' collate SQL_Latin1_General_CP1_CS_AS


-------------------------------------------------------------------------------------------------------------------
--posts_TagFeaturesTrain-Intermediate Table for integrating Tag Features table with that of posts table of Train Data

select 
id,
avg([TagFrequency]) as TagFrequency,
sum([TagPopularity_25Threshold]) as [numofPopulartags_25Threshold],
sum([TagPopularity_50Threshold]) as [numofPopulartags_50Threshold],
sum([TagPopularity_100Threshold]) as [numofPopulartags_100Threshold],
avg([ActiveSubscribers_10Threshold]) as [ActiveSubscribers_10Threshold],
avg([ActiveSubscribers_20Threshold]) as [ActiveSubscribers_20Threshold],
avg([ActiveSubscribers_30Threshold]) as [ActiveSubscribers_30Threshold],
avg([PercentActiveSubscribers_10Threshold]) as [PercentActiveSubscribers_10Threshold],
avg([PercentActiveSubscribers_20Threshold]) as [PercentActiveSubscribers_20Threshold],
avg([PercentActiveSubscribers_30Threshold]) as [PercentActiveSubscribers_30Threshold],
avg([Subscriberspertag]) as Subscribers,
avg([ResponsiveSubscribersPerTag]) as ResponsiveSubscribers,
avg([PercentResponsiveSubscribersPerTag]) as [PercentResponsiveSubscribers]
into posts_TagFeaturesTrain
from Posts_TQID_FAD
group by id
------------------------------------------------------------------------------------------------------
-- Posts_FullFeaturesTrainFinal- Full features table of training data excluding user reputation

select
p.[ID],
p.[Tags],
p.[CreationDate],
p.[FirstAnswerDate],
p.[AcceptedAnswerID],
p.[Body],
p.[OwnerUserId],
p.[Title],
p.[NPostedQuestionsByUser],
p.[AvgAnswerTime],
p.[FirstAnswerTime],
p.[Num_of_images],
p.[Num_code_Snippets],
p.[Body_Length],
p.[Title_Length],
p.[End_Question_mark],
p.[Begin_Question_word],
p.[IS_Weekend],
T.TagFrequency,
T.[numofPopulartags_25Threshold],
T.[numofPopulartags_50Threshold],
T.[numofPopulartags_100Threshold],
T.[ActiveSubscribers_10Threshold],
T.[ActiveSubscribers_20Threshold],
T.[ActiveSubscribers_30Threshold],
T.[PercentActiveSubscribers_10Threshold],
T.[PercentActiveSubscribers_20Threshold],
T.[PercentActiveSubscribers_30Threshold],
T.Subscribers,
T.ResponsiveSubscribers,
T.[PercentResponsiveSubscribers]
into Posts_FullFeaturesTrainFinal
from 
dbo.Posts_ThirdQuarterFinal_FAD P
inner join posts_TagFeaturesTrain T
on P.Id=T.id

---------------------------------------------------------------------------------------------------------
-- Posts_FullFeaturesTrainFinal_R-Final Full features table of training data including user reputation
               /****************** Final Feature table of Train Data*************************/
select
T.[ID],
[Tags],
T.[CreationDate],
[FirstAnswerDate],
[AcceptedAnswerID],
[Body],
[OwnerUserId],
[Title],
[NPostedQuestionsByUser],
[AvgAnswerTime],
[FirstAnswerTime],
[Num_of_images],
[Num_code_Snippets],
[Body_Length],
[Title_Length],
[End_Question_mark],
[Begin_Question_word],
[IS_Weekend],
TagFrequency,
[numofPopulartags_25Threshold],
[numofPopulartags_50Threshold],
[numofPopulartags_100Threshold],
[ActiveSubscribers_10Threshold],
[ActiveSubscribers_20Threshold],
[ActiveSubscribers_30Threshold],
[PercentActiveSubscribers_10Threshold],
[PercentActiveSubscribers_20Threshold],
[PercentActiveSubscribers_30Threshold],
Subscribers,
ResponsiveSubscribers,
[PercentResponsiveSubscribers],
isnull(Reputation,0) as [UserReputation] 
into Posts_FullFeaturesTrainFinal_R
from 
Posts_FullFeaturesTrainFinal T 
left join 
dbo.Users U
on  
T.[OwnerUserId]=U.Id

--------------------------------------------------------------------------------------------------------------------
--Posts_TQTst_FAD-Intermediate Table for integrating Tag Features table with that of posts table of Test Data

select 
P.id,
T.[TagFrequency],
case when T.[TagPopularity_25Threshold]=25 then 0 else 1 end as [TagPopularity_25Threshold],
case when T.[TagPopularity_50Threshold]=25 then 0 else 1 end as [TagPopularity_50Threshold],
case when T.[TagPopularity_100Threshold]=25 then 0 else 1 end as [TagPopularity_100Threshold],
T.[ActiveSubscribers_10Threshold],
T.[ActiveSubscribers_20Threshold],
T.[ActiveSubscribers_30Threshold],
T.[PercentActiveSubscribers_10Threshold],
T.[PercentActiveSubscribers_20Threshold],
T.[PercentActiveSubscribers_30Threshold],
T.[Subscriberspertag],
T.[ResponsiveSubscribersPerTag],
T.[PercentResponsiveSubscribersPerTag],
1 as flag
into Posts_TQTstID_FAD
from posts_thirdquarterTestfinal_FAD P 
inner join 
TagFeatures T 
on P.Tags like '%<'+T.TagName+'>%' collate SQL_Latin1_General_CP1_CS_AS

-------------------------------------------------------------------------------------------------------------------
--posts_TagFeaturesTest-Intermediate Table for integrating Tag Features table with that of posts table of Test Data

select 
id,
avg([TagFrequency]) as TagFrequency,
sum([TagPopularity_25Threshold]) as [numofPopulartags_25Threshold],
sum([TagPopularity_50Threshold]) as [numofPopulartags_50Threshold],
sum([TagPopularity_100Threshold]) as [numofPopulartags_100Threshold],
avg([ActiveSubscribers_10Threshold]) as [ActiveSubscribers_10Threshold],
avg([ActiveSubscribers_20Threshold]) as [ActiveSubscribers_20Threshold],
avg([ActiveSubscribers_30Threshold]) as [ActiveSubscribers_30Threshold],
avg([PercentActiveSubscribers_10Threshold]) as [PercentActiveSubscribers_10Threshold],
avg([PercentActiveSubscribers_20Threshold]) as [PercentActiveSubscribers_20Threshold],
avg([PercentActiveSubscribers_30Threshold]) as [PercentActiveSubscribers_30Threshold],
avg([Subscriberspertag]) as Subscribers,
avg([ResponsiveSubscribersPerTag]) as ResponsiveSubscribers,
avg([PercentResponsiveSubscribersPerTag]) as [PercentResponsiveSubscribers]
into posts_TagFeaturesTest
from Posts_TQTstID_FAD
group by id
------------------------------------------------------------------------------------------------------
-- Posts_FullFeaturesTestFinal-Final Full features table of test data excluding user reputation
           
select
p.[ID],
p.[Tags],
p.[CreationDate],
p.[FirstAnswerDate],
p.[AcceptedAnswerID],
p.[Body],
p.[OwnerUserId],
p.[Title],
p.[NPostedQuestionsByUser],
p.[AvgAnswerTime],
p.[FirstAnswerTime],
p.[Num_of_images],
p.[Num_code_Snippets],
p.[Body_Length],
p.[Title_Length],
p.[End_Question_mark],
p.[Begin_Question_word],
p.[IS_Weekend],
T.TagFrequency,
T.[numofPopulartags_25Threshold],
T.[numofPopulartags_50Threshold],
T.[numofPopulartags_100Threshold],
T.[ActiveSubscribers_10Threshold],
T.[ActiveSubscribers_20Threshold],
T.[ActiveSubscribers_30Threshold],
T.[PercentActiveSubscribers_10Threshold],
T.[PercentActiveSubscribers_20Threshold],
T.[PercentActiveSubscribers_30Threshold],
T.Subscribers,
T.ResponsiveSubscribers,
T.[PercentResponsiveSubscribers]
into Posts_FullFeaturesTestFinal
from 
dbo.Posts_ThirdQuarterTestFinal_FAD P
inner join 
posts_TagFeaturesTest T
on P.Id=T.id
-------------------------------------------------------------------------------------------------------------------------
-- Posts_FullFeaturesTestFinal_R-Final Full features table of test data including user reputation
             /****************** Final Feature table of Test Data*************************/

select
T.[ID],
[Tags],
T.[CreationDate],
[FirstAnswerDate],
[AcceptedAnswerID],
[Body],
[OwnerUserId],
[Title],
[NPostedQuestionsByUser],
[AvgAnswerTime],
[FirstAnswerTime],
[Num_of_images],
[Num_code_Snippets],
[Body_Length],
[Title_Length],
[End_Question_mark],
[Begin_Question_word],
[IS_Weekend],
TagFrequency,
[numofPopulartags_25Threshold],
[numofPopulartags_50Threshold],
[numofPopulartags_100Threshold],
[ActiveSubscribers_10Threshold],
[ActiveSubscribers_20Threshold],
[ActiveSubscribers_30Threshold],
[PercentActiveSubscribers_10Threshold],
[PercentActiveSubscribers_20Threshold],
[PercentActiveSubscribers_30Threshold],
Subscribers,
ResponsiveSubscribers,
[PercentResponsiveSubscribers],
isnull(Reputation,0) as [UserReputation] 
into Posts_FullFeaturesTestFinal_R
from 
Posts_FullFeaturesTestFinal T 
left join 
dbo.Users U
on  
T.[OwnerUserId]=U.Id

---------------------------------------------------------------------------------------------------------------------

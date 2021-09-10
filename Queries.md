### How many reviewers are in the dataset with more than 40 reviews each?
```
SELECT COUNT(*)
FROM (  SELECT `userid`, count(*) as occurrence
        FROM `rawrev` -- this table contains all the reviews
        group by userid
        having count(*)>40 ) a;
```
###  A summary table (ID, total thumb-ups received, total thumb-downs received, difference between total thumb-ups and thumb-downs)
```
create table revwrsummary  -- reviewer summary
as  SELECT userid, SUM(up) as totalup, SUM(down) as totaldown, sum(diff) as totaldiff
    FROM rawrev            
    GROUP BY userid;
```
### Another field: helpratio
```
UPDATE `revwrsummary` 
SET`helpratio`= (`totalup`/(`totalup`+`totaldown`));
```

### Create a table of best 50 reviewers based on their respective helpfulness ratio
-- table `revwrsummary` contains fields: `userid`,`totalup`,`totaldown`,`totaldiff`,`helpratio`
-- table useridg40 contains `userid` of those reviewers who have more than 40 reviews each
```
CREATE TABLE ratio_best50
AS
select `userid`,`totalup`,`totaldown`,`totaldiff`,`helpratio` 
FROM `revwrsummary` inner join  useridg40 USING(userid)
ORDER by helpratio DESC, totaldiff DESC
limit 50;
```

## Queries related to the [first algorithm](https://www.sciencedirect.com/science/article/pii/S2405844021015127#enun_Algorithm_1)

### (Figure 4: step-1) Create a table of reviewers with more than 50 reviews each
```
CREATE TABLE more_than_50
AS
SELECT `userid`, count(*) as occur
FROM `allreviews` 
group by userid
having count(*)>50;
```

### (Figure 4: step-2) Select top 1000 reviewers based on their thumb-up to thumb-down difference (within the set of reviewers with more than 50 reviews each)

```
create table top1000_hlp_diff
as
select *
from `revwrsummary` inner join `more_than_50` USING(userid)
order by `totaldiff` desc
limit 1000;
```

### (Figure 4: step-3) Select top 500 reviewers from the above 1000- based on their thumb-up to total-thumb ratio
```
create table top500ratio
as
select *
from top1000_hlp_diff
order by helpratio desc
limit 500;
```

### (Figure 4: step-4) Select top 200 reviewers from the above 500- based on their lowest MSE
##### Calculating MSE requires several queries. At first, find the products associated with the 500 reviewers:
```
create table `eligiblemovies`
as
SELECT DISTINCT productid
FROM 
allreviews t1 INNER JOIN mthanfifty t2 
    ON t1.userid = t2.userid;
```

##### Now find all the reviews associated to these products:
```
create table eligiblereviews
as
select *
from
allreviews t1 INNER JOIN  eligiblemovies t2 
using productid
```

##### Calculate the rating of each product considering the helpfulness votes
Update a column that denotes the weighted rating [(Eq.5)](https://www.sciencedirect.com/science/article/pii/S2405844021015127#fd5) of that review:
```
UPDATE `eligiblereviews` 
SET `wtdrating`= (`up`+1)*`rating`;
```

Calculate the rating:
```
create table sfs_rating_hlp
as
SELECT productid, SUM(wtdrating) as totalrating, SUM(up) as totalup, count(*) as totalrev
FROM `eligiblereviews`            
GROUP BY productid;

-- Now add column `rating_with_hlp` and store the rating in this column
UPDATE `sfs_rating_hlp` SET `rating_with_hlp`=`totalrating`/(`totalup`+`totalrev`)
```

##### Calculate the rating of each product without considering the helpfulness votes (simple average)
[See (Eq.1)](https://www.sciencedirect.com/science/article/pii/S2405844021015127#fd1)
```
create table sfs_rating_avg
as
SELECT productid, SUM(rating) as totalrating, count(*) as totalrev
FROM `eligiblereviews`            
GROUP BY productid;

//add column avg_rating
UPDATE `sfs_rating_avg` SET `avg_rating`=`totalrating`/`totalrev`
```

##### Finally select the top 200 reviewers from the above 500- based on their lowest MSE (true rating --> considering the helpfulness votes)
```
CREATE table sfs_mse_200_hlp
AS
select userid, avger, mse 
from (
    SELECT userid, (sum(abs(rating-rating_with_hlp))/count(*)) as avger, (sum(abs(rating-rating_with_hlp)*abs(rating-rating_with_hlp))/count(*)) as mse 
    FROM (   rawrev INNER JOIN  top500ratio       	using (userid)   )
    group by userid)t1
order by mse ASC
limit 200;
```

##### Finally select the top 200 reviewers from the above 500- based on their lowest MSE (true rating --> simple average)
```
CREATE table sfs_mse_200_avg
AS
select userid, avger, mse 
from (
    SELECT userid, (sum(abs(rating-avg_rating))/count(*)) as avger, (sum(abs(rating-avg_rating)*abs(rating-avg_rating))/count(*)) as mse 
    FROM (   rawrev INNER JOIN  top500ratio       	using (userid)   )
    group by userid)t1
order by mse ASC
limit 200;
```

### Calculating the avg. error and avg. MSE of the final set of recommended reviewers
**(true rating --> considering the helpfulness votes)**
```
select avg(avger), avg(mse) 
from (
    SELECT userid, (sum(abs(rating-`rating_with_hlp`))/count(*)) as avger, (sum(abs(rating-`rating_with_hlp`)*abs(rating-`rating_with_hlp`))/count(*)) as mse 
    FROM (rawrev  INNER  JOIN sfs_mse_200_hlp using (userid)   )
    group by userid
    )t1; 
```

**(true rating --> simple average)**
```
select avg(avger), avg(mse) 
from (
    SELECT userid, (sum(abs(rating-`avg_rating`))/count(*)) as avger, (sum(abs(rating-`avg_rating`)*abs(rating-`avg_rating`))/count(*)) as mse 
    FROM (rawrev  INNER  JOIN sfs_mse_200_avg using (userid)   )
    group by userid
    )t1; 
```

## Queries related to the [second algorithm](https://www.sciencedirect.com/science/article/pii/S2405844021015127#enun_Algorithm_2)

### Create a root set of eligible reviewers
In step-1 of our experiment, we construct the root set of potential experts. We experimented with two different root sets. Set-1 was created by selecting all reviewers, each of whom has made more than 50 reviews. Set-2 contained the top 2000 reviewers who have the largest thumb-ups to the thumb-downs difference from set-1.

**Root Set 1:**
```
CREATE TABLE userid_50
AS
SELECT `userid`
FROM `rawrev` -- this table contains all the reviews
group by userid
having count(*)>50;
```

**Root Set 2:**
```
CREATE TABLE userid_diff_2000
AS
select *
from `revwrsummary` inner join `userid_50` USING(userid)
order by `totaldiff` desc
limit 2000;
```
```
ALTER TABLE `userid_50` -- OR `userid_diff_2000`
ADD `totalerr` DOUBLE NOT NULL DEFAULT '0.0000000' AFTER `userid`, 
ADD `ttlwtmvcount` INT(10) NOT NULL DEFAULT '0' AFTER `totalerr`, 
ADD `avgerr DOUBLE NOT NULL DEFAULT '0.00000' AFTER `ttlwtmvcount`, 
ADD `revcount` INT(10) NOT NULL DEFAULT '0' AFTER `avgerr`, 
ADD `reviewer_weight` DOUBLE NOT NULL DEFAULT '0.00000' AFTER `revcount`;
```

### Assign weights to products according to the number of reviews. 

Create the set of associated products and associated reviews (related to the root set)
**A table of products, their earliest posted review time, and number of reviews**
```
CREATE table `product_stats`
AS
SELECT productid, MIN(timee), COUNT(*) as originalrevcount
FROM `rawrev` 
GROUP BY productid;
```

**A table for products associated with the reviewers of Root set-1**
```
CREATE table `associated_products_50`
AS
SELECT `productid`, `timee`, `originalrevcount`
FROM `product_stats` INNER JOIN (
                                SELECT DISTINCT productid
                                FROM userid_50 INNER JOIN rawrev
                                using (userid)
	                              )temp using (productid)
ORDER BY timee ASC;

-- Add a column representing weights of the product

ALTER TABLE `associated_products_50` 
ADD `product_weight` INT(10) NOT NULL DEFAULT '1' 
AFTER `originalrevcount`;
```

Finally assign the weights:
```
UPDATE `associated_products_50`
SET product_weight = CASE
         WHEN originalrevcount < 31 THEN 1
         WHEN originalrevcount > 30 AND originalrevcount < 61 THEN 2
         WHEN originalrevcount > 60 AND originalrevcount < 91 THEN 3
         WHEN originalrevcount > 90 AND originalrevcount < 121 THEN 4
         WHEN originalrevcount > 120 AND originalrevcount < 151 THEN 5
         WHEN originalrevcount > 150 AND originalrevcount < 181 THEN 6
         WHEN originalrevcount > 180 AND originalrevcount < 211 THEN 7
         WHEN originalrevcount > 210 AND originalrevcount < 241 THEN 8
         WHEN originalrevcount > 240 AND originalrevcount < 271 THEN 9
         WHEN originalrevcount > 270 THEN 10
       END
;
```
Alternatively:
```
UPDATE `associated_products_50` SET `product_weight`=1 where `originalrevcount`>0;
UPDATE `associated_products_50` SET `product_weight`=2 where `originalrevcount`>30;
UPDATE `associated_products_50` SET `product_weight`=3 where `originalrevcount`>60;
UPDATE `associated_products_50` SET `product_weight`=4 where `originalrevcount`>90;
UPDATE `associated_products_50` SET `product_weight`=5 where `originalrevcount`>120;
UPDATE `associated_products_50` SET `product_weight`=6 where `originalrevcount`>150;
UPDATE `associated_products_50` SET `product_weight`=7 where `originalrevcount`>180;
UPDATE `associated_products_50` SET `product_weight`=8 where `originalrevcount`>210;
UPDATE `associated_products_50` SET `product_weight`=9 where `originalrevcount`>240;
UPDATE `associated_products_50` SET `product_weight`=10 where `originalrevcount`>270;
```


### Mutual Reinforcement

**At first, a table for all the reviews associated with the products reviewed by the set-1 reviewers**
```
INSERT INTO `associated_reviews_50plus`(`productid`, `userid`, `up`, `totalvote`, `rating`, `timee`, `wtdrating`)
SELECT productid, userid, up, totalvote, rating, timee as timee, (up+1)*rating as wtdrating 
FROM  `rawrev` 
      INNER join ( SELECT `productid` FROM `associated_products_50`) r 
      USING (`productid`);

-- Add two columns representing weight of the reviewer and `product_weight`

ALTER TABLE `associated_reviews_50plus` 
ADD `reviewer_weight` INT(10) NOT NULL DEFAULT '1' AFTER `wtdrating`,
ADD `product_weight` INT(10) NOT NULL DEFAULT '1' AFTER `reviewer_weight`,
ADD `tr` DOUBLE NOT NULL DEFAULT '0.0000000' AFTER `product_weight`;

```

**Update the products' weights in the associated reviews.**
```
UPDATE `associated_reviews_50plus` INNER JOIN `associated_products_50` USING(productid) 
SET `associated_reviews_50plus`.`product_weight` = `associated_products_50`.`product_weight`;
```

Here's the trigger that estimates the true-rating of each products and also updates user's weight
```
DELIMITER $$
CREATE TRIGGER `calculate_true_rating` AFTER INSERT ON `insert1by1` FOR EACH ROW
BEGIN
    DECLARE myTR  FLOAT DEFAULT 0.0000;
    DECLARE aver INT DEFAULT 1;
    DECLARE p INT DEFAULT 0;
    DECLARE  q INT DEFAULT 0;
    DECLARE  r FLOAT DEFAULT 0.0000;
    DECLARE tt TINYTEXT;
    SET tt = NEW.productid;
    
   	CREATE TEMPORARY TABLE temp -- (`productid`, `userid`, `up`, `totalvote`, `rating`, `timee`, `wtdrating`, `reviewer_weight`, `product_weight`, `tr`)
	  SELECT *
	  FROM `associated_reviews_50plus`
   	where productid=tt;

	  UPDATE temp 
    SET `reviewer_weight`=1;
    
    UPDATE temp INNER JOIN userid_50 #user's weight is transferred
    ON temp.userid=userid_50.userid
    SET temp.`reviewer_weight` = userid_50.`reviewer_weight`;
    
    UPDATE `associated_products_50` -- #true rating getting calculated
    SET truerating = (select @myTR:= (sum(wtdrating*`reviewer_weight`)/sum((up+1)*`reviewer_weight`))	from temp)
    where associated_products_50.productid=tt;

    UPDATE temp  -- truerating updated in temp
    SET tr=(@myTR);

    UPDATE userid_50 INNER JOIN `temp`
    ON temp.userid=userid_50.userid
    SET 
      `totalerr` = (@p:=`totalerr`+ `product_weight`*(abs(rating-tr))), 
      `ttlwtmvcount` = (@q:=`ttlwtmvcount` + (`product_weight`)), 
      `avgerr` = (@r:= (@p/@q)),
      `revcount` = `revcount` + 1,
      `weight`= CASE
                    WHEN round(@r*100) < 35 THEN 5
                    WHEN round(@r*100) > 34 AND round(@r*100) < 40 THEN 4
                    WHEN round(@r*100)> 39 AND round(@r*100) < 45 THEN 3
                    WHEN round(@r*100)> 44 AND round(@r*100) < 50 THEN 2
                    WHEN round(@r*100) > 49 THEN 1
	  END;
    Drop temporary table temp;
END$$
DELIMITER;

-- This query invokes the trigger 

INSERT INTO `insert1by1`(`productid`, `timee`, `weight`, `originalrevcount`, `truerating`) 
SELECT `productid`, `timee`, `weight`, `originalrevcount`, `truerating`
FROM some_table.associated_products_50
ORDER BY timee ASC;
```


To calculate the 'score' of reviewers and to obtain their peformance measures (i.e., avg. error and avg. MSE), let's add the derived true-rating in the associated reviews.
```
UPDATE `associated_reviews_50plus` INNER JOIN `associated_products_50` USING (productid)
SET `tr`= truerating;
```

Avg. error of the population is required to calculate the 'score'.
```
select avg(avger)
from
(
    SELECT (   SUM(ABS(rating-tr)) / COUNT(*)   ) AS avger
        FROM (  SELECT `associated_reviews_50plus`.`userid`, `associated_reviews_50plus`.rating, `associated_reviews_50plus`.tr 
              FROM `associated_reviews_50plus` INNER JOIN userid_50
              using (userid)
             ) b 
    	group by userid)
a;
```

-- suppose the population avg error is found to be 0.2379448806520861 in a query above.
-- Use the population avg. error along with different values of M to get different scores.
-- Then use those score to create different sets of recommended reviewers
```
update userid_50 set score= (avgerr*ttlwtmvcount)/(ttlwtmvcount);
create table M0top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*25)/(ttlwtmvcount+25);
create table M25top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*50)/(ttlwtmvcount+50);
create table M50top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*75)/(ttlwtmvcount+75);
create table M75top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*100)/(ttlwtmvcount+100);
create table M100top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*200)/(ttlwtmvcount+200);
create table M200top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*300)/(ttlwtmvcount+300);
create table M300top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*400)/(ttlwtmvcount+400);
create table M400top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*500)/(ttlwtmvcount+500);
create table M500top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*1000)/(ttlwtmvcount+1000);
create table M1000top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*1500)/(ttlwtmvcount+1500);
create table M1500top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*2000)/(ttlwtmvcount+2000);
create table M2000top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*3000)/(ttlwtmvcount+3000);
create table M3000top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*5000)/(ttlwtmvcount+5000);
create table M5000top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;

update userid_50 set score= (avgerr*ttlwtmvcount + 0.2379448806520861*10000)/(ttlwtmvcount+10000);
create table M10000top50 select `userid`, `ttlwtmvcount`, `totalerr`, `avgerr`, `weight`, `revcount`, `score` FROM userid_50 ORDER BY `score` ASC limit 50;
```

##### **Performance Report**

Let's calculate the avg. error and avg. MSE of the different sets of reviewers.
```
-- avg. error and avg. MSE of the reviewers from set `m0top50`
SELECT avg(avger), avg(mse) 
FROM (
      SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse 
      FROM (`associated_reviews_50plus` INNER JOIN  `m0top50` 	USING (userid)) 
      GROUP BY userid)t1; 

-- avg. error and avg. MSE of the reviewers from set `m25top50`
SELECT avg(avger), avg(mse) 
FROM (
      SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse 
      FROM (`associated_reviews_50plus` INNER JOIN  `m25top50`	USING (userid))     
      GROUP BY userid)t1; 

-- avg. error and avg. MSE of the reviewers from set `m50top50` to `m10000top50`
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m50top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m75top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m100top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m200top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m300top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m400top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m500top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m1000top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m1500top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m2000top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m3000top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m5000top50`	USING (userid)   )     GROUP BY userid)t1; 
SELECT avg(avger), avg(mse) FROM (SELECT userid, (SUM(ABS(rating-tr))/COUNT(*)) AS avger, (SUM(ABS(rating-tr)*abs(rating-tr))/COUNT(*)) AS mse FROM (`associated_reviews_50plus` INNER JOIN  `m10000top50`	USING (userid)   )     GROUP BY userid)t1; 
```

You may wish to measure how the avg. error and avg. mse differ for these set of reviewers if the estimated true-product rating is derived from different algorithms. In that case, just update the `tr` field with the alternative product rating and run the above queries to calculate the avg. error and avg. MSE.
```
UPDATE `associated_reviews_50plus` INNER JOIN `sfs_rating_avg` USING (productid)
SET `tr`= `avg_rating`;

-- OR

UPDATE `associated_reviews_50plus` INNER JOIN `sfs_rating_hlp` USING (productid)
SET `tr`= `rating_with_hlp`;
```

Let's calculate the product coverage for the different sets of recommended reviewers.
```
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m0top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m25top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m50top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m75top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m100top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m200top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m300top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m400top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m500top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m1000top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m1500top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m2000top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m3000top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m5000top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
SELECT count(*) FROM ( SELECT DISTINCT productid     FROM `m10000top50` INNER JOIN `associated_reviews_50plus` USING (userid) ) a;
```

Let's calculate the total number of reviews made by the different sets of recommended reviewers.
```
SELECT count(*) FROM `m0top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m25top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m50top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m75top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m100top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m200top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m300top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m400top50` INNER JOIN `associated_reviews_50plus` using (userid);
SELECT count(*) FROM `m500top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m1000top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m1500top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m2000top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m3000top50` INNER JOIN `associated_reviews_50plus` using (userid);
SELECT count(*) FROM `m5000top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
SELECT count(*) FROM `m10000top50` INNER JOIN `associated_reviews_50plus` using (userid) ;
```

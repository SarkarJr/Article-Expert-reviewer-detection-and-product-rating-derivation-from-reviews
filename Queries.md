1. How many reviewers are in the dataset with more than 40 reviews each?
```
SELECT COUNT(*)
FROM
(
SELECT `userid`, count(*) as occurrence
FROM `rawrev` -- this table contains all the reviews
group by userid
having count(*)>40) a;
```

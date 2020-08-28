drop procedure postReview;

DELIMITER //


CREATE PROCEDURE postReview (IN productName VARCHAR(100),IN customerId INT, IN title VARCHAR(100), IN detail VARCHAR(500), IN rateV1 INT, IN rateV2 INT, IN name VARCHAR(50), IN createdAt timestamp)

BEGIN

SET @PRODUCT_NAME= productName;
SET @CUSTOMER_ID = customerId;
SET @REVIEW_TITLE = title;
SET @REVIEW_DETAIL = detail;
SET @REVIEW_RATING1 = rateV1; -- Between 1 to 5
SET @REVIEW_RATING2 = rateV2; -- Between 1 to 5
SET @REVIEW_NICKNAME = name;
SET @REVIEW_CREATED_AT  = createdAt; -- OR date in YY-mm-dd HH:ii:ss format

SELECT entity_id INTO @PRODUCT_ID FROM m2catalog_product_entity_varchar WHERE attribute_id=71 AND value=@PRODUCT_NAME;

INSERT INTO `m2review` (`created_at`, `entity_id`, `entity_pk_value`, `status_id`) VALUES (@REVIEW_CREATED_AT, 1, @PRODUCT_ID, 1); 

SET @REVIEW_ID = (SELECT LAST_INSERT_ID());

INSERT INTO `m2review_detail` (`title`, `detail`, `nickname`, `store_id`, `customer_id`, `review_id`) VALUES (@REVIEW_TITLE, @REVIEW_DETAIL, @REVIEW_NICKNAME, @STORE_ID, @CUSTOMER_ID, @REVIEW_ID);

DELETE FROM `m2review_store` WHERE (review_id = @REVIEW_ID);
INSERT INTO `m2review_store` (`store_id`, `review_id`) VALUES (1, @REVIEW_ID);
INSERT INTO `m2review_store` (`store_id`, `review_id`) VALUES (0, @REVIEW_ID);

INSERT INTO `m2rating_option_vote` (`option_id`, `review_id`, `percent`, `value`, `remote_ip`, `remote_ip_long`, `customer_id`, `entity_pk_value`, `rating_id`) VALUES (@REVIEW_RATING1, @REVIEW_ID, (@REVIEW_RATING1*20), @REVIEW_RATING1, '99.99.99.99', '9999', @CUSTOMER_ID, @PRODUCT_ID, 1);

INSERT INTO `m2rating_option_vote` (`option_id`, `review_id`, `percent`, `value`, `remote_ip`, `remote_ip_long`, `customer_id`, `entity_pk_value`, `rating_id`) VALUES (5+@REVIEW_RATING2, @REVIEW_ID, (@REVIEW_RATING2*20), @REVIEW_RATING2, '99.99.99.99', '99.99.99.99', @CUSTOMER_ID, @PRODUCT_ID, 2);


-- Aggregated rating

SET @RATE_N=1;

SET @STORE_ID=0;

INSERT INTO m2rating_option_vote_aggregated (rating_id, entity_pk_value, vote_count, vote_value_sum, percent, percent_approved, store_id) SELECT * FROM (SELECT @RATE_N AS rating_id, @PRODUCT_ID AS entity_pk_value, 0 AS vote_count, 0 AS vote_value_sum, 0 AS percent, 0 AS percent_approved, @STORE_ID AS store_id) AS T WHERE NOT EXISTS (SELECT * FROM m2rating_option_vote_aggregated WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID));

SELECT `m2rating_option_vote_aggregated`.`primary_id` INTO @PRIMARY_ID FROM `m2rating_option_vote_aggregated` WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID);

SELECT COUNT(vote.vote_id), SUM(vote.value), COUNT(IF(review.status_id=1, vote.vote_id, NULL)), SUM(IF(review.status_id=1, vote.value, 0)) INTO @VC, @VS, @VCA, @VSA FROM `m2rating_option_vote` AS `vote`  INNER JOIN `m2review` AS `review` ON vote.review_id=review.review_id  LEFT JOIN `m2review_store` AS `store` ON vote.review_id=store.review_id  INNER JOIN `m2rating_store` AS `rstore` ON vote.rating_id=rstore.rating_id AND rstore.store_id=store.store_id WHERE (vote.rating_id = @RATE_N) AND (store.store_id=@STORE_ID) AND (vote.entity_pk_value = @PRODUCT_ID) GROUP BY `vote`.`rating_id`,   `vote`.`entity_pk_value`,   `store`.`store_id`;

UPDATE `m2rating_option_vote_aggregated` SET `rating_id` = @RATE_N, `entity_pk_value` = @PRODUCT_ID, `vote_count` = @VC, `vote_value_sum` = @VS, `percent` = @VS*20/@VC, `percent_approved` = @VSA*20/@VCA, `store_id` = @STORE_ID WHERE (primary_id = @PRIMARY_ID);

SET @STORE_ID=1;

INSERT INTO m2rating_option_vote_aggregated (rating_id, entity_pk_value, vote_count, vote_value_sum, percent, percent_approved, store_id) SELECT * FROM (SELECT @RATE_N AS rating_id, @PRODUCT_ID AS entity_pk_value, 0 AS vote_count, 0 AS vote_value_sum, 0 AS percent, 0 AS percent_approved, @STORE_ID AS store_id) AS T WHERE NOT EXISTS (SELECT * FROM m2rating_option_vote_aggregated WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID));

SELECT `m2rating_option_vote_aggregated`.`primary_id` INTO @PRIMARY_ID FROM `m2rating_option_vote_aggregated` WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID);

SELECT COUNT(vote.vote_id), SUM(vote.value), COUNT(IF(review.status_id=1, vote.vote_id, NULL)), SUM(IF(review.status_id=1, vote.value, 0)) INTO @VC, @VS, @VCA, @VSA FROM `m2rating_option_vote` AS `vote`  INNER JOIN `m2review` AS `review` ON vote.review_id=review.review_id  LEFT JOIN `m2review_store` AS `store` ON vote.review_id=store.review_id  INNER JOIN `m2rating_store` AS `rstore` ON vote.rating_id=rstore.rating_id AND rstore.store_id=store.store_id WHERE (vote.rating_id = @RATE_N) AND (store.store_id=@STORE_ID) AND (vote.entity_pk_value = @PRODUCT_ID) GROUP BY `vote`.`rating_id`,   `vote`.`entity_pk_value`,   `store`.`store_id`;

UPDATE `m2rating_option_vote_aggregated` SET `rating_id` = @RATE_N, `entity_pk_value` = @PRODUCT_ID, `vote_count` = @VC, `vote_value_sum` = @VS, `percent` = @VS*20/@VC, `percent_approved` = @VSA*20/@VCA, `store_id` = @STORE_ID WHERE (primary_id = @PRIMARY_ID);

SET @STORE_ID=2;

INSERT INTO m2rating_option_vote_aggregated (rating_id, entity_pk_value, vote_count, vote_value_sum, percent, percent_approved, store_id) SELECT * FROM (SELECT @RATE_N AS rating_id, @PRODUCT_ID AS entity_pk_value, 0 AS vote_count, 0 AS vote_value_sum, 0 AS percent, 0 AS percent_approved, @STORE_ID AS store_id) AS T WHERE NOT EXISTS (SELECT * FROM m2rating_option_vote_aggregated WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID));

SELECT `m2rating_option_vote_aggregated`.`primary_id` INTO @PRIMARY_ID FROM `m2rating_option_vote_aggregated` WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID);

SELECT COUNT(vote.vote_id), SUM(vote.value), COUNT(IF(review.status_id=1, vote.vote_id, NULL)), SUM(IF(review.status_id=1, vote.value, 0)) INTO @VC, @VS, @VCA, @VSA FROM `m2rating_option_vote` AS `vote`  INNER JOIN `m2review` AS `review` ON vote.review_id=review.review_id  LEFT JOIN `m2review_store` AS `store` ON vote.review_id=store.review_id  INNER JOIN `m2rating_store` AS `rstore` ON vote.rating_id=rstore.rating_id AND rstore.store_id=store.store_id WHERE (vote.rating_id = @RATE_N) AND (store.store_id=@STORE_ID) AND (vote.entity_pk_value = @PRODUCT_ID) GROUP BY `vote`.`rating_id`,   `vote`.`entity_pk_value`,   `store`.`store_id`;

UPDATE `m2rating_option_vote_aggregated` SET `rating_id` = @RATE_N, `entity_pk_value` = @PRODUCT_ID, `vote_count` = @VC, `vote_value_sum` = @VS, `percent` = @VS*20/@VC, `percent_approved` = @VSA*20/@VCA, `store_id` = @STORE_ID WHERE (primary_id = @PRIMARY_ID);


SET @RATE_N=2;

SET @STORE_ID=0;

INSERT INTO m2rating_option_vote_aggregated (rating_id, entity_pk_value, vote_count, vote_value_sum, percent, percent_approved, store_id) SELECT * FROM (SELECT @RATE_N AS rating_id, @PRODUCT_ID AS entity_pk_value, 0 AS vote_count, 0 AS vote_value_sum, 0 AS percent, 0 AS percent_approved, @STORE_ID AS store_id) AS T WHERE NOT EXISTS (SELECT * FROM m2rating_option_vote_aggregated WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID));

SELECT `m2rating_option_vote_aggregated`.`primary_id` INTO @PRIMARY_ID FROM `m2rating_option_vote_aggregated` WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID);

SELECT COUNT(vote.vote_id), SUM(vote.value), COUNT(IF(review.status_id=1, vote.vote_id, NULL)), SUM(IF(review.status_id=1, vote.value, 0)) INTO @VC, @VS, @VCA, @VSA FROM `m2rating_option_vote` AS `vote`  INNER JOIN `m2review` AS `review` ON vote.review_id=review.review_id  LEFT JOIN `m2review_store` AS `store` ON vote.review_id=store.review_id  INNER JOIN `m2rating_store` AS `rstore` ON vote.rating_id=rstore.rating_id AND rstore.store_id=store.store_id WHERE (vote.rating_id = @RATE_N) AND (store.store_id=@STORE_ID) AND (vote.entity_pk_value = @PRODUCT_ID) GROUP BY `vote`.`rating_id`,   `vote`.`entity_pk_value`,   `store`.`store_id`;

UPDATE `m2rating_option_vote_aggregated` SET `rating_id` = @RATE_N, `entity_pk_value` = @PRODUCT_ID, `vote_count` = @VC, `vote_value_sum` = @VS, `percent` = @VS*20/@VC, `percent_approved` = @VSA*20/@VCA, `store_id` = @STORE_ID WHERE (primary_id = @PRIMARY_ID);

SET @STORE_ID=1;

INSERT INTO m2rating_option_vote_aggregated (rating_id, entity_pk_value, vote_count, vote_value_sum, percent, percent_approved, store_id) SELECT * FROM (SELECT @RATE_N AS rating_id, @PRODUCT_ID AS entity_pk_value, 0 AS vote_count, 0 AS vote_value_sum, 0 AS percent, 0 AS percent_approved, @STORE_ID AS store_id) AS T WHERE NOT EXISTS (SELECT * FROM m2rating_option_vote_aggregated WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID));

SELECT `m2rating_option_vote_aggregated`.`primary_id` INTO @PRIMARY_ID FROM `m2rating_option_vote_aggregated` WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID);

SELECT COUNT(vote.vote_id), SUM(vote.value), COUNT(IF(review.status_id=1, vote.vote_id, NULL)), SUM(IF(review.status_id=1, vote.value, 0)) INTO @VC, @VS, @VCA, @VSA FROM `m2rating_option_vote` AS `vote`  INNER JOIN `m2review` AS `review` ON vote.review_id=review.review_id  LEFT JOIN `m2review_store` AS `store` ON vote.review_id=store.review_id  INNER JOIN `m2rating_store` AS `rstore` ON vote.rating_id=rstore.rating_id AND rstore.store_id=store.store_id WHERE (vote.rating_id = @RATE_N) AND (store.store_id=@STORE_ID) AND (vote.entity_pk_value = @PRODUCT_ID) GROUP BY `vote`.`rating_id`,   `vote`.`entity_pk_value`,   `store`.`store_id`;

UPDATE `m2rating_option_vote_aggregated` SET `rating_id` = @RATE_N, `entity_pk_value` = @PRODUCT_ID, `vote_count` = @VC, `vote_value_sum` = @VS, `percent` = @VS*20/@VC, `percent_approved` = @VSA*20/@VCA, `store_id` = @STORE_ID WHERE (primary_id = @PRIMARY_ID);

SET @STORE_ID=2;

INSERT INTO m2rating_option_vote_aggregated (rating_id, entity_pk_value, vote_count, vote_value_sum, percent, percent_approved, store_id) SELECT * FROM (SELECT @RATE_N AS rating_id, @PRODUCT_ID AS entity_pk_value, 0 AS vote_count, 0 AS vote_value_sum, 0 AS percent, 0 AS percent_approved, @STORE_ID AS store_id) AS T WHERE NOT EXISTS (SELECT * FROM m2rating_option_vote_aggregated WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID));

SELECT `m2rating_option_vote_aggregated`.`primary_id` INTO @PRIMARY_ID FROM `m2rating_option_vote_aggregated` WHERE (rating_id = @RATE_N) AND (store_id = @STORE_ID) AND (entity_pk_value = @PRODUCT_ID);

SELECT COUNT(vote.vote_id), SUM(vote.value), COUNT(IF(review.status_id=1, vote.vote_id, NULL)), SUM(IF(review.status_id=1, vote.value, 0)) INTO @VC, @VS, @VCA, @VSA FROM `m2rating_option_vote` AS `vote`  INNER JOIN `m2review` AS `review` ON vote.review_id=review.review_id  LEFT JOIN `m2review_store` AS `store` ON vote.review_id=store.review_id  INNER JOIN `m2rating_store` AS `rstore` ON vote.rating_id=rstore.rating_id AND rstore.store_id=store.store_id WHERE (vote.rating_id = @RATE_N) AND (store.store_id=@STORE_ID) AND (vote.entity_pk_value = @PRODUCT_ID) GROUP BY `vote`.`rating_id`,   `vote`.`entity_pk_value`,   `store`.`store_id`;

UPDATE `m2rating_option_vote_aggregated` SET `rating_id` = @RATE_N, `entity_pk_value` = @PRODUCT_ID, `vote_count` = @VC, `vote_value_sum` = @VS, `percent` = @VS*20/@VC, `percent_approved` = @VSA*20/@VCA, `store_id` = @STORE_ID WHERE (primary_id = @PRIMARY_ID);



-- Average rating

SET @STORE_ID=0;

SELECT (SUM(rating_vote.`percent`)) / COUNT(*) into @PERCENTAGE FROM `m2rating_option_vote` AS `rating_vote`
 INNER JOIN `m2review` AS `review` ON rating_vote.review_id=review.review_id
 LEFT JOIN `m2review_store` AS `review_store` ON rating_vote.review_id=review_store.review_id
 INNER JOIN `m2rating_store` AS `rating_store` ON rating_store.rating_id = rating_vote.rating_id AND rating_store.store_id = review_store.store_id
 INNER JOIN `m2review_status` AS `review_status` ON review.status_id = review_status.status_id WHERE (review_status.status_code = 'Approved') AND (rating_vote.entity_pk_value = @PRODUCT_ID) AND (`review_store`.`store_id`=@STORE_ID) GROUP BY `rating_vote`.`entity_pk_value`;

 SELECT COUNT(*) INTO @COUNT FROM `m2review` WHERE (m2review.entity_pk_value = @PRODUCT_ID) AND (m2review.status_id = 1);

 INSERT INTO m2review_entity_summary (entity_pk_value, entity_type, reviews_count, rating_summary, store_id) SELECT * FROM  (SELECT @PRODUCT_ID AS entity_pk_value, 1 AS entity_type, 0 AS reviews_count, 0 AS rating_summary, @STORE_ID AS store_id) AS T  WHERE NOT EXISTS (SELECT * FROM m2review_entity_summary WHERE (entity_pk_value=@PRODUCT_ID) AND (entity_type=1) AND (store_id=@STORE_ID));

 SELECT `m2review_entity_summary`.primary_id into @PRIMARY FROM `m2review_entity_summary` WHERE (entity_pk_value = @PRODUCT_ID) AND (entity_type = 1) AND (store_id = 0);

 UPDATE `m2review_entity_summary` SET `reviews_count` = @COUNT, `entity_pk_value` = @PRODUCT_ID, `entity_type` = 1, `rating_summary` = @PERCENTAGE, `store_id` = 0 WHERE (m2review_entity_summary.primary_id = @PRIMARY);



SET @STORE_ID = 1;

SELECT ((SUM(rating_vote.`percent`)) / (COUNT(*))) into @PERCENTAGE FROM `m2rating_option_vote` AS `rating_vote`
 INNER JOIN `m2review` AS `review` ON rating_vote.review_id=review.review_id
 LEFT JOIN `m2review_store` AS `review_store` ON rating_vote.review_id=review_store.review_id
 INNER JOIN `m2rating_store` AS `rating_store` ON rating_store.rating_id = rating_vote.rating_id AND rating_store.store_id = review_store.store_id
 INNER JOIN `m2review_status` AS `review_status` ON review.status_id = review_status.status_id WHERE (review_status.status_code = 'Approved') AND (rating_vote.entity_pk_value = @PRODUCT_ID) AND (`review_store`.`store_id`=@STORE_ID) GROUP BY `rating_vote`.`entity_pk_value`;

SELECT COUNT(*) INTO @COUNT FROM `m2review`
 INNER JOIN `m2review_store` AS `store` ON m2review.review_id=store.review_id AND store.store_id = @STORE_ID WHERE (m2review.entity_pk_value = @PRODUCT_ID) AND (m2review.status_id = 1);

 INSERT INTO m2review_entity_summary (entity_pk_value, entity_type, reviews_count, rating_summary, store_id) SELECT * FROM  (SELECT @PRODUCT_ID AS entity_pk_value, 1 AS entity_type, 0 AS reviews_count, 0 AS rating_summary, @STORE_ID AS store_id) AS T  WHERE NOT EXISTS (SELECT * FROM m2review_entity_summary WHERE (entity_pk_value=@PRODUCT_ID) AND (entity_type=1) AND (store_id=@STORE_ID));

 SELECT `m2review_entity_summary`.primary_id into @PRIMARY FROM `m2review_entity_summary` WHERE (entity_pk_value = @PRODUCT_ID) AND (entity_type = 1) AND (store_id = @STORE_ID);

 UPDATE `m2review_entity_summary` SET `reviews_count` = @COUNT, `entity_pk_value` = @PRODUCT_ID, `entity_type` = 1, `rating_summary` = @PERCENTAGE, `store_id` = @STORE_ID WHERE (m2review_entity_summary.primary_id = @PRIMARY);


 SET @STORE_ID = 2;

SELECT (SUM(rating_vote.`percent`)) / COUNT(*) into @PERCENTAGE FROM `m2rating_option_vote` AS `rating_vote`
 INNER JOIN `m2review` AS `review` ON rating_vote.review_id=review.review_id
 LEFT JOIN `m2review_store` AS `review_store` ON rating_vote.review_id=review_store.review_id
 INNER JOIN `m2rating_store` AS `rating_store` ON rating_store.rating_id = rating_vote.rating_id AND rating_store.store_id = review_store.store_id
 INNER JOIN `m2review_status` AS `review_status` ON review.status_id = review_status.status_id WHERE (review_status.status_code = 'Approved') AND (rating_vote.entity_pk_value = @PRODUCT_ID) AND (`review_store`.`store_id`=@STORE_ID) GROUP BY `rating_vote`.`entity_pk_value`;

SELECT COUNT(*) INTO @COUNT FROM `m2review`
 INNER JOIN `m2review_store` AS `store` ON m2review.review_id=store.review_id AND store.store_id = @STORE_ID WHERE (m2review.entity_pk_value = @PRODUCT_ID) AND (m2review.status_id = 1);

 INSERT INTO m2review_entity_summary (entity_pk_value, entity_type, reviews_count, rating_summary, store_id) SELECT * FROM  (SELECT @PRODUCT_ID AS entity_pk_value, 1 AS entity_type, 0 AS reviews_count, 0 AS rating_summary, @STORE_ID AS store_id) AS T  WHERE NOT EXISTS (SELECT * FROM m2review_entity_summary WHERE (entity_pk_value=@PRODUCT_ID) AND (entity_type=1) AND (store_id=@STORE_ID));

 SELECT `m2review_entity_summary`.primary_id into @PRIMARY FROM `m2review_entity_summary` WHERE (entity_pk_value = @PRODUCT_ID) AND (entity_type = 1) AND (store_id = @STORE_ID);

 UPDATE `m2review_entity_summary` SET `reviews_count` = @COUNT, `entity_pk_value` = @PRODUCT_ID, `entity_type` = 1, `rating_summary` = @PERCENTAGE, `store_id` = @STORE_ID WHERE (m2review_entity_summary.primary_id = @PRIMARY);


 END //
 DELIMITER ;

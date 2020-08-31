post approved reviews Magento 2 SQL Query

This creates a stored procedure (saved queries) for posting the reviews.

reference https://blog.magepsycho.com/import-product-reviews-in-magento-via-sql/

General structure is:

1. store data in variables
2. post reviews as approved
3. calculate aggregated rating for each rating ID/store ID and update
4. Calculate summary rating for store ID and update

The queries were captured from Magento 2 while posting a review and marking it as approved.
After which it was generalised.
source: https://github.com/Smile-SA/magento2-module-debug-toolbar

I am currently using 'm2' as prefix for database.

Aggregated rating stores average rating for each rating id and storeID. 

If you are extending ratings, you will have to extend 31 and 33 lines (optionID)

Hope this was useful

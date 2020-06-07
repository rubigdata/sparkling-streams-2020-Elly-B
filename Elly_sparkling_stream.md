
# My streaming expirience

## Elly Bahovska s1001758
### (not to be confused with my game streaming expirience, which is different. Go go Spark!)
My expirience so far with streams was in the the course Object Orienttation, where we had to manually turn our arrays into stream, so that we can do one line commands on them, which was a bit counter intuitive. Using an input and output streams here is very nice, to illustrate what streams actually do. I find the experimental – run the stream for a second and see what you have in the memory very descriptive. Additionally, this assignment makes it all fit together, as sql seems to be made specifically to work with streams. 

Before we get into the assignment, I would like to note how cool it is that the data is DnD gear!

## Parsing the input stream
### a.k.a. Regex stream fun.

In the first assignment we need to do – split the strings of the type 
“Dragon Mace was sold for 390650gp” into columns “material” “tpe” and price” the struct-type class was easily changed into “
case class RuneData(material: String, tpe: String, price: Integer)”. 
We do have a missing import that we will need to add, as it seems this notebook wants the imports to be in the same cell:

*import spark.implicits._*

However, the regex to modify seemed to be confusing – a hit that using x instead of [A-Z][a-z] would be a bet better as the example does not work before the edits – for material and price alone. After a good look up into how to use regex, a solution of 

*val myregex = "\"^([\\\\w].+) ([\\\\w].+) was sold for (\\\\d+)\""*

was reached. That goes finally to getting the sql command updated wth the type:

*val q = f"select regexp_extract(value, $myregex%s, 1) as material, regexp_extract(value, $myregex%s, 2) as tpe, cast(regexp_extract(value, $myregex%s, 3) as Integer) as price from memoryDF"*

to get three columns.

## Stream Processing
### Console output  a.k.a. This is so cool! 
Currently working with multi-threading in OO seeing how the stream does things  in the background  and changes your output real time is very nice to see. With my limited memory, however, I did not  dare to keep it running for  a while. Additionally, this one of the examples for a notebook being so much better than a whole code executing environment. 
### Now to structure it
Here we use the solution from the above section. However, since we are taking the information to structure in the columns from a stream that should be live to use, and currently is now, we cannot show our solution yet – we will see that is correct later, when we start the stream.
q is now: 
*val q = f"select regexp_extract(value, $myregex%s, 1) as material, regexp_extract(value, $myregex%s, 2) as tpe, cast(regexp_extract(value, $myregex%s, 3) as Integer) as price from runeUpdatesDF"*
## Writing to the disk
…but be careful not to fill the disk. It is very easy to save the entire data. 
We get our running streaming query, after we set up writing to the disc:
*val streamWriterDisk = runes
  .writeStream
  .outputMode("append")
  .option("checkpointLocation", "/tmp/checkpoint")
  .trigger(Trigger.ProcessingTime("2 seconds"))*

Now when we start the stream, we see that the streaming quert is: 

*stream2disk: org.apache.spark.sql.streaming.StreamingQuery = org.apache.spark.sql.execution.streaming.StreamingQueryWrapper@4b58406d*

And indeed the active one (because we neatly stopped everything else
**res79: Array[org.apache.spark.sql.streaming.StreamingQuery] = Array(org.apache.spark.sql.execution.streaming.StreamingQueryWrapper@4b58406d).**
We cans see how our disk is doing:
A minute (roughly) after starting it:
When we want to see the size of the /bigdata folder we made to write in: 
**361k /bigdata/_spark_metadata 996k /bigdata**
And two minutes (roughly) later:
**541k /bigdata/_spark_metadata 1.5M /bigdata**

PS: Our timing was wrong – counting the checkpoints, they were 115. And since, we know that a checkpoint (a backup that we can revert to in case of a fault) were made every other second, we can derive that we had the stream running for 230 second – almost 4 minutes!

## Work with the data
Finally, after experimenting how to get the data, we get to work with it.
Now, the examples given there of course, are not completely representative. That is because w only got a part of the stream on our disk. We can only work on the data that we have – we take it from what we wrote. So, we cannot decisively say what is average (even though we can approximate)  and definitely cannot give a max and min price – as we don’t know the entire dataset and we don’t know if there will be a bigger/smaller one.
Thus making analysis here, is better for predictions for example, where it is normal and expected that we don’t know everything – but just go with likelihood.
### Analysis
-	Total count
We can go around that, however. We will define variables for what we want to establish – for example the number of the runes.
var total_runes = 0
Then, when we count the runes sold in this batch:
val counted  = spark.sql("select COUNT(material) FROM runes")
Now, counted is a framework with only one element under column count(material).  Then, the fight  begins, to turn that into the Int, that we can add to total_runes:

*val c:Int = counted.first().get(0).toString.toInt

type c  //c: Int = 21610*

 Now, finally, we can add it to total runes every time we get a new batch:
*total_runes = total_runes + c*
This will keep get updated, as long as we don’t accidentally reset total_runes to 0. Just in case, we will comment that line out.
-	Total gold spent
We will use the same trick to count  the total gold. We have it originally at zero (because this is a huge dataset, the total god quickly adds up beyond Int’s capacity. For now, we will use long, but over all this might not be a good thing to keep track of, as will grow very much) :

*var total_gold: Long = 0*

Doing it for a second time, - now with SUM instead of COUNT, we can afford to show off with a one liner:

*total_gold += spark.sql("select SUM(price) FROM runes").first().get(0).toString.toLong
print(total_gold)*

-	Switch by type
Finally, instead of making a zero-table for all the possible times, we will group  by type (we could also do it by material, but will leave that for future development) and calculate the total number of items of each type. Then, for each of the micro batches, we will add the total-sold column to the ones from the first table.

Let’s get to work, where we can also add a column of the average prices, and even  order by the average prices.

*spark.sql("SELECT tpe, avg(price) AS avg_price, COUNT(price) AS count_price FROM runes GROUP BY tpe ORDER BY avg_price").show()*

This, for the first batch we have, will give us something like (sorry, the tabs of git keep not working, but I think the idea is clear):

+---------+------------------+------------+ 

| tpe| avg_price| count_price|

+---------+------------------+------------+ 

| Dagger| 171105.1667844523| 1415| 

| Sword| 184008.9785254116| 1397|  

|Scimitar|198968.50480109738| 1458| 

| Mace|211697.24916722186| 1501| 

|Longsword|236170.33917734324| 1483| 

| Hatchet|254254.91411042944| 1467| 

|Battleaxe|266129.17226277373| 1370| 

|Warhammer| 281797.3213545266| 1447| 

| sword| 303162.4985895628| 1418| 

| Pickaxe| 312473.6433615819| 1416| 

| Spear| 329128.3283884019| 1483| 

| Hasta|348118.94053662074| 1379| 

| Claws| 372073.9526132404| 1435| 

| Halberd|384711.04972752044| 1468| 

| Defender|389753.16632722336| 1473| 

+---------+------------------+------------+

We see some capitalization problem, that we might want to fix, but we have done that in precious assignments and are not the focus. 

We save this table.

*var mixed = spark.sql("SELECT tpe, avg(price) AS avg_price, COUNT(price) AS count_price FROM runes GROUP BY tpe ORDER BY avg_price")*

Then, when the next batch comes, we do the same to it (note that the first one is changeable var, and the second – static val):

*val mixed2 = spark.sql("SELECT tpe, avg(price) AS avg_price, COUNT(price) AS count_price FROM runes GROUP BY tpe ORDER BY avg_price")*

Now, we want to sum mixed and mixed2 in mixed. For the count(price) we do a plain sum. For average, we want the weighted sum based on the counts in each of the two tables we are summing.
Before that there  is one small problem – mixed and mixed2 are sql products, and we can only apply an sql command on a table. So, we need to make them tables again:

*mixed.createOrReplaceTempView("m");
mixed2.createOrReplaceTempView("mixed2");*

Then we sum tow two tables, as we described above:

*mixed = spark.sql("SELECT A.tpe, A.avg_price*(1/B.count_price) + B.avg_price*(1/A.count_price) AS avg_price, A.count_price + B.count_price AS count_price FROM m A JOIN mixed2 B ON A.tpe = B.tpe")*

And we are done. If we call these functions on every minbatch, we consistently, we will eventually get those statistics for the entire dataset.

## Conclusion
Working with streams is very fun. It is interesting to keep the dynamic updates per mini batches running, too and that is a small price to pay for the ease of work to for each execution to take just a little bit of time, and not get the computer stuck on too much data. I would say that is my favourite framework, so far.

See the whole notebook [Here](https://github.com/rubigdata/sparkling-streams-2020-Elly-B/blob/master/Spark%20Streaming_Elly.zpln)

Have a great day!

Elitsa Bahovska 
s1001758

Recently, we have gotten an influx of tickets coming in that talk about the Question Bank and Quiz Statistics pages in Moodle 4 being slow...or not loading at all due to timing out in the browser.

After over a week of research and trying different approaches, we have come up with a solution that speeds up loading the page significantly!

[This solution has been reported to Moodle HQ](https://tracker.moodle.org/browse/MDL-79392), and as the time of writing, is being looked at by them to see if they can implement it.

***

### Solution

Since upgrading to Moodle 4, the question bank and quiz statistics pages were taking upwards of 40 seconds to load, if they did not timeout. We were also seeing the timeout on the quiz_statistics cron task, so it never finished. We used the cron script for debugging and found a delete query that was taking 2.5-4 seconds per question during the task in question/classes/statistics/responses/analysis_for_question.php:211:
```
$DB->delete_records_select('question_response_count',
            'analysisid IN (
                SELECT id
                  FROM {question_response_analysis}
                 WHERE hashcode= ? AND whichtries = ? AND questionid = ?
                )', [$qubaids->get_hash_code(), $whichtries, $questionid]);
```
We replaced this query with:
```
$DB->execute("
            DELETE qrc
            FROM mdl_question_response_count qrc
            JOIN mdl_question_response_analysis qra
            ON qrc.analysisid = qra.id
            WHERE qra.hashcode = ?
            AND qra.whichtries = ?
            AND qra.questionid = ?",
            [$qubaids->get_hash_code(), $whichtries, $questionid]);
```
and now the query runs in 0.00038 seconds, we were able to get the cron task to complete, and the pages load significantly faster now.

We also added indexes to a few of the impacted tables:
```
create index mdl_quiz_statistics_test_idx on mdl_quiz_statistics (hashcode);
create index mdl_question_response_analysis_test_idx on mdl_question_response_analysis (hashcode, whichtries, questionid);
create index mdl_question_response_count_test_idx on mdl_question_response_count (analysisid);
```
***

### Long Version

We had been seeing since upgrading to Moodle 4 that the question bank and quiz statistic pages both were running very slowly. Eventually, it got to the point where the page would time out before loading, prompting tickets to start to come in from faculty about the issue.

Investigation that led to solving the issue started on 9/11/2023, where we along with our Systems team, followed a few different threads to figure out where things were going on. We figured pretty early on that it was a query that had been changed in Moodle 4 that was taking too long to run, but we also investigated [MDL-78580](https://tracker.moodle.org/browse/MDL-78580), which seemed like it could be related to our issue. After tracing through the logs, though, we didn’t see any traces of deadlocks happening in our question banks, just really slow queries. This made us think that [MDL-78588](https://tracker.moodle.org/browse/MDL-78588) was more similar to our issue and might be something separate.

Following that thread further, we added mtrace lines to files to identify where the slow query was. Since it is related to the question bank, we instinctively looked towards analysis_for_question.php, and managed to find out that the query below around line 211 was taking 2.5-4 seconds per question…taking a very long time:
```
$DB->delete_records_select('question_response_count',
            'analysisid IN (
                SELECT id
                  FROM {question_response_analysis}
                 WHERE hashcode= ? AND whichtries = ? AND questionid = ?
                )', [$qubaids->get_hash_code(), $whichtries, $questionid]);
```
After some further playing around with SQL queries, a member of our Systems team tried to use new indices to speed up the query…
```
create index mdl_quiz_statistics_test_idx on mdl_quiz_statistics (hashcode);
create index mdl_question_response_analysis_test_idx on mdl_question_response_analysis (hashcode, whichtries, questionid);
create index mdl_question_response_count_test_idx on mdl_question_response_count (analysisid);
```
But these alone didn’t work, since Moodle’s delete_records_select function doesn’t expect these when it receives a query. Creating a new query that used JOIN instead of SELECT seemed to make the query run much faster manually, although then we had to figure out a way to run it in Moodle. We decided upon the $DB->exec function instead to run a custom MySQL query:
```
$DB->execute("
            DELETE qrc
            FROM mdl_question_response_count qrc
            JOIN mdl_question_response_analysis qra
            ON qrc.analysisid = qra.id
            WHERE qra.hashcode = ?
            AND qra.whichtries = ?
            AND qra.questionid = ?",
            [$qubaids->get_hash_code(), $whichtries, $questionid]);
```
When we replaced the former query with this new one that relies on these indices, the entire query was able to run in 0.00038 seconds, which was a HUGE improvement. We have switched out this query on our prod environment and been enjoying these massive speedups, and have not gotten a ticket about the pages now being able to load since ;)

Currently, we are working with Moodle to actually get this query implemented into core, but have to come up with equivalent queries for all of the different DBs that Moodle supports:

- PostgreSQL
- MariaDB (done with our query)
- MySQL (done with our query)
- Amazon Aurora MySQL (done with our query)
- MSSQL
- Oracle

Our progress on that effort can be tracked at [MDL-79392](https://tracker.moodle.org/browse/MDL-79392).

Fun fact: There’s this ticket [MDL-29589](https://tracker.moodle.org/browse/MDL-29589) that has been open since 2011 about this kind of slow DELETE…SELECT query…and we just might be the ones to close it ;)

Author - DW ([@derbear444](https://github.com/derbear444))

## Requirements
### Important (MVP)
- user sign up
- user sign in
- follow user
- add post (image/video with caption)
- delete post (image/video with caption)
- view post
- like post
- unlike post
- add comment on post
- delete comment from post
- see timeline/feed
- see other user's profile
- check other user's posts


### Other
- message user
- stories
- image filters
- hashtags
- search user
- user follow recommendations
- push notifications


## Assumptions
- Total users = 1 B
- DAU = 300 M
- Posts added per day = 500 M
- Posts viewed per day = 5 B
- Feed request per day = 1.5 B
- Avg post file size = 500 KB
- avg post metadata size = 10 KB
- Data size per user = 10 KB


## Storage
- Post file storage size per day = 500 M * 500 KB = 250 TB
- Post file storage for 5 years = 250 TB * 365 * 5 = 500 PB (approx)
- Posts metadata size per day = 500M * 1 KB = 5 GB
- Posts metadata size for 5 years = 5 GB * 365 * 5 = 10 TB (approx)
- User data = 1 B * 5 KB = 5 TB

- Since data size is immensely huge, it is impossible to store in a single machine, so we can shard data and store
    - post files(images/videos) in [Hadoop Distributed File System(HDFS)](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html).
    - Other data in DB
        - User (userid, username, first_name, last_name, salted_password_hash, phone_number, email, bio, photo)
        - Like (user_id, post_id)
        - View (user_id, post_id)
        - Comment (comment_id, comment, post_id, user_id, created_at)
        - Posts (post_id, timestamp, user_id, caption, file_url)
        - Follow (user_id, follows)
        - LastLogin (user_id, timestamp)


## Computation
- Post add qps = 500M/86400 = 6K qps, approx
- Post view qps = 5B/86400 = 60K qps, approx
- Feed qps = 1.5B/86400 = 17.5K qps, approx
- Total app servers needed = 450, approx


## Design Tradeoff
This system should be highly reliable and images should never be lost after upload. So we need to have replicas of image servers and databases too. Here we would prefer our system to be always **available** rather than **consistent**.


## Overall data flow
1. User sends an api request
2. Load balancer receives request and redirect it to an app server
3. An app server receieves that request
4. The app server tries to fulfill the request, after input validation and sanitization
5. The app server sends ok response with or without requested data if everything worked fine, otherwise a specific error response


## API
- signup (username, first_name, last_name salted_password_hash, phone_number, email, bio, photo)
    - adds user to user table

- login (username, salted_password_hash)
    - login and update last login time

- search_user (search_string, auth_token)
    - return public user data for given search string (can be searched in user first name, last name and username)

- get_user_by_id(userid, auth_token)
    - return public user data for given user id

- follow_user(user_id, target_user_id, auth_token)
    - Add follow data in DB

- add_post (file, caption, user_id, auth_token)
    - upload file to file storage server
    - add other post data to posts table

- delete_post (user_id, post_id, auth_token)
    - delete given user's given post along with its metadata(use soft delete).

- get_feed (user_id, count, offset, timestamp, auth_token)
    - return top posts after the given timestamp of users followed by given user according to count and offset.

- get_user_posts (user_id, count, offset, auth_token)
    - return posts of given user according to count and offset 

- post_like (user_id, post_id, auth_token)
    - add given post id to given user's likes

- post_unlike (user_id, post_id, auth_token)
    - remove given post id from given user's likes

- add_comment (user_id, post_id, comment)
    - add comment to given user's comment on given post

- delete_comment (user_id, comment_id)
    - delete given user's comment of given comment id


## Feed
### Generation
#### Approach 1
1. Get user_ids of all users followed by given user
2. Get all posts after given timestamp for each user_id
3. Sort these posts on the basis of recency(say).
4. Store top 'K' of these in cache.
5. Return next 'count' number of posts after 'offset' number of posts.

We can improve the time complexity by following steps:
1. We can continuously generate user's feed using separate servers and store them in user feed table. 
2. To get a user's feed we can query this table and return the results.
3. To generate an user's feed, we can query the user feed table to get previously generated feed. Then generate new feed after the previous feed's timestamp.

### Storage
Number of posts in feed = 50
Feed size per user = 50 * post_size = 50 * 1KB = 50KB
Feed size for 1 B users = 50KB * 1 B = 50TB

### Distribute
- Pull:
    - In this, clients requests feed from server regularly or dynamically based on user actions (scroll, load more etc).
    - But It doesn't handle new data very well.
    - Users have to manually refresh to get new data.
    - Also when client regularly requests data, most of the time, if there is no new data response would be empty .
- Push:
    - In this, server can push new posts of a user to all his/her followers.
    - long polling can be used.
    - Problem with this is that when a user has a lot of followers, the server has to send a large number of requests.
- Hybrid
    - We can use 'Pull' method for large followers users and 'Push' for others.



## Cache
- As the users are distributed globally and we have to improve our latency we can host files on CDNs.
- To reduce feeds, post fetch time we can also use cache on app servers for Feeds and posts. LRU can be used as the cache eviction. We can determine the cache size using 80/20 rule which says almost 80% of our traffic comes from 20% of our daily data.
- Posts metadata cache size = 0.2 * 5TB = 1 TB
- Feed cache size = 0.2 * 50TB = 10 TB
- Since cache size is large cache must also be distributed.


## Load Balancing
- We need a load balancer for user requests.
- We can use round robin technique to distribute requests among app servers. But if a server goes down, a request can be sent to it. 
- As a workaround we can use heartbeat mechanism in which each server pings the LB at a certain interval to let LB know that it's not down.
- Since DB and cache servers are also distributed we need load balancers for them. Since they both are user specific, we can use consistent hashing to determine which request should go to which server.
        Main Load Balancer
             |
        App Servers
             | LB
        Cache Servers
     ________|______________
    | LB                   |
DB Servers          Image Storage Servers

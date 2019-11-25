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
- Posts metadata size per day = 500M * 10 KB = 5 TB
- Posts metadata size for 5 years = 5 TB * 365 * 5 = 10 PB (approx)
- User data = 1 B * 10 KB = 10 TB

- Since data size is immensely huge, it is impossible to store in a single machine, so we have to shard data and store
    - post files(images/videos) in [Hadoop Distributed File System(HDFS)](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html).
    - other post data in DB
        - timestamp
        - uploader id
        - likes
        - comments
        - views
        - caption
        - etc
    - User data in different tables (static data in one, dynamic data separately)
        - User (userid,username, first_name, last_name salted_password_hash, phone_number, email, bio, photo)
        - Like (user_id, post_id)
        - View (user_id, post_id)
        - Comment (comment_id, comment, post_id, user_id, created_at)
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

- follow_user(user_id, target_user_id, auth_token)
  - Add follow data in DB

- add_post (file, caption, user_id, auth_token)
  - upload file to file storage server
  - add other post data to posts table

- add_post (user_id, post_id, auth_token)
  - delete given user's given post along with its metadata(use soft delete).

- get_feed (user_id, count, offset, timestamp, auth_token)
  - return top posts after the given timestamp of users followed by given user according to count and offset.

- get_user_posts (user_id, count, offset, auth_token)
  - return posts of given user according to count and offset.

- post_like (user_id, post_id, auth_token)
  - add given post id to given user's likes

- post_unlike (user_id, post_id, auth_token)
  - remove given post id from given user's likes

- post_comment (user_id, post_id, comment)
  - add comment to given user's comment on given post


## Feed
### Generation
1. Get user_ids of all users followed by given user
2. Get all posts after given timestamp for each user_id
3. Sort these posts on the basis of recency.
4. Store top 'K' of these in cache.
5. Return next 'count' number of posts after 'offset' number of posts.

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
- Files can be cached on CDNs.
- LRU Cache on app servers can be used for Feeds with the 80/20 rule.

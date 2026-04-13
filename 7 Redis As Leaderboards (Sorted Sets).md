# 7. Redis As Leaderboards (Sorted Sets)

**Description**: Redis Sorted Sets are perfectly suited for building real-time leaderboards and ranking systems due to their ability to store members with scores and retrieve them by rank or score range.

**Real-World Scenarios**:

- **Gaming**: Global and per-game leaderboards.
- **Social Media**: Ranking trending topics or popular posts.
- **Analytics**: Top N lists (e.g., top 10 most active users).

**Code Examples (Python)**:

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

# Add/update player scores
r.zadd('game:leaderboard', {'player:Alice': 1500, 'player:Bob': 1200, 'player:Charlie': 1800, 'player:David': 1500})

# Get top 3 players
top_players = r.zrevrange('game:leaderboard', 0, 2, withscores=True)
print("Top 3 Players:")
for player, score in top_players:
    print(f"  {player.decode()}: {int(score)}")

# Get rank of a specific player (0-indexed)
charlie_rank = r.zrank('game:leaderboard', 'player:Charlie')
print(f"Charlie's rank: {charlie_rank + 1}") # +1 for 1-indexed rank

# Get players with scores between 1400 and 1600
middle_tier = r.zrangebyscore('game:leaderboard', 1400, 1600, withscores=True)
print("Players in middle tier (1400-1600 scores):")
for player, score in middle_tier:
    print(f"  {player.decode()}: {int(score)}")
```
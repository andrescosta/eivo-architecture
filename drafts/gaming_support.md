Models:

Data:
- Player --- associated to ---> User
- Boards --- associated to --> Eivolet, Player

Pre Game:
  - Rooms Id generation (bar code)

In Game:
- Rooms/Sessions --- associated to --> Eivolet 
  - Submission
  - Board
  - Players
  - Bots


Mechanics:

Game Types:

  

- Arena a lo Kahoot
Host creates room → players join.
Host starts game → broadcast question.
Players send answer.
Server validates → updates scores.
Broadcast leaderboard.
Repeat until finished.

- Offline with leadership board(Competition)
Host creates contest → players join.
Problems broadcast (list of tasks).
Player submits code → backend sends to Judge0 API.
Judge0 returns verdict → backend updates score + leaderboard.
Broadcast leaderboard updates in real-time.
Contest finishes → final standings persisted.

https://socket.io/
https://mdobekidis.medium.com/timer-sync-across-clients-with-supabase-realtime-and-reactjs-6ca3c7730d66
https://github.com/sgoedecke/socket-io-game/blob/master/BLOG.md
https://medium.com/@projectyang/simple-multiplayer-game-with-socket-io-tutorial-part-one-setup-and-movement-ee202024f0ef
https://dev.to/nitdgplug/learn-the-basics-of-socket-io-by-making-a-multiplayer-game-394g
https://medium.com/dailyjs/combining-react-with-socket-io-for-real-time-goodness-d26168429a34
https://github.com/aapatre/Socket.io-Controllable-and-Synced-Counter
https://www.reddit.com/r/github/comments/js4v91/real_time_synced_counter_built_using_socketio/?sort=old

Strategy:


Type of games by Eivolet:
- Solo game with leader board (ideal for programming type of courses)
- Group game 
  - Participants:
    - Friends
    - Bots
  - Type of games:
    - 4 options with score (20 questions)
    - Fill blank
    - Elimination Rounds (short fast paced)
    - Word Unscramble (?)


Steps:
------

Championships:
  1 Select the Eivolet:
    2 Select the lesson
      3 Select the type of competition -> Multiple option,  Fill blank, Short programming 
        4 run


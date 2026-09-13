# Playrooms

## UX

Select the action: 

* Join a room 
* Create a room

### Create a room

Select the Eivolet
Select a lesson or all of them
Select the type of competition -> Multiple option,  Fill blank, Programming 
Enter room name and characteristics (open, etc.)
Select start while waiting (Host lobby)
          
### Join a room
Enter basic data
Enter the ID
Enter the password (optional)
Select Avatar
Select the name
Wait
Play
Display score

## Data    
- material
- type of competition (type field of the spec)
- Room info (code, password(opt), name (opt))
- Participants
- Scores

## UI:
- Material Selection 
- Enter data
- Host waiting/start screen
- Lobby for pariticipants
- Game screens

## API

POST /playrooms (create new playroom in Redis)
GET /playrooms/{id} (returns complete playroom state)
POST /playrooms/{id}/join (validate password)
POST /playrooms/{id}/start
POST /playrooms/{id}/close (timer reaches 0)
POST /playrooms/{id}/cancel (host cancels or everyone quits)
GET /playrooms/{id}/players (all players info)
POST /playrooms/{id}/players/{id}/join {password}
POST /playrooms/{id}/players/{id}/ready
GET /playrooms/{id}/players/{id} (individual player scores)
POST /playrooms/{id}/players/{id}/challenge/correct
POST /playrooms/{id}/players/{id}/challenge/incorrect
DELETE /playrooms/{id}/players/{id}
GET /playrooms/{id}/leaderboard (real-time rankings)

## Entities

* Playroom
  id
  password
  status [created, waiting, in_progress, cancelled, finished]
  host_id
  material
  players:Player[]
  characteristics (not sure which ones yet)
    - duration
    - difficulty
    - host_plays
    - rounds
  - created_at / updated_at

* Player
  id
  user_id
  role (host, participant)
  ready (not sure yet)
  avatar (data type depends on the library to use)
  playroom_name (display name, could be empty)
  status (playing, inerte, out) Player lifecycle: playing-(after n rounds of not participating)-> inerte-(after n rounds of not participating)-> out
  score




# Challenge

## Data
Challenge:
material, params, etc.
Leader board:
challenge_id, user, score

## Api
/create
/get public
/get by code
/post

## UI
List chanlenges with option to create a new one or enter a code
Play

- Create a new one
List eivolets
List llmmaterial
List type of championship
Save with name, code if needed

# Championship's Leaderboard

## Data
Leaderboard:
material, type, user, score


## Api
/post
/get 

## UI
List eivolets
List llmmaterial
List type of championship
Play
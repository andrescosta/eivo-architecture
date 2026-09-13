No, Base session must include the basic concepts: [init, start, end, cancel] and(ids, dates) -> Trial Session [skip exercise, timeout exercise, next exercise]  -> Multiplayer Session [validate with user id] and stats -> Single player session does not add anything just specialization over multi player


Current design:
                Trial Session                                  Learning Session                Challenge Session
                     |
                 Base Trial
                     |
Binary                      Structured 
(correct/incorrect)         (code with attempts 
                             and testing support)
                            
           |                    | 
    Quiz Session           Coding Session

Helpers:
Guardian -> Defines when the session can continue, etc. 
Session Stats-> Maintained by the Session
Exercise Validators->very basic most of the code in the session
Queue: exercises queue with autoloading 


New design:

              Base Session [init, start, end, cancel] and(ids, dates)
                                  |
                  Trial        Learning           Challenge 
                    |
                Multi player
                    |
                Single Player
                    
Providers object:

Guardian -> Defines when the session can continue, etc. 
Session Stats-> Maintained by the Guardian
Exercise Validators->Multiple choice, Coding, Single answer, etc.
Queue: exercises queue with autoloading 

Sessions:

Base Session
- Id
- Status
* Init
* Start
* Cancel
* Finish
* Save 
* Reconstruct
|
Multi Player Trial Session
- Material Id(used for the exercises)
- Guardian with Stats
- Session Parameters
- Exercises Queue
- Current Exercise
- Validator
- Players
* Next exercise
* Skip exercise
* OnSessionTimeOff
* OnExerciseTimeOff
* Validate(playerid, answer)
* Try (playerid, answer)
|
Single Player
- Player id(user id)
* Validate(answer)

Providers:

Guardian
- Session timer
- Exercise timer
- Stats
- Session Parameters
* ExerciseCorrect
* ExerciseError
* CanContinue
* CanSkip
* CanFinish
* OnSessionTimeOff
* OnExerciseTimeOff
* ToDTO

Stats
- Errors
- Corrects
- Skipped
- Time taken

Queue
- queue
* pop
* peek

Session Store
* Save
* Load
     |
Redis Memory

Validators<T,R>
* validate(T):R
                |
OneAnswer   Code   Multiple choice    

Session Factories  
* Create for Gym
* Create for Championship
* Create for Playroom

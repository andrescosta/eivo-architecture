# New features:
- No skip, no next, no auto play (controlled by timer)
- No parent session
- Each player has its own session with the same timer
- The queue is the coordiator:
  - 
    Get next exercise(current exercise ID)
      <LUA name:get_next>
      - If current participant exercise is different from queue current exercise
          Return queue current exercise
        Else 
          Try to get a lock for the queue:
            If not adquired
              If current participant exercise is different from queue current exercise
                Return queue current exercise
              Else
                Return retry
              End If
            If adquired:
              If current participant exercise is different from queue current exercise
                Return queue current exercise
              Else
                If there are elements in the queue
                  Move first element from the queue to queue current exercise 
                  Return queue current exercise
                Else
                  Return error no elements
                End If
              End If
            End If
          End Try
        End If
      </LUA>
    If response was wait, start active waiting (spinning)
      max = 30
      While max-- >0
        Call <LUA name:get_next>
        If LUA get_next returned an exercise different from the current one, 
          {return the exercise, start try feed the queue}
        End If
      End While
    Else
      If response was an exercise different to the current one: 
        {return the exercise, start try feed the queue}
      Else 
        return error
      End If
    End If
  End  
        

Algorithm:

Host: Wait for players
Players: Join and standby (timeout 5 minutes)
Host: clicks on starts:
  - Show a waiting and feed the queue
  - Notify players
Players: gets a server event either with cancel or start
Players and host, on Started:
  Players when starts: 
    - Calls "get the next exercise (flow Get next exercise(current exercise ID))", 
    -   Backend check session
    -   Backend return the exercise, starts queue feeder 
    - Client starts the timer and shows the exercise
  Players when exercise timer gets 0:
    - Calls "get the next exercise (flow Get next exercise(current exercise ID))" with the current answer
    -   Backend check answer and session
    -   Backend return next exercise and status, starts queue feeder 
    - Client restart the timer and shows the new exercise (this could create a de coordination that I can accept but does not change the architrecture)
  Players when end of match timer gets 0:
    - Calls backend to get the leaderboard.
  Players can get a server event informing that the room has closed or a player has left the room

Feed the queue:
  - Check the if the feeder must be trigger:
    - If it must be trigger:
      <LUA>
        Try to Create a feeder lock, 
          If lock adquired, return lock adquired
          If not, return lock not adquired
      </LUA>
      If lock adquired:
        - Calls forge to get exercises
        - Store the exercises using LUA:
          <LUA name: feeder_store>
            Try Lock the queue
              Add the exercises
              Unlock the queue
              Unlock the feeder
            If lock not adquired
              return retry
          </LUA>
            If return is ok, end
            If retry
              Wait a bit and
              max=3
              While max-- >0
                <LUA name: feeder_store>
                If return is ok, end
              End While



Key tenets:
  - There is no central coordination, Redis coordinate the players
  - The players own timers coordinates the call with the backend

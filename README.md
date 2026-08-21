
  **[Live demo](https://bird-spec.github.io/slam-playground/)** — no install, runs in the browser.


  ## What you're looking at

  The robot knows nothing at the start. Everything on screen is inferred:

  - **teal** — cells it has decided are open
  - **orange** — cells it has decided are wall
  - **green** — *frontiers*: open cells that touch unknown space. These are the
    only places worth flying to, because they are the only places that can
    teach it anything.
  - **orange line** — the live sensor beam

  It stops when it stops learning, not when the map is full.

  ## How it works

  ### The map is a belief, not a picture

  Each cell holds a **log-odds** value rather than a boolean. A reading doesn't
  set a cell to "wall" — it *nudges* the cell's score, and the cell only reads as
  wall once the evidence crosses a threshold.

  This matters because range sensors lie. A glossy surface, a bad angle, a stray
  reflection: any of them produce one wrong reading. With booleans, one bad
  reading permanently corrupts a cell. With log-odds it's outvoted by the next
  twenty readings that disagree.

  ### Rays clear space, not just endpoints

  Every beam does two things. It walks the cells between the robot and the hit
  point with Bresenham's line algorithm and pushes each one *toward open*, then
  pushes the cell it hit *toward wall*.

  The subtlety: when the beam hits nothing within range, the endpoint is **not**
  marked as wall. "I can see 2 m and there's nothing there" is evidence of open
  space, not evidence of an obstacle at exactly 2 m. Getting this wrong builds a
  phantom wall in a ring around every position the robot visits.

  ### Choosing where to go: information per metre

  The naive strategy is fly to the nearest frontier. It performs badly — the
  robot nibbles at the closest edge and takes forever to notice the doorway
  across the room.

  Instead each candidate is scored:

  ```
  score = information_gain / (travel_cost + 0.5)
  ```

  where gain is how many unknown cells would come into sensor range, and cost is
  the true path distance from a breadth-first search over cells the robot can
  actually fit through — not straight-line distance, which happily routes
  through walls.

  That ratio makes it willing to cross the whole room for a big payoff, while
  still preferring the cheap win when payoffs are equal. The `+ 0.5` stops
  division blowing up for frontiers that are already underfoot.

  ### Two clearance thresholds

  The robot needs more room to *stop* than to *pass through*. Requiring the
  resting clearance everywhere makes doorways impassable, and the robot maps one
  room and declares the building finished. So pathing tries the generous standoff
  first and only accepts the tighter transit clearance when every heading is
  blocked — which is exactly the doorway case.

  ### It remembers what didn't work

  A purely greedy planner has no memory of failure. It picks the highest-scoring
  frontier, fails to reach it, returns, re-scores, and the same unreachable
  frontier wins again — forever. When a route fails, that cell and its
  neighbours get blacklisted, which is the difference between ~47% and ~58%
  average coverage across randomized rooms.

  ### Knowing when to stop

  Waiting for zero frontiers waits forever: a frontier hard against a wall can
  never be cleared. So the robot watches its own **rate of learning** and stops
  when coverage stops climbing for several consecutive sweeps.

  ## What this is not

  It's called a playground for a reason. **There is no localization** — the robot
  is told its own position. Real SLAM solves mapping and localization at the same
  time, each depending on the other, which is the genuinely hard part and needs a
  particle filter or pose-graph optimization.

  This is the mapping and exploration half, done properly, with the localization
  half assumed. Adding a particle filter is the obvious next step.

  ## Controls

| Control | What it does |
|---|---|
| Sensor beams | 1 to 16. At 1 beam it must spin to see, which is how real single-point rangefinders work. |
| Sensor range | Short range means more trips and a slower, more careful map. |
| Speed | Simulation steps per frame. |
| New room | Generates a fresh floor plan with random dividers, doorways and furniture. |

Try dropping the beam count to 1 and watching it cope.

## Running locally

No build step, no dependencies.

```bash
git clone https://github.com/bird-spec/slam-playground.git
cd slam-playground
```

Open `index.html` in a browser, or serve it:

```bash
python3 -m http.server 8000
```

## Next

- Particle filter so the pose is estimated rather than given
- Click and drag to draw your own walls
- Click to drop the robot somewhere new and watch it re-plan

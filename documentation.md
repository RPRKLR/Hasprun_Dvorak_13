# LRS-Hasprun-Dvorak-13

## Tasks

1. __Map handling.__
    - implementation or tool - map loading - **required**
    - implementation of obstacle inflation - **required**
    - implementation of map transformations, rotations etc. - **if needed**
2. __Map search algorithm. (Path finding)__
    - implementation or tool - path finding/search algorithm - **required**
        - Flood fill, RRT, A*...
3. __Trajectory planning.__
    - implementation or tool - optimization of path - **if needed**
        - minimal requirement - elimination of not necessary points (elimination of sequences of horizontal, vertical, diagonal paths...)          
4. __ROS Drone control node__
    - implementation - trajectory loading - **required**
    - implementation - mission tasks/commands - **required**
    - implementation - position controller - **required** 
5. __Point specification/task/command__
    - implementation - land and takeoff - **required**
    - implementation - point radius (precision of position controller for given point - Hard/Soft) - **required**
    - implementation - yaw control in a specific angle - **required**
6. __Documentation__ 
    - analysis of each used approach - **required**
        - pros and cons
        - explanation of the algorithms or implementations
    - overal solution diagram - **required**
        - data processing paths
        - ROS Drone control diagram

## Solution


1. __Map handling__
    - Map loading
        - For the map handling, we implemented a python script. The python script loads the .pgm file. The points of the map are converted into an int number between 0 and 255. 255 will represent the free space on the map. 0 will represent the occupied space on the map.  
    - Implementation of obstacle inflation
        - For the inflation, we go trough the whole map, if we find an occupied point, then we will check the surrounding points, and change them to ones. We change the new points into ones, so that we are not going into an infinite loop. After changing the nearby points, we change the ones to zeros, so those points will also be occupied points on the map. 

2. ___Map search algorithm__

    For the path planning we decided we will go with the Flood Fill. After inflating the map, we give the flood fill algorithm a starting position in X and Y, and a goal position in X and Y. the Flood fill algorithm finds the shortest path to the goal point.

3. __Trajectory planning__

    To optimize the path, we used the Douglas Peucker algorithm[1]. The Ramer–Douglas–Peucker algorithm, also known as the Douglas–Peucker algorithm and iterative end-point fit algorithm, is an algorithm that decimates a curve composed of line segments to a similar curve with fewer points. It was one of the earliest successful algorithms developed for cartographic generalization.

4. __ROS drone control node__

    - Trajectory loading
        - We loaded the .csv file, and received the tasks, and then based on the drone position, and the task requirements, we called our python script to calculate the trajectory. The python script then creates a .csv file, which contains the filtered path points in 2D space(x,y).
    - Mission task/commands
        - We created separate function for every task, and based on the required task, we just called for the given task function, and waited until the task is done.
    - Position controller
        - For the position controller we used the Euclid distance to checked whether  the drone got into the goal position with the given window tolerance.

5. __Point specification/task/command__

    - Land, takeoff
        - For the land we created a separate function which sends the command trough the land service client. After the sending we wait until the future is done. We also created a check_land function, which makes sure, that the drone landed successfully.
        - For the takeoff, we did the same. Trough the land service client we are sending a service message to take off to a given altitude, and waiting until the future is done, and then the check_land makes sure that we are in the correct altitude.
    - Point radius
        - For the point radius we specified two different tolerances. Soft = 10cm and Hard = 5cm. Based on the required precision, the drone will fly into the given area, with the given precision. As previously mentioned we check the distance between the drone and the goal point with the Euclid distance function.
    - Yaw control in specific angle
        - After the drone reached the goal position, it will rotate in place to the specific angle provided by the task.

6. __Documentation__

### Pros 
- ROS communication makes it easier to start a project, because the communication system is already written.
### Cons
- Confusing at times due to lots of feature.
- Hard to debug.

### Algorithm explanation
- local_pos_cb - Subscriber callback for the drone position.
- state_cb - Subscriber callback to get the drone state.
- mode_check - Checks if the drone is in the correct mode, if not, then spin the node every 0.5s.
- arm - Sends the request to arm the motors on the drone, and waits until the future is done.
- takeoff - Sends a CommandTOL service request, to take off to the given altitude, with the requested yaw value. 
- land - Sends a CommandTOL service request, to land the drone.
- move - Publishes a PositionTarget message, with the requested x,y,z,yaw values. 
- check_altitude - Spins the node until the drone reached the requested altitude.
- check_land - Checks if the drone actually landed. If not landed yet, it will just spin the node until success.
- check_position - Checks that the drone reached given goal position, with the task precision.
- euclid_distance - Calculate the distance between two 3D point.
- chose_map - returns a string, based on the drone altitude. It is required for the trajectory creating, to select the correct map.
- generate_trajectory - Generates the trajectory, based on the start and goal positions, and altitude.
- read_mission_csv - Reads the mission points, and saves it to a vector of TaskPoints. TaskPoints is a custom structure, that holds one task information such as: x,y,z precision, task
- read_points_csv - Reads the .csv file that we gained from the trajectory generation.
- move_trough_points - The trajectory generation can give back multiple simplified points. This function goes trough all the simplified points, until the final goal position is reached.

### Solution diagram
![Solution diagram](solution_diagram.png)


### Startup functions

1. Open terminator with LRS layout. 
2. In 1st terminal launch gazebo: `gazebo <path_to_world>/fei_lrs_gazebo.world`
3. In 2nd terminal launch ArduPilot SITL: 
```
cd ardupilot/ArduCopter
sim_vehicle.py -f gazebo-iris --console -l 48.15084570555732,17.072729745416016,150,0
```
4. Launch mavros `ros2 run mavros mavros_node --ros-args -p fcu_url:=udp://127.0.0.1:14551@14555`
5. Build the template_drone_control package, source it and run the package. The mission tasks will start automatically.
```
colcon build
. install/setup.bash
ros2 run template_drone_control template_drone_control_node
```

## 2. Task

### Updated path planning

In task 1 we were using the given flood fill algorithm for path planning. In the second task we had to decide between RRT or A* path planning, we decided that we are going to use A*.

## Theoretical Background of A* Algorithm

A* (pronounced "A-star") is a computer algorithm that is widely used in pathfinding and graph traversal. The core idea behind A* is that it uses a best-first search and finds the least-cost path from a given initial node to a goal node (assuming one exists). It employs a heuristic approach which significantly speeds up the pathfinding process compared to algorithms that search blindly.

### How A* Works

1. **Heuristic Function**: A* uses a heuristic function `h(n)` to estimate the cost to reach the goal from node `n`. The effectiveness of the algorithm largely depends on the quality of this heuristic.

2. **Cost Function**: It also considers `g(n)`, the cost to reach node `n` from the start node. The total cost function `f(n)` used by A* is a sum of `g(n)` and `h(n)`.

3. **Priority Queue**: A* maintains a priority queue (often implemented as a min-heap) of nodes to be explored, prioritized by their `f(n)` values.

4. **Exploration**: At each step, the node with the lowest `f(n)` is removed from the queue, and its neighbors are examined. For each neighbor, the algorithm calculates `f(n)` and updates the queue accordingly.

5. **Optimality and Completeness**: When a node is expanded, if its `f(n)` value is greater than the `f(n)` of the goal node, it can be discarded. A* is both complete and optimal, provided that the heuristic function `h(n)` is admissible (never overestimates the actual cost) and consistent (monotonically increasing).


## A* vs Flood Fill
### A* Algorithm
#### Advantages
 * Optimal Pathfinding: Efficiently finds the shortest path in complex scenarios.
 * Weighted Graphs: Can handle different terrain costs, useful in realistic pathfinding.
 * Heuristic Function: Allows for faster search by estimating distances.
#### Disadvantages
 * Higher Complexity: More complex to implement than Flood Fill.
 * Memory Usage: Can be memory-intensive, storing multiple paths and nodes.

### Flood Fill Algorithm
#### Advantages
 * Simplicity: Easier to implement and understand.
 * Efficient for Area Coverage: Ideal for filling connected regions, like in image processing.
 * Lower Memory Usage: Typically uses less memory, especially in iterative implementations.
#### Disadvantages
 * Limited to Area Filling: Not suitable for pathfinding.
 * Inefficient in Large, Complex Graphs: Less effective in scenarios requiring path optimization.

https://www.geeksforgeeks.org/a-search-algorithm/

### Circle

We created a python script called: ```circle_move.py```. It is called when a ```circle``` command is called in the during completing tasks. After receiving the circle task, the drone will go forward, and do a circle. The circle is declared that way, that it is got 7 points declared, and the drone will go to all of the points one by one, after completing the full circle, the drone will go back to the original position, where the circle command started, and continues completing task.

### Interrupt

#### Basic information about interrupt

We created a subscriber, which is subscribing to a topic ```/mavros/interrupt```, and the message type is ```std_msgs/msg/Int32```. Until the interrupt command is enabled, the drone will stay in one position, after it receives a continue command, the drone will continue the mission. If it receives a value, which is not declared in the logic (other than 0 and 1) in the terminal we get a message that the interrupt command is incorrect.

Interrupt values: 

 * 0 - Continue
 * 1 - Stop/Interrupt

#### To call interrupt

If you want to call interrupt use the the value: 1 for data, if you want to call continue, then use the value: 0 for data.
```shell
ros2 topic pub /mavros/interrupt std_msgs/msg/Int32 '{data: 1}' -1 
```


[1][Ramer–Douglas–Peucker algorithm](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm)

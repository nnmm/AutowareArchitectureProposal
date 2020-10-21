Porting ROS1 code to ROS2
=======================

## Setting up the environment
Do the following to setup and start ADE environment

1. Setup ade home
```
$ cd ~
$ mkdir ade-home
$ cd ade-home
$ touch .adehome
```

2. Setup AutowareArchitectureProposal
```
$ git clone https://github.com/tier4/AutowareArchitectureProposal
$ cd AutowareArchitectureProposal
$ git checkout ros2
```

3. enter ADE
```
$ cd ~/ade-home/AutowareArchitectureProposal
$ ade start --enter
$ cd AutowareArchitectureProposal
```

All commands that follow are to be entered in ADE. Next step is to fetch the sub-repos:

    cd ~/AutowareArchitectureProposal
    vcs import < autoware.proj.repos

For instance, the `shift_decider` package is in the repository `github.com:tier4/pilot.auto.git`, which is now in the `autoware/pilot.auto` subdirectory.

Now branch off `ros2` inside that subdirectory and delete the `COLCON_IGNORE` file in the package you want to port.


## Important changes
The best source on migrating is the [migration guide](https://index.ros.org/doc/ros2/Contributing/Migration-Guide/). It doesn't mention everything though, so this section lists some areas with important changes.

A good general strategy is to try to implement those changes, then iteratively run `colcon build --packages-up-to <your_package>` and fix the first compiler error.


### Rewriting `package.xml`
The migration guide covers this well. See also [here](https://www.ros.org/reps/rep-0149.html) for a reference of the most recent version of this format.


### Rewriting `CMakeLists.txt`
Pretty straightforward by following the example of the already ported `simple_planning_simulator` package and the [pub-sub tutorial](https://index.ros.org/doc/ros2/Tutorials/Writing-A-Simple-Cpp-Publisher-And-Subscriber/#cpppubsub). Better yet, use `ament_auto` to get terse `CMakeLists.txt` that do not have as much redundancy with `package.xml` as an explicit `CMakeLists.txt`. See [this commit](https://github.com/tier4/Pilot.Auto/pull/7/commits/ef382a9b430fd69cb0a0f7ca57016d66ed7ef29d) for an example.


### Replacing `std_msgs`
In ROS2, you should define semantically meaningful wrappers around primitive (number) types. They are deprecated in Foxy.


### Changing the namespaces and header files for generated message types
If you follow the migration guide and change the included headers to have an extra `/msg` in the path and convert to `snake_case`, you might get a cryptic error. Turns out _two_ files are being generated: One for C types (`.h` headers) and and one for CPP types (`.hpp` headers). So don't forget to change `.h` to `.hpp` too. Also, don't forget to insert an additional `::msg` between the package namespace and the class name.

A tip: Sublime Text has a handy "Case Conversion" package for converting to snake case.


### Adapting message definitions
If your message included something like `Header header`, it needs to be changed to `std_msgs/Header header`. Otherwise you'll see messages like `fatal error: autoware_api_msgs/msg/detail/header__struct.hpp: No such file or directory`.


### Inheriting from Node instead of NodeHandle members
That's where differences start to show – I decided to make the `VehicleCmdGate` a `Node` even though the filename would suggest that `vehicle_cmd_gate_node.cpp` would be it. That's because it has publishers, subscribers, and logging. It previously had _two_ NodeHandles, a public one and a private one (`"~"`). The public one was unused and could be removed. Private nodes are not supported in ROS2, so I simply made it public, but that is an area that needs to be brought up in review.


### Latched topics
For each latched publisher, I just used `transient_local` QoS on the publisher, and we will need to do the same on all subscribers to that topic.


### Timing issues
First, if the timer can be replaced with a data-driven pattern, it is the preferred alternative for the long term:

#### The Problem with Timer-Driven Patterns
It is well understood that a polling or timer-driven pattern increases jitter (i.e. variance of latency). (Consider, for example: if every data processing node in a chain operates on a timer what is the best and worst case latency?) As a consequence for more timing-sensitive applications, it is generally not preferred to use a timer-driven pattern.

On top of this, it is also reasonably well known that [use of the clock is nondeterministic](https://martinfowler.com/articles/nonDeterminism.html) and internally this has been a large source of frustration with bad, or timing sensitive tests. Such tests typically require specific timing and/or implicitly require a certain execution order (loosely enforced by timing assumptions rather than explicitly via the code).

As a whole, introducing the clock explicitly (or implicitly via timers) is problematic because it introduces additional state, and thus assumptions on the requirements for the operation of the component. Consider also leap seconds and how that might ruin the operation and/or assumptions needed for the proper operation of the component.

#### Preferred Patterns
In general, a data-driven pattern should be preferred to a timer-driven pattern. One reasonable exception to this guideline is the state estimator/filter at the end of localization. A timer-driven pattern in this context is useful to provide smooth behavior and promote looser coupling between the planning stack and the remainder of the stack.

The core idea behind a data-driven pattern is that as soon as data arrives, it should be appropriately processed. Furthermore, the system clock (or any other source of time) should not be used to manipulate data or the timestamps. This pattern is valuable since it implicitly cuts down on hidden state (being the clock), and thus simplifies assumptions needed for the node to work.

For examples of this kind of pattern, see the lidar object detection stack in Autoware.Auto. By not using any mention of the clock save for in the drivers, the stack can run equivalently on bag data, simulation data, or live data. A similar pattern with multiple inputs can be seen in the MPC implementation both internally and externally.

#### Replicating `ros::Timer`
Assuming you still want to replicate the existing `ros::Timer` functionality: There is `rclcpp::WallTimer`, which has a similar interface, but it's not equivalent. The wall timer uses a wall clock (`RCL_STEADY_TIME` clock), i.e. it doesn't listen to the `/clock` topic populated by simulation time. That the timer doesn't stop when simulation time stops, and doesn't go faster/slower when simulation time goes faster or slower.

By contrast, the `GenericTimer` provides an interface to supply a clock, but there is no convenient function for setting up such a timer, comparable to `Node::create_wall_timer`. For now, this works:

    auto timer_callback = std::bind(&VehicleCmdGate::onTimer, this);
    auto period = std::chrono::duration_cast<std::chrono::nanoseconds>(
      std::chrono::duration<double>(update_period_));
    timer_ = std::make_shared<rclcpp::GenericTimer<decltype(timer_callback)>>(
      this->get_clock(), period, std::move(timer_callback),
      this->get_node_base_interface()->get_context());
    this->get_node_timers_interface()->add_timer(timer_, nullptr);

Also, this doesn't work, even with a subsequent `add_timer()` call:

    timer_ = rclcpp::create_timer(this, this->get_clock(), period, timer_callback);


#### Rosbag recording
Unfortunately, one additional problem remains. `ros2 bag` does not record `/clock` (aka sim time) whereas `rosbag` does. This implies that in order to get the same behavior in ROS 2, either:

    * `rosbag` along with the `ros1_brdge` must be used
    * Some explicit time source must be used and explicitly recorded by `ros2 bag`


### Parameters
It's not strictly necessary, but you probably want to make sure the filename is `xyz.param.yaml`. Then come two steps:


#### Adjust code
    double vel_lim;
    pnh_.param<double>("vel_lim", vel_lim, 25.0);

becomes

    const double vel_lim = declare_parameter("vel_lim", 25.0);

which is equivalent to

    const double vel_lim = declare_parameter<double>("vel_lim", 25.0);


#### Adjust param file
Two levels of hierarchy need to be added around the parameters themselves:

    <node name or /**>:
      ros__parameters:
        <params>

Also, ROS1 didn't have a problem when you specify an integer, e.g. `28` for a `double` parameter, but ROS2 does:

    [vehicle_cmd_gate-1] terminate called after throwing an instance of 'rclcpp::exceptions::InvalidParameterTypeException'
    [vehicle_cmd_gate-1]   what():  parameter 'vel_lim' has invalid type: expected [double] got [integer]

Best to just change `28` to `28.0` in the param file. See also [this issue](https://github.com/ros2/rclcpp/issues/979).


### Launch file
There is a [migration guide](https://index.ros.org/doc/ros2/Tutorials/Launch-files-migration-guide/). One thing it doesn't mention is that the `.launch` file also needs to be renamed to `.launch.xml`.


### Replacing `tf2_ros::Buffer`
A `tf2_ros::Buffer` member that is filled by a `tf2_ros::TransformListener` can become a `tf2::BufferCore` in most cases. This reduces porting effort, since the a `tf2::BufferCore` can be constructed like a ROS1 `tf2_ros::Buffer`. For an example, see [this PR](https://github.com/tier4/Pilot.Auto/pull/11).

However, in some cases the extra functionality of `tf2_ros::Buffer` is needed. For instance, waiting for a transform to arrive.


#### Waiting for a transform to arrive
You might expect to be able to use `tf2_ros::Buffer::lookupTransform()` with a timeout out of the box, but this will throw an error:

    Do not call canTransform or lookupTransform with a timeout unless you are using another thread for populating data. 
    Without a dedicated thread it will always timeout. 
    If you have a seperate thread servicing tf messages, call setUsingDedicatedThread(true) on your Buffer instance.

You could do therefore try setting up a dedicated thread, but you could also use the `waitForTransform()` function like this:

    // In the node definition
    tf2_ros::Buffer tf_buffer_;
    tf2_ros::TransformListener tf_listener_;
    ...
    // In the constructor
    auto cti = std::make_shared<tf2_ros::CreateTimerROS>(this->get_node_base_interface(), this->get_node_timers_interface());
    tf_buffer_.setCreateTimerInterface(cti);
    ...
    // In the function processing data
    tf_buffer_.waitForTransform(a, b, msg_time, std::chrono::milliseconds(1000), [this](const std::shared_future<geometry_msgs::msg::TransformStamped>& tf){
        // Here the future is available
    });

The callback will always be called, but only after some time: when the transform becomes available or when the timeout is reached. In the latter case, if the transform is not ready yet, calling `.get()` on the future will throw a `tf2::TimeoutException`.

The `waitForTransform()` function will return immediately and also return a future. However, calling `.get()` or `.wait()` on that future does not respect the timeout. That is, it will wait however long it takes until a transform arrives and never throw an exception.


### Service clients
There is no synchronous API for service calls, and the futures API can not be used from inside a node, only the callback API.

The futures API is what is used in tutorials such as [Writing a simple service and client](https://index.ros.org/doc/ros2/Tutorials/Writing-A-Simple-Cpp-Service-And-Client/#write-the-client-node), but note that the call to `rclcpp::spin_until_future_complete()` does not happen from inside any subscriber callback or similar. If you do call it from inside a node, you will get

    terminate called after throwing an instance of 'std::runtime_error'
      what():  Node has already been added to an executor.

The node itself is already added to the executor in `rclcpp::spin()` function inside the main function, and `rclcpp::spin_until_future_complete()` tries to add the node to another executor.

You might note that the function already returned a `std::shared_future` on which you could wait. But that just hangs forever.

So you're left with using a [callback](http://docs.ros2.org/foxy/api/rclcpp/classrclcpp_1_1Client.html#a62e48edd618bcb73538bfdc3ee3d5e63). Unfortunately that leaves you with no option to handle failure: For instance, if the service dies, your callback will never get called. There's no easy way to say that you only want to wait for 2 seconds for a result.

Another idea for a workaround is to do something similar to what is done in the `rclcpp::spin_until_future_complete()` function by ourselves. Another possible avenue is using multithreaded executors, see [this post](https://answers.ros.org/question/343279/ros2-how-to-implement-a-sync-service-client-in-a-node/) for some more detail.


### Logging
The node name is now automatically prepended to the log message, so that part can be removed.


### Shutting down a subscriber
The `shutdown()` method doesn't exist anymore, but you can just throw away the subscriber with `this->subscription_ = nullptr;` or similar, for instance inside the subscription callback. Curiously, this works even though the `subscription_` member variable is not the sole owner – the `use_count` is 3 in the `minimal_subscriber` example.


## Alternative: Semi-automated porting with ros2-migration-tools (not working yet)
Following the instructions at https://github.com/awslabs/ros2-migration-tools:

    pip3 install parse_cmake
    git clone https://github.com/awslabs/ros2-migration-tools.git
    wget https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04.tar.xz
    tar xaf clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04.tar.xz
    cp -r clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04/lib/libclang.so ros2-migration-tools/clang/
    cp -r clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04/lib/libclang.so.10 ros2-migration-tools/clang/
    cp -r clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04/include/ ros2-migration-tools/clang/clang

The package needs to be **built with ROS1**. I followed http://wiki.ros.org/noetic/Installation/Ubuntu outside of ADE

    sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
    sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
    sudo apt update
    sudo apt install ros-melodic-ros-base

In https://github.com/awslabs/ros2-migration-tools#setup-the-ros1-packages, it instructs me to compile a ros1 package with colcon to get started.

    colcon build --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

And that fails. I thought `colcon` is only for ROS2 so it should only work after porting, not before. I retried with `catkin_make` but also ran into issues there
```
frederik.beaujean@frederik-beaujean-01:~/ade-home/AutowareArchitectureProposal$ catkin_make --source src/autoware/autoware.iv/control/shift_decider/ -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
Base path: /home/frederik.beaujean/ade-home/AutowareArchitectureProposal
Source space: /home/frederik.beaujean/ade-home/AutowareArchitectureProposal/src/autoware/autoware.iv/control/shift_decider
Build space: /home/frederik.beaujean/ade-home/AutowareArchitectureProposal/build
Devel space: /home/frederik.beaujean/ade-home/AutowareArchitectureProposal/devel
Install space: /home/frederik.beaujean/ade-home/AutowareArchitectureProposal/install
####
#### Running command: "cmake /home/frederik.beaujean/ade-home/AutowareArchitectureProposal/src/autoware/autoware.iv/control/shift_decider -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCATKIN_DEVEL_PREFIX=/home/frederik.beaujean/ade-home/AutowareArchitectureProposal/devel -DCMAKE_INSTALL_PREFIX=/home/frederik.beaujean/ade-home/AutowareArchitectureProposal/install -G Unix Makefiles" in "/home/frederik.beaujean/ade-home/AutowareArchitectureProposal/build"
####
CMake Error: The source "/home/frederik.beaujean/ade-home/AutowareArchitectureProposal/src/autoware/autoware.iv/control/shift_decider/CMakeLists.txt" does not match the source "/home/frederik.beaujean/ade-home/AutowareArchitectureProposal/src/CMakeLists.txt" used to generate cache.  Re-run cmake with a different source directory.
Invoking "cmake" failed
```
According to an expert:
>  You should be able to compile it with colcon, cause it works for both ROS 1 and ROS 2 code. You are getting the same error with catkin so it's probably something related to ROS 1 and the build instructions.


## Strange errors and their causes
Some error messages are so unhelpful that it might help to collect them and their causes.


### package.xml errors
If you forget `<build_type>ament_cmake</build_type>`, or you use package format 2 in combination with a package format 3 tag like `<member_of_group>`, you'll get the unhelpful error

    CMake Error at /usr/share/cmake-3.16/Modules/FindPackageHandleStandardArgs.cmake:146 (message):
      Could NOT find FastRTPS (missing: FastRTPS_INCLUDE_DIR FastRTPS_LIBRARIES)


### YAML param file
Used tabs instead of spaces in your param.yaml file? _Clearly_, the most user-friendly error message is

    $ ros2 launch mypackage mypackage.launch.xml
    [INFO] [launch]: All log files can be found below /home/user/.ros/log/2020-10-19-19-09-13-676799-t4-30425
    [INFO] [launch]: Default logging verbosity is set to INFO
    [INFO] [mypackage-1]: process started with pid [30427]
    [mypackage-1] [ERROR] [1603127353.755503075] [rcl]: Failed to parse global arguments
    [mypackage-1]
    [mypackage-1] >>> [rcutils|error_handling.c:108] rcutils_set_error_state()
    [mypackage-1] This error state is being overwritten:
    [mypackage-1]
    [mypackage-1]   'Couldn't parse params file: '--params-file /home/user/workspace/install/mypackage/share/mypackage/param/myparameters.yaml'. Error: Error parsing a event near line 1, at /tmp/binarydeb/ros-foxy-rcl-yaml-param-parser-1.1.8/src/parse.c:599, at /tmp/binarydeb/ros-foxy-rcl-1.1.8/src/rcl/arguments.c:391'
    [mypackage-1]
    [mypackage-1] with this new error message:
    [mypackage-1]
    [mypackage-1]   'context is zero-initialized, at /tmp/binarydeb/ros-foxy-rcl-1.1.8/src/rcl/context.c:51'
    [mypackage-1]
    [mypackage-1] rcutils_reset_error() should be called after error handling to avoid this.
    [mypackage-1] <<<
    [mypackage-1] [ERROR] [1603127353.755523149] [rclcpp]: failed to finalize context: context is zero-initialized, at /tmp/binarydeb/ros-foxy-rcl-1.1.8/src/rcl/context.c:51
    [mypackage-1] terminate called after throwing an instance of 'rclcpp::exceptions::RCLInvalidROSArgsError'
    [mypackage-1]   what():  failed to initialize rcl: error not set
    [ERROR] [mypackage-1]: process has died [pid 30427, exit code -6, cmd '/home/user/workspace/install/mypackage/lib/mypackage/mypackage --ros-args -r __node:=mypackage --params-file /home/user/workspace/install/mypackage/share/mypackage/param/myparameters.yaml'].

and that is indeed what ROS2 will tell you.

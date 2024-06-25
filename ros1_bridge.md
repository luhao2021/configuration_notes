This is to show the steps to compile and run ros1_bridge with customized message communication.

1. Compile ROS1 Noetic
   
   ```$ mkdir -p ~/Workspace/ROS/ros1_noetic_ws/src```
   
   follow https://wiki.ros.org/noetic/Installation/Source to download the source
   
   ```
   $ cd ~/Workspace/ROS/ros1_noetic_ws
   $ vcs import --input noetic-desktop.rosinstall ./src
   $ rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic -y
   $ ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release
   ```
   
   [Issues when build ros1-noetic on Ubuntu 22.04]
   
   1> POCO library not found
      sudo apt install libpoco-dev
   
   2> Log4cxx library not found and not matched.
      sudo apt install liblog4cxx-dev
      The above command install the liblog4cxx 0.12.0 version on Ubuntu 22.04 by default. However, the ros1-noetic rosconsole package requires the 0.11.0 version and does not compatible with the 0.12.0 version.
      There will be compilation error during the build of ros1. The solution can either be downgrade liblog4cxx to 0.11.0 version or add fixes to rosconsole to make it compatible with 0.12.0 version.
      I choose the later one, and use the patch from: https://github.com/ros/rosconsole/pull/51/commits/c9d161a6d946590bf808eb6006430c8c8d8b5cd6.
   
   3> /usr/include/log4cxx/boost-std-configuration.h:10:18: error: ‘shared_mutex’ in namespace ‘std’ does not name a type
   
        10 |     typedef std::shared_mutex shared_mutex;
   
      modify /usr/include/log4cxx/boost-std-configuration.h, from this:
   
        #define STD_SHARED_MUTEX_FOUND 1
        #define Boost_SHARED_MUTEX_FOUND 0
   
      to this:
   
        #define STD_SHARED_MUTEX_FOUND 0
        #define Boost_SHARED_MUTEX_FOUND 1
   
   4> Could not find a package configuration file provided by "urdfdom_headers"
   
        sudo apt install liburdfdom-dev
        sudo apt install liburdfdom-headers-dev
   
   5> Make Error: The following variables are used in this project, but they are set to NOTFOUND.
      Please set them or make sure they are set and tested correctly in the CMake files:
   
        BZ2_LIBRARY
        GPGME_LIBRARY
   
      sudo apt install libgpgme-dev libgz2-dev

2. Compile ROS2
   $ mkdir -p ~/Workspace/ROS/ros2_humble_ws/src
   
   follow https://docs.ros.org/en/humble/Installation/Alternatives/Ubuntu-Development-Setup.html for the detailed instructions.$ mkdir
   
   ```
   $ cd ~/Workspace/ROS/ros2_humble_ws/
   $ vcs import --input https://raw.githubusercontent.com/ros2/ros2/humble/ros2.repos src
   $ sudo rosdep init
   $ rosdep update
   $ rosdep install --from-paths src --ignore-src -y --skip-keys "fastcdr rti-connext-dds-6.0.1 urdfdom_headers"
   $ colcon build --symlink-install
   ```

3. Compile ros1_bridge
   
   ```
   $ mkdir -p ~/Workspace/ROS/ros_bridge_ws/src
   
   # follow https://github.com/ros2/ros1_bridge
   
   $ cd ~/Workspace/ROS/ros_bridge_ws/src
   $ git clone https://github.com/ros2/ros1_bridge.git
   $ cd ~/Workspace/ROS/ros_bridge_ws/
   $ export ROS1_INSTALL_PATH=/home/hao/Workspace/ROS/ros1_noetic_ws/install_isolated
   $ export ROS2_INSTALL_PATH=/home/hao/Workspace/ROS/ros2_humble_ws/install
   $ colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure
   ```

4. Implement ROS2 message
   
   ```
   $ mkdir -p ~/Workspace/ROS/ros2_ws/src
   
   # follow https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Custom-ROS2-Interfaces.html to create msg files
   
   $ ros2 pkg create --build-type ament_cmake --license Apache-2.0 ros2_bridge_msgs
   $ cd ~/Workspace/ROS/ros2_ws/src/ros2_bridge_msgs
   $ mkdir msg
   $ echo "float64 position" > msg/JointCommand.msg
   $ touch ~/Workspace/ROS/ros2_ws/src/ros2_bridge_msgs/my_mapping_rules.yaml
   ```
   
   [my_mapping_rules.yaml]
   -
   
   ```
   - 
    ros1_package_name: 'ros1_bridge_msgs'
    ros1_message_name: 'JointCommand'
    ros2_package_name: 'ros2_bridge_msgs'
    ros2_message_name: 'JointCommand'
   ```
   
   [talker.cpp]
   
   ```
      #include <chrono>
      #include <memory>
   
      #include "rclcpp/rclcpp.hpp"
      #include "ros2_bridge_msgs/msg/joint_command.hpp"
   
      using namespace std::chrono_literals;
   
      class MinimalPublisher : public rclcpp::Node
      {
      public:
   
        MinimalPublisher() : Node("minimal_publisher"), count_(0)
        {
            publisher_ = this->create_publisher<ros2_bridge_msgs::msg::JointCommand>("position", 3.1415988);  // CHANGE
            timer_ = this->create_wall_timer(500ms, std::bind(&MinimalPublisher::timer_callback, this));
        }
   
      private:
   
        void timer_callback()
        {
            auto message = ros2_bridge_msgs::msg::JointCommand();                                   // CHANGE
            message.position = 3.1415926 * this->count_++;                                                     // CHANGE
            RCLCPP_INFO_STREAM(this->get_logger(), "Publishing: '" << message.position << "'");    // CHANGE
            publisher_->publish(message);
        }
        rclcpp::TimerBase::SharedPtr timer_;
        rclcpp::Publisher<ros2_bridge_msgs::msg::JointCommand>::SharedPtr publisher_;             // CHANGE
        size_t count_;
   
      };
   
      int main(int argc, char * argv[])
      {
   
        rclcpp::init(argc, argv);
        rclcpp::spin(std::make_shared<MinimalPublisher>());
        rclcpp::shutdown();
        return 0;
   
      }
   ```
   
   [CMakeLists.txt]

       find_package(rosidl_default_generators REQUIRED)
    
       rosidl_generate_interfaces(${PROJECT_NAME} 
        "msg/JointCommand.msg"
       )
       add_executable(talker src/talker.cpp)
        ament_target_dependencies(talker rclcpp) # CHANGE
        rosidl_target_interfaces(talker ${PROJECT_NAME} "rosidl_typesupport_cpp")
        target_include_directories(talker PUBLIC
         $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
         $<INSTALL_INTERFACE:include>)
         target_compile_features(talker PUBLIC c_std_99 cxx_std_17) # Require C99 and C++17
    
       install(TARGETS talker
         DESTINATION lib/${PROJECT_NAME})
       install(  
         FILES my_mapping_rules.yaml
         DESTINATION share/${PROJECT_NAME})

   [package.xml]

        <buildtool_depend>rosidl_default_generators</buildtool_depend>
         <exec_depend>rosidl_default_runtime</exec_depend>
         <member_of_group>rosidl_interface_packages</member_of_group>
         <export>
             <build_type>ament_cmake</build_type>
             <ros1_bridge mapping_rules="my_mapping_rules.yaml"/>
         </export>

```
$ cd ~/Workspace/ROS/ros2_ws
$ colcon build
```

5. Implement ROS1 message
   
   ```
   $ mkdir -p ~/Workspace/ROS/ros1_ws/src
   $ follow https://wiki.ros.org/ROS/Tutorials/CreatingMsgAndSrv for more details
   $ catkin_create_pkg ros1_bridge_msgs std_msgs rospy roscpp
   $ roscd ros1_bridge_msgs
   $ mkdir msg
   $ echo "float64 position" > msg/JointCommand.msg
   ```
   
   [package.xml]
   
   ```
     <build_depend>message_generation</build_depend>
     <exec_depend>message_runtime</exec_depend>
   ```
   
   [CMakeLists.txt]
   
   ```
      find_package(catkin REQUIRED COMPONENTS
   
        roscpp
        rospy
        std_msgs
        message_generation
   
      )
   
      catkin_package(
   
        ...
        CATKIN_DEPENDS message_runtime ...
        ...)
   
      add_message_files(
   
        FILES
        JointCommand.msg
   
      )
   
      generate_messages(
   
        DEPENDENCIES
        std_msgs
   
      )
   ```
   
   ```
   $ catkin_make
   $ rosmsg info ros1_bridge_msgs/JointCommand
   ```

6. Re-compile the ros1_bridge to include the mapping of JointCommand message we just created
   
   ```
   $ source $ROS1_INSTALL_PATH/setup.bash
   $ source ../ros1_ws/devel/setup.bash
   $ source $ROS2_INSTALL_PATH/setup.bash
   $ source ../ros2_ws/install/setup.bash
   $ colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure
   $ source install/setup.bash
   $ ros2 run ros1_bridge dynamic_bridge --print-pairs | grep ros1_bridge_msgs
      \- 'ros2_bridge_msgs/msg/JointCommand' (ROS 2) <=> 'ros1_bridge_msgs/JointCommand' (ROS 1)
   $ ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
   ```
   
   # shell 1 (ROS1)
   
   ```
   $ source $ROS1_INSTALL_PATH/setup.bash
   $ roscore
   ```
   
   # shell 2 (ROS2)
   
   ```
   $ source $ROS2_INSTALL_PATH/setup.bash
   $ ros2 topic pub /joint_command ros2_bridge_msgs/JointCommand "{position: 0.123}"
   $ ros2 run ros2_bridge_msgs talker
   ```
   
   # shell 3 (ROS1 + ROS2)
   
   ```
   $ source $ROS1_INSTALL_PATH/setup.bash
   $ source ../ros1_ws/devel/setup.bash
   $ source $ROS2_INSTALL_PATH/setup.bash
   $ source ../ros2_ws/install/setup.bash
   $ source install/setup.bash
   $ ros2 run ros1_bridge dynamic_bridge --print-pairs | grep ros1_bridge_msgs
      \- 'ros2_bridge_msgs/msg/JointCommand' (ROS 2) <=> 'ros1_bridge_msgs/JointCommand' (ROS 1)
   $ ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
   ```
   
   # shell 4 (ROS1)
   
   ```
   $ rostopic list
   $ rostopic echo /joint_command
   $ rostopic echo /position
   ```

7. This Youtube video gives a very good demonstration of the demo:
    https://www.youtube.com/watch?v=vBlUFIOHEIo
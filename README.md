# Software Pipeline for Jetson TX2

---

<br/>

### Section: Camera

<br/>

#### GStreamer

We can run a preliminary test to see if we can get data from the camera via gstreamer:

```bash
gst-launch-1.0 nvcamerasrc ! 'video/x-raw(memory:NVMM), width=640, height=480, framerate=30/1, format=NV12' ! nvvidconv ! nvegltransform ! nveglglessink
```

Sometimes the camera resource may be locked, (giving an error that the camera failed to pause), in which case we can simply reboot the TX2.  If we change external cameras later on, we may need to adjust the gstreamer pipeline.

<br/>

#### Publish to ROS Topic

The first step is to send the camera data from the external camera to a ROS node.  This is accomplished through the [gscam package](http://wiki.ros.org/gscam).  Install this package from the [Github source](https://github.com/ros-drivers/gscam) by doing:

```bash
cd catkin_workspace
cd src
git clone https://github.com/ros-drivers/gscam
cd ..
catkin_make
```

We should see the package `gscam` mentioned in the build output.  Now we are ready to launch gscam.

Assuming we already have `roscore` running in another terminal, we run gscam in a new terminal as follows:

```bash
cd catkin_workspace
source devel/setup.bash
roscd gscam
cd bin # Make this directory if it doesn't exist
export GSCAM_CONFIG="nvcamerasrc ! video/x-raw(memory:NVMM),width=1280, height=720,format=I420, framerate=30/1 ! nvvidconv ! video/x-raw, format=BGRx ! videoconvert ! ffmpegcolorspace"
rosrun gscam gscam
```

Now we can verify that the data is being published with RViz (run `rviz &` from any terminal), and click Add -> By topic -> /camera/image_raw -> Image (nb: Image not Camera, because we are recieving an "Image" data type, "Camera" may be the camera metadata type or somethinge else).  We should now see the live camera feed in the bottom left corner of the RViz window.

<br/>

#### Work with published image in OpenCV

Next, if we want to connect this topic to any image processing software that uses OpenCV to read images, we can use [cv_bridge](http://wiki.ros.org/cv_bridge/Tutorials/).  To install,

```bash
sudo apt-get install ros-kinetic-cv-bridge
```

To setup a python subscriber to the ROS node we created above, we follow the [cv_bridge python tutorial](http://wiki.ros.org/cv_bridge/Tutorials/ConvertingBetweenROSImagesAndOpenCVImagesPython) with a couple of modifications.

To create the package, we use `catkin_create_pkg <package_name> [depend1] [depend2] [depend3]` instead of `roscreate-pkg`.  After running this, we need to edit `CMakeLists.txt` so that the `find_package` line looks like (changed `opencv2` to `OpenCV`):
```
find_package(catkin REQUIRED COMPONENTS
  cv_bridge
  rospy
  OpenCV
  sensor_msgs
  std_msgs
)
```

Finally, inside the python source we make two changes. First, adjust the `roslib.load_manifest('package')` line to contain our package name, e.g. `roslib.load_manifest('imgpass')`.  Second, update the `self.image_sub` declaration in `__init__` to
```
self.image_sub = rospy.Subscriber("/camera/image_raw",Image,self.callback)
```

Now we can build and run the python script, e.g. if our package was named `imgpass` and our python source was named `listener.py`, we run at the terminal (after sourcing `/devel/setup.bash`):
```bash
roscd imgpass
mkdir -p build
cd build
cmake ..
rosrun imgpass listener.py
```

If we get the CMake warning about OpenCV:

```
  Could not find a package configuration file provided by "OpenCV" with any
  of the following names:

    OpenCVConfig.cmake
    opencv-config.cmake

  Add the installation prefix of "OpenCV" to CMAKE_PREFIX_PATH or set
  "OpenCV_DIR" to a directory containing one of the above files.  If "OpenCV"
  provides a separate development package or SDK, be sure it has been
  installed.
```
The solution is to first find this `OpenCVConfig.cmake` file by running:

```bash
sudo find / -name OpenCVConfig.cmake
```

This should give something like:
```
/usr/share/OpenCV/OpenCVConfig.cmake
/usr/local/share/OpenCV/OpenCVConfig.cmake
```

Supposing we got the first one above, we would add to `CMakeLists.txt` the following (*before* the find_package line):
```
set(OpenCV_DIR /usr/share/OpenCV)
```




<br/>

#### Image Processing and Inference

I have tested [DeepTextSpotter](https://github.com/MichalBusta/DeepTextSpotter) with various sample images, to some degree of success.  Setup required building OpenCV a la [JetsonHacks script](https://github.com/jetsonhacks/buildOpenCVTX2) and a careful installation of caffe.  In a python terminal, we should be able to run the following without any errors:

```python
import numpy
import cv2
import caffe
print(cv2.__version__)
```
This should show an OpenCV version of 3.x.

Given that the above is working, we should be ready to build DeepTextSpotter.  I ran a modified version of the demo script which takes an image file as input.  That plan is to connect this to the ROS topic which is publishing the camera feed using cv_bridge.  Then DeepTextSpotter will search the image for areas that contain text and attempt to read them.  It reads on a character-by-character basis, so it often gets a letter or two wrong (if we were looking for certain words, we could use a string distance metric to compare the output to the target).  It can also read numbers, but that seems to be more difficult and more difficult to error correct (much less redundancy in numbers, though perhaps we could assume e.g. speed limits are a multiple of 5).

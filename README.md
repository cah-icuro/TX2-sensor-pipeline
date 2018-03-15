Section: Camera

We can run a preliminary test to see if we can get data from the camera via gstreamer:

gst-launch-1.0 nvcamerasrc ! 'video/x-raw(memory:NVMM), width=640, height=480, framerate=30/1, format=NV12' ! nvvidconv ! nvegltransform ! nveglglessink

If we get some sort of resource lock error, rebooting the TX2 should fix the issue.  If we change external cameras, we may need to adjust the gstreamer pipeline.

Once the basic gstreamer functionality works, the first step is to send the camera data from the external camera to a ROS node.  This is accomplished through the gscam package (http://wiki.ros.org/gscam).  Remember to always do source devel/setup.bash when trying to use ROS in a new terminal.  After installing the package, we need to set the GSCAM_CONFIG variable and then launch the package:

roscd gscam
cd bin # Make this directory if it doesn't exist
export GSCAM_CONFIG="nvcamerasrc ! video/x-raw(memory:NVMM),width=1280, height=720,format=I420, framerate=30/1 ! nvvidconv ! video/x-raw, format=BGRx ! videoconvert ! ffmpegcolorspace"
rosrun gscam gscam



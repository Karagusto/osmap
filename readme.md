# Osmap: ORB-SLAM2 Map serialization

[Osmap documentation with Doxygen](https://alejandrosilvestri.github.io/osmap/html/class_o_r_b___s_l_a_m2_1_1_osmap.html)

OSMAP stands for Orb-Slam2 Map.  It's a serialization addendum for ORB-SLAM2.

ORB-SLAM2 is a visual SLAM that generate a point cloud (sort of) map from a video stream, so it can localize itself in that map.  ORB-SLAM2 code in GitHub ar https://github.com/raulmur/ORB_SLAM2 is open source, and was made as a proof of concept to support its paper *ORB-SLAM2: an Open-Source SLAM System for Monocular, Stereo and RGB-D Cameras.*

ORB-SLAM2 is not a final product.  It generates maps but can't save them to a file, neither load them from a file.  Many proposals was made to add this functionality to ORB-SLAM2 code on GitHub, but saving maps to files ir beyond the project's porpuse.

After analizing many serializartion proposals, making one algorithm myself and having some experience with map files, I decide to restart this project from scratch and define a file format with broader goals in mind:

- Flexible evolution: let improve the format while maintaining compatibility
- Mind the size: shrink when possible, reconstruct when desirable.
- Format with options: map files can grow huge.  Let the user decide which parts to save and which parts to automatically reconstruct.
- Transparent, accesible and well documented: simplify format for debug and analisys, so third developers can do code with different porpuses than running on orb-slam2, like visualizing the map, editing it, analisyng its content, converting from a format to another, etc.  The serialization format can be used to real-time transfer the map to a visualizer via network.

This is why this serialization format has the following features:

- One map is serialized in many files.  If you want to analise mappoints only, you access that file alone.  In the future these files could be unified in a single zip file.
- Protocol buffers instead of boost::serialization.  proto files are a way to both document and generate serialization code.  Plus it applies some data compression.
- YAML as a header, mainly because ORB-SLAM2 already uses YAML, otherwise it could be JSON or XML.  That human readable header helps people who wants to analise or manipulate the map.


## The files in the project
- osmap.proto is the only protocol buffer definition, that serves as format documentation.

- osmap.pb.cc and osmap.pb.h are protocol buffer automatically generated code that defines messages classes.

- osmap.cpp and osmap.h defines the osmap class responsible for saving and loading maps.

- dummymap.h is provided to load and save a map without having to compile you application with orbslam2.  You can make map analisys applications without the burden of compiling with orbslam2.  To use dummymap.h instead of orbslam2's map you must define the preprocessor symbol OSMAP\_DUMMY\_MAP in you environment.  It is only used in osmap.h.

Example folder has some test files, which create some dummy map, saves it, loads it and show its values to verify the whole process.



## How to bundle with ORB-SLAM2
Right now osmap aims monocular SLAM only, so it won't serialize some variables needed for stereo SLAM and RGB-D SLAM.  But osmap could be extended to do that.

Because I don't pretend osmap be added to Raúl Mur's ORB-SLAM2, and because that project could still receive minor changes, this is the recipe to merge osmap with up to date orb-slam2.  It need some editing and compiling.

1- Install Google's Protocol Buffers and generate __osmap.pb.cc__ and __osmap.pb.h__ with the following command line:

    $ $ protoc --cpp_out=. osmap.proto

From now on, you must use these locally generated files instead of the provided in this repository.

2- Add Osmap files to ORB-SLAM2 project.  Copy osmap.pb.cc and osmap.cpp to src folder, and osmap.pb.cc and osmap.h files to include folder.  You don't need the extra files: nor dummymap.h, nor osmap.proto, etc.

3- Write the code to call save and load, usually attached to UI.  As an example, in Orb-Slam2's main.cc, this code will save and load a map:

    ...
    #include "Osmap.h"
    ...
    // Construct the osmap object, can be right after SLAM construction.  You only need one instance to load and save as many maps you want.
    ORB_SLAM2::Osmap osmap = ORB_SLAM2::Osmap(SLAM);
    ...
    // When you already has a map to save
    osmap.mapSave("myFirstMap");	// "myFirstMap" or "myFirstMap.yaml", same thing
    ...
    // Now you want to load the map
    osmap.mapLoad("myFirstMap.yaml");

4- Compile, run.


## About save options
There are many options that let you optimize map file size.  Options must be set before calling mapSave.  Most relevant are:

### ONLY\_MAPPOINTS\_FEATURES

    yourOsmapInstance.options.set(ORB_SLAM2::Osmap::ONLY_MAPPOINTS_FEATURES, 1);   
    
ORB-SLAM2 detects a lot of features, but uses only those belonging to a mappoint.  The other are useful to find new mappoints, but the chance to do it are pretty small.  ONLY\_MAPPOINTS\_FEATURES skips saving those unwanted features, shrinking your map files A LOT, like 5 times smaller.


### NO\_FEATURES\_DESCRIPTORS

    yourOsmapInstance.options.set(ORB_SLAM2::Osmap::NO_FEATURES_DESCRIPTORS | ORB_SLAM2::Osmap::ONLY_MAPPOINTS_FEATURES, 1);   

This option saves the descriptor on each mappoints, avoiding saving it on each mappoint observation.  Using NO\_FEATURES\_DESCRIPTORS with ONLY\_MAPPOINTS\_FEATURES (it usually doesn't make sense using it alone) your map file will shrink A _LOTTER_, like 20 times smaller.
 

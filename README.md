cocoapod-to-cordova
===================

Build tools the creates and updates a cordova plugman plugin.xml and vendored product from a cocoapod spec.
 * Fetch and build a cocoapod dependency.
 * Install and rename the cocoapod source files into the plugin.
 * Create and set the required `<platform name="ios">` plugin.xml elements.
 * Correctly handle some common issues in building a cordova plugin
 * Freeze the cocoapod dependency version as specified in the podfile.
 * Easy to pull in updates from the cocoapod dependency
 * Easy to use the best practice of only placing a compiled lib and not source files in the plugin

This build tool will not:
 * Template or write any obj-c wrappers / interfaces
 * Template or write any javascript APIs
 * Specify modules / clobbering or handle other sections of plugin.xml

## Install
 * Start creating your plugin following the [Plugin Spec](http://cordova.apache.org/docs/en/3.4.0/plugin_ref_spec.md). Note that you must have the `<platform name="ios">` element in your plugin.xml before proceeding.
 * Clone this repo into your plugin repo. Recommended to put it in 'scripts/update-ios-cocoapod'
```sh
cd plugin-xml-dir && mkdir -p scripts && cd scripts
git clone https://github.com/mkcode/cocoapod-to-cordova update-ios-cocoapod
```
 * Configure the `podfile` in `scripts/update-ios-cocoapod`. See below.
 * Build
```sh
cd scripts/update-ios-cocoapod
make                # For an ios release
make build-sim      # For debugging in the ios simulator
make && make clean  # Build and cleanup
```

## Update
 * Update the pod dependency version in `podfile`
```ruby
pod 'POD_NAME', '-> 3.2'
```

 * Rebuild
```sh
cd scripts/update-ios-cocoapod
make && make clean
```
 * Update CDVPlugin interfaces if there were any API changes

## Configuring the podfile
See http://guides.cocoapods.org/using/the-podfile.html for more info on using podfiles

Add a platform and a pod entry. See
```ruby
platform :ios, '6.0'
pod 'POD_NAME', 'POD_VERSION'
```

Integrate `CocoapodToCordovaBuilder` into the post_install hook
```ruby
post_install do |installer|
  require './cocoapod-to-cordova'
  build = CocoapodToCordovaBuilder.new('POD_NAME', installer.project)
  build.update_xcode_project!
end
```

Set the root_path and destination if they are not the defaults
```ruby
  # The directory path where plugin.xml lives
  build.root_path = File.expand_path(File.join('.', '../..')) # default

  # The root installation directory. Relative to the root_path.
  build.destination = 'src/ios/vendor' #default
```

CocoapodToCordovaBuilder#configure can take a number of options
```ruby
  build.configure({
    # Product is renamed and moved to the destination dir
    product: { name: "libmypod.a" },
    # Include the spanish localization from 'es.lproj'.
    localization: 'es',
    # Don't copy some headers. Copy the rest to 'destination/head'
    headers: { exclude: ['private_header.h', 'other_header.h'], sub_dir: 'head' },
    # Exclude this framework
    frameworks: { exclude: ['Foundation.framework'] },
    # Exclude this resource and copy the rest to 'destination/assets'
    resources: { sub_dir: 'assets', exclude: ['other-img.png'] }
  })
```
## Cordova Notes
 * Cordova plugins will fail to install if there is a filename conflict between the existing cordova project and the incomming plugin. If a generated plugin fails to install when installing with `cordova plugin add ...`, then add the conflicting files to the appropriate `exclude` section of the build configuration.
 * Localization files (en.lproj, es.lproj, etc) will always result in file name conflicts. This tool handles them specifically; by excluding them from the resources and only copying the specified .lproj's contained files into the project. See `localization` in `configure`

## Examples
This tool was extracted from [Cordova-DBCamera](https://github.com/vulume/Cordova-DBCamera).
See it in action there `scripts/update-ios/cocoapod`

Podfile in [Cordova-DBCamera](https://github.com/vulume/Cordova-DBCamera)
```ruby
platform :ios, '6.0'
pod 'DBCamera', git: 'git://github.com/danielebogo/dbcamera.git'

post_install do |installer|
  require './cocoapod-to-cordova'
  build = CocoapodToCordovaBuilder.new('DBCamera', installer.project)
  build.configure({
    product: { name: 'libdbcamera.a'},
    localization: 'en',
    headers: {
      exclude: [
        'DBCameraBaseCropViewController+Private.h',
        'DBCameraBaseCropViewController.h',
        'DBCameraCollectionViewController.h',
        'DBCameraCropView.h',
        'DBCameraGridView.h',
        'DBCameraLibraryViewController.h',
        'DBCameraMacros.h',
        'DBCameraManager.h',
        'DBCameraSegueViewController.h',
        'DBCollectionViewCell.h',
        'DBCollectionViewFlowLayout.h',
        'DBLibraryManager.h',
        'UIImage+Crop.h'
      ]
    }
  })
  build.update_plugin!
end
```

And the generated plugin.xml
```xml
<?xml version='1.0' encoding='UTF-8'?>
<plugin id='com.vulume.cordova.dbcamera' version='0.0.1' xmlns='http://apache.org/cordova/ns/plugins/1.0' xmlns:android='http://schemas.android.com/apk/res/android'>
  <name>
     dbcamera
  </name>
  <description>
     Plugman compatible wrapper for DBCamera.
  </description>
  <author>
     Chris Ewald, Vulume Inc.
  </author>
  <keywords>
     camera, ios
  </keywords>
  <license>
     MIT
  </license>
  <js-module name='dbcamera' src='www/dbcamera.js'>
    <clobbers target='cordova.plugins.dbcamera'/>
  </js-module>
  <platform name='ios'>
    <config-file parent='/*' target='config.xml'>
      <feature name='DBCamera'>
        <param name='ios-package' value='CDVdbcamera'/>
      </feature>
    </config-file>
    <source-file src='src/ios/CDVdbcamera.m'/>
    <framework src='Foundation.framework' autogen='true'/>
    <framework src='AVFoundation.framework' autogen='true'/>
    <framework src='CoreMedia.framework' autogen='true'/>
    <header-file src='src/ios/vendor/headers/DBCameraContainerViewController.h' autogen='true'/>
    <header-file src='src/ios/vendor/headers/DBCameraViewController.h' autogen='true'/>
    <header-file src='src/ios/vendor/headers/DBCameraDelegate.h' autogen='true'/>
    <header-file src='src/ios/vendor/headers/DBCameraView.h' autogen='true'/>
    <resource-file src='src/ios/vendor/resources/DBCameraImages.xcassets' autogen='true'/>
    <resource-file src='src/ios/vendor/resources/en.lproj/DBCamera.strings' autogen='true'/>
    <source-file framework='true' src='src/ios/vendor/libdbcamera.a' autogen='true'/>
  </platform>
</plugin>
```

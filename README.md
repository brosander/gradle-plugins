This is a suite of plugins designed to facilitate porting of Pentaho build logic to gradle, below are example settings.gradle and build.gradle for usage within a project.  These are currently under development and as such it is easier to use gradle files as they are built dynamically.  The finished product will be able to be published and resolved at the beginning of the build. These are targetted at a multiproject build that has many projects under one main directory.  These files would go at the root to allow automatic discovery and configuration of the other projects.

Example build.gradle:

    def rootProject = project

    file('/home/bryan/platform-codebase/gradle').listFiles().findAll { it.name.endsWith('.gradle')}.each { plugin ->
        println "Applying: " + plugin + " to " + project.name
        project.apply from: plugin 
    }

    subprojects {
      apply plugin: rootProject.ext['pentaho-ivy']
      sourceSets {
        main {
          java {
            srcDir 'src'
          }
        }
        test {
          java {
            srcDir 'test-src'
          }
        }
      }
    }

Example settings.gradle (adapted from http://gradle.1045684.n5.nabble.com/Nested-multi-projects-and-caveats-td4480779.html):

    def path = [] as LinkedList 

    rootDir.traverse( 
        type: groovy.io.FileType.FILES, 
        nameFilter: ~/build\.gradle/, 
        maxDepth: 3, 
        preDir: { path << it.name }, 
        postDir: { path.removeLast() }) { if (path) include path.join(":") } 

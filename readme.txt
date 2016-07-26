Hosting a Maven repository on github

The best solution I've been able to find consists of these steps:

Create a branch called mvn-repo to host your maven artifacts.
Use the github site-maven-plugin to push your artifacts to github.
Configure maven to use your remote mvn-repo as a maven repository.
There are several benefits to using this approach:

Maven artifacts are kept separate from your source in a separate branch called mvn-repo, much like github pages are kept in a separate branch called gh-pages (if you use github pages)
Unlike some other proposed solutions, it doesn't conflict with your gh-pages if you're using them.
Ties in naturally with the deploy target so there are no new maven commands to learn. Just use mvn deploy as you normally would
The typical way you deploy artifacts to a remote maven repo is to use mvn deploy, so let's patch into that mechanism for this solution.

First, tell maven to deploy artifacts to a temporary staging location inside your target directory. Add this to your pom.xml:

<distributionManagement>
    <repository>
        <id>internal.repo</id>
        <name>Temporary Staging Repository</name>
        <url>file://${project.build.directory}/mvn-repo</url>
    </repository>
</distributionManagement>

<plugins>
    <plugin>
        <artifactId>maven-deploy-plugin</artifactId>
        <version>2.8.1</version>
        <configuration>
            <altDeploymentRepository>internal.repo::default::file://${project.build.directory}/mvn-repo</altDeploymentRepository>
        </configuration>
    </plugin>
</plugins>
Now try running mvn clean deploy. You'll see that it deployed your maven repository to target/mvn-repo. The next step is to get it to upload that directory to GitHub.

Add your authentication information to ~/.m2/settings.xml so that the github site-maven-plugin can push to GitHub:

<!-- NOTE: MAKE SURE THAT settings.xml IS NOT WORLD READABLE! -->
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>YOUR-USERNAME</username>
      <password>YOUR-PASSWORD</password>
    </server>
  </servers>
</settings>
(As noted, please make sure to chmod 700 settings.xml to ensure no one can read your password in the file. If someone knows how to make site-maven-plugin prompt for a password instead of requiring it in a config file, let me know.)

Then tell the GitHub site-maven-plugin about the new server you just configured by adding the following to your pom:

<properties>
    <!-- github server corresponds to entry in ~/.m2/settings.xml -->
    <github.global.server>github</github.global.server>
</properties>
Finally, configure the site-maven-plugin to upload from your temporary staging repo to your mvn-repo branch on Github:

<build>
    <plugins>
        <plugin>
            <groupId>com.github.github</groupId>
            <artifactId>site-maven-plugin</artifactId>
            <version>0.11</version>
            <configuration>
                <message>Maven artifacts for ${project.version}</message>  <!-- git commit message -->
                <noJekyll>true</noJekyll>                                  <!-- disable webpage processing -->
                <outputDirectory>${project.build.directory}/mvn-repo</outputDirectory> <!-- matches distribution management repository url above -->
                <branch>refs/heads/mvn-repo</branch>                       <!-- remote branch name -->
                <includes><include>**/*</include></includes>
                <repositoryName>YOUR-REPOSITORY-NAME</repositoryName>      <!-- github repo name -->
                <repositoryOwner>YOUR-GITHUB-USERNAME</repositoryOwner>    <!-- github username  -->
            </configuration>
            <executions>
              <!-- run site-maven-plugin's 'site' target as part of the build's normal 'deploy' phase -->
              <execution>
                <goals>
                  <goal>site</goal>
                </goals>
                <phase>deploy</phase>
              </execution>
            </executions>
        </plugin>
    </plugins>
</build>
The mvn-repo branch does not need to exist, it will be created for you.

Now run mvn clean deploy again. You should see maven-deploy-plugin "upload" the files to your local staging repository in the target directory, then site-maven-plugin committing those files and pushing them to the server.

[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building DaoCore 1.3-SNAPSHOT
[INFO] ------------------------------------------------------------------------
...
[INFO] --- maven-deploy-plugin:2.5:deploy (default-deploy) @ greendao ---
Uploaded: file:///Users/mike/Projects/greendao-emmby/DaoCore/target/mvn-repo/com/greendao-orm/greendao/1.3-SNAPSHOT/greendao-1.3-20121223.182256-3.jar (77 KB at 2936.9 KB/sec)
Uploaded: file:///Users/mike/Projects/greendao-emmby/DaoCore/target/mvn-repo/com/greendao-orm/greendao/1.3-SNAPSHOT/greendao-1.3-20121223.182256-3.pom (3 KB at 1402.3 KB/sec)
Uploaded: file:///Users/mike/Projects/greendao-emmby/DaoCore/target/mvn-repo/com/greendao-orm/greendao/1.3-SNAPSHOT/maven-metadata.xml (768 B at 150.0 KB/sec)
Uploaded: file:///Users/mike/Projects/greendao-emmby/DaoCore/target/mvn-repo/com/greendao-orm/greendao/maven-metadata.xml (282 B at 91.8 KB/sec)
[INFO] 
[INFO] --- site-maven-plugin:0.7:site (default) @ greendao ---
[INFO] Creating 24 blobs
[INFO] Creating tree with 25 blob entries
[INFO] Creating commit with SHA-1: 0b8444e487a8acf9caabe7ec18a4e9cff4964809
[INFO] Updating reference refs/heads/mvn-repo from ab7afb9a228bf33d9e04db39d178f96a7a225593 to 0b8444e487a8acf9caabe7ec18a4e9cff4964809
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 8.595s
[INFO] Finished at: Sun Dec 23 11:23:03 MST 2012
[INFO] Final Memory: 9M/81M
[INFO] ------------------------------------------------------------------------
Visit github.com in your browser, select the mvn-repo branch, and verify that all your binaries are now there.

enter image description here

Congratulations!

You can now deploy your maven artifacts to a poor man's public repo simply by running mvn clean deploy.

There's one more step you'll want to take, which is to configure any poms that depend on your pom to know where your repository is. Add the following snippet to any project's pom that depends on your project:

<repositories>
    <repository>
        <id>YOUR-PROJECT-NAME-mvn-repo</id>
        <url>https://raw.github.com/YOUR-USERNAME/YOUR-PROJECT-NAME/mvn-repo/</url>
        <snapshots>
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </snapshots>
    </repository>
</repositories>
Now any project that requires your jar files will automatically download them from your github maven repository.

Edit: to avoid the problem mentioned in the comments ('Error creating commit: Invalid request. For 'properties/name', nil is not a string.'), make sure you state a name in your profile on github.
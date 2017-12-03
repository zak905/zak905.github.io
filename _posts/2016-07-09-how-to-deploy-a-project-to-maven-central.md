---
layout: post
title: "How to deploy your project to Maven Central"
author: "Zakaria"
---


So, you have finally got your java library/project finalized, tested, and ready to go. if want to make it available to users through their build system (Maven, Gradle,..), you need to deploy it to Maven Central. This post presents the steps to do so. 

1. <ins>**Create a ticket in Sonatype for the creation of your Group ID:**<ins>

    This can be done by creating an issue in Sonatype's tracker: [https://issues.sonatype.org/](https://issues.sonatype.org/). You usually get a response within one day.

2. <ins>**Get your project ready to deploy:**<ins>

    Before deploying your project, you need to make sure that your pom.xml contains a number of elements: project name, description, URL, license information, developer information, and SCM information. If you do not include these elements, your pom file will not be validated and you will not be able to deploy to Maven Central. Here is a pom example: [https://github.com/gwidgets/gwty-leaflet/blob/master/pom.xml](https://github.com/gwidgets/gwty-leaflet/blob/master/pom.xml)

3. <ins>**Snapshot VS Non snapshot:**<ins>

    If your release is a snapshot version, you need to include Sonatype's snapshots server. On deploy, Maven automatically detects if your version ends with -SNAPSHOT and deploys to the snapshot server.

   ``` 
   <distributionManagement>
    <snapshotRepository>
        <id>ossrh</id>
        <url>https://oss.sonatype.org/content/repositories/snapshots</url>
    </snapshotRepository>
    </distributionManagement>
    ```

4. <ins>**Generating Javadocs and including sources:**<ins>

    Including javadocs and sources is also mandatory, and checked by Sonatype at release time. There are several ways you can do it, but the easiest way is to include Maven javadocs and sources plugins which automatically generates them for you on build time.

    Maven javadoc plugin : [https://maven.apache.org/plugins/maven-javadoc-plugin/](https://maven.apache.org/plugins/maven-javadoc-plugin/)

    Maven source plugin:  [https://maven.apache.org/plugins/maven-source-plugin/](https://maven.apache.org/plugins/maven-source-plugin/)

5. <ins>**Sign your jar and pom:**<ins>

    This is the trickiest part. To be able to successfully release your project to Maven central, you need to generate a signature using GPG which will be used by Sonatype to verify your articacts. First of all, you will need to install [GPG](https://www.gnupg.org/download/). The next step is to generate a public/private key pair. This step is descibed in more details in GPG's manual. Aftewards, you need to distribute your public key to a key server. You can use [https://pgp.mit.edu/](https://pgp.mit.edu/) which allows you to directly copy paste the key. There are plenty of servers, it's a matter of preference. You can refer once again to the manual for how to send your keys to the server. Once done, you can sign your artifacts, you will have to sign both the jar and the pom. You can either do it manually from the termnial using gpg or using maven gpg plugin: [http://maven.apache.org/plugins/maven-gpg-plugin/](http://maven.apache.org/plugins/maven-gpg-plugin/)

6. <ins>**Prepare your deploy:**<ins>

    Before deploying you will need to configure the server and its access information:

    ```
    <distributionManagement>
    <snapshotRepository>
        <id>ossrh</id>
        <url>https://oss.sonatype.org/content/repositories/snapshots</url>
    </snapshotRepository>
    <repository>
        <id>ossrh</id>
        <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
    </repository>
    </distributionManagement>

    <settings>
    <servers>
        <server>
        <id>ossrh</id>
        <username>your-username</username>
        <password>your-pwd</password>
        </server>
    </servers>
    </settings>
    ```

    It's better to include these information in a build profile, so that they are not shared if you ever make your project source code open to public, and also to be able to reuse them for other projects.
    It's also a good practice to include maven's source, javadocs, and gpg plugins in a build profile, to avoid having to include them in each project.

7. <ins>**Deploy to staging:**<ins>

    deploying to staging can simply be done by invoking Maven's deploy goal: `mvn deploy`

8. <ins>**Relase your project to Maven Central:**<ins>

    At this stage, you will able to find your project on Sonatype. You need to go to [https://oss.sonatype.org/](https://oss.sonatype.org/) and click on staging repositories. After choosing your repository, you need to click on --> close, as shown below. Once the repository is closed, you can click on release and your project will be released to Maven Central. 

    ![Sonatype]({{site.url}}/assets/images/sonatype-release.png "Sonatype release")
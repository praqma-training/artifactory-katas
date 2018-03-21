
## Creating an Artifactory Server Instance
 Manage | Configure System.

### setup

* Go in and make a new pipeline job in Jenkins called `jenkins-artifactory-plugin`.
* Go into the job and paste the following skeleton into the pipeline part.

```groovy
node {
     def buildInfo = Artifactory.newBuildInfo()
     buildInfo.env.capture = true
     def server = Artifactory.newServer url: 'http://nginx/artifactory', username: 'admin', password: 'lkvmxxcv'
    stage('Preparation') { // for display purposes
        // Get some code from a GitHub repository
                // Step 1 goes here.....
    }
    stage('Build') {
        // "build" some software
        sh "echo 'this is our "binary"'>build.txt"
    }
    stage('Upload') {
        //Upload the artifact to Artifactory
    }
    stage('Download') {
    deleteDir() //Deletes the entire workspace, so we rely 100% on artifactory
    //Insert filespec for downloading the artifact from Artifactory again
    }

    stage ('promote'){
        // Promote the build to the next level
    }

}
```

This is our baseline.

> Note: we set the credentials and url of the Artifactory server directly in the jenkins file here which are not considered best practice. Consult the [web docs](https://www.jfrog.com/confluence/display/RTF/Working+With+Pipeline+Jobs+in+Jenkins) for better ways of creating the artifactory connection.

### Upload the artifact

File Specs are JSON objects that are used to specify the details of files you want to upload or download to or from Artifactory. They consist of a `pattern` and a `target`, and a number of optional properties can be added to them.

> Note: The filespec has a lot of options you can work with. For more in depth information, consult the [web docs](https://www.jfrog.com/confluence/display/RTF/Using+File+Specs).

This is the basic structure of an upload spec:
```json
{
  "files": [
    {
      "pattern": "[Mandatory]",
      "target": "[Mandatory]",
      "props": "[Optional]", // Properties that will be set to all artifacts uploaded to artifactory
      "recursive": "[Optional, Default: 'true']",
      "flat" : "[Optional, Default: 'true']",
      "regexp": "[Optional, Default: 'false']",
      "explode": "[Optional, Default: false]",
      "excludePatterns": ["[Optional]"]
    }
  ]
}
```

In a Jenkins pipeline, you can define an upload spec as a multi-line string:
```groovy
def uploadSpec = """{
  "files": [
    {
      "pattern": "yourPatternGoesHere",
      "target": "yourTargetGoesHere"
    }
  ]
}"""
```

Once you have your upload spec, you can trigger the upload within the pipeline:

```groovy
server.upload spec: uploadSpec, buildInfo: buildInfo
```

In order for us to upload a file, we need to define two things; `pattern` and `target`

* `pattern` Specifies the local file system path to artifacts which should be uploaded to Artifactory. You can specify multiple artifacts by using wildcards or a regular expression as designated by the regexp property. If you use a regexp, you need to escape any reserved characters (such as ".", "?", etc.) used in the expression using a backslash "\\".
* `target` Specifies the target path in Artifactory in the following format: [repository_name]/[repository_path]

**tasks:**

* in the `upload` step of your jenkins file, make an upload spec that takes the `build.txt` and uploads it to the `*-generic-gradle-1` repo under `com/acme/1.${currentBuild.number}/build-1.${currentBuild.number}.txt`
* go into artifactory UI and look at the corresponding file being uploaded into the correct repository and directory.

> hint: the `${currentBuild.number}` is a Jenkins variable for accessing the build number.

### Download the artifact

If we need to download artifacts, we can use File Specs again:

```json
{
  "files": [
    {
      "pattern" or "aql": "[Mandatory]",
      "target": "[Optional, Default: ./]",
      "props": "[Optional]",
      "recursive": "[Optional, Default: true]",
      "flat": "[Optional, Default: false]",
      "build": "[Optional]",
      "explode": "[Optional, Default: false]",
      "excludePatterns": ["[Optional]"]
    }
  ]
}
```

There are two key differences to notice:

* You can use `aql` to search for artifacts as well as the normal pattern based search.
* The `props` attribute is no longer used for setting properties, but as a filter for only downloading the ones where the given property is set.

**tasks:**

* Modify your pipeline so it downloads the file you just uploaded
* check within the pipeline that a folder is made. You could for instance use `sh : "ls"`
* use the `flat` property to download the file to the root of the workspace, ignoring the repository folder structure.

>hint: you can both make a search on the specific layout, or use the `@` notation to search for build number and name (details can be found [here](https://www.jfrog.com/confluence/display/RTF/Artifactory+Query+Language#ArtifactoryQueryLanguage-ConstructingSearchCriteria))

### Promotion

As your build progresses through the pipeline, your artifacts gets promoted to a higher maturity repository.

Promotion has a different JSON structure than uploading and downloading. Here you can see an example of a promotion structure:

```groovy
def promotionConfig = [
    // Mandatory parameters
    'buildName'          : buildInfo.name,
    'buildNumber'        : buildInfo.number,
    'targetRepo'         : '', //name of the repo to promote the artifacts to

    // Optional parameters
    'comment'            : '', //Comment that will be seen on the build entity
    'sourceRepo'         : '', //name of the repo to promote the artifacts from
    'status'             : '', //Denotion of maturity level for the build.
    'includeDependencies': true, // If your artifact has any dependencies, should they be promoted as well?
    'copy'               : true, // Should the artifacts be moved or copied when promoting from one repo to another.
    // 'failFast' is true by default.
    // Set it to false, if you don't want the promotion to abort upon receiving the first error.
    'failFast'           : true
] 
// Promote build
server.promote promotionConfig
```

**tasks:**

* In the `promote` stage, make a promotion config that copies the artifacts from `-generic-gradle-1` to `-generic-gradle-2`.
* Execute the pipeline to check that the artifacts gets copied over.
* Make a new stage `interactive promote` 
* In that stage make a promotionConfig that promotes the artifacts from `-generic-gradle-2` to `-generic-gradle-3`.
* Instead of making an automated promotion replace `server.promote promotionConfig` with `Artifactory.addInteractivePromotion server: server, promotionConfig: promotionConfig` to make an interactive promotion.
* Observe in Artifactory that `-generic-gradle-2` has the artifact, and `-generic-gradle-3` does not yet
* Go back into the jenkins Ui and click "promote" to promote the artifact.
* Observe in Artifactory that `-generic-gradle-3` has the artifact now as well.

### Properties

* properties

### Best practices

* I have multiple stages that I upload things from, but I can only
* Should I move or copy my artifacts when i promote?

### Resulting pipeline

```groovy
node {
     def buildInfo = Artifactory.newBuildInfo()
     buildInfo.env.capture = true
     buildInfo.retention maxBuilds: 2, deleteBuildArtifacts: true
     def server = Artifactory.newServer url: 'http://ec2-18-197-71-5.eu-central-1.compute.amazonaws.com:8081/artifactory', username: 'admin', password: 'orange'
    stage('Preparation') { // for display purposes
        // Get some code from a GitHub repository
    }
    stage('Build') {
        // Run the maven build
        sh "echo 'hej'>build.txt"
        sh "ls"

    }
    stage('Upload') {
        buildNumber = currentBuild.number;
        def uploadSpec = """{
            "files": [
            {
           "pattern": "build.txt",
            "target": "sal-generic-gradle-1/org/module/1.${buildNumber}/module-1.${buildNumber}.txt"
            
                , "props": "hej=hej"
            }
            ]
            }"""
            echo "before buildInfo"
        server.upload spec: uploadSpec, buildInfo: buildInfo
        echo "after buildInfo"
        server.publishBuildInfo buildInfo
    }
    stage('Download') {
    deleteDir() //Deletes the entire workspace, so we rely 100% on artifactory
    def downloadSpec= """{
  "files": [
    {
     "aql": {
        "items.find":{
                "repo": "sal-generic-gradle-1",
                "@build.name":"jenkins-artifactory-plugin",
                "@build.number":"13"
                }
    },
    "flat":"true"
   }
  ]
}"""
server.download spec: downloadSpec, buildInfo: buildInfo
sh 'ls'
    }

    stage ('promote'){
        def promotionConfig = [
        //Mandatory parameters
        'buildName'          : buildInfo.name,
        'buildNumber'        : buildInfo.number,
        'targetRepo'         : 'sal-generic-gradle-2',

        //Optional parameters
        'comment'            : 'this is the promotion comment',
        'status'             : 'Released',
        'includeDependencies': true,
        'failFast'           : true,
        'copy'               : true
    ]

    // Promote build
server.promote promotionConfig
//Artifactory.addInteractivePromotion server: server, promotionConfig: promotionConfig, displayName: "Promote me please"
    }
      stage ('promote again'){
        def promotionConfig = [
        //Mandatory parameters
        'buildName'          : buildInfo.name,
        'buildNumber'        : buildInfo.number,
        'targetRepo'         : 'sal-generic-gradle-3',

        //Optional parameters
        'comment'            : 'this is the promotion comment',
        'sourceRepo'         : 'sal-generic-gradle-2',
        'status'             : 'Released',
        'includeDependencies': true,
        'failFast'           : true,
        'copy'               : true
    ]

    // Promote build
Artifactory.addInteractivePromotion server: server, promotionConfig: promotionConfig
    }
        stage ('dance'){
            echo 'hep hey'
        }

}
```
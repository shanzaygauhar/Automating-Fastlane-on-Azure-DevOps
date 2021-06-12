**FASTLANE IOS BUILD AND AZURE DEVOPS**

**DOCUMENTATION BY:**

**SHANZAY GAUHAR**


With fastlane, it is recommended to keep most of the actions in the fastfile and only the

necessary/command line actions in the pipeline i.e. yaml file.

Add --verbose command in the BASH SCRIPT to track the process and spot error if any.

# FASTLANE AND AZURE DEVOPs

I implemented the fastlane using the concept of environment variables. The variables are added in the azure pipeline using the variable tab. The variable corresponding to the certain action would have the prefix of that action name. For instance, if I&#39;m using match and I need to specify the type for match, I would do so by MATCH\_TYPE.

 Moreover, the secret variables in the pipeline are not accessed by the build process unless they are explicitly mentioned or stated where required.

**THE DETAILS OF ANY FASTLANE ACTION CAN BE OBTAINED BY RUNNING ON TERMINAL:**

| **fastlane action action\_name** |
| --- |

## **IOS BUILD - PIPELINE**

The actions that pipeline does is it specifies the virtual image that is required. The common used are &quot;macOS-latest&quot; or &quot;macOS 10.14&quot;.

### **Fastlane commands and their setup on pipeline**

1. **SSH Keys for Fastlane sync\_code\_signing**

Fastlane doc link: [https://docs.fastlane.tools/actions/sync\_code\_signing/](https://docs.fastlane.tools/actions/sync_code_signing/)

The provisioning profile and certificates are kept in a private repository and match downloads and uses those to sign the ipa. In order to enable the match command to install certificates successfully, we used ssh keys where the repository was hosted on azureDevOps. SSH keys were created manually on desktop and then configured for Azure devOps.

**SSH KEYS**

**Commands to generate ssh keys**

[**https://docs.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops**](https://docs.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops)

[**https://docs.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent**](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

**Ssh-keygen -p keyname →** to change the passphrase of the private ssh key

**LEAVING PASSPHRASE FOR SSH KEY EMPTY/ SKIPPING THAT STEP LED TO UNEXPECTED ERRORS. DEFINE PASSPHRASE AND STORE IT IN** _ **MATCH\_PASSWORD** _ **VARIABLE.**

These commands generate private/public key pairs. The private key is uploaded to library/securefiles and then the task InstallSSHKeys is added. The private key is added as following

![](RackMultipart20210612-4-mrsqvz_html_617b835edaacb1e6.png)

Choose the SSH public key and then add a new key and in it copy paste the contents of .pub file - i.e corresponding public key.

I had issues with known\_host and details of known\_host can be found in the following link i.e. [https://www.ssh.com/ssh/host-key](https://www.ssh.com/ssh/host-key)

The input parameter for the pipeline InstallSSHKey@0 task includes:

| **knownHostsEntry**** : **** //specify the known\_hosts that hosts the repository ****sshKeySecureFile**** : **** //specify the name of private key uploaded to secure file**
 |
| --- |

In the first run readonly parameter is set to false, and this creates and uploads the certificates to be used later. This step needs to be done by the admin of the developer account and in my case, this step was performed manually through homebrew.

In the later runs, the readonly parameter is set to true because we don&#39;t want the match command to create the certificates itself.

**DO NOT MANUALLY ADD CERTIFICATES AS IT CAN LEAD TO POTENTIAL ERRORS. This option is available but is often discouraged.**

1. **Ruby Version**

Specify the ruby version to greater than equal to 2.4 to successfully run/compile fastfile.

1. **Node - npm tool**

Node modules are required to accommodate the built of app and can be done using azure provided task as displayed below:

| task: NodeAndNpmTool@1inputs:versionSpec: &#39;10.x&#39;displayName: &#39;Install Node&#39;- script: npm installdisplayName: &#39;Install dependencies&#39;
 |
| --- |

1. **Bash Script**

Add the following lines to BASH Script to ensure latest xcode tools are installed and bundle update ensures that it&#39;s dependencies and all the other dependencies that are in our gemfile are up to date.

**xcode-select --install**

**bundle update --bundler**

## IOS BUILD - FASTFILE / GEMFILE

&quot;_A Gemfile describes the gem dependencies required to execute associated Ruby code. Place the Gemfile in the root of the directory containing the associated code. &quot;_

iOS built requires cocoapods and we require fastlane as well. In our gemfile, we will add along with all the dependencies we need for our ruby code:

gem(&quot;fastlane&quot;)

gem(&quot;gym&quot;) //Didn&#39;t try this. Read it on some document

gem(&quot;cocoapods&quot;)

Bundle init → creates the gemfile.

Bundle install → installs the gems listed in gemfile that are associated with ruby.

Bundle update --bundler → updates all the gems listed in gemfile along with their

dependencies

These above commands can be included on a pipeline in the BASH Script task or manually in the local machine to create gemfile.

### **APP\_CREATION LANE**

###### **create\_app\_online**

Fastlane docs: [https://docs.fastlane.tools/actions/create\_app\_online/](https://docs.fastlane.tools/actions/create_app_online/)

First of all, we need to provide the details of our apple account so we can start with the process of fastlane for ios. The following variable allows us to do so:

1. FASTLANE\_USER → Apple ID
2. FASTLANE\_PASSWORD → Apple account password
3. FASTLANE\_TEAM\_NAME
4. FASTLANE\_ITC\_TEAM\_NAME

In case, the account uses two way authentication, we may declare a variable FASTLANE\_SESSION in our pipeline to tackle this situation. In order to generate the value for this variable run the following command on your terminal/ homebrew:

| fastlane spaceauth -u &quot;yourappleID&quot; |
| --- |

This command will generate a value for you which you will assign to FASTLANE\_SESSION in your pipeline.

Moreover, this is an alias for the &quot;produce&quot; action of fastlane. According to documentation,

&quot;_produce_ creates new iOS apps on both the Apple Developer Portal and App Store Connect with the minimum required information.&quot; The parameter I used includes the following:

1. PRODUCE\_APP\_IDENTIFIER
2. PRODUCE\_APP\_NAME
3. PRODUCE\_PLATFORM
4. PRODUCE\_SKU → declare any unique number yourself

This will check if the profile already exists else it&#39;ll create one.

### **SIGN LANE:**

###### **sync\_code\_signing**

Fastlane docs: [https://docs.fastlane.tools/actions/sync\_code\_signing/](https://docs.fastlane.tools/actions/sync_code_signing/)

It syncs the certificates and profiles across the team via match. The git storage for the match command on azure devOps is discussed above.

Matchfile is generated by command match init. It creates the matchfile which can be configured in the following manner:

git\_url(&quot;SPECIFY SSH LINK HERE&quot;)

storage\_mode(&quot;git&quot;)

type(&quot;appstore&quot;) # The default type, can be: appstore, adhoc, enterprise or development

The rest of the parameters can be set or accessed through environment variables on the pipeline. The environment variables include:

1. MATCH\_APP\_IDENTIFIER
2. MATCH\_CLONE\_BRANCH\_DIRECTLY
3. MATCH\_GIT\_URL
4. MATCH\_GIT\_USER\_EMAIL
5. MATCH\_KEYCHAIN\_NAME
6. MATCH\_PASSWORD
7. MATCH\_READONLY
8. MATCH\_TEAM\_NAME
9. MATCH\_TYPE

The actions can create lane variables that facilitate the communication between actions. Hence in our fastfile, after we call the function sync\_code\_signing we might call:

mapping=Actions.lane\_context[SharedValues::MATCH\_PROVISIONING\_PROFILE\_MAPPING]

This gives the provisioning profile obtained from the match command so it can be used with other actions like done below with match.

###### **update\_code\_signing\_setting**

Fastlane docs: [https://docs.fastlane.tools/actions/update\_code\_signing\_settings/](https://docs.fastlane.tools/actions/update_code_signing_settings/)

| update\_code\_signing\_settings(profile\_name: mapping[ENV[&#39;MATCH\_APP\_IDENTIFIER&#39;]]use\_automatic\_signing:false,path:&quot;fastlane/ios/Project.xcodeproj&quot;) |
| --- |

This function updates and configures the xcode signing configurations:

The environment variables in the pipeline includes:

1. FL\_CODE\_SIGN\_IDENTITY
2. FL\_OUTPUT\_DIR
3. FL\_PROJECT\_SIGNING\_BUILD\_CONFIGURATIONS
4. FL\_PROJECT\_SIGNING\_PROJECT\_PATH
5. FL\_PROJECT\_SIGNING\_TARGETS

profile\_name: provisioning profile name to use for code signing. For instance type is development and identifier is xxx.com.xxx, this command &quot;mapping[ENV[&#39;MATCH\_APP\_IDENTIFIER&#39;]]&quot; uses the mapping variable declared abovesets it to match Development xxx.xom.xxx in the XCODE project configuration.

### **Beta\_build LANE:**

###### **cocoapods**

Fastlane docs: [https://docs.fastlane.tools/actions/cocoapods/](https://docs.fastlane.tools/actions/cocoapods/)

| cocoapods( clean\_install: true, podfile: &quot;./ios&quot;) |
| --- |

This runs pod install and creates the podfile required for iOS built. This function is added in fastfile in either before\_all block or in the build lane. Many suggest adding it in before\_all in platform: ios but I implemented it in this lane.

###### **create\_keychain**

Fastlane docs:[https://docs.fastlane.tools/actions/create\_keychain/](https://docs.fastlane.tools/actions/create_keychain/)

setup\_ci() is an alternative method to declare a keychain for a pipeline but in effort to resolve some other error I came across this method. We can declare our own keychain to store match certificates and profiles. The environment variable set into the pipeline includes the following:

1. KEYCHAIN\_NAME
2. KEYCHAIN\_PASSWORD
3. KEYCHAIN\_UNLOCK --------→ true so we can install and use our certificates
4. KEYCHAIN\_LOCK\_WHEN\_SLEEPS
5. KEYCHAIN\_DEFAULT\_KEYCHAIN
6. KEYCHAIN\_ADD\_TO\_SEARCH\_LIST

In order to use this keychain, set MATCH\_KEYCHAIN\_NAME equal to the value of KEYCHAIN\_NAME and MATCH\_KEYCHAIN\_PASSWORD equal to KEYCHAIN\_PASSWORD.

Add the following lines to the Fastfile,

unlock\_keychain(path:&quot;PATH TO KEYCHAIN&quot;, password:&quot;PASSWORD&quot;)

sh &quot;security set-keychain-settings -t 10000 -l ~/Library/Keychains/NAME\_OF\_KEYCHAIN.keychain&quot;

###### **disable\_automatic\_code\_signing**

This only requires a path to the xcode project.

**This is deprecated version and this can be done with update\_code\_signing\_setting use\_automatic\_signing set to false**

###### **update\_project\_provisioning &amp;&amp; update\_project\_team**

This generally is not used when we use match and gym because it automatically maps but pipeline built was not picking up profile or the team ID so I added the following to my Fastfile after setting up a keychain.

| update\_project\_provisioning(xcodeproj: &quot;ios/Project.xcodeproj&quot;,profile: ENV[&quot;sigh\_com.gamechefs.vs\_development\_profile-path&quot;],build\_configuration: &quot;Release&quot;)update\_project\_team(path: &quot;ios/Project.xcodeproj&quot;,teamid: &quot;HQ6CGYLV6L&quot;) |
| --- |

ENV[&quot;sigh\_com.gamechefs.vs\_development\_profile-path&quot;] → this environment variable is generated by match command and contains the path to the profile.

**Syntax - not sure - sigh\_yourBundleID\_yourMatchCommandType\_profile-path**

###### **gym/build\_ios\_app**

Now we may call a gym or build\_ios\_app command that will generate the required signed .ipa file which we will later upload to our App Store account. The parameters of the gym command or build\_ios\_app command includes the following:

gym(scheme: &quot;Project&quot;, workspace: &quot;ios/Project.xcworkspace&quot;,export\_method: &quot;app-store&quot;,output\_directory: &quot;ios/output&quot;,clean: &quot;true&quot;, export\_options:{signingStyle: &quot;manual&quot;})

Export method basically is the method that is being used to export the archive. Possible values include: app-store, ad-hoc, package, enterprise, development, developer-id.

![](RackMultipart20210612-4-mrsqvz_html_48a16c6c248dc03d.png)

These sharedValue variables can be utilized similarly as was displayed above for match. This can also be used to communicate an ipa path to upload\_to\_testflight action or upload\_to\_app\_store or any other action that requires an ipa file.

###### **Automatic\_code\_signing**

Fastlane docs: [https://docs.fastlane.tools/actions/automatic\_code\_signing/](https://docs.fastlane.tools/actions/automatic_code_signing/)

We enable automatic signing after the gym command.

###### **increment\_build\_number\_in\_plist**

This is one of fastlane plugins that might be used to update the build number. This function is added in fastfile.

Syntax:

| **increment\_build\_number\_in\_plist** (xcodeproj:&#39;ios/Project.xcodeproj&#39;, target: &#39;Project&#39;) |
| --- |

###### **upload\_to\_testflight**

Fastlane docs: [https://docs.fastlane.tools/actions/upload\_to\_testflight/](https://docs.fastlane.tools/actions/upload_to_testflight/)

This uploads the binary file - .ipa to AppStore Connect for Testflight beta testing.

![](RackMultipart20210612-4-mrsqvz_html_f08b8de4dea30809.png)

It is recommended that password does not contain any special characters otherwise unexpected errors might occur.

Application specific passwords may be used to upload. For this inorder to avoid two way authentication or 2FA, these two variables must be properly set - skip\_waiting\_for\_build\_processing and apple\_id. This password is set in the environment variable: [FASTLANE\_APPLE\_APPLICATION\_SPECIFIC\_PASSWORD](https://docs.fastlane.tools/best-practices/continuous-integration/#application-specific-passwords)

The documents to perform this step can be located here: [https://docs.fastlane.tools/best-practices/continuous-integration/#application-specific-passwords](https://docs.fastlane.tools/best-practices/continuous-integration/#application-specific-passwords)

The several parameters can be located in the upload\_to\_testflight fastlane docs. The IPA path can be obtained similar to the method as demonstrated above in the document here [https://docs.google.com/document/d/1CQnOwxQtn6f\_OvmtDnMq8HkrMzFFgB7kC0v764vw9KY/edit#heading=h.gg278zx2m39a](https://docs.google.com/document/d/1CQnOwxQtn6f_OvmtDnMq8HkrMzFFgB7kC0v764vw9KY/edit#heading=h.gg278zx2m39a) ----\&gt; In the link it is using the SharedValue lane variable to obtain path to profile.

###### **commit\_version\_bump**

Fastlane docs: [https://docs.fastlane.tools/actions/commit\_version\_bump/](https://docs.fastlane.tools/actions/commit_version_bump/)

This command is run after we have incremented the built number. It then creates a version bump commit. It checks for changes in xcodeproj or plist file.

###### **push\_to\_git\_remote**

Fastlane docs: [https://docs.fastlane.tools/actions/push\_to\_git\_remote/](https://docs.fastlane.tools/actions/push_to_git_remote/)

This command pushes local changes to remote repositories. According to link above,

![](RackMultipart20210612-4-mrsqvz_html_c45c7e9df6d0abb6.png)

###### **private lane :update\_version**

**Pre-Condition:** Semantic versioning is chosen in the xcode/xcworkspace project.

**1.2.3 →** the leftmost is major, middle minor, rightmost patch.

This self declared private lane updates the version number by comparing the version number listed on appstore and plist and if they are the same, we increment the minor value else we increment the patch. If both versions are the same then it might be a major update and we increment the minor. If they are not the same versions, we might increment the patch.

| private lane :update\_version do app\_store\_version = get\_app\_store\_version\_number(bundle\_id:&#39;&#39;YOUR BUNDLE ID&quot;) plist\_version = get\_version\_number\_from\_plist(xcodeproj:&#39;./ios/Project.xcodeproj&#39;) if Gem::Version.new(plist\_version.to\_f) == Gem::Version.new(app\_store\_version.to\_f) increment\_version\_number\_in\_plist(xcodeproj:&#39;ios/Project.xcodeproj&#39;,bump\_type:&#39;minor&#39;) else increment\_version\_number\_in\_plist(xcodeproj:&#39;ios/Project.xcodeproj&#39;,bump\_type:&#39;patch&#39;) |
| --- |

###### **before/after\_all action**

1. **before\_all**

**ensure\_git\_status\_clean** → this is done to make sure that there is no uncommitted change or anything.

**ensure\_git\_branch** → ensures this is the master branch or the branch you need to work on.

**git\_pull →** executes a simple git pull command to pull together all the changes

Fastlane docs: [https://docs.fastlane.tools/actions/ensure\_git\_status\_clean/](https://docs.fastlane.tools/actions/ensure_git_status_clean/)

[https://docs.fastlane.tools/actions/ensure\_git\_branch/](https://docs.fastlane.tools/actions/ensure_git_branch/)

[https://docs.fastlane.tools/actions/git\_pull/](https://docs.fastlane.tools/actions/git_pull/)

1. **after\_all**

**reset\_git\_repo** → discard all uncommitted changes

Fastlane docs: [https://docs.fastlane.tools/actions/reset\_git\_repo/](https://docs.fastlane.tools/actions/reset_git_repo/)

###### **Project Code Reference** : [https://github.com/shanzaygauhar/Fastlane](https://github.com/shanzaygauhar/Fastlane)

##### **Errors I faced:**

**MATCH TYPE: &quot;DEVELOPMENT&quot;** → stuck at `Running script &#39;[CP] Copy Pods Resources&#39;

**MATCH TYPE: &quot; APPSTORE&quot; →** asked me for a development certificate.

Probable solution → update\_project\_team

# cakephp-updater
Create a script that automagically updates your cakephp3 app to the latest
version

These instructions are made for [Cakephp](http://cakephp.org/) v3. Use on your
own risk.

Improvements are welcome - just make a pull request :)

# Introduction
Cakephp releases minor updates to its framework nearly every two weeks.
When I started developing my own cakephp app (cakephp was in alpha stage back
then) I started updating cakephp as inplace upgrade until I noticed some odd
behaviours:
* my vendor folder wasn't cleaned up, so it got bigger and bigger - uploading
the app via ftp was getting slower
* some changes to the cakephp core are not visible when updating because
existing files are not overwritten - e.g. changes to the *app_default.php* that
may be important for your *app.php*.

I ended up with an app, full of pieces of junk that behaved sometimes
irrational - at one time during alpha, cakephp changed the integration of
plugins which caused me some hours until I figured out what happend. By using
my script I see changes to the *app_default.php* and other files. 

# Requirements / Assumptions
1. use git
1. use [Composer](https://getcomposer.org/) for dependency management
1. directory structure of your repository looks like
		/.git
		/app			<= this is your cakephp app
			|bin
			|config
			|src
			|vendor
			|plugins
			|tests
			|webroot
		/scripts		<= the script will be dropped here
		/composer.phar

# Stop talking - gimme that script
Because each cakephp app has it's customizations I can just give you a skeleton
and some instructions to adopt it to your needs.

# Script Skeleton
	#!/bin/bash
	# Source of this script: https://github.com/michha/cakephp-updater
	
	# Exit the script as soon as an executed command does not succeed.
	set -o errexit
	# Exit the script when an unintialized variable is used 
	set -o nounset
	
	echo "Hello $USER"
	echo "Please run me from the root directory (containing your app folder)"
	echo "and be sure that no webserver (apache, cake server) is running"
	
	########################### CONFIGURATION ###########################
	backupDir="app.old.$(date +%Y%m%d)"
	########################### CONFIGURATION ###########################
	
	continueQuestion() {
	    while true; do
	        read -p "Continue? (next step: $1) " yn
	        case $yn in
	            [Yy]* ) break;;
	            [Nn]* ) exit;;
	            * ) echo "Please answer y(es) or n(o).";;
	        esac
	    done
	}
	
	continueQuestion "Execute Script"
	
	echo "updating composer..."
	./composer.phar self-update
	
	echo "renaming app/ to $backupDir..."
	mv "app/" "$backupDir"
	
	echo "creating app..."
	./composer.phar create-project --prefer-dist cakephp/app app
	
	echo "replacing default folders with old one..."
	rm -r -d app/plugins/
	rm -r -d app/src/
	rm -r -d app/tests/
	
	cp -r $backupDir/plugins/ app/plugins/
	cp -r $backupDir/src/ app/src/
	cp -r $backupDir/tests/ app/tests/
	
	copyFile() {
	    cp -f $backupDir/$1 app/$1
	}
	
	echo "copying some files from old app..."
	# do not copy files created by *create-project*
	# we would not detect changes in cake  files
	# exception: favicon.ico
	copyFile "webroot/favicon.ico"
	#copyFile "webroot/custom.file"
	#copyFile "webroot/js/custom.js"
	#copyFile "config/acl.ini"
	copyFile "config/app.php"
	#copyFile "config/app_custom.php"
	#copyFile "config/app_local.php"
	
	echo "cleaning files up..."
	#rm app/webroot/js/empty
	
	# TODO uncomment if you have additional packages
	#cd app
	#echo "adding required packages via composer..."
	#../composer.phar require cakephp/cakephp-codesniffer=2.* dereuromark/cakephp-tinyauth=dev-master
	#cd ..
	
	# TODO uncomment after adopting the script
	#echo "patching files..."
	#patch -p1 < scripts/update-cake-aftermath.patch
	
	cd app
	echo "dump composer autoload..."
	../composer.phar dump-autoload
	cd ..
	
	echo "update successful :)"

# Preparations
1. add the line
		app.old*
to your */.gitignore* file so backups wont be picked up by git
1. make sure you have no pending changes in your repository

# Adoption
1. save the script from above to a file in */scripts/* and run it
1. run *git status* to get a list of the differences
1. now is the time to review the diffs and think about each one
	* is it intended? somethings you mistakenly added files to your repository
or set wrong file mode bits
	* if you see something that shouldn't be in your repository clean it up now
before moving on
1. so right here, every pending change should be done by the script:
	1. if you see some removes, e.g. in your webroot, expand the script block
*copying some files from old app...* to include these files
	1. if you see some adds, e.g. *empty* files, expand the script block
*cleaning files up...* to include these files
	1. if you see some changed files, eg. */app/.gitignore*, */app/composer.json*,
you need to create a patch file
			git diff -R > update-cake-aftermath.patch
1. time for testing the automagic stuff
	1. delete your app folder
	1. rename the backuped up *app.old.DATE* to *app*
	1. run the script again
	1. *git status* should now just show diffs that result from updating your
cakephp
1. congratulations - updating your cake app to the newest bits is now just a
matter of seconds. Independent how easy updating is - always review the changes
before commiting to your repository.

# Explaining the innards of the script
1. update composer, just to make sure it has everything needed for updating
1. renaming (instead of deleting) the current app directory
1. create a new cakephp app 
1. copy directories *plugins*, *src* and *test* containing your code into the
app
1. copy some files outside of these directories that are not part of the
default cakephp app
1. cleanup some *empty* files
1. adding required packages
1. applying patch file for updating files that are elements of the default
cakephp app
1. clear autoload cache - fixes MissingHelperException if you have custom
helper classes

# Why?
### Before CI
- Developers push to remote repository
- Every once in a while, someone pull from the repo, build, and run test
### After CI
- Every time there is a commit, CI server will pull the latest code from the repo to try to build and test it
- If anything fails, developer will be notified
- If everything pass, build will be stored as artifacts or even can be deployed by CI server

[[Introduction to Jenkins]]
[[Making a Debian Package with Jenkins]]
[[Making a NuGet Package with Jenkins]]